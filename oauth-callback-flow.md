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
