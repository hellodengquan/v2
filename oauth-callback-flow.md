# OAuth2 回调流程分析

> 基于代码路径 `internal/ui/oauth2_*.go`、`internal/oauth2/`、`internal/model/web_session.go` 整理。

---

## 1. 端到端流程概览

```
浏览器 → GET /oauth2/{provider}/redirect
                ↓
        生成 state + PKCE code_verifier
        存入 WebSession.OAuth2
        302 → Provider 授权页
                ↓
        用户在 Provider 侧授权
                ↓
        Provider 回调 → GET /oauth2/{provider}/callback?code=…&state=…
                ↓
        校验 state → 用 code + code_verifier 换 token → 获取 profile
                ↓
        ┌─ 已登录用户 → 账号绑定
        └─ 未登录用户 → 查找/创建用户 → 建立会话
```

三条路由注册在 `internal/ui/ui.go:152-154`：

| 方法 | 路径 | Handler | 说明 |
|------|------|---------|------|
| GET | `/oauth2/{provider}/redirect` | `oauth2Redirect` | 发起授权 |
| GET | `/oauth2/{provider}/callback` | `oauth2Callback` | 回调处理 |
| POST | `/oauth2/{provider}/unlink` | `oauth2Unlink` | 解除绑定 |

> **公共路由**：`/oauth2/*` 的 redirect 和 callback 路径被标记为 public（`routes.go:45`），无需认证即可访问——这是必要的，因为回调时用户可能尚未登录。

---

## 2. State 校验机制

### 2.1 生成阶段 — `oauth2Redirect`

`internal/ui/oauth2_redirect.go:33-35`：

```go
auth := oauth2.GenerateAuthorization(authProvider.Config())
request.WebSession(r).StartOAuth2Flow(auth.State(), auth.CodeVerifier())
response.HTMLRedirect(w, r, auth.RedirectURL())
```

`GenerateAuthorization`（`internal/oauth2/authorization.go`）执行：

1. 生成 32 字节随机 `codeVerifier`（PKCE）
2. 计算 `SHA-256(codeVerifier)` → `codeChallenge`（S256 方式）
3. 生成 24 字节随机 `state`
4. 调用 `oauth2.Config.AuthCodeURL(state, code_challenge_method=S256, code_challenge=…)` 构造授权 URL

**存储**：`StartOAuth2Flow` 将 `state` 和 `codeVerifier` 写入 `WebSession.state.OAuth2`（`web_session.go:229-234`），以 JSON 持久化在 `web_sessions` 表中。

### 2.2 校验阶段 — `oauth2Callback`

`internal/ui/oauth2_callback.go:39-47`：

```go
state := request.QueryStringParam(r, "state", "")
expectedState := sess.OAuth2State()
if expectedState == "" || subtle.ConstantTimeCompare([]byte(state), []byte(expectedState)) == 0 {
    slog.Warn("Invalid OAuth2 state value received", ...)
    response.HTMLRedirect(w, r, h.routePath("/"))
    return
}
```

关键安全属性：

| 属性 | 说明 |
|------|------|
| **常量时间比较** | 使用 `crypto/subtle.ConstantTimeCompare`，防止时序侧信道攻击 |
| **空值拒绝** | `expectedState == ""` 时直接拒绝——防止未发起 OAuth 流程的请求通过 |
| **一次性消费** | 校验通过后立即调用 `sess.ClearOAuth2Flow()`，清除 session 中的 state 和 code_verifier，防止重放 |

### 2.3 PKCE 校验

state 校验之后，`codeVerifier` 被从 session 中取出用于 token 交换（`oauth2_callback.go:49`）：

```go
codeVerifier := sess.OAuth2CodeVerifier()
sess.ClearOAuth2Flow()   // 一次性消费
```

在 `Profile()` 中：

```go
token, err := conf.Exchange(ctx, code, oauth2.SetAuthURLParam("code_verifier", codeVerifier))
```

Provider 会验证 `SHA-256(codeVerifier)` 是否等于授权请求时传的 `code_challenge`，防止授权码拦截攻击。

---

## 3. 账号绑定逻辑

回调处理根据当前会话是否已认证，分两条路径：

### 3.1 已登录用户 → 绑定

`oauth2_callback.go:60-108`，当 `request.IsAuthenticated(r)` 为 true 时：

```
已登录用户点击"关联账号"
        ↓
获取 loggedUser
        ↓
┌─ 边界检查 1: AnotherUserWithFieldExists
│   该 OAuth profile 是否已被另一个用户绑定？
│   → 是: error.duplicate_linked_account, 重定向 /settings
│
├─ 边界检查 2: existingProfileID != "" && existingProfileID != profile.ID
│   当前用户是否已绑定该 provider 的另一个 identity？
│   → 是: error.duplicate_linked_account, 重定向 /settings
│
└─ 通过检查
    → PopulateUserWithProfileID(loggedUser, profile)
    → store.UpdateUser(loggedUser)
    → alert.account_linked, 重定向 /settings
```

#### 边界检查 1 详解：`AnotherUserWithFieldExists`

`internal/storage/user.go:474-479`：

```sql
SELECT true FROM users WHERE id <> $1 AND {field} = $2 LIMIT 1
```

- `field` = `profile.Key`，即 provider 对应的数据库字段（`google_id` 或 `openid_connect_id`）
- `value` = `profile.ID`，即 OAuth 提供商返回的用户唯一标识
- 含义：**排除自身后，是否还有其他用户已绑定了同一个 OAuth identity**？

#### 边界检查 2 详解：`UserProfileID` 比对

```go
existingProfileID := authProvider.UserProfileID(loggedUser)
if existingProfileID != "" && existingProfileID != profile.ID
```

- `UserProfileID` 读取当前用户在该 provider 字段上的值
- 如果用户已绑定了该 provider 的一个 identity（`existingProfileID != ""`），但回调返回的 identity 不同，则拒绝
- 即：**一个用户在同一 provider 上只能绑定一个 identity，不允许切换**

### 3.2 未登录用户 → 登录/注册

`oauth2_callback.go:110-154`：

```
未登录用户回调
        ↓
store.UserByField(profile.Key, profile.ID) 查找用户
        ↓
┌─ 用户存在 → 直接登录
│   → SetLastLogin + authenticateWebSession
│   → 重定向到用户默认首页
│
└─ 用户不存在 → 检查是否允许自动创建
    ├─ OAUTH2_USER_CREATION != 1
    │   → 403 Forbidden
    │
    ├─ 用户名已存在 (store.UserExists)
    │   → 400 error.user_already_exists
    │
    └─ 允许创建
        → CreateUser + SetLastLogin + authenticateWebSession
        → 重定向到用户默认首页
```

#### 会话建立细节 — `authenticateWebSession`

`internal/ui/auth.go:22-33`：

```go
func authenticateWebSession(w, r, store, user) error {
    session.SetUser(user)           // 绑定 userID + 复制语言/主题偏好
    oldID, secret := session.Rotate()  // 轮换 session ID 和 secret（防 session fixation）
    store.RotateWebSession(oldID, session)
    setSessionCookie(w, session, secret)
}
```

**安全要点**：认证成功后立即轮换 session 标识符（`Rotate`），使旧的 session cookie 失效，防止 session fixation 攻击。

---

## 4. 重复登录边界处理

### 4.1 场景矩阵

| 场景 | 条件 | 结果 |
|------|------|------|
| A. 已登录 + 新 OAuth identity，无冲突 | 另一用户未绑定该 identity，且当前用户未绑定该 provider | ✅ 绑定成功 |
| B. 已登录 + OAuth identity 已被他人绑定 | `AnotherUserWithFieldExists` = true | ❌ `error.duplicate_linked_account` |
| C. 已登录 + 当前用户已绑定该 provider 的另一个 identity | `existingProfileID != "" && != profile.ID` | ❌ `error.duplicate_linked_account` |
| D. 已登录 + 当前用户已绑定同一 identity（重复点击绑定） | `existingProfileID == profile.ID` | ✅ 幂等通过，再次写入同一值 |
| E. 未登录 + OAuth identity 对应用户存在 | `UserByField` 找到用户 | ✅ 登录，更新 last_login |
| F. 未登录 + OAuth identity 不存在 + 允许创建 + 用户名不冲突 | `OAUTH2_USER_CREATION=1` 且 `UserExists=false` | ✅ 创建用户并登录 |
| G. 未登录 + OAuth identity 不存在 + 不允许创建 | `OAUTH2_USER_CREATION != 1` | ❌ 403 Forbidden |
| H. 未登录 + OAuth identity 不存在 + 用户名冲突 | `UserExists=true` | ❌ 400 `error.user_already_exists` |

### 4.2 场景 D 详解：幂等绑定

这是容易忽略的边界情况。当已登录用户重复点击"关联账号"按钮时：

1. `AnotherUserWithFieldExists(loggedUser.ID, profile.Key, profile.ID)` → **false**（因为 `id <> $1` 排除了自身）
2. `existingProfileID == profile.ID` → 两个条件都不满足（`existingProfileID != ""` 但等于 `profile.ID`）
3. 进入 `PopulateUserWithProfileID`，写入与已有值相同的 `profile.ID`
4. `UpdateUser` 执行一次无害的 UPDATE

**结论**：重复绑定同一 identity 是安全的幂等操作。

### 4.3 场景 B 与 C 的区别

- **场景 B**：OAuth identity 被其他用户占用 → **跨用户冲突**（`AnotherUserWithFieldExists`）
- **场景 C**：当前用户已在同一 provider 上绑了不同的 identity → **同用户跨 identity 冲突**（`UserProfileID` 比对）

两者都返回同一个错误信息 `error.duplicate_linked_account`，但日志区分了两种情况：

```go
// B: "already associated with another user"
slog.Error("Oauth2 user cannot be associated because it is already associated with another user", ...)

// C: "already linked to a different identity"
slog.Error("Oauth2 user cannot be associated because this user is already linked to a different identity", ...)
```

### 4.4 解绑边界 — `oauth2Unlink`

`internal/ui/oauth2_unlink.go`：

```
POST /oauth2/{provider}/unlink
        ↓
├─ DISABLE_LOCAL_AUTH = true → 拒绝（防止用户失去所有登录方式）
│
├─ 用户无密码 (HasPassword = false)
│   → error.unlink_account_without_password
│
└─ 通过检查
    → UnsetUserProfileID(user)  // 清空 google_id / openid_connect_id
    → UpdateUser
    → alert.account_unlinked
```

**安全边界**：如果禁用了本地认证（`DISABLE_LOCAL_AUTH`），则完全阻止解绑操作；如果用户没有设置密码，也不允许解绑——确保用户至少保留一种登录方式。

---

## 5. Provider 实现

系统支持两种 OAuth2 provider，均实现 `oauth2.Provider` 接口：

| Provider | `UserExtraKey()` | 数据库字段 | Username 来源 |
|----------|-------------------|-----------|---------------|
| Google | `google_id` | `users.google_id` | `user.Email` |
| OIDC | `openid_connect_id` | `users.openid_connect_id` | `preferred_username > email > name > profile` |

### 5.1 OIDC Profile 获取

`internal/oauth2/oidc.go:64-114`：

1. `code` + `code_verifier` → Exchange 换 token
2. 从 token 中提取 `id_token` → `oidc.Provider.Verifier.Verify` 验证签名和 issuer
3. 额外调用 `UserInfo` 端点获取详细 claims
4. **双重校验**：`idToken.Subject != userInfo.Subject` 时拒绝——确保 ID Token 和 UserInfo 指向同一用户
5. Username 按优先级依次尝试：`preferred_username → email → name → profile`，全部为空则返回 `ErrEmptyUsername`

### 5.2 Google Profile 获取

`internal/oauth2/google.go:57-81`：

1. `code` + `code_verifier` → Exchange 换 token
2. 用 token 调 `googleapis.com/oauth2/v3/userinfo`
3. 直接从 JSON 中取 `sub`（唯一标识）和 `email`（用户名）

---

## 6. 安全措施汇总

| 措施 | 位置 | 说明 |
|------|------|------|
| State 参数 + 常量时间比较 | `oauth2_callback.go:42-47` | 防 CSRF |
| PKCE (S256) | `authorization.go` | 防授权码拦截 |
| 一次性消费 state/verifier | `ClearOAuth2Flow()` | 防重放 |
| Session Rotation | `auth.go:26` | 防 session fixation |
| 跨用户冲突检查 | `AnotherUserWithFieldExists` | 防 OAuth identity 被多次绑定 |
| 同用户跨 identity 检查 | `UserProfileID` 比对 | 防止替换已绑定的 identity |
| 解绑前密码检查 | `HasPassword` | 确保用户保留登录方式 |
| DISABLE_LOCAL_AUTH 阻止解绑 | `oauth2_unlink.go:17` | 防止无登录方式 |
| OIDC Subject 双重校验 | `oidc.go:87-89` | 确保 ID Token 与 UserInfo 一致 |

---

## 7. Token 续期失败时的回退路径

### 7.1 核心前提：Miniflux 不保存 Access Token / Refresh Token

**关键架构决策**：Miniflux 的 OAuth2 仅用于**首次身份认证**，换取用户 profile 后立即丢弃 access_token。
数据库中**不保存**任何 OAuth access_token 或 refresh_token，也不存在 token 自动续期逻辑。
用户后续的认证完全依赖 `WebSession` cookie（会话时长由 `CLEANUP_REMOVE_SESSIONS_INTERVAL` 控制）。

因此，"token 续期"在 Miniflux v2 中不存在代码层面的实现，实际对应的是两条路径：

### 7.2 路径一：授权码换 Token 阶段失败（回调中）

在 `oauth2_callback.go:59` 调用 `authProvider.Profile()` 时，内部 `conf.Exchange()` 失败：

```
回调进入 → state 校验通过 → FindProvider
        ↓
authProvider.Profile(code, codeVerifier)
        ↓
    ┌─ conf.Exchange() 失败（授权码无效/过期/重放）
    │   → 返回 wrapped error: "google/oidc: failed to exchange token: %w"
    │   → oauth2_callback.go:60-71 捕获
    │   → slog.Warn("Unable to get OAuth2 profile from provider")
    │   → 302 重定向回首页 "/"（无任何错误提示）
    │
    ├─ id_token 验证失败（OIDC 专用）
    │   → 签名无效 / audience 不匹配 / expired 等
    │   → "oidc: failed to verify id token: %w"
    │   → 同上，Warn 日志 + 302 到首页
    │
    ├─ id_token.Subject != userInfo.Subject（OIDC 专用）
    │   → "oidc: id token subject %q does not match userinfo subject %q"
    │   → 同上，Warn 日志 + 302 到首页
    │
    └─ userinfo 端点调用失败 / status != 200
        → "oidc: failed to get user info: %w"
        → "google: unexpected status code %d from userinfo endpoint"
        → 同上，Warn 日志 + 302 到首页
```

**回退策略**：静默失败，不展示错误原因给终端用户（仅写 slog.Warn 日志），统一 302 到首页。
这是基于 OAuth 失败通常是临时性/攻击性质的考虑——不让攻击者获得细节反馈，正常用户可重新点击登录按钮发起新一轮 OAuth 流程。

### 7.3 路径二：会话过期后的"隐式续期"——重新登录

当 WebSession 过期（超过 `CLEANUP_REMOVE_SESSIONS_INTERVAL`，默认值见 man page）：

```
用户请求受保护路由
        ↓
中间件检测到 WebSession 无效或已清除
        ↓
302 → /login（附带 redirect_url 参数）
        ↓
登录页根据配置显示：
  ├─ disableLocalAuth=false → 用户名密码表单 + OAuth 按钮
  ├─ disableLocalAuth=true  → 仅 OAuth 按钮（local 表单隐藏）
  └─ WebAuthn 已配置 → 额外显示 Passkey 按钮
        ↓
用户点击 OAuth 按钮
        ↓
完整的 OAuth redirect → provider 授权 → callback 流程重新执行
（视为全新登录，state/code_verifier 全部重新生成）
```

**关键区别**：传统 OAuth 资源服务器用 refresh_token 透明续期 access_token；
Miniflux 采用的"会话 + 重登录"模式意味着——用户 OAuth session 过期 = 重新走一遍完整授权流程。
如果用户在 provider 侧仍有登录态，通常会跳过授权确认页，实现透明"伪续期"。

### 7.4 边界：OIDC 初始化失败的回退

`NewManager`（`manager.go:34-56`）中，OIDC provider 初始化失败（discovery endpoint 不可用）时：

```go
if oidcProvider, err := NewOidcProvider(ctx, clientID, clientSecret, redirectURL, oidcDiscoveryEndpoint); err != nil {
    slog.Error("Failed to initialize OIDC provider", slog.Any("error", err))
    // 注意：不 return，继续执行。Manager 中该 provider 未被 AddProvider
}
```

**后果**：`providers` map 中没有 `"oidc"` 条目，后续 `FindProvider("oidc")` 返回 error。
此时即使 `OAUTH2_PROVIDER=oidc` 已配置，用户点击 OIDC 登录按钮后：
`oauth2Redirect.go:24-29` → `FindProvider` 失败 → `slog.Error` + 302 到首页。
这属于**启动时静默失败 + 运行时才暴露问题**的模式，运维需关注启动日志中的 `"Failed to initialize OIDC provider"`。

---

## 8. PKCE code_verifier 校验的完整代码挂载点

PKCE 流程横跨 5 个文件，共 **8 个挂载点**，完整调用链如下：

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │  ① 生成 (redirect 阶段)                                               │
 │  internal/oauth2/authorization.go:40-45                              │
 │    codeVerifier := crypto.GenerateRandomStringHex(32)   // 64 hex 字符 │
 │    sum := sha256.Sum256([]byte(codeVerifier))                        │
 │    state := crypto.GenerateRandomStringHex(24)                       │
 │    code_challenge = base64.RawURLEncoding.EncodeToString(sum[:])     │
 └───────────────────────────┬─────────────────────────────────────────┘
                             │ 返回到 Authorization{url, state, codeVerifier}
                             ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  ② 存储入 session (redirect 阶段)                                     │
 │  internal/ui/oauth2_redirect.go:35                                   │
 │    sess.StartOAuth2Flow(auth.State(), auth.CodeVerifier())           │
 └───────────────────────────┬─────────────────────────────────────────┘
                             │ 内部写 WebSession.state.OAuth2 结构体
                             ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  ③ session 持久化层封装                                               │
 │  internal/model/web_session.go:48-52, 163-167, 229-234, 238-240       │
 │    字段定义: CodeVerifier string `json:"code_verifier,omitempty"`      │
 │    存储:   StartOAuth2Flow(state, codeVerifier) → 标记 dirty         │
 │    读取:   OAuth2CodeVerifier() string                               │
 │    清除:   ClearOAuth2Flow() → OAuth2 = nil, dirty=true              │
 └───────────────────────────┬─────────────────────────────────────────┘
                             │ 回调时从 session 读取
                             ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  ④ 从 session 取出 + 一次性清除 (callback 阶段)                       │
 │  internal/ui/oauth2_callback.go:46-49                                │
 │    codeVerifier := sess.OAuth2CodeVerifier()                         │
 │    sess.ClearOAuth2Flow()   // ← 关键: 校验通过后立即清除，防止重放     │
 └───────────────────────────┬─────────────────────────────────────────┘
                             │ 作为参数传入 Profile()
                             ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  ⑤ Provider 接口签名约束                                             │
 │  internal/oauth2/provider.go:23                                      │
 │    Profile(ctx context.Context, code, codeVerifier string)           │
 └───────────────────────────┬─────────────────────────────────────────┘
              ┌──────────────┴────────────────┐
              ▼                               ▼
 ┌─────────────────────────────┐  ┌───────────────────────────────┐
 │  ⑥ Google 实现              │  │  ⑦ OIDC 实现                  │
 │  internal/oauth2/google.go:57-60 │ │  internal/oauth2/oidc.go:64-68    │
 │  conf.Exchange(ctx, code,    │  │  conf.Exchange(ctx, code,     │
 │    oauth2.SetAuthURLParam(   │  │    oauth2.SetAuthURLParam(    │
 │      "code_verifier",        │  │      "code_verifier",         │
 │      codeVerifier))          │  │      codeVerifier))           │
 └──────────────┬──────────────┘  └──────────────┬────────────────┘
                │                                 │
                └────────────────┬────────────────┘
                                 ▼
          ┌──────────────────────────────────────────────────────┐
          │  ⑧ golang.org/x/oauth2 库内部校验                     │
          │  oauth2.Config.Exchange() 发起到 provider Token 端点   │
          │  的 POST 请求，body 包含:                              │
          │    grant_type=authorization_code                      │
          │    code=<授权码>                                       │
          │    code_verifier=<明文 verifier>                      │
          │  Provider 服务器端:                                    │
          │    1. 取出该 code 对应的 code_challenge                │
          │    2. SHA256(code_verifier) == code_challenge ?       │
          │    3. 相等 → 发放 token；不相等 → 返回 invalid_grant   │
          └──────────────────────────────────────────────────────┘
```

### 8.1 关键设计点

| 关注点 | 位置 | 说明 |
|--------|------|------|
| **随机源** | `crypto.GenerateRandomStringHex` | 走 `crypto/rand`，不是伪随机 |
| **长度** | `authorization.go:40` | 32 字节 → 64 hex 字符，超过 RFC 7636 要求的 43~128 字符下限 |
| **清除时机** | `oauth2_callback.go:49` | 在调用 `FindProvider` **之前**清除——即使后续 provider 初始化失败，state 和 verifier 也已销毁，防止重放 |
| **Session 恢复** | `web_session.go:280-286` `UnmarshalState` | code_verifier 通过 JSON blob 从 `web_sessions.state` 列反序列化恢复，因此 session 持久化必须在回调前不被清理 |
| **OIDC 透传** | `oidc.go:66` | `conf.Exchange()` 自动把 `code_verifier` 放到 POST body，`golang.org/x/oauth2` 库负责 |

### 8.2 失败时的状态

如果 `Exchange()` 因 `code_verifier` 不匹配返回 `invalid_grant`：
- 回到 **7.2** 的回退路径 → slog.Warn + 302 到首页
- 由于 `ClearOAuth2Flow()` 已在前面执行，用户必须发起全新 OAuth 流程（不能重试同一个 code）

---

## 9. 多 Provider 同时存在时优先级合并的代码路径

### 9.1 核心结论：单 Provider 独占模式，不存在真正的"合并"

**配置层硬性限制**：`OAUTH2_PROVIDER` 是单值字符串，`config/options.go:468-472` 的 validator：

```go
validator: func(rawValue string) error {
    return validateChoices(rawValue, []string{"oidc", "google"})
}
```

只能二选一：`"oidc"` 或 `"google"`，不能同时指定。因此"多 provider 同时存在"在 Miniflux v2 中仅在以下两个语义层面发生：

### 9.2 层面一：Manager 内部 map 结构理论上支持多 Provider（但永远不会被用到）

`oauth2/manager.go` 的设计采用 `map[string]Provider`：

```go
type Manager struct {
    providers map[string]Provider
}
func (m *Manager) AddProvider(name string, provider Provider) { ... }
func (m *Manager) FindProvider(name string) (Provider, error) { ... }
```

但 **唯一调用 `AddProvider` 的入口** `NewManager()`（`manager.go:34-56`）使用 `switch`，一次只走一个 case：

```go
switch provider {      // provider = config.Opts.OAuth2Provider()，单值
case "oidc":
    m.AddProvider("oidc", oidcProvider)
case "google":
    m.AddProvider("google", NewGoogleProvider(...))
default:
    slog.Error("Unsupported OAuth2 provider")
}
```

**结果**：Manager 的 map 在运行时永远只有 0 或 1 个条目。
`FindProvider` 本质是"按名称查找唯一可能的 provider"，不存在多 provider 之间的优先级或 fallback 逻辑。

### 9.3 层面二：登录模板渲染层的互斥优先级

在 `login.html:47-55` 模板中，**使用 `if/else if` 而不是两个独立的 `if`**，这是全代码库中唯一出现"两个 provider 并列判断"的位置：

```gotemplate
{{ if hasOAuth2Provider "google" }}
    <a href="/oauth2/google/redirect">Google Sign-in</a>
{{ else if hasOAuth2Provider "oidc" }}
    <a href="/oauth2/oidc/redirect">Sign in with {{ oidcProviderName }}</a>
{{ end }}
```

其中 `hasOAuth2Provider`（`template/functions.go:54-56`）定义为：

```go
"hasOAuth2Provider": func(provider string) bool {
    return config.Opts.OAuth2Provider() == provider
}
```

**优先级结果**：当（不可能的情况下）`OAUTH2_PROVIDER` 同时等于两个值，`google` 优先于 `oidc` 被渲染。
但由于配置 validator 只允许二选一，`else if` 实际上是一个**防御性编程**，而不是真实路径。

### 9.4 配置互斥关系汇总（与非 OAuth 登录方式一起）

| 组合 | `OAUTH2_PROVIDER` | `DISABLE_LOCAL_AUTH` | `AUTH_PROXY_HEADER` | 结果 |
|------|-------------------|----------------------|---------------------|------|
| A | 空 | false | 空 | 仅本地用户名密码登录 |
| B | google | false | 空 | 本地表单 + Google 按钮（Google 优先级见模板） |
| C | oidc | false | 空 | 本地表单 + OIDC 按钮 |
| D | google | true | 空 | ✅ 仅 Google 按钮（通过 Validate） |
| E | oidc | true | 空 | ✅ 仅 OIDC 按钮（通过 Validate） |
| F | 空 | true | X-Forwarded-User | ✅ 仅 Auth Proxy |
| G | google | true | X-Forwarded-User | ✅ Google + Auth Proxy 共存（但模板只显示 OAuth，Auth Proxy 在请求头层生效） |
| H | 空 | true | 空 | ❌ Validate 报错：必须启用 OAuth 或 Auth Proxy 之一 |
| I | invalid_value | - | - | ❌ validator 报错：必须是 oidc 或 google |
| J | oidc + 缺 DISCOVERY_ENDPOINT | - | - | ❌ Validate 报错：discovery endpoint 必配 |

### 9.5 假设扩展：若要真正支持多 Provider 同时启用

当前架构需要修改的层：

| 层 | 当前 | 修改方向 |
|----|------|---------|
| 配置 | `OAUTH2_PROVIDER` 单值 | 改为多值（逗号分隔或按 provider 拆分配置项） |
| Manager | `NewManager` switch 单例 | `AddProvider` 在多个分支都执行，每个 provider 有独立的 clientID/secret/redirectURL |
| 模板 | `if/else if` 互斥 | 改为独立的 `range` 或多个 `if`，同时显示多个 OAuth 按钮 |
| settings 页面 | 绑定/解绑逻辑按 provider | 已支持（路由用 `{provider}` 参数化），无需改动 |
| `oauth2_callback` 回调 | 已按 `{provider}` 路由参数分发 | 已支持，无需改动 |

> 结论：回调层和绑定层天然支持多 provider（因为 provider 是 URL 参数化的），瓶颈在配置层和模板渲染层人为限制了单 provider。

---

## 10. 第三方账号解绑流程的完整代码挂载点

### 10.1 端到端调用链

```
用户在 Settings 页点击 "Unlink Google/OIDC Account" 按钮
        ↓
POST /oauth2/{provider}/unlink
  csrf=<hidden token>
        ↓
┌── 中间件层（ui.go:184 的链式调用） ──────────────────────────────┐
│                                                                 │
│  ① webSessionMiddleware (web_session_middleware.go:27-67)        │
│     → 从 cookie 加载 WebSession                                 │
│     → 未认证 + 非公共路由 → 302 到 /login（解绑路由非公共，必须有登录态）│
│     → session 存入 r.Context                                    │
│                                                                 │
│  ② csrfMiddleware (csrf_middleware.go:27-38)                     │
│     → POST 方法，需要 CSRF 校验                                  │
│     → 比对 session.CSRF() 与 form.csrf / X-Csrf-Token header    │
│     → 使用 crypto.ConstantTimeCmp 常量时间比较                   │
│     → 失败 → 400 Bad Request                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
        ↓ 校验通过，进入 handler
┌── Handler 层 (oauth2_unlink.go:16-70) ──────────────────────────┐
│                                                                 │
│  ③ DisableLocalAuth 门禁 (:17-23)                               │
│     config.Opts.DisableLocalAuth() == true                      │
│     → slog.Warn + 302 到 "/"，完全阻断                           │
│     设计意图：禁用本地认证后，解绑 OAuth 会让用户失去所有登录方式  │
│                                                                 │
│  ④ 路由参数解析 (:25-30)                                        │
│     provider := request.RouteStringParam(r, "provider")         │
│     → "" → 302 到 "/"                                           │
│                                                                 │
│  ⑤ getOAuth2Manager + FindProvider (:32-40)                     │
│     每次 HTTP 请求都新建 Manager（见 §11 的"每次实例化"问题）      │
│     FindProvider(provider) 失败 → 302 到 /settings              │
│                                                                 │
│  ⑥ 查询当前用户 (:42-46)                                        │
│     h.store.UserByID(request.UserID(r))                         │
│     从 session context 中取 userID → 查 DB                       │
│                                                                 │
│  ⑦ HasPassword 检查 (:48-60)                                    │
│     h.store.HasPassword(request.UserID(r))                      │
│     SQL: SELECT true FROM users WHERE id=$1 AND password <> ''  │
│     → 无密码 → error.unlink_account_without_password            │
│     → 302 到 /settings + 错误闪现消息                           │
│     设计意图：确保用户解绑后仍能用密码登录                        │
│                                                                 │
│  ⑧ Provider.UnsetUserProfileID (:62)                            │
│     Google: user.GoogleID = ""           (google.go:96-98)      │
│     OIDC:   user.OpenIDConnectID = ""    (oidc.go:129-131)      │
│     仅修改内存中 User struct 的字段，尚未落盘                    │
│                                                                 │
│  ⑨ h.store.UpdateUser(user) (:63-66)                            │
│     storage/user.go:174 → SQL UPDATE users SET ...               │
│     google_id=$16, openid_connect_id=$17 ... WHERE id=$31       │
│     空字符串 "" 写入数据库 → 实际效果等于清空绑定                 │
│                                                                 │
│  ⑩ 成功反馈 (:68-69)                                            │
│     sess.SetSuccessMessage(printer.Print("alert.account_unlinked"))│
│     302 → /settings（闪现成功消息）                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 各挂载点文件与行号索引

| 步骤 | 代码位置 | 职责 |
|------|---------|------|
| 路由注册 | `ui.go:152` | `POST /oauth2/{provider}/unlink` 仅在 `OAUTH2_PROVIDER != ""` 时注册 |
| 会话中间件 | `web_session_middleware.go:27-67` | 加载 session、认证检查、脏写回存 |
| CSRF 中间件 | `csrf_middleware.go:27-38` | POST 方法校验 CSRF token（常量时间比较） |
| DisableLocalAuth 门禁 | `oauth2_unlink.go:17-23` | 阻止在禁用本地认证时解绑 |
| Provider 查找 | `oauth2_unlink.go:32-40` | 每次请求新建 Manager + FindProvider |
| 用户查询 | `oauth2_unlink.go:42-46` | 从 session userID 查完整 user 对象 |
| 密码检查 | `oauth2_unlink.go:48-60` | `HasPassword` SQL 查询 |
| UnsetUserProfileID | `oidc.go:129-131` / `google.go:96-98` | 内存中清空 provider ID 字段 |
| UpdateUser | `storage/user.go:174` | SQL UPDATE 全字段（含 google_id / openid_connect_id） |
| 闪现消息 | `oauth2_unlink.go:68-69` | session 中写入成功消息，下次页面渲染时消费 |

### 10.3 Settings 模板渲染逻辑

`settings.html:11-39` 的条件渲染链：

```gotemplate
{{ if and (not disableLocalAuth) (hasOAuth2Provider "google") }}
    {{ if .user.GoogleID }}
        <!-- 显示 "Unlink Google" 按钮（POST form + csrf token）-->
    {{ else }}
        <!-- 显示 "Link Google" 链接（GET /oauth2/google/redirect）-->
    {{ end }}
{{ else if and (not disableLocalAuth) (hasOAuth2Provider "oidc") }}
    {{ if .user.OpenIDConnectID }}
        <!-- 显示 "Unlink OIDC" 按钮 -->
    {{ else }}
        <!-- 显示 "Link OIDC" 链接 -->
    {{ end }}
{{ end }}
```

关键细节：

- **外层 `if/else if`**：同一时间只显示一个 provider 的绑定/解绑区块（与 §9.3 的单 Provider 限制一致）
- **内层 `if .user.GoogleID` / `if .user.OpenIDConnectID`**：根据数据库中对应字段是否为空决定显示"绑定"还是"解绑"
- **`not disableLocalAuth`**：本地认证被禁用时，整个 fieldset 不渲染——与 `oauth2_unlink.go:17` 的门禁逻辑双重保障
- **CSRF token**：解绑表单内嵌 `<input type="hidden" name="csrf" value="{{ .csrf }}">`，由 `csrf_middleware.go` 校验

### 10.4 解绑后数据状态的完整视角

```
解绑前 (以 OIDC 为例):
  users 表: openid_connect_id = "sub-12345"  (OIDC subject)
  User struct: OpenIDConnectID = "sub-12345"

UnsetUserProfileID:
  User struct: OpenIDConnectID = ""           (内存修改)

UpdateUser:
  SQL: UPDATE users SET ... openid_connect_id='' ... WHERE id=$31
  users 表: openid_connect_id = ""            (落盘)
```

**注意**：解绑只是清空了 `openid_connect_id` / `google_id` 字段，用户的 Miniflux 账号本身不受影响（username、password 等全部保留）。
下次同一 OIDC 用户登录时，会走"用户不存在 + 自动创建"路径（如果 `OAUTH2_USER_CREATION=1`），创建一个全新的 Miniflux 账号——而非关联到原有账号。

### 10.5 解绑与绑定的安全对称性

| 维度 | 绑定（callback，已登录路径） | 解绑（unlink） |
|------|--------------------------|---------------|
| HTTP 方法 | GET（由 OAuth 回调触发） | POST（表单提交） |
| CSRF 校验 | 无（GET 请求豁免，但 state 参数防 CSRF） | 有（csrfMiddleware 校验 form.csrf） |
| 登录态要求 | 需要（`request.IsAuthenticated`） | 需要（路由非公共，webSessionMiddleware 拦截） |
| 冲突检查 | `AnotherUserWithFieldExists` + `UserProfileID` | 无（清空操作无冲突可能） |
| 退路保障 | 无（绑定不会丢失登录方式） | `HasPassword` / `DisableLocalAuth`（防止失去登录方式） |
| Provider 实例化 | `getOAuth2Manager` 每次 new | `getOAuth2Manager` 每次 new |
| 数据操作 | `PopulateUserWithProfileID` + `UpdateUser` | `UnsetUserProfileID` + `UpdateUser` |

---

## 11. OIDC Discovery 动态加载的代码路径

### 11.1 核心机制

Miniflux 使用 `github.com/coreos/go-oidc/v3` 库的 `oidc.NewProvider(ctx, discoveryEndpoint)` 实现 OIDC Discovery。
该函数向 discovery endpoint 发起 HTTP GET 请求，获取并解析 OpenID Connect Discovery 文档（`.well-known/openid-configuration`），
从中提取 `authorization_endpoint`、`token_endpoint`、`userinfo_endpoint`、`jwks_uri` 等关键端点，
构造一个 `oidc.Provider` 对象，后续用于 token 验证和 userinfo 获取。

### 11.2 完整调用链

```
┌── 每次请求触发 ────────────────────────────────────────────────┐
│  oauth2_unlink.go:32 / oauth2_redirect.go:23 / oauth2_callback.go:49  │
│    getOAuth2Manager(r.Context())                               │
│        ↓                                                       │
│  auth.go:55-64                                                 │
│    oauth2.NewManager(                                          │
│      ctx,                                                      │
│      config.Opts.OAuth2Provider(),          // "oidc"          │
│      config.Opts.OAuth2ClientID(),                           │
│      config.Opts.OAuth2ClientSecret(),                       │
│      config.Opts.OAuth2RedirectURL(),                        │
│      config.Opts.OAuth2OIDCDiscoveryEndpoint(),              │
│    )                                                           │
│        ↓                                                       │
│  manager.go:34-56  NewManager()                               │
│    switch provider {                                           │
│    case "oidc":                                                │
│      NewOidcProvider(ctx, clientID, clientSecret,              │
│                      redirectURL, oidcDiscoveryEndpoint)       │
│        ↓                                                       │
│  oidc.go:36-48  NewOidcProvider()                             │
│    oidc.NewProvider(ctx, discoveryEndpoint)                    │
│    ↑                                                           │
│    │  这是 go-oidc/v3 库的核心调用                              │
│    │  内部执行:                                                 │
│    │    GET {discoveryEndpoint}                                │
│    │    → 解析 JSON → 提取各端点 URL                            │
│    │    → 预取 JWKS (JSON Web Key Set) 用于 id_token 签名验证   │
│    │    → 返回 *oidc.Provider                                  │
│    ↓                                                           │
│  返回 &oidcProvider{                                           │
│    clientID, clientSecret, redirectURL,                        │
│    provider: *oidc.Provider  // 包含 discovery 结果             │
│  }                                                             │
│        ↓                                                       │
│  manager.AddProvider("oidc", oidcProvider)                     │
│  Manager.providers["oidc"] = oidcProvider                      │
│        ↓                                                       │
│  返回 Manager → FindProvider("oidc") → 使用                    │
└───────────────────────────────────────────────────────────────┘
```

### 11.3 "每次请求实例化"的关键影响

**`getOAuth2Manager` 不是单例，每次 HTTP 请求都重新创建 Manager + 重新执行 OIDC Discovery。**

这意味着：

| 影响 | 说明 |
|------|------|
| **性能开销** | 每次 OAuth 相关请求都发一次 GET 到 discovery endpoint + 可能的 JWKS 预取 |
| **网络依赖** | 如果 discovery endpoint 不可达，所有 OAuth 请求（包括 redirect 和 callback）都会失败 |
| **动态配置生效** | 修改 IdP 的 endpoint 配置后无需重启 Miniflux——下次请求自动获取最新值 |
| **无缓存** | go-oidc 的 `NewProvider` 不内置缓存，每次调用都是完整的 HTTP roundtrip |

### 11.4 Discovery 产出物的使用路径

`oidcProvider` 保存 discovery 结果于 `o.provider` 字段，在以下两处被消费：

#### 11.4.1 `Config()` — 构造 OAuth2 授权配置

```go
// oidc.go:54-61
func (o *oidcProvider) Config() *oauth2.Config {
    return &oauth2.Config{
        RedirectURL:  o.redirectURL,
        ClientID:     o.clientID,
        ClientSecret: o.clientSecret,
        Scopes:       []string{oidc.ScopeOpenID, "profile", "email"},
        Endpoint:     o.provider.Endpoint(),   // ← discovery 结果
    }
}
```

`o.provider.Endpoint()` 返回 `oauth2.Endpoint{AuthURL, TokenURL, …}`，来源于 discovery 文档中的
`authorization_endpoint` 和 `token_endpoint`。这些 URL 被用于：

1. **redirect 阶段**：`authorization.go:49` → `config.AuthCodeURL(state, ...)` 构造授权 URL
2. **callback 阶段**：`oidc.go:66` → `conf.Exchange(ctx, code, ...)` 换 token

#### 11.4.2 `Profile()` — id_token 验证和 userinfo 获取

```go
// oidc.go:76
verifier := o.provider.Verifier(&oidc.Config{ClientID: o.clientID})
idToken, err := verifier.Verify(ctx, rawIDToken)
```

`o.provider.Verifier()` 创建一个 ID Token 验证器，内部使用 discovery 获取的 JWKS（`jwks_uri` 端点）
来验证 id_token 的签名。验证内容包括：

- 签名有效性（RSA/ECDSA 公钥验证）
- `iss`（issuer）匹配 discovery 文档的 issuer
- `aud`（audience）匹配 clientID
- `exp`（过期时间）未过期
- `iat`（签发时间）有效

```go
// oidc.go:82
userInfo, err := o.provider.UserInfo(ctx, oauth2.StaticTokenSource(token))
```

`o.provider.UserInfo()` 使用 discovery 文档中的 `userinfo_endpoint` 获取用户信息。

### 11.5 Discovery 失败的完整影响矩阵

| 失败场景 | 发生阶段 | 代码位置 | 行为 |
|---------|---------|---------|------|
| discovery endpoint 不可达 | `getOAuth2Manager` → `NewOidcProvider` | `oidc.go:37-39` | `NewProvider` 返回 error → `NewManager` 不 AddProvider → `FindProvider` 返回 "provider not found" → redirect/callback/unlink 均 302 到首页或 /settings |
| discovery endpoint 返回非法 JSON | 同上 | go-oidc 内部 | 同上 |
| discovery 文档缺少 `authorization_endpoint` | `Config()` → `Endpoint()` | go-oidc 内部 | `AuthCodeURL` 构造失败或 URL 不合法 |
| discovery 文档缺少 `token_endpoint` | `Profile()` → `Exchange()` | go-oidc 内部 | token exchange 请求发往空/错误 URL → Exchange 失败 → callback 302 到首页 |
| `jwks_uri` 不可达 | `Profile()` → `Verifier.Verify()` | `oidc.go:76-79` | id_token 签名无法验证 → Verify 返回 error → callback 302 到首页 |
| `userinfo_endpoint` 不可达 | `Profile()` → `UserInfo()` | `oidc.go:82-84` | 无法获取用户信息 → callback 302 到首页 |
| discovery endpoint 响应慢 | 所有涉及 OAuth 的请求 | 全链路 | 每个 redirect/callback/unlink 请求都被阻塞直到 discovery 完成（无超时控制） |

### 11.6 配置层约束

`config/parser.go:55-57`：

```go
if c.OAuth2Provider() == "oidc" && c.OAuth2OIDCDiscoveryEndpoint() == "" {
    return errors.New("OAUTH2_OIDC_DISCOVERY_ENDPOINT must be configured when using the OIDC provider")
}
```

**启动时校验**：`OAUTH2_PROVIDER=oidc` 必须同时提供 `OAUTH2_OIDC_DISCOVERY_ENDPOINT`，否则拒绝启动。
但仅校验非空，不校验 URL 可达性——可达性延迟到运行时首次请求才暴露。

`OAUTH2_OIDC_PROVIDER_NAME`（默认值 `"OpenID Connect"`）纯粹是 UI 展示用，在 `login.html:53` 和 `settings.html:27,31,35`
中作为模板变量显示，不参与任何 discovery 或认证逻辑。

---

## 12. 第三方 Token 失效后用户主动绑定其他 Provider 的代码路径

### 12.1 语义澄清：何为"Token 失效 + 绑定其他 Provider"

在 Miniflux 的架构中，不存在"保存第三方 access_token → access_token 过期 → refresh → 失败"这条链路（参见 §7.1）。
因此"第三方 token 失效后用户主动绑定其他 provider"在本系统中实际对应以下**三种业务场景**：

| 场景编号 | 业务语义 | 触发原因 |
|---------|---------|---------|
| 场景 A | 用户 WebSession 过期（相当于"第三方登录态失效"），需要重新走 OAuth 登录，此时系统配置的 provider 与上次登录时**不同**（运维改了配置） | `OAUTH2_PROVIDER` 被从 `google` 改为 `oidc` 或反之 |
| 场景 B | 用户已通过本地密码登录，想在 Settings 页面绑定一个**尚未绑定过**的 provider（此前用户可能用另一 provider 登录过，或者从未绑过任何 provider） | 用户主动操作 Settings 页的"Link XXX Account" |
| 场景 C | 用户已通过 provider X 绑定了账号，但后来 provider X 侧的账号被封禁/注销（此时用户的 `google_id` / `openid_connect_id` 仍在 DB 中，但实际无法通过 X 登录），用户想改用密码登录后绑定 provider Y | 跨 provider 账号迁移/自救 |

### 12.2 场景 A：Session 过期 + 系统切换到其他 Provider

#### 触发条件
- 用户上次以 `OAUTH2_PROVIDER=google` 登录，session 过期后 DB 中 `users.google_id="sub-abc"`
- 运维将 `OAUTH2_PROVIDER` 改为 `oidc`（重启进程后生效）
- 用户再次访问，被 302 到 `/login`

#### 登录页渲染逻辑

`login.html:47-55` 根据当前 `OAUTH2_PROVIDER` 渲染按钮：

```gotemplate
{{ if hasOAuth2Provider "google" }}      ← 现在是 oidc，走 else if 分支
    Google Sign-in
{{ else if hasOAuth2Provider "oidc" }}   ← 现在渲染 "Sign in with OpenID Connect"
    Sign in with {{ oidcProviderName }}
{{ end }}
```

#### 回调处理逻辑（未登录路径）

用户点击 OIDC 按钮完成授权 → `oauth2_callback.go:110-154` 未登录分支：

```
oauth2_callback.go:115
  store.UserByField(profile.Key, profile.ID)
    → profile.Key = "openid_connect_id"
    → profile.ID = OIDC 返回的 subject
    → SELECT ... FROM users WHERE openid_connect_id=$1

  ┌─ 结果: 未找到（DB 中 openid_connect_id 为空，只有 google_id 有值）
  │
  ├─ OAUTH2_USER_CREATION = 1
  │   ├─ store.UserExists(username)
  │   │   ├─ true（该用户名已被原有用户占用）
  │   │   │   → oauth2_callback.go:131-136
  │   │   │   → error.user_already_exists → 400 Bad Request
  │   │   │
  │   │   └─ false（username 不冲突）
  │   │       → 创建新用户 + authenticateWebSession
  │   │       → 后果: 同一人拥有两个独立账号
  │   │
  │   └─ OAUTH2_USER_CREATION != 1
  │       → 403 Forbidden（oauth2_callback.go:125-128）
  │
  └─ 结果: 找到了（需要该 OIDC subject 恰好等于某个已存在用户的 openid_connect_id）
      → 直接登录该用户
```

**关键发现：场景 A 存在"账号分裂"风险。**

由于系统只按当前 provider 的 `profile.Key` 查找（单值 `OAUTH2_PROVIDER`），切换 provider 后
`UserByField("openid_connect_id", …)` 找不到已有 `google_id` 的记录——如果 `OAUTH2_USER_CREATION=1`
且用户名不冲突，会错误地创建第二个账号。

#### 风险矩阵

| `OAUTH2_USER_CREATION` | `UserExists(username)` | 结果 | 风险等级 |
|------------------------|----------------------|------|---------|
| 1 | true | 400 `error.user_already_exists` | 中（用户困惑，但没有数据分裂） |
| 1 | false | 创建新用户，账号分裂 | **高**（同一人两个独立账号，feeds/favorites 不共享） |
| != 1 | N/A | 403 Forbidden | 低（安全但用户无法登录） |

#### 运维操作建议：切换 Provider 前的正确流程

1. 要求所有用户先设置密码（Settings 页修改密码）
2. 切换 `OAUTH2_PROVIDER` 前确保 `DISABLE_LOCAL_AUTH=false`
3. 切换后用户先用密码登录，再在 Settings 页绑定新 provider（即走场景 B）
4. 确认用户都迁移完成后再考虑 `DISABLE_LOCAL_AUTH`

### 12.3 场景 B：已登录用户绑定另一个 Provider（通过 Settings 页）

#### 页面渲染

用户已通过本地密码登录 → 访问 `/settings`

`settings_show.go:18-81` 渲染 Settings 页面，其中 `settings.html:11-39` 的 OAuth 区块：

```gotemplate
{{ if and (not disableLocalAuth) (hasOAuth2Provider "google") }}
    {{ if .user.GoogleID }}
        <!-- 已绑定: 显示 Unlink 按钮 -->
    {{ else }}
        <!-- 未绑定: 显示 Link Google 链接 → GET /oauth2/google/redirect -->
    {{ end }}
{{ else if and (not disableLocalAuth) (hasOAuth2Provider "oidc") }}
    {{ if .user.OpenIDConnectID }}
        <!-- 已绑定: 显示 Unlink 按钮 -->
    {{ else }}
        <!-- 未绑定: 显示 Link OIDC 链接 → GET /oauth2/oidc/redirect -->
    {{ end }}
{{ end }}
```

**关键约束**：外层 `if/else if` 意味着同一时间只能显示**当前配置的**那个 provider 的绑定入口。
如果 `OAUTH2_PROVIDER=oidc`，即使该用户的 `google_id` 有值，也永远不会看到 Google 的绑定/解绑区域。

#### 主动绑定的完整链路

```
用户在 Settings 点击 "Link OIDC Account"
        ↓
GET /oauth2/oidc/redirect  → oauth2_redirect.go:32-36
  → 生成 state/code_verifier → StartOAuth2Flow
  → 302 → OIDC 授权页
        ↓
OIDC 回调 → GET /oauth2/oidc/callback
        ↓
oauth2_callback.go:39-51
  state 校验 → code_verifier 取出 → ClearOAuth2Flow
        ↓
oauth2_callback.go:55-67: request.IsAuthenticated(r) = true
        ↓
oauth2_callback.go:60-67: 冲突检查
  ① AnotherUserWithFieldExists(loggedUser.ID, profile.Key, profile.ID)
    → 是否有其他用户绑了同一个 OIDC subject？
    → true: error.duplicate_linked_account → 302 /settings
  ② existingProfileID := authProvider.UserProfileID(loggedUser)
    → 即 loggedUser.OpenIDConnectID
    → existingProfileID != "" && != profile.ID
      → error.duplicate_linked_account → 302 /settings
        ↓
oauth2_callback.go:100-103: 冲突通过
  authProvider.PopulateUserWithProfileID(loggedUser, profile)
    → oidc.go:122-127: loggedUser.OpenIDConnectID = profile.ID
  h.store.UpdateUser(loggedUser)
    → storage/user.go:257-329: UPDATE ... openid_connect_id=$16 ...
        ↓
  alert.account_linked → sess.SetSuccessMessage
  302 → /settings
```

#### 场景 B 的边界细节

| 边界条件 | 检查点 | 结果 |
|---------|-------|------|
| 用户当前 `google_id="abc"`，系统配置 `OAUTH2_PROVIDER=oidc`，想绑 OIDC | `AnotherUserWithFieldExists` 查 `openid_connect_id`，`existingProfileID` 读 `OpenIDConnectID`（为空） | ✅ 通过，绑定成功。最终用户同时拥有 `google_id="abc"` + `openid_connect_id="xyz"` **两个字段都有值** |
| 绑完 OIDC 后，用户想再绑 Google（切换回 `OAUTH2_PROVIDER=google`） | `AnotherUserWithFieldExists` 查 `google_id`（等于 "abc"，`id<>$1` 排除自身后无冲突）；`existingProfileID = "abc"`，且 Google 回调的 `profile.ID` 也为 "abc" | ✅ 通过，幂等 UPDATE，无变化 |
| 绑完 OIDC 后，系统仍为 `OAUTH2_PROVIDER=oidc`，用户点击不同 OIDC 账号授权 | `existingProfileID != "" && != profile.ID`（新账号 subject 不同） | ❌ `error.duplicate_linked_account` |
| `DISABLE_LOCAL_AUTH=true` 时尝试绑定 | Settings 模板 `if not disableLocalAuth` → 整个绑定区块不渲染 | ⚠️ 前端隐藏，但路由 `/oauth2/{provider}/redirect` 仍是公共路由——**理论上可手动构造请求** |

> **重要**：场景 B 是**唯一能让一个用户同时拥有 `google_id` 和 `openid_connect_id` 两个非空值的路径**。
> 因为只有先配一个 provider 绑定，再改配置配另一个 provider 再绑定——而回调冲突检查只检查"当前 provider 维度"，不阻塞"另一个 provider 字段已非空"。

### 12.4 场景 C：原 Provider 账号失效 → 密码登录 → 绑定新 Provider

这是场景 B 的前置 + 场景 A 的补救组合：

```
用户原 Google 账号被封禁 → 无法通过 Google OAuth 登录
        ↓
使用本地密码登录 /login（DISABLE_LOCAL_AUTH 必须为 false）
  → webSessionMiddleware: 认证成功 → 建立 session
        ↓
（运维已切换 OAUTH2_PROVIDER=oidc，或用户等管理员切换）
        ↓
访问 /settings → 显示 "Link OIDC Account"（场景 B 路径）
        ↓
点击 Link OIDC Account → 完整 redirect → callback 流程
        ↓
绑定成功: 用户的 google_id 仍保留（但无法再用），openid_connect_id 新绑定
        ↓
用户未来可通过 OIDC 登录
  → UserByField("openid_connect_id", ...) → 找到用户 → 正常登录
```

#### 场景 C 的关键约束

| 约束 | 说明 |
|------|------|
| **必须有密码** | 原 Google 账号失效后，用户唯一能进入系统的方式是本地密码登录。如果用户从未设置过密码（纯 OAuth 注册），此路径关闭 |
| **`DISABLE_LOCAL_AUTH` 必须为 false** | 如果设置了禁用本地认证，用户连登录都做不到 |
| **原 `google_id` 不会自动清空** | 绑定 OIDC 后，Google ID 仍保留在 DB 中。如果未来恢复 Google 配置，用户可用两种方式登录 |
| **管理员干预** | 若用户没有密码，管理员需通过 `miniflux -reset-password` CLI 命令重置（`cli.go:117-143`） |

---

## 13. OAuth Provider 配置热更新的代码挂载点

### 13.1 核心结论：不支持真正的"热更新"

Miniflux 没有配置热加载机制。`config.Opts` 是**启动时一次性初始化**的全局变量（`config/config.go:9`），
没有任何在运行时重新读取环境变量或配置文件的代码路径。

但由于 **OAuth 相关配置的读取方式** 与 **Provider 实例化模式** 的组合，存在一种"准热更新"的假象——
下文详细拆解三层。

### 13.2 配置初始化的唯一入口

`cli.go:39-96`，启动时执行一次：

```go
// cli.go:80-96
cfg := config.NewConfigParser()

if flagConfigFile != "" {
    config.Opts, err = cfg.ParseFile(flagConfigFile)   // ① 先读 --config-file
    if err != nil { printErrorAndExit(err) }
}

config.Opts, err = cfg.ParseEnvironmentVariables()     // ② 环境变量覆盖文件
if err != nil { printErrorAndExit(err) }

if err := config.Opts.Validate(); err != nil {          // ③ 校验互斥关系
    printErrorAndExit(err)                              //    不通过则退出进程
}
```

`ParseEnvironmentVariables()`（`parser.go`）遍历 `configOptions.options` map 的每一项，
从 `os.Getenv()` 读取值 → 解析 → 写入 `config.Opts`。**此后 `config.Opts` 不再被写入**。

### 13.3 信号处理层：没有 SIGHUP

`cli/daemon.go:26-28` 注册的信号：

```go
signal.Notify(stop, os.Interrupt)   // SIGINT
signal.Notify(stop, syscall.SIGTERM)
```

**没有注册 SIGHUP**（常见于 Nginx 等服务的 reload 信号）。因此 `kill -HUP <pid>` 不会触发任何配置重载，
反而会走默认行为——终止进程（或被忽略，取决于 Go runtime 版本）。

### 13.4 三层"准热更新"的代码挂载点

虽然没有真正的热加载，但有三个挂载点使得**某些 OAuth 相关配置变化**在不重启进程时就可体现：

#### 挂载点 1：`getOAuth2Manager` 每次请求读 `config.Opts` — 配置值层面的"热"

`internal/ui/auth.go:55-64`：

```go
func getOAuth2Manager(ctx context.Context) *oauth2.Manager {
    return oauth2.NewManager(
        ctx,
        config.Opts.OAuth2Provider(),           // ← 每次请求实时读全局变量
        config.Opts.OAuth2ClientID(),           // ← 每次请求实时读
        config.Opts.OAuth2ClientSecret(),       // ← 每次请求实时读
        config.Opts.OAuth2RedirectURL(),        // ← 每次请求实时读
        config.Opts.OAuth2OIDCDiscoveryEndpoint(),  // ← 每次请求实时读
    )
}
```

`OAuth2Provider()` 等方法只是简单的 getter（`config/options.go` 的 accessor），从 `config.Opts.options` map 中读取已解析值。

**关键点**：如果能在运行时**修改 `config.Opts.options` map** 中对应 key 的值（例如通过 unsafe 或反射），
由于 `getOAuth2Manager` 每次请求都读，**新请求会立即使用新值**——无需重启。但这依赖于外部手段修改内存中的全局变量，
不是 Miniflux 设计或维护的正式接口。

**正常情况下**：修改环境变量后，已有进程的 `os.Getenv()` **不会**重新读取子进程环境变量表（操作系统语义），
因此正常修改环境变量不能触发此挂载点生效——必须重启。

#### 挂载点 2：OIDC Discovery 每次重新 HTTP GET — 端点发现层面的"热"

参见 §11。`NewOidcProvider(ctx, discoveryEndpoint)` 每次调用都会向 discovery endpoint 发 HTTP GET：

- 如果运维修改了 OIDC IdP 侧 discovery 文档内容（例如换了 `authorization_endpoint`、`token_endpoint` 或轮换了 `jwks_uri`），
  **无需重启 Miniflux**，下一次 OAuth 相关请求的 `NewManager → NewOidcProvider → oidc.NewProvider` 会自动获取最新值
- 这不是 Miniflux 的"配置热更新"，而是 IdP discovery 协议本身的"动态"特性

**热生效路径**：
```
IdP 管理员修改 .well-known/openid-configuration
        ↓
下一次用户点击 OAuth 登录按钮
        ↓
GET /oauth2/oidc/redirect → getOAuth2Manager() → NewManager()
        ↓
NewOidcProvider → oidc.NewProvider(ctx, "https://idp.example.com/.well-known/openid-configuration")
        ↓
HTTP GET → 拿到最新的 authorization_endpoint / token_endpoint / jwks_uri
        ↓
用户被跳转到新的 authorization_endpoint（已热切换）
```

#### 挂载点 3：模板函数每次请求读 `config.Opts` — UI 展示层面的"热"

`internal/template/functions.go:54-56, 60-62`：

```go
"hasOAuth2Provider": func(provider string) bool {
    return config.Opts.OAuth2Provider() == provider   // ← 每次模板渲染实时读
},
"oidcProviderName": func() string {
    return config.Opts.OAuth2OIDCProviderName()       // ← 每次模板渲染实时读
},
```

这些函数在每次渲染登录页、设置页时被调用。假设通过某种手段（如反射修改）改变了 `config.Opts.OAuth2Provider()`，
**下次页面刷新**就会：

- `hasOAuth2Provider("google")` 从 true → false，Google 按钮消失
- `hasOAuth2Provider("oidc")` 从 false → true，OIDC 按钮出现

但同样——这不是受支持的热更新路径。

### 13.5 路由注册层：永远的"冷"

`internal/ui/ui.go:137-154`：

```go
if config.Opts.HasOAuth2Provider() {   // ← 仅在 StartServer 时判断一次
    mux.HandleFunc("GET /oauth2/{provider}/redirect", handler.oauth2Redirect)
    mux.HandleFunc("GET /oauth2/{provider}/callback", handler.oauth2Callback)
    mux.HandleFunc("POST /oauth2/{provider}/unlink", handler.oauth2Unlink)
}
```

**路由注册在进程启动时执行一次**（`StartWebServer`）。即使通过反射修改了 `config.Opts.HasOAuth2Provider()` 的返回值，
已注册的路由不会消失，未注册的路由也不会出现——除非重启进程。

这意味着：

| 操作 | 不重启是否生效 | 说明 |
|------|--------------|------|
| 从 `""` → `"oidc"` 启用 OAuth | ❌ 不生效 | 路由没注册，访问 `/oauth2/oidc/redirect` 返回 404 |
| 从 `"oidc"` → `""` 禁用 OAuth | ⚠️ 部分生效 | 路由仍存在，URL 可访问但 `FindProvider` 永远失败 → 302 到首页 |
| 从 `"google"` → `"oidc"` 切换 provider | ⚠️ 部分生效 | 路由仍注册（都走通用参数化路径），`FindProvider` 会根据新值查找，**实际可工作** |

最意外的是最后一行：**切换 provider（`google ↔ oidc`）在不重启进程时有可能部分工作**——
因为路由是 `{provider}` 参数化的，`redirect`、`callback`、`unlink` 的 handler 内部都走 `getOAuth2Manager` → `FindProvider(provider)`，
如果新的 `config.Opts.OAuth2Provider()` 值与 URL 中的 `{provider}` 参数匹配，流程就能正常走完。
但这完全依赖于"用户恰好知道新的 provider 名称并手动构造 URL，或 `hasOAuth2Provider` 热更新后按钮渲染成正确的 URL"。

### 13.6 OAuth 相关配置项在各层的"冷/热"全景

| 配置项 | 类型 | config.Opts accessor | 启动时 Validate | 路由注册时读取 | 每次请求读取 | 每次模板渲染读取 | 准热更新 |
|--------|------|---------------------|----------------|---------------|-------------|-----------------|---------|
| `OAUTH2_PROVIDER` | 字符串 | `OAuth2Provider()` | validator 限制 `oidc`/`google` | `ui.go:137`（注册开关） | `auth.go:58`（getOAuth2Manager） | `functions.go:55`（hasOAuth2Provider） | ⚠️ 部分（路由已注册则 handler 生效；未注册则 404） |
| `OAUTH2_CLIENT_ID` | 字符串 | `OAuth2ClientID()` | 非空校验 | 否 | `auth.go:59` | 否 | ✅（getOAuth2Manager 每次读取） |
| `OAUTH2_CLIENT_SECRET` | 字符串 | `OAuth2ClientSecret()` | 非空校验 | 否 | `auth.go:60` | 否 | ✅ |
| `OAUTH2_REDIRECT_URL` | 字符串 | `OAuth2RedirectURL()` | 非空校验 | 否 | `auth.go:61` | 否 | ✅ |
| `OAUTH2_OIDC_DISCOVERY_ENDPOINT` | 字符串 | `OAuth2OIDCDiscoveryEndpoint()` | 当 provider=oidc 时非空 | 否 | `auth.go:62` | 否 | ✅（同时触发 §13.4 的 HTTP GET） |
| `OAUTH2_OIDC_PROVIDER_NAME` | 字符串 | `OAuth2OIDCProviderName()` | 无 | 否 | 否 | `functions.go:62`（oidcProviderName） | ✅（模板层） |
| `OAUTH2_USER_CREATION` | 布尔 | `OAuth2UserCreation()` | 无 | 否 | `oauth2_callback.go:124` | 否 | ✅（callback 每次判断） |
| `DISABLE_LOCAL_AUTH` | 布尔 | `DisableLocalAuth()` | 与 AUTH_PROXY_HEADER/OAUTH2_PROVIDER 互斥校验 | 否 | `oauth2_unlink.go:17` | `settings.html:11,27` `login.html:44` | ✅（解绑门禁 + 模板渲染） |

### 13.7 安全影响：`DISABLE_LOCAL_AUTH` 的准热生效漏洞

根据上表，`DISABLE_LOCAL_AUTH` 的**启动时校验**（`parser.go`）保证了"禁用本地认证时必须有 OAuth 或 Auth Proxy"，
但这只是一次性校验。

假设：
1. 启动时 `DISABLE_LOCAL_AUTH=true` + `OAUTH2_PROVIDER=oidc` → 校验通过，进程启动
2. 此时如果通过某种手段将内存中 `config.Opts.OAuth2Provider()` 改为 `""`

那么：
- 路由注册保持（启动时 `HasOAuth2Provider()=true` 注册了三条路由）
- 但 `FindProvider("oidc")` 永远找不到 → redirect/callback/unlink 全部 302
- 本地认证 `DISABLE_LOCAL_AUTH=true` 仍生效 → `/login` 模板不渲染密码表单
- `AUTH_PROXY_HEADER` 未配置

**结果：用户失去所有登录方式，等价于"全部锁死"状态。**

这不是严重漏洞（需要能修改进程内存的能力才能触发），但说明了校验层（启动时一次性）与执行层（每次请求读取）之间的**时间差风险**。
正式环境中应通过不可变部署（容器、只读配置）来规避此类风险。
