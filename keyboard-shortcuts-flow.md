# Miniflux 键盘快捷键注册与用户偏好持久化流程分析

## 一、整体架构概览

Miniflux 的键盘快捷键系统采用"服务端渲染模板注入 + 前端 JavaScript 监听"的经典架构，不存在独立的 WebSocket 或实时双向同步协议。整个流程通过传统的 HTTP 请求（表单提交 + JSON API）完成用户偏好的双向同步。

```
┌──────────────────────┐     HTTP GET (HTML)     ┌──────────────────────┐
│   PostgreSQL 数据库   │ ◄───────────────────── │   Go HTTP 后端服务    │
│  (users 表存储偏好)   │                         │  (渲染模板注入偏好)  │
└──────────────────────┘                         └──────────┬───────────┘
                                                           │
                                                           ▼
                                                  ┌──────────────────────┐
                                                  │  HTML <body> data-*  │
                                                  │   属性承载偏好值     │
                                                  └──────────┬───────────┘
                                                             │
                                                             ▼
                                                  ┌──────────────────────┐
                                                  │  前端 JS 读取 data-* │
                                                  │  KeyboardHandler 监听│
                                                  └──────────┬───────────┘
                                                             │
                                                             ▼
                                                  ┌──────────────────────┐
                                                  │  POST /settings 表单 │
                                                  │  或 PATCH /v1/me API │
                                                  └──────────────────────┘
```

---

## 二、前端键盘快捷键注册与监听

### 2.1 KeyboardHandler 类 (`internal/ui/static/js/keyboard_handler.js`)

核心类提供快捷键注册和按键队列机制：

```javascript
class KeyboardHandler {
    constructor() {
        this.queue = [];          // 按键序列队列，支持 "g u" 这类组合键
        this.shortcuts = new Map(); // combination -> { keys, callback }
        this.triggers = new Set();  // 所有组合键的首键集合，用于快速过滤
    }
}
```

**注册方法 `on(combination, callback)`**：
- 将按键组合（如 `"g u"` 或 `"j"`）按空格分割为按键数组
- 存储到 `shortcuts` Map 中
- 将组合的首个按键加入 `triggers` 集合用于快速忽略无关按键

**监听方法 `listen()`**：
- 绑定 `document.onkeydown` 事件
- 通过 `KeyboardHandler.getKey(event)` 规范化键名（兼容旧浏览器如 `Esc` → `Escape`）
- 忽略规则：
  - 目标元素是 `<input>` 或 `<textarea>` 时跳过
  - 按下修饰键（Ctrl/Alt/Meta）时跳过
  - 队列为空且按键不在 `triggers` 集合中时跳过
- 按键匹配逻辑：
  - 单键：`keys.length === 1` 且键名相等即触发
  - 序列键（如 `"g u"`）：依次比较 `queue` 中的每个元素与 `keys` 对应位置
  - 匹配成功后清空队列并执行回调
  - 队列超过 2 个未匹配元素时自动清空

### 2.2 快捷键注册位置 (`internal/ui/static/js/app.js:1165-1224`)

`initializeKeyboardShortcuts()` 函数在页面加载时执行：

```javascript
function initializeKeyboardShortcuts() {
    // 关键：检查 body 的 data-disable-keyboard-shortcuts 属性
    if (document.querySelector("body[data-disable-keyboard-shortcuts=true]")) return;

    const keyboardHandler = new KeyboardHandler();

    // 导航类: g u, g b, g h, g f, g c, g s, g g, G, /
    // 条目导航: ArrowLeft, ArrowRight, k, p, j, n, h, l, z t
    // 条目操作: o, Enter, v, V, c, C
    // 条目管理: m, M, A, s, d, f
    // 订阅源操作: F, R, +, #
    // UI 操作: ?, a

    keyboardHandler.listen();
}
```

**偏好开关的前端读取**：
- 没有独立的 AJAX 请求获取偏好
- 直接读取 `<body>` 标签上的 `data-disable-keyboard-shortcuts` 属性
- 若属性值为 `"true"` 则完全不初始化快捷键监听

---

## 三、后端用户偏好持久化

### 3.1 数据模型 (`internal/model/user.go`)

`User` 结构体包含 `KeyboardShortcuts bool` 字段（第 29 行），JSON tag 为 `keyboard_shortcuts`。用户偏好作为 `users` 表的列直接存储，不使用独立的 key-value 偏好表。

完整的偏好字段清单：

| 字段 | 类型 | 说明 |
|------|------|------|
| `Theme` | string | 主题 |
| `Language` | string | 语言 |
| `Timezone` | string | 时区 |
| `EntryDirection` | string | 条目排序方向 |
| `EntryOrder` | string | 条目排序字段 |
| `EntriesPerPage` | int | 每页条目数 |
| `KeyboardShortcuts` | bool | **键盘快捷键开关** |
| `ShowReadingTime` | bool | 显示阅读时间 |
| `EntrySwipe` | bool | 条目滑动手势 |
| `GestureNav` | string | 手势导航模式 |
| `DisplayMode` | string | PWA 显示模式 |
| `DefaultReadingSpeed` | int | 默认阅读速度 |
| `CJKReadingSpeed` | int | CJK 阅读速度 |
| `DefaultHomePage` | string | 默认首页 |
| `CategoriesSortingOrder` | string | 分类排序 |
| `MarkReadOnView` | bool | 浏览时标记已读 |
| `MarkReadOnMediaPlayerCompletion` | bool | 媒体播放完成标记已读 |
| `MediaPlaybackRate` | float64 | 媒体播放速率 |
| `Stylesheet` | string | 自定义 CSS |
| `CustomJS` | string | 自定义 JS |
| `ExternalFontHosts` | string | 外部字体域名 |
| `BlockFilterEntryRules` | string | 条目过滤规则（屏蔽） |
| `KeepFilterEntryRules` | string | 条目过滤规则（保留） |
| `AlwaysOpenExternalLinks` | bool | 始终打开外部链接 |
| `OpenExternalLinksInNewTab` | bool | 新标签页打开外部链接 |

`UserModificationRequest` 使用指针类型（如 `*bool`）以区分"未修改"和"设为 false"。其 `Patch(user)` 方法将非 nil 字段合并到现有 User 对象。

### 3.2 存储层 (`internal/storage/user.go`)

**`UpdateUser(user *model.User)`**：
- 执行 PostgreSQL `UPDATE users SET ... WHERE id=$N`
- 区分两种 SQL：带密码更新（31 个参数）和不带密码更新（30 个参数）
- `keyboard_shortcuts` 对应参数位置：带密码是 `$9`，不带密码是 `$8`

**`UserByID(userID int64)`**：
- `SELECT` 语句包含完整 31 列，通过 `fetchUser` 辅助方法扫描到 `model.User` 结构体

**`CreateUser`**：
- 插入后使用 `RETURNING` 子句返回所有列的默认值（包括 `keyboard_shortcuts` 的数据库默认值）

### 3.3 数据库表结构 (`internal/database/migrations.go`)

初始迁移创建 `users` 表，`keyboard_shortcuts` 等偏好列通过后续迁移逐步添加（代码中未显示具体迁移号，但从 `CreateUser` 的 RETURNING 子句可见完整列清单）。

---

## 四、偏好从后端到前端的单向传递（服务器端渲染）

### 4.1 流程链路

```
HTTP 请求到达
    │
    ▼
webSessionMiddleware (internal/ui/web_session_middleware.go)
    │  从 cookie 加载 session，从 DB 读取用户，注入 request context
    ▼
业务 Handler (如 showUnreadPage, showSettingsPage 等)
    │  调用 h.store.UserByID(request.UserID(r)) 获取 *model.User
    │  调用 view.New().Set("user", user).Set(...) 设置模板变量
    ▼
view.New (internal/ui/view/view.go)
    │  从 session 读取 csrf/language/theme 等默认变量
    ▼
模板引擎渲染 common/layout.html
    │  读取 .user.KeyboardShortcuts 等字段
    │  生成 <body data-disable-keyboard-shortcuts="true">
    ▼
HTML 响应返回浏览器
```

### 4.2 模板注入点 (`internal/template/templates/common/layout.html:54-56`)

```html
<body
    ...
    {{ if .user }}
        {{ if not .user.KeyboardShortcuts }}data-disable-keyboard-shortcuts="true"{{ end }}
        data-mark-as-read-on-view="{{ if .user.MarkReadOnView }}true{{ else }}false{{ end }}"
    {{ end }}>
```

设计要点：
- 仅当 `KeyboardShortcuts = false` 时才输出 `data-disable-keyboard-shortcuts="true"` 属性
- 当偏好为 `true`（启用）时，该属性完全不出现，前端以"不存在即启用"逻辑判断
- 其他偏好（如 `mark-as-read-on-view`）类似地通过 `data-*` 属性传递

### 4.3 Web Session 中间件 (`internal/ui/web_session_middleware.go`)

作用：
- 从 Cookie 解析 `sessionID.secret`
- 查询 `web_sessions` 表获取会话
- 会话绑定用户后，通过 `request.UserID(r)` 供后续 Handler 使用
- 会话变更时（如登录后 `SetUser`）在请求结束时自动持久化到 DB

注意：**用户偏好本身不存储在 session 中**，session 仅存 `userID`，每次请求都从 `users` 表重新读取偏好数据，保证数据实时性。

---

## 五、偏好从前端到后端的持久化路径

提供两条独立路径：**Web UI 表单提交** 和 **REST JSON API**。

### 5.1 路径一：Web UI 表单提交

#### 5.1.1 设置页面表单 (`internal/template/templates/views/settings.html:219`)

```html
<label>
    <input type="checkbox" name="keyboard_shortcuts" value="1"
           {{ if .form.KeyboardShortcuts }}checked{{ end }}>
    {{ t "form.prefs.label.keyboard_shortcuts" }}
</label>
```

表单 method=POST，action=`/settings`，包含 CSRF hidden input。

#### 5.1.2 表单解析 (`internal/ui/form/settings.go:168-215`)

`NewSettingsForm(r)` 使用 `r.FormValue()` 读取：

```go
KeyboardShortcuts: r.FormValue("keyboard_shortcuts") == "1",
```

未勾选时浏览器不发送该字段，`FormValue` 返回空字符串，结果为 `false`。

#### 5.1.3 表单合并 (`internal/ui/form/settings.go:94-131`)

`SettingsForm.Merge(user)` 将表单值写回 `*model.User`：

```go
user.KeyboardShortcuts = s.KeyboardShortcuts
```

#### 5.1.4 Handler 处理 (`internal/ui/settings_update.go:19-97`)

```go
func (h *handler) updateSettings(w http.ResponseWriter, r *http.Request) {
    user, _ := h.store.UserByID(request.UserID(r))     // 1. 读取当前用户
    settingsForm := form.NewSettingsForm(r)             // 2. 解析表单
    settingsForm.Validate()                             // 3. 校验
    h.store.UpdateUser(settingsForm.Merge(user))        // 4. 合并+写入DB
    sess := request.WebSession(r)
    sess.SetUser(user)                                  // 5. 更新session缓存(language/theme)
    sess.SetSuccessMessage(...)                         // 6. Flash 消息
    response.HTMLRedirect(w, r, h.routePath("/settings")) // 7. 302重定向
}
```

路由注册 (`internal/ui/ui.go:127`)：`mux.HandleFunc("POST /settings", handler.updateSettings)`

### 5.2 路径二：REST JSON API

#### 5.2.1 获取当前用户

```
GET /v1/me
Authorization: Bearer <api_token>
```

Handler (`internal/api/user_handlers.go:18-26`)：
- 通过 API Token 中间件认证（区别于 Web UI 的 Cookie/Session）
- 返回完整 `*model.User` 的 JSON 表示，包含 `keyboard_shortcuts: true/false`

#### 5.2.2 更新当前用户

```
PUT /v1/users/{userID}
Content-Type: application/json
Authorization: Bearer <api_token>

{
    "keyboard_shortcuts": false,
    "theme": "dark"
}
```

Handler (`internal/api/user_handlers.go:54-102`)：
- 反序列化 JSON 到 `model.UserModificationRequest`（字段为指针类型）
- 权限检查：非管理员只能修改自己，且不能提升为管理员
- `userModificationRequest.Patch(originalUser)` 合并非 nil 字段
- `h.store.UpdateUser(originalUser)` 写入数据库
- 返回更新后的完整 User JSON

#### 5.2.3 API 与 Web UI 的区别

| 维度 | Web UI /settings | REST API /v1/users/{id} |
|------|------------------|------------------------|
| 认证方式 | Session Cookie | Bearer Token (X-Auth-Token) |
| Content-Type | application/x-www-form-urlencoded | application/json |
| 响应类型 | 302 重定向到 HTML | 201 Created 返回 JSON |
| 修改粒度 | 全量表单提交 | 部分更新（Patch 语义） |
| 调用方 | 浏览器用户 | 第三方客户端 / 移动端 App |

---

## 六、"双向同步"的真实含义

Miniflux **没有实时双向同步协议**（如 WebSocket / Server-Sent Events）。所谓"同步"是通过以下机制间接实现的：

### 6.1 后端 → 前端

- **时机**：每次页面加载（导航跳转、刷新、POST 后重定向）
- **载体**：HTML `<body>` 标签的 `data-*` 属性
- **协议**：HTTP 1.1 GET，响应为 text/html
- **延迟**：用户主动触发页面跳转时才更新

### 6.2 前端 → 后端

- **时机**：用户点击设置页面的"Update"按钮（Web UI）或客户端主动发起 PUT 请求（API）
- **载体**：表单编码 POST body 或 JSON PUT body
- **协议**：HTTP 1.1 POST / PUT，带 CSRF Token 或 API Token
- **延迟**：用户主动提交时才更新

### 6.3 为什么不需要实时同步？

键盘快捷键开关等偏好属于"低频变更"属性，用户很少修改。每次页面加载重新从 DB 读取并注入 HTML 的策略，既简单又足够满足需求，避免了引入复杂的实时通信机制。

---

## 七、关键安全设计

1. **CSRF 防护**：Web UI 所有 POST 表单包含 `csrf` hidden 字段，由 `csrfMiddleware` 校验
2. **密码哈希**：更新用户时使用 `crypto.HashPassword` (bcrypt)，且 UpdateUser 分两条 SQL 避免覆盖密码
3. **权限隔离**：REST API 中非管理员用户不能修改他人，也不能自提升管理员权限
4. **会话轮换**：登录时 `WebSession.Rotate()` 更换 session ID，防止会话固定攻击
5. **CSP**：通过模板 `csp` 函数注入 Content-Security-Policy meta 标签，限制脚本来源

---

## 八、关键文件速查表

| 文件 | 职责 |
|------|------|
| `internal/ui/static/js/keyboard_handler.js` | 键盘事件监听与快捷键匹配 |
| `internal/ui/static/js/app.js` | 注册所有具体快捷键、读取 body data-* |
| `internal/model/user.go` | User / UserModificationRequest 数据模型 |
| `internal/storage/user.go` | users 表 CRUD SQL |
| `internal/ui/settings_show.go` | GET /settings 渲染设置页 |
| `internal/ui/settings_update.go` | POST /settings 保存偏好 |
| `internal/ui/form/settings.go` | SettingsForm 解析、校验、合并 |
| `internal/api/user_handlers.go` | REST API 的 GET/PUT 用户 |
| `internal/template/templates/common/layout.html` | <body data-* 注入偏好 |
| `internal/template/templates/views/settings.html` | 设置页表单 HTML |
| `internal/ui/web_session_middleware.go` | 会话加载、用户认证、会话持久化 |
| `internal/ui/ui.go` | 所有 UI 路由注册 |
| `internal/database/migrations.go` | 数据库 schema 演进 |
