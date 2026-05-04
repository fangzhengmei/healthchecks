# Healthchecks 自托管部署配置分析

本文档分析 Healthchecks 自托管部署中，各类配置如何影响站点 URL、邮件链接、ping endpoint、静态资源和安全配置。

---

## 目录

1. [核心配置概述](#核心配置概述)
2. [SITE_ROOT - 站点根 URL 配置](#site_root---站点根-url-配置)
3. [PING_ENDPOINT - Ping 端点配置](#ping_endpoint---ping-端点配置)
4. [邮件配置与链接生成](#邮件配置与链接生成)
5. [静态资源配置](#静态资源配置)
6. [安全配置、反向代理与 HTTPS Header](#安全配置反向代理与-https-header)
7. [HTTPS Header 配置错误的连锁影响](#https-header-配置错误的连锁影响)
8. [可落地的排查步骤](#可落地的排查步骤)
9. [配置依赖关系图](#配置依赖关系图)
10. [快速配置检查清单](#快速配置检查清单)

---

## 核心配置概述

Healthchecks 的配置主要通过环境变量读取，定义在 `hc/settings.py` 中。以下是与 URL 相关的核心配置项：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `SITE_ROOT` | `http://localhost:8000` | 站点的公开基础 URL |
| `SITE_NAME` | `Mychecks` | 站点名称 |
| `PING_ENDPOINT` | `SITE_ROOT + "/ping/"` | Ping 端点的基础 URL |
| `ALLOWED_HOSTS` | SITE_ROOT 的域名部分 | 允许的主机名列表 |
| `STATIC_URL` | SITE_ROOT path + "/static/" | 静态资源 URL 前缀 |
| `LOGIN_URL` | SITE_ROOT path + "/accounts/login/" | 登录页面 URL |
| `SECURE_PROXY_SSL_HEADER` | None | 反向代理 HTTPS header 配置 |

---

## SITE_ROOT - 站点根 URL 配置

### 配置定义

**文件**: `hc/settings.py:259`

```python
SITE_ROOT = os.getenv("SITE_ROOT", "http://localhost:8000").removesuffix("/")
```

### 影响范围

SITE_ROOT 是整个系统中最重要的配置之一，它影响以下方面：

#### 1. ALLOWED_HOSTS 自动推导

**文件**: `hc/settings.py:273-279`

```python
if v := os.getenv("ALLOWED_HOSTS"):
    ALLOWED_HOSTS = v.split(",")
else:
    domain, _ = split_domain_port(_site_root_parts.netloc)
    ALLOWED_HOSTS = [domain]
```

如果未显式设置 `ALLOWED_HOSTS`，系统会自动从 `SITE_ROOT` 中提取域名作为允许的主机名。

#### 2. URL 路由前缀

**文件**: `hc/urls.py:11-13`

```python
prefix = ""
if _path := urlparse(settings.SITE_ROOT).path.lstrip("/"):
    prefix = f"{_path}/"
```

当 `SITE_ROOT` 包含路径前缀时（如 `http://example.com/monitoring`），所有 URL 路由都会自动添加此前缀。

#### 3. LOGIN_URL 和 STATIC_URL

**文件**: `hc/settings.py:270-272`

```python
_site_root_parts = urlparse(SITE_ROOT)
LOGIN_URL = f"{_site_root_parts.path}/accounts/login/"
STATIC_URL = f"{_site_root_parts.path}/static/"
```

登录 URL 和静态资源 URL 都基于 `SITE_ROOT` 的 path 部分构建。

#### 4. PING_ENDPOINT 默认值

**文件**: `hc/settings.py:263`

```python
PING_ENDPOINT = os.getenv("PING_ENDPOINT", SITE_ROOT + "/ping/")
```

`PING_ENDPOINT` 默认为 `SITE_ROOT` 加上 `/ping/` 后缀。

### 绝对 URL 生成

系统提供了两个核心函数来生成绝对 URL：

**文件**: `hc/lib/urls.py:10-21`

```python
def absolute_url(path: str) -> str:
    subpath = urlparse(settings.SITE_ROOT).path
    return settings.SITE_ROOT.removesuffix(subpath) + path

def absolute_reverse(
    viewname: str | Callable[..., HttpResponse],
    args: Sequence[Any] | None = None,
    query: dict[str, str] | None = None,
) -> str:
    return absolute_url(reverse(viewname, args=args, query=query))
```

**关键点**：
- `absolute_url()` 会移除 `SITE_ROOT` 中的路径部分，然后拼接传入的 path
- 这确保了当 `SITE_ROOT` 包含子路径时，生成的 URL 仍然正确
- **重要**：这些函数直接使用 `settings.SITE_ROOT`，**不依赖**请求信息或 `X-Forwarded-*` header

### 使用场景

`SITE_ROOT` 在以下场景中被使用：

| 场景 | 代码位置 | 用途 |
|------|----------|------|
| 邮件链接 | `hc/lib/urls.py` | 生成邮件中的绝对链接 |
| API 响应 | `hc/api/models.py:453` | 生成 `update_url`、`pause_url`、`resume_url` |
| 模板标签 | `hc/front/templatetags/hc_extras.py:74` | `{% site_root %}` 模板标签 |
| OAuth 回调 | 各集成 views.py | 生成 OAuth 重定向 URI |
| Webhook 配置 | `hc/api/management/commands/settelegramwebhook.py` | 设置 Telegram Webhook URL |

### 系统检查

**文件**: `hc/api/apps.py:25-43`

系统会在启动时检查 `SITE_ROOT` 的有效性：

```python
site_root_parts = urlsplit(settings.SITE_ROOT)
if not site_root_parts.scheme:
    # 警告：SITE_ROOT 应该以 http:// 或 https:// 开头

host, _ = split_domain_port(site_root_parts.netloc)
if site_root_parts.scheme and not validate_host(host, settings.ALLOWED_HOSTS):
    # 错误：SITE_ROOT 中的主机名不在 ALLOWED_HOSTS 中
```

### 子路径部署示例

如果需要在子路径下部署（如 `http://example.com/monitoring`）：

```bash
SITE_ROOT=http://example.com/monitoring
```

系统会自动：
1. 将 `ALLOWED_HOSTS` 设置为 `['example.com']`
2. 将 `LOGIN_URL` 设置为 `/monitoring/accounts/login/`
3. 将 `STATIC_URL` 设置为 `/monitoring/static/`
4. 所有 URL 路由添加 `/monitoring/` 前缀

---

## PING_ENDPOINT - Ping 端点配置

### 配置定义

**文件**: `hc/settings.py:263-264`

```python
PING_ENDPOINT = os.getenv("PING_ENDPOINT", SITE_ROOT + "/ping/")
PING_EMAIL_DOMAIN = os.getenv("PING_EMAIL_DOMAIN", "localhost")
```

### 重要特性

根据官方文档（`templates/docs/self_hosted_configuration.md`）：

> **关键点**: `PING_ENDPOINT` 仅用于**显示** ping URL，**不影响** HTTP 请求的路由。

这意味着：
- 你可以将 `PING_ENDPOINT` 设置为与 `SITE_ROOT` 不同的域名（如 `https://ping.example.org/`）
- 但你需要在反向代理中配置相应的路由规则
- **重要**：`PING_ENDPOINT` 直接使用配置值，**不依赖** `X-Forwarded-*` header

### 使用场景

#### 1. Check.ping_url() 方法

**文件**: `hc/api/models.py:249-257`

```python
@property
def ping_url(self) -> str | None:
    if self.project_id and self.project.show_slugs:
        if not self.slug:
            return None
        key = self.project.ping_key or "{ping_key}"
        return settings.PING_ENDPOINT + key + "/" + self.slug

    return settings.PING_ENDPOINT + str(self.code)
```

根据项目配置，生成两种格式的 ping URL：
- 使用 slug: `PING_ENDPOINT/<ping-key>/<slug>`
- 使用 UUID: `PING_ENDPOINT/<uuid>`

#### 2. 邮件 Ping 地址

**文件**: `hc/api/models.py:270-285`

```python
def email(self) -> str | None:
    if self.project_id and self.project.show_slugs:
        # ...
        return f"{key}+{self.slug}@{settings.PING_EMAIL_DOMAIN}"

    return f"{self.code}@{settings.PING_EMAIL_DOMAIN}"
```

#### 3. API 响应

**文件**: `hc/api/models.py:449`

```python
result["ping_url"] = settings.PING_ENDPOINT + str(self.code)
```

API 返回的 check 对象中包含 `ping_url` 字段。

#### 4. 模板中的格式化显示

**文件**: `hc/front/templatetags/hc_extras.py:242-253`

```python
FORMATTED_PING_ENDPOINT_TMPL = (
    f"""<span class="base">{settings.PING_ENDPOINT}</span>{{}}"""
)

@register.filter
def format_ping_endpoint(ping_url: str) -> SafeString:
    assert ping_url.startswith(settings.PING_ENDPOINT)
    tail = ping_url.removeprefix(settings.PING_ENDPOINT)
    return format_html(FORMATTED_PING_ENDPOINT_TMPL, tail)
```

在前端页面中，ping URL 会被格式化显示，基础部分会被特殊样式包裹。

### 独立 Ping 域名配置示例

如果你希望使用独立的域名作为 ping 端点：

```bash
SITE_ROOT=https://hc.example.org
PING_ENDPOINT=https://ping.example.org/
```

**注意**：你需要在反向代理中配置 `ping.example.org` 的路由，将请求转发到 Healthchecks 应用。

---

## 邮件配置与链接生成

### 邮件配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `DEFAULT_FROM_EMAIL` | `healthchecks@example.org` | 发件人地址 |
| `EMAIL_HOST` | `""` | SMTP 服务器地址 |
| `EMAIL_PORT` | `587` | SMTP 端口 |
| `EMAIL_HOST_USER` | `""` | SMTP 用户名 |
| `EMAIL_HOST_PASSWORD` | `""` | SMTP 密码 |
| `EMAIL_USE_TLS` | `True` | 是否使用 TLS |
| `EMAIL_USE_SSL` | `False` | 是否使用 SSL |
| `EMAIL_USE_VERIFICATION` | `True` | 是否需要邮件验证 |
| `EMAIL_MAIL_FROM_TMPL` | `""` | 自定义 MAIL FROM 模板 |

### 邮件发送机制

**文件**: `hc/lib/emails.py:40-65`

```python
def make_message(
    name: str, to: str | list[str], ctx: dict[str, Any], headers: dict[str, str] = {}
) -> Message:
    subject = render(f"emails/{name}-subject.html", ctx).strip()
    body = render(f"emails/{name}-body-text.html", ctx).replace("\xa0", " ")
    html = render(f"emails/{name}-body-html.html", ctx)
    # ...
```

邮件使用 Django 模板系统渲染，支持纯文本和 HTML 两种格式。

### 邮件中的链接生成

邮件中的链接通过以下方式生成：

#### 1. absolute_reverse() 函数

**文件**: `hc/api/models.py:1025`（验证邮件链接）

```python
def verify_link(self) -> str:
    args = [self.project.code, signed_token(self.email, salt="verify-email")]
    verify_link = absolute_reverse("hc-verify-email", args=args)
```

**文件**: `hc/api/models.py:1032`（退订链接）

```python
def unsub_link(self, check: "Check") -> str:
    args = [check.code, signed_token(self.email, salt="unsubscribe-alerts")]
    return absolute_reverse("hc-unsubscribe-alerts", args=args)
```

#### 2. 模型方法生成的链接

| 方法 | 用途 | 示例 |
|------|------|------|
| `check.cloaked_url()` | 隐藏真实 URL 的检查详情链接 | `SITE_ROOT/cloaked/{unique_key}/` |
| `check.details_url()` | 检查详情页链接 | `SITE_ROOT/checks/{code}/details/` |
| `project.checks_url` | 项目检查列表链接 | `SITE_ROOT/projects/{code}/checks/` |

#### 3. 模板标签

**文件**: `hc/front/templatetags/hc_extras.py`

```python
@register.simple_tag
def site_root() -> str:
    return settings.SITE_ROOT

@register.simple_tag
def absolute_site_logo_url() -> str:
    url = settings.SITE_LOGO_URL or static("img/logo.png")
    if url.startswith("/"):
        url = absolute_url(url)
    return url
```

### 邮件模板示例

**登录邮件模板** (`templates/emails/login-body-html.html`):

```html
<a href="{% site_root %}">{% site_name %}</a>
```

**告警邮件模板** (`templates/emails/alert-body-html.html`):

```html
<a href="{{ check.cloaked_url }}">View on {% site_name %}&hellip;</a>
<!-- ... -->
<a href="{{ unsub_link }}" target="_blank">Unsubscribe</a>
```

### 邮件基础模板

**文件**: `templates/emails/base.html`

所有邮件模板都继承自此基础模板，其中包含：

```html
<img alt="{% site_name %}" src="{% absolute_site_logo_url %}" ...>
```

Logo URL 会自动转换为绝对 URL。

### 关键要点

**邮件中的所有链接都直接使用 `SITE_ROOT` 配置值，不依赖 `X-Forwarded-*` header。**

这意味着：
- 如果 `SITE_ROOT=http://example.com`，邮件中的链接就是 `http://...`
- 如果 `SITE_ROOT=https://example.com`，邮件中的链接就是 `https://...`
- 反向代理的 header 配置**不会影响**邮件中链接的协议

---

## 静态资源配置

### 配置项

| 配置项 | 值 | 说明 |
|--------|-----|------|
| `STATICFILES_DIRS` | `[BASE_DIR / "static"]` | 静态资源源目录 |
| `STATIC_ROOT` | `BASE_DIR / "static-collected"` | 收集后的静态资源目录 |
| `STATIC_URL` | 动态派生 | 静态资源 URL 前缀 |
| `COMPRESS_OFFLINE` | `True` | 是否启用离线压缩 |

### STATIC_URL 的派生

**文件**: `hc/settings.py:272`

```python
STATIC_URL = f"{_site_root_parts.path}/static/"
```

`STATIC_URL` 基于 `SITE_ROOT` 的 path 部分动态生成：

| SITE_ROOT | STATIC_URL |
|-----------|------------|
| `http://localhost:8000` | `/static/` |
| `http://example.com/monitoring` | `/monitoring/static/` |

### 静态文件查找器

**文件**: `hc/settings.py:283-287`

```python
STATICFILES_FINDERS = (
    "django.contrib.staticfiles.finders.FileSystemFinder",
    "django.contrib.staticfiles.finders.AppDirectoriesFinder",
    "compressor.finders.CompressorFinder",
)
```

### 压缩配置

**文件**: `hc/settings.py:288-299`

```python
COMPRESS_OFFLINE = True
COMPRESS_CSS_HASHING_METHOD = "content"
COMPRESS_STORAGE = "compressor.storage.GzipCompressorFileStorage"
COMPRESS_FILTERS = {
    "css": [
        "compressor.filters.css_default.CssRelativeFilter",
        "compressor.filters.cssmin.rCSSMinFilter",
    ],
    "js": ["compressor.filters.jsmin.rJSMinFilter"],
}
```

**关键点**: 使用 `CssRelativeFilter` 而不是 `CssAbsoluteFilter`，这是为了**修复子目录部署时的图标字体加载问题**。

### WhiteNoise 中间件

**文件**: `hc/settings.py:140-141`

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    # ...
]
```

**文件**: `hc/settings.py:302-306`

```python
def immutable_file_test(path: Any, url: str) -> bool:
    return "/static/CACHE/" in url or "/static/fonts/" in url

WHITENOISE_IMMUTABLE_FILE_TEST = immutable_file_test
```

WhiteNoise 用于高效地提供静态文件服务，并且为缓存文件和字体文件设置了不可变缓存规则。

### 静态资源与 HTTPS Header 的关系

静态资源的 URL 路径（`STATIC_URL`）基于 `SITE_ROOT` 派生，但**实际访问时**：

1. **浏览器请求**: 浏览器通过页面中的 `<link>` 和 `<script>` 标签加载静态资源
2. **相对路径**: 静态资源通常使用相对路径（如 `/static/css/base.css`）
3. **协议继承**: 浏览器会使用当前页面的协议（http 或 https）来请求静态资源

**关键点**：
- `STATIC_URL` 只定义路径部分，不包含协议和域名
- 静态资源的加载协议由页面的访问协议决定
- 如果页面通过 https 访问，静态资源也通过 https 加载
- 但这依赖于浏览器正确识别页面的协议

---

## 安全配置、反向代理与 HTTPS Header

### SECURE_PROXY_SSL_HEADER 配置

#### 配置定义

**文件**: `hc/settings.py:79-80`

```python
if v := os.getenv("SECURE_PROXY_SSL_HEADER"):
    SECURE_PROXY_SSL_HEADER = tuple(v.split(",", maxsplit=1))
```

#### 环境变量格式

```bash
SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https
```

或者在 `local_settings.py` 中：

```python
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
```

#### 这个配置的作用

`SECURE_PROXY_SSL_HEADER` 是 Django 的标准配置，告诉 Django：

> "当请求包含某个 header 且值为特定值时，认为这个请求是通过 HTTPS 发送的。"

**工作原理**：

```
用户浏览器 --(HTTPS)--> 反向代理(Nginx) --(HTTP)--> Django(uWSGI)
                                    ↓
                    设置 X-Forwarded-Proto: https
                                    ↓
                    Django 读取 SECURE_PROXY_SSL_HEADER 配置
                                    ↓
                    request.is_secure() 返回 True
```

#### 系统检查

**文件**: `hc/api/apps.py:54-62`

```python
v = settings.SECURE_PROXY_SSL_HEADER
if v is not None and (not isinstance(v, tuple) or len(v) != 2):
    items.append(
        Warning(
            "settings.SECURE_PROXY_SSL_HEADER is not 2-element tuple",
            hint="See https://healthchecks.io/docs/self_hosted_configuration/#SECURE_PROXY_SSL_HEADER",
            id="hc.api.W005",
        )
    )
```

### 反向代理配置

根据官方文档（`templates/docs/self_hosted_docker.md`），当使用反向代理时，需要正确配置以下 Header：

#### 1. X-Forwarded-Proto

这个 Header 告诉 Django 客户端原始请求使用的协议（HTTP 或 HTTPS）。

**重要**: Docker 部署使用的 uWSGI 依赖 `X-Forwarded-Proto` Header 来确定请求是否安全。

#### 2. X-Forwarded-For

这个 Header 用于记录客户端的真实 IP 地址。

**安全警告**: 如果反向代理没有设置 `X-Forwarded-For`，客户端可以伪造自己的 IP 地址，从而绕过 IP 限流等安全措施。

### Nginx 配置示例

```nginx
location / {
    proxy_pass http://localhost:8000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

**关键配置项**：
- `proxy_set_header X-Forwarded-Proto $scheme;` - 将原始请求协议传递给后端
- `proxy_set_header Host $host;` - 将原始 Host header 传递给后端

### HAProxy 配置示例

```haproxy
http-request set-header X-Forwarded-Proto https if { ssl_fc }
http-request set-header X-Forwarded-Proto http unless { ssl_fc }
```

### 其他安全相关配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `SECRET_KEY` | `"---"` | Django 密钥，用于加密签名 |
| `DEBUG` | `True` | 调试模式，生产环境必须设为 False |
| `REGISTRATION_OPEN` | `True` | 是否开放用户注册 |
| `RP_ID` | `None` | WebAuthn 依赖方 ID（用于无密码登录） |

### SECRET_KEY 的安全加载

**文件**: `hc/settings.py:49-66`

```python
def envsecret(s: str, default: str | None = None) -> str | None:
    """Load a secret from an environment variable or from the filesystem.

    This function either reads the secret from a file (if s + "_FILE" environment
    variable has a non-empty value), or calls os.getenv().
    """
    if secret_path := os.getenv(s + "_FILE"):
        p = Path(secret_path)
        if not p.is_file():
            raise Exception(f"Error reading {s}_FILE ({secret_path})")
        return p.read_text().strip()

    return os.getenv(s, default)
```

支持两种方式加载密钥：
1. 直接从环境变量：`SECRET_KEY=your-secret-key`
2. 从文件加载（Docker Secrets 风格）：`SECRET_KEY_FILE=/path/to/secret/file`

支持 `*_FILE` 后缀的配置项：
- `SECRET_KEY_FILE`
- `DB_PASSWORD_FILE`
- `DISCORD_CLIENT_SECRET_FILE`
- `GITHUB_CLIENT_SECRET_FILE`
- `GITHUB_PRIVATE_KEY_FILE`
- `MATRIX_ACCESS_TOKEN_FILE`
- `NTFY_SH_TOKEN_FILE`
- `PUSHOVER_API_TOKEN_FILE`
- `PUSHBULLET_CLIENT_SECRET_FILE`
- `SLACK_CLIENT_SECRET_FILE`
- `TELEGRAM_TOKEN_FILE`
- `TRELLO_APP_KEY_FILE`
- `TWILIO_AUTH_FILE`

### 调试模式警告

**文件**: `hc/front/templatetags/hc_extras.py:89-108`

```python
@register.simple_tag
def debug_warning() -> str:
    if settings.DEBUG:
        return mark_safe("""<div>Running in debug mode, do not use in production.</div>""")
    
    if settings.SECRET_KEY == "---":
        return mark_safe("""<div>Running with an insecure SECRET_KEY value...</div>""")
    
    return ""
```

当 `DEBUG=True` 或 `SECRET_KEY` 使用默认值时，页面会显示警告。

---

## HTTPS Header 配置错误的连锁影响

### 核心概念澄清

在分析影响之前，需要明确一个**关键区别**：

| 配置项 | 用途 | 依赖 |
|--------|------|------|
| `SITE_ROOT` | 生成绝对 URL（邮件、API 响应等） | **不依赖**请求信息 |
| `SECURE_PROXY_SSL_HEADER` | 告诉 Django 如何判断请求是否为 HTTPS | 依赖 `X-Forwarded-*` header |
| `PING_ENDPOINT` | 显示 ping URL | **不依赖**请求信息 |

**这是一个非常重要的区别**：
- `SITE_ROOT` 和 `PING_ENDPOINT` 是**静态配置**，应用启动时确定
- 邮件链接、API 响应中的 URL 直接使用这些配置值
- `X-Forwarded-*` header 只影响**当前请求**的处理逻辑

### 配置错误的场景分类

#### 场景 1: SITE_ROOT 协议错误（http vs https）

**配置示例**：
```bash
# 错误：用户通过 https 访问，但 SITE_ROOT 用的是 http
SITE_ROOT=http://hc.example.com
SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https
```

**影响分析**：

| 受影响项 | 具体影响 | 严重程度 |
|----------|----------|----------|
| **邮件链接** | 邮件中的链接是 `http://...`，用户点击后可能被浏览器重定向到 https，或者显示安全警告 | 🔴 高 |
| **API 响应** | API 返回的 `ping_url`、`update_url` 等都是 `http://...` | 🟡 中 |
| **前端页面** | 页面本身通过 https 访问（因为反向代理处理了 TLS），但页面中的 `{% site_root %}` 输出的是 `http://...` | 🟡 中 |
| **OAuth 集成** | Slack、Discord 等 OAuth 回调 URL 配置错误，导致集成失败 | 🔴 高 |
| **静态资源** | 静态资源使用相对路径，不受直接影响 | 🟢 低 |

**实际案例**：
- 用户收到告警邮件，点击链接 `http://hc.example.com/checks/...`
- 浏览器显示"不安全"警告，或者被 HSTS 策略强制跳转到 https
- 用户体验差，甚至可能放弃访问

#### 场景 2: SECURE_PROXY_SSL_HEADER 未配置，但实际使用 HTTPS

**配置示例**：
```bash
# 错误：反向代理发送 X-Forwarded-Proto: https，但 Django 不信任它
SITE_ROOT=https://hc.example.com
# SECURE_PROXY_SSL_HEADER 未设置！
```

**影响分析**：

这是一个**更隐蔽**的问题，影响的是 Django 的运行时行为：

| 受影响项 | 具体影响 | 严重程度 |
|----------|----------|----------|
| **`request.is_secure()`** | 返回 `False`，Django 认为请求是 HTTP | 🔴 高 |
| **Session Cookie** | 如果设置了 `SESSION_COOKIE_SECURE=True`，Cookie 的 `Secure` 属性会被设置，但 Django 认为请求是 HTTP，可能导致会话问题 | 🟡 中 |
| **CSRF 验证** | Django 的 CSRF 保护在某些情况下会检查协议，可能导致 403 错误 | 🔴 高 |
| **uWSGI 行为** | 根据官方文档，uWSGI 依赖 `X-Forwarded-Proto` 来判断请求安全性，配置错误会导致 CSRF 验证失败 | 🔴 高 |
| **WebAuthn** | WebAuthn（无密码登录）要求 HTTPS 环境，`request.is_secure()` 返回 `False` 会导致 WebAuthn 无法使用 | 🟡 中 |
| **SITE_ROOT** | 不受影响，因为是静态配置 | 🟢 无 |
| **邮件链接** | 不受影响，因为使用 SITE_ROOT | 🟢 无 |

**官方文档引用**（`templates/docs/self_hosted_docker.md`）：

> **Important:** This Dockerfile uses uWSGI, which relies on the [X-Forwarded-Proto](...) header to determine if a request is secure or not. Without this information you may run into **HTTP 403 "CSRF verification failed." errors** when using your Healthchecks instance.

#### 场景 3: 反向代理未正确设置/覆盖 X-Forwarded-Proto

**风险场景**：
```
用户 --(HTTP)--> 攻击者 --(伪造 X-Forwarded-Proto: https)--> 反向代理 ---> Django
```

如果反向代理**信任**用户发送的 `X-Forwarded-Proto` header 而不覆盖它：

| 风险 | 说明 |
|------|------|
| **协议欺骗** | 用户可以发送 `X-Forwarded-Proto: https`，让 Django 认为是 HTTPS 请求 |
| **安全绕过** | 某些安全检查可能被绕过 |
| **Cookie 泄露** | 如果 Cookie 没有 `Secure` 属性，可能在 HTTP 连接中泄露 |

**正确的反向代理配置应该**：
1. 丢弃用户发送的 `X-Forwarded-*` header
2. 根据实际连接情况重新设置这些 header

#### 场景 4: X-Forwarded-For 配置错误

**配置问题**：
- 反向代理没有设置 `X-Forwarded-For`
- 或者信任用户发送的 `X-Forwarded-For`

**影响分析**：

| 受影响项 | 具体影响 | 严重程度 |
|----------|----------|----------|
| **IP 限流** | 登录表单的 IP 限流可能被绕过，攻击者可以暴力破解密码 | 🔴 高 |
| **日志记录** | 日志中记录的客户端 IP 不正确，影响安全审计 | 🟡 中 |
| **速率限制** | API 速率限制可能被绕过 | 🟡 中 |

**官方文档警告**（`templates/docs/self_hosted_docker.md`）：

> **Important:** configure the reverse proxy to set the `X-Forwarded-For` request header. Healthchecks trusts it to determine the client's IP address. If the proxy does not set the `X-Forwarded-For` header, the clients can pass their own value and circumvent, among other things, the **IP-based rate limiting in the login form**.

### 连锁影响总结

#### 直接影响 vs 间接影响

```
┌─────────────────────────────────────────────────────────────────┐
│                     配置错误影响链                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SITE_ROOT=http://... (协议错误)                                │
│       │                                                        │
│       ├──► 邮件链接是 http://...                               │
│       │         │                                              │
│       │         ├──► 用户点击时浏览器安全警告                   │
│       │         ├──► HSTS 强制重定向（如果有）                 │
│       │         └──► 某些邮件客户端可能阻止访问                 │
│       │                                                        │
│       ├──► API 响应中的 URL 是 http://...                      │
│       │         │                                              │
│       │         └──► 客户端使用 http 调用，可能被重定向或拦截   │
│       │                                                        │
│       └──► OAuth 回调 URL 配置错误                             │
│                 │                                              │
│                 └──► Slack/Discord 等集成无法使用              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SECURE_PROXY_SSL_HEADER 未配置                                │
│       │                                                        │
│       ├──► request.is_secure() 返回 False                     │
│       │         │                                              │
│       │         ├──► CSRF 验证可能失败 (403 错误)             │
│       │         ├──► WebAuthn 无法使用                        │
│       │         └──► 某些安全中间件行为异常                    │
│       │                                                        │
│       └──► uWSGI 无法正确判断请求协议                          │
│                 │                                              │
│                 └──► 官方文档明确警告会导致 CSRF 问题          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  X-Forwarded-For 未正确配置                                    │
│       │                                                        │
│       ├──► 客户端可以伪造 IP 地址                              │
│       │         │                                              │
│       │         ├──► 绕过登录限流，暴力破解密码                │
│       │         └──► 日志中 IP 不可信，影响安全审计            │
│       │                                                        │
│       └──► 如果反向代理信任用户 header                         │
│                 │                                              │
│                 └──► 攻击者可以完全控制这些 header 的值        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 受影响功能的详细列表

| 功能模块 | SITE_ROOT 协议错误 | SECURE_PROXY_SSL_HEADER 错误 | X-Forwarded-For 错误 |
|----------|-------------------|-------------------------------|----------------------|
| 邮件链接 | 🔴 协议错误 | 🟢 无影响 | 🟢 无影响 |
| API 响应 URL | 🔴 协议错误 | 🟢 无影响 | 🟢 无影响 |
| Ping 展示地址 | 🔴 协议错误 | 🟢 无影响 | 🟢 无影响 |
| OAuth 集成 | 🔴 回调 URL 错误 | 🟢 无影响 | 🟢 无影响 |
| CSRF 保护 | 🟢 无影响 | 🔴 可能 403 错误 | 🟢 无影响 |
| Session 管理 | 🟢 无影响 | 🟡 可能异常 | 🟢 无影响 |
| WebAuthn 登录 | 🟢 无影响 | 🟡 无法使用 | 🟢 无影响 |
| 登录限流 | 🟢 无影响 | 🟢 无影响 | 🔴 可被绕过 |
| 安全审计 | 🟢 无影响 | 🟢 无影响 | 🔴 IP 不可信 |

---

## 可落地的排查步骤

### 排查前的准备

在开始排查之前，确认以下信息：

1. **用户访问方式**：用户是通过 http 还是 https 访问？
2. **反向代理类型**：使用的是 Nginx、HAProxy、Traefik 还是其他？
3. **部署方式**：Docker、直接运行、还是其他方式？
4. **症状表现**：具体是什么问题？（邮件链接错误？403 错误？无法登录？）

### 第一阶段：配置静态检查

#### 步骤 1: 检查 SITE_ROOT 配置

**检查项**：
- `SITE_ROOT` 的协议是否正确？
- `SITE_ROOT` 的域名是否正确？
- 如果是子路径部署，路径是否正确？

**验证方法**：

```bash
# 方法 1: 检查环境变量
echo $SITE_ROOT

# 方法 2: 通过 Django shell 检查
python manage.py shell -c "from django.conf import settings; print('SITE_ROOT:', settings.SITE_ROOT)"

# 方法 3: 检查系统检查
python manage.py check
```

**预期结果**：
- 应该以 `https://` 开头（如果是生产环境）
- 域名应该与用户实际访问的域名一致

**常见错误**：
```bash
# 错误：使用 http
SITE_ROOT=http://hc.example.com

# 错误：端口不对
SITE_ROOT=https://hc.example.com:8000

# 错误：域名不对
SITE_ROOT=https://wrong-domain.com
```

#### 步骤 2: 检查 SECURE_PROXY_SSL_HEADER 配置

**检查项**：
- 是否配置了 `SECURE_PROXY_SSL_HEADER`？
- 配置格式是否正确？

**验证方法**：

```bash
# 检查环境变量
echo $SECURE_PROXY_SSL_HEADER

# 通过 Django shell 检查
python manage.py shell -c "
from django.conf import settings
print('SECURE_PROXY_SSL_HEADER:', getattr(settings, 'SECURE_PROXY_SSL_HEADER', 'NOT SET'))
"

# 运行系统检查
python manage.py check
```

**预期结果**（生产环境使用反向代理时）：
```
SECURE_PROXY_SSL_HEADER: ('HTTP_X_FORWARDED_PROTO', 'https')
```

**常见错误**：
```bash
# 错误：未设置
SECURE_PROXY_SSL_HEADER= （空或未设置）

# 错误：格式错误（元组格式不对）
SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO
```

#### 步骤 3: 检查 ALLOWED_HOSTS 配置

**检查项**：
- `ALLOWED_HOSTS` 是否包含用户访问的域名？

**验证方法**：

```bash
# 通过 Django shell 检查
python manage.py shell -c "
from django.conf import settings
print('ALLOWED_HOSTS:', settings.ALLOWED_HOSTS)
"
```

**预期结果**：
- 应该包含用户实际访问的域名
- 如果 `SITE_ROOT` 配置正确，这通常会自动推导正确

#### 步骤 4: 检查 PING_ENDPOINT 配置（如果使用独立域名）

**检查项**：
- 如果使用独立的 ping 域名，`PING_ENDPOINT` 是否正确？

**验证方法**：

```bash
# 通过 Django shell 检查
python manage.py shell -c "
from django.conf import settings
print('SITE_ROOT:', settings.SITE_ROOT)
print('PING_ENDPOINT:', settings.PING_ENDPOINT)
"
```

**预期结果**：
- 如果不使用独立 ping 域名，`PING_ENDPOINT` 应该是 `SITE_ROOT + '/ping/'`
- 如果使用独立域名，应该是正确的 `https://ping.example.com/`

### 第二阶段：运行时行为检查

#### 步骤 5: 检查请求协议识别

**目标**：验证 Django 是否正确识别请求是 HTTP 还是 HTTPS

**验证方法**：

创建一个临时测试视图，或者使用 Django shell 模拟请求：

```python
# 在 Django shell 中运行
python manage.py shell

# 输入以下内容：
from django.test import RequestFactory
from django.conf import settings

factory = RequestFactory()

# 模拟带有 X-Forwarded-Proto header 的请求
request = factory.get('/', HTTP_X_FORWARDED_PROTO='https')

# 检查 request.is_secure() 的返回值
print(f"request.is_secure(): {request.is_secure()}")
print(f"SECURE_PROXY_SSL_HEADER: {getattr(settings, 'SECURE_PROXY_SSL_HEADER', 'NOT SET')}")
```

**预期结果（HTTPS 环境）**：
```
request.is_secure(): True
```

**异常结果及处理**：

| 结果 | 可能原因 | 解决方案 |
|------|----------|----------|
| `request.is_secure()` 返回 `False` | `SECURE_PROXY_SSL_HEADER` 未配置或配置错误 | 正确配置 `SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https` |

#### 步骤 6: 验证反向代理 Header 设置

**目标**：确认反向代理是否正确设置了必要的 header

**验证方法**：

创建一个临时的调试视图来显示请求 header：

```python
# 在 hc/front/views.py 中临时添加（记得之后删除）

def debug_headers(request):
    from django.http import JsonResponse
    headers = {k: v for k, v in request.META.items() if k.startswith('HTTP_')}
    return JsonResponse({
        'is_secure': request.is_secure(),
        'scheme': request.scheme,
        'headers': headers,
        'HOST': request.META.get('HTTP_HOST'),
        'X_FORWARDED_FOR': request.META.get('HTTP_X_FORWARDED_FOR'),
        'X_FORWARDED_PROTO': request.META.get('HTTP_X_FORWARDED_PROTO'),
    })
```

然后通过浏览器或 curl 访问：

```bash
curl https://hc.example.com/debug-headers/
```

**预期结果**：
```json
{
  "is_secure": true,
  "scheme": "https",
  "X_FORWARDED_FOR": "真实客户端IP",
  "X_FORWARDED_PROTO": "https",
  "HOST": "hc.example.com"
}
```

**异常结果**：

| 异常 | 可能原因 | 解决方案 |
|------|----------|----------|
| `X_FORWARDED_PROTO` 不存在 | 反向代理未设置此 header | 在反向代理配置中添加 `proxy_set_header X-Forwarded-Proto $scheme;` |
| `X_FORWARDED_PROTO` 是 `http` | 反向代理配置错误，或者用户通过 http 访问 | 检查反向代理配置，确认用户是否通过 https 访问 |
| `is_secure` 是 `false` 但 `X_FORWARDED_PROTO` 是 `https` | `SECURE_PROXY_SSL_HEADER` 未配置 | 配置 `SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https` |

#### 步骤 7: 检查邮件链接

**目标**：验证邮件中的链接是否正确

**验证方法**：

1. **触发一封测试邮件**（如登录链接、告警邮件）
2. **检查邮件内容**中的链接

或者通过 Django shell 直接测试 URL 生成：

```bash
python manage.py shell -c "
from django.conf import settings
from hc.lib.urls import absolute_reverse, absolute_url

print('=== 配置检查 ===')
print(f'SITE_ROOT: {settings.SITE_ROOT}')
print(f'PING_ENDPOINT: {settings.PING_ENDPOINT}')

print()
print('=== URL 生成测试 ===')
print(f'absolute_url(\"/accounts/login/\"): {absolute_url(\"/accounts/login/\")}')
print(f'absolute_reverse(\"hc-login\"): {absolute_reverse(\"hc-login\")}')

print()
print('=== 协议检查 ===')
print(f'SITE_ROOT 使用的协议: {\"https\" if \"https://\" in settings.SITE_ROOT else \"http\"}')
"
```

**预期结果**：
- 所有生成的 URL 都应该使用正确的协议（https）
- 域名应该正确

**异常处理**：

如果 `SITE_ROOT=http://...` 但需要使用 https：

```bash
# 修正配置
SITE_ROOT=https://hc.example.com
```

#### 步骤 8: 检查 CSRF 和登录功能

**目标**：验证登录表单是否正常工作，CSRF 验证是否通过

**验证方法**：

1. **尝试登录**：
   - 打开登录页面
   - 输入用户名密码（或创建测试用户）
   - 点击登录

2. **观察结果**：
   - 成功登录？
   - 还是出现 403 Forbidden 错误？
   - 还是其他错误？

**常见问题及解决**：

| 症状 | 可能原因 | 解决方案 |
|------|----------|----------|
| 403 CSRF verification failed | `SECURE_PROXY_SSL_HEADER` 未配置或 `X-Forwarded-Proto` 错误 | 检查反向代理 header 和 Django 配置 |
| 登录成功但会话丢失 | Cookie 的 `Secure` 属性问题 | 检查 `SESSION_COOKIE_SECURE` 配置 |
| 页面可以访问但表单提交失败 | `Host` header 问题 | 检查反向代理是否正确传递 `Host` header |

### 第三阶段：反向代理配置验证

#### 步骤 9: 检查 Nginx 配置（如果使用 Nginx）

**关键配置项检查**：

```nginx
# 必须配置项检查：

# 1. 检查是否设置了 X-Forwarded-Proto
grep -r "X-Forwarded-Proto" /etc/nginx/

# 2. 检查是否设置了 Host header
grep -r "proxy_set_header Host" /etc/nginx/

# 3. 检查是否设置了 X-Forwarded-For
grep -r "X-Forwarded-For" /etc/nginx/
```

**推荐的 Nginx 配置**：

```nginx
location / {
    proxy_pass http://localhost:8000;
    
    # 关键：传递原始协议
    proxy_set_header X-Forwarded-Proto $scheme;
    
    # 关键：传递原始 Host
    proxy_set_header Host $host;
    
    # 传递客户端真实 IP
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    # 其他建议配置
    proxy_redirect off;
    proxy_http_version 1.1;
}
```

**常见错误**：

```nginx
# 错误：使用 $http_host 而不是 $host
proxy_set_header Host $http_host;  # 不推荐

# 错误：未设置 X-Forwarded-Proto
# （缺少这一行）

# 错误：直接传递用户的 X-Forwarded-Proto
# （应该使用 $scheme 而不是 $http_x_forwarded_proto）
```

#### 步骤 10: 检查其他反向代理

**HAProxy** 关键配置：

```haproxy
# 必须配置：根据 SSL 终止情况设置 X-Forwarded-Proto
http-request set-header X-Forwarded-Proto https if { ssl_fc }
http-request set-header X-Forwarded-Proto http unless { ssl_fc }
```

**Traefik** 关键配置：

Traefik 通常会自动设置这些 header，但需要确认：

```yaml
# 在动态配置或 Docker labels 中
traefik.http.middlewares.sslheader.headers.customRequestHeaders.X-Forwarded-Proto=https
```

### 第四阶段：综合验证

#### 步骤 11: 端到端测试

执行以下完整测试流程：

1. **访问首页**：
   ```bash
   curl -I https://hc.example.com/
   ```
   - 检查返回状态码（应该是 200 或 302 到登录页）
   - 检查 `Location` header（如果有重定向）是否使用正确的协议

2. **访问登录页**：
   - 用浏览器打开 `https://hc.example.com/accounts/login/`
   - 检查页面源代码中的表单 action
   - 应该是正确的 https URL

3. **测试登录**：
   - 尝试登录
   - 确认没有 403 错误
   - 确认登录后会话保持

4. **触发测试邮件**：
   - 使用"忘记密码"或其他会发送邮件的功能
   - 检查收到的邮件中的链接
   - 确认链接使用 https 协议

5. **测试 API**：
   ```bash
   curl -H "X-Api-Key: your-api-key" https://hc.example.com/api/v2/checks/
   ```
   - 检查响应中的 `ping_url`、`update_url` 等字段
   - 确认使用正确的协议

#### 步骤 12: 运行系统检查

```bash
# 运行 Django 的系统检查
python manage.py check

# 运行数据库迁移检查
python manage.py migrate --check

# 如果有自定义的健康检查
python manage.py check --deploy
```

### 快速排查 Checklist

| # | 检查项 | 命令/方法 | 预期结果 |
|---|--------|-----------|----------|
| 1 | `SITE_ROOT` 协议 | `echo $SITE_ROOT` | 以 `https://` 开头 |
| 2 | `SITE_ROOT` 域名 | 与用户访问域名对比 | 一致 |
| 3 | `SECURE_PROXY_SSL_HEADER` | `python manage.py shell -c "from django.conf import settings; print(getattr(settings, 'SECURE_PROXY_SSL_HEADER', 'NOT SET'))"` | `('HTTP_X_FORWARDED_PROTO', 'https')` |
| 4 | `ALLOWED_HOSTS` | `python manage.py shell -c "from django.conf import settings; print(settings.ALLOWED_HOSTS)"` | 包含访问域名 |
| 5 | 反向代理 `X-Forwarded-Proto` | 调试视图或日志 | `https` |
| 6 | 反向代理 `Host` | 调试视图或日志 | 正确的域名 |
| 7 | `request.is_secure()` | Django shell 测试 | `True` |
| 8 | 邮件链接协议 | 检查测试邮件 | `https://` |
| 9 | API 响应 URL 协议 | 调用 API 检查 | `https://` |
| 10 | 登录功能 | 实际登录测试 | 成功，无 403 错误 |

### 常见问题解决方案速查

#### 问题 1: 邮件中的链接是 http 而不是 https

**解决方案**：
```bash
# 设置正确的 SITE_ROOT
SITE_ROOT=https://hc.example.com
```

**注意**：这与 `SECURE_PROXY_SSL_HEADER` 无关！邮件链接直接使用 `SITE_ROOT` 配置值。

#### 问题 2: 登录表单返回 403 CSRF 错误

**解决方案**：

步骤 1: 配置 Django 信任反向代理 header
```bash
SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https
```

步骤 2: 确认反向代理正确设置 header

**Nginx**:
```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Host $host;
```

#### 问题 3: 可以访问页面但所有 POST 请求都失败

**可能原因**：`Host` header 不正确

**解决方案**：

在反向代理中正确传递 `Host` header：

**Nginx**:
```nginx
proxy_set_header Host $host;
# 不是 $http_host！
```

同时确认 `ALLOWED_HOSTS` 配置正确。

#### 问题 4: 登录成功后立即被登出

**可能原因**：Session Cookie 的 `Secure` 属性问题

**检查**：
```bash
# 检查 SESSION_COOKIE_SECURE 配置
python manage.py shell -c "
from django.conf import settings
print('SESSION_COOKIE_SECURE:', getattr(settings, 'SESSION_COOKIE_SECURE', 'DEFAULT'))
"
```

**解决方案**：
- 如果通过 https 访问，确保 `request.is_secure()` 返回 `True`
- 或者显式设置：`SESSION_COOKIE_SECURE=True`（如果始终使用 https）

#### 问题 5: WebAuthn（无密码登录）无法使用

**可能原因**：`request.is_secure()` 返回 `False`

WebAuthn 规范要求在安全上下文中使用（HTTPS 或 localhost）。

**解决方案**：
- 正确配置 `SECURE_PROXY_SSL_HEADER`
- 确认 `X-Forwarded-Proto` 正确传递

---

## 配置依赖关系图

```
                    ┌─────────────────────────────────┐
                    │      外部请求 (用户浏览器)        │
                    │    https://hc.example.com        │
                    └───────────────┬─────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────┐
                    │      反向代理 (Nginx/HAProxy)    │
                    │  TLS 终止，设置 header:         │
                    │  X-Forwarded-Proto: https      │
                    │  X-Forwarded-For: 真实IP        │
                    │  Host: hc.example.com           │
                    └───────────────┬─────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────┐
                    │      Django (uWSGI/Gunicorn)    │
                    │                                 │
                    │  读取配置:                        │
                    │  ├── SITE_ROOT (静态)            │
                    │  ├── PING_ENDPOINT (静态)        │
                    │  └── SECURE_PROXY_SSL_HEADER     │
                    │                                 │
                    │  运行时判断:                      │
                    │  ├── request.is_secure()?        │
                    │  ├── request.scheme?              │
                    │  └── CSRF 验证?                   │
                    └───────────────┬─────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
    │  静态 URL 生成  │    │  运行时行为    │    │  安全功能     │
    │               │    │               │    │               │
    │  使用配置值:   │    │  依赖 header:  │    │  依赖 header: │
    │  SITE_ROOT    │    │  X-Forwarded-* │    │  X-Forwarded-* │
    │  PING_ENDPOINT│    │               │    │               │
    │               │    │  - request.is_ │    │  - CSRF 验证  │
    │  影响:         │    │    secure()   │    │  - Session    │
    │  - 邮件链接    │    │  - 相对 URL   │    │  - WebAuthn   │
    │  - API 响应    │    │    协议继承    │    │  - IP 限流    │
    │  - OAuth 回调  │    │  - 重定向 URL  │    │               │
    │  - 页面展示    │    │               │    │               │
    └───────────────┘    └───────────────┘    └───────────────┘
```

### 配置与影响关系表

| 配置项 | 影响静态 URL 生成 | 影响运行时行为 | 影响安全功能 |
|--------|------------------|----------------|--------------|
| `SITE_ROOT` | ✅ 直接决定 | ❌ 无影响 | ❌ 无影响 |
| `PING_ENDPOINT` | ✅ 直接决定 | ❌ 无影响 | ❌ 无影响 |
| `SECURE_PROXY_SSL_HEADER` | ❌ 无影响 | ✅ 直接决定 | ✅ 直接决定 |
| 反向代理 `X-Forwarded-Proto` | ❌ 无影响 | ✅ 直接决定 | ✅ 直接决定 |
| 反向代理 `X-Forwarded-For` | ❌ 无影响 | ❌ 无影响 | ✅ 直接决定 |
| 反向代理 `Host` | ❌ 无影响 | ✅ 影响重定向 | ✅ 影响 CSRF |

---

## 快速配置检查清单

### 生产环境必须配置

```bash
# ==================== 基础配置 ====================
# 关键：必须使用 https 协议！
SITE_ROOT=https://your-domain.com
SITE_NAME=Your Monitoring

# 安全配置
SECRET_KEY=your-secure-secret-key-at-least-50-chars-long
DEBUG=False

# 注册控制（生产环境建议关闭公开注册）
REGISTRATION_OPEN=False

# ==================== 数据库配置 ====================
DB=postgres
DB_HOST=db
DB_NAME=hc
DB_USER=postgres
DB_PASSWORD=your-db-password

# ==================== 邮件配置 ====================
EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_HOST_USER=noreply@example.com
EMAIL_HOST_PASSWORD=your-smtp-password
DEFAULT_FROM_EMAIL=noreply@example.com
EMAIL_USE_TLS=True

# ==================== 反向代理与 HTTPS ====================
# 关键：告诉 Django 信任 X-Forwarded-Proto header
SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https

# 允许的主机名（通常自动从 SITE_ROOT 推导）
ALLOWED_HOSTS=your-domain.com

# ==================== 可选：其他功能 ====================
# WebAuthn（无密码登录）- 需要与 SITE_ROOT 域名一致
RP_ID=your-domain.com

# 如果使用独立的 ping 域名
# PING_ENDPOINT=https://ping.your-domain.com/
```

### 子路径部署

```bash
SITE_ROOT=https://your-domain.com/monitoring
# 其他配置同上...
```

系统会自动：
- `STATIC_URL=/monitoring/static/`
- `LOGIN_URL=/monitoring/accounts/login/`
- 所有 URL 路由添加 `/monitoring/` 前缀

### 独立 Ping 域名

```bash
SITE_ROOT=https://hc.example.com
PING_ENDPOINT=https://ping.example.com/
# 需要在反向代理中配置 ping.example.com 的路由
```

### Nginx 配置模板

```nginx
server {
    listen 443 ssl http2;
    server_name hc.example.com;

    # SSL 配置
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location / {
        proxy_pass http://localhost:8000;
        
        # 关键：协议传递
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 关键：Host 传递
        proxy_set_header Host $host;
        
        # 客户端 IP 传递
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # 其他优化
        proxy_redirect off;
        proxy_http_version 1.1;
        
        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # 静态文件（如果不使用 WhiteNoise）
    # location /static/ {
    #     alias /path/to/static-collected/;
    #     expires 30d;
    #     add_header Cache-Control "public, immutable";
    # }
}

# HTTP 到 HTTPS 重定向
server {
    listen 80;
    server_name hc.example.com;
    return 301 https://$server_name$request_uri;
}
```

---

## 参考文件

| 文件路径 | 说明 |
|----------|------|
| `hc/settings.py` | 核心配置文件 |
| `hc/lib/urls.py` | 绝对 URL 生成函数 |
| `hc/lib/emails.py` | 邮件发送逻辑 |
| `hc/api/models.py` | 数据模型与 URL 生成方法 |
| `hc/api/apps.py` | 系统检查与配置验证 |
| `hc/front/templatetags/hc_extras.py` | 模板标签与辅助函数 |
| `hc/urls.py` | URL 路由配置 |
| `templates/docs/self_hosted_docker.md` | 官方 Docker 部署文档 |
| `templates/docs/self_hosted_configuration.md` | 官方配置文档 |
