# Miniflux 身份认证框架边界分析

## 1. 概述

Miniflux 的身份认证系统由多种认证方式组成，它们共享同一套用户身份模型，但入口和载体各不相同。本文档分析回调绑定、会话续期与外部接口鉴权之间的关联，并指出当前设计中的边界模糊问题。

---

## 2. 核心身份模型

### 2.1 用户实体（User）

所有认证方式最终都收敛到同一个 `User` 实体（`internal/model/user.go:13`）。

**关键身份字段：**

| 字段 | 类型 | 用途 |
|------|------|------|
| `ID` | int64 | 用户唯一标识（主键） |
| `Username` | string | 本地登录用户名 |
| `Password` | string | bcrypt 哈希后的本地密码 |
| `GoogleID` | string | Google OAuth2 外部身份 ID |
| `OpenIDConnectID` | string | OIDC 外部身份 ID |
| `IsAdmin` | bool | 管理员权限标记 |

**设计特点：**
- 单一用户表承载所有身份类型的信息
- 外部身份（Google/OIDC）以字段形式"粘贴"在用户表上
- 没有独立的"身份凭证"表，多种身份混在同一模型中

### 2.2 Web 会话（WebSession）

浏览器会话模型（`internal/model/web_session.go:25`），通过 Cookie 识别。

**结构：**

```
WebSession
├── ID: string                 # 会话标识
├── SecretHash: []byte         # 会话密钥哈希（SHA-256）
├── CreatedAt: time.Time       # 创建时间
├── UserAgent: string          # 浏览器 UA
├── IP: string                 # 客户端 IP
├── userID: *int64             # 绑定的用户 ID（可为 nil）
└── state: webSessionState     # JSON 序列化的会话状态
    ├── CSRF                   # CSRF Token
    ├── OAuth2                 # OAuth2 流程临时状态
    │   ├── State              # OAuth2 state 参数
    │   └── CodeVerifier       # PKCE 代码验证器
    ├── Language               # 会话语言偏好
    ├── Theme                  # 会话主题偏好
    └── ...
```

**关键方法：**
- `SetUser(user)` - 将会话绑定到用户（`internal/model/web_session.go:244`）
- `IsAuthenticated()` - 检查会话是否已认证（`internal/model/web_session.go:98`）
- `Rotate()` - 轮换会话 ID 和密钥，防止会话固定（`internal/model/web_session.go:70`）

### 2.3 API 密钥（APIKey）

REST API 认证用的密钥（`internal/model/api_key.go:13`）。

**结构：**

| 字段 | 类型 | 用途 |
|------|------|------|
| `ID` | int64 | 密钥 ID |
| `UserID` | int64 | 关联用户 ID |
| `Token` | string | 令牌值（32 字节十六进制随机数） |
| `Description` | string | 密钥描述 |
| `LastUsedAt` | *time.Time | 最后使用时间 |
| `CreatedAt` | time.Time | 创建时间 |

---

## 3. 认证入口全景

Miniflux 存在 **6 种独立的认证入口**，最终都映射到同一个 User 实体：

```
                        ┌─────────────────┐
                        │   User 实体     │
                        │  (users 表)     │
                        └────────┬────────┘
                                 │
           ┌───────────┬─────────┼─────────┬───────────┐
           │           │         │         │           │
     ┌─────▼────┐ ┌────▼───┐ ┌──▼────┐ ┌──▼─────┐ ┌───▼───────┐
     │WebSession│ │API Key │ │Basic  │ │Fever   │ │Google     │
     │  Cookie  │ │X-Auth- │ │ Auth  │ │API Key │ │Reader API │
     │          │ │ Token  │ │       │ │        │ │   Token   │
     └──────────┘ └────────┘ └───────┘ └────────┘ └───────────┘
           │
     ┌─────▼──────────────────────┐
     │  登录方式                   │
     │  ───────                   │
     │  1. 本地账号密码            │
     │  2. OAuth2 (Google/OIDC)   │
     │  3. 反向代理认证            │
     │  4. WebAuthn               │
     └────────────────────────────┘
```

---

## 4. OAuth2 回调绑定机制

### 4.1 流程概述

OAuth2 认证流程（`internal/ui/oauth2_redirect.go:15` → `internal/ui/oauth2_callback.go:19`）：

```
用户点击登录
    │
    ▼
oauth2Redirect 处理器
    ├── 生成 state 和 code_verifier (PKCE)
    ├── 存入 WebSession.state.OAuth2
    └── 重定向到 OAuth2 提供商
    │
    ▼ （用户在提供商处登录并授权）
    │
oauth2Callback 处理器
    ├── 验证 state 参数（防 CSRF）
    ├── 用 code + code_verifier 换取 access token
    ├── 获取用户 profile
    │
    ├── 【分支1】用户已登录（绑定账号）
    │   ├── 检查该外部 ID 是否已绑定其他用户
    │   ├── 检查当前用户是否已绑定其他同类型账号
    │   └── 将外部 ID 写入用户表字段
    │
    └── 【分支2】用户未登录（登录/注册）
        ├── 根据外部 ID 查找用户
        ├── 找到 → 直接登录
        └── 未找到 → 自动创建新用户（如配置允许）
            └── 使用 profile.Username 作为本地用户名
```

### 4.2 绑定关系

OAuth2 身份通过**直接修改用户表字段**实现绑定，没有独立的关联表：

- Google 认证 → `users.google_id` 字段
- OIDC 认证 → `users.openid_connect_id` 字段

**绑定逻辑代码位置：**
- 绑定已有用户：`internal/ui/oauth2_callback.go:102` - `authProvider.PopulateUserWithProfileID(loggedUser, profile)`
- 新用户创建时绑定：`internal/ui/oauth2_callback.go:131` - `authProvider.PopulateUserCreationWithProfileID(userCreationRequest, profile)`

### 4.3 单一 Provider 模式与多 IdP 共存的限制

#### 4.3.1 配置层：全局单 Provider

Miniflux 实际上**不支持多 IdP 并存**。通过 `OAUTH2_PROVIDER` 配置项只能选择 `oidc` 或 `google` 其中之一（`internal/config/options.go:464`）：

```go
"OAUTH2_PROVIDER": {
    valueType: stringType,
    validator: func(rawValue string) error {
        return validateChoices(rawValue, []string{"oidc", "google"})
    },
},
```

**全局 OAuth2 配置项：**

| 配置项 | 说明 |
|--------|------|
| `OAUTH2_PROVIDER` | 二选一：`oidc` 或 `google` |
| `OAUTH2_CLIENT_ID` | 单一 Client ID |
| `OAUTH2_CLIENT_SECRET` | 单一 Client Secret |
| `OAUTH2_REDIRECT_URL` | 单一回调地址 |
| `OAUTH2_OIDC_DISCOVERY_ENDPOINT` | 仅 OIDC 使用 |
| `OAUTH2_OIDC_PROVIDER_NAME` | 仅 OIDC 使用，显示名称 |
| `OAUTH2_USER_CREATION` | 是否允许自动创建用户 |

#### 4.3.2 Provider 初始化与路由绑定

Provider 在启动时通过 `oauth2.NewManager()` 创建单一实例（`internal/oauth2/manager.go:15`）：

```go
func NewManager() *Manager {
    provider := config.Opts.OAuth2Provider()
    clientID := config.Opts.OAuth2ClientID()
    clientSecret := config.Opts.OAuth2ClientSecret()
    redirectURL := config.Opts.OAuth2RedirectURL()

    switch provider {
    case "google":
        return &Manager{provider: NewGoogleProvider(clientID, clientSecret, redirectURL)}
    case "oidc":
        discoveryEndpoint := config.Opts.OAuth2OIDCDiscoveryEndpoint()
        return &Manager{provider: NewOIDCProvider(clientID, clientSecret, redirectURL, discoveryEndpoint)}
    default:
        return nil
    }
}
```

**路由绑定**（`internal/ui/ui.go`）：
- 重定向路径：`/oauth2/{provider}/redirect` — 但实际上 `{provider}` 参数在代码中并未用于动态选择 Provider
- 回调路径：`/oauth2/{provider}/callback` — 同样 `{provider}` 仅用于路由匹配

#### 4.3.3 状态参数与会话的绑定

OAuth2 的 state 参数和 code_verifier（PKCE）存储在**当前 WebSession** 的 state JSON 中（`internal/model/web_session.go:49`）：

```
WebSession.state.OAuth2
├── State: string         // 随机生成的防 CSRF state
└── CodeVerifier: string  // PKCE code_verifier
```

**state 生命周期：**
1. `oauth2Redirect` → `session.SetOAuth2State(state)` + `session.SetOAuth2CodeVerifier(verifier)` → 存入会话
2. 用户跳转到 IdP
3. `oauth2Callback` → `session.OAuth2State()` 取出 state 与 URL 参数比对 → 验证通过后清理

**边界问题：**
- **同一浏览器同一时刻只能进行一个 OAuth2 流程**：新的重定向会覆盖旧的 state 和 code_verifier
- **state 与会话强耦合**：如果用户在 OAuth2 流程中切换浏览器或会话过期，state 丢失导致流程失败
- **无 Provider 标识**：state 中不包含是哪个 Provider 的信息，因为全局只有一个 Provider

#### 4.3.4 回调地址绑定

`OAUTH2_REDIRECT_URL` 是全局单一配置，在 Provider 初始化时直接传入构造函数：
- Google：`NewGoogleProvider(clientID, clientSecret, redirectURL)`
- OIDC：`NewOIDCProvider(clientID, clientSecret, redirectURL, discoveryEndpoint)`

**代码路径：**
- `internal/oauth2/google.go:36` → `googleProvider.redirectURL` → `Config().RedirectURL`
- `internal/oauth2/oidc.go:34` → `oidcProvider.redirectURL` → `Config().RedirectURL`

### 4.4 边界问题

1. **身份类型硬编码**：每增加一种 OAuth2 提供商，就需要在 User 结构体和数据库表中新增字段。
2. **单一外部身份限制**：每个用户只能绑定一个 Google ID 和一个 OIDC ID，无法绑定多个同类型身份。
3. **解绑不彻底**：解绑（`oauth2_unlink.go`）只是清空对应字段，没有审计记录。
4. **会话与 OAuth2 状态耦合**：OAuth2 流程状态存储在 WebSession 的 state JSON 中，与会话生命周期绑定。
5. **全局单 Provider 限制**：`OAUTH2_PROVIDER` 只能选 oidc/google 其一，无法同时启用多个 IdP。
6. **state 无并发支持**：同一浏览器同一时刻只能进行一个 OAuth2 流程，后启动的会覆盖前者。

---

## 5. 会话续期机制

### 5.1 会话创建与加载

**会话加载流程**（`internal/ui/web_session_middleware.go:27`）：

```
请求到达
    │
    ▼
从 Cookie 读取 MinifluxSessionID
    │
    ├── Cookie 不存在 → 创建新的未认证会话
    │   ├── 生成随机 ID 和 secret
    │   ├── secret 做 SHA-256 哈希后存储
    │   └── 写入 Cookie: ID.secret
    │
    └── Cookie 存在 → 验证并加载
        ├── 分割 ID 和 secret
        ├── 按 ID 从数据库查会话
        ├── 用 secret 哈希比对（ConstantTimeCompare）
        └── 验证通过 → 放入请求上下文
```

### 5.2 会话续期

**注意：Miniflux 的 WebSession 没有滑动过期机制。**

- `CreatedAt` 只在创建时设置，更新会话状态时不会刷新
- Cookie 的 `Expires` 设置为 `now + CleanupRemoveSessionsInterval`（`internal/ui/auth.go:50`）
- 数据库通过 `CleanOldWebSessions` 定期清理过期会话（`internal/storage/web_session.go:230`）
- 清理周期可配置，但**会话不会因为用户活跃而自动续期**

**会话轮换（Rotate）：**
- 仅在登录认证时调用（`internal/ui/auth.go:26`）
- 目的是防止会话固定攻击（Session Fixation）
- 轮换会生成新的 ID 和 secret，并更新 `created_at`

### 5.3 会话状态持久化

- 会话状态（state）以 JSON 形式存储在 `web_sessions.state` 字段
- 每次请求结束时检查 `IsDirty()`，如有变更则写回数据库（`internal/ui/web_session_middleware.go:59`）
- 这意味着 OAuth2 临时状态、消息提示、语言/主题偏好等都随会话持久化

---

## 6. 外部接口鉴权

### 6.1 REST API v1 鉴权

API v1 使用三层中间件链（`internal/api/api.go:77`）：

```
请求 → withCORSHeaders
     → validateAPIKeyAuth (X-Auth-Token)
     → validateBasicAuth (HTTP Basic)
     → 业务处理器
```

**鉴权逻辑：**

1. **API Key 优先**（`internal/api/middleware.go:37`）：
   - 检查 `X-Auth-Token` 请求头
   - 通过 `api_keys.token` 关联到用户
   - 认证成功 → 写入上下文：`UserID`, `UserTimezone`, `IsAdmin`, `IsAuthenticated`
   - 认证失败 → 返回 401
   - 无 Token → **放行到下一层**（不直接拒绝）

2. **Basic Auth 兜底**（`internal/api/middleware.go:90`）：
   - 仅当前面未认证时才执行
   - 支持 HTTP Basic Authentication
   - 使用本地用户名密码验证
   - 认证成功 → 同样写入上下文

**关键特点：**
- 两种认证方式是**串联**的，API Key 优先
- 上下文是统一的身份传递介质
- 每次认证成功都会更新 `last_login_at`
- API Key 认证还会更新 `api_keys.last_used_at`

### 6.2 Fever API 鉴权

Fever API 使用独立的中间件（`internal/fever/middleware.go:17`）：

- 从 `api_key` 表单参数获取密钥
- 通过 `UserByFeverToken` 查询用户
- 认证成功 → 写入上下文

**注意：** Fever API 使用的是**独立的 token 系统**，与通用 API Key 不共享。

### 6.3 Google Reader API 鉴权

Google Reader API 使用独立的 token 系统（`internal/googlereader/middleware.go:29`）：

- 支持两种传参方式：
  - POST 表单的 `T` 字段
  - `Authorization: GoogleLogin auth=xxx` 头
- Token 格式：`username/hmac_sha256(username+password)`
- Token 存储在 `integrations` 表中，与用户关联

### 6.4 API Key 与 Session 的优先级与冲突处理

#### 6.4.1 路由级别的认证体系隔离

Miniflux 通过**路由前缀**将不同的认证体系完全隔离，避免了 API Key 与 Session 的直接冲突：

| 路由前缀 | 认证方式 | 中间件 | 是否接受 Session Cookie |
|----------|----------|--------|------------------------|
| `/`（Web UI） | WebSession Cookie | `webSessionMiddleware` + `authProxyMiddleware` | ✅ 是，核心认证方式 |
| `/v1/`（REST API） | X-Auth-Token + Basic Auth | `validateAPIKeyAuth` → `validateBasicAuth` | ❌ 否，无 WebSession 中间件 |
| `/fever/` | api_key 参数 | Fever 专用中间件 | ❌ 否，无 WebSession 中间件 |
| `/reader/`（Google Reader） | GoogleLogin Token | GR 专用中间件 | ❌ 否，无 WebSession 中间件 |
| `/metrics` | Basic Auth | 服务器级 Basic Auth 检查 | ❌ 否 |

**路由注册代码**（`internal/http/server/routes.go`）：
```
Web UI Handler   →  webSessionMiddleware → authProxyMiddleware → UI Handler
REST API Handler →  withCORSHeaders → validateAPIKeyAuth → validateBasicAuth → API Handler
Fever Handler    →  Fever Middleware → Fever Handler
GR Handler       →  GR Auth Middleware → GR Handler
```

#### 6.4.2 REST API v1 的内部优先级

REST API v1 内部存在两层认证串联（`internal/api/api.go:77`）：

```
请求
  │
  ├─ validateAPIKeyAuth (X-Auth-Token)
  │    ├─ Token 有效 → 写入上下文 → 跳过下一层
  │    ├─ Token 无效 → 返回 401
  │    └─ 无 Token   → 放行到下一层
  │
  └─ validateBasicAuth (HTTP Basic)
       ├─ 凭证有效 → 写入上下文
       ├─ 凭证无效 → 返回 401
       └─ 无凭证 → 返回 401
```

**关键代码逻辑**（`internal/api/middleware.go:37`）：
```go
func (m *authMiddleware) validateAPIKeyAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 如果上下文已有身份（理论上不会发生），直接跳过
        if request.IsAuthenticated(r) {
            next.ServeHTTP(w, r)
            return
        }
        // 无 X-Auth-Token → 放行到 Basic Auth
        if r.Header.Get("X-Auth-Token") == "" {
            next.ServeHTTP(w, r)
            return
        }
        // Token 验证失败 → 直接 401
        ...
    })
}
```

**冲突处理机制：**
- `validateAPIKeyAuth` 入口检查 `request.IsAuthenticated(r)`，如果上下文已有身份则直接跳过
- Basic Auth 中间件同理：已认证则跳过
- 两层中间件是串联的，不存在同时使用两种凭证认证同一请求的情况

#### 6.4.3 Web UI 的反向代理降级

Web UI 路径中除了 WebSession 外，还存在 `authProxyMiddleware`（`internal/ui/auth_proxy_middleware.go:26`）：

```
请求到达 Web UI
  │
  ├─ webSessionMiddleware → 加载/创建 WebSession
  │
  └─ authProxyMiddleware
       ├─ request.IsAuthenticated(r) == true → 跳过（已有 Session）
       ├─ AUTH_PROXY_HEADER 未配置 → 跳过
       ├─ 请求来源不在可信网络 → 跳过
       ├─ Header 中无用户名 → 跳过
       └─ Header 中有用户名 → 查找/创建用户 → 建立 WebSession → 重定向
```

**这是 Web UI 层唯一的"降级/旁路"路径**：反向代理认证可以绕过本地登录直接建立会话。

#### 6.4.4 上下文双通道的隐含优先级

`request.IsAuthenticated()` 和 `request.UserID()` 的双通道回退设计（`internal/http/request/context.go:48`）隐含了优先级：

```go
func UserID(r *http.Request) int64 {
    // 通道1: 上下文直存值（API Key / Basic Auth / Fever / GR）
    if userID, ok := getContextInt64Value(r, UserIDContextKey); ok {
        return userID
    }
    // 通道2: WebSession 对象（Web UI）
    if session := WebSession(r); session != nil {
        return session.UserID()
    }
    return 0
}
```

**优先级：上下文直存 > WebSession**

但由于路由隔离，实际生产中同一请求不会同时存在两种身份源。只有在中间件链错误配置时才可能出现冲突，此时上下文直存值优先级更高。

### 6.5 外部 Webhook / Fever / GoogleReader 兼容接口的鉴权降级路径

#### 6.5.1 Webhook：出站调用，无入站鉴权

Webhook 是 Miniflux 作为**客户端**向外部服务发起的调用，不存在入站鉴权问题（`internal/integration/webhook/webhook.go:117`）：

```
Miniflux (Client) ──POST──▶ 第三方 Webhook URL
  │
  ├─ Header: X-Miniflux-Signature: HMAC-SHA256(webhook_secret, body)
  ├─ Header: X-Miniflux-Event-Type: new_entries | save_entry
  └─ Body: JSON Payload
```

**Webhook 凭证管理**（`internal/model/integration.go:93`）：
- `WebhookEnabled`：是否启用
- `WebhookURL`：全局 Webhook URL（可被 Feed 级覆盖）
- `WebhookSecret`：HMAC 签名密钥

**Feed 级覆盖**（`internal/integration/integration.go:422`）：
```go
var webhookURL string
if entry.Feed != nil && entry.Feed.WebhookURL != "" {
    webhookURL = entry.Feed.WebhookURL  // Feed 级优先
} else {
    webhookURL = userIntegrations.WebhookURL  // 用户级兜底
}
```

Webhook 方向是出站的，Miniflux 只负责签名，不接收外部 Webhook 请求，因此无入站鉴权降级。

#### 6.5.2 Fever API：单一 Token 鉴权，无降级

Fever API 的鉴权是**单一模式**，没有任何降级路径（`internal/fever/middleware.go:17`）：

```
请求到达 /fever/
  │
  ├─ 读取 api_key 参数（POST 表单 or GET query）
  ├─ api_key 为空 → 返回 auth_version + last_refreshed_on_time + 0 api_id（未认证标志）
  ├─ api_key 存在 → 查询 integrations.fever_token
  │    ├─ 找到且 fever_enabled='t' → 认证成功，写入上下文
  │    └─ 未找到/未启用 → 返回空响应（等价于未认证）
  └─ 无任何其他认证方式
```

**Fever Token 存储**（`internal/storage/integration.go:32`）：
```sql
SELECT users.id, ...
FROM users
LEFT JOIN integrations ON integrations.user_id=users.id
WHERE integrations.fever_enabled='t' AND lower(integrations.fever_token)=lower($1)
```

**鉴权降级路径：无**
- Fever API 不接受 Session Cookie
- 不接受 HTTP Basic Auth
- 不接受 X-Auth-Token
- 唯一凭证：`api_key` 参数 → `integrations.fever_token`

**兼容特性**：Fever API 即使认证失败，也不会返回 401，而是返回带有 `api_id=0` 的基础响应，这是 Fever 协议规范要求的"软失败"模式。

#### 6.5.3 Google Reader API：两阶段鉴权，ClientLogin 为降级入口

Google Reader API 采用**两阶段认证**模式，存在 ClientLogin → Token 的降级/兑换路径：

```
阶段1: ClientLogin（用户名密码认证，换取 Token）
  POST /accounts/ClientLogin
    ├─ Form: Email + Passwd
    ├─ 查询 integrations.googlereader_username + bcrypt(googlereader_password)
    ├─ 验证通过 → 返回 Token: username/hmac_sha256(username+password)
    └─ 验证失败 → 401

阶段2: Token 认证（所有 /reader/api/0/* 接口）
  GET/POST /reader/api/0/...
    ├─ POST Form: T=<token>
    │  或
    ├─ Header: Authorization: GoogleLogin auth=<token>
    │
    ├─ Token 格式校验: username/hash
    ├─ 从 integrations 表查用户
    ├─ 重新计算 HMAC 比对（ConstantTimeCmp）
    └─ 验证通过 → 写入上下文
```

**Token 生成机制**（`internal/googlereader/middleware.go:182`）：
```go
func getAuthToken(username, password string) string {
    // Token = username + "/" + hex(hmac_sha256(username+password))
    token := hex.EncodeToString(hmac.New(sha256.New, []byte(username+password)).Sum(nil))
    token = username + "/" + token
    return token
}
```

**关键特性：**
- Token **不是随机值**，而是用户名和密码的确定性 HMAC 推导值
- 只要用户名和密码不变，Token 永远相同（无过期，无轮换）
- `/reader/api/0/token` 接口可以在已认证状态下重新获取 Token（`internal/googlereader/handler.go:149`）
- Google Reader API 同样不接受 Session Cookie

#### 6.5.4 兼容接口鉴权降级总结

| 接口 | 主认证方式 | 降级/旁路路径 | 是否接受 Session |
|------|-----------|--------------|-----------------|
| Webhook（出站） | HMAC-SHA256 签名 | 无（方向是出站） | N/A |
| Fever API | `api_key` 参数 → integrations.fever_token | **无**，api_id=0 软失败 | ❌ |
| Google Reader API | GoogleLogin Token | ClientLogin（用户名/密码换 Token） | ❌ |
| REST API v1 | X-Auth-Token → Basic Auth | Basic Auth 兜底 | ❌ |
| Web UI | WebSession Cookie | 反向代理 Header 认证 | ✅ |

### 6.6 鉴权边界问题

1. **多种 API 认证系统并存**：REST API Key、Fever Token、Google Reader Token 三套独立的凭证体系，都关联到同一个用户。
2. **上下文键混用**：所有认证方式都写入同一组上下文键（`UserIDContextKey`, `IsAuthenticatedContextKey` 等），但 Google Reader API 额外有 `GoogleReaderTokenKey`。
3. **认证失败处理不一致**：
   - REST API：无 API Key 时放行到下一层（Basic Auth）
   - Fever API：无 API Key 返回软失败响应（api_id=0）
   - Google Reader API：无 Token 返回 401
   - Web UI：未认证重定向到登录页
4. **权限粒度粗**：只有"是否管理员"二元权限，没有针对 API Key 的细粒度权限控制。
5. **路由隔离依赖人工维护**：API Key 与 Session 的不冲突完全依赖路由前缀正确，没有底层机制强制隔离。
6. **GR Token 永不过期**：Google Reader Token 由用户名密码确定性推导，无法单独吊销，修改密码会使所有 Token 失效。

---

## 7. 统一身份上下文

### 7.1 请求上下文键

所有认证方式最终通过 `context.Context` 传递身份信息（`internal/http/request/context.go:16`）：

| Context Key | 类型 | 说明 |
|-------------|------|------|
| `UserIDContextKey` | int64 | 用户 ID |
| `UserNameContextKey` | string | 用户名 |
| `UserTimezoneContextKey` | string | 用户时区 |
| `IsAdminUserContextKey` | bool | 是否管理员 |
| `IsAuthenticatedContextKey` | bool | 是否已认证 |
| `WebSessionContextKey` | *WebSession | Web 会话对象 |
| `GoogleReaderTokenKey` | string | Google Reader Token |

### 7.2 身份获取的双通道

`request.IsAuthenticated()` 和 `request.UserID()` 采用**双通道回退**设计（`internal/http/request/context.go:48`）：

```go
func IsAuthenticated(r *http.Request) bool {
    // 通道1: 直接从上下文值获取（API 认证方式）
    if getContextBoolValue(r, IsAuthenticatedContextKey) {
        return true
    }
    // 通道2: 从 WebSession 间接获取（Web UI 方式）
    if session := WebSession(r); session != nil {
        return session.IsAuthenticated()
    }
    return false
}
```

**边界问题：**
- 两种身份来源（直接上下文 vs 会话对象）并存，增加了理解和维护成本
- Web UI 请求中，即使会话已认证，`IsAuthenticatedContextKey` 也可能为 false
- 代码需要同时处理两种情况，容易遗漏

---

## 8. 边界模糊点总结

### 8.1 用户模型边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 身份凭证混在一起 | 本地密码、Google ID、OIDC ID 都在 users 表 | `internal/model/user.go:13` |
| 缺少身份抽象层 | 没有统一的 Identity 接口或实体 | - |
| 扩展困难 | 新增认证方式需修改 User 结构和数据库迁移 | - |
| integrations 表承载 API 凭证 | Fever Token、GR 用户名密码 + 20+ 种第三方集成凭证塞在同一张表 | `internal/model/integration.go:7` |

### 8.2 会话边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 会话既是认证载体又是状态存储 | OAuth2 流程状态、消息、偏好都塞在 session state 里 | `internal/model/web_session.go:37` |
| 无滑动续期 | 会话创建时间不随活动刷新，可能导致用户活跃中被登出 | `internal/storage/web_session.go:183` |
| 认证状态双重来源 | 上下文直存 + 会话对象，两种方式并存 | `internal/http/request/context.go:48` |
| OAuth2 state 无并发支持 | 同一浏览器同一时刻只能进行一个 OAuth2 流程 | `internal/model/web_session.go:49` |

### 8.3 OAuth2 边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 外部身份直接写入用户表 | 没有独立的 user_identities 关联表 | `internal/model/user.go:26-27` |
| OAuth2 状态与会话耦合 | state 和 code_verifier 存在会话 state JSON 中 | `internal/model/web_session.go:49` |
| 自动创建用户权限边界不清 | OAuth2 登录可能绕过本地用户创建策略 | `internal/ui/oauth2_callback.go:120` |
| 全局单 Provider 限制 | 只能在 Google 和 OIDC 之间二选一，无法多 IdP 并存 | `internal/config/options.go:464` |
| 回调地址硬编码为单实例 | `OAUTH2_REDIRECT_URL` 全局唯一，不支持多回调 | `internal/oauth2/manager.go:15` |

### 8.4 API 鉴权边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 三套 API 凭证体系 | REST API Key、Fever Token、Google Reader Token 各自独立 | `internal/storage/api_key.go`, `internal/fever/middleware.go`, `internal/googlereader/middleware.go` |
| 认证中间件行为不一致 | REST API 放行兜底、Fever 软失败、GR 硬 401、Web UI 重定向 | 各 middleware.go |
| Token 与 Session 隔离依赖路由 | 不同前缀挂不同中间件链，无底层强制隔离 | `internal/http/server/routes.go` |
| GR Token 永不过期 | 由用户名密码确定性推导，无法单独吊销 | `internal/googlereader/middleware.go:182` |
| integrations 表凭证职责不清 | Fever/GR API 凭证 + 第三方集成密钥混存 | `internal/storage/integration.go:109` |

### 8.5 兼容接口鉴权降级模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| Fever 软失败与 GR 硬失败 | Fever 返回 api_id=0，GR 返回 401，语义不一致 | `internal/fever/middleware.go:17` vs `internal/googlereader/middleware.go:35` |
| REST API Basic Auth 兜底 | X-Auth-Token 不存在时静默降级到用户名密码 | `internal/api/middleware.go:90` |
| Web UI 反向代理旁路 | 可信网络 + Header 可绕过本地认证直接建会话 | `internal/ui/auth_proxy_middleware.go:26` |
| GR ClientLogin 暴露密码认证 | 兼容接口接受明文用户名密码（HTTPS 保护） | `internal/googlereader/handler.go:72` |

---

## 9. 数据流汇总图

```
┌──────────────────────────────────────────────────────────────────┐
│                         HTTP 请求                                 │
└───────────────────────────────┬──────────────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          ▼                     ▼                     ▼
    ┌───────────┐          ┌─────────┐          ┌───────────┐
    │  Web UI   │          │ REST API│          │  其他API  │
    │  路由     │          │ v1 路由 │          │ (Fever/GR)│
    └─────┬─────┘          └────┬────┘          └─────┬─────┘
          │                     │                      │
          ▼                     ▼                      ▼
    WebSession           validateAPIKeyAuth        Fever/GR 专用
    中间件               validateBasicAuth          中间件
          │                     │                      │
          │                     └─────────┬────────────┘
          │                               │
          │                               ▼
          │                     Context 写入身份
          │                     (UserID, IsAdmin, ...)
          │                               │
          │                               │
          └───────────────┬───────────────┘
                          │
                          ▼
                  业务处理器 / Handler
                          │
                          ▼
                  读取上下文身份
                  (request.UserID(),
                   request.IsAdmin(),
                   request.IsAuthenticated())
```

---

## 10. 代码索引

### 10.1 核心身份模型

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 用户模型 | `internal/model/user.go` | 13 |
| Web 会话模型 | `internal/model/web_session.go` | 25 |
| API Key 模型 | `internal/model/api_key.go` | 13 |
| 集成/兼容 API 凭证模型 | `internal/model/integration.go` | 7 |

### 10.2 OAuth2 与多 IdP

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| OAuth2 Provider 接口 | `internal/oauth2/provider.go` | 15 |
| OAuth2 Manager（单 Provider 工厂） | `internal/oauth2/manager.go` | 15 |
| Google Provider 实现 | `internal/oauth2/google.go` | 29 |
| OIDC Provider 实现 | `internal/oauth2/oidc.go` | - |
| OAuth2 配置项校验 | `internal/config/options.go` | 464 |
| OAuth2 重定向处理器 | `internal/ui/oauth2_redirect.go` | 15 |
| OAuth2 回调处理器 | `internal/ui/oauth2_callback.go` | 19 |

### 10.3 会话与认证中间件

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| Web 会话中间件 | `internal/ui/web_session_middleware.go` | 27 |
| Web 会话公共路由判断 | `internal/ui/routes.go` | 12, 31 |
| Web 会话认证（登录绑定） | `internal/ui/auth.go` | 22 |
| 本地登录检查 | `internal/ui/login_check.go` | 20 |
| 反向代理认证中间件 | `internal/ui/auth_proxy_middleware.go` | 26 |
| 请求上下文与双通道身份 | `internal/http/request/context.go` | 16, 48 |

### 10.4 API 鉴权中间件

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| REST API 中间件（API Key + Basic） | `internal/api/middleware.go` | 37, 90 |
| REST API Handler 组装 | `internal/api/api.go` | 77 |
| Fever API 中间件 | `internal/fever/middleware.go` | 17 |
| Fever API Handler | `internal/fever/handler.go` | 31 |
| Google Reader API 中间件 | `internal/googlereader/middleware.go` | 29 |
| Google Reader ClientLogin | `internal/googlereader/handler.go` | 72 |
| Google Reader Token 生成 | `internal/googlereader/middleware.go` | 182 |
| 服务器路由注册 | `internal/http/server/routes.go` | - |

### 10.5 存储层

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 用户存储 | `internal/storage/user.go` | 58, 481 |
| 会话存储 | `internal/storage/web_session.go` | 16, 97 |
| API Key 存储 | `internal/storage/api_key.go` | 73 |
| 集成/兼容 API 凭证存储 | `internal/storage/integration.go` | 32, 56, 109 |

### 10.6 Webhook 与出站集成

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| Webhook 出站客户端 | `internal/integration/webhook/webhook.go` | 28, 117 |
| 集成调度入口 | `internal/integration/integration.go` | 41, 511 |
