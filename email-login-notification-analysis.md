# 账号邮箱登录、验证、团队邀请和告警通知分析

## 目录
1. [邮箱登录流程](#1-邮箱登录流程)
2. [邮箱验证流程](#2-邮箱验证流程)
3. [团队邀请邮件](#3-团队邀请邮件)
4. [告警通知邮件](#4-告警通知邮件)
5. [定期报告和持续提醒的调度循环](#5-定期报告和持续提醒的调度循环)
6. [邮件模板系统](#6-邮件模板系统)
7. [用户偏好对邮件发送的影响](#7-用户偏好对邮件发送的影响)

---

## 1. 邮箱登录流程

### 1.1 登录方式

系统支持两种邮箱登录方式：
- **密码登录**：使用邮箱+密码组合
- **免密登录（魔法链接）**：通过邮件发送登录链接

### 1.2 免密登录流程

#### 核心代码位置
- `hc/accounts/views.py:162-201` - login视图函数
- `hc/accounts/models.py:138-152` - send_instant_login_link方法
- `hc/lib/emails.py:85-86` - login邮件发送函数

#### 流程步骤

1. **用户提交邮箱**
   - 用户在登录页面输入邮箱地址
   - `EmailLoginForm` 验证邮箱格式和速率限制
   - 检查用户是否存在

2. **生成登录令牌**
   ```python
   # hc/accounts/models.py:122-128
   def prepare_token(self) -> str:
       token = token_urlsafe(24)
       # 存储登录令牌的哈希转换
       self.token = make_password(token, "login")
       self.save()
       # 签署令牌以便后续检查有效期
       return TimestampSigner().sign(token)
   ```
   - 生成24字节的URL安全令牌
   - 使用 `make_password` 哈希存储令牌
   - 使用 `TimestampSigner` 签署令牌以验证有效期（默认1小时）

3. **构建登录链接**
   ```python
   # hc/accounts/models.py:138-152
   def send_instant_login_link(
       self, membership: Member | None = None, redirect_url: str | None = None
   ) -> None:
       token = self.prepare_token()
       query = {"next": redirect_url} if redirect_url else None
       url = absolute_reverse(
           "hc-check-token", args=[self.user.username, token], query=query
       )
   ```
   - 链接格式：`/check_token/<username>/<token>`
   - 可选参数 `next` 用于登录后重定向

4. **发送登录邮件**
   ```python
   # hc/lib/emails.py:85-86
   def login(to: str, ctx: dict[str, Any]) -> None:
       send(make_message("login", to, ctx))
   ```
   - 使用 `login` 模板发送邮件
   - 支持团队邀请场景（传入 `membership` 参数）

5. **验证登录链接**
   ```python
   # hc/accounts/views.py:254-284
   def check_token(
       request: HttpRequest, username: str, token: str, new_email: str | None = None
   ) -> HttpResponse:
       # 自动登录防护机制：仅POST或带有auto-login cookie时自动登录
       if request.method != "POST" and "auto-login" not in request.COOKIES:
           return render(request, "accounts/check_token_submit.html")
       
       user = authenticate(username=username, token=token)
       if user is not None and user.is_active:
           # 处理邮箱变更（如果是邮箱变更链接）
           if new_email:
               if User.objects.filter(email=new_email).exists():
                   request.session["bad_link"] = True
                   return redirect("hc-login")
               user.email = new_email
               user.save()
           
           user.profile.token = ""
           user.profile.save()
           return _check_2fa(request, user)
   ```
   - **安全机制**：为防止邮件服务器自动扫描链接导致意外登录，要求：
     - 请求方法为 POST，或者
     - 携带 `auto-login` cookie（发送邮件时设置，有效期5分钟）
   - 否则显示确认表单，需要用户手动点击提交

6. **双因素认证检查**
   ```python
   # hc/accounts/views.py:125-145
   def _check_2fa(request: HttpRequest, user: User) -> HttpResponse:
       have_keys = user.credentials.exists()
       profile = Profile.objects.for_user(user)
       if have_keys or profile.totp:
           # 存储用户ID、邮箱和时间戳到session
           request.session["2fa_user"] = [user.id, user.email, int(time.time())]
           
           # 跳转到对应的2FA验证页面
           route = "hc-login-webauthn" if have_keys else "hc-login-totp"
           return redirect(reverse(route, query=query))
       
       auth_login(request, user)
       return _redirect_after_login(request)
   ```
   - 检查是否配置了 WebAuthn 安全密钥或 TOTP 二次验证
   - 2FA验证会话有效期：5分钟（300秒）

### 1.3 登录邮件模板

#### 邮件标题模板 (`templates/emails/login-subject.html`)
```django
{% load hc_extras %}
{% if membership %}
    You have been invited to join {{ membership.project|safe }} on {% site_name %}
{% else %}
    Log in to {% site_name %}
{% endif %}
```
- 如果是团队邀请，标题显示邀请信息
- 普通登录显示"Log in to {站点名称}"

#### 邮件内容模板 (`templates/emails/login-body-html.html`)
```django
{% extends "emails/base.html" %}
{% load hc_extras %}

{% block content %}
Hello,
<br />

{% if membership %}
    {% if membership.project.name %}
        <strong>{{ membership.project.owner.email }}</strong> invites you to their
        <a href="{% site_root %}">{% site_name %}</a>
        project <strong>{{ membership.project }}</strong>.
    {% else %}
        <strong>{{ membership.project.owner.email }}</strong> invites you to their
        <a href="{% site_root %}">{% site_name %}</a> account.
    {% endif %}
    <br /><br />

    {% if membership.role == "r" %}
    You will be able to view their existing monitoring checks, but not modify them.
    {% else %}
    You will be able to manage their existing monitoring checks and set up new ones.
    {% endif %}
    ...
{% endif %}

To log into <a href="{% site_root %}">{% site_name %}</a>,
please press the button below:
{% endblock %}
```

### 1.4 速率限制

登录过程中的速率限制通过 `TokenBucket` 模型实现：
- `TokenBucket.authorize_login_email(v)` - 限制同一邮箱的登录尝试次数
- `TokenBucket.authorize_auth_ip(request)` - 限制同一IP的认证请求次数

---

## 2. 邮箱验证流程

### 2.1 验证场景

邮箱验证主要用于以下场景：
1. **添加邮件通知渠道**：当用户添加非自身邮箱作为告警通知渠道时
2. **邮箱变更**：用户修改账号邮箱时（通过特殊的登录链接实现）

### 2.2 通知渠道邮箱验证

#### 核心代码位置
- `hc/integrations/email/views.py:17-64` - email_form视图
- `hc/api/models.py:1023-1026` - send_verify_link方法
- `hc/integrations/email/views.py:74-81` - verify视图

#### 验证规则

```python
# hc/integrations/email/views.py:25-34
if not settings.EMAIL_USE_VERIFICATION:
    # 自托管设置中，管理员可设置 EMAIL_USE_VERIFICATION=False 禁用邮箱验证
    channel.email_verified = True
elif form.cleaned_data["value"] == request.user.email:
    # 如果用户添加的是自己的邮箱，跳过验证步骤
    channel.email_verified = True
else:
    channel.email_verified = False
```

**验证规则**：
1. **全局禁用**：如果 `settings.EMAIL_USE_VERIFICATION = False`，所有邮箱都不需要验证
2. **自身邮箱**：如果添加的是用户自己的账号邮箱，自动验证通过
3. **第三方邮箱**：其他邮箱需要点击验证链接确认

#### 验证流程

1. **生成验证令牌**
   ```python
   # hc/api/models.py:1018-1021
   def make_token(self) -> str:
       seed = str(self.code) + settings.SECRET_KEY
       seed_bytes = seed.encode()
       return hashlib.sha256(seed_bytes).hexdigest()
   ```
   - 使用 Channel 的 code 和 SECRET_KEY 生成 SHA256 哈希作为令牌

2. **发送验证邮件**
   ```python
   # hc/api/models.py:1023-1026
   def send_verify_link(self) -> None:
       args = [self.code, self.make_token()]
       verify_link = absolute_reverse("hc-verify-email", args=args)
       emails.verify_email(self.email.value, {"verify_link": verify_link})
   ```

3. **验证链接处理**
   ```python
   # hc/integrations/email/views.py:74-81
   def verify(request: HttpRequest, code: UUID, token: str) -> HttpResponse:
       channel = get_object_or_404(Channel, code=code)
       if channel.make_token() == token:
           channel.email_verified = True
           channel.save()
           return render(request, "front/verify_email_success.html")
       
       return render(request, "bad_link.html")
   ```

### 2.3 邮箱变更验证

#### 核心代码位置
- `hc/accounts/views.py:603-628` - change_email视图
- `hc/accounts/models.py:154-167` - send_change_email_link方法
- `hc/accounts/views.py:631-637` - change_email_verify视图

#### 流程说明

1. **用户提交新邮箱**
   - 检查新邮箱是否已被注册
   - 生成特殊的登录链接

2. **发送变更链接到新邮箱**
   ```python
   # hc/accounts/models.py:154-167
   def send_change_email_link(self, new_email: str) -> None:
       payload = {
           "u": self.user.username,
           "t": self.prepare_token(),
           "e": new_email,
       }
       signed_payload = TimestampSigner().sign_object(payload)
       url = absolute_reverse("hc-change-email-verify", args=[signed_payload])
       
       ctx = {
           "button_text": "Log In",
           "button_url": url,
       }
       emails.login(new_email, ctx)
   ```
   - 载荷包含：用户名、登录令牌、新邮箱地址
   - 使用 `TimestampSigner` 签名整个载荷
   - 发送到新邮箱地址

3. **验证并变更邮箱**
   ```python
   # hc/accounts/views.py:631-637
   def change_email_verify(request: HttpRequest, signed_payload: str) -> HttpResponse:
       try:
           payload = TimestampSigner().unsign_object(signed_payload, max_age=900)
       except BadSignature:
           return render(request, "bad_link.html")
       
       return check_token(request, payload["u"], payload["t"], payload["e"])
   ```
   - 签名有效期：15分钟（900秒）
   - 调用 `check_token` 时传入新邮箱，登录成功后自动更新邮箱

### 2.4 验证邮件模板

#### 邮件标题 (`templates/emails/verify-email-subject.html`)
```django
{% load hc_extras %}
Verify email address on {% site_name %}
```

#### 邮件内容 (`templates/emails/verify-email-body-html.html`)
```django
{% load hc_extras %}
<p>Hello,</p>

<p>To start receiving {% site_name %} notification to this address,
please click the link below:</p>
<p><a href="{{ verify_link }}">{{ verify_link }}</a></p>

<p>
    --<br />
    Regards,<br />
    {% site_name %}
</p>
```

---

## 3. 团队邀请邮件

### 3.1 邀请流程

#### 核心代码位置
- `hc/accounts/views.py:413-437` - project视图中的邀请处理
- `hc/accounts/models.py:462-475` - Project.invite方法
- `hc/accounts/models.py:138-152` - Profile.send_instant_login_link方法

#### 流程步骤

1. **提交邀请表单**
   ```python
   # hc/accounts/views.py:413-437
   if "invite_team_member" in request.POST:
       invite_form = forms.InviteTeamMemberForm(request.POST)
       if invite_form.is_valid():
           email = invite_form.cleaned_data["email"]
           
           # 检查邀请建议列表（避免重复邀请）
           invite_suggestions = project.invite_suggestions()
           if not invite_suggestions.filter(email=email).exists():
               # 检查速率限制
               if not TokenBucket.authorize_invite(request.user):
                   return render(request, "try_later.html")
   ```

2. **查找或创建用户**
   ```python
   try:
       user = User.objects.get(email=email)
   except User.DoesNotExist:
       # 用户不存在时创建新用户（不创建默认项目）
       user = _make_user(email, with_project=False)
   ```

3. **创建成员关系并发送邀请**
   ```python
   # hc/accounts/models.py:462-475
   def invite(self, user: User, role: str) -> bool:
       # 检查是否已是成员或项目所有者
       if Member.objects.filter(user=user, project=self).exists():
           return False
       if self.owner_id == user.id:
           return False
       
       # 创建成员关系
       m = Member.objects.create(user=user, project=self, role=role)
       checks_url = reverse("hc-checks", args=[self.code])
       
       # 发送邀请邮件
       if settings.EMAIL_HOST:
           profile = Profile.objects.for_user(user)
           profile.send_instant_login_link(membership=m, redirect_url=checks_url)
       return True
   ```

4. **发送带邀请信息的登录链接**
   - 调用 `send_instant_login_link` 时传入 `membership` 参数
   - 登录链接会在登录后重定向到项目的检查列表页面

### 3.2 成员角色

```python
# hc/accounts/models.py:589-598
class Member(models.Model):
    class Role(models.TextChoices):
        READONLY = "r", "Read-only"
        REGULAR = "w", "Member"
        MANAGER = "m", "Manager"
```

**角色权限**：
- **Read-only (r)**：只能查看监控检查，不能修改
- **Member (w)**：可以管理现有检查和创建新检查
- **Manager (m)**：拥有管理权限，可以邀请/移除团队成员

### 3.3 邀请邮件模板复用

团队邀请邮件复用了登录邮件模板，但通过 `membership` 参数区分显示内容：

**模板中的条件逻辑**：
```django
{% if membership %}
    <!-- 显示邀请信息 -->
    <strong>{{ membership.project.owner.email }}</strong> invites you to their
    project <strong>{{ membership.project }}</strong>.
    
    {% if membership.role == "r" %}
    You will be able to view their existing monitoring checks, but not modify them.
    {% else %}
    You will be able to manage their existing monitoring checks and set up new ones.
    {% endif %}
{% else %}
    <!-- 普通登录提示 -->
    To log into <a href="{% site_root %}">{% site_name %}</a>,
    please press the button below:
{% endif %}
```

---

## 4. 告警通知邮件

### 4.1 告警触发机制

#### 核心代码位置
- `hc/api/models.py:630-650` - Check.create_flip方法
- `hc/api/models.py:1098-1130` - Channel.notify方法
- `hc/integrations/email/transport.py:12-67` - Email.notify方法

#### 状态变更（Flip）

当检查状态发生变化时（up → down 或 down → up），系统会创建一个 `Flip` 对象：

```python
# hc/api/models.py:630-650
def create_flip(
    self, new_status: str, reason: str = "", mark_as_processed: bool = False
) -> None:
    """
    Flip对象记录检查状态变化，有两个用途：
    1. 异步发送通知（在web进程中创建flip对象，单独的sendalerts进程处理）
    2. 用于停机时间统计计算
    """
    flip = Flip(owner=self)
    flip.created = now()
    if mark_as_processed:
        flip.processed = flip.created
    flip.old_status = self.status
    flip.new_status = new_status
    flip.reason = reason
    flip.save()
```

#### 通知发送流程

1. **创建通知记录**
   ```python
   # hc/api/models.py:1098-1130
   def notify(self, flip: Flip, is_test: bool = False) -> str:
       # 检查是否需要跳过（基于用户偏好设置）
       if self.transport.is_noop(flip.new_status):
           return "no-op"
       
       # 创建通知记录
       n = Notification(channel=self)
       if not is_test:
           n.owner = flip.owner
       n.check_status = flip.new_status
       n.error = "Sending"
       n.save()
   ```

2. **发送邮件通知**
   ```python
   # hc/integrations/email/transport.py:12-67
   class Email(Transport):
       def notify(self, flip: Flip, notification: Notification) -> None:
           # 检查邮箱是否已验证
           if not self.channel.email_verified:
               raise TransportError("Email not verified")
           
           # 构建退订链接
           unsub_link = self.channel.get_unsub_link()
           
           # 构建邮件头
           headers = {
               "List-Unsubscribe": f"<{unsub_link}>",
               "List-Unsubscribe-Post": "List-Unsubscribe=One-Click",
               "X-Bounce-ID": sign_bounce_id(f"n.{notification.code}"),
           }
   ```

3. **时区和用户上下文处理**
   ```python
   # hc/integrations/email/transport.py:25-35
   # 如果该邮箱地址有关联的账号：
   # - 包含该账号有权访问的项目摘要
   # - 使用他们的首选时区格式化日期时间
   # 否则，使用渠道所有者的首选时区
   try:
       profile = Profile.objects.get(user__email=self.channel.email.value)
       projects = list(profile.projects())
   except Profile.DoesNotExist:
       profile = Profile.objects.for_user(self.channel.project.owner)
       projects = None
   ```

4. **发送告警邮件**
   ```python
   ctx = {
       "flip": flip,
       "check": flip.owner,
       "ping": ping,
       "body": body,
       "subject": subject,
       "projects": projects,
       "unsub_link": unsub_link,
       "tz": profile.tz,
       "ping_attached": attachment is not None,
   }
   
   emails.alert(self.channel.email.value, ctx, headers, attachment)
   ```

### 4.2 告警邮件模板

#### 邮件标题 (`templates/emails/alert-subject.html`)
```django
{{ flip.new_status|upper }} | {{ check.name_then_code|safe }}
```
- 例如：`DOWN | My First Check` 或 `UP | My First Check`

#### 邮件内容 (`templates/emails/alert-body-html.html`)

**状态提示**：
```django
<p>
    {% if flip.new_status == "down" %}
    "{{ check.name_then_code }}" is DOWN{% if flip.reason %} ({{ flip.reason_long }}){% endif %}.
    {% else %}
    "{{ check.name_then_code }}" is UP.
    {% endif %}
    <a href="{{ check.cloaked_url }}">View on {% site_name %}&hellip;</a>
</p>
```

**检查详情表格**：
- 项目名称（如果有）
- 标签
- 周期（simple类型）或 调度计划+时区（cron/oncalendar类型）
- 总ping次数
- 最后一次ping信息
- 状态变更时间

**项目概览**（如果收件人有账号）：
```django
{% if projects %}
<p><b>Projects Overview</b></p>
<table>
    {% for project in projects %}
    <tr>
        <td><a href="{{ project.checks_url }}">{{ project }}</a></td>
        <td>
            {% with project.get_n_down as n_down %}
                {% if n_down %}
                <b>{{ n_down }} check{{ n_down|pluralize }} down</b>
                {% else %}
                OK, all checks up
                {% endif %}
            {% endwith %}
        </td>
    </tr>
    {% endfor %}
</table>
{% endif %}
```

### 4.3 告警邮件特殊功能

#### 1. 邮件附件
- 如果最后一次ping是通过邮件发送的（`ping.scheme == "email"`），将原始邮件作为附件
- 处理多部分邮件时，移除 `message/rfc822` 类型的部分以避免递归问题

#### 2. 一键退订
```python
headers = {
    "List-Unsubscribe": f"<{unsub_link}>",
    "List-Unsubscribe-Post": "List-Unsubscribe=One-Click",
}
```
- 支持 RFC 8058 一键退订标准
- 邮件客户端可显示退订按钮

#### 3. 退订链接
```python
# hc/api/models.py:1028-1032
def get_unsub_link(self) -> str:
    signer = TimestampSigner(salt="alerts")
    signed_token = signer.sign(self.make_token())
    args = [self.code, signed_token]
    return absolute_reverse("hc-unsubscribe-alerts", args=args)
```

#### 4. 退订处理
```python
# hc/integrations/email/views.py:84-113
def unsubscribe(request: HttpRequest, code: UUID, signed_token: str) -> HttpResponse:
    # 检查签名（先不验证时间戳）
    try:
        token = signer.unsign(signed_token)
    except signing.BadSignature:
        return render(request, "bad_link.html")
    
    # 检查时间戳是否超过5分钟
    try:
        signer.unsign(signed_token, max_age=300)
    except signing.SignatureExpired:
        ctx["autosubmit"] = True  # 自动提交表单
    
    # POST请求时禁用渠道
    if request.method == "POST":
        Channel.objects.filter(id=channel.id).update(disabled=True)
        return render(request, "front/unsubscribe_success.html")
```

### 4.4 后台发送Worker中的过滤机制

#### 核心代码位置
- `hc/api/management/commands/sendalerts.py` - 后台告警发送Worker
- `hc/api/models.py:1334-1352` - Flip.select_channels方法
- `hc/api/models.py:1098-1100` - Channel.notify中的is_noop检查
- `hc/integrations/email/transport.py:69-73` - Email.is_noop具体实现

#### 发送Worker主循环

```python
# hc/api/management/commands/sendalerts.py:182-213
def handle(self, num_workers: int, pool: bool, **options: Any) -> str:
    # ...
    while not self.shutdown:
        # 为超时的检查创建Flips（状态变更事件）
        while self.handle_going_down() and not self.shutdown:
            pass

        # 提交未处理的Flips到线程池执行
        while self.process_one_flip() and not self.shutdown:
            pass

        # 所有worker忙或没有未处理的Flips，等待2秒
        if not self.shutdown:
            time.sleep(2)
```

#### 处理单个状态变更（Flip）

```python
# hc/api/management/commands/sendalerts.py:89-119
def process_one_flip(self) -> bool:
    """查找未处理的flip，发送通知
    
    返回True表示主循环应立即继续
    返回False表示主循环应等待一段时间再继续
    """

    if not self.seats.acquire(timeout=1):
        return False  # Worker忙，主线程应等待

    # 查找第一个未处理的flip
    flip = Flip.objects.filter(processed=None).first()
    if flip is None:
        self.seats.release()
        return False  # 没有工作，主线程应等待

    # 乐观锁：标记flip为已处理
    q = Flip.objects.filter(id=flip.id, processed=None)
    num_updated = q.update(processed=now())
    if num_updated != 1:
        self.seats.release()
        # 没有更新成功：其他sendalerts进程已抢先处理
        return True

    # 提交到线程池执行
    f = self.executor.submit(notify, flip)
    f.add_done_callback(self.on_notify_done)
    return True
```

#### 通知执行函数

```python
# hc/api/management/commands/sendalerts.py:24-53
def notify(flip: Flip) -> str | None:
    # 线程池中的线程可能有已打开的db连接，如果最近未运行
    # db连接可能已超时，调用close_old_connections()确保有可用连接
    if not connection.in_atomic_block:
        close_old_connections()

    # 设置或清除后续持续提醒的日期
    check = flip.owner
    check.project.update_next_nag_dates()
    
    # 【关键】筛选需要通知的渠道
    channels = flip.select_channels()
    if not channels:
        return None

    # 遍历渠道发送通知
    send_start = now()
    logs = [f"{check.code} goes {flip.new_status}"]
    for ch in channels:
        notify_start = time.time()
        error = ch.notify(flip)  # 这里会再次检查is_noop
        # ...记录日志
```

#### Flip.select_channels 渠道筛选逻辑

```python
# hc/api/models.py:1334-1352
def select_channels(self) -> list[Channel]:
    """返回需要通知的渠道列表
    
    筛选规则：
    * 排除 new->up 和 paused->up 转换
    * 排除禁用的渠道
    * 排除 transport.is_noop(status) 返回True的渠道
    * 按last_notify_duration排序（耗时短的优先）
    """

    # 第一层过滤：状态转换过滤
    # 不发送 new->up 和 paused->up 的通知
    # （新创建的检查第一次成功、暂停的检查恢复成功，不通知）
    if self.new_status == "up" and self.old_status in ("new", "paused"):
        return []

    if self.new_status not in ("up", "down"):
        raise NotImplementedError(f"Unexpected status: {self.new_status}")

    # 第二层过滤：排除禁用的渠道
    q = self.owner.channel_set.exclude(disabled=True)
    
    # 按上次通知耗时排序（快的优先，避免慢渠道阻塞其他通知）
    q = q.order_by(F("last_notify_duration").asc(nulls_last=True))
    
    # 第三层过滤：基于状态偏好的过滤（is_noop检查）
    return [ch for ch in q if not ch.transport.is_noop(self.new_status)]
```

#### 禁用渠道过滤详解

在 `select_channels` 中：
```python
# 排除 disabled=True 的渠道
q = self.owner.channel_set.exclude(disabled=True)
```

**禁用渠道的场景**：
1. **用户主动禁用**：用户在集成页面禁用某个通知渠道
2. **退订导致禁用**：用户点击告警邮件中的退订链接
   ```python
   # hc/integrations/email/views.py:113
   Channel.objects.filter(id=channel.id).update(disabled=True)
   ```
3. **发送错误导致禁用**：永久错误（如邮箱不存在）会导致渠道被禁用
   ```python
   # hc/api/models.py:1118-1119
   except transports.TransportError as e:
       disabled = True if e.permanent else disabled
   ```

#### 基于状态的通知过滤（is_noop）

在 `select_channels` 的最后一步：
```python
return [ch for ch in q if not ch.transport.is_noop(self.new_status)]
```

**基类定义**：
```python
# hc/api/transports.py:61-70
class Transport:
    def is_noop(self, status: str) -> bool:
        """如果transport会忽略当前状态，返回True
        
        在Webhook子类中被重写，用户可以配置up和down事件的webhook url，且都是可选的
        """
        return False
```

**Email渠道的具体实现**：
```python
# hc/integrations/email/transport.py:69-73
def is_noop(self, status: str) -> bool:
    if status == "down":
        # 如果是down事件，且用户关闭了down通知，则跳过
        return not self.channel.email.notify_down
    else:
        # 如果是up事件，且用户关闭了up通知，则跳过
        return not self.channel.email.notify_up
```

**配置存储**：
```python
# hc/api/models.py:895-906
class EmailConf(BaseModel):
    value: str
    notify_up: bool = Field(alias="up")
    notify_down: bool = Field(alias="down")
```

**配置解析**：
- 如果 `value` 是纯邮箱字符串（非JSON），默认 `up=True, down=True`
- 如果 `value` 是JSON格式，解析 `up` 和 `down` 字段

#### 双层过滤机制总结

告警发送路径中有**两层过滤**：

| 过滤层级 | 位置 | 过滤条件 |
|---------|------|---------|
| 第一层 | `Flip.select_channels()` | 状态转换过滤 + 禁用渠道过滤 + is_noop预过滤 |
| 第二层 | `Channel.notify()` | 再次检查 `is_noop()` |

**为什么需要两层检查？**
- `select_channels` 中的检查用于快速筛选，减少不必要的通知记录创建
- `Channel.notify` 中的检查是防御性编程，确保即使在测试或其他场景下也不会发送错误的通知

```python
# hc/api/models.py:1098-1100
def notify(self, flip: Flip, is_test: bool = False) -> str:
    # 再次检查是否需要跳过
    if self.transport.is_noop(flip.new_status):
        return "no-op"
    # ...
```

---

## 5. 定期报告和持续提醒的调度循环

### 5.1 调度器核心代码位置
- `hc/api/management/commands/sendreports.py` - 报告和持续提醒调度器
- `hc/accounts/models.py:342-349` - update_next_nag_date方法
- `hc/accounts/models.py:351-376` - choose_next_report_date方法

### 5.2 调度器主循环

```python
# hc/api/management/commands/sendreports.py:100-132
def handle(self, loop: bool, **options: Any) -> str:
    self.shutdown = False
    signal.signal(signal.SIGTERM, self.on_signal)
    signal.signal(signal.SIGINT, self.on_signal)

    self.stdout.write("sendreports is now running")
    while not self.shutdown:
        # db连接可能已超时，确保有可用连接
        if not connection.in_atomic_block:
            close_old_connections()

        # 第一阶段：处理到期的定期报告
        while not self.shutdown and self.handle_one_report():
            pass

        # 第二阶段：处理到期的持续提醒（Nags）
        while not self.shutdown and self.handle_one_nag():
            pass

        if not loop:
            break  # 非循环模式，执行一次退出

        # 循环模式：睡眠60秒后再次检查
        for i in range(0, 60):
            if not self.shutdown:
                time.sleep(1)

    return "Done."
```

### 5.3 定期报告调度处理

```python
# hc/api/management/commands/sendreports.py:34-67
def handle_one_report(self) -> bool:
    # 查询条件1：next_report_date < 当前时间（已到期）
    report_due = Q(next_report_date__lt=now())
    # 查询条件2：next_report_date 为 null（从未调度过）
    report_not_scheduled = Q(next_report_date__isnull=True)

    # 筛选：到期或未调度，且报告未关闭
    q = Profile.objects.filter(report_due | report_not_scheduled)
    q = q.exclude(reports="off")
    profile = q.first()

    if profile is None:
        # 没有匹配的Profile，当前无事可做
        return False

    # 【乐观锁】：使用当前next_report_date值作为条件
    # 确保在并发场景下不会重复发送
    qq = Profile.objects.filter(
        id=profile.id, next_report_date=profile.next_report_date
    )

    # 场景1：从未调度过 → 先调度，不发送
    if profile.next_report_date is None:
        qq.update(next_report_date=profile.choose_next_report_date())
        return True

    # 场景2：已到期 → 先调度下次，再发送
    # 先更新下次发送时间（乐观锁避免重复发送）
    num_updated = qq.update(next_report_date=profile.choose_next_report_date())
    if num_updated != 1:
        # next_report_date已被其他进程更新，跳过
        return True

    # 发送报告
    if profile.send_report():
        self.stdout.write(self.tmpl % profile.user.email)
        # 发送后暂停3秒，避免触发邮件服务配额限制
        self.pause()

    return True
```

### 5.4 持续提醒（Nag）调度处理

```python
# hc/api/management/commands/sendreports.py:69-93
def handle_one_nag(self) -> bool:
    now_value = now()
    # 查询条件：next_nag_date < 当前时间，且nag_period不是"禁用"
    q = Profile.objects.filter(next_nag_date__lt=now_value)
    q = q.exclude(nag_period=NO_NAG)
    profile = q.first()

    if profile is None:
        return False

    # 乐观锁
    qq = Profile.objects.filter(id=profile.id, next_nag_date=profile.next_nag_date)

    # 【先更新时间，后发送】
    # 与报告不同，nag是持续提醒，先更新下次时间避免重复发送
    num_updated = qq.update(next_nag_date=now_value + profile.nag_period)
    if num_updated != 1:
        # 已被其他进程更新，跳过
        return True

    # 发送持续提醒（nag=True表示只包含当前down的检查）
    if profile.send_report(nag=True):
        self.stdout.write(f"Sent nag to {profile.user.email}")
        self.pause()
    else:
        # 发送失败：可能是没有down的检查了
        # 清空next_nag_date，直到有检查down时再重新调度
        profile.next_nag_date = None
        profile.save()

    return True
```

### 5.5 持续提醒的触发条件更新

当检查状态变化时，会更新项目所有成员的 `next_nag_date`：

```python
# hc/api/management/commands/sendalerts.py:32-34
def notify(flip: Flip) -> str | None:
    # ...
    check = flip.owner
    check.project.update_next_nag_dates()  # 【关键】
    # ...
```

**update_next_nag_dates 实现**：
```python
# hc/accounts/models.py:477-487
def update_next_nag_dates(self) -> None:
    """更新项目所有成员的next_nag_date"""

    # 筛选条件：项目所有者 + 项目成员，且nag_period不是禁用
    is_owner = Q(user_id=self.owner_id)
    is_member = Q(user__memberships__project=self)
    q = Profile.objects.filter(is_owner | is_member).exclude(nag_period=NO_NAG)

    # 遍历每个符合条件的Profile，更新其nag调度
    for profile in q:
        profile.update_next_nag_date()
```

**update_next_nag_date 实现**：
```python
# hc/accounts/models.py:342-349
def update_next_nag_date(self) -> None:
    # 检查用户有权访问的所有项目中是否有down的检查
    any_down = self.checks_from_all_projects().filter(status="down").exists()

    # 条件1：有检查down + 未设置下次nag日期 + 已启用nag
    if any_down and self.next_nag_date is None and self.nag_period:
        self.next_nag_date = now() + self.nag_period
        self.save(update_fields=["next_nag_date"])
    
    # 条件2：没有检查down + 已设置下次nag日期
    elif not any_down and self.next_nag_date:
        self.next_nag_date = None
        self.save(update_fields=["next_nag_date"])
```

### 5.6 下次发送时间计算

#### 定期报告下次时间计算

```python
# hc/accounts/models.py:351-376
def choose_next_report_date(self) -> datetime | None:
    """计算下一次月度/周度报告的目标日期
    
    发送时间规则：
    - 月度报告：每月1日，用户时区的 9AM-11AM 之间
    - 周度报告：每周一，用户时区的 9AM-11AM 之间
    - 日报：每天，用户时区的 9AM-11AM 之间
    """
    if self.reports == "off":
        return None

    # 获取当前时间在用户时区的表示
    dt = now().astimezone(ZoneInfo(self.tz))
    
    # 在 9:00-11:00 之间随机选择分钟
    # 避免所有用户在同一时间收到邮件，分散负载
    dt = dt.replace(hour=9, minute=0) + td(minutes=random.randrange(0, 120))

    # 找到未来的第一个符合条件的日期
    while True:
        dt += td(days=1)
        if self.reports == "daily":
            return dt  # 日报：明天
        if self.reports == "monthly" and dt.day == 1:
            return dt  # 月报：下个月1日
        elif self.reports == "weekly" and dt.weekday() == 0:
            return dt  # 周报：下周一
```

#### 持续提醒下次时间计算

持续提醒的下次时间有两种计算方式：

**方式1：状态变化时触发**（在 `update_next_nag_date` 中）
```python
# 当有检查down时，设置为：当前时间 + nag_period
self.next_nag_date = now() + self.nag_period
```

**方式2：发送后调度**（在 `handle_one_nag` 中）
```python
# 发送nag后，更新为：当前时间 + nag_period
qq.update(next_nag_date=now_value + profile.nag_period)
```

### 5.7 乐观锁机制详解

#### 为什么需要乐观锁？

在分布式部署场景下，可能有多个 `sendreports` 进程同时运行。如果没有锁机制，可能会导致：
- 同一报告被重复发送多次
- 同一持续提醒被重复发送

#### 乐观锁实现模式

```python
# 步骤1：读取记录
profile = q.first()

# 步骤2：使用读取时的字段值作为更新条件
qq = Profile.objects.filter(
    id=profile.id, 
    next_report_date=profile.next_report_date  # 【关键】乐观锁条件
)

# 步骤3：执行更新
num_updated = qq.update(next_report_date=...)

# 步骤4：检查更新行数
if num_updated != 1:
    # 其他进程已更新，跳过
    return True
```

#### 报告与Nag的调度顺序对比

| 类型 | 调度顺序 | 发送成功后 | 发送失败/无数据时 |
|-----|---------|-----------|------------------|
| 定期报告 | **先更新 `next_report_date`，后发送** | 无特殊处理（下次时间已设置） | 无特殊处理（下次时间已设置） |
| 持续提醒 | **先更新 `next_nag_date`，后发送** | 无特殊处理（下次时间已设置） | **清空 `next_nag_date`**（没有 down 的检查时） |

**两者的调度顺序是相同的**：都是先更新时间（乐观锁避免重复发送），后发送邮件。

**核心区别**：
- 定期报告：发送失败不做特殊处理，下次调度时间已在数据库中设置
- 持续提醒：如果 `send_report(nag=True)` 返回 `False`（没有 down 的检查），会主动清空 `next_nag_date`，直到有检查 down 时才会重新调度

### 5.8 调度循环数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                    sendreports 主循环 (每60秒)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           第一阶段：处理定期报告                            │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │                                                           │  │
│  │  SELECT * FROM profile                                    │  │
│  │  WHERE (next_report_date < NOW()                         │  │
│  │         OR next_report_date IS NULL)                     │  │
│  │    AND reports != 'off'                                   │  │
│  │  ORDER BY id LIMIT 1                                      │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │ │ 场景1: next_report_date IS NULL                       │ │  │
│  │ │   → 只调度，不发送                                    │ │  │
│  │ │   UPDATE profile SET next_report_date = ?            │ │  │
│  │ │   WHERE id = ? AND next_report_date IS NULL          │ │  │
│  │ └─────────────────────────────────────────────────────┘ │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │ │ 场景2: next_report_date < NOW()                      │ │  │
│  │ │   → 先更新下次时间，再发送                            │ │  │
│  │ │   UPDATE profile SET next_report_date = ?            │ │  │
│  │ │   WHERE id = ? AND next_report_date = ?              │ │  │
│  │ │   IF 1 row updated:                                   │ │  │
│  │ │      profile.send_report()                            │ │  │
│  │ └─────────────────────────────────────────────────────┘ │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │           第二阶段：处理持续提醒 (Nags)                     │  │
│  ├───────────────────────────────────────────────────────────┤  │
│  │                                                           │  │
│  │  SELECT * FROM profile                                    │  │
│  │  WHERE next_nag_date < NOW()                              │  │
│  │    AND nag_period != '0'                                  │  │
│  │  ORDER BY id LIMIT 1                                      │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │ │ 【先更新时间，后发送】                                 │ │  │
│  │ │ UPDATE profile SET next_nag_date = NOW() + nag_period│ │  │
│  │ │ WHERE id = ? AND next_nag_date = ?                    │ │  │
│  │ │                                                        │ │  │
│  │ │ IF 1 row updated:                                      │ │  │
│  │ │   if profile.send_report(nag=True):                    │ │  │
│  │ │       # 发送成功                                        │ │  │
│  │ │   else:                                                 │ │  │
│  │ │       # 没有down的检查了，清除调度                      │ │  │
│  │ │       profile.next_nag_date = NULL                     │ │  │
│  │ │       profile.save()                                    │ │  │
│  │ └─────────────────────────────────────────────────────┘ │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.9 持续提醒触发时机

持续提醒的 `next_nag_date` 会在以下时机被更新：

| 触发时机 | 触发位置 | 更新逻辑 |
|---------|---------|---------|
| 检查状态变化 | `sendalerts.py:34` | 调用 `project.update_next_nag_dates()` |
| 发送Nag后 | `sendreports.py:80` | `next_nag_date = now + nag_period` |
| 没有down的检查 | `sendreports.py:90` | `next_nag_date = None` |
| 用户修改nag设置 | `accounts/views.py:577-583` | 调用 `update_next_nag_date()` |

---

## 6. 邮件模板系统

### 5.1 邮件发送核心模块

#### 核心代码位置
- `hc/lib/emails.py` - 邮件发送核心模块

#### 消息构建

```python
# hc/lib/emails.py:40-65
def make_message(
    name: str, to: str | list[str], ctx: dict[str, Any], headers: dict[str, str] = {}
) -> Message:
    # 渲染标题模板
    subject = render(f"emails/{name}-subject.html", ctx).strip()
    # 渲染文本正文（替换非换行空格为普通空格）
    body = render(f"emails/{name}-body-text.html", ctx).replace("\xa0", " ")
    # 渲染HTML正文
    html = render(f"emails/{name}-body-html.html", ctx)
    
    # 构建Message-ID
    domain = settings.DEFAULT_FROM_EMAIL.split("@")[-1].strip(">")
    headers["Message-ID"] = make_msgid(domain=domain)
    
    # 设置发件人
    if "From" not in headers:
        headers["From"] = settings.DEFAULT_FROM_EMAIL
    
    # 自定义MAIL FROM地址（用于退信处理）
    bounce_id = headers.pop("X-Bounce-ID", "bounces")
    if settings.EMAIL_MAIL_FROM_TMPL:
        from_email = settings.EMAIL_MAIL_FROM_TMPL % bounce_id
    else:
        from_email = settings.DEFAULT_FROM_EMAIL
    
    # 创建多部分邮件
    to_list = [to] if isinstance(to, str) else to
    msg = Message(subject, body, from_email, to_list, headers=headers)
    msg.attach_alternative(html, "text/html")
    return msg
```

#### 模板命名约定

每个邮件类型需要三个模板文件：
- `{name}-subject.html` - 邮件标题
- `{name}-body-text.html` - 纯文本正文
- `{name}-body-html.html` - HTML正文

### 5.2 可用邮件类型

| 邮件类型 | 函数名 | 模板名 | 用途 |
|---------|-------|-------|-----|
| 登录/邀请 | `login()` | `login` | 免密登录链接、团队邀请 |
| 项目转让请求 | `transfer_request()` | `transfer-request` | 项目所有权转让通知 |
| 告警通知 | `alert()` | `alert` | 检查状态变更通知 |
| 邮箱验证 | `verify_email()` | `verify-email` | 通知渠道邮箱验证 |
| 定期报告 | `report()` | `report` | 月度/周度/日报 |
| 持续告警 | `nag()` | `nag` | 检查持续down时的提醒 |
| 抖动通知 | `flapping_notice()` | `flapping-notice` | 检查频繁切换状态 |
| 删除通知 | `deletion_notice()` | `deletion-notice` | 不活跃账号通知 |
| 计划删除 | `deletion_scheduled()` | `deletion-scheduled` | 账号即将删除 |
| SMS限制 | `sms_limit()` | `sms-limit` | SMS额度用完 |
| 电话限制 | `call_limit()` | `phone-call-limit` | 电话额度用完 |
| 特权验证码 | `sudo_code()` | `sudo-code` | 敏感操作二次验证 |
| Signal限流 | `signal_rate_limited()` | `signal-rate-limited` | Signal发送限流 |

### 5.3 异步发送机制

```python
# hc/lib/emails.py:68-82
def send(message: Message, block: bool = False) -> None:
    t = EmailThread(message)
    if block or hasattr(settings, "BLOCKING_EMAILS"):
        # 测试环境同步发送，以便检查外发邮件
        t.run()
    else:
        # 生产环境异步发送，避免用户等待
        t.start()
```

**邮件线程类**：
```python
# hc/lib/emails.py:15-37
class EmailThread(Thread):
    MAX_TRIES = 3
    
    def run(self) -> None:
        for attempt in range(0, self.MAX_TRIES):
            try:
                # 每次重试创建新连接
                self.message.connection = None
                self.message.send()
                return
            except (SMTPServerDisconnected, SMTPDataError) as e:
                if attempt + 1 == self.MAX_TRIES:
                    raise e
                # 等待1秒后重试
                time.sleep(1)
```

### 5.4 基础模板

所有HTML邮件模板继承自 `templates/emails/base.html`，包含：
- 统一的按钮样式
- 页脚信息
- 站点链接

---

## 6. 用户偏好对邮件发送的影响

### 6.1 Profile模型中的通知设置

#### 核心代码位置
- `hc/accounts/models.py:77-112` - Profile模型定义
- `hc/accounts/models.py:200-283` - send_report方法

#### 定期报告设置

```python
# hc/accounts/models.py:77-82
class Profile(models.Model):
    next_report_date = models.DateTimeField(null=True, blank=True)
    reports = models.CharField(max_length=10, default="monthly", choices=REPORT_CHOICES)
```

**报告频率选项**：
```python
REPORT_CHOICES = (
    ("off", "Off"),
    ("daily", "Daily"),
    ("weekly", "Weekly"),
    ("monthly", "Monthly"),
)
```

#### 持续告警（Nag）设置

```python
# hc/accounts/models.py:37-49
NO_NAG = td()
NAG_PERIODS = (
    (NO_NAG, "Disabled"),
    (td(hours=1), "Hourly"),
    (td(days=1), "Daily"),
)

class Profile(models.Model):
    nag_period = models.DurationField(default=NO_NAG, choices=NAG_PERIODS)
    next_nag_date = models.DateTimeField(null=True, blank=True)
```

### 6.2 定期报告发送逻辑

```python
# hc/accounts/models.py:200-283
def send_report(self, nag: bool = False) -> bool:
    # 检查是否有活跃ping（最近6个月内）
    q = self.checks_from_all_projects()
    result = q.aggregate(models.Max("last_ping"))
    last_ping = result["last_ping__max"]
    
    six_months_ago = now() - td(days=180)
    if last_ping is None or last_ping < six_months_ago:
        return False  # 没有活跃检查，不发送报告
    
    # 构建报告上下文
    ctx: dict[str, Any] = {
        "unsub_link": self.reports_unsub_url(),
        "notifications_url": self.notifications_url(),
        "tz": self.tz,
    }
    
    if not nag:
        # 定期报告：计算停机时间统计
        if self.reports == "monthly":
            boundaries = month_boundaries(3, self.tz)
        elif self.reports == "weekly":
            boundaries = week_boundaries(3, self.tz)
        elif self.reports == "daily":
            boundaries = day_boundaries(3, self.tz)
        
        # 计算每个检查的停机时间
        for check in checks:
            downtimes = check.downtimes_by_boundary(boundaries, self.tz)
            check.past_downtimes = downtimes[:-1]
        
        # 发送报告
        emails.report(self.user.email, ctx, headers)
    
    if nag:
        # 持续告警：只包含当前down的检查
        checks = [c for c in checks if c.get_status() == "down"]
        if not checks:
            return False
        
        ctx["checks"] = checks
        ctx["num_down"] = len(checks)
        ctx["nag_period"] = self.nag_period.total_seconds()
        emails.nag(self.user.email, ctx, headers)
    
    return True
```

### 6.3 报告发送调度

```python
# hc/accounts/models.py:351-376
def choose_next_report_date(self) -> datetime | None:
    """计算下一次月度/周度报告的目标日期
    
    月度报告：每月1日，用户时区的9AM-11AM之间
    周度报告：每周一，用户时区的9AM-11AM之间
    日报：每天，用户时区的9AM-11AM之间
    """
    if self.reports == "off":
        return None
    
    dt = now().astimezone(ZoneInfo(self.tz))
    # 在9AM-11AM之间随机选择时间
    dt = dt.replace(hour=9, minute=0) + td(minutes=random.randrange(0, 120))
    
    while True:
        dt += td(days=1)
        if self.reports == "daily":
            return dt
        if self.reports == "monthly" and dt.day == 1:
            return dt
        elif self.reports == "weekly" and dt.weekday() == 0:
            return dt
```

### 6.4 持续告警调度

```python
# hc/accounts/models.py:342-349
def update_next_nag_date(self) -> None:
    any_down = self.checks_from_all_projects().filter(status="down").exists()
    
    # 如果有检查down且未设置下次nag日期且启用了nag
    if any_down and self.next_nag_date is None and self.nag_period:
        self.next_nag_date = now() + self.nag_period
        self.save(update_fields=["next_nag_date"])
    # 如果没有检查down但设置了下次nag日期
    elif not any_down and self.next_nag_date:
        self.next_nag_date = None
        self.save(update_fields=["next_nag_date"])
```

### 6.5 报告退订

```python
# hc/accounts/views.py:640-680
def unsubscribe_reports(request: HttpRequest, signed_username: str) -> HttpResponse:
    signer = TimestampSigner(salt="reports")
    
    # 检查签名
    try:
        username = signer.unsign(signed_username)
    except BadSignature:
        return render(request, "bad_link.html")
    
    # 检查时间戳（超过5分钟则自动提交）
    try:
        autosubmit = False
        username = signer.unsign(signed_username, max_age=300)
    except SignatureExpired:
        autosubmit = True
    
    # POST请求时禁用报告
    if request.method == "POST":
        profile = Profile.objects.for_user(user)
        profile.reports = "off"
        profile.next_report_date = None
        profile.nag_period = td()
        profile.next_nag_date = None
        profile.save()
        return render(request, "accounts/unsubscribed.html")
```

### 6.6 时区偏好

```python
# hc/accounts/models.py:103
tz = models.CharField(max_length=36, default="UTC")
```

时区影响：
1. **报告发送时间**：基于用户时区计算报告发送时间
2. **日期时间格式化**：邮件中的所有时间显示使用用户时区
3. **停机时间计算**：基于用户时区的边界计算停机时间

### 6.7 通知设置表单

```python
# hc/accounts/forms.py:108-118
class ReportSettingsForm(forms.Form):
    reports = forms.ChoiceField(choices=REPORT_CHOICES)
    nag_period = forms.IntegerField(min_value=0, max_value=86400)
    
    def clean_nag_period(self) -> td:
        seconds = self.cleaned_data["nag_period"]
        
        # 有效值：0（禁用）、3600（每小时）、86400（每天）
        if seconds not in (0, 3600, 86400):
            raise forms.ValidationError(f"Bad nag_period: {seconds}")
        
        return td(seconds=seconds)
```

### 6.8 通知设置视图

```python
# hc/accounts/views.py:546-574
def notifications(request: AuthenticatedHttpRequest) -> HttpResponse:
    if request.method == "POST":
        form = forms.ReportSettingsForm(request.POST)
        if form.is_valid():
            profile.reports = form.cleaned_data["reports"]
            profile.next_report_date = profile.choose_next_report_date()
            
            if profile.nag_period != form.cleaned_data["nag_period"]:
                profile.nag_period = form.cleaned_data["nag_period"]
                # 更新下次nag日期
                if profile.nag_period:
                    profile.update_next_nag_date()
                else:
                    profile.next_nag_date = None
            
            profile.save()
            ctx["status"] = "info"
```

### 6.9 影响邮件发送路径的配置总结

| 配置项 | 存储位置 | 影响 |
|-------|---------|-----|
| `reports` | Profile.reports | 定期报告频率（off/daily/weekly/monthly） |
| `nag_period` | Profile.nag_period | 持续告警频率（禁用/每小时/每天） |
| `tz` | Profile.tz | 报告发送时间、日期时间格式化 |
| `notify_up/notify_down` | Channel.value | 单个邮件渠道的通知过滤 |
| `email_verified` | Channel.email_verified | 未验证邮箱无法发送告警 |
| `disabled` | Channel.disabled | 禁用的渠道不发送通知 |
| `EMAIL_USE_VERIFICATION` | settings | 全局邮箱验证开关 |
| `EMAIL_MAIL_FROM_TMPL` | settings | 自定义退信地址模板 |

---

## 附录：关键配置项

### Django Settings 配置

| 配置项 | 用途 |
|-------|-----|
| `EMAIL_HOST` | SMTP服务器地址（为空则禁用邮件功能） |
| `EMAIL_PORT` | SMTP端口 |
| `EMAIL_HOST_USER` | SMTP用户名 |
| `EMAIL_HOST_PASSWORD` | SMTP密码 |
| `EMAIL_USE_TLS/SSL` | 加密方式 |
| `DEFAULT_FROM_EMAIL` | 默认发件人 |
| `EMAIL_USE_VERIFICATION` | 是否启用邮箱验证 |
| `EMAIL_MAIL_FROM_TMPL` | 退信地址模板（如 `bounces+%s@example.com`） |
| `REGISTRATION_OPEN` | 是否开放注册 |
| `SESSION_COOKIE_SECURE` | 会话Cookie是否仅HTTPS |
| `SITE_NAME` | 站点名称 |
| `SITE_ROOT` | 站点根URL |

### 相关模型关系

```
User (Django内置)
  └── Profile (1:1)
        ├── reports: 报告频率
        ├── nag_period: 持续告警间隔
        ├── tz: 时区
        └── token: 登录令牌哈希
  └── Member (1:N)
        ├── project: 所属项目
        └── role: 角色 (r/w/m)
  └── Credential (1:N) - WebAuthn安全密钥

Project
  ├── owner: User (项目所有者)
  ├── Member (1:N) - 团队成员
  ├── Check (1:N) - 监控检查
  └── Channel (1:N) - 通知渠道
        ├── kind: 渠道类型 (email/sms/slack/...)
        ├── value: 配置数据 (JSON或纯文本)
        ├── email_verified: 邮箱是否验证
        └── disabled: 是否禁用

Check
  ├── status: 当前状态 (up/down/new/paused)
  └── Flip (1:N) - 状态变更记录
        └── Notification (1:N) - 通知记录
```
