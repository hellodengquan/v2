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
