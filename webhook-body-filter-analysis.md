# Ping 请求 Body/Method/Header 过滤机制分析

## 概述

当 ping 不只是简单打点时，Healthchecks 系统支持基于 **HTTP Method**、**请求 Body 关键词** 等条件来判断 ping 请求应该被标记为成功、失败还是忽略。本文档详细分析这一机制。

---

## 一、请求信息读取

### 1.1 核心代码位置

文件：`hc/api/views.py`，函数：`ping`（第 180-243 行）

### 1.2 读取的请求信息

系统从 `HttpRequest` 对象中读取以下信息：

```python
headers = request.META                                    # 所有 HTTP 请求头
remote_addr = headers.get("HTTP_X_FORWARDED_FOR", headers["REMOTE_ADDR"])  # 客户端 IP
scheme = headers.get("HTTP_X_FORWARDED_PROTO", "http")   # 协议 (http/https)
method = headers["REQUEST_METHOD"]                        # HTTP 方法 (GET/POST 等)
ua = headers.get("HTTP_USER_AGENT", "")                   # User-Agent
body = request.body[: settings.PING_BODY_LIMIT]          # 请求 Body（有大小限制）
```

**关键要点**：
- `body` 会被截断到 `settings.PING_BODY_LIMIT` 指定的大小
- `remote_addr` 会处理 `X-Forwarded-For` 头，并处理 Azure 等环境的 `ip:port` 格式

---

## 二、过滤机制详解

### 2.1 过滤流程概览

```
请求到达
    ↓
检查 HTTP Method 限制
    ↓ （如果未被忽略）
检查 Body 关键词过滤
    ↓
确定最终 action (success/fail/ign/start)
    ↓
调用 check.ping() 处理
```

### 2.2 HTTP Method 过滤

**代码位置**：`hc/api/views.py:215-216`

```python
if check.methods == "POST" and method != "POST":
    action = "ign"
```

**规则说明**：
- 当 `check.methods` 设置为 `"POST"` 时，只有 POST 请求会被处理
- 非 POST 请求会被标记为 `"ign"`（忽略）
- `check.methods` 为空字符串时，表示不限制方法，所有方法都接受

**相关模型字段**（`hc/api/models.py:204`）：
```python
methods = models.CharField(max_length=30, blank=True)
```

可选值：
- `""`（空字符串）：不限制方法
- `"POST"`：仅接受 POST 请求

---

### 2.3 Body 内容过滤（关键词匹配）

**代码位置**：`hc/api/views.py:218-229`

```python
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

**执行顺序和优先级**：

| 优先级 | 条件 | 结果 action | 说明 |
|--------|------|-------------|------|
| 1 | `filter_http_body` 为 False | 不进入过滤 | 直接使用 URL 路径指定的 action |
| 2 | 匹配 `failure_kw`（失败关键词） | `"fail"` | 最高优先级，匹配即失败 |
| 3 | 匹配 `success_kw`（成功关键词） | `"success"` | 次高优先级 |
| 4 | 匹配 `start_kw`（开始关键词） | `"start"` | 用于标记任务开始 |
| 5 | `filter_default_fail` 为 True | `"fail"` | 无匹配时默认失败 |
| 6 | 其他情况 | `"ign"` | 无匹配且不默认失败，则忽略 |

---

### 2.4 关键词匹配函数

**代码位置**：`hc/lib/string.py:54-60`

```python
def match_keywords(haystack: str, keywords: str) -> bool:
    for s in keywords.split(","):
        s = s.strip()
        if s and s in haystack:
            return True

    return False
```

**匹配规则**：
1. 关键词按逗号 `,` 分割
2. 每个关键词去除首尾空白字符
3. 只要有 **任意一个** 关键词出现在文本中，即返回 `True`
4. 区分大小写（原文匹配）

**示例**：
- `keywords = "SUCCESS,OK,DONE"`：只要 Body 包含 `"SUCCESS"`、`"OK"` 或 `"DONE"` 中的任意一个，即匹配成功

---

## 三、相关配置字段

### 3.1 Check 模型字段

**文件**：`hc/api/models.py:197-204`

```python
filter_subject = models.BooleanField(default=False)      # 邮件主题过滤（不用于 HTTP ping）
filter_body = models.BooleanField(default=False)         # 邮件 Body 过滤（不用于 HTTP ping）
filter_http_body = models.BooleanField(default=False)    # HTTP Body 过滤开关 ← 关键
filter_default_fail = models.BooleanField(default=False) # 无匹配时是否默认失败 ← 关键
start_kw = models.CharField(max_length=200, blank=True)  # 开始关键词
success_kw = models.CharField(max_length=200, blank=True) # 成功关键词
failure_kw = models.CharField(max_length=200, blank=True) # 失败关键词
methods = models.CharField(max_length=30, blank=True)     # 允许的 HTTP 方法
```

### 3.2 字段说明

| 字段名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `filter_http_body` | Boolean | `False` | 是否启用 HTTP Body 过滤 |
| `filter_default_fail` | Boolean | `False` | 当过滤启用但无关键词匹配时，是否标记为失败（否则忽略） |
| `success_kw` | CharField(200) | `""` | 成功关键词，逗号分隔 |
| `failure_kw` | CharField(200) | `""` | 失败关键词，逗号分隔 |
| `start_kw` | CharField(200) | `""` | 开始关键词，逗号分隔（用于标记任务启动，配合 rid 计算耗时） |
| `methods` | CharField(30) | `""` | 允许的 HTTP 方法，空表示不限制，`"POST"` 表示仅接受 POST |

---

## 四、Action 类型及处理

### 4.1 Action 类型

| action | 含义 | 处理行为 |
|--------|------|----------|
| `"success"` | 成功 | 更新 `last_ping`，状态设为 `"up"` |
| `"fail"` | 失败 | 更新 `last_ping`，状态设为 `"down"`，触发告警 |
| `"start"` | 开始 | 更新 `last_start` 和 `last_start_rid`，不更新 `last_ping` |
| `"ign"` | 忽略 | 记录 ping 但不改变检查状态 |
| `"log"` | 日志 | 记录 ping 但不改变检查状态 |

### 4.2 Check.ping() 处理逻辑

**代码位置**：`hc/api/models.py:467-552`

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
    with transaction.atomic():
        self = Check.objects.select_for_update().get(id=self.id)
        frozen_now = now()

        # 暂停且手动恢复模式下，所有 ping 都忽略
        if self.status == "paused" and self.manual_resume:
            action = "ign"

        if action == "start":
            self.last_start = frozen_now
            self.last_start_rid = rid
        elif action == "ign":
            pass
        elif action == "log":
            pass
        else:
            self.last_ping = frozen_now
            self.last_duration = None
            # 计算耗时（如果有匹配的 rid）
            if self.last_start:
                if self.last_start_rid == rid:
                    self.last_duration = self.last_ping - self.last_start
                    self.last_start = None
                elif action == "fail" or rid is None:
                    self.last_start = None

            # 更新状态
            new_status = "down" if action == "fail" else "up"
            if self.status != new_status:
                reason = "fail" if action == "fail" else ""
                self.create_flip(new_status, reason=reason)
                self.status = new_status

        self.alert_after = self.going_down_after()
        self.n_pings = models.F("n_pings") + 1
        # 检查是否包含确认链接
        body_lowercase = body.decode(errors="replace").lower()
        self.has_confirmation_link = "confirm" in body_lowercase
        self.save()

        # 创建 Ping 记录
        ping = Ping(owner=self)
        ping.n = self.n_pings
        ping.created = frozen_now
        if action in ("start", "fail", "ign", "log"):
            ping.kind = action  # 默认为 None，表示 success

        ping.remote_addr = remote_addr
        ping.scheme = scheme
        ping.method = method
        ping.ua = ua[:200]
        # Body 存储：超过 100 字节且配置了 S3 则存 S3，否则存数据库
        if len(body) > 100 and settings.S3_BUCKET:
            ping.object_size = len(body)
        else:
            ping.body_raw = body
        ping.rid = rid
        ping.exitstatus = exitstatus
        ping.save()
```

**关键要点**：
1. `action == "start"` 时，只记录开始时间，不更新 `last_ping`
2. 当 `rid` 匹配时，可以计算 `last_duration`（任务耗时）
3. `ping.kind` 字段：
   - `None` 表示成功（success）
   - `"start"`、`"fail"`、`"ign"`、`"log"` 分别对应各自类型
4. Body 存储策略：超过 100 字节且配置 S3 则存 S3，否则存数据库

---

## 五、测试用例分析

### 5.1 测试文件位置

`hc/api/tests/test_ping.py:435-540`

### 5.2 典型测试场景

#### 场景 1：成功关键词匹配
```python
self.check.filter_http_body = True
self.check.success_kw = "SUCCESS"
self.check.save()

r = self.client.post(self.url, data="SUCCESS!", content_type="text/plain")
# 结果：ping.kind = None (success)
```

#### 场景 2：成功关键词不匹配（无默认失败）
```python
self.check.filter_http_body = True
self.check.success_kw = "SUCCESS"
self.check.save()

r = self.client.post(self.url, data="hello world", content_type="text/plain")
# 结果：ping.kind = "ign" (忽略)
```

#### 场景 3：失败关键词匹配
```python
self.check.filter_http_body = True
self.check.failure_kw = "FAIL"
self.check.save()

r = self.client.post(self.url, data="FAIL!", content_type="text/plain")
# 结果：ping.kind = "fail" (失败)
```

#### 场景 4：开始关键词匹配
```python
self.check.filter_http_body = True
self.check.start_kw = "START"
self.check.save()

r = self.client.post(self.url, data="STARTING", content_type="text/plain")
# 结果：ping.kind = "start" (开始)
```

#### 场景 5：无匹配 + 默认失败
```python
self.check.filter_http_body = True
self.check.success_kw = "SUCCESS"
self.check.filter_default_fail = True
self.check.save()

r = self.client.post(self.url, data="no keywords", content_type="text/plain")
# 结果：ping.kind = "fail" (失败)
```

#### 场景 6：Method 限制（优先级高于关键词过滤）
```python
self.check.filter_http_body = True
self.check.filter_default_fail = True  # 本应触发失败
self.check.methods = "POST"             # 但 Method 限制优先级更高
self.check.save()

r = self.client.get(self.url)  # GET 请求
# 结果：ping.kind = "ign" (忽略，因为不是 POST)
```

#### 场景 7：暂停 + 手动恢复模式（优先级最高）
```python
self.check.filter_http_body = True
self.check.success_kw = "SUCCESS"
self.check.manual_resume = True
self.check.status = "paused"
self.check.save()

r = self.client.post(self.url, data="SUCCESS!", content_type="text/plain")
# 结果：ping.kind = "ign" (忽略，因为暂停且手动恢复模式)
```

---

## 六、优先级总结

以下条件按优先级从高到低排列：

1. **暂停 + 手动恢复模式**（`status == "paused"` 且 `manual_resume == True`）
   - 所有 ping 强制标记为 `"ign"`
   - 代码位置：`hc/api/models.py:488-489`

2. **HTTP Method 限制**（`methods == "POST"`）
   - 非 POST 请求标记为 `"ign"`
   - 代码位置：`hc/api/views.py:215-216`

3. **URL 路径中的 exitstatus**
   - `/<code>/<exitstatus>` 格式中 `exitstatus > 0` 时，`action = "fail"`
   - 代码位置：`hc/api/views.py:212-213`

4. **Body 关键词过滤**（`filter_http_body == True`）
   - 按优先级：`failure_kw` > `success_kw` > `start_kw`
   - 无匹配时：`filter_default_fail` 决定是 `"fail"` 还是 `"ign"`
   - 代码位置：`hc/api/views.py:218-229`

5. **URL 路径中的 action**
   - `/<code>/fail` → `"fail"`
   - `/<code>/start` → `"start"`
   - 默认 → `"success"`

---

## 七、数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HTTP 请求到达                                    │
│  URL: /<code> 或 /<code>/<action> 或 /<code>/<exitstatus>            │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  1. 读取请求信息                                                        │
│     - method = request.META["REQUEST_METHOD"]                        │
│     - body = request.body[: PING_BODY_LIMIT]                         │
│     - headers, remote_addr, scheme, ua                                │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  2. 从 URL 路径确定初始 action                                          │
│     - /<code>/fail         → action = "fail"                         │
│     - /<code>/start        → action = "start"                        │
│     - /<code>/<exitstatus> → exitstatus > 0 时 action = "fail"      │
│     - 默认                    → action = "success"                    │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  3. HTTP Method 过滤                                                   │
│     if check.methods == "POST" and method != "POST":                 │
│         action = "ign"                                                 │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  4. Body 关键词过滤（当 action != "ign" 且 filter_http_body=True 时）   │
│     body_text = body.decode()                                          │
│                                                                         │
│     ┌─ 匹配 failure_kw? ── Yes ──→ action = "fail"                   │
│     │                                                                    │
│     └─ No ─┬─ 匹配 success_kw? ── Yes ──→ action = "success"        │
│            │                                                             │
│            └─ No ─┬─ 匹配 start_kw? ── Yes ──→ action = "start"     │
│                   │                                                      │
│                   └─ No ─┬─ filter_default_fail?                       │
│                          ├─ Yes ──→ action = "fail"                    │
│                          └─ No  ──→ action = "ign"                     │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  5. 调用 check.ping() 处理                                            │
│     - 检查暂停 + 手动恢复模式（可能再次改为 "ign"）                      │
│     - 根据 action 更新检查状态                                          │
│     - 创建 Ping 记录                                                   │
│     - 存储 Body（数据库或 S3）                                          │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        返回 HTTP 200 OK                                │
│  Header: Ping-Body-Limit（如果配置了限制）                              │
│  Header: Access-Control-Allow-Origin: *                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| ping 视图函数 | `hc/api/views.py` | 180-243 |
| HTTP Method 过滤 | `hc/api/views.py` | 215-216 |
| Body 关键词过滤 | `hc/api/views.py` | 218-229 |
| 关键词匹配函数 | `hc/lib/string.py` | 54-60 |
| Check.ping() 方法 | `hc/api/models.py` | 467-552 |
| Check 模型字段定义 | `hc/api/models.py` | 197-204 |
| 过滤机制测试用例 | `hc/api/tests/test_ping.py` | 435-540 |

---

## 九、使用示例

### 示例 1：监控脚本执行结果

**配置**：
- `filter_http_body = True`
- `success_kw = "SUCCESS,COMPLETE,DONE"`
- `failure_kw = "ERROR,FAILED,EXCEPTION"`
- `filter_default_fail = True`

**请求**：
```bash
# 成功场景
curl -X POST -d "Backup completed SUCCESSFULLY" http://hc.example.com/ping/<uuid>
# 结果：action = "success"

# 失败场景
curl -X POST -d "Backup FAILED with error code 500" http://hc.example.com/ping/<uuid>
# 结果：action = "fail"

# 无匹配但默认失败
curl -X POST -d "Some unknown output" http://hc.example.com/ping/<uuid>
# 结果：action = "fail"
```

### 示例 2：监控长任务执行

**配置**：
- `filter_http_body = True`
- `start_kw = "JOB_STARTED"`
- `success_kw = "JOB_FINISHED"`
- `failure_kw = "JOB_FAILED"`

**请求**：
```bash
# 任务开始
curl -X POST -d "JOB_STARTED" "http://hc.example.com/ping/<uuid>?rid=abc123"
# 结果：action = "start"，记录 last_start

# 任务完成（rid 匹配）
curl -X POST -d "JOB_FINISHED" "http://hc.example.com/ping/<uuid>?rid=abc123"
# 结果：action = "success"，计算 last_duration = last_ping - last_start
```

### 示例 3：仅接受 POST 请求

**配置**：
- `methods = "POST"`

**请求**：
```bash
# POST 请求（正常处理）
curl -X POST http://hc.example.com/ping/<uuid>
# 结果：正常处理

# GET 请求（被忽略）
curl http://hc.example.com/ping/<uuid>
# 结果：action = "ign"
```

---

## 十、注意事项

1. **关键词区分大小写**：`match_keywords` 函数使用原文匹配，`"Success"` 不会匹配 `"SUCCESS"`

2. **Body 大小限制**：`body` 会被截断到 `settings.PING_BODY_LIMIT`，过长的 Body 可能导致关键词匹配失败

3. **优先级问题**：
   - Method 限制优先级高于关键词过滤
   - 暂停 + 手动恢复模式优先级最高

4. **默认行为**：
   - 不启用 `filter_http_body` 时，所有 ping 都按 URL 路径的 action 处理
   - `filter_default_fail` 默认为 `False`，即无匹配时忽略而非失败

5. **Email 过滤 vs HTTP 过滤**：
   - `filter_subject` 和 `filter_body` 用于邮件检查（smtpd）
   - `filter_http_body` 用于 HTTP ping，两者是独立的
