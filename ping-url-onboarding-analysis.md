# Ping URL 生成与首次 Ping 引导分析

## 一、Ping URL 生成机制

### 1.1 Check 模型中的 URL 生成

在 `hc/api/models.py` 中，`Check` 模型提供了两种生成 ping URL 的方法：

#### `url()` 方法（第 241-257 行）

```python
def url(self) -> str | None:
    """Return check's ping url in user's preferred style."""
    if self.project_id and self.project.show_slugs:
        if not self.slug:
            return None
        # If ping_key is not set, use dummy placeholder
        key = self.project.ping_key or "{ping_key}"
        return settings.PING_ENDPOINT + key + "/" + self.slug

    return settings.PING_ENDPOINT + str(self.code)
```

**两种 URL 风格：**

1. **UUID 风格**（默认）：
   - 格式：`{PING_ENDPOINT}{uuid}`
   - 示例：`https://hc-ping.com/550e8400-e29b-41d4-a716-446655440000`
   - 使用 `check.code`（UUIDField，自动生成）

2. **Slug 风格**（需项目配置）：
   - 格式：`{PING_ENDPOINT}{ping_key}/{slug}`
   - 示例：`https://hc-ping.com/my-secret-key/daily-backup`
   - 条件：`project.show_slugs = True` 且 `check.slug` 已设置

#### `email()` 方法（第 270-285 行）

```python
def email(self) -> str | None:
    """Return check's ping email address in user's preferred style."""
    if self.project_id and self.project.show_slugs:
        if not self.slug:
            return None
        key = self.project.ping_key or "{ping_key}"
        return f"{key}+{self.slug}@{settings.PING_EMAIL_DOMAIN}"

    return f"{self.code}@{settings.PING_EMAIL_DOMAIN}"
```

**两种邮件格式：**

1. **UUID 风格**：`{uuid}@{PING_EMAIL_DOMAIN}`
2. **Slug 风格**：`{ping_key}+{slug}@{PING_EMAIL_DOMAIN}`

### 1.2 API 响应中的 URL

在 `to_dict()` 方法（第 413-465 行）中，API 响应会包含：

```python
if readonly:
    result["unique_key"] = self.unique_key
else:
    result["uuid"] = str(self.code)
    result["ping_url"] = settings.PING_ENDPOINT + str(self.code)
    # ... 其他 URL
```

**注意**：API 中的 `ping_url` 始终使用 UUID 格式，不考虑 slug 配置。

---

## 二、集成示例生成机制

### 2.1 Usage Examples 模态框

在 `templates/front/show_usage_modal.html` 中，系统为用户提供了多种编程语言和平台的集成示例。

#### 支持的集成类型

| Tab 名称 | 说明 | 包含的 Snippets |
|---------|------|----------------|
| Crontab | Cron 任务调度 | 智能生成的 cron 表达式 + curl |
| Bash | Shell 脚本 | `bash_curl.html`, `bash_wget.html` |
| Python | Python 脚本 | `python_urllib2.html`, `python_requests.html` |
| Ruby | Ruby 脚本 | `ruby.html` |
| Node.js | Node.js 脚本 | `node.html` |
| Go | Go 语言 | `go.html` |
| PHP | PHP 脚本 | `php.html` |
| C# | C# 程序 | `cs.html` |
| Browser | 浏览器 JavaScript | `browser.html` |
| PowerShell | Windows PowerShell | `powershell.html`, `powershell_inline.html` |
| Email | 邮件通知 | 直接显示邮件地址 |

#### 模板变量传递

```django
{% with ping_url=check.url %}
    <!-- 所有 snippet 都可以使用 {{ ping_url }} 变量 -->
    {% include "front/snippets/bash_curl.html" %}
{% endwith %}
```

### 2.2 智能 Cron 表达式生成

`guess_schedule` 过滤器（`hc/front/templatetags/hc_extras.py` 第 211-239 行）根据 check 的配置智能生成 cron 表达式：

```python
@register.filter
def guess_schedule(check: Check) -> str | None:
    if check.kind == "cron":
        return check.schedule

    v = int(check.timeout.total_seconds())

    # every minute
    if v == 60:
        return "* * * * *"

    # every hour
    if v == 3600:
        return "0 * * * *"

    # every day
    if v == 3600 * 24:
        return "0 0 * * *"

    # every X minutes, if 60 is divisible by X
    minutes, seconds = divmod(v, 60)
    if minutes in (2, 3, 4, 5, 6, 10, 12, 15, 20, 30) and seconds == 0:
        return f"*/{minutes} * * * *"

    # every X hours, if 24 is divisible by X
    hours, seconds = divmod(v, 3600)
    if hours in (2, 3, 4, 6, 8, 12) and seconds == 0:
        return f"0 */{hours} * * *"

    return None
```

**智能映射规则：**

| Timeout 值 | 生成的 Cron 表达式 | 说明 |
|------------|-------------------|------|
| 60 秒 | `* * * * *` | 每分钟 |
| 3600 秒 (1小时) | `0 * * * *` | 每小时整点 |
| 86400 秒 (1天) | `0 0 * * *` | 每天零点 |
| X 分钟（2,3,4,5,6,10,12,15,20,30） | `*/X * * * *` | 每 X 分钟 |
| X 小时（2,3,4,6,8,12） | `0 */X * * *` | 每 X 小时 |

如果无法匹配，则返回 `None`，模板中显示 `"* * * * *"` 并提示用户替换。

### 2.3 集成示例模板示例

#### Bash Curl 示例（`snippets/bash_curl.html`）

```django
<div class="highlight"><pre><span></span><span class="c1"># using curl (10 second timeout, retry up to 5 times):</span>
curl -m <span class="m">10</span> --retry <span class="m">5</span> {{ ping_url }}
</pre></div>
```

#### Python Requests 示例（`snippets/python_requests.html`）

```django
<div class="highlight"><pre><span></span><span class="c1"># Using the requests library:</span>
<span class="kn">import</span> <span class="nn">requests</span>

<span class="k">try</span><span class="p">:</span>
    <span class="n">requests</span><span class="o">.</span><span class="n">get</span><span class="p">(</span><span class="s2">&quot;{{ ping_url }}&quot;</span><span class="p">,</span> <span class="n">timeout</span><span class="o">=</span><span class="mi">10</span><span class="p">)</span>
<span class="k">except</span> <span class="n">requests</span><span class="o">.</span><span class="n">RequestException</span> <span class="k">as</span> <span class="n">e</span><span class="p">:</span>
    <span class="c1"># Log ping failure here...</span>
    <span class="nb">print</span><span class="p">(</span><span class="s2">&quot;Ping failed: </span><span class="si">%s</span><span class="s2">&quot;</span> <span class="o">%</span> <span class="n">e</span><span class="p">)</span>
</pre></div>
```

#### Crontab 完整示例

```django
{% with check|guess_schedule as schedule %}
<div class="highlight">
    <pre><span class="c1"># A sample crontab entry. Note the curl call appended after the command.</span>{% if not schedule %}
<span class="c1"># FIXME: replace "* * * * *" below with the correct cron expression!</span>{% endif %}
<span class="c1"># FIXME: replace "/your/command.sh" below with the correct command!</span>
{{ schedule|default:"* * * * *" }} /your/command.sh && curl -fsS -m 10 --retry 5 -o /dev/null {{ ping_url }}</pre>
</div>
{% endwith %}
```

---

## 三、首次 Ping 引导机制

### 3.1 状态显示与引导

在 `templates/front/log_status_text.html` 中，根据 check 的状态显示不同的引导信息：

```django
{% if status == "down" %}
    This check is down. Last ping was {{ check.last_ping|naturaltime }}.
{% elif status == "up" %}
    This check is up. Last ping was {{ check.last_ping|naturaltime }}.
{% elif status == "new" and check.n_pings %}
    This check is ready for pings.
{% elif status == "new" %}
    This check has never received a ping.  <!-- 首次引导关键提示 -->
{% endif %}
```

**状态流转与引导：**

| 状态 | 条件 | 显示文本 | 引导含义 |
|-----|------|---------|---------|
| `new` | `n_pings == 0` | "This check has never received a ping." | 明确提示用户需要发送首次 ping |
| `new` | `n_pings > 0` | "This check is ready for pings." | 已收到 ping，准备就绪 |
| `up` | - | "This check is up. Last ping was..." | 正常运行状态 |
| `down` | - | "This check is down. Last ping was..." | 需要注意的异常状态 |

### 3.2 详情页的 Ping Now 按钮

在 `templates/front/details.html` 第 144-158 行：

```django
{% if check.url %}
{% if project.show_slugs and not project.ping_key %}
    {% if rw %}
    <button
        data-toggle="modal"
        data-target="#no-ping-key-modal"
        class="btn btn-sm btn-default">Ping Now!</button>
    {% endif %}
{% else %}
<button
    id="ping-now"
    data-url="{{ check.url }}"
    class="btn btn-sm btn-default">Ping Now!</button>
{% endif %}
{% endif %}
```

**两种情况处理：**

1. **Slug 模式但无 ping_key**：
   - 点击按钮弹出 `no-ping-key-modal`，引导用户去项目设置生成 ping key

2. **正常情况**：
   - 按钮有 `id="ping-now"` 和 `data-url="{{ check.url }}"` 属性
   - 由前端 JavaScript 处理点击事件

### 3.3 前端 Ping 实现

在 `static/js/details.js` 第 55-66 行：

```javascript
$("#ping-now").click(function(e) {
    var button = this;
    $.post(this.dataset.url, function() {
        button.textContent = "Success!";
    });
});

$("#ping-now").mouseout(function(e) {
    setTimeout(function() {
        e.target.textContent = "Ping Now!";
    }, 300);
});
```

**功能说明：**
- 点击按钮时，通过 POST 请求直接访问 ping URL
- 成功后按钮文本变为 "Success!"
- 鼠标移出后 300ms 恢复原文本

### 3.4 Usage Examples 按钮

在 `templates/front/details.html` 第 113-118 行：

```django
{% if check.url %}
<button
    data-toggle="modal"
    data-target="#show-usage-modal"
    class="btn btn-sm btn-default">Usage Examples</button>
{% endif %}
```

点击后弹出包含各种集成示例的模态框，引导用户选择适合自己技术栈的集成方式。

### 3.5 新建 Check 后的提示

当复制 check 时（`is_copied` 为 true），在 `templates/front/details.html` 第 17-26 行：

```django
{% if is_copied %}
<div class="col-sm-12">
    <p id="new-check-alert" class="alert alert-success">
        <strong>Copy created!</strong>
        This is a brand new check, with details copied over from your existing check.
        You might now want to
        <a data-target="edit-name" href="#">update its name and tags</a>.
    </p>
</div>
{% endif %}
```

**JavaScript 处理**（`static/js/details.js` 第 33-36 行）：

```javascript
$("#new-check-alert a").click(function() {
    $("#" + this.dataset.target).click();
    return false;
});
```

点击链接会触发编辑名称的模态框，引导用户完善 check 信息。

### 3.6 URL 风格切换

在 `templates/front/details.html` 第 74-78 行：

```django
<div class="btn-group pull-right">
    <a href="?urls=uuid" class="btn btn-default btn-xs {% if not project.show_slugs %}active{% endif %}">uuid</a>
    <a href="?urls=slug" class="btn btn-default btn-xs {% if project.show_slugs %}active{% endif %}">slug</a>
</div>
```

允许用户在 UUID 风格和 Slug 风格之间切换，后端在 `hc/front/views.py` 的 `details` 函数中处理：

```python
if request.GET.get("urls") in ("uuid", "slug") and rw:
    check.project.show_slugs = request.GET["urls"] == "slug"
    check.project.save()
```

---

## 四、完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        新建 Check 流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 用户提交 Add Check 表单                                          │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ hc/front/views.py: add_check()                          │    │
│     │ - 创建 Check 对象，自动生成 code (UUID)                  │    │
│     │ - 设置 name, slug, tags, kind, timeout, schedule 等    │    │
│     │ - 调用 check.assign_all_channels() 分配所有通道         │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              ↓                                       │
│  2. 重定向到 checks 页面 或 直接进入 details 页面                   │
│                              ↓                                       │
│  3. 详情页显示                                                        │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ "How To Ping" 区域                                       │    │
│     │ - 显示 {{ check.url }} (HTTP ping URL)                 │    │
│     │ - 显示 {{ check.email }} (邮件 ping 地址)               │    │
│     │ - "Usage Examples" 按钮 → 弹出集成示例模态框            │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              ↓                                       │
│  4. 状态显示与引导                                                    │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 状态: new                                                 │    │
│     │ 文本: "This check has never received a ping."           │    │
│     │ 按钮: "Ping Now!" → 一键测试 ping                        │    │
│     └─────────────────────────────────────────────────────────┘    │
│                              ↓                                       │
│  5. 用户选择集成方式                                                  │
│     ┌─────────────┬─────────────┬─────────────┐                  │
│     │  Crontab    │   Bash      │   Python    │  ... 更多语言    │
│     │  (智能生成   │  (curl/     │  (requests/ │                  │
│     │   cron)      │   wget)     │   urllib2)  │                  │
│     └─────────────┴─────────────┴─────────────┘                  │
│                              ↓                                       │
│  6. 复制示例代码，集成到自己的脚本/任务中                            │
│                              ↓                                       │
│  7. 首次 ping 成功后                                                  │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 状态: new → up                                           │    │
│     │ 文本: "This check is up. Last ping was X ago."         │    │
│     │ 监控开始正常工作                                          │    │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键代码文件索引

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| Ping URL 生成 | `hc/api/models.py` | `Check.url()`, `Check.email()`, `Check.to_dict()` |
| 智能 Cron 生成 | `hc/front/templatetags/hc_extras.py` | `guess_schedule()` |
| 集成示例模板 | `templates/front/show_usage_modal.html` | 主模态框模板 |
| 语言特定示例 | `templates/front/snippets/*.html` | 各语言示例代码 |
| 状态显示引导 | `templates/front/log_status_text.html` | 状态文本模板 |
| 前端 Ping 功能 | `static/js/details.js` | `$("#ping-now").click()` |
| 详情页视图 | `hc/front/views.py` | `details()`, `add_check()` |
| Check 模型 | `hc/api/models.py` | `class Check` |

---

## 六、设计亮点总结

1. **双模式 URL 设计**：
   - UUID 模式：简单、安全、无需额外配置
   - Slug 模式：可读性好，适合需要在代码中硬编码的场景

2. **智能示例生成**：
   - `guess_schedule` 过滤器根据 timeout 自动推断合适的 cron 表达式
   - 减少用户手动配置的错误概率

3. **多语言/平台覆盖**：
   - 支持 11 种主流编程语言和平台
   - 每种语言提供最佳实践（超时设置、重试机制等）

4. **渐进式引导**：
   - 状态文本明确提示 "never received a ping"
   - "Ping Now!" 按钮提供一键测试
   - "Usage Examples" 提供详细集成指南
   - URL 风格切换让用户选择最适合的方式

5. **代码可复制性**：
   - 所有示例中的 `{{ ping_url }}` 会被替换为真实 URL
   - 用户可以直接复制使用，无需手动替换
