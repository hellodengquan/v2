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
