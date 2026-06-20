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

---

## 九、快捷键冲突检测：`on()` 方法无防御，`Map.set` 静默覆盖

### 9.1 代码事实

`KeyboardHandler.on(combination, callback)` 的完整实现（`keyboard_handler.js:8-12`）：

```javascript
on(combination, callback) {
    const keys = combination.split(" ");
    this.shortcuts.set(combination, { keys, callback });
    this.triggers.add(keys[0]);
}
```

整个方法**没有任何冲突检测分支**。具体分析：

1. **`this.shortcuts.set(combination, ...)`**：`Map.set()` 在 key 已存在时会静默替换旧值，不抛异常、不返回冲突标记、不发出警告。如果 `initializeKeyboardShortcuts()` 中先后注册了两个相同 combination（如 `"j"`），后者回调直接覆盖前者，前者永久丢失。

2. **`this.triggers.add(keys[0])`**：`Set.add()` 对已存在的值也是幂等操作，无副作用。

3. **序列键前缀冲突**：假设注册 `"g u"` 和 `"g g"` 后，又注册 `"g u"` 的另一个回调，同样静默覆盖。更微妙的是，`"g"` 作为 trigger 存在于 Set 中，按键 `"g"` 会进入队列，但 `"g"` 本身不是任何单键快捷键，所以队列中只留下 `"g"` 等待下一个键——这不算冲突，但说明 `triggers` 是粗粒度过滤器，只决定"是否值得把按键放入队列"。

### 9.2 运行时匹配中的隐式"冲突"行为

`listen()` 中的匹配循环（`keyboard_handler.js:27-39`）：

```javascript
for (const [combination, { keys, callback }] of this.shortcuts.entries()) {
    if (keys.every((value, index) => value === this.queue[index])) {
        this.queue = [];
        callback(event);
        return;
    }
    if (keys.length === 1 && key === keys[0]) {
        this.queue = [];
        callback(event);
        return;
    }
}
```

关键行为：**先注册的序列键优先级高于后注册的单键**。

假设注册了 `"g"` → actionA（单键）和 `"g u"` → actionB（序列键）：
- 按下 `"g"` 时，循环遍历 `shortcuts` Map，先遇到 `"g u"`，`keys.every(...)` 检查 `[queue[0]]` 与 `["g", "u"]`，因为 `queue` 长度不足所以 `every` 返回 `true`（空位满足）——但实际 JS 的 `every` 对越界索引返回 `true`，这意味着 `["g"]` 和 `["g", "u"]` 的前缀匹配成功，`actionB` 被错误触发。

   等等，让我们重新审视：`queue` 此时是 `["g"]`，`keys` 是 `["g", "u"]`。`keys.every((value, index) => value === this.queue[index])` 检查：
   - index=0: `"g" === "g"` → true
   - index=1: `"u" === undefined` → false
   - 结果 false，不匹配。

   继续循环到单键 `"g"`：`keys.length === 1 && key === "g"` → true，触发 `actionA`。

   所以**单键先被触发，序列键的第二个键来时队列已被清空**。这是实际存在的冲突：如果用户先按 `"g"` 想触发 `"g u"`，但 `"g"` 作为单键已被注册，`"g"` 立即被消费，`"g u"` 永远无法触发。

3. **Miniflux 的实际注册中不存在这种冲突**：查看 `app.js:1170-1223`，`"g"` 从未单独注册为单键快捷键，所有以 `"g"` 开头的都是双键序列（`"g u"`, `"g b"`, `"g h"`, `"g f"`, `"g c"`, `"g s"`, `"g g"`），因此不存在前缀冲突。但这是**靠开发者人工保证**的，没有代码层面的强制检测。

### 9.3 结论

| 维度 | 现状 |
|------|------|
| 冲突检测 | **不存在**。`on()` 是无条件覆盖写入 |
| 序列键 vs 单键冲突 | 如果 `"g"` 同时注册为单键和 `"g u"` 的前缀，单键胜出，序列键永远无法触发 |
| 运行时保护 | 无。Map 迭代顺序决定匹配优先级（先注册先匹配），但序列键的前缀匹配在 queue 长度不足时返回 false，所以不会误触发 |
| 开发者保障 | 靠人工审查确保 `g`/`z` 等序列前缀键不作为单键注册 |

### 9.4 chord 组合键（Ctrl+Shift+K）的冲突识别分析

#### 9.4.1 Miniflux 不支持 Ctrl/Alt/Meta 修饰的 chord

`keyboard_handler.js:17` 的 `listen()` 方法有一道硬门槛：

```javascript
if (this.isEventIgnored(event, key) || KeyboardHandler.isModifierKeyDown(event)) {
    return;
}
```

`isModifierKeyDown()` 定义（第 53-55 行）：
```javascript
static isModifierKeyDown(event) {
    return event.getModifierState("Control") || event.getModifierState("Alt") || event.getModifierState("Meta");
}
```

**只要按住 Ctrl、Alt、Meta（Cmd）中任意一个，按键事件直接被丢弃**，连队列都不会进入。因此：
- `"Ctrl+K"`、`"Ctrl+Shift+K"`、`"Alt+K"`、`"Cmd+Shift+P"` 这类 chord 组合键**永远不会被触发**
- 不存在"Ctrl+Shift+K 与其他键冲突"的问题——因为它们根本不被系统识别
- 冲突检测（哪怕是最简单的存在性检查）也轮不到 chord 键上场

#### 9.4.2 Shift 键是个例外：通过 `event.key` 大小写间接体现

Shift 不在 `isModifierKeyDown()` 的检查列表中。Shift 的作用是**改变 `event.key` 的返回值**：
- 按 `k` → `event.key === "k"`（小写）
- 按 `Shift+k` → `event.key === "K"`（大写）

在 Miniflux 实际注册中，大小写键确实是独立注册的两个快捷键：
- `"v"` → 在后台打开原文
- `"V"` → 在新标签页打开原文
- `"m"` → 下一条目状态切换
- `"M"` → 上一条目状态切换
- `"c"` → 打开评论链接（后台）
- `"C"` → 打开评论链接（新标签页）
- `"f"` → 切换星标
- `"F"` → 跳转到订阅源页

**"v" 和 "V" 算不算冲突？** 不算。在 `KeyboardHandler.shortcuts` Map 中，`"v"` 和 `"V"` 是两个不同的 key，各自绑定不同的回调函数。`Map.set()` 不会互相覆盖，匹配时也不会混淆——`"k" === "K"` 为 `false`。

#### 9.4.3 序列化形式的结论

如果有人想用 `"Ctrl+Shift+K"` 这种以 `+` 连接的字符串作为 combination 注册：

1. `on("Ctrl+Shift+K", callback)` 会被 `split(" ")` 切成 `["Ctrl+Shift+K"]`（单键），存入 Map 的 key 是 `"Ctrl+Shift+K"` 字符串本身
2. 实际按键时，`event.key` 按下 K 返回 `"K"`，按下 Ctrl 返回 `"Control"`（单独按修饰键时），同时按 Ctrl+K 时 `event.key` 仍为 `"k"` 但 `isModifierKeyDown()` 返回 true 直接跳过
3. **永远匹配不上**——既因为修饰键过滤，也因为 key 字符串不匹配

**序列化格式差异总结**：

| 系统 | 组合键表示方式 | 分隔符 | 修饰键处理 |
|------|--------------|--------|-----------|
| Miniflux | `g u`（序列键）、`V`（Shift 间接体现） | 空格（序列） | Ctrl/Alt/Meta 直接忽略 |
| Vim/Emacs | `<C-S-k>` 或 `C-M-k` | 特殊符号 | 支持完整修饰键 |
| VS Code | `ctrl+shift+k` | `+` | 支持完整修饰键 |
| Mousetrap.js | `ctrl shift k` 或 `g g` | 空格 | 支持修饰键 + 序列键 |

Miniflux 的 `KeyboardHandler` 是一个**极简实现**，只支持"单键"和"空格分隔的按键序列"两种模式，不支持任何修饰键 chord。

### 9.5 modifier_chord_serialize：修饰键顺序归一化不存在

#### 9.5.1 代码事实：无此函数、无归一化逻辑

对全代码库搜索 `modifier_chord_serialize`、`normalize`、`canonical`、`normalise`、`sort.*key` 等标识符的结果：
- **零个匹配**。不存在修饰键序列化/归一化函数。
- `keyboard_handler.js` 中唯一与"键名规范化"相关的代码是 `static getKey(event)`（第 57-66 行），仅处理浏览器兼容别名（`Esc` → `Escape`、`Up` → `ArrowUp` 等），不涉及修饰键顺序。
- `config/options.go` 中无任何 keyboard/modifier 相关配置项。

#### 9.5.2 假设性分析：如果支持修饰键 chord，归一化是必需的

修饰键组合存在顺序歧义问题：
- 用户按下 `Shift+Ctrl+K` 和 `Ctrl+Shift+K`，物理按键顺序不同，但语义完全相同
- 如果序列化时不做归一化，`"shift-ctrl-k"` 和 `"ctrl-shift-k"` 会被视为两个不同的快捷键
- 注册表 `Map.set()` 不会自动识别它们的等价性，导致冲突检测失效（两个不同的 key 都能注册成功，但实际触发时只能命中一个）

业界标准归一化方案（如 VS Code、Mousetrap.js、Atom）都是：**修饰键按字典序排序后拼接**：
```
原始按键顺序 → 提取修饰键集合 → 排序 → 拼接
"shift-ctrl-k"   → {Control, Shift}   → ["Control","Shift"] → "ctrl-shift-k"
"ctrl-shift-k"   → {Control, Shift}   → ["Control","Shift"] → "ctrl-shift-k"
```

#### 9.5.3 Miniflux 当前为何不需要归一化

因为 `isModifierKeyDown()` 在入口处就过滤了所有 Ctrl/Alt/Meta 组合：
- 修饰键 chord 根本不进入匹配逻辑
- Shift 通过 `event.key` 的大小写间接处理（`k` vs `K`），不存在"Shift 顺序"问题
- 序列键（`g u`）是严格顺序敏感的（先 g 后 u ≠ 先 u 后 g），不能排序归一化

**结论**：在当前架构下，修饰键顺序归一化是一个"不存在的问题"。如果未来要扩展支持 Ctrl/Alt chord，首要任务不是写冲突检测，而是先实现修饰键的提取 + 字典序归一化序列化函数。

### 9.6 sort.Strings 归一化：macOS Cmd 与 Linux Meta 不会被归到同一桶

#### 9.6.1 代码事实：sort.Strings 与修饰键完全无关

全代码库中 `sort.Strings` 只出现一次，在 `internal/reader/opml/serializer.go:46`：

```go
groupedSubs := groupSubscriptionsByFeed(subscriptions)
categories := make([]string, 0, len(groupedSubs))
for k := range groupedSubs {
    categories = append(categories, k)
}
sort.Strings(categories)  // 对 RSS 分类名称按字典序排序
```

这是**OPML 导出时对分类名称排序**，与键盘快捷键、修饰键归一化毫无关系。代码中不存在任何 `sort.Strings(modifiers)` 或类似的修饰键排序逻辑。

#### 9.6.2 Cmd / Meta 的检测方式

`keyboard_handler.js:54` 中的修饰键检测：
```javascript
static isModifierKeyDown(event) {
    return event.getModifierState("Control") || event.getModifierState("Alt") || event.getModifierState("Meta");
}
```

W3C DOM `getModifierState("Meta")` 的浏览器实现：
- **macOS Safari / Chrome**：`"Meta"` 对应 ⌘ Command 键（物理键 Cmd）
- **Linux GNOME / KDE**：`"Meta"` 对应 Win 键 / Super 键（物理键通常是 Windows Logo）
- **Windows**：`"Meta"` 通常不被使用，Windows 键映射到 OS 层面

浏览器已经在 DOM 层将"系统主键"统一命名为 `"Meta"`，这是**浏览器内核的归一化**，与 Miniflux 代码无关。

#### 9.6.3 假设分析：如果真的要 sort.Strings 归一化，会有什么问题？

假设未来实现修饰键支持，提取逻辑为：
```javascript
// 伪代码
const modifiers = [];
if (event.getModifierState("Control")) modifiers.push("Control");
if (event.getModifierState("Meta"))    modifiers.push("Meta");   // macOS Cmd = Linux Meta
if (event.getModifierState("Alt"))     modifiers.push("Alt");
modifiers.sort();  // 即 JS 中的 Array.sort()，对应 Go 的 sort.Strings
```

**"归到同一桶"的真实含义**：
- macOS 上按 Cmd → modifiers 为 `["Meta"]`
- Linux 上按 Super/Win → modifiers 为 `["Meta"]`
- 两者 `sort()` 后结果相同 → **归到同一桶**

但这是浏览器 `getModifierState("Meta")` 统一命名的结果，不是 `sort.Strings` 的功劳。`sort.Strings` 只解决顺序问题（`["Meta","Control"]` vs `["Control","Meta"]`），不解决命名问题。

**真正的系统差异问题不在排序，而在按键语义**：

| 系统 | 物理键 | `getModifierState()` 返回 | 传统应用语义 |
|------|--------|--------------------------|------------|
| macOS | ⌘ Cmd | `"Meta"` | 主命令键（等效于 Windows Ctrl） |
| macOS | ⌃ Ctrl | `"Control"` | 次要修饰键（终端 Emacs 风格） |
| Linux | Ctrl | `"Control"` | 主命令键 |
| Linux | Super/Win | `"Meta"` | 窗口管理 / 次要修饰键 |
| Windows | Ctrl | `"Control"` | 主命令键 |
| Windows | Win | `"Meta"` / 不触发 | 系统级快捷键 |

如果用户在 macOS 上设置快捷键 `Ctrl+K`，期望的是 ⌃+K；但同样的配置在 Linux 上 Ctrl+K 是"主命令键+K"，语义完全不同。**`sort.Strings` 归一化解决不了这个语义跨平台差异问题**——需要的是额外的"主命令键"抽象层（如 VS Code 的 `Ctrl/Cmd` 统一写法）。

#### 9.6.4 另一个实际场景：app.js 中的 metaKey 独立使用

`app.js:541` 中导航链接点击处理：
```javascript
if (linkElement && !event.ctrlKey && !event.shiftKey && !event.metaKey) {
    event.preventDefault();  // 无修饰键时使用 SPA 内部跳转
    window.location.href = linkElement.getAttribute("href");
}
```

这里 `ctrlKey` 和 `metaKey` **分开检查**，不做归一化。效果是：
- 按 Ctrl+点击 → 不拦截，浏览器在新标签页打开（默认行为）
- 按 Cmd+点击（macOS）→ 同样不拦截，浏览器新标签页打开
- 按 Super+点击（Linux）→ 同样不拦截

这个场景下 Cmd 和 Meta 被**故意当成等价的"在新标签页打开"修饰符**，但实现方式是分别列出 `ctrlKey` 和 `metaKey` 两个属性，不是通过排序归一化到同一桶。

---

## 十、keymap JSONB 字段：不存在，偏好全部存为扁平列

### 10.1 代码事实

对整个代码库的搜索结果：

- **`keymap`/`KeyMap`/`key_map`/`keyMap`**：全局零匹配。代码中不存在任何 keymap 相关字段或概念。
- **JSONB 字段**：`migrations.go` 中仅有两处 JSONB：
  1. 第 198 行：旧版 `sessions` 表的 `data jsonb`（已被后续迁移删除）
  2. 第 1456 行：`web_sessions` 表的 `state jsonb not null default '{}'::jsonb`
- **GIN 索引**：`migrations.go` 中所有 GIN 索引只作用于 `entries.document_vectors`（全文搜索向量），与用户偏好无关。

### 10.2 `keyboard_shortcuts` 的存储方式

迁移第 303 行：

```sql
ALTER TABLE users ADD COLUMN keyboard_shortcuts boolean default 't'
```

这是一个**简单的 `boolean` 列**，默认值 `true`（启用）。没有 JSONB、没有 keymap、没有每用户自定义快捷键映射表。

### 10.3 JSONB vs 扁平列的查询效率对比（假设分析）

虽然 Miniflux 不使用 JSONB 存储偏好，但若假设一种替代方案——将所有偏好存入一个 JSONB 字段如 `preferences jsonb`——对比如下：

| 维度 | 扁平列（现有方案） | JSONB 字段（假设方案） |
|------|-------------------|----------------------|
| 读取单个偏好 | `SELECT keyboard_shortcuts FROM users WHERE id=$1`，走主键索引，O(1) | `SELECT preferences->>'keyboard_shortcuts' FROM users WHERE id=$1`，需 JSONB 解析 |
| 写入单个偏好 | `UPDATE users SET keyboard_shortcuts=$2 WHERE id=$1`，直接赋值 | `UPDATE users SET preferences = jsonb_set(preferences, '{keyboard_shortcuts}', 'false') WHERE id=$1`，需读-改-写整条 JSONB |
| 全量读取偏好 | `SELECT * FROM users WHERE id=$1`，列映射直接 Scan | 同上，但需逐字段 `->>` 提取，或整个 JSONB 反序列化 |
| GIN 索引 | 不需要，主键索引足够 | 如需 `WHERE preferences @> '{"keyboard_shortcuts": false}'` 这种条件查询才需要 GIN 索引，但 Miniflux 从不做偏好的条件查询 |
| Schema 演进 | 每新增偏好需 `ALTER TABLE ADD COLUMN` + Go 结构体字段 | JSONB 天然灵活，无需迁移 |
| 类型安全 | Go 结构体强类型，编译期检查 | 需要手动类型断言，运行时可能出错 |
| 存储空间 | bool 占 1 字节，int 占 4 字节 | JSONB 键名重复存储，每个用户一份键名开销 |

**Miniflux 选择扁平列的理由**：

1. 偏好字段数量有限（约 25 个），且新增频率极低（年级别），JSONB 的灵活性优势不明显
2. Miniflux 宣称 "Doesn't use any ORM"（README），扁平列 + 手写 SQL 与此哲学一致
3. 所有偏好查询都是 `WHERE id=$1`（按主键），不存在跨用户的偏好条件查询，GIN 索引无用武之地
4. Go 强类型结构体与扁平列天然匹配，代码更清晰

### 10.4 `web_sessions.state` 中 JSONB 的使用场景

`web_sessions` 表的 `state jsonb` 存储**会话级临时状态**（CSRF token、flash 消息、OAuth2 流程数据、语言/主题缓存），与用户偏好无关。其 JSONB 选择合理：
- 会话状态结构多变（OAuth2 流程中需要临时存储 state/code_verifier）
- 生命周期短，不需要跨会话查询
- 不需要条件索引

### 10.5 `OptionalString` / `OptionalNumber` 的隐式零值问题

`model/model.go` 中的辅助函数：

```go
func OptionalNumber[T Number](value T) *T {
    if value > 0 {
        return &value
    }
    return nil
}

func OptionalString(value string) *string {
    if value != "" {
        return &value
    }
    return nil
}
```

注意 `OptionalNumber` 的判断是 `value > 0`，这意味着 `0` 和负数都会返回 `nil`。在 `settings_update.go:69-80` 中：

```go
userModificationRequest := &model.UserModificationRequest{
    EntriesPerPage:      model.OptionalNumber(settingsForm.EntriesPerPage),
    DefaultReadingSpeed: model.OptionalNumber(settingsForm.DefaultReadingSpeed),
    CJKReadingSpeed:     model.OptionalNumber(settingsForm.CJKReadingSpeed),
    MediaPlaybackRate:   model.OptionalNumber(settingsForm.MediaPlaybackRate),
}
```

如果用户在表单中填写 `0`，`OptionalNumber(0)` 返回 `nil`，`Patch()` 不会覆盖该字段——**这意味着通过 REST API 无法将数值型偏好设为 0**（虽然验证逻辑会拒绝 ≤0 的值，所以实际不会出问题，但这是一个潜在的语义缺陷）。

**更关键的是**：`KeyboardShortcuts` 等布尔字段**没有** `OptionalBool` 辅助函数（`model.go` 中不存在），因此在 `settings_update.go` 中这些字段**不通过** `UserModificationRequest` 传递，而是通过 `SettingsForm.Merge(user)` 直接覆盖到 User 对象。这导致 Web UI 表单提交是全量覆盖语义，无法实现部分更新。

### 10.6 JSONB + GIN 索引频繁更新的膨胀代价分析

虽然 Miniflux 的偏好字段不使用 JSONB + GIN，但我们可以结合代码库中 `web_sessions.state`（JSONB 但无索引）和 `entries.document_vectors`（有 GIN 索引的 tsvector）的实现模式，深入分析"如果 keymap 用 JSONB 存储并加 GIN 索引，频繁更新的代价"。

#### 10.6.1 现有代码中 JSONB 的更新模式：整体替换

`storage/web_session.go:183-196` 中 `UpdateWebSession` 的实现：

```go
query := `
    UPDATE
        web_sessions
    SET
        user_id=$2,
        state=$3    -- 整体替换整个 JSONB 字段
    WHERE
        id=$1
`

stateJSON, err := session.MarshalState()  // Go 层序列化为完整 JSON 字符串
_, err := s.db.Exec(query, session.ID, session.NullUserID(), stateJSON)
```

特点：
- **全量写入**：每次更新都是把整个 state 对象序列化为 JSON，整体替换 `state` 列
- **读-改-写在应用层**：先从 DB 读出 → 反序列化为 Go struct → 修改字段 → 序列化 → 写回
- **不使用 `jsonb_set` / `jsonb_insert`**：没有利用 PostgreSQL 原生的 JSONB 部分更新操作符
- 即使只改一个字段（如 flash message），也是整段 JSON 写回

这种模式下，如果有 GIN 索引，**每次更新都是删除旧索引条目 + 插入全部新索引条目**，代价是 O(n) 的索引重建（n 为 JSONB 内键值对数量）。

#### 10.6.2 GIN 索引的工作原理与写入代价

GIN（Generalized Inverted Index）是倒排索引，为每个被索引的元素（对于 JSONB 来说是每个 key 和每个 value）建立 posting list（行号列表）。

对于一个假想的 `keymap jsonb` 字段加 `CREATE INDEX idx_keymap ON users USING gin(keymap)`：

```json
{
  "navigate_down": "j",
  "navigate_up": "k",
  "open_original": "v",
  "toggle_star": "f"
}
```

GIN 索引内部大致是：

```
"navigate_down" → [row1, row5, row12, ...]
"j"             → [row1, row3, ...]
"navigate_up"   → [row1, ...]
"k"             → [row1, row2, ...]
...
```

**每次更新整个 JSONB 的索引代价**：
1. 旧 JSONB 的所有 key 和 value 对应的 posting list 都要删除该行
2. 新 JSONB 的所有 key 和 value 对应的 posting list 都要插入该行
3. 如果 key 数量是 n，每次更新涉及 2n 次索引操作

对比扁平列的 B-Tree 索引：
- 只更新变更的列，每次更新涉及 1 次索引操作（如果该列有索引）
- 偏好列基本都没有单独的索引（主键索引除外），几乎零额外代价

#### 10.6.3 GIN 索引膨胀问题

PostgreSQL 的 GIN 索引有两个与更新相关的膨胀机制：

1. **FASTUPDATE 待处理列表**
   - `gin_pending_list_limit` 控制一个内存中的 pending list（默认 4MB）
   - 快速更新先写入 pending list，不立即合并到主索引结构
   - 当 pending list 满了或 VACUUM 时才批量合并到主索引
   - Miniflux 代码中未设置任何 GIN 相关参数，使用默认值
   - 好处：写入快；坏处：查询时要扫 pending list，且积累到阈值才合并，会有突发延迟

2. **碎片与膨胀**
   - GIN 索引的 posting list 是按页存储的
   - 频繁删除+插入会导致 posting list 页碎片化，空间利用率降低
   - 需要定期 `VACUUM` 或 `REINDEX` 回收空间
   - `web_sessions` 表因为会话有过期清理（`CleanOldWebSessions`），如果 state 有 GIN 索引，删除会话也会产生索引碎片

#### 10.6.4 Miniflux 的实际选择：为什么偏好不用 JSONB

从代码架构可以看出设计考量：

| 考量维度 | 扁平列（现状） | JSONB + GIN（假设） |
|---------|--------------|-------------------|
| **写入模式** | 全列 UPDATE，但列数少（20+列），只修改实际变更的列（REST API 用 Patch 指针） | 必须整体替换或用 `jsonb_set`，整体替换 = 所有键都重写索引 |
| **查询模式** | 按主键查单行，`SELECT *` 扫一行，无额外索引代价 | 按主键查也是扫一行，但 JSONB 解析需要 CPU 时间；如果有键值条件查询才用 GIN |
| **条件查询需求** | 无。偏好从不作为 WHERE 条件过滤用户列表 | GIN 索引的价值在于 `WHERE preferences @> '{"keyboard_shortcuts": false}'` 这类查询，但 Miniflux 不需要 |
| **Schema 演进频率** | 低。偏好字段相对稳定，年级别才新增一个 | JSONB 无需迁移，但 Miniflux 偏好稳定，灵活性不是刚需 |
| **类型安全** | Go 结构体强类型 + PostgreSQL 列类型约束 | 运行时解析，类型靠应用层保证 |

**核心结论**：Miniflux 没有 keymap JSONB 字段，也没有 GIN 索引，因为偏好是**按主键读写的单行列数据**，不需要条件查询，GIN 索引用不上，反而会带来：
- 写入放大（每次更新触发 2n 次索引操作）
- 索引膨胀（频繁更新 + FASTUPDATE 合并）
- 类型不安全
- 查询性能无提升（仍然走主键索引）

### 10.7 VACUUM ANALYZE 触发频率：完全交给 PostgreSQL autovacuum

#### 10.7.1 代码事实：Miniflux 不主动触发 VACUUM

对全代码库搜索 `VACUUM`、`vacuum`、`ANALYZE`、`analyze`（排除 feed parser 语义）、`pg_stat`、`autovacuum` 的结果：
- **零个匹配**。Go 代码中不存在任何 `db.Exec("VACUUM")` 或 `db.Exec("ANALYZE")` 调用。
- `config/options.go` 的 60+ 个配置项中，没有任何与 VACUUM 频率、阈值、成本限制相关的参数。
- `internal/cli/cleanup_tasks.go` 的 `runCleanupTasks()` 只执行 4 项清理：旧 web sessions、归档已读条目、归档未读条目、清理孤立图标 —— **不包含数据库维护任务**。

#### 10.7.2 清理调度器的实际内容

`internal/cli/scheduler.go:27-30` 启动清理调度器：
```go
go cleanupScheduler(
    store,
    config.Opts.CleanupFrequency(),  // 默认 24 小时
)
```

`config/options.go:143-150` 的 `CLEANUP_FREQUENCY_HOURS` 定义：
```go
"CLEANUP_FREQUENCY_HOURS": {
    parsedDuration: time.Hour * 24,  // 默认 24h
    rawValue:       "24",
    valueType:      hourType,
    validator:      validateGreaterOrEqualThan(1, rawValue),  // ≥1h
},
```

这个频率控制的是**应用层业务数据清理**（条目归档、会话删除），不是 PostgreSQL 的 VACUUM。

运维侧可配置的清理参数完整清单（`config/options.go`）：

| 环境变量 | 默认值 | 用途 | 与 VACUUM 的关系 |
|---------|--------|------|----------------|
| `CLEANUP_FREQUENCY_HOURS` | 24 | 清理任务调度周期 | 无。只决定业务清理运行频率 |
| `CLEANUP_ARCHIVE_READ_DAYS` | 60 | 已读条目保留天数 | 间接：删除条目产生死元组，触发 autovacuum |
| `CLEANUP_ARCHIVE_UNREAD_DAYS` | 180 | 未读条目保留天数 | 同上 |
| `CLEANUP_ARCHIVE_BATCH_SIZE` | 10000 | 每次归档的条目数量上限 | 间接：批量大小影响死元组产生速率 |
| `CLEANUP_REMOVE_SESSIONS_DAYS` | 30 | web session 保留天数 | 同上 |

#### 10.7.3 VACUUM 完全依赖 PostgreSQL 原生 autovacuum

Miniflux 的运维文档（README / man page / Docker 配置）中没有任何 VACUUM 相关说明。运维侧实际配置 VACUUM 策略的方式是通过 **PostgreSQL 自身配置**，与 Miniflux 应用层无关：

```ini
# postgresql.conf 中的相关参数（运维侧配置，不在 Miniflux 中）
autovacuum = on                    # 默认开启
autovacuum_vacuum_threshold = 50   # 表死元组达到 50 条触发
autovacuum_vacuum_scale_factor = 0.2  # 表大小的 20% 作为附加阈值
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1  # 使用系统全局限制
```

这些参数完全由 DBA 在 PostgreSQL 配置文件或 `ALTER TABLE ... SET (...)` 中设置，Miniflux 应用层不暴露、不管理、也不感知。

#### 10.7.4 运维侧配置 vs 应用层配置的边界

```
┌─────────────────────────────────────────────────────────────┐
│                     运维可配置范围                            │
├──────────────────────────┬──────────────────────────────────┤
│  Miniflux 环境变量        │  PostgreSQL postgresql.conf      │
│  (config/options.go)      │  (DBA 直接操作数据库)            │
├──────────────────────────┼──────────────────────────────────┤
│  CLEANUP_FREQUENCY_HOURS  │  autovacuum                      │
│  CLEANUP_ARCHIVE_*_DAYS   │  autovacuum_vacuum_threshold    │
│  CLEANUP_ARCHIVE_BATCH    │  autovacuum_analyze_threshold   │
│  CLEANUP_REMOVE_SESSIONS  │  autovacuum_vacuum_cost_delay   │
│  ...                      │  gin_pending_list_limit         │
│                          │  work_mem / maintenance_work_mem │
└──────────────────────────┴──────────────────────────────────┘
```

**结论**：VACUUM ANALYZE 触发频率**完全没有暴露给 Miniflux 运维侧配置**，运维必须通过 PostgreSQL 原生参数进行管理。这符合"数据库职责归数据库"的设计哲学——应用层只管业务数据清理，DBMS 的内部维护由 DBA 负责。

### 10.8 autovacuum_naptime：SIGHUP 热重载，无需重启 PostgreSQL

#### 10.8.1 代码事实：Miniflux 完全不涉及此参数

全代码库搜索 `autovacuum_naptime`、`naptime`、`SIGHUP`、`pg_reload_conf`、`pg_reload_config` 的结果：
- **零个匹配**。Miniflux 代码中没有任何 PostgreSQL 参数管理逻辑，不调用 `SELECT pg_reload_conf()`，也不向 postmaster 发信号。

这是 PostgreSQL 内核层面的运维问题，以下分析基于 PostgreSQL 官方文档和通用运维经验，不涉及 Miniflux 代码。

#### 10.8.2 PostgreSQL 参数的三类上下文

PostgreSQL 的所有配置参数按修改方式分为三类（可通过 `SELECT name, context FROM pg_settings` 查看）：

| context | 修改方式 | 是否需要重启 | 示例参数 |
|---------|---------|------------|---------|
| `internal` | 编译时确定，不可改 | — | `server_version` |
| `postmaster` | 修改配置文件后需重启 postmaster | **是** | `shared_buffers`、`port`、`max_connections` |
| `sighup` | 修改配置文件后发 SIGHUP 即可 | **否** | `autovacuum_naptime`、`log_*`、`work_mem` |
| `backend` / `user` / `superuser` | 会话级，`SET` 命令即时生效 | 否 | `search_path`、`statement_timeout` |

`autovacuum_naptime` 的 context 是 **`sighup`**，因此**修改后不需要重启 PostgreSQL 进程**。

#### 10.8.3 三种热重载方式

运维侧修改 `postgresql.conf` 中的 `autovacuum_naptime = 2min`（默认 1min）后，可通过以下任一方式生效：

1. **shell 发信号**：
   ```bash
   kill -HUP <postgres_pid>
   # 或
   pg_ctl reload -D /path/to/data
   ```

2. **SQL 函数调用**：
   ```sql
   SELECT pg_reload_conf();  -- 需要 superuser 权限
   ```

3. **systemd 服务管理**：
   ```bash
   systemctl reload postgresql
   # 或
   service postgresql reload
   ```

验证是否生效：
```sql
SELECT name, setting, pending_restart
FROM pg_settings
WHERE name = 'autovacuum_naptime';
-- pending_restart 为 false 表示已生效
```

#### 10.8.4 可能误以为需要重启的原因

容易混淆的是 PostgreSQL 的**共享内存参数**（如 `shared_buffers`、`max_connections`）确实需要重启，因为它们影响 postmaster 启动时分配的共享内存段大小。但 `autovacuum_naptime` 只是 autovacuum launcher 进程的**轮询间隔**，不涉及共享内存布局，所以可以热重载。

#### 10.8.5 autovacuum_naptime 的实际影响范围

`autovacuum_naptime` 控制 autovacuum launcher 多久醒来检查一次哪些表需要清理（默认 1 分钟）。与 Miniflux 相关的实际影响：

| 参数 | 默认值 | Miniflux 相关场景 |
|------|--------|-----------------|
| `autovacuum_naptime` | 1min | 控制 `entries`、`entry_tombstones`、`web_sessions` 表死元组被清理的检查频率 |
| `autovacuum_vacuum_threshold` | 50 | 单表死元组达到 50 条才触发 VACUUM（与 scale_factor 叠加） |
| `autovacuum_vacuum_scale_factor` | 0.2 | 表大小的 20% 附加阈值 |
| `autovacuum_vacuum_cost_delay` | 2ms | 每次 I/O 后的休眠时间，控制对查询性能的影响 |

Miniflux 的 `CLEANUP_FREQUENCY_HOURS`（默认 24h）产生批量删除后，autovacuum 会在下一次 naptime 周期内感知到死元组并开始清理。如果 naptime 设为 10min，最坏情况下需要等 10min 才开始回收空间。

#### 10.8.6 与 Miniflux 的边界关系

```
Miniflux 应用层                          PostgreSQL 内核
───────────────                         ──────────────
CLEANUP_ARCHIVE_READ_DAYS = 60    ──┐
CLEANUP_ARCHIVE_UNREAD_DAYS = 180  ──┼─►  DELETE 产生死元组
CLEANUP_FREQUENCY_HOURS = 24       ──┘
                                       │
                                       ▼
                                  autovacuum 检测死元组
                                  (由 autovacuum_naptime 控制检查频率)
                                       │
                                       ▼
                                  VACUUM 回收空间 + 更新统计信息
                                  (无需重启 PostgreSQL，SIGHUP 可热调)
```

**结论**：`autovacuum_naptime` 调整**不需要重启 PostgreSQL**，通过 SIGHUP 或 `pg_reload_conf()` 即可热重载。此参数完全在 PostgreSQL 运维层面管理，Miniflux 应用层不感知也不参与。

---

## 十一、前端启动时 default 与 user 偏好合并的优先级覆盖策略

### 11.1 三层默认值体系

Miniflux 的偏好默认值分布在三个层级，优先级从低到高：

```
1. PostgreSQL 列默认值     (DDL: DEFAULT 't' / DEFAULT 'en_US' / DEFAULT 'UTC')
2. Go 代码常量             (WebSession: defaultSessionLanguage / defaultSessionTheme)
3. 用户显式设置值          (users 表中的实际数据)
```

### 11.2 PostgreSQL 层默认值（最低优先级）

`migrations.go:303`：
```sql
ALTER TABLE users ADD COLUMN keyboard_shortcuts boolean default 't'
```

新用户创建时，`keyboard_shortcuts` 列默认为 `true`。其他偏好的 DB 默认值包括：
- `language` → `'en_US'`
- `timezone` → `'UTC'`
- `theme` → `'light_serif'`（后续迁移从 `'default'` 更新而来）
- `entries_per_page` → `100`
- `display_mode` → `'standalone'`

`CreateUser` 的 SQL 使用 `RETURNING` 子句读回所有列值（含默认值），所以 Go 层拿到的 `User` 对象已经携带了 DB 默认值，不存在 Go 零值问题。

### 11.3 Go 层默认值（Session 未绑定用户时的回退）

`web_session.go:20-21`：
```go
const (
    defaultSessionLanguage = "en_US"
    defaultSessionTheme    = "system_serif"
)
```

`Language()` 和 `Theme()` 方法在 session 未绑定用户时提供回退值：

```go
func (s *WebSession) Language() string {
    if s.state.Language != "" {
        return s.state.Language
    }
    return defaultSessionLanguage
}
```

**注意两层默认值的不一致**：
- DB 的 theme 默认是 `"light_serif"`
- Session 的 theme 默认是 `"system_serif"`
- 登录前（匿名 session）用户看到 `"system_serif"` 主题
- 登录后 `SetUser()` 将 user.Theme 复制到 session，覆盖为 `"light_serif"`（如果用户未修改过）

### 11.4 `SetUser()` 的优先级覆盖策略

`web_session.go:244-254`：

```go
func (s *WebSession) SetUser(user *User) {
    if user == nil {
        return
    }
    s.dirty = true
    userID := user.ID
    s.userID = &userID
    s.state.Language = user.Language
    s.state.Theme = user.Theme
}
```

**无条件覆盖**：不管 session 中原有的 `Language`/`Theme` 是什么，直接用 user 表的值覆盖。这意味着：

1. **登录瞬间**：匿名 session 的语言/主题偏好被丢弃，完全以 user 表为准
2. **设置保存后**：`settings_update.go:94-96` 中再次调用 `sess.SetUser(user)`，将更新后的 user 偏好同步到 session，保证重定向后的页面使用新偏好
3. **只有 Language 和 Theme 被同步到 session**：其他偏好（如 `KeyboardShortcuts`、`MarkReadOnView`）不经过 session，每次都通过模板渲染从 user 对象直接注入 HTML

### 11.5 前端 JS 层的"不存在即默认"策略

前端 **没有任何 JavaScript 变量存储默认偏好值**。偏好的"默认行为"完全依赖 HTML 属性的**有/无**来区分：

**`keyboard_shortcuts`**（`layout.html:55`）：
```html
{{ if not .user.KeyboardShortcuts }}data-disable-keyboard-shortcuts="true"{{ end }}
```
- `KeyboardShortcuts = true` → 属性**不存在** → JS 初始化快捷键（默认行为=启用）
- `KeyboardShortcuts = false` → 属性**存在** → JS 跳过初始化

**`mark_read_on_view`**（`layout.html:56`）：
```html
data-mark-as-read-on-view="{{ if .user.MarkReadOnView }}true{{ else }}false{{ end }}"
```
- 这里**总是输出属性**，用 `"true"` / `"false"` 字符串区分
- JS 端（`app.js:1288`）：`document.body.dataset.markAsReadOnView === "true"`

**两种策略的取舍**：

| 策略 | 用于 | 优点 | 缺点 |
|------|------|------|------|
| 有/无属性 | `keyboard_shortcuts` | 属性缺失 = 默认启用，减少 HTML 体积 | 前端只能感知"禁用"，无法区分"未登录"和"启用" |
| true/false 字符串 | `mark_read_on_view` | 语义明确，前端可直接判断 | 每次都输出属性 |

Miniflux 选择对 `keyboard_shortcuts` 使用"有/无"策略，因为键盘快捷键默认启用的设计意图与"属性不存在=不限制"天然对齐。

### 11.6 完整偏好传递优先级图

```
用户打开页面
    │
    ▼
session 已认证？
    ├─ 否 → 使用 session 默认值 (language=en_US, theme=system_serif)
    │        其他偏好不可用（未登录页面不注入 data-*）
    │
    └─ 是 → 从 users 表读取完整偏好
              │
              ├─ 语言/主题 → 同步写入 session.state (SetUser)
              │                → view.New() 从 session 读取 language/theme
              │                → 模板渲染使用 session 值
              │
              └─ 其他偏好 → 模板渲染直接从 .user 读取
                             → 输出为 <body data-*> 属性
                             → JS 读取 data-* 或检测属性是否存在
```

**关键洞察**：语言和主题是唯一通过 session 中转的偏好，因为它们影响**所有页面的 CSS 和布局渲染**（在 view 初始化阶段就需要），而其他偏好只在具体交互时才被 JS 读取。这就是为什么 `SetUser()` 只同步 `Language` 和 `Theme` 两个字段到 session。

### 11.7 浅合并策略：删除 default 键时回退还是真删？

这个问题需要在三个不同层面分别分析，因为 Miniflux 存在三种"合并/默认"机制，语义完全不同。

#### 11.7.1 层面一：`UserModificationRequest.Patch()` —— nil = 不修改，不是"删除回退"

`model/user.go:90-202` 的 `Patch()` 方法是典型的"指针 = 是否修改"模式：

```go
if u.KeyboardShortcuts != nil {
    user.KeyboardShortcuts = *u.KeyboardShortcuts
}
```

**语义**：
- `*bool = nil` → 跳过这个字段，**保持原值不变**
- `*bool = &false` → 设为 false
- `*bool = &true` → 设为 true

**这里没有"删除回退到默认值"的概念**。nil 指针的含义是"调用者不想改这个字段"，而不是"调用者想把这个字段恢复成默认值"。

举例：
- 用户当前 `KeyboardShortcuts = false`
- API 调用 `PUT /v1/users/{id}` 传 `{}`（空 JSON）
- 反序列化后 `u.KeyboardShortcuts == nil`
- `Patch()` 跳过该字段
- 结果：`KeyboardShortcuts` 仍然是 `false`，**没有回退到默认的 true**

如果想"恢复默认值"，调用方必须**显式知道默认值是什么**，然后传默认值进去。后端不提供"重置为默认"的语义。

#### 11.7.2 层面二：`webSessionState` JSONB —— `omitempty` 省略 ≠ 删除回退

`model/web_session.go:37-46` 的 `webSessionState` 结构体：

```go
type webSessionState struct {
    CSRF               string                `json:"csrf,omitempty"`
    SuccessMessage     string                `json:"success_message,omitempty"`
    ErrorMessage       string                `json:"error_message,omitempty"`
    OAuth2             *WebSessionOAuth2     `json:"oauth2,omitempty"`
    Language           string                `json:"language,omitempty"`
    Theme              string                `json:"theme,omitempty"`
}
```

`omitempty` 的行为：
- 序列化时：零值字段（空字符串、nil 指针、0）不出现在 JSON 中
- 反序列化时：JSON 中不存在的字段，Go struct 对应字段就是**零值**（空字符串、nil 等）

**关键区分**：序列化时被省略 ≠ 数据被删除。

举个完整生命周期：
1. `NewWebSession()` 创建 session，`state.CSRF = rand.Text()`（非空）
2. 序列化为 JSON：`{"csrf":"abc123",...}`
3. 存入 `web_sessions.state` 列
4. 下次请求读出 → 反序列化 → `state.CSRF == "abc123"`（正常保留）
5. 如果某处代码把 `state.CSRF = ""`（清空）
6. 序列化后 JSON 中不再有 `csrf` 键
7. 写回 DB，DB 中的 JSONB 也没了 `csrf` 键
8. 下次读出 → 反序列化 → `state.CSRF == ""`（零值）

**这是"清零"，不是"回退默认"**。`csrf` 没了就是没了，不会自动恢复成新的随机值。

那"默认值"从哪来？答案是**`Language()` 和 `Theme()` getter 方法**：

```go
func (s *WebSession) Theme() string {
    if s.state.Theme != "" {
        return s.state.Theme
    }
    return defaultSessionTheme  // "system_serif"
}
```

所以回退逻辑是 **Get 时的 fallback**，不是 Set/Delete 时的自动恢复：
- `state.Theme = ""` → JSON 中 theme 键消失 → 下次读 `Theme()` 返回 `defaultSessionTheme`
- 看起来像是"删除后回退到默认"，但实际上是 getter 在零值时返回了默认常量
- **DB 层面的数据确实被删了（变成空串/omitted）**，但行为层面前端感知到的是"回退到默认"

| 字段 | 有 getter 方法 | 零值时行为 |
|------|--------------|-----------|
| `Language` | 有 → `Language()` | 空串 → 返回 `en_US`（看起来像回退） |
| `Theme` | 有 → `Theme()` | 空串 → 返回 `system_serif`（看起来像回退） |
| `CSRF` | 无（直接读字段） | 空串 → 就是空串，不回退 |
| `SuccessMessage` | 无 | 空串 → 就是空串 |
| `OAuth2` | 无 | nil → 就是 nil |

#### 11.7.3 层面三：前端 data-* 属性 —— "属性不存在"即默认

前端层面的"删除/回退"是最宽松的：

1. **`keyboard_shortcuts`（有/无策略）**
   - 属性不存在 → 启用快捷键（默认行为）
   - 属性存在且为 `"true"` → 禁用
   - 从"有"变"无" = 从禁用恢复为启用 = "回退默认"

2. **`mark_read_on_view`（true/false 字符串）**
   - 属性一定存在，值为 `"true"` 或 `"false"`
   - 没有"不存在即默认"的语义
   - 想回退默认必须显式传默认值（`false`）

但注意：**前端属性的"有/无"完全由后端模板控制**，前端自身不能"删除偏好"。用户不能在浏览器里改个什么东西就让 data 属性消失——偏好的增删改都是服务端渲染时决定的。

#### 11.7.4 三层模型对比总结

| 层面 | 存储模型 | "删除"操作 | 删除后行为 | 是真删还是回退？ |
|------|---------|-----------|-----------|----------------|
| `users` 扁平列 | 每偏好一列，DB 有 DEFAULT | 不存在"删除列值"的概念，只能 UPDATE 为某个值 | 列值保持不变或被覆盖为新值 | 没有删除语义，只有覆盖语义 |
| `web_sessions.state` JSONB | key-value JSONB 对象 | 设为零值 → `omitempty` 序列化时省略键 | DB 中键确实消失了，但 getter 可能返回默认值 | **DB 层面真删**，但 getter 层有回退 fallback |
| 前端 `data-*` 属性 | HTML 属性 | 模板不输出该属性 | 前端按默认行为运行 | 是"不注入"，前端用默认逻辑，本质是回退 |

**最容易产生误解的是 webSessionState 层**：`omitempty` + getter fallback 的组合，让"设零值"这个操作在 DB 层面表现为"删除键"，在行为层面表现为"回退默认"，容易让人以为有一套自动的"删除键→回退默认"机制。实际上是两个独立机制碰巧叠在了一起：序列化时的省略规则 + 读取时的零值回退。

### 11.8 keymap_tombstone：不存在该概念，但 entry_tombstones 有无限增长隐患

#### 11.8.1 代码事实：没有 keymap_tombstone JSONB 数组

对全代码库搜索 `keymap_tombstone`、`key_map_tomb`、`tombstone`、`graveyard` 的结果：
- **`keymap_tombstone` 零匹配**。不存在任何与快捷键/keymap 相关的墓碑表或 JSONB 数组。
- 唯一实际存在的墓碑机制是 `entry_tombstones` 表（`internal/database/migrations.go:1472-1480`），用于**已删除 RSS 条目的去重防护**，与键盘快捷键完全无关。

由于用户提到的是"keymap_tombstone JSONB 数组"，这是一个与实际代码不匹配的假设概念。但我们可以顺着代码中唯一存在的 `entry_tombstones` 表来分析"墓碑无限增长"这个通用问题。

#### 11.8.2 entry_tombstones 的表结构与写入路径

`migrations.go:1472-1480` 定义：
```sql
CREATE TABLE entry_tombstones (
    feed_id bigint not null references feeds(id) on delete cascade,
    hash text not null check (hash <> ''),
    deleted_at timestamp with time zone not null default now(),
    primary key (feed_id, hash)
);

CREATE INDEX entry_tombstones_deleted_at_idx
    ON entry_tombstones (deleted_at);
```

写入发生在两处（`storage/entry.go`）：
1. **`ArchiveEntries()`（第 362-404 行）**：按时间归档旧条目时，将被删除的 (feed_id, hash) 批量插入墓碑表，`ON CONFLICT (feed_id, hash) DO NOTHING` 幂等
2. **`FlushHistory()`（第 491-501 行）**：用户手动清空历史时同样写入墓碑表

#### 11.8.3 无限增长隐患：无清理策略

对 `entry_tombstones` 相关代码的完整扫描结果（全代码库 8 处引用）：

| 文件:行号 | 操作 | 是否清理 |
|----------|------|---------|
| `storage/entry.go:118` | `createEntry` WHERE NOT EXISTS 检查 | 否 |
| `storage/entry.go:287` | `IsNewEntry` EXISTS 检查 | 否 |
| `storage/entry.go:386` | `ArchiveEntries` INSERT | 否 |
| `storage/entry.go:499` | `FlushHistory` INSERT | 否 |
| `database/migrations.go:1472-1495` | DDL + 初始数据迁移 | 否 |
| `cli/cleanup_tasks.go` | 全文件 4 项清理任务 | **无 entry_tombstones 清理** |

**`entry_tombstones` 表没有任何清理代码**。虽然表上有 `entry_tombstones_deleted_at_idx` 索引（暗示设计时考虑过按时间清理），但实际没有任何 DELETE 语句使用这个索引。

增长速率估算：
- RSS 订阅源每篇文章产生一条墓碑记录
- 一个中等活跃用户 100 个订阅源 × 每源每天 5 篇 × 365 天 = 每年 ~18 万条
- 多用户实例 N×M 增长
- 每条记录约 40 字节（bigint + text hash + timestamp），18 万条约 7 MB/年/用户，单表增长不算快但永久不清理

#### 11.8.4 唯一隐式清理路径：ON DELETE CASCADE

表定义 `feed_id bigint not null references feeds(id) on delete cascade` 意味着：
- **删除订阅源时**，该 feed 的所有墓碑记录级联删除
- 这是唯一能让 entry_tombstones 缩小的机制
- 用户保留的订阅源越多，墓碑积累越永久

#### 11.8.5 如果 keymap 真用 JSONB 数组存储墓碑，该怎么设计？

作为架构对比分析，假设存在 `keymap_tombstones jsonb` 存储已删除的快捷键映射（`"Ctrl+Shift+K"`, `"Alt+F"` 等），需要解决的清理问题包括：

| 维度 | entry_tombstones 现状 | keymap_tombstone JSONB 假设 |
|------|----------------------|--------------------------|
| 结构 | 独立表 + 主键 | JSONB 数组内嵌在 users 表 |
| 上限 | 无（行数无限） | 数组长度无检查 → 单行无限膨胀 |
| 写入 | INSERT + ON CONFLICT | `jsonb_set(keymap_tombstones, ...)` 每次全量写回 |
| 按时间清理 | 有 deleted_at 索引但无清理逻辑 | 需在 JSON 内存储删除时间戳，查询效率低 |
| 级联删除 | ON DELETE CASCADE (feed) | 随用户删除自动清理 |
| 去重 | PRIMARY KEY (feed_id, hash) | JSONB 数组需应用层去重 |

**keymap_tombstone JSONB 数组比 entry_tombstones 独立表更差**：
- JSONB 数组不能用 PRIMARY KEY 去重，需应用层检查 `?` 操作符
- 无上限更严重：单个 users 行的 JSONB 数组膨胀会拖慢整行读取（SELECT *）
- 按时间清理需要解析每一个数组元素的时间戳，GIN 索引也帮不上忙
- PostgreSQL TOAST 存储在单行 JSONB 超过 ~2KB 时触发，进一步降低读取性能

#### 11.8.6 总结

| 问题 | 代码事实 |
|------|---------|
| keymap_tombstone 字段 | **不存在**。代码中无 keymap、无 tombstone JSONB 数组 |
| 唯一的墓碑机制 | `entry_tombstones` 表，用于 RSS 条目去重，与键盘无关 |
| entry_tombstones 的清理 | **无清理策略**。无上限增长，仅通过删除订阅源级联回收 |
| deleted_at 索引 | 存在但未被使用（无 DELETE ... WHERE deleted_at < ...） |
| 用 JSONB 数组存墓碑 | 比独立表更差：无法主键去重、单行膨胀、时间清理低效 |

**设计洞察**：Miniflux 在 entry_tombstones 上选择独立表而非 JSONB 数组是正确的架构决策（有主键去重、有级联删除、有时间索引）。但**遗漏了定期按时间清理的代码**——这是一个真实的、虽然缓慢的存储泄漏。

### 11.9 max_tombstones=50：不存在此配置，无 LRU / FIFO 淘汰算法

#### 11.9.1 代码事实：全代码库无 max_tombstones、无 LRU、无 FIFO

对全代码库的搜索结果：

| 关键词 | 匹配数 | 匹配位置 |
|--------|--------|---------|
| `max_tombstone` / `MAX_TOMBSTONE` | 0 | 无 |
| `tombstone.*limit` / `limit.*tombstone` | 0 | 无 |
| `LRU` / `lru` | 0 | 无 |
| `FIFO` / `fifo` | 0 | 无 |
| `evict` / `eviction` | 0 | 无 |
| `least.*recent` / `first.*in` | 0 | 无 |
| 数字 `50` + tombstone 同文件 | 0 | 无 |

**`max_tombstones=50` 在 Miniflux 代码中完全不存在。** 没有这个常量、没有这个配置项、也没有任何基于这个阈值的清理逻辑。

#### 11.9.2 实际情况：entry_tombstones 无上限、无淘汰、无清理

如 §11.8 已分析，`entry_tombstones` 表的写入发生在两处：
1. `ArchiveEntries()`（`storage/entry.go:362-404`）：定时归档旧条目时批量 INSERT
2. `FlushHistory()`（`storage/entry.go:491-501`）：用户手动清空历史时批量 INSERT

两处写入都使用 `ON CONFLICT (feed_id, hash) DO NOTHING` 幂等插入。**没有任何代码在写入前检查表大小，也没有任何代码在写入后触发清理。**

实际增长行为：
```
时间线：
T0: 表空
T1: 第一次归档 → 插入 N1 行 (N1 ≤ CLEANUP_ARCHIVE_BATCH_SIZE, 默认 10000)
T2: 第二次归档 → 插入 N2 行 (可能重复，ON CONFLICT 跳过)
T3: 第三次归档 → 插入 N3 行
...
→ 行数无限单调增长，无回落
唯一缩小路径：用户删除 feed → ON DELETE CASCADE 级联删除该 feed 的所有墓碑
```

#### 11.9.3 如果真的要实现 max_tombstones=50，LRU vs FIFO 该怎么选？

作为架构分析，假设存在一个 `max_tombstones_per_feed = 50` 的配置需求，比较两种淘汰策略：

| 维度 | FIFO（先进先出） | LRU（最近最少使用） |
|------|----------------|------------------|
| 实现复杂度 | 低。只需 `deleted_at` 时间戳，删除最旧的 N 行 | 高。每次检查命中时需要更新"最后使用时间" |
| 额外存储 | 不需要额外列（已有 `deleted_at`） | 需要额外列 `last_accessed_at` 或维护独立访问计数 |
| PostgreSQL 实现 | `DELETE FROM entry_tombstones WHERE feed_id=$1 ORDER BY deleted_at ASC LIMIT (count - 50)` | 每次 `IsNewEntry()` 检查命中时需 `UPDATE entry_tombstones SET last_accessed_at=now() WHERE ...`，淘汰时 `ORDER BY last_accessed_at ASC` |
| 写入开销 | 清理时一次批量 DELETE | 每次存在性检查都产生一次 UPDATE（热路径） |
| 语义合理性 | RSS 条目天然按时间线性产生，最早的最可能已失效 | RSS 条目不会被"反复使用"，墓碑只用于一次性去重，LRU 的"最近使用"语义没有意义 |

**对于 RSS 条目墓碑场景，FIFO 显然优于 LRU**：
- RSS 条目是时间敏感的：一篇 3 年前被删除的文章，RSS 源几乎不可能再推送同样内容
- 墓碑的唯一用途是"防止被重新 ingestion"，一旦条目超过源站历史保留期（通常 30-180 天），墓碑就失去价值
- 墓碑不存在"被使用"的概念——只在插入新条目时做一次存在性检查，检查命中与否都不需要更新墓碑状态
- LRU 的 `last_accessed_at` 更新会放大 `IsNewEntry()` 这个热路径的写入开销

#### 11.9.4 为什么 Miniflux 不需要 max_tombstones

从实际数据量来看：

```
单用户场景估算：
- 100 个订阅源 × 每源每天 5 篇 × 365 天 = 182,500 篇/年
- 约 80% 最终被归档产生墓碑 → ~146,000 条墓碑/年
- 每条约 40 字节 (bigint 8 + text hash 24 + timestamp 8)
- 年增长约 5.8 MB，索引另算约同量级
- 10 年增长约 58 MB，对现代 PostgreSQL 实例完全可忽略
```

对比：一张 `entries` 表存活跃条目（默认已读 60 天 + 未读 180 天）的体量通常是墓碑表的数倍到数十倍。**墓碑表的存储开销在实际使用中可以忽略不计**，这可能是 Miniflux 没有实现清理和淘汰的根本原因——没有足够的收益值得投入工程成本。

#### 11.9.5 三类"有上限清理"在代码库中的对比

虽然没有 max_tombstones，但 Miniflux 确实存在其他"有上限批量处理"的模式：

| 机制 | 上限常量 | 清理策略 | 位置 |
|------|---------|---------|------|
| 条目归档批量 | `CLEANUP_ARCHIVE_BATCH_SIZE` (默认 10000) | 按时间窗口，每次最多处理 N 条，不是淘汰而是分批处理 | `config/options.go:125-132` |
| Feed 刷新批量 | `BATCH_SIZE` (默认 100) | 每次从待刷新队列取 N 条 | `config/options.go:107-114` |
| 连接池上限 | `DATABASE_MAX_CONNS` (默认 20) | Go `database/sql` 内置连接池管理（非 LRU/FIFO，空闲连接按时间复用） | `config/options.go:169-176` |
| **entry_tombstones** | **不存在** | **无** | — |

这三类上限都是"批处理大小"或"资源池容量"，与墓碑表的"总量淘汰"问题不属于同一类。

#### 11.9.6 总结

| 问题 | 代码事实 |
|------|---------|
| `max_tombstones=50` | **不存在**。无此常量、无此配置、无此逻辑 |
| LRU 淘汰算法 | 不存在。代码中零 LRU 相关引用 |
| FIFO 淘汰算法 | 不存在。代码中零 FIFO 相关引用 |
| entry_tombstones 实际行为 | 无限增长，仅依赖删除 feed 的 `ON DELETE CASCADE` 间接清理 |
| 如果要实现淘汰 | FIFO 比 LRU 更适合 RSS 场景（墓碑无"最近使用"语义、实现简单、无热路径写入） |
| 为什么没实现 | 数据量太小（年增长约 6 MB/用户），工程投入无收益 |
