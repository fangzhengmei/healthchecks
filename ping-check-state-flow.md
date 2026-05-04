# Ping Check State Flow 分析报告

## 1. 概述

本文档详细分析 Healthchecks 系统中 ping 接收、grace period 处理以及状态从正常（up）变为异常（down）的完整流程。

---

## 2. 核心概念与数据模型

### 2.1 状态定义

系统定义了以下四种状态（存储在数据库中）：

```python
STATUSES = (("up", "Up"), ("down", "Down"), ("new", "New"), ("paused", "Paused"))
```
[hc/api/models.py:37](hc/api/models.py#L37-L37)

此外，还有一个用于显示的"grace"状态（不存储在数据库中），表示检查已超时但仍在宽限期内。

### 2.2 关键字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `timeout` | DurationField | 1天 | 超时时间，simple类型检查的预期ping间隔 |
| `grace` | DurationField | 1小时 | 宽限期，超时后等待的额外时间 |
| `last_ping` | DateTimeField | null | 最后一次成功ping的时间 |
| `last_start` | DateTimeField | null | 最后一次start事件的时间 |
| `alert_after` | DateTimeField | null | 下次需要检查状态的时间 |
| `status` | CharField | "new" | 当前状态 |

[hc/api/models.py:184-215](hc/api/models.py#L184-L215)

---

## 3. Ping 接收与处理流程

### 3.1 入口视图

ping 请求通过 `ping()` 视图函数处理：

```python
def ping(
    request: HttpRequest,
    code: UUID,
    check: Check | None = None,
    action: str = "success",
    exitstatus: int | None = None,
) -> HttpResponse:
```
[hc/api/views.py:180-186](hc/api/views.py#L180-L186)

### 3.2 Ping 处理步骤

1. **查找 Check 对象**：根据 UUID 查找对应的检查
2. **解析请求参数**：
   - `remote_addr`：客户端 IP
   - `method`：HTTP 方法
   - `ua`：User-Agent
   - `body`：请求体
3. **确定 action 类型**：
   - 默认 `"success"`
   - 如果 `exitstatus > 0`，设为 `"fail"`
   - 如果检查配置了 `methods="POST"` 但请求不是 POST，设为 `"ign"`
   - 如果配置了 `filter_http_body`，根据请求体内容判断 action
4. **调用 Check.ping() 方法**处理业务逻辑

[hc/api/views.py:180-243](hc/api/views.py#L180-L243)

### 3.3 Check.ping() 核心逻辑

```python
def ping(
    self,
    remote_addr: str,
    scheme: str,
    method: str,
    ua: str,
    body: bytes,
    action: str,
    rid: uuid.UUID | None,
    exitstatus: int | None = None,
) -> None:
```
[hc/api/models.py:467-477](hc/api/models.py#L467-L477)

**处理流程：**

1. **事务与锁**：使用数据库事务和行级锁防止并发问题
2. **根据 action 类型处理**：
   - **"start"**：更新 `last_start` 和 `last_start_rid`
   - **"ign"** / **"log"**：不更新状态，仅记录 ping
   - **"fail"**：设置新状态为 "down"
   - **"success"**（默认）：设置新状态为 "up"
3. **状态转换处理**：
   - 如果当前状态与新状态不同，创建 `Flip` 对象记录状态变化
   - 更新 `self.status` 为新状态
4. **更新 alert_after**：
   ```python
   self.alert_after = self.going_down_after()
   ```
5. **创建 Ping 记录**：记录本次 ping 的详细信息

[hc/api/models.py:467-552](hc/api/models.py#L467-L552)

---

## 4. Grace Period 计算机制

### 4.1 核心方法

#### get_grace_start()

计算 grace period 的开始时间：

```python
def get_grace_start(self, *, with_started: bool = True) -> datetime | None:
    """Return the datetime when the grace period starts.

    If the check is currently new, paused or down, return None.
    """
```
[hc/api/models.py:292-296](hc/api/models.py#L292-L296)

**计算逻辑：**

1. **Simple 类型检查**：
   ```python
   if self.kind == "simple" and self.status == "up":
       result = self.last_ping + self.timeout
   ```
   - grace_start = 最后一次ping时间 + 超时时间

2. **Cron 类型检查**：
   ```python
   elif self.kind == "cron" and self.status == "up":
       last_local = self.last_ping.astimezone(ZoneInfo(self.tz))
       result = next(CronSim(self.schedule, last_local))
       result = result.astimezone(timezone.utc)
   ```
   - 根据 cron 表达式计算下次预期时间

3. **OnCalendar 类型检查**：
   类似 cron，使用 `OnCalendar` 库计算

4. **考虑 running 状态**：
   ```python
   if with_started and self.last_start and self.status != "down":
       result = min(result, self.last_start)
   ```
   - 如果有 last_start，取更早的时间作为 grace_start

[hc/api/models.py:292-329](hc/api/models.py#L292-L329)

#### going_down_after()

计算检查变为 down 的时间：

```python
def going_down_after(self) -> datetime | None:
    """Return the datetime when the check goes down.

    If the check is new or paused, and not currently running, return None.
    If the check is already down, also return None.
    """
    grace_start = self.get_grace_start()
    if grace_start is not None:
        return grace_start + self.grace
    return None
```
[hc/api/models.py:331-341](hc/api/models.py#L331-L341)

**公式：**
```
going_down_after = grace_start + grace_period
```

---

## 5. 状态判断逻辑

### 5.1 get_status() 方法

该方法返回当前状态用于显示，可能返回 "up"、"down"、"grace"、"new"、"paused"、"started"：

```python
def get_status(self, *, with_started: bool = False) -> str:
    """Return current status for display."""
    frozen_now = now()
```
[hc/api/models.py:347-349](hc/api/models.py#L347-L349)

**判断流程：**

1. **检查 running 状态**：
   ```python
   if self.last_start:
       if frozen_now >= self.last_start + self.grace:
           return "down"
       elif with_started:
           return "started"
   ```
   - 如果 running 时间超过 grace，返回 "down"
   - 如果 `with_started=True` 且仍在 grace 内，返回 "started"

2. **检查终态**：
   ```python
   if self.status in ("new", "paused", "down"):
       return self.status
   ```

3. **计算 grace 时间范围**：
   ```python
   grace_start = self.get_grace_start(with_started=False)
   if grace_start is None:
       return "up"  # 永远不会超时
   
   grace_end = grace_start + self.grace
   ```

4. **判断当前状态**：
   ```python
   if frozen_now >= grace_end:
       return "down"
   if frozen_now >= grace_start:
       return "grace"
   return "up"
   ```

[hc/api/models.py:347-372](hc/api/models.py#L347-L372)

### 5.2 状态转换时间线

```
时间轴：
├───[last_ping]─────────────────────────────────────►
│
├───[grace_start = last_ping + timeout]─────────────►
│   │
│   ├─── 在此之前：状态为 "up"
│   │
│   ├───[grace_end = grace_start + grace]──────────►
│   │   │
│   │   ├─── grace_start 到 grace_end 之间：
│   │   │    显示状态为 "grace"
│   │   │    数据库状态仍为 "up"
│   │   │
│   │   └─── grace_end 之后：
│   │        需要 sendalerts 进程将状态变为 "down"
```

---

## 6. 状态从 Up 变为 Down 的流程

### 6.1 触发机制

状态从 "up" 变为 "down" 不是自动发生的，而是由 `sendalerts` 管理命令定期检查处理。

**sendalerts 主循环**：

```python
while not self.shutdown:
    # Create flips for any checks going down
    while self.handle_going_down() and not self.shutdown:
        pass

    # Submit unprocessed flips to the self.executor
    while self.process_one_flip() and not self.shutdown:
        pass

    # Wait a bit
    if not self.shutdown:
        time.sleep(2)
```
[hc/api/management/commands/sendalerts.py:199-211](hc/api/management/commands/sendalerts.py#L199-L211)

### 6.2 handle_going_down() 方法

这是处理状态变为 down 的核心方法：

```python
def handle_going_down(self) -> bool:
    """Process a single check going down.

    1. Find a check with alert_after in the past, and status other than "down".
    2. Calculate its current status.
    3. If calculation throws an exception, push alert_after forward and re-raise.
    4. If the current status is not "down", update alert_after and return.
    5. Update the check's status in the database to "down".
    6. If exactly 1 row gets updated, create a Flip object.
    """
```
[hc/api/management/commands/sendalerts.py:121-131](hc/api/management/commands/sendalerts.py#L121-L131)

### 6.3 详细处理步骤

**步骤 1：查找需要检查的 Check**

```python
q = Check.objects.filter(alert_after__lt=now()).exclude(status="down")
check = q.order_by("alert_after").first()
```
[hc/api/management/commands/sendalerts.py:133-135](hc/api/management/commands/sendalerts.py#L133-L135)

- 查找条件：`alert_after < 当前时间` 且 `status != "down"`

**步骤 2：计算当前状态**

```python
try:
    status = check.get_status()
except Exception as e:
    # 异常处理：推迟 1 小时再检查
    q.update(alert_after=now() + td(hours=1))
    raise e
```
[hc/api/management/commands/sendalerts.py:142-149](hc/api/management/commands/sendalerts.py#L142-L149)

**步骤 3：判断是否需要变为 down**

```python
if status != "down":
    # 还没到 down 状态，更新 alert_after 下次再检查
    q.update(alert_after=check.going_down_after())
    return True
```
[hc/api/management/commands/sendalerts.py:151-154](hc/api/management/commands/sendalerts.py#L151-L154)

**步骤 4：状态变为 down**

```python
flip_time = check.going_down_after()
assert flip_time

# 原子更新状态
num_updated = q.update(alert_after=None, status="down")
if num_updated != 1:
    # 其他进程已处理
    return True

# 创建 Flip 对象记录状态变化
flip = Flip(owner=check)
flip.created = flip_time  # 注意：使用 going_down_after 的时间，不是当前时间
flip.old_status = old_status
flip.new_status = "down"
flip.reason = "timeout"
flip.save()
```
[hc/api/management/commands/sendalerts.py:156-173](hc/api/management/commands/sendalerts.py#L156-L173)

### 6.4 状态转换的关键点

1. **alert_after 的作用**：
   - 每次收到 ping 时更新：`self.alert_after = self.going_down_after()`
   - sendalerts 根据这个字段判断是否需要检查

2. **状态判断的时机**：
   - `get_status()` 方法可以实时返回 "down" 或 "grace"
   - 但数据库中的 `status` 字段只有 sendalerts 进程才会更新为 "down"

3. **Flip 对象的 created 时间**：
   - 不是当前时间，而是 `going_down_after()` 的计算时间
   - 这样可以准确记录检查实际应该变为 down 的时间

---

## 7. 完整流程图

### 7.1 Ping 处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                      收到 Ping 请求                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 根据 UUID 查找 Check 对象                                 │
│  2. 解析请求参数 (IP, method, UA, body)                      │
│  3. 根据条件确定 action 类型                                   │
│     - exitstatus>0 → "fail"                                  │
│     - 方法不匹配 → "ign"                                      │
│     - body 关键词匹配 → 根据配置决定                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Check.ping() 方法                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. 开启事务，获取行级锁                                  │  │
│  │ 2. 根据 action 处理：                                    │  │
│  │    - "start": 更新 last_start                          │  │
│  │    - "fail": 新状态为 "down"                           │  │
│  │    - "success": 新状态为 "up"                          │  │
│  │ 3. 如果状态变化，创建 Flip 对象                          │  │
│  │ 4. 更新 alert_after = going_down_after()               │  │
│  │ 5. 创建 Ping 记录                                        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      返回 "OK" 响应                           │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 状态变为 Down 的流程

```
┌─────────────────────────────────────────────────────────────┐
│                 sendalerts 进程 (定期运行)                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  查找条件：alert_after < now() AND status != "down"          │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   找到这样的 Check？   │
              └───────────┬───────────┘
                    │             │
                   否            是
                    │             │
                    ▼             ▼
              ┌─────────┐   ┌──────────────────┐
              │ 等待 2s │   │ get_status() 计算 │
              └─────────┘   │   当前状态        │
                            └────────┬─────────┘
                                     │
                        ┌────────────┴────────────┐
                        │  get_status() == "down"? │
                        └────────────┬────────────┘
                              │              │
                             否             是
                              │              │
                              ▼              ▼
                    ┌──────────────┐  ┌─────────────────────┐
                    │ 更新         │  │ 原子更新：           │
                    │ alert_after  │  │ status = "down"     │
                    │ 下次再检查   │  │ alert_after = null   │
                    └──────────────┘  └──────────┬──────────┘
                                                   │
                                                   ▼
                                         ┌──────────────────┐
                                         │ 创建 Flip 对象    │
                                         │ - reason="timeout"│
                                         │ - created=实际变   │
                                         │   为down的时间     │
                                         └──────────────────┘
                                                   │
                                                   ▼
                                         ┌──────────────────┐
                                         │ process_one_flip │
                                         │ 发送通知给用户    │
                                         └──────────────────┘
```

---

## 8. 关键代码位置汇总

| 功能 | 文件 | 行号 |
|------|------|------|
| 状态定义 | hc/api/models.py | 37 |
| Check 模型字段 | hc/api/models.py | 184-215 |
| get_grace_start() | hc/api/models.py | 292-329 |
| going_down_after() | hc/api/models.py | 331-341 |
| get_status() | hc/api/models.py | 347-372 |
| Check.ping() | hc/api/models.py | 467-552 |
| ping 视图 | hc/api/views.py | 180-243 |
| sendalerts 主循环 | hc/api/management/commands/sendalerts.py | 199-211 |
| handle_going_down() | hc/api/management/commands/sendalerts.py | 121-175 |

---

## 9. 总结

### 9.1 核心要点

1. **Grace Period 的两层含义**：
   - 显示层面：`get_status()` 返回 "grace" 表示已超时但在宽限期内
   - 数据库层面：`status` 字段仍为 "up"，直到 sendalerts 进程处理

2. **状态变为 Down 的两个条件**：
   - 当前时间 >= `grace_start + grace`（即 `going_down_after()`）
   - sendalerts 进程执行 `handle_going_down()` 方法

3. **alert_after 的作用**：
   - 作为"闹钟"机制，告诉 sendalerts 何时需要检查
   - 每次 ping 后更新为 `going_down_after()`

4. **Flip 对象的重要性**：
   - 记录状态变化的精确时间（使用 `going_down_after()` 的计算值）
   - 用于异步发送通知
   - 用于计算停机时间统计

### 9.2 时间线示例

假设配置：
- `timeout = 1 小时`
- `grace = 30 分钟`

事件序列：
1. **10:00**：收到 ping，`last_ping = 10:00`
   - `grace_start = 11:00`
   - `alert_after = 11:30`
   - 状态："up"

2. **11:15**：无 ping
   - `get_status()` 返回 "grace"
   - 数据库 `status` 仍为 "up"

3. **11:30**：无 ping
   - `alert_after` 已过期
   - `get_status()` 返回 "down"
   - 但数据库 `status` 仍为 "up"

4. **11:32**：sendalerts 进程运行
   - 发现 `alert_after < now()`
   - 调用 `get_status()` 返回 "down"
   - 更新数据库 `status = "down"`
   - 创建 Flip 对象，`reason = "timeout"`
   - 发送通知给用户
