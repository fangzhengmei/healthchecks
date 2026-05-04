# 通知渠道分析

## 一、通知渠道配置方式

### 1. 核心数据模型

通知渠道的核心数据模型是 `Channel` 类，定义在 `hc/api/models.py` 中：

```python
class Channel(models.Model):
    name = models.CharField(max_length=100, blank=True)
    code = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)
    project = models.ForeignKey(Project, models.CASCADE)
    created = models.DateTimeField(default=now)
    kind = models.CharField(max_length=20, choices=CHANNEL_KINDS)
    value = models.TextField(blank=True)
    email_verified = models.BooleanField(default=False)
    disabled = models.BooleanField(default=False)
    last_notify = models.DateTimeField(null=True, blank=True)
    last_notify_duration = models.DurationField(null=True, blank=True)
    last_error = models.CharField(max_length=200, blank=True)
    checks = models.ManyToManyField(Check)
```
[hc/api/models.py:967-979](hc/api/models.py#L967-L979)

### 2. 支持的渠道类型

所有支持的渠道类型定义在 `TRANSPORTS` 字典中：

```python
TRANSPORTS: dict[str, tuple[str, type[transports.Transport] | str]] = {
    "apprise": ("Apprise", "hc.integrations.apprise.transport.Apprise"),
    "call": ("Phone Call", "hc.integrations.call.transport.Call"),
    "discord": ("Discord", "hc.integrations.discord.transport.Discord"),
    "email": ("Email", "hc.integrations.email.transport.Email"),
    "github": ("GitHub", "hc.integrations.github.transport.GitHub"),
    "googlechat": ("Google Chat", "hc.integrations.googlechat.transport.GoogleChat"),
    "gotify": ("Gotify", "hc.integrations.gotify.transport.Gotify"),
    "group": ("Group", "hc.integrations.group.transport.Group"),
    "matrix": ("Matrix", "hc.integrations.matrix.transport.Matrix"),
    "mattermost": ("Mattermost", "hc.integrations.mattermost.transport.Mattermost"),
    "msteamsw": ("Microsoft Teams", "hc.integrations.msteamsw.transport.MsTeamsWorkflow"),
    "ntfy": ("ntfy", "hc.integrations.ntfy.transport.Ntfy"),
    "opsgenie": ("Opsgenie", "hc.integrations.opsgenie.transport.Opsgenie"),
    "pagertree": ("PagerTree", "hc.integrations.pagertree.transport.PagerTree"),
    "pd": ("PagerDuty", "hc.integrations.pd.transport.PagerDuty"),
    "po": ("Pushover", "hc.integrations.po.transport.Pushover"),
    "pushbullet": ("Pushbullet", "hc.integrations.pushbullet.transport.Pushbullet"),
    "rocketchat": ("Rocket.Chat", "hc.integrations.rocketchat.transport.RocketChat"),
    "shell": ("Shell Command", "hc.integrations.shell.transport.Shell"),
    "signal": ("Signal", "hc.integrations.signal.transport.Signal"),
    "slack": ("Slack", "hc.integrations.slack.transport.Slack"),
    "sms": ("SMS", "hc.integrations.sms.transport.Sms"),
    "spike": ("Spike", "hc.integrations.spike.transport.Spike"),
    "telegram": ("Telegram", "hc.integrations.telegram.transport.Telegram"),
    "trello": ("Trello", "hc.integrations.trello.transport.Trello"),
    "victorops": ("Splunk On-Call", "hc.integrations.victorops.transport.VictorOps"),
    "webhook": ("Webhook", "hc.integrations.webhook.transport.Webhook"),
    "whatsapp": ("WhatsApp", "hc.integrations.whatsapp.transport.WhatsApp"),
    "zulip": ("Zulip", "hc.integrations.zulip.transport.Zulip"),
}
```
[hc/api/models.py:47-78](hc/api/models.py#L47-L78)

### 3. 渠道配置存储方式

不同类型的渠道配置存储在 `value` 字段中，存储格式因渠道类型而异：

#### 3.1 Webhook 渠道
Webhook 配置以 JSON 格式存储，包含分别用于 "up" 和 "down" 事件的配置：

```python
def webhook_spec(self, status: str) -> WebhookSpec:
    assert self.kind == "webhook"
    assert status in ("up", "down")

    doc = json.loads(self.value)
    return WebhookSpec(
        method=doc[f"method_{status}"],
        url=doc[f"url_{status}"],
        body=doc[f"body_{status}"],
        headers=doc[f"headers_{status}"],
    )
```
[hc/api/models.py:1146-1156](hc/api/models.py#L1146-L1156)

#### 3.2 Email 渠道
Email 渠道配置通过 `email` 属性访问，包含邮箱地址和通知设置：

```python
def is_noop(self, status: str) -> bool:
    if status == "down":
        return not self.channel.email.notify_down
    else:
        return not self.channel.email.notify_up
```
[hc/integrations/email/transport.py:69-73](hc/integrations/email/transport.py#L69-L73)

#### 3.3 SMS/Signal 渠道
SMS 和 Signal 渠道使用 `phone` 属性获取配置：

```python
def is_noop(self, status: str) -> bool:
    if status == "down":
        return not self.channel.phone.notify_down
    else:
        return not self.channel.phone.notify_up
```
[hc/integrations/sms/transport.py:36-40](hc/integrations/sms/transport.py#L36-L40)

#### 3.4 Slack 渠道
Slack 渠道配置存储 Webhook URL：

```python
@property
def slack_webhook_url(self) -> str:
    assert self.kind == "slack"
    ...
    return doc["url"]
```
[hc/integrations/slack/transport.py:89](hc/integrations/slack/transport.py#L89)

---

## 二、多渠道分发消息机制

### 1. 告警触发流程

告警分发的入口是 `sendalerts` 管理命令，主要流程如下：

1. **检测检查状态变化** (`handle_going_down`):
   - 查找 `alert_after` 已过期且状态不是 "down" 的检查
   - 计算当前状态，如果状态变为 "down"，创建 `Flip` 对象

2. **处理 Flip 对象** (`process_one_flip`):
   - 查找未处理的 `Flip` 对象
   - 标记为已处理
   - 提交到线程池执行通知发送

```python
def notify(flip: Flip) -> str | None:
    # 关闭旧的数据库连接，确保使用有效连接
    if not connection.in_atomic_block:
        close_old_connections()

    # 设置后续提醒日期
    check = flip.owner
    check.project.update_next_nag_dates()
    
    # 选择需要通知的渠道
    channels = flip.select_channels()
    if not channels:
        return None

    send_start = now()
    logs = [f"{check.code} goes {flip.new_status}"]
    
    # 遍历所有渠道，逐个发送通知
    for ch in channels:
        notify_start = time.time()
        error = ch.notify(flip)
        secs = time.time() - notify_start
        code8 = str(ch.code)[:8]
        if error:
            logs.append(f"  {code8} ({ch.kind}) Error in {secs:.1f}s: {error}")
        else:
            logs.append(f"  {code8} ({ch.kind}) OK in {secs:.1f}s")

    return "\n".join(logs)
```
[hc/api/management/commands/sendalerts.py:24-53](hc/api/management/commands/sendalerts.py#L24-L53)

### 2. 渠道选择机制

`Flip.select_channels()` 方法负责选择需要通知的渠道：

```python
def select_channels(self) -> list[Channel]:
    """Return a list of channels that need to be notified.

    * Exclude all channels for new->up and paused->up transitions.
    * Exclude disabled channels
    * Exclude channels where transport.is_noop(status) returns True
    * Sort channels by last_notify_duration (shorter durations first)
    """

    # 不发送 new->up 和 paused->up 状态转换的通知
    if self.new_status == "up" and self.old_status in ("new", "paused"):
        return []

    if self.new_status not in ("up", "down"):
        raise NotImplementedError(f"Unexpected status: {self.new_status}")

    # 排除已禁用的渠道
    q = self.owner.channel_set.exclude(disabled=True)
    
    # 按上次通知耗时排序（耗时短的优先）
    q = q.order_by(F("last_notify_duration").asc(nulls_last=True))
    
    # 排除 is_noop 为 True 的渠道
    result = []
    for channel in q:
        if not channel.transport.is_noop(self.new_status):
            result.append(channel)
    return result
```
[hc/api/models.py:1334-1351](hc/api/models.py#L1334-L1351)

### 3. 渠道分组机制

系统支持 `Group` 类型的渠道，可以将多个渠道组合成一个逻辑组：

```python
class Group(Transport):
    def notify(self, flip: Flip, notification: Notification) -> None:
        channels = self.channel.group_channels
        # 如果 notification 的 owner 字段为 None，则这是一个测试通知
        # 我们需要将 is_test=True 传递给 channel.notify() 调用
        is_test = notification.owner is None
        error_count = 0
        for channel in channels:
            error = channel.notify(flip, is_test=is_test)
            if error and error != "no-op":
                error_count += 1
        if error_count:
            raise TransportError(
                f"{error_count} out of {len(channels)} notifications failed"
            )
```
[hc/integrations/group/transport.py:7-21](hc/integrations/group/transport.py#L7-L21)

### 4. 并发分发机制

通知发送使用 `ThreadPoolExecutor` 实现并发处理：

```python
class Command(BaseCommand):
    def __init__(self, *args: Any, **kwargs: Any):
        super().__init__(*args, **kwargs)
        self.executor = ThreadPoolExecutor(max_workers=10)
        self.seats = BoundedSemaphore(10)  # 限制并发数
        self.shutdown = False

    def process_one_flip(self) -> bool:
        # 获取信号量，控制并发数
        if not self.seats.acquire(timeout=1):
            return False

        flip = Flip.objects.filter(processed=None).first()
        if flip is None:
            self.seats.release()
            return False

        # 标记 flip 为已处理
        q = Flip.objects.filter(id=flip.id, processed=None)
        num_updated = q.update(processed=now())
        if num_updated != 1:
            self.seats.release()
            return True  # 其他进程已处理

        # 提交到线程池执行
        f = self.executor.submit(notify, flip)
        f.add_done_callback(self.on_notify_done)
        return True
```
[hc/api/management/commands/sendalerts.py:56-119](hc/api/management/commands/sendalerts.py#L56-L119)

---

## 三、发送失败后的处理逻辑

### 1. 错误类型定义

系统定义了 `TransportError` 异常类，支持标记错误是否为永久性错误：

```python
class TransportError(Exception):
    def __init__(self, message: str, permanent: bool = False) -> None:
        self.message = message
        self.permanent = permanent
```
[hc/api/transports.py:41-44](hc/api/transports.py#L41-L44)

### 2. 通用重试机制

`HttpTransport` 类实现了通用的重试机制：

```python
@classmethod
def request(
    cls,
    method: str,
    url: str,
    *,
    retry: bool,
    params: curl.Params = None,
    data: curl.Data = None,
    json: Any = None,
    headers: curl.Headers = None,
    auth: curl.Auth = None,
) -> None:
    tries_left = 3 if retry else 1
    while True:
        try:
            return cls._request(
                method,
                url,
                params=params,
                data=data,
                json=json,
                headers=headers,
                auth=auth,
            )
        except TransportError as e:
            # 永久性错误不重试
            tries_left = 0 if e.permanent else tries_left - 1
            # 没有重试次数了，重新抛出异常
            if tries_left == 0:
                raise e
```
[hc/api/transports.py:152-183](hc/api/transports.py#L152-L183)

**重试规则**：
- 默认最多重试 3 次
- 永久性错误 (`permanent=True`) 不重试
- 测试通知 (`retry=False`) 不重试

### 3. 渠道特定的重试机制

#### 3.1 邮件重试机制

邮件发送有独立的重试逻辑：

```python
class EmailThread(Thread):
    MAX_TRIES = 3

    def run(self) -> None:
        for attempt in range(0, self.MAX_TRIES):
            try:
                # 确保每次重试都创建新连接
                self.message.connection = None
                self.message.send()
                # 没有异常，退出重试循环
                return
            except (SMTPServerDisconnected, SMTPDataError) as e:
                if attempt + 1 == self.MAX_TRIES:
                    # 这是最后一次尝试，失败后重新抛出异常
                    raise e

                # 等待 1 秒后重试
                time.sleep(1)
```
[hc/lib/emails.py:15-37](hc/lib/emails.py#L15-L37)

#### 3.2 Signal 重试机制

Signal 通知有独立的重试逻辑：

```python
def notify(self, flip: Flip, notification: Notification) -> None:
    if not settings.SIGNAL_CLI_SOCKET:
        raise TransportError("Signal notifications are not enabled")

    from hc.api.models import TokenBucket

    if not TokenBucket.authorize_signal(self.channel.phone.value):
        raise TransportError("Rate limit exceeded")

    ctx = {
        "flip": flip,
        "check": flip.owner,
        "status": flip.new_status,
        "ping": self.last_ping(flip),
        "down_checks": self.down_checks(flip.owner),
    }
    text = self.tmpl("signal_message.html", **ctx)
    
    # Signal 最多重试 2 次
    tries_left = 2
    while True:
        try:
            return self.send(self.channel.phone.value, text)
        except SignalRateLimitFailure as e:
            # 速率限制错误，发送提醒邮件
            self.channel.send_signal_captcha_alert(e.token, e.reply.decode())
            plaintext, _ = extract_signal_styles(text)
            self.channel.send_signal_rate_limited_notice(text, plaintext)
            raise e
        except TransportError as e:
            tries_left -= 1
            # 永久性错误或重试次数用完，则抛出异常
            if e.permanent or tries_left == 0:
                raise e
            logger.debug("Retrying signal-cli call")
```
[hc/integrations/signal/transport.py:158-188](hc/integrations/signal/transport.py#L158-L188)

### 4. 永久性错误处理

某些错误被标记为永久性错误，会导致渠道被自动禁用：

#### 4.1 Telegram 永久性错误

```python
@classmethod
def raise_for_response(cls, response: curl.Response) -> NoReturn:
    message = f"Received status code {response.status_code}"
    try:
        m = Telegram.ErrorModel.model_validate_json(response.content)
    except ValidationError:
        raise TransportError(message)

    if m.parameters:
        # 如果错误 payload 包含 migrate_to_chat_id 字段
        # 抛出 MigrationRequiredError，包含新的 chat_id
        chat_id = m.parameters.migrate_to_chat_id
        raise MigrationRequiredError(m.description, chat_id)

    permanent = False
    message += f' with a message: "{m.description}"'
    
    # 以下情况标记为永久性错误
    if m.description == "Forbidden: the group chat was deleted":
        permanent = True
    elif m.description == "Forbidden: bot was blocked by the user":
        permanent = True
    elif m.description == "Forbidden: user is deactivated":
        permanent = True

    raise TransportError(message, permanent=permanent)
```
[hc/integrations/telegram/transport.py:28-51](hc/integrations/telegram/transport.py#L28-L51)

#### 4.2 Slack 永久性错误

```python
@classmethod
def raise_for_response(cls, response: curl.Response) -> NoReturn:
    message = f"Received status code {response.status_code}"
    permanent = False
    
    # Slack 返回 404 表示此端点不太可能再次工作
    if response.status_code == 404:
        permanent = True
    elif response.status_code == 400:
        if response.content == b"invalid_token":
            # 使用已停用用户的令牌发送到私有频道
            # 理论上可以恢复，但实践中不太可能
            permanent = True
        else:
            logger.debug("Slack returned HTTP 400 with body: %s", response.content)

    raise TransportError(message, permanent=permanent)
```
[hc/integrations/slack/transport.py:93-112](hc/integrations/slack/transport.py#L93-L112)

#### 4.3 SMS 永久性错误

```python
@classmethod
def raise_for_response(cls, response: curl.Response) -> NoReturn:
    if response.status_code == 400:
        try:
            doc = Sms.ErrorModel.model_validate_json(response.content, strict=True)
            # 无效的电话号码是永久性错误
            if doc.code == 21211:
                raise TransportError("Invalid phone number", permanent=True)
        except ValidationError:
            pass

        logger.debug("Twilio Messages HTTP 400 with body: %s", response.content)

    raise TransportError(f"Received status code {response.status_code}")
```
[hc/integrations/sms/transport.py:23-34](hc/integrations/sms/transport.py#L23-L34)

### 5. 错误记录和渠道禁用

`Channel.notify()` 方法处理错误记录和渠道禁用：

```python
def notify(self, flip: Flip, is_test: bool = False) -> str:
    # 如果渠道应该忽略此状态，返回 no-op
    if self.transport.is_noop(flip.new_status):
        return "no-op"

    # 创建 Notification 记录
    n = Notification(channel=self)
    if is_test:
        # 测试通知时，owner 字段为 null
        pass
    else:
        n.owner = flip.owner

    n.check_status = flip.new_status
    n.error = "Sending"
    n.save()

    start, error, disabled = now(), "", self.disabled
    try:
        self.transport.notify(flip, notification=n)

    except transports.TransportError as e:
        # 永久性错误会禁用渠道
        disabled = True if e.permanent else disabled
        error = e.message

    # 更新 Notification 错误信息
    Notification.objects.filter(id=n.id).update(error=error)
    
    # 更新 Channel 状态
    Channel.objects.filter(id=self.id).update(
        last_notify=start,
        last_notify_duration=now() - start,
        last_error=error,
        disabled=disabled,
    )

    return error
```
[hc/api/models.py:1098-1130](hc/api/models.py#L1098-L1130)

### 6. 特殊错误处理

#### 6.1 速率限制处理

Telegram 和 Signal 都有速率限制处理：

```python
# Telegram 速率限制
def notify(self, flip: Flip, notification: Notification) -> None:
    from hc.api.models import TokenBucket

    if not TokenBucket.authorize_telegram(self.channel.telegram.id):
        raise TransportError("Rate limit exceeded")
```
[hc/integrations/telegram/transport.py:65-69](hc/integrations/telegram/transport.py#L65-L69)

#### 6.2 特殊迁移处理

Telegram 支持群组迁移后的自动更新：

```python
def notify(self, flip: Flip, notification: Notification) -> None:
    try:
        self.send(self.channel.telegram.id, self.channel.telegram.thread_id, text)
    except MigrationRequiredError as e:
        # 保存新的 chat_id，然后重新发送
        self.channel.update_telegram_id(e.new_chat_id)
        self.send(self.channel.telegram.id, self.channel.telegram.thread_id, text)
```
[hc/integrations/telegram/transport.py:84-89](hc/integrations/telegram/transport.py#L84-L89)

#### 6.3 配额超限处理

SMS 有月度配额限制：

```python
def notify(self, flip: Flip, notification: Notification) -> None:
    ...
    profile = self.channel.project.owner_profile
    if not profile.authorize_sms():
        # 月度 SMS 配额超限，发送提醒邮件
        self.channel.send_sms_limit_notice("SMS", text)
        raise TransportError("Monthly SMS limit exceeded")
```
[hc/integrations/sms/transport.py:54-57](hc/integrations/sms/transport.py#L54-L57)

---

## 四、总结

### 通知渠道配置
- 基于 `Channel` 数据模型，使用 `kind` 字段区分渠道类型
- 支持 30+ 种通知渠道类型，通过 `TRANSPORTS` 字典动态加载
- 配置存储在 `value` 字段，格式因渠道类型而异（JSON、特定格式等）

### 多渠道分发机制
- **顺序分发**：在 `sendalerts.py` 中按 `last_notify_duration` 排序后逐个发送
- **分组分发**：`Group` 渠道支持将多个子渠道组合为一个逻辑组
- **并发处理**：使用 `ThreadPoolExecutor` 处理多个检查的告警
- **智能选择**：`select_channels()` 方法根据状态、禁用状态、noop 标志过滤渠道

### 失败处理逻辑
- **通用重试**：`HttpTransport` 实现最多 3 次重试，永久性错误不重试
- **渠道特定重试**：Email（3次，间隔1秒）、Signal（2次）
- **永久性错误**：某些错误（如用户拉黑、群组删除、无效号码）会自动禁用渠道
- **错误记录**：错误信息记录在 `Notification.error` 和 `Channel.last_error`
- **特殊处理**：速率限制、配额超限、Telegram 群组迁移等场景有专门处理
