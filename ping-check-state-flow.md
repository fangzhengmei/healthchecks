# Ping Check State Flow 分析报告

## 1. 概述

本文档详细分析 Healthchecks 系统中 ping 接收、grace period 处理以及状态从正常（up）变为异常（down）的完整流程。

**特别关注**：三种状态变为异常的触发条件及其优先级和触发时机：
1. **失败心跳（fail action）**：主动发送的失败信号
2. **running 超过宽限期**：start 后未收到结束信号
3. **超时后的异步降级**：正常超时 + grace period

---

## 2. 核心概念与数据模型

### 2.1 状态定义

系统定义了以下四种状态（存储在数据库中）：

```python
STATUSES = (("up", "Up"), ("down", "Down"), ("new", "New"), ("paused", "Paused"))
```
[hc/api/models.py:37](hc/api/models.py#L37-L37)

此外，还有两个用于显示的状态（不存储在数据库中）：
- **"grace"**：表示检查已超时但仍在宽限期内
- **"started"**：表示正在运行中（需 `with_started=True`）

### 2.2 关键字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `timeout` | DurationField | 1天 | 超时时间，simple类型检查的预期ping间隔 |
| `grace` | DurationField | 1小时 | 宽限期，超时后等待的额外时间 |
| `last_ping` | DateTimeField | null | 最后一次成功ping的时间 |
| `last_start` | DateTimeField | null | 最后一次start事件的时间 |
| `alert_after` | DateTimeField | null | 下次需要检查状态的时间 |
| `status` | CharField | "new" | 当前状态（存储在数据库中） |

[hc/api/models.py:184-215](hc/api/models.py#L184-L215)

---

## 3. 三种异常触发条件详解

### 3.1 条件一：失败心跳（Fail Action）

**场景**：收到 `/fail` 路径的 ping，或 exitstatus > 0 的 ping。

**处理方式**：**同步，立即生效**

**核心代码**（Check.ping() 方法）：

```python
else:  # action 是 success 或 fail
    self.last_ping = frozen_now
    self.last_duration = None
    
    # 清除 last_start（如果有的话）
    if self.last_start:
        if self.last_start_rid == rid:
            self.last_duration = self.last_ping - self.last_start
            self.last_start = None
        elif action == "fail" or rid is None:
            self.last_start = None  # fail 事件会清除 running 状态

    # 关键：立即设置新状态
    new_status = "down" if action == "fail" else "up"
    if self.status != new_status:
        reason = "fail" if action == "fail" else ""
        self.create_flip(new_status, reason=reason)  # 立即创建 Flip
        self.status = new_status  # 立即更新数据库状态

    self.alert_after = self.going_down_after()
    # ...
```
[hc/api/models.py:499-519](hc/api/models.py#L499-L519)

**触发时机**：收到 fail ping 的瞬间

**效果**：
1. 立即将数据库 `status` 设为 "down"
2. 立即创建 Flip 对象，`reason = "fail"`
3. `alert_after = going_down_after()`（状态为 down 时返回 None）

**特点**：这是**唯一**能同步改变数据库 `status` 字段的异常触发条件。

---

### 3.2 条件二：Running 超过宽限期

**场景**：收到 `/start` 路径的 ping 后，在 grace 时间内没有收到对应的 success 或 fail ping。

**处理方式**：**异步，需要 sendalerts 进程处理**

#### 3.2.1 Start 事件的处理

当收到 start ping 时：

```python
if action == "start":
    self.last_start = frozen_now
    self.last_start_rid = rid
    # Don't update "last_ping" field.
```
[hc/api/models.py:491-494](hc/api/models.py#L491-L494)

**注意**：start 事件只更新 `last_start`，**不更新** `last_ping`。

#### 3.2.2 get_status() 中的优先级判断

```python
def get_status(self, *, with_started: bool = False) -> str:
    frozen_now = now()

    # 优先级 1：首先检查 running 超时
    if self.last_start:
        if frozen_now >= self.last_start + self.grace:
            return "down"  # running 超时，直接返回 "down"
        elif with_started:
            return "started"

    # 优先级 2：检查数据库中的终态
    if self.status in ("new", "paused", "down"):
        return self.status

    # 优先级 3：检查正常超时（基于 last_ping）
    grace_start = self.get_grace_start(with_started=False)
    if grace_start is None:
        return "up"
    
    grace_end = grace_start + self.grace
    if frozen_now >= grace_end:
        return "down"
    if frozen_now >= grace_start:
        return "grace"
    
    return "up"
```
[hc/api/models.py:347-372](hc/api/models.py#L347-L372)

**关键点**：
- `last_start` 的检查**优先级最高**，在检查数据库 `status` 字段之前
- 只要 `now >= last_start + grace`，就返回 "down"
- 但这只是**显示状态**，数据库 `status` 仍为 "up"

#### 3.2.3 alert_after 的计算

```python
def get_grace_start(self, *, with_started: bool = True) -> datetime | None:
    result = NEVER

    # 计算基于 last_ping 的 grace_start
    if self.kind == "simple" and self.status == "up":
        result = self.last_ping + self.timeout
    # ... cron / oncalendar 类似 ...

    # 关键：如果有 last_start，取更早的时间
    if with_started and self.last_start and self.status != "down":
        result = min(result, self.last_start)

    return result if result != NEVER else None

def going_down_after(self) -> datetime | None:
    grace_start = self.get_grace_start()  # with_started=True
    if grace_start is not None:
        return grace_start + self.grace
    return None
```
[hc/api/models.py:292-341](hc/api/models.py#L292-L341)

**公式**：
```
grace_start = min(last_ping + timeout, last_start)  # 如果有 last_start
alert_after = grace_start + grace
```

**含义**：如果有 running 状态，`alert_after` 会取更早的时间（`last_start + grace`）。

---

### 3.3 条件三：超时后的异步降级（正常超时）

**场景**：正常 ping 后，在 `timeout + grace` 时间内没有收到下一个 ping。

**处理方式**：**异步，需要 sendalerts 进程处理**

#### 3.3.1 时间线计算

```python
# Simple 类型
grace_start = last_ping + timeout
grace_end = grace_start + grace = last_ping + timeout + grace

# Cron/OnCalendar 类型
grace_start = 下次预期时间（根据调度表达式计算）
grace_end = grace_start + grace
```

#### 3.3.2 get_status() 中的处理

在 `get_status()` 中，正常超时的检查优先级**低于** running 超时检查：

```python
# 优先级 3：正常超时检查（在 running 检查之后）
grace_start = self.get_grace_start(with_started=False)  # 注意：with_started=False
if grace_start is None:
    return "up"

grace_end = grace_start + self.grace
if frozen_now >= grace_end:
    return "down"
if frozen_now >= grace_start:
    return "grace"
```
[hc/api/models.py:360-371](hc/api/models.py#L360-L371)

**注意**：这里调用 `get_grace_start(with_started=False)`，**不考虑** `last_start`。

---

## 4. 三种条件的统一时间线与优先级

### 4.1 优先级总结

| 条件 | 触发方式 | 数据库 status 更新 | get_status() 优先级 |
|------|----------|-------------------|---------------------|
| 失败心跳 | 同步（收到 fail ping 时） | 立即更新 | N/A（数据库状态已改变） |
| Running 超时 | 异步（需 sendalerts） | 需 sendalerts 更新 | **最高**（第 1 位检查） |
| 正常超时 | 异步（需 sendalerts） | 需 sendalerts 更新 | **较低**（第 3 位检查） |

### 4.2 统一时间线示例

假设配置：
- `timeout = 1 小时`
- `grace = 30 分钟`

**场景**：先收到 success ping，然后收到 start ping，之后没有任何 ping。

```
时间轴：
├─── 10:00 收到 success ping
│      - last_ping = 10:00
│      - last_start = null
│      - status = "up"
│      - grace_start（基于 last_ping）= 11:00
│      - alert_after = 11:30
│
├─── 10:15 收到 start ping
│      - last_start = 10:15（更新）
│      - last_ping = 10:00（不变）
│      - status = "up"（不变）
│      - grace_start = min(11:00, 10:15) = 10:15
│      - alert_after = 10:15 + 30min = 10:45（更新！）
│
├─── 10:30 无 ping
│      - 检查 get_status()：
│        - last_start 存在，检查 10:15 + 30min = 10:45
│        - now(10:30) < 10:45，继续
│      - 数据库 status 仍为 "up"
│      - get_status() 返回 "up"（或 "started" 如果 with_started=True）
│
├─── 10:45 无 ping
│      - alert_after(10:45) 已过期
│      - get_status()：
│        - last_start 存在，检查 10:15 + 30min = 10:45
│        - now(10:45) >= 10:45
│        - 返回 "down"
│      - 但数据库 status 仍为 "up"！
│
├─── 10:46 sendalerts 进程运行
│      - 查找 alert_after < now() 且 status != "down" 的检查
│      - 发现此检查（alert_after=10:45 < 10:46，status="up"）
│      - 调用 get_status() 返回 "down"
│      - 原子更新：status = "down", alert_after = null
│      - 创建 Flip 对象：
│        - created = going_down_after() = 10:45（注意：不是当前时间 10:46）
│        - reason = "timeout"
│        - old_status = "up", new_status = "down"
│      - 发送通知给用户
│
├─── 11:00 无 ping（原本的 grace_start）
│      - 但状态已于 10:45 变为 down，此处无影响
│
├─── 11:30 无 ping（原本的正常超时时间）
│      - 状态已为 down，无影响
```

### 4.3 另一个场景：先 start 后 success

假设配置相同：`timeout = 1 小时`，`grace = 30 分钟`

```
时间轴：
├─── 10:00 收到 start ping
│      - last_start = 10:00
│      - last_ping = null（或之前的值）
│      - alert_after = 10:00 + 30min = 10:30
│
├─── 10:20 收到 success ping（rid 匹配）
│      - last_ping = 10:20（更新）
│      - last_duration = 10:20 - 10:00 = 20min
│      - last_start = null（清除！因为 rid 匹配）
│      - status = "up"
│      - grace_start = 10:20 + 1h = 11:20
│      - alert_after = 11:20 + 30min = 11:50（更新）
│
├─── 10:30 无 ping
│      - 原本的 alert_after(10:30) 已过期
│      - 但 last_start 已被清除，running 超时检查不触发
│      - 正常超时检查：grace_end = 11:50
│      - get_status() 返回 "up"
│
├─── 后续：按正常超时逻辑处理
```

### 4.4 三种条件同时存在的优先级

假设有一个检查：
- `last_ping = 10:00`（1 小时前）
- `last_start = 10:40`（20 分钟前）
- `timeout = 30 分钟`
- `grace = 15 分钟`

计算：
- 正常超时的 `grace_end = 10:00 + 30min + 15min = 10:45`
- Running 超时的 `last_start + grace = 10:40 + 15min = 10:55`

**当前时间 10:50**：

`get_status()` 判断流程：
1. 检查 `last_start`：`now(10:50) >= 10:40 + 15min = 10:55`？**否**（10:50 < 10:55）
2. 检查数据库状态：`status = "up"`
3. 检查正常超时：`now(10:50) >= 10:45`？**是**
4. 返回 `"down"`

**但等等，alert_after 是多少？**

```python
grace_start = min(last_ping + timeout, last_start)
            = min(10:00 + 30min = 10:30, 10:40)
            = 10:30

alert_after = grace_start + grace = 10:30 + 15min = 10:45
```

所以 `alert_after = 10:45`，sendalerts 会在 10:45 后触发检查。

**关键点**：
- `alert_after` 的计算考虑 `last_start`（取 min）
- 但 `get_status()` 中 running 超时的判断是独立的
- 如果 running 超时时间（10:55）晚于正常超时时间（10:45），则**正常超时先触发**

---

## 5. Ping 接收与处理流程

### 5.1 入口视图

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

### 5.2 Action 类型的确定

```python
# 1. exitstatus > 0 时设为 fail
if exitstatus is not None and exitstatus > 0:
    action = "fail"

# 2. 方法不匹配时设为 ign
if check.methods == "POST" and method != "POST":
    action = "ign"

# 3. 如果配置了 filter_http_body，根据请求体内容判断
if action != "ign" and check.filter_http_body:
    body_text = body.decode()
    if check.failure_kw and match_keywords(body_text, check.failure_kw):
        action = "fail"
    elif check.success_kw and match_keywords(body_text, check.success_kw):
        action = "success"
    elif check.start_kw and match_keywords(body_text, check.start_kw):
        action = "start"
    elif check.filter_default_fail:
        action = "fail"
    else:
        action = "ign"
```
[hc/api/views.py:212-229](hc/api/views.py#L212-L229)

### 5.3 Check.ping() 核心逻辑

```python
def ping(self, ..., action: str, ...) -> None:
    with transaction.atomic():
        self = Check.objects.select_for_update().get(id=self.id)
        frozen_now = now()

        if self.status == "paused" and self.manual_resume:
            action = "ign"

        if action == "start":
            self.last_start = frozen_now
            self.last_start_rid = rid
            # Don't update "last_ping" field.
        elif action == "ign":
            pass
        elif action == "log":
            pass
        else:  # success 或 fail
            self.last_ping = frozen_now
            self.last_duration = None
            if self.last_start:
                if self.last_start_rid == rid:
                    self.last_duration = self.last_ping - self.last_start
                    self.last_start = None
                elif action == "fail" or rid is None:
                    self.last_start = None

            # 状态转换
            new_status = "down" if action == "fail" else "up"
            if self.status != new_status:
                reason = "fail" if action == "fail" else ""
                self.create_flip(new_status, reason=reason)
                self.status = new_status

        self.alert_after = self.going_down_after()
        self.n_pings = models.F("n_pings") + 1
        self.save()

        # 创建 Ping 记录...
```
[hc/api/models.py:467-542](hc/api/models.py#L467-L542)

---

## 6. 异步降级机制（sendalerts）

### 6.1 sendalerts 主循环

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

```python
def handle_going_down(self) -> bool:
    # 步骤 1：查找 alert_after 已过期且状态不是 down 的检查
    q = Check.objects.filter(alert_after__lt=now()).exclude(status="down")
    check = q.order_by("alert_after").first()
    if check is None:
        return False

    old_status = check.status
    q = Check.objects.filter(id=check.id, status=old_status)

    # 步骤 2：计算当前状态（调用 get_status()）
    try:
        status = check.get_status()
    except Exception as e:
        q.update(alert_after=now() + td(hours=1))
        raise e

    # 步骤 3：如果不是 down，更新 alert_after 下次再检查
    if status != "down":
        q.update(alert_after=check.going_down_after())
        return True

    # 步骤 4：状态变为 down
    flip_time = check.going_down_after()
    assert flip_time

    # 原子更新
    num_updated = q.update(alert_after=None, status="down")
    if num_updated != 1:
        return True  # 其他进程已处理

    # 创建 Flip 对象
    flip = Flip(owner=check)
    flip.created = flip_time  # 注意：使用 going_down_after() 的时间，不是当前时间
    flip.old_status = old_status
    flip.new_status = "down"
    flip.reason = "timeout"
    flip.save()

    return True
```
[hc/api/management/commands/sendalerts.py:121-175](hc/api/management/commands/sendalerts.py#L121-L175)

### 6.3 关键点理解

1. **alert_after 的作用**：
   - 作为"闹钟"机制，告诉 sendalerts 何时需要检查
   - 每次 ping 后更新为 `going_down_after()`
   - `going_down_after()` = `min(grace_start, last_start) + grace`（如果有 last_start）

2. **Flip.created 的时间**：
   - 不是当前时间，而是 `going_down_after()` 的计算时间
   - 这样可以准确记录检查**实际应该**变为 down 的时间，而不是 sendalerts 处理的时间

3. **两种异步触发的统一处理**：
   - `handle_going_down()` 不区分是 running 超时还是正常超时
   - 它只依赖 `get_status()` 的返回值
   - 而 `get_status()` 内部已经处理了优先级

---

## 7. 完整流程图

### 7.1 Ping 处理流程

```
┌─────────────────────────────────────────────────────────────┐
│                      收到 Ping 请求                           │
│  (路径可能是 /success, /fail, /start, /{exitstatus})        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 根据 UUID 查找 Check 对象                                 │
│  2. 解析请求参数 (IP, method, UA, body, rid, exitstatus)     │
│  3. 确定 action 类型：                                         │
│     - exitstatus>0 → "fail"                                  │
│     - 方法不匹配 → "ign"                                      │
│     - filter_http_body 匹配 → 根据关键词决定                   │
│     - 默认 → "success"                                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Check.ping() 方法                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. 开启事务，获取行级锁                                  │  │
│  │ 2. 根据 action 处理：                                    │  │
│  │    ├─ "start": 更新 last_start, last_start_rid          │  │
│  │    │         不更新 last_ping                            │  │
│  │    ├─ "ign"/"log": 不更新状态                            │  │
│  │    └─ "success"/"fail":                                 │  │
│  │         ├─ 更新 last_ping                                │  │
│  │         ├─ 如果有 last_start 且 rid 匹配：               │  │
│  │         │   计算 last_duration，清除 last_start          │  │
│  │         ├─ 如果 action=="fail" 或 rid==None：           │  │
│  │         │   清除 last_start                              │  │
│  │         └─ 状态转换：                                     │  │
│  │             ├─ action=="fail" → new_status="down"       │  │
│  │             ├─ action=="success" → new_status="up"      │  │
│  │             └─ 如果状态变化：                             │  │
│  │                 创建 Flip（reason="fail" 或 ""）          │  │
│  │                 更新 status 字段                          │  │
│  │ 3. 更新 alert_after = going_down_after()                 │  │
│  │ 4. 创建 Ping 记录                                         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                      返回 "OK" 响应                           │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 异步降级流程（sendalerts）

```
┌─────────────────────────────────────────────────────────────┐
│                 sendalerts 进程 (定期运行)                    │
│                    主循环：每 2 秒检查一次                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  查找条件：                                                   │
│  alert_after < now() AND status != "down"                   │
│  按 alert_after 升序排列，取最早的一个                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │   找到这样的 Check？   │
              └───────────┬───────────┘
                    │             │
                   否            是
                    │             │
                    ▼             ▼
              ┌─────────┐   ┌──────────────────────────┐
              │ 等待 2s │   │ 调用 check.get_status()   │
              └─────────┘   │ 计算当前显示状态          │
                            └──────────┬───────────────┘
                                       │
                        ┌──────────────┴──────────────┐
                        │  get_status() == "down"?    │
                        └──────────────┬──────────────┘
                              │                │
                             否               是
                              │                │
                              ▼                ▼
                    ┌──────────────┐  ┌─────────────────────────┐
                    │ 更新         │  │ 原子更新数据库：          │
                    │ alert_after  │  │ status = "down"          │
                    │ = going_down │  │ alert_after = null        │
                    │ _after()     │  └──────────┬──────────────┘
                    │ 下次再检查   │             │
                    └──────────────┘             │
                                                 ▼
                                       ┌───────────────────┐
                                       │ 创建 Flip 对象     │
                                       │ - created =        │
                                       │   going_down_after │
                                       │   (不是当前时间)    │
                                       │ - reason="timeout" │
                                       └──────────┬────────┘
                                                  │
                                                  ▼
                                       ┌───────────────────┐
                                       │ process_one_flip  │
                                       │ 异步发送通知给用户  │
                                       └───────────────────┘
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

### 9.1 三种异常触发条件对比

| 条件 | 触发方式 | 数据库 status | get_status 优先级 | Flip.reason | 典型场景 |
|------|----------|---------------|-------------------|-------------|----------|
| **失败心跳** | 同步 | 立即更新为 "down" | N/A | `"fail"` | 脚本执行失败，主动调用 `/fail` |
| **Running 超时** | 异步 | 需 sendalerts 更新 | **最高** | `"timeout"` | 调用 `/start` 后未收到结束信号 |
| **正常超时** | 异步 | 需 sendalerts 更新 | 较低 | `"timeout"` | 定期任务未按时执行 |

### 9.2 核心要点

1. **失败心跳是唯一同步异常**：
   - 收到 fail ping 时立即更新数据库 `status` 为 "down"
   - 立即创建 Flip 对象，`reason = "fail"`
   - 这是最紧急的异常，用户需要立即感知

2. **Running 超时优先级高于正常超时**：
   - `get_status()` 中首先检查 `last_start`
   - `alert_after` 计算时取 `min(grace_start, last_start) + grace`
   - 这意味着：如果有 running 状态，会更早触发检查

3. **异步异常的"实际发生时间"**：
   - `Flip.created` 使用 `going_down_after()` 的计算时间
   - 不是 sendalerts 处理的当前时间
   - 这样可以准确计算停机时长

4. **显示状态与数据库状态的分离**：
   - `get_status()` 返回的是显示状态
   - 数据库 `status` 字段只有两种情况会改变：
     - 收到 fail ping（同步）
     - sendalerts 处理异步超时

### 9.3 时间线总览示例

配置：`timeout = 1 小时`，`grace = 30 分钟`

```
事件时间线：

10:00 收到 success ping
  - last_ping = 10:00
  - status = "up"
  - alert_after = 11:30

10:15 收到 start ping
  - last_start = 10:15
  - alert_after = min(11:00, 10:15) + 30min = 10:45（更新）

10:30 无 ping
  - get_status() 返回 "up"（或 "started"）

10:45 无 ping
  - alert_after 已过期
  - get_status() 返回 "down"（因为 running 超时）
  - 但数据库 status 仍为 "up"

10:46 sendalerts 处理
  - 更新数据库 status = "down"
  - 创建 Flip，created = 10:45，reason = "timeout"
  - 发送通知

10:50 收到 fail ping（假设用户手动触发）
  - 清除 last_start（因为 action == "fail"）
  - new_status = "down"
  - 当前 status 已经是 "down"，不创建 Flip
  - alert_after = None

11:30 原本的正常超时时间
  - 状态已为 down，无影响
```

### 9.4 设计意图理解

1. **两层状态设计**：
   - 显示状态（`get_status()`）：实时计算，用于 UI 展示和 API 返回
   - 数据库状态（`status` 字段）：只有明确的状态变化才更新
   - 分离原因：异步处理需要时间，避免"状态已变但通知未发"的不一致

2. **alert_after 的"闹钟"机制**：
   - 不是每秒检查所有检查，而是按 `alert_after` 排序
   - 效率更高，尤其在检查数量多的时候
   - 每次 ping 后重新计算，确保准确性

3. **Flip.created 的回溯时间**：
   - 使用 `going_down_after()` 而不是当前时间
   - 这样停机时间统计更准确
   - 通知中显示的"已停机 X 分钟"也更准确

---

## 10. 复杂场景分析：Start → Fail → Success 同一次检查

### 10.1 场景描述

用户场景：**同一次检查（同一个 rid）先收到 start，随后 fail，再收到 success**

需要回答的问题：
- 数据库状态最终以哪次事件为准？
- Flip 记录如何创建？
- 通知发送如何触发？

### 10.2 核心代码逻辑回顾

```python
else:  # action 是 success 或 fail
    self.last_ping = frozen_now
    self.last_duration = None
    
    # 处理 running 状态
    if self.last_start:
        if self.last_start_rid == rid:
            # rid 匹配：计算 last_duration，清除 last_start
            self.last_duration = self.last_ping - self.last_start
            self.last_start = None
        elif action == "fail" or rid is None:
            # fail 事件无论 rid 是否匹配，都会清除 last_start
            # success 事件如果没有 rid，也会清除 last_start
            self.last_start = None

    # 状态更新逻辑（独立于 rid 匹配）
    new_status = "down" if action == "fail" else "up"
    if self.status != new_status:
        reason = "fail" if action == "fail" else ""
        self.create_flip(new_status, reason=reason)
        self.status = new_status
```
[hc/api/models.py:499-517](hc/api/models.py#L499-L517)

### 10.3 关键发现

| 发现点 | 说明 |
|--------|------|
| **状态更新独立** | 每个 success/fail 事件都会根据自己的 action 更新状态，与 rid 匹配无关 |
| **fail 事件强制清除 running** | 无论 rid 是否匹配，fail 事件都会清除 `last_start` |
| **success 事件条件清除** | 只有 rid 匹配或 rid 为 None 时，success 事件才清除 `last_start` |
| **最终状态** | 以**最后一次事件**的 action 为准 |

### 10.4 场景详细分析

**假设初始状态**：
- `status = "up"`
- `last_start = null`
- `last_start_rid = null`
- 配置了 down 通知和 up 通知

---

**事件 1：收到 start ping（rid=A）**

```
时间点：10:00
action: "start"
rid: A
```

**处理逻辑**：
```python
if action == "start":
    self.last_start = frozen_now      # last_start = 10:00
    self.last_start_rid = rid         # last_start_rid = A
    # Don't update "last_ping" field.  # last_ping 不变
```

**结果**：
| 字段 | 变化 |
|------|------|
| `last_start` | null → 10:00 |
| `last_start_rid` | null → A |
| `status` | "up"（不变） |
| `last_ping` | 不变 |
| Flip 记录 | 无（start 不改变状态） |
| 通知 | 无 |

---

**事件 2：收到 fail ping（rid=A，同一个 rid）**

```
时间点：10:05
action: "fail"
rid: A
```

**处理逻辑**：
```python
else:  # action == "fail"
    self.last_ping = frozen_now           # last_ping = 10:05
    self.last_duration = None
    
    if self.last_start:                    # 是（last_start = 10:00）
        if self.last_start_rid == rid:     # A == A，是
            self.last_duration = 10:05 - 10:00 = 5min
            self.last_start = None          # 清除！
            self.last_start_rid = None      # 清除！
    
    new_status = "down"                     # 因为 action == "fail"
    if self.status != new_status:           # "up" != "down"，是
        reason = "fail"
        self.create_flip("down", reason="fail")  # 创建 Flip
        self.status = "down"                # 更新状态
```

**结果**：
| 字段 | 变化 |
|------|------|
| `last_ping` | 原值 → 10:05 |
| `last_duration` | null → 5分钟 |
| `last_start` | 10:00 → null（被清除） |
| `last_start_rid` | A → null（被清除） |
| `status` | "up" → "down" |
| Flip 记录 | 创建：up → down，reason="fail" |
| 通知 | 触发 down 通知（sendalerts 处理后发送） |

**关键点**：
- rid 匹配，所以计算了 `last_duration = 5min`
- `last_start` 被清除
- 状态从 "up" 变为 "down"
- 创建了 Flip 记录

---

**事件 3：收到 success ping（rid=A，同一个 rid）**

```
时间点：10:10
action: "success"
rid: A
```

**处理逻辑**：
```python
else:  # action == "success"
    self.last_ping = frozen_now           # last_ping = 10:10
    self.last_duration = None
    
    if self.last_start:                    # 否（已被事件 2 清除）
        # 不进入此分支
    
    new_status = "up"                       # 因为 action == "success"
    if self.status != new_status:           # "down" != "up"，是
        reason = ""                          # success 事件 reason 为空
        self.create_flip("up", reason="")   # 创建 Flip
        self.status = "up"                  # 更新状态
```

**结果**：
| 字段 | 变化 |
|------|------|
| `last_ping` | 10:05 → 10:10 |
| `last_duration` | 5min → null |
| `last_start` | null（不变） |
| `status` | "down" → "up" |
| Flip 记录 | 创建：down → up，reason="" |
| 通知 | 触发 up 通知（sendalerts 处理后发送） |

**关键点**：
- `last_start` 已经是 null，所以不处理 running 状态
- 状态从 "down" 变为 "up"
- 创建了新的 Flip 记录

---

### 10.5 最终状态汇总

| 项目 | 最终结果 |
|------|----------|
| **数据库 status** | `"up"`（以最后一次 success 事件为准） |
| **Flip 记录数量** | 2 条 |
| **Flip 记录 1** | up → down，reason="fail"，created=10:05 |
| **Flip 记录 2** | down → up，reason=""，created=10:10 |
| **通知发送** | 先发送 down 通知，再发送 up 通知 |
| **last_duration** | null（被 success 事件重置） |
| **last_start** | null（被 fail 事件清除） |

### 10.6 时序图

```
时间轴：

10:00 收到 start（rid=A）
  ├── last_start = 10:00
  ├── last_start_rid = A
  └── status 不变（仍为 "up"）

10:05 收到 fail（rid=A）
  ├── last_ping = 10:05
  ├── last_duration = 5min（rid 匹配）
  ├── last_start = null（被清除）
  ├── new_status = "down"
  ├── status: "up" → "down"
  ├── 创建 Flip: up→down, reason="fail"
  └── 触发 down 通知

10:10 收到 success（rid=A）
  ├── last_ping = 10:10
  ├── last_duration = null（重置）
  ├── last_start 已是 null，不处理
  ├── new_status = "up"
  ├── status: "down" → "up"
  ├── 创建 Flip: down→up, reason=""
  └── 触发 up 通知

最终：
  ├── status = "up"
  ├── Flip 记录：2 条
  └── 通知：down 通知 + up 通知
```

### 10.7 变体场景：rid 不匹配的情况

**场景**：
- 事件 1：start（rid=A）
- 事件 2：fail（rid=B，不匹配）
- 事件 3：success（rid=A）

**事件 2 分析（fail，rid=B 不匹配）**：
```python
if self.last_start:                    # 是
    if self.last_start_rid == rid:     # A == B？否
    elif action == "fail" or rid is None:  # action == "fail"，是
        self.last_start = None          # 仍然清除！

new_status = "down"
# ... 创建 Flip，状态变为 down
```

**关键点**：**fail 事件无论 rid 是否匹配，都会清除 `last_start`**

代码注释明确说明：
```python
# clear last_start (exit the "running" state) on:
# - "success" event with no rid
# - "fail" event, regardless of rid mismatch  ← 注意这行
```
[hc/api/models.py:507-510](hc/api/models.py#L507-L510)

**事件 3 分析（success，rid=A）**：
- `last_start` 已经被事件 2 清除（null）
- 所以不处理 running 状态
- `new_status = "up"`
- 状态从 "down" 变为 "up"

**最终结果**：与 rid 匹配的场景相同！

### 10.8 设计意图理解

1. **fail 事件的"终止"语义**：
   - fail 事件表示"这次执行失败了"
   - 无论 rid 是否匹配，都应该终止当前的 running 状态
   - 这是一种"安全"设计：只要收到失败信号，就认为当前执行已结束

2. **success 事件的"确认"语义**：
   - success 事件表示"这次执行成功了"
   - 只有 rid 匹配时，才认为是对应当前 running 状态的结束
   - 这是一种"精确"设计：确保 start 和 success 是同一次执行

3. **状态更新的"独立"设计**：
   - 每个 success/fail 事件都会独立改变状态
   - 这意味着：在短时间内收到多个事件时，状态会"闪烁"
   - 但 Flip 记录会完整记录每次状态变化
   - 通知也会每次状态变化都发送（如果配置了）

4. **为什么这样设计？**：
   - **可靠性优先**：宁可不厌其烦地发送通知，也不错过任何状态变化
   - **可追溯性**：Flip 记录完整记录每次状态变化，便于事后审计
   - **灵活性**：用户可以根据自己的需求选择使用或不使用 rid
