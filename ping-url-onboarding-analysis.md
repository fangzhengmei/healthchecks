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
   - **注意**：如果 `ping_key` 未设置，会使用占位符 `{ping_key}`

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

## 二、三类集成示例详细分析

### 2.1 Cron 集成示例

#### 示例来源

Cron 集成示例位于 `templates/front/show_usage_modal.html` 的第一个 Tab（Crontab），是 "Usage Examples" 模态框的默认显示内容。

**模板结构**（第 49-63 行）：

```django
{% with ping_url=check.url %}
<div role="tabpanel" class="tab-pane active" id="crontab">
    {% with check|guess_schedule as schedule %}
    <div class="highlight">
        <pre><span class="c1"># A sample crontab entry. Note the curl call appended after the command.</span>{% if not schedule %}
<span class="c1"># FIXME: replace "* * * * *" below with the correct cron expression!</span>{% endif %}
<span class="c1"># FIXME: replace "/your/command.sh" below with the correct command!</span>
{{ schedule|default:"* * * * *" }} /your/command.sh && curl -fsS -m 10 --retry 5 -o /dev/null {{ ping_url }}</pre>
    </div>
    <!-- 单独的 curl 命令片段，方便复制 -->
    <div class="highlight">
        <pre><span class="c1"># Here's the part you need to append, provided here separately for easy copy/pasting:</span>
&& curl -fsS -m 10 --retry 5 -o /dev/null {{ ping_url }}</pre>
    </div>
    {% endwith %}
</div>
{% endwith %}
```

#### 变量替换机制

Cron 示例使用两种变量替换方式：

| 变量 | 来源 | 替换方式 |
|-----|------|---------|
| `{{ ping_url }}` | `check.url()` 方法的返回值 | Django 模板变量，在服务端渲染时替换为真实 URL |
| `{{ check|guess_schedule }}` | `guess_schedule` 模板过滤器 | 根据 check 的配置智能生成 cron 表达式 |

**智能 Cron 表达式生成**（`hc/front/templatetags/hc_extras.py` 第 211-239 行）：

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

如果无法匹配，则返回 `None`，模板中使用 `default:"* * * * *"` 并显示 `# FIXME` 提示用户替换。

#### 用户复制机制

Cron 示例的代码块支持点击复制，实现机制如下：

**前端复制实现**（`static/js/details.js` 第 162-171 行）：

```javascript
$(".click-to-copy").tooltip({ container: "body", title: "Click to copy" });
$(".click-to-copy").click(function (e) {
    if (window.getSelection().toString()) {
        // do nothing, selection not empty
        return;
    }

    navigator.clipboard.writeText(this.textContent);
    $(".tooltip-inner").text("Copied!");
});
```

**复制流程**：

1. 用户点击带有 `class="click-to-copy"` 的 `<code>` 或 `<pre>` 元素
2. JavaScript 检查是否有文本被选中（避免与手动选择复制冲突）
3. 使用 `navigator.clipboard.writeText()` 将元素的 `textContent` 写入剪贴板
4. 更新 tooltip 显示 "Copied!" 反馈

**注意**：Crontab Tab 中的代码块**没有** `click-to-copy` class，详情页 "How To Ping" 区域的 ping URL 和 email 才有这个 class。

---

### 2.2 CI 集成示例

#### 示例来源

CI 集成示例主要通过**文档页面**提供，而不是 "Usage Examples" 模态框。

**GitHub Actions 文档**：`templates/docs/github_actions.md`

```markdown
# GitHub Actions

You can augment your GitHub Actions workflows to report success and
failure to SITE_NAME:

```yaml
name: Hourly Housekeeping
on:
  schedule:
    - cron: '15 * * * *'
jobs:
  Main-Job:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Running housekeeping tasks..."
  Ping-Success:
    runs-on: ubuntu-latest
    needs: [Main-Job]
    steps:
      - run: curl -m 10 --retry 5 ${{ secrets.ping_url }}
  Ping-Failure:
    runs-on: ubuntu-latest
    if: ${{ failure() }}
    needs: [Main-Job]
    steps:
      - run: curl -m 10 --retry 5 ${{ secrets.ping_url }}/fail
```
```

#### 变量替换机制

CI 文档中的变量替换通过 `_replace_placeholders` 函数（`hc/front/views.py` 第 445-472 行）实现：

```python
def _replace_placeholders(doc: str, html: str) -> str:
    if doc.startswith("self_hosted"):
        return html

    limit = settings.PING_BODY_LIMIT or 100
    if limit % 1000 == 0:
        limit_fmt = f"{limit // 1000} kB"
    else:
        limit_fmt = f"{limit} bytes"

    replaces = {
        "{{ default_timeout }}": str(int(DEFAULT_TIMEOUT.total_seconds())),
        "{{ default_grace }}": str(int(DEFAULT_GRACE.total_seconds())),
        "SITE_NAME": settings.SITE_NAME,
        "SITE_ROOT": settings.SITE_ROOT,
        "SITE_HOSTNAME": site_hostname(),
        "SITE_SCHEME": urlparse(settings.SITE_ROOT).scheme,
        "PING_ENDPOINT": settings.PING_ENDPOINT,
        "PING_URL": settings.PING_ENDPOINT + "your-uuid-here",  # 关键：使用通用占位符
        "PING_BODY_LIMIT_FORMATTED": limit_fmt,
        "PING_BODY_LIMIT": str(limit),
        "IMG_URL": os.path.join(settings.STATIC_URL, "img/docs"),
    }

    for placeholder, value in replaces.items():
        html = html.replace(placeholder, value)

    return html
```

**重要区别**：

| 对比项 | Usage Examples 模态框 | 文档页面（如 GitHub Actions） |
|-------|---------------------|------------------------------|
| `PING_URL` 替换 | `{{ ping_url }}` → 真实 URL（如 `https://hc-ping.com/xxx`） | `PING_URL` → 通用占位符（如 `https://hc-ping.com/your-uuid-here`） |
| 个性化 | ✅ 每个 check 有专属的 URL | ❌ 所有用户看到相同的示例 |
| 复制后操作 | 无需修改，直接可用 | 需要手动替换 `your-uuid-here` 或配置 `secrets.ping_url` |

#### 用户复制与集成流程

**GitHub Actions 集成步骤**：

1. 用户从详情页复制自己的真实 ping URL
2. 在 GitHub 仓库的 Settings → Secrets 中添加 `ping_url` secret
3. 复制文档中的 workflow YAML 代码
4. 根据实际需求修改 `Main-Job` 中的任务
5. 提交到仓库，GitHub Actions 会自动执行

**Workflow 设计要点**：
- 使用 `needs: [Main-Job]` 定义任务依赖关系
- `Ping-Success` 只在 `Main-Job` 成功后执行
- `Ping-Failure` 使用 `if: ${{ failure() }}` 只在失败时执行
- ping URL 从 `secrets.ping_url` 读取，避免硬编码

---

### 2.3 Cloud Provider 集成示例

#### 示例来源

系统**没有**为特定云服务提供商（AWS, GCP, Azure 等）提供专属的集成示例模板。但用户可以通过以下方式集成：

1. **通用语言示例**：使用 Bash、Python、Go 等通用示例
2. **第三方工具**：在 `templates/docs/resources.md` 中列出的社区工具
3. **容器化方案**：Docker 和 Kubernetes 相关资源

**第三方资源示例**（`templates/docs/resources.md`）：

```markdown
## Command Runners, Shell Wrappers

* [runitor](https://github.com/bdd/runitor) - A command runner with Healthchecks.io integration to keep your scripts and containers simple.
* [crontask.sh](https://github.com/pforret/crontask) – Bash wrapper to use in crontab. Supports pinging.

## Tools for Self-Hosting

* [linuxserver/docker-healthchecks](https://github.com/linuxserver/docker-healthchecks) – Alternative Docker image
* [Elestio](https://elest.io/open-source/healthchecks) – Managed hosting platform with Healthchecks support

## API Wrappers

### Terraform
* [terraform-provider-healthchecksio](https://github.com/kristofferahl/terraform-provider-healthchecksio) – Terraform Provider for Healthchecks.io
```

#### 变量替换机制

云服务集成使用与通用语言示例相同的变量替换机制：

**在 "Usage Examples" 模态框中**：
- `{{ ping_url }}` 被替换为真实 URL
- 用户可以直接复制使用

**在云服务的实际使用中**：
- 用户通常会将 ping URL 存储为环境变量或 secret
- 例如：AWS Lambda 的环境变量、GCP Cloud Functions 的环境变量、Kubernetes 的 Secrets

#### 典型云服务集成方案

**AWS Lambda 示例**（用户自行编写）：

```python
import os
import requests

def lambda_handler(event, context):
    # 从环境变量获取 ping URL
    ping_url = os.environ.get('PING_URL')
    
    # 执行实际任务...
    result = do_some_work()
    
    # 发送 ping
    try:
        if result.success:
            requests.get(ping_url, timeout=10)
        else:
            requests.get(f"{ping_url}/fail", timeout=10)
    except requests.RequestException:
        pass  # 静默失败，不影响主任务
    
    return result
```

**Kubernetes CronJob 示例**（用户自行编写）：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-job
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: my-backup-image:latest
            env:
            - name: PING_URL
              valueFrom:
                secretKeyRef:
                  name: healthchecks-secret
                  key: ping-url
            command: ["/bin/sh", "-c"]
            args:
            - |
              /run-backup.sh && curl -fsS -m 10 --retry 5 $PING_URL
          restartPolicy: OnFailure
```

---

## 三、首次 Ping 引导链路关键分支分析

### 3.1 完整引导流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        首次 Ping 引导完整流程                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户进入 Check 详情页                                                        │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    "How To Ping" 区域显示                            │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │ 显示内容：                                                      │  │   │
│  │  │ • HTTP Ping URL: {{ check.url }}                              │  │   │
│  │  │ • Email Ping: {{ check.email }}                               │  │   │
│  │  │ • "Usage Examples" 按钮                                        │  │   │
│  │  │ • "Filtering Rules" 按钮（如有权限）                           │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    "Current Status" 区域显示                         │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │ 状态显示逻辑（log_status_text.html）：                         │  │   │
│  │  │ • status == "new" AND n_pings == 0:                          │  │   │
│  │  │   → "This check has never received a ping."                   │  │   │
│  │  │ • status == "new" AND n_pings > 0:                           │  │   │
│  │  │   → "This check is ready for pings."                         │  │   │
│  │  │ • status == "up":                                             │  │   │
│  │  │   → "This check is up. Last ping was X ago."                │  │   │
│  │  │ • status == "down":                                           │  │   │
│  │  │   → "This check is down. Last ping was X ago."              │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    "Ping Now!" 按钮显示                              │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │ 条件判断（details.html 第 144-158 行）：                     │  │   │
│  │  │                                                                  │  │   │
│  │  │ {% if check.url %}  <!-- 只有 url 不为空才显示按钮 -->        │  │   │
│  │  │   {% if project.show_slugs and not project.ping_key %}        │  │   │
│  │  │       ┌─────────────────────────────────────────────────┐      │  │   │
│  │  │       │ 分支 A: Slug 模式，无 Ping Key                  │      │  │   │
│  │  │       │ • 按钮绑定：data-target="#no-ping-key-modal"   │      │  │   │
│  │  │       │ • 无 id="ping-now"（JS 不会绑定点击事件）      │      │  │   │
│  │  │       └─────────────────────────────────────────────────┘      │  │   │
│  │  │   {% else %}                                                    │  │   │
│  │  │       ┌─────────────────────────────────────────────────┐      │  │   │
│  │  │       │ 分支 B: 正常情况（UUID 模式 或 Slug 模式有 Key）│      │  │   │
│  │  │       │ • 按钮属性：id="ping-now"                        │      │  │   │
│  │  │       │ • data-url="{{ check.url }}"                     │      │  │   │
│  │  │       └─────────────────────────────────────────────────┘      │  │   │
│  │  │   {% endif %}                                                   │  │   │
│  │  │ {% endif %}                                                     │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 分支 A：Slug 模式无 Ping Key 的处理

#### 触发条件

```django
{% if rw and project.show_slugs and not project.ping_key %}
```

三个条件同时满足：
1. `rw`：当前用户有读写权限
2. `project.show_slugs`：项目启用了 Slug 模式
3. `not project.ping_key`：项目尚未生成 Ping Key

#### 按钮行为

**模板代码**（`templates/front/details.html` 第 144-151 行）：

```django
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
```

**关键差异**：

| 属性 | 正常情况 | Slug 模式无 Key |
|-----|---------|-----------------|
| `id` | `id="ping-now"` | 无 `id` 属性 |
| `data-target` | 无 | `data-target="#no-ping-key-modal"` |
| `data-url` | `data-url="{{ check.url }}"` | 无 |
| JS 事件绑定 | ✅ `$("#ping-now").click()` | ❌ 无绑定 |
| 点击行为 | 发送 POST 请求 | 弹出模态框 |

#### 模态框引导

**模态框模板**（`templates/front/details.html` 第 345-366 行）：

```django
{% if rw and project.show_slugs and not project.ping_key %}
<div id="no-ping-key-modal" class="modal">
    <div class="modal-dialog">
        <div class="modal-content">
            <div class="modal-header">
                <button type="button" class="close" data-dismiss="modal">&times;</button>
                <h4>Ping Key Required</h4>
            </div>
            <div class="modal-body">
                <p>This project does not yet have a ping key.<br />
                   To ping this check, please generate the
                   ping key first.
                </p>
            </div>
            <div class="modal-footer">
                <button type="button" class="btn btn-default" data-dismiss="modal">Cancel</button>
                <a href="{% url 'hc-project-settings' project.code %}" class="btn btn-primary">Open Project Settings</a>
            </div>
        </div>
    </div>
</div>
{% endif %}
```

**引导流程**：

1. 用户点击 "Ping Now!" 按钮
2. 弹出 "Ping Key Required" 模态框
3. 用户看到提示："This project does not yet have a ping key. To ping this check, please generate the ping key first."
4. 用户点击 "Open Project Settings" 按钮
5. 跳转到项目设置页面
6. 用户在设置页面生成 Ping Key（通过 `Project.set_ping_key()` 方法）
7. 生成后返回详情页，按钮恢复正常行为

#### Ping Key 生成机制

**`Project.set_ping_key()` 方法**（`hc/accounts/models.py` 第 556-567 行）：

```python
def set_ping_key(self) -> str:
    # The ping key will be:
    # - 22 characters long, consisting of [a-z0-9]
    # - no "_" or "-" characters for aesthetic reasons
    # - no uppercase characters to avoid case-sensitivity issues
    #   in email addresses.
    # The ping key will have ~113 bits of entropy.
    while True:
        self.ping_key = token_urlsafe(16).lower()
        if "_" not in self.ping_key and "-" not in self.ping_key:
            break
    return self.ping_key
```

**Ping Key 特性**：
- 22 字符长度
- 仅包含小写字母和数字 `[a-z0-9]`
- 不包含 `_` 或 `-`（美观考虑）
- 不包含大写字母（避免邮件地址中的大小写敏感性问题）
- 约 113 位熵值（安全性）

### 3.3 分支 B：正常情况的处理

#### 触发条件

满足以下任一条件：
1. `not project.show_slugs`：项目使用 UUID 模式（默认）
2. `project.show_slugs and project.ping_key`：项目使用 Slug 模式且已有 Ping Key

#### 按钮行为

**前端实现**（`static/js/details.js` 第 55-66 行）：

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

**执行流程**：

1. 用户点击 "Ping Now!" 按钮
2. jQuery 通过 `$.post()` 发送 POST 请求到 `this.dataset.url`（即 `check.url`）
3. 请求成功后，按钮文本变为 "Success!"
4. 鼠标移出按钮后 300ms，文本恢复为 "Ping Now!"

**状态更新**：

前端通过 `adaptiveSetInterval` 定期轮询状态（`static/js/details.js` 第 88-124 行）：

```javascript
adaptiveSetInterval(function() {
    $.ajax({
        url: statusUrl + (lastUpdated ? "?u=" + lastUpdated : ""),
        dataType: "json",
        timeout: 2000,
        success: function(data) {
            if (data.status_text != lastStatusText) {
                lastStatusText = data.status_text;
                $("#current-status-icon").attr("class", "status ic-" + data.status);
                $("#current-status-text").html(data.status_text);
                // ...
            }
            // ...
        }
    });
}, true);
```

### 3.4 两种模式的 URL 显示差异

#### UUID 模式（默认）

**条件**：`project.show_slugs = False`

**URL 生成**：
```python
return settings.PING_ENDPOINT + str(self.code)
```

**显示示例**：
```
HTTP Ping URL: https://hc-ping.com/550e8400-e29b-41d4-a716-446655440000
Email Ping: 550e8400-e29b-41d4-a716-446655440000@hc-ping.com
```

**"Ping Now!" 按钮**：正常可用，点击直接发送请求

#### Slug 模式有 Ping Key

**条件**：`project.show_slugs = True` 且 `project.ping_key` 已设置

**URL 生成**：
```python
key = self.project.ping_key
return settings.PING_ENDPOINT + key + "/" + self.slug
```

**显示示例**：
```
HTTP Ping URL: https://hc-ping.com/fqOOd6-F4MMNuCEnzTU01w/db-backups
Email Ping: fqOOd6-F4MMNuCEnzTU01w+db-backups@hc-ping.com
```

**"Ping Now!" 按钮**：正常可用，点击直接发送请求

#### Slug 模式无 Ping Key

**条件**：`project.show_slugs = True` 且 `project.ping_key is None`

**URL 生成**：
```python
key = self.project.ping_key or "{ping_key}"  # 使用占位符
return settings.PING_ENDPOINT + key + "/" + self.slug
```

**显示示例**：
```
HTTP Ping URL: https://hc-ping.com/{ping_key}/db-backups
Email Ping: {ping_key}+db-backups@hc-ping.com
```

**"Ping Now!" 按钮**：点击弹出 "Ping Key Required" 模态框，引导用户去项目设置生成 key

---

## 四、完整流程图（更新版）

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           新建 Check 与首次 Ping 完整流程                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  阶段 1: Check 创建                                                              │
│  ───────────────────────────────────────────────────────────────────────────    │
│                                                                                  │
│  用户提交 Add Check 表单                                                         │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ hc/front/views.py: add_check()                                          │   │
│  │ • 创建 Check 对象，自动生成 code (UUID)                                  │   │
│  │ • 设置 name, slug, tags, kind, timeout, schedule, tz, grace 等        │   │
│  │ • 调用 check.assign_all_channels() 分配所有通道                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                        │
│         ▼                                                                        │
│  重定向到 checks 页面 或 直接进入 details 页面（如果是复制操作）                 │
│                                                                                  │
│  ───────────────────────────────────────────────────────────────────────────    │
│  阶段 2: 详情页显示与引导                                                        │
│  ───────────────────────────────────────────────────────────────────────────    │
│                                                                                  │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        "How To Ping" 区域                                │   │
│  │  • 显示 HTTP Ping URL: {{ check.url }}                                  │   │
│  │  • 显示 Email Ping: {{ check.email }}                                   │   │
│  │  • "Usage Examples" 按钮 → 弹出集成示例模态框                            │   │
│  │  • "Filtering Rules" 按钮（如有读写权限）                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      "Current Status" 区域                                │   │
│  │  状态: new                                                                │   │
│  │  文本: "This check has never received a ping."                          │   │
│  │  明确提示用户需要发送首次 ping                                             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    URL 风格切换按钮（如有读写权限）                        │   │
│  │  [uuid] [slug]                                                           │   │
│  │  • 点击 "slug" → 设置 project.show_slugs = True                         │   │
│  │  • 点击 "uuid" → 设置 project.show_slugs = False                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ───────────────────────────────────────────────────────────────────────────    │
│  阶段 3: 关键分支判断                                                            │
│  ───────────────────────────────────────────────────────────────────────────    │
│                                                                                  │
│         │                                                                        │
│         ▼                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │               判断: project.show_slugs AND NOT project.ping_key?        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│         │                              │                                        │
│         │ Yes                          │ No                                     │
│         ▼                              ▼                                        │
│  ┌──────────────────┐       ┌─────────────────────────────────────────────┐   │
│  │   分支 A         │       │              分支 B                         │   │
│  │ Slug 模式无 Key  │       │   UUID 模式 或 Slug 模式有 Key            │   │
│  └──────────────────┘       └─────────────────────────────────────────────┘   │
│         │                              │                                        │
│         ▼                              ▼                                        │
│  ┌──────────────────────────┐  ┌─────────────────────────────────────────┐   │
│  │ "Ping Now!" 按钮行为:    │  │ "Ping Now!" 按钮行为:                   │   │
│  │ • 无 id="ping-now"       │  │ • id="ping-now"                        │   │
│  │ • data-target="#no-ping- │  │ • data-url="{{ check.url }}"            │   │
│  │   key-modal"             │  │ • JS 绑定点击事件                       │   │
│  └──────────────────────────┘  └─────────────────────────────────────────┘   │
│         │                              │                                        │
│         ▼                              ▼                                        │
│  ┌──────────────────────────┐  ┌─────────────────────────────────────────┐   │
│  │ 点击按钮弹出:            │  │ 点击按钮执行:                           │   │
│  │ ┌──────────────────────┐ │  │ • $.post(this.dataset.url, ...)       │   │
│  │ │  Ping Key Required   │ │  │ • 按钮文本变为 "Success!"              │   │
│  │ │                      │ │  │ • 状态自动更新为 "up"                  │   │
│  │ │ This project does    │ │  │ • 状态文本变为 "This check is up..."  │   │
│  │ │ not yet have a ping  │ │  └─────────────────────────────────────────┘   │
│  │ │ key. To ping this    │ │           │                                    │
│  │ │ check, please generate│ │           ▼                                    │
│  │ │ the ping key first.   │ │  ┌─────────────────────────────────────────┐   │
│  │ │                      │ │  │         首次 Ping 完成!                   │   │
│  │ │ [Cancel] [Open Proj- │ │  │  监控开始正常工作，等待下一次预期的 ping   │   │
│  │ │  ect Settings]        │ │  └─────────────────────────────────────────┘   │
│  │ └──────────────────────┘ │                                                  │
│  └──────────────────────────┘                                                  │
│         │                                                                        │
│         ▼                                                                        │
│  点击 "Open Project Settings"                                                    │
│         │                                                                        │
│         ▼                                                                        │
│  跳转到项目设置页面                                                              │
│         │                                                                        │
│         ▼                                                                        │
│  生成 Ping Key（调用 Project.set_ping_key()）                                   │
│         │                                                                        │
│         ▼                                                                        │
│  返回详情页，现在属于分支 B（Slug 模式有 Key）                                   │
│         │                                                                        │
│         ▼                                                                        │
│  正常使用 "Ping Now!" 按钮或集成示例                                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 五、关键代码文件索引

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| Ping URL 生成 | `hc/api/models.py` | `Check.url()`, `Check.email()`, `Check.to_dict()` |
| 智能 Cron 生成 | `hc/front/templatetags/hc_extras.py` | `guess_schedule()` |
| Usage Examples 模态框 | `templates/front/show_usage_modal.html` | 主模态框模板 |
| 语言特定示例 | `templates/front/snippets/*.html` | 各语言示例代码 |
| CI 文档（GitHub Actions） | `templates/docs/github_actions.md` | CI 集成文档 |
| 文档占位符替换 | `hc/front/views.py` | `_replace_placeholders()` |
| 状态显示引导 | `templates/front/log_status_text.html` | 状态文本模板 |
| 前端 Ping 功能 | `static/js/details.js` | `$("#ping-now").click()`, `$(".click-to-copy").click()` |
| 无 Ping Key 模态框 | `templates/front/details.html` | `#no-ping-key-modal` |
| Ping Key 生成 | `hc/accounts/models.py` | `Project.set_ping_key()` |
| 详情页视图 | `hc/front/views.py` | `details()`, `add_check()` |
| Check 模型 | `hc/api/models.py` | `class Check` |
| Project 模型 | `hc/accounts/models.py` | `class Project` |

---

## 六、设计亮点总结

### 6.1 双模式 URL 设计

- **UUID 模式**：简单、安全、无需额外配置，适合快速上手
- **Slug 模式**：可读性好，适合需要在代码中硬编码的场景，支持同一项目共享 ping key
- **平滑切换**：用户可以随时在两种模式间切换，不影响已有功能

### 6.2 渐进式引导设计

**多层引导机制**：

1. **状态文本提示**："This check has never received a ping." 明确告知用户当前状态
2. **一键测试按钮**："Ping Now!" 提供零成本测试方式
3. **详细集成示例**："Usage Examples" 提供 11 种语言/平台的示例代码
4. **文档资源**：详细的文档页面（如 GitHub Actions、Cron 监控等）

**异常分支处理**：

- Slug 模式无 ping key 时，不是让按钮失效或报错，而是通过模态框明确引导用户去生成 key
- 提供 "Open Project Settings" 快捷按钮，减少用户操作步骤

### 6.3 智能示例生成

- `guess_schedule` 过滤器根据 timeout 自动推断合适的 cron 表达式
- 减少用户手动配置的错误概率
- 无法推断时显示 `# FIXME` 提示，而非静默失败

### 6.4 变量替换的分层设计

| 场景 | 替换方式 | 特点 |
|-----|---------|------|
| Usage Examples 模态框（Cron 示例） | Django 模板变量 `{{ ping_url }}` | 每个 check 有专属的真实 URL，复制即可用 |
| Usage Examples 模态框（通用语言示例） | Django 模板变量 `{{ ping_url }}` | 每个 check 有专属的真实 URL，复制即可用 |
| 文档页面（CI 示例，如 GitHub Actions） | `_replace_placeholders()` 函数 | 使用通用占位符 `your-uuid-here`，适合文档场景 |
| 文档页面（第三方资源，如 Terraform） | 无替换，用户手动配置 | 社区提供的第三方库/工具，需要自行集成 |
| API 响应 | 硬编码 UUID 格式 | 始终返回 UUID 格式，不考虑 slug 配置 |

### 6.5 安全考虑

- Ping Key 生成使用 `token_urlsafe(16)`，约 113 位熵值
- Slug 模式中，slug 可以公开（硬编码在脚本中），真正的 secret 是 ping key
- UUID 模式中，UUID 本身就是 secret（UUID 地址空间足够大，难以猜测）

### 6.6 用户体验优化

- **点击复制**：`click-to-copy` class 让用户可以一键复制 URL 和示例代码
- **即时反馈**："Ping Now!" 点击后显示 "Success!"，鼠标移出后恢复
- **状态轮询**：自动定期更新状态，无需手动刷新
- **Tooltip 提示**：复制功能有 "Click to copy" / "Copied!" 反馈
