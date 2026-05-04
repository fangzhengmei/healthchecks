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
7. [配置依赖关系图](#配置依赖关系图)

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

**安全警告**: 如果反向代理没有设置 `X-Forwarded-For`，客户端可以伪造自己的 IP 地址。

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

## 配置依赖关系图

```
                    ┌─────────────────┐
                    │   SITE_ROOT     │
                    │ (环境变量)       │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
            ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ ALLOWED_HOSTS│ │  LOGIN_URL   │ │  STATIC_URL  │
    │ (自动推导)    │ │ (path 派生)   │ │ (path 派生)   │
    └──────────────┘ └──────────────┘ └──────────────┘
            │                │                │
            ▼                ▼                ▼
    ┌─────────────────────────────────────────────────┐
    │              PING_ENDPOINT (默认值)              │
    │         SITE_ROOT + "/ping/"                     │
    └─────────────────────────┬───────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────┐
    │              URL 路由前缀 (prefix)               │
    │  当 SITE_ROOT 包含路径时，所有 URL 自动添加此前缀  │
    └─────────────────────────┬───────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │ absolute_url │ │absolute_rever│ │  模板标签     │
    │   函数       │ │    se 函数    │ │ {% site_root %}│
    └──────────────┘ └──────────────┘ └──────────────┘
              │               │               │
              ▼               ▼               ▼
    ┌─────────────────────────────────────────────────┐
    │              生成的绝对 URL 应用场景              │
    │  - 邮件中的链接                                   │
    │  - API 响应中的 URL                               │
    │  - OAuth 回调 URL                                 │
    │  - Webhook 配置 URL                               │
    └─────────────────────────────────────────────────┘
```

---

## 快速配置检查清单

### 生产环境必须配置

```bash
# 基础配置
SITE_ROOT=https://your-domain.com
SITE_NAME=Your Monitoring
SECRET_KEY=your-secure-secret-key
DEBUG=False

# 数据库配置
DB=postgres
DB_HOST=db
DB_NAME=hc
DB_USER=postgres
DB_PASSWORD=your-db-password

# 邮件配置
EMAIL_HOST=smtp.example.com
EMAIL_PORT=587
EMAIL_HOST_USER=noreply@example.com
EMAIL_HOST_PASSWORD=your-smtp-password
DEFAULT_FROM_EMAIL=noreply@example.com

# 反向代理与 HTTPS
SECURE_PROXY_SSL_HEADER=HTTP_X_FORWARDED_PROTO,https
ALLOWED_HOSTS=your-domain.com

# 注册控制
REGISTRATION_OPEN=False  # 生产环境建议关闭公开注册
```

### 子路径部署

```bash
SITE_ROOT=https://your-domain.com/monitoring
# 其他配置同上...
```

### 独立 Ping 域名

```bash
SITE_ROOT=https://hc.example.com
PING_ENDPOINT=https://ping.example.com/
# 需要在反向代理中配置 ping.example.com 的路由
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
