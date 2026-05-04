# Healthchecks 自托管部署配置分析

本文档分析 Healthchecks 自托管部署中，各类配置如何影响站点 URL、邮件链接、ping endpoint、静态资源和安全配置。

---

## 目录

1. [核心配置概述](#核心配置概述)
2. [SITE_ROOT - 站点根 URL 配置](#site_root---站点根-url-配置)
3. [PING_ENDPOINT - Ping 端点配置](#ping_endpoint---ping-端点配置)
4. [邮件配置与链接生成](#邮件配置与链接生成)
5. [静态资源配置](#静态资源配置)
6. [子路径部署下静态资源路径错配的完整故障链](#子路径部署下静态资源路径错配的完整故障链)
7. [安全配置、反向代理与 HTTPS Header](#安全配置反向代理与-https-header)
8. [HTTPS Header 配置错误的连锁影响](#https-header-配置错误的连锁影响)
9. [WebAuthn 与会话问题的真正原因分析](#webauthn-与会话问题的真正原因分析)
10. [可落地的排查步骤](#可落地的排查步骤)
11. [配置依赖关系图](#配置依赖关系图)
12. [快速配置检查清单](#快速配置检查清单)

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
| `RP_ID` | None | WebAuthn 依赖方 ID |
| `SESSION_COOKIE_SECURE` | Django 默认 | Session Cookie 的 Secure 属性 |

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

## 子路径部署下静态资源路径错配的完整故障链

### 核心概念：子路径部署的多层配置

子路径部署（如 `https://example.com/monitoring/`）涉及多层配置，任何一层出错都可能导致静态资源 404：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        子路径部署配置层                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 1: SITE_ROOT 配置                                             │
│  ├── STATIC_URL = /monitoring/static/  (从 SITE_ROOT 派生)         │
│  ├── URL prefix = "monitoring/"  (URL 路由前缀)                    │
│  └── LOGIN_URL = /monitoring/accounts/login/                        │
│                                                                      │
│  Layer 2: 反向代理配置 (Nginx/HAProxy)                               │
│  ├── 路径匹配: location /monitoring/                                 │
│  ├── 路径重写: proxy_pass 是否有末尾斜杠?                            │
│  └── Header 传递: Host, X-Forwarded-*                               │
│                                                                      │
│  Layer 3: CSS 压缩配置 (django-compressor)                          │
│  ├── CssRelativeFilter vs CssAbsoluteFilter                         │
│  └── CSS 中的相对路径如何处理?                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### CSS 中的相对路径分析

Healthchecks 的 CSS 文件使用相对路径引用其他资源：

**文件**: `static/css/icomoon.css:3`
```css
@font-face {
    src: url("../fonts/icomoon.woff2?12387");  /* 相对路径 */
}
```

**文件**: `static/css/channels.css:245`
```css
content: url("../img/github-white.png");  /* 相对路径 */
```

### CssRelativeFilter vs CssAbsoluteFilter 的区别

**文件**: `hc/settings.py:291-292`（代码注释）
```python
# Use CssRelativeFilter instead of CssAbsoluteFilter to fix
# icon font loading when serving Healthchecks from a subdirectory
```

| Filter | 处理方式 | 子路径部署问题 |
|--------|----------|----------------|
| `CssAbsoluteFilter` | 将 CSS 中的 `url("../fonts/...")` 转换为绝对路径 `url("/static/fonts/...")` | 丢失子路径前缀 `/monitoring/` |
| `CssRelativeFilter` | 保持相对路径不变 | 浏览器根据 CSS 文件位置正确解析 |

**问题演示**：

```
使用 CssAbsoluteFilter（旧配置，有问题）：
────────────────────────────────────────

原始 CSS:
  @font-face {
      src: url("../fonts/icomoon.woff2");
  }

CssAbsoluteFilter 处理后:
  @font-face {
      src: url("/static/fonts/icomoon.woff2");  ← 绝对路径，但缺少子路径前缀
  }

子路径部署场景:
  SITE_ROOT = https://example.com/monitoring
  STATIC_URL = /monitoring/static/
  
  浏览器请求: https://example.com/static/fonts/icomoon.woff2
  实际文件位置: https://example.com/monitoring/static/fonts/icomoon.woff2
  结果: 404 错误，图标字体无法加载

────────────────────────────────────────

使用 CssRelativeFilter（当前配置，修复后）：
────────────────────────────────────────

原始 CSS:
  @font-face {
      src: url("../fonts/icomoon.woff2");
  }

CssRelativeFilter 处理后:
  @font-face {
      src: url("../fonts/icomoon.woff2");  ← 保持相对路径不变
  }

子路径部署场景:
  CSS 文件 URL: https://example.com/monitoring/static/css/icomoon.css
  相对路径: ../fonts/icomoon.woff2
  浏览器解析为: https://example.com/monitoring/static/fonts/icomoon.woff2 ✓
  结果: 正确加载
```

### 完整故障链分析

#### 故障场景 1: SITE_ROOT 缺少子路径前缀

**配置错误**：
```bash
# 错误配置
SITE_ROOT=https://example.com  # 应该是 https://example.com/monitoring

# 实际访问
https://example.com/monitoring/
```

**派生配置**：
| 配置项 | 期望值 | 实际值（错误） |
|--------|--------|----------------|
| `STATIC_URL` | `/monitoring/static/` | `/static/` |
| URL prefix | `"monitoring/"` | `""` |
| `LOGIN_URL` | `/monitoring/accounts/login/` | `/accounts/login/` |

**故障链**：

```
配置层:
  SITE_ROOT = https://example.com (缺少 /monitoring)
       ↓
  STATIC_URL = /static/ (错误，应该是 /monitoring/static/)
       ↓
  URL prefix = "" (错误，应该是 "monitoring/")

页面生成:
  <link href="/static/css/base.css" rel="stylesheet">  ← 缺少 /monitoring 前缀
       ↓
浏览器请求:
  https://example.com/static/css/base.css
       ↓
反向代理 (Nginx):
  location /monitoring/ { ... }  ← 只有 /monitoring/ 的路由
       ↓
结果:
  404 Not Found，样式无法加载
```

#### 故障场景 2: 反向代理路径重写错误（Nginx 配置问题）

**配置正确**：
```bash
SITE_ROOT=https://example.com/monitoring  # 正确
```

**Nginx 配置错误**：
```nginx
# 错误配置：proxy_pass 末尾的斜杠会导致路径重写
location /monitoring/ {
    proxy_pass http://localhost:8000/;  # 注意末尾的斜杠！
}
```

**故障链**：

```
浏览器请求:
  https://example.com/monitoring/static/css/base.css
       ↓
Nginx 处理:
  location /monitoring/ 匹配成功
  proxy_pass http://localhost:8000/;  ← 末尾斜杠导致路径重写
       ↓
转发到后端:
  http://localhost:8000/static/css/base.css  ← 丢失了 /monitoring/ 前缀！
       ↓
Django 收到请求:
  PATH_INFO = /static/css/base.css
       ↓
Django URL 路由:
  URL prefix = "monitoring/"  (从 SITE_ROOT 派生)
  实际查找路径 = /monitoring/ + /static/css/base.css
                 = /monitoring/static/css/base.css
       ↓
但请求路径是 /static/css/base.css，不匹配！
       ↓
结果:
  404 Not Found
```

**正确的 Nginx 配置**：
```nginx
# 正确配置 1：不使用末尾斜杠，保持路径不变
location /monitoring/ {
    proxy_pass http://localhost:8000;  # 没有末尾斜杠
}

# 转发结果：
# 浏览器请求: /monitoring/static/css/base.css
# 转发到后端: /monitoring/static/css/base.css  ✓ 保持完整路径

# 或者：

# 正确配置 2：显式重写路径
location /monitoring/ {
    proxy_pass http://localhost:8000/monitoring/;
}
```

#### 故障场景 3: 子路径 + CSS 相对路径问题（已修复）

**历史问题**（使用 `CssAbsoluteFilter` 时）：

```
配置:
  SITE_ROOT = https://example.com/monitoring
  STATIC_URL = /monitoring/static/

原始 CSS (icomoon.css):
  @font-face {
      src: url("../fonts/icomoon.woff2");
  }

CssAbsoluteFilter 处理后:
  @font-face {
      src: url("/static/fonts/icomoon.woff2");  ← 问题：缺少 /monitoring 前缀
  }

浏览器解析:
  CSS 文件 URL: /monitoring/static/css/icomoon.css
  字体 URL:    /static/fonts/icomoon.woff2  (绝对路径)
       ↓
  实际请求: https://example.com/static/fonts/icomoon.woff2
  实际位置: https://example.com/monitoring/static/fonts/icomoon.woff2
       ↓
结果: 404 错误
```

**当前解决方案**（使用 `CssRelativeFilter`）：

```
CssRelativeFilter 处理后:
  @font-face {
      src: url("../fonts/icomoon.woff2");  ← 保持相对路径
  }

浏览器解析:
  CSS 文件 URL: /monitoring/static/css/icomoon.css
  相对路径: ../fonts/icomoon.woff2
  解析结果: /monitoring/static/fonts/icomoon.woff2  ✓
       ↓
结果: 正确加载
```

### 子路径部署静态资源验证步骤

#### 验证步骤 1: 检查配置派生

```bash
# 检查 STATIC_URL 和 URL 前缀
python manage.py shell -c "
from django.conf import settings
from urllib.parse import urlparse

site_root_parts = urlparse(settings.SITE_ROOT)
print('=== 配置检查 ===')
print(f'SITE_ROOT: {settings.SITE_ROOT}')
print(f'SITE_ROOT path: {site_root_parts.path}')
print(f'STATIC_URL: {settings.STATIC_URL}')
print(f'LOGIN_URL: {settings.LOGIN_URL}')
print()
print('=== 预期值对比 ===')
expected_static_url = f'{site_root_parts.path}/static/'
print(f'预期 STATIC_URL: {expected_static_url}')
print(f'实际 STATIC_URL: {settings.STATIC_URL}')
print(f'匹配: {settings.STATIC_URL == expected_static_url}')
"
```

**预期结果**：
```
=== 配置检查 ===
SITE_ROOT: https://example.com/monitoring
SITE_ROOT path: /monitoring
STATIC_URL: /monitoring/static/
LOGIN_URL: /monitoring/accounts/login/

=== 预期值对比 ===
预期 STATIC_URL: /monitoring/static/
实际 STATIC_URL: /monitoring/static/
匹配: True
```

#### 验证步骤 2: 检查页面中的静态资源引用

```bash
# 方法 1: 使用 curl 检查页面源代码
curl -s https://example.com/monitoring/ | grep -E 'href="/monitoring/static|src="/monitoring/static'

# 方法 2: 检查是否缺少子路径前缀（问题迹象）
curl -s https://example.com/monitoring/ | grep -E 'href="/static[^/]|src="/static[^/]'
```

**预期结果**（正确）：
```
<link href="/monitoring/static/CACHE/css/base.abc123.css" rel="stylesheet">
<script src="/monitoring/static/CACHE/js/base.def456.js"></script>
```

**异常结果**（错误，缺少前缀）：
```
<link href="/static/CACHE/css/base.abc123.css" rel="stylesheet">  ← 缺少 /monitoring
```

#### 验证步骤 3: 直接测试静态资源访问

```bash
# 测试 CSS 文件访问
curl -I https://example.com/monitoring/static/CACHE/css/base.abc123.css

# 测试字体文件访问
curl -I https://example.com/monitoring/static/fonts/icomoon.woff2

# 测试图片访问
curl -I https://example.com/monitoring/static/img/logo.png
```

**预期结果**：
```
HTTP/2 200
content-type: text/css
```

**异常结果**：
```
HTTP/2 404  ← 路径错误
```

#### 验证步骤 4: 检查 Nginx 配置（如果使用 Nginx）

```bash
# 检查 Nginx 配置语法
nginx -t

# 检查 location 配置
grep -A 5 "location /monitoring" /etc/nginx/sites-available/*.conf

# 特别检查 proxy_pass 是否有末尾斜杠
grep "proxy_pass" /etc/nginx/sites-available/*.conf
```

**关键点检查**：

| proxy_pass 配置 | 路径重写行为 | 是否适合子路径部署 |
|-----------------|--------------|-------------------|
| `proxy_pass http://localhost:8000;` | 保持路径不变 | ✅ 适合 |
| `proxy_pass http://localhost:8000/;` | 去掉匹配的前缀 | ❌ 可能导致问题 |
| `proxy_pass http://localhost:8000/monitoring/;` | 显式重写 | ✅ 适合 |

#### 验证步骤 5: 检查 CSS 中的相对路径

```bash
# 查找压缩后的 CSS 文件
ls static-collected/CACHE/css/

# 检查 CSS 文件内容，确认是否使用相对路径
grep -r "url(" static-collected/CACHE/css/
```

**预期结果**（使用 `CssRelativeFilter`）：
```css
url("../fonts/icomoon.woff2")  # 相对路径 ✓
```

**异常结果**（使用 `CssAbsoluteFilter`）：
```css
url("/static/fonts/icomoon.woff2")  # 绝对路径，可能缺少前缀 ✗
```

#### 验证步骤 6: 端到端测试

```bash
# 1. 访问首页
curl -s -o /dev/null -w "%{http_code} %{url_effective}\n" https://example.com/monitoring/

# 2. 检查登录页面
curl -s https://example.com/monitoring/accounts/login/ | grep -c "Login"

# 3. 检查页面中的静态资源引用是否正确
curl -s https://example.com/monitoring/accounts/login/ | grep -o 'href="[^"]*\.css"' | head -3
```

### 子路径部署配置最佳实践

#### 完整配置示例

**环境变量**：
```bash
# 关键点：SITE_ROOT 必须包含完整的子路径
SITE_ROOT=https://example.com/monitoring
SITE_NAME=My Monitoring

# 其他配置...
```

**Nginx 配置**：
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL 配置...
    ssl_certificate /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    # 子路径配置
    location /monitoring/ {
        # 关键点 1: proxy_pass 不带末尾斜杠，保持路径不变
        proxy_pass http://localhost:8000;
        
        # 关键点 2: 传递必要的 header
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 其他配置...
        proxy_redirect off;
        proxy_http_version 1.1;
    }

    # 可选：直接提供静态文件（如果不使用 WhiteNoise）
    # location /monitoring/static/ {
    #     alias /path/to/static-collected/;
    #     expires 30d;
    #     add_header Cache-Control "public, immutable";
    # }
}

# HTTP 到 HTTPS 重定向
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### 子路径部署故障排查速查表

| 症状 | 可能原因 | 排查方法 | 解决方案 |
|------|----------|----------|----------|
| 页面样式丢失，控制台显示 CSS 404 | `SITE_ROOT` 缺少子路径前缀 | 检查 `STATIC_URL` 配置 | 修正 `SITE_ROOT` 配置 |
| 直接访问静态资源 URL 返回 404 | Nginx `proxy_pass` 路径重写错误 | 检查 Nginx 配置中的 `proxy_pass` | 移除 `proxy_pass` 末尾的斜杠，或显式重写路径 |
| 图标字体不显示，控制台显示字体 404 | CSS 中的相对路径被错误转换 | 检查 `COMPRESS_FILTERS` 配置 | 确保使用 `CssRelativeFilter` |
| 页面可以访问但所有链接缺少前缀 | URL 路由前缀错误 | 检查 `urls.py` 中的 prefix 变量 | 修正 `SITE_ROOT` 配置 |
| 登录后重定向到错误路径 | `LOGIN_URL` 或 `LOGIN_REDIRECT_URL` 错误 | 检查 `LOGIN_URL` 配置 | 确认 `SITE_ROOT` 包含正确的子路径 |

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
| **CSRF 验证** | Django 的 CSRF 保护在某些情况下会检查协议，可能导致 403 错误 | 🔴 高 |
| **uWSGI 行为** | 根据官方文档，uWSGI 依赖 `X-Forwarded-Proto` 来判断请求安全性，配置错误会导致 CSRF 验证失败 | 🔴 高 |
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
│       │         └──► CSRF 验证可能失败 (403 错误)             │
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
| 登录限流 | 🟢 无影响 | 🟢 无影响 | 🔴 可被绕过 |
| 安全审计 | 🟢 无影响 | 🟢 无影响 | 🔴 IP 不可信 |

---

## WebAuthn 与会话问题的真正原因分析

### 关键纠正：之前的归因不准确

在之前的分析中，我错误地将 WebAuthn 和会话问题直接归因于 `SECURE_PROXY_SSL_HEADER` 配置错误和 `request.is_secure()` 返回值。

**实际情况更为复杂**：

| 功能 | 真正的依赖 | `request.is_secure()` 的角色 |
|------|-----------|------------------------------|
| **WebAuthn** | `RP_ID` 配置 + 浏览器 HTTPS 要求 | 间接因素（可能影响重定向） |
| **Session Cookie** | `SESSION_COOKIE_SECURE` 配置 | 间接因素（Django 某些中间件可能依赖） |

### WebAuthn（无密码登录）的完整分析

#### 1. WebAuthn 的核心配置：RP_ID

**文件**: `hc/settings.py:319`

```python
# WebAuthn
RP_ID = os.getenv("RP_ID")
```

**文件**: `hc/lib/webauthn.py:18-20`

```python
def __init__(self, rp_id: str, credentials: Iterable[bytes]):
    rp = PublicKeyCredentialRpEntity(id=rp_id, name="healthchecks")
    self.server = Fido2Server(rp)
```

**官方文档**（`templates/docs/self_hosted_configuration.md`）：

> The [Relying Party identifier](https://www.w3.org/TR/webauthn-2/#relying-party-identifier), required by the WebAuthn second-factor authentication feature.
> 
> Set its value to your site's domain without scheme and without port. For example, if your site runs on `https://my-hc.example.org`, set `RP_ID` to `my-hc.example.org`.

#### 2. WebAuthn 的真正依赖

| 依赖项 | 说明 | 代码位置 |
|--------|------|----------|
| **`RP_ID` 配置** | 必须设置为站点域名（不带协议） | `hc/settings.py:319` |
| **浏览器 HTTPS 要求** | WebAuthn 规范要求安全上下文 | 浏览器/WebAuthn API 限制 |
| **域名匹配** | `RP_ID` 必须与实际访问的域名匹配 | `hc/lib/webauthn.py` |

**代码中没有直接检查 `request.is_secure()`**：

查看 `hc/lib/webauthn.py`、`hc/accounts/views.py` 中的 WebAuthn 相关代码，没有发现任何对 `request.is_secure()` 的直接检查。

#### 3. WebAuthn 无法使用的故障链

**故障场景 1: `RP_ID` 未配置或配置错误**

```
配置：
  RP_ID = None （未设置）
  或
  RP_ID = wrong-domain.com （与实际域名不匹配）

代码检查：
  hc/accounts/views.py:721
  if not settings.RP_ID:
      # WebAuthn 功能被禁用

故障链：
1. 用户访问 /accounts/two_factor/webauthn/
2. Django 检查 settings.RP_ID
3. 如果未配置，功能不可用
4. 结果：WebAuthn 无法使用
```

**故障场景 2: 浏览器 HTTPS 要求（最常见）**

```
配置：
  RP_ID = hc.example.com （正确）
  SITE_ROOT = https://hc.example.com （正确）

但实际访问：
  http://hc.example.com （通过 HTTP 访问）
  或
  https://hc.example.com 但浏览器不认为是安全上下文

故障链：
1. 用户尝试添加 WebAuthn 密钥
2. 浏览器调用 navigator.credentials.create()
3. WebAuthn API 检查安全上下文
4. 如果不是 HTTPS（或 localhost），浏览器抛出错误
5. 结果：WebAuthn 无法使用

官方文档引用：
> Note that WebAuthn requires HTTPS, even if running on localhost.
```

**故障场景 3: `SECURE_PROXY_SSL_HEADER` 配置错误（间接因素）**

```
配置：
  SITE_ROOT = https://hc.example.com
  RP_ID = hc.example.com
  SECURE_PROXY_SSL_HEADER = 未配置

故障链：
1. 用户通过 HTTPS 访问 https://hc.example.com
2. 反向代理终止 TLS，通过 HTTP 转发到 Django
3. 反向代理设置 X-Forwarded-Proto: https
4. 但 SECURE_PROXY_SSL_HEADER 未配置，Django 不信任这个 header
5. request.is_secure() 返回 False
6. Django 的某些中间件可能根据这个值改变行为
7. 例如：SecurityMiddleware 可能不会设置某些安全 header
8. 或者重定向逻辑可能使用 http:// 而不是 https://
9. 结果：可能导致某些功能异常，但不是 WebAuthn 无法使用的直接原因
```

#### 4. WebAuthn 故障排查步骤

```bash
# 步骤 1: 检查 RP_ID 配置
python manage.py shell -c "
from django.conf import settings
print('=== WebAuthn 配置检查 ===')
print(f'RP_ID: {settings.RP_ID}')
print(f'SITE_ROOT: {settings.SITE_ROOT}')
print()

# 检查 RP_ID 是否与 SITE_ROOT 域名匹配
from urllib.parse import urlparse
site_root_domain = urlparse(settings.SITE_ROOT).netloc.split(':')[0]
print(f'SITE_ROOT 域名: {site_root_domain}')
print(f'RP_ID 匹配: {settings.RP_ID == site_root_domain if settings.RP_ID else \"未配置\"}')
"
```

```bash
# 步骤 2: 检查访问协议
# 用浏览器访问，检查地址栏是否显示 https://

# 或者用 curl 检查
curl -I https://hc.example.com/accounts/two_factor/webauthn/

# 检查是否有 HSTS 头
curl -sI https://hc.example.com/ | grep -i strict-transport-security
```

```bash
# 步骤 3: 检查浏览器控制台错误
# 打开浏览器开发者工具，查看 Console 中是否有 WebAuthn 相关错误
# 常见错误：
# - "The operation is insecure." (非 HTTPS 环境)
# - "The relying party ID is not a registrable domain suffix of, nor equal to the current domain." (RP_ID 不匹配)
```

### Session Cookie 问题的完整分析

#### 1. Session Cookie 的核心配置

**文件**: `hc/accounts/views.py:151-158`

```python
def _set_autologin_cookie(response: HttpResponse) -> None:
    response.set_cookie(
        "auto-login",
        "1",
        max_age=300,
        httponly=True,
        samesite="Lax",
        secure=bool(settings.SESSION_COOKIE_SECURE),  # 关键点
    )
```

**Django 的标准行为**：

| 配置项 | 作用 | 默认值 |
|--------|------|--------|
| `SESSION_COOKIE_SECURE` | Cookie 是否只在 HTTPS 连接中发送 | `False` (Django 默认) |
| `SESSION_COOKIE_HTTPONLY` | Cookie 是否只通过 HTTP 访问（防止 XSS） | `True` |
| `SESSION_COOKIE_SAMESITE` | SameSite 属性 | `Lax` |

#### 2. Session Cookie 问题的真正依赖

| 依赖项 | 说明 | 代码位置 |
|--------|------|----------|
| **`SESSION_COOKIE_SECURE`** | 决定 Cookie 的 `Secure` 属性 | `hc/accounts/views.py:157` |
| **`request.is_secure()`** | Django 某些中间件可能依赖 | Django SecurityMiddleware |

**关键区别**：
- `secure=bool(settings.SESSION_COOKIE_SECURE)` 直接使用配置值
- 不依赖 `request.is_secure()` 的返回值
- 但 Django 的 `SecurityMiddleware` 可能根据 `request.is_secure()` 改变某些行为

#### 3. Session Cookie 无法工作的故障链

**故障场景 1: `SESSION_COOKIE_SECURE=True` 但通过 HTTP 访问**

```
配置：
  SESSION_COOKIE_SECURE = True
  SECURE_PROXY_SSL_HEADER = HTTP_X_FORWARDED_PROTO,https

访问方式：
  http://hc.example.com （通过 HTTP 访问）

故障链：
1. 用户登录，Django 设置 session cookie
2. Cookie 带有 Secure 属性（因为 SESSION_COOKIE_SECURE=True）
3. 浏览器收到 cookie
4. 浏览器检查：Secure 属性的 cookie 只在 HTTPS 连接中发送
5. 用户后续请求通过 HTTP 发送
6. 浏览器不发送带有 Secure 属性的 cookie
7. 结果：用户看起来"登录成功但立即被登出"
```

**故障场景 2: `SECURE_PROXY_SSL_HEADER` 配置错误导致重定向问题**

```
配置：
  SITE_ROOT = https://hc.example.com
  SESSION_COOKIE_SECURE = True
  SECURE_PROXY_SSL_HEADER = 未配置

访问方式：
  https://hc.example.com （通过 HTTPS 访问）

故障链：
1. 用户通过 HTTPS 访问登录页面
2. 反向代理终止 TLS，通过 HTTP 转发
3. 反向代理设置 X-Forwarded-Proto: https
4. 但 SECURE_PROXY_SSL_HEADER 未配置
5. request.is_secure() 返回 False
6. Django 的某些逻辑可能认为请求是 HTTP
7. 例如：某些重定向可能使用 http:// 而不是 https://
8. 或者 SecurityMiddleware 不会设置某些安全 header
9. 结果：可能导致会话异常，但这是间接影响
```

**故障场景 3: 混合配置问题**

```
配置：
  SITE_ROOT = http://hc.example.com （错误，应该是 https）
  SESSION_COOKIE_SECURE = True
  SECURE_PROXY_SSL_HEADER = HTTP_X_FORWARDED_PROTO,https

故障链：
1. 页面中的链接使用 http://（因为 SITE_ROOT=http://）
2. 用户点击链接，浏览器请求 http://hc.example.com
3. 如果有 HSTS 或反向代理重定向，可能跳转到 https
4. 但 cookie 是为 https 设置的
5. 或者某些请求通过 http 发送
6. 结果：会话不稳定
```

#### 4. Session Cookie 故障排查步骤

```bash
# 步骤 1: 检查 SESSION_COOKIE_SECURE 配置
python manage.py shell -c "
from django.conf import settings
print('=== Session Cookie 配置检查 ===')
print(f'SESSION_COOKIE_SECURE: {getattr(settings, \"SESSION_COOKIE_SECURE\", \"DEFAULT (False)\")}')
print(f'SESSION_COOKIE_HTTPONLY: {getattr(settings, \"SESSION_COOKIE_HTTPONLY\", \"DEFAULT\")}')
print(f'SESSION_COOKIE_SAMESITE: {getattr(settings, \"SESSION_COOKIE_SAMESITE\", \"DEFAULT\")}')
print()
print(f'SITE_ROOT: {settings.SITE_ROOT}')
print(f'SECURE_PROXY_SSL_HEADER: {getattr(settings, \"SECURE_PROXY_SSL_HEADER\", \"NOT SET\")}')
"
```

```bash
# 步骤 2: 检查实际访问协议
# 用浏览器开发者工具检查：
# 1. Network 标签页，查看请求的 Protocol 列
# 2. Application 标签页，查看 Cookies 的 Secure 属性

# 或者用 curl 检查
curl -I https://hc.example.com/accounts/login/ -c cookies.txt
cat cookies.txt
# 检查是否有 Secure 属性
```

```bash
# 步骤 3: 测试登录流程
# 1. 清除所有浏览器 cookie
# 2. 访问登录页面
# 3. 登录
# 4. 检查浏览器是否收到 sessionid cookie
# 5. 刷新页面，检查是否保持登录状态
```

### WebAuthn 和 Session Cookie 问题对比

| 问题 | 主要依赖 | `SECURE_PROXY_SSL_HEADER` 的角色 | `request.is_secure()` 的角色 |
|------|----------|-----------------------------------|------------------------------|
| **WebAuthn 无法使用** | `RP_ID` 配置 + 浏览器 HTTPS 要求 | 间接因素（可能影响重定向） | 间接因素 |
| **Session Cookie 问题** | `SESSION_COOKIE_SECURE` 配置 | 间接因素 | 间接因素 |

### 快速诊断表

| 症状 | 最可能的原因 | 验证方法 |
|------|-------------|----------|
| WebAuthn 页面显示功能不可用 | `RP_ID` 未配置 | 检查 `settings.RP_ID` |
| 浏览器控制台显示 "The operation is insecure" | 非 HTTPS 环境 | 检查地址栏是否为 https:// |
| 浏览器控制台显示 RP_ID 不匹配 | `RP_ID` 配置错误 | 对比 `RP_ID` 和实际域名 |
| 登录成功后立即被登出 | `SESSION_COOKIE_SECURE=True` 但通过 HTTP 访问 | 检查 Cookie 的 Secure 属性 |
| 会话在 http 和 https 之间不稳定 | `SITE_ROOT` 协议错误 | 检查 `SITE_ROOT` 使用的协议 |

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

#### 步骤 4: 检查 RP_ID 配置（如果使用 WebAuthn）

**检查项**：
- `RP_ID` 是否配置？
- `RP_ID` 是否与 `SITE_ROOT` 的域名匹配？

**验证方法**：

```bash
# 通过 Django shell 检查
python manage.py shell -c "
from django.conf import settings
from urllib.parse import urlparse

print('=== WebAuthn 配置检查 ===')
print(f'RP_ID: {settings.RP_ID}')
print(f'SITE_ROOT: {settings.SITE_ROOT}')

# 提取 SITE_ROOT 的域名
if settings.SITE_ROOT:
    parsed = urlparse(settings.SITE_ROOT)
    domain = parsed.netloc.split(':')[0]  # 去掉端口
    print(f'SITE_ROOT 域名: {domain}')
    
    if settings.RP_ID: