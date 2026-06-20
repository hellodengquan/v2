# Miniflux 第三方集成 Token 轮换与失效检测机制分析

## 概述

Miniflux 的第三方集成 Token 管理体系可以分为两个完全独立的子系统：

1. **Miniflux 自身的用户认证 Token 系统**（Web Session、API Key、Fever Token、Google Reader Token、OAuth2 登录）
2. **Miniflux 作为客户端访问第三方服务时的 Token 管理**（Wallabag、Shaarli、Shiori、Matrix 等）

两者的轮换与失效检测策略截然不同。

---

## 一、Miniflux 自身用户认证 Token 体系

### 1.1 Web Session Token 轮换

**核心文件：**
- `internal/model/web_session.go` — Session 模型与轮换逻辑
- `internal/ui/auth.go` — 认证入口与 Cookie 设置
- `internal/ui/web_session_middleware.go` — Session 加载/验证中间件
- `internal/storage/web_session.go` — Session 持久化与清理

**轮换流程：**

```
用户登录 → authenticateWebSession()
  ├── session.SetUser(user)          // 绑定用户到 session
  ├── session.Rotate()               // 轮换 session ID + secret
  │   ├── 生成新 ID (rand.Text())
  │   ├── 生成新 secret (rand.Text())
  │   ├── 新 secret 做 SHA-256 哈希存入 SecretHash
  │   └── 返回 (oldID, newSecret)
  ├── store.RotateWebSession(oldID, session)  // 原子更新数据库行
  │   └── UPDATE web_sessions SET id=$2, secret_hash=$3, user_id=$4, state=$5, created_at=now() WHERE id=$1
  └── setSessionCookie(w, session, secret)    // 写入新 Cookie
      └── Cookie: MinifluxSessionID = {sessionID}.{secret}
          Expires = now() + CLEANUP_REMOVE_SESSIONS_DAYS (默认30天)
```

**关键设计点：**

- **认证时轮换防 Session Fixation**：`Rotate()` 在 `web_session.go:70` 中实现，每次成功认证后强制替换 session ID 和 secret，防止 session fixation 攻击。
- **Secret 使用 SHA-256 哈希存储**：数据库中只存 `SecretHash`，不存明文 secret，验证时通过 `subtle.ConstantTimeCompare` 进行常量时间比较防时序攻击。
- **Cookie 格式**：`{sessionID}.{rawSecret}`，中间件从 cookie 解析出 ID 和 secret 后分别查询和验证。
- **无预刷新机制**：Miniflux 的 Web Session 没有过期前的主动续期。Session 的"存活"完全依赖 `CLEANUP_REMOVE_SESSIONS_DAYS` 配置的定时清理，而非 token 本身的 TTL。

### 1.2 Web Session 失效检测与清理

**失效判定方式：基于创建时间的被动清理**

```
调度器 (scheduler.go) → cleanupScheduler() → runCleanupTasks()
  └── store.CleanOldWebSessions(interval)
      └── DELETE FROM web_sessions WHERE created_at < now() - $1::interval
```

- 清理间隔由 `CLEANUP_FREQUENCY` 配置控制（调度器的运行频率）
- Session 存活时长由 `CLEANUP_REMOVE_SESSIONS_DAYS` 控制（默认 30 天，最小 1 天）
- **无主动失效检测**：不检查 session 是否仍在活跃使用，仅按创建时间一刀切删除
- **无滚动续期**：不像 JWT 的 refresh token 那样在每次使用时延长有效期

**手动失效：**

- `store.FlushAllSessions()`：CLI 的 `-flush-sessions` 命令，删除所有 session，强制全员下线
- `store.RemoveUserWebSession(userID, sessionID)`：用户主动注销某个 session
- Session 中间件验证失败时（cookie 为空、ID/secret 不匹配、数据库查无此 session），静默创建新 session

### 1.3 API Key 认证（无轮换）

**核心文件：** `internal/api/middleware.go`, `internal/model/api_key.go`

- API Key 是长期有效的静态 token，存储在 `api_keys` 表中
- 认证时通过 `X-Auth-Token` Header 传入
- **无过期时间，无轮换机制**：一旦创建，永久有效直到手动删除
- 唯一的状态追踪是 `LastUsedAt` 字段（`store.SetAPIKeyUsedTimestamp()`），仅用于审计
- 失效检测完全被动：只在请求时查数据库，查不到就返回 401

### 1.4 Fever API Token（无轮换）

**核心文件：** `internal/fever/middleware.go`

- Fever 使用 `api_key` 字段认证（MD5 哈希生成，存储在 `user_integrations.fever_token`）
- **无过期、无轮换**：fever token 是从用户密码派生的静态值
- 每次请求时查数据库 `store.UserByFeverToken(apiKey)`，查不到即失效

### 1.5 Google Reader Token（无轮换）

**核心文件：** `internal/googlereader/middleware.go`

- Token 格式：`username/HMAC-SHA256(username+password)`
- 认证时从 `Authorization: GoogleLogin auth=xxx` 或 POST 表单 `T` 字段获取
- 通过 `getAuthToken()` 重新计算期望 token，然后与提交的 token 做常量时间比较
- **无过期、无轮换**：token 完全由用户名+密码决定，改变密码即失效

### 1.6 OAuth2 用户登录（一次性 Code Exchange，无持久 Token）

**核心文件：**
- `internal/oauth2/provider.go` — Provider 接口定义
- `internal/oauth2/authorization.go` — PKCE 授权生成
- `internal/oauth2/oidc.go` — OIDC Provider 实现
- `internal/oauth2/google.go` — Google Provider 实现
- `internal/ui/oauth2_redirect.go` — 重定向到 OAuth2 提供方
- `internal/ui/oauth2_callback.go` — 处理 OAuth2 回调

**流程：**

```
用户点击 OAuth2 登录 → oauth2Redirect()
  ├── authProvider = manager.FindProvider(provider)
  ├── auth = GenerateAuthorization(authProvider.Config())
  │   ├── 生成 codeVerifier (随机32字节hex)
  │   ├── codeChallenge = SHA256(codeVerifier) 的 base64url
  │   ├── 生成 state (随机24字节hex, 防 CSRF)
  │   └── 拼接 AuthCodeURL(state, code_challenge_method=S256, code_challenge=...)
  ├── session.StartOAuth2Flow(auth.State(), auth.CodeVerifier())  // 存入 session state
  └── 302 重定向到 OAuth2 提供方

用户在 OAuth2 提供方授权后回调 → oauth2Callback()
  ├── 验证 state 参数 (constant-time compare)
  ├── 取出 codeVerifier (从 session state)
  ├── session.ClearOAuth2Flow()
  ├── authProvider.Profile(ctx, code, codeVerifier)
  │   ├── conf.Exchange(ctx, code, code_verifier=codeVerifier)  // 换取 token
  │   ├── 从 token 中提取 id_token (OIDC) / 调用 userinfo 端点 (Google)
  │   └── 返回 UserProfile{Key, ID, Username}
  ├── 查找或创建本地用户
  └── authenticateWebSession(w, r, store, user)  // 绑定到 Web Session
```

**关键设计点：**

- **PKCE 保护**：使用 S256 code challenge 方法，verifier 存在 session state 中，回调时一次性消费
- **OAuth2 Token 不持久化**：Miniflux 只用 OAuth2 来完成用户身份验证（获取 profile），**不保存 access_token/refresh_token**。认证完成后，用户会话完全由 Miniflux 自己的 Web Session 系统管理。
- **无 Token 刷新**：因为不持有 OAuth2 的 access_token，也就不存在 refresh 逻辑
- **Provider 管理**：`oauth2.Manager` 管理 OIDC 和 Google 两个 provider，每次回调时通过 `getOAuth2Manager()` 临时重建 Manager 实例

### 1.7 小结：Miniflux 自身认证 Token 体系对比

| Token 类型 | 轮换机制 | 预刷新 | 失效检测 | 回退路径 |
|---|---|---|---|---|
| Web Session | 认证时轮换 ID+Secret | ❌ 无 | 定时清理过期 session | Cookie 验证失败→创建新 session→跳转登录页 |
| API Key | ❌ 无 | ❌ 无 | 请求时查库 | 401 Unauthorized |
| Fever Token | ❌ 无 | ❌ 无 | 请求时查库 | 返回 auth failure JSON |
| Google Reader Token | ❌ 无 | ❌ 无 | 请求时重新计算验证 | 返回 401 |
| OAuth2 (登录) | ❌ 一次性 code exchange | ❌ 不适用 | state 验证失败 | 重定向到首页 |

---

## 二、Miniflux 访问第三方服务时的 Token 管理

这是"集成 token 轮换"问题的核心。Miniflux 作为客户端，需要向第三方服务（Wallabag、Shaarli、Shiori、Matrix 等）发送请求时携带认证信息。

### 2.1 认证模式分类

根据第三方集成的认证方式，可以分为以下四类：

#### 类型 A：静态 API Key / Token（无轮换需求）

这类集成使用用户提供的静态 API Key 或 Token，不会过期：

| 集成 | Token 字段 | 认证方式 |
|---|---|---|
| Pinboard | `PinboardToken` | API Token |
| Notion | `NotionToken` | Bearer Token |
| Espial | `EspialAPIKey` | API Key |
| Readwise | `ReadwiseAPIKey` | Bearer Token |
| LinkAce | `LinkAceAPIKey` | Bearer Token |
| Linkding | `LinkdingAPIKey` | Bearer Token |
| Linktaco | `LinktacoAPIToken` | Bearer Token |
| Linkwarden | `LinkwardenAPIKey` | Bearer Token |
| Readeck | `ReadeckAPIKey` | Bearer Token |
| Omnivore | `OmnivoreAPIKey` | Bearer Token |
| Karakeep | `KarakeepAPIKey` | Bearer Token |
| Raindrop | `RaindropToken` | Bearer Token |
| Betula | `BetulaToken` | Bearer Token |
| NunuxKeeper | `NunuxKeeperAPIKey` | Basic Auth |
| Cubox | `CuboxAPILink` | URL 内嵌 |
| Webhook | `WebhookSecret` | HMAC 签名 |
| Discord | `DiscordWebhookLink` | URL 内嵌 |
| Slack | `SlackWebhookLink` | URL 内嵌 |
| Apprise | `AppriseURL` + `AppriseServicesURL` | URL 内嵌 |
| Pushover | `PushoverUser` + `PushoverToken` | API 参数 |

**这些集成的共同特征：**
- Token 存储在 `user_integrations` 表中，由用户在设置页面手动填入
- **无过期时间，无轮换，无刷新**
- 失效时返回错误日志，无自动回退
- 用户需要自行在第三方平台管理 token 的生命周期

#### 类型 B：每次请求前重新获取 Token（无状态认证）

这类集成在每次调用时先用用户名/密码换取临时 token，然后使用该 token 发起实际请求：

##### Wallabag（OAuth2 Resource Owner Password Grant）

**核心文件：** `internal/integration/wallabag/wallabag.go`

```
CreateEntry(entryURL, entryTitle, entryContent)
  ├── accessToken = getAccessToken()                    // 每次都重新获取
  │   ├── POST {baseURL}/oauth/v2/token
  │   │   grant_type=password
  │   │   client_id + client_secret
  │   │   username + password
  │   ├── 解析 tokenResponse { access_token, expires_in, refresh_token, scope, token_type }
  │   └── 返回 access_token
  └── createEntry(accessToken, ...)                     // 使用 token 调用 API
      ├── POST {baseURL}/api/entries.json
      ├── Authorization: Bearer {accessToken}
      └── response.StatusCode >= 400 → 返回错误
```

**关键问题：**

1. **忽略了 refresh_token**：`tokenResponse` 结构体中解析了 `refresh_token` 和 `expires_in` 字段，但**完全未使用**。每次调用都使用 `grant_type=password` 重新获取 token。
2. **无 Token 缓存**：即使短时间内多次调用（如批量推送条目），每次都会发起独立的 token 请求。
3. **无预刷新**：因为每次都重新获取 token，不存在"过期前刷新"的概念。
4. **失效检测**：仅在 HTTP 响应状态码 >= 400 时判定失败，无更细粒度的 token 过期检测。
5. **无回退路径**：获取 token 失败或使用 token 失败都直接返回错误，不会尝试 refresh_token 或其他回退策略。

##### Shiori（用户名密码登录换 Token）

**核心文件：** `internal/integration/shiori/shiori.go`

```
CreateBookmark(entryURL, entryTitle)
  ├── token = authenticate()                           // 每次都重新登录
  │   ├── POST {baseURL}/api/v1/auth/login
  │   │   { username, password, remember_me: false }
  │   ├── 解析 authResponse { ok, message: { session, token } }
  │   └── 返回 message.token
  └── POST {baseURL}/api/bookmarks
      ├── Authorization: Bearer {token}
      └── response.StatusCode != 200 → 返回错误
```

- 同样是**每次请求都重新认证**，不缓存 token
- 登录响应中的 `session` 字段未被使用
- 无预刷新、无回退

##### Matrix Bot（每次推送都完整登录流程）

**核心文件：** `internal/integration/matrixbot/matrixbot.go`, `internal/integration/matrixbot/client.go`

```
PushEntries(feed, entries, matrixBaseURL, username, password, roomID)
  ├── DiscoverEndpoints()                              // 发现 .well-known
  │   └── GET {matrixBaseURL}/.well-known/matrix/client
  ├── Login(homeServerURL, username, password)         // 每次都登录
  │   ├── POST {homeServerURL}/_matrix/client/v3/login
  │   │   type=m.login.password
  │   │   identifier: { type: m.id.user, user: username }
  │   │   password
  │   └── 返回 LoginResponse { user_id, access_token, device_id, home_server }
  └── SendFormattedTextMessage(homeServerURL, accessToken, roomID, text, formattedText)
      ├── PUT {homeServerURL}/_matrix/client/v3/rooms/{roomID}/send/m.room.message/{txnID}
      ├── Authorization: Bearer {accessToken}
      └── 返回 RoomEventResponse { event_id }
```

- **三步流程，每步都可能失败**：发现端点 → 登录 → 发送消息
- 登录获取的 `access_token` 和 `device_id` 仅在当次请求中使用
- **长期后果**：每次推送都会创建一个新的 Matrix device session，可能在 Matrix 服务器上积累大量设备

#### 类型 C：基于密钥的即时签名 Token（无需远程获取）

##### Shaarli（JWT 签名，本地生成）

**核心文件：** `internal/integration/shaarli/shaarli.go`

```
CreateLink(entryURL, entryTitle)
  ├── bearerToken = generateBearerToken()              // 本地生成 JWT
  │   ├── header = base64url({"typ":"JWT","alg":"HS512"})
  │   ├── payload = base64url({"iat": 当前Unix时间戳})
  │   ├── signature = base64url(HMAC-SHA512(header.payload, apiSecret))
  │   └── JWT = header.payload.signature
  └── POST {baseURL}/api/v1/links
      ├── Authorization: Bearer {JWT}
      └── response.StatusCode != 201 → 返回错误
```

- **无需远程获取 token**：用 API Secret 本地签发 JWT
- JWT 的 `iat`（签发时间）设为当前时间，由 Shaarli 服务端验证时效性
- 无需缓存、无需刷新——每次请求生成新 JWT 即可
- 失效检测仅看 HTTP 状态码

##### Ntfy（Token + Basic Auth 双重认证）

**核心文件：** `internal/integration/ntfy/ntfy.go`

- 优先使用 Bearer Token（`ntfyApiToken`），其次使用 Basic Auth（`ntfyUsername` + `ntfyPassword`）
- 两者可以同时设置，但**不是回退关系**——两者都会被设置到请求头中

#### 类型 D：用户名密码直接认证（无 Token）

##### Instapaper

- 直接使用 `username` + `password` 作为请求参数（不是 HTTP Auth）
- 无 token 概念

---

## 三、核心发现：Miniflux 第三方集成中"不存在"传统意义上的 Token 轮换

### 3.1 没有预刷新机制

**Miniflux 的所有第三方集成都没有实现 token 过期前的预刷新。** 原因如下：

1. **静态 API Key 类集成**（类型 A）：Token 本身不过期，不需要刷新
2. **每次重取 Token 类集成**（类型 B）：每次请求都重新认证，绕过了 token 过期问题
3. **本地签名类集成**（类型 C）：每次请求本地生成新 token，不存在过期

### 3.2 没有传统 Token 缓存与 Refresh 机制

Wallabag 是最接近需要 refresh token 机制的集成——它的 OAuth2 端点返回了 `refresh_token` 和 `expires_in`，但 Miniflux 完全忽略了这些字段：

```go
// wallabag.go:144-150 — 解析了但未使用
type tokenResponse struct {
    AccessToken  string `json:"access_token"`
    Expires      int    `json:"expires_in"`      // ← 忽略
    RefreshToken string `json:"refresh_token"`   // ← 忽略
    Scope        string `json:"scope"`
    TokenType    string `json:"token_type"`
}
```

### 3.3 失效检测与回退路径

**统一的失效处理模式：**

```
请求第三方服务 → 失败 → slog.Error/Warn → 返回 error → 上层忽略错误
```

具体路径：

1. `integration.SendEntry()` 调用各集成的 `CreateEntry()`/`CreateBookmark()` 等
2. 集成客户端发起 HTTP 请求
3. 失败时返回 error
4. `SendEntry()` 中用 `slog.Error()` 记录日志，**然后继续处理下一个集成**
5. **没有任何回退操作**：不重试、不刷新 token、不标记集成失效、不通知用户

```go
// integration.go:127-136 — 典型的错误处理模式
if err := client.CreateEntry(entry.URL, entry.Title, entry.Content); err != nil {
    slog.Error("Unable to send entry to Wallabag",
        slog.Int64("user_id", userIntegrations.UserID),
        slog.Int64("entry_id", entry.ID),
        slog.String("entry_url", entry.URL),
        slog.Any("error", err),
    )
    // ← 错误仅被记录，无任何回退
}
```

### 3.4 两种推送入口的差异

| 入口 | 函数 | 调用时机 | 错误级别 |
|---|---|---|---|
| 手动保存 | `SendEntry()` | 用户点击"Save"按钮 | `slog.Error` |
| 自动推送 | `PushEntries()` | Feed 刷新时自动推送新条目 | 部分用 `slog.Warn` |

`PushEntries()` 对部分集成（Webhook、Ntfy、Apprise、Discord、Slack、Pushover）使用了 `slog.Warn` 而非 `slog.Error`，这暗示自动推送场景下失败更可容忍。

---

## 四、Web Session 轮换的完整时序

这是 Miniflux 中唯一真正意义上的"Token 轮换"：

```
[浏览器]                          [Miniflux Server]                      [PostgreSQL]
   │                                   │                                    │
   │  GET / (无 Cookie)                │                                    │
   │──────────────────────────────────>│                                    │
   │                                   │  NewWebSession()                   │
   │                                   │  ├─ 生成 ID, Secret                │
   │                                   │  ├─ SHA256(Secret) → SecretHash    │
   │                                   │  └─ 生成 CSRF token               │
   │                                   │  CreateWebSession(session)         │
   │                                   │──────────────────────────────────>│
   │  Set-Cookie: MinifluxSessionID    │                                    │
   │    = {id}.{secret}                │                                    │
   │<──────────────────────────────────│                                    │
   │                                   │                                    │
   │  POST /login (带 Cookie)          │                                    │
   │──────────────────────────────────>│                                    │
   │                                   │  loadWebSessionFromCookie()        │
   │                                   │  ├─ 解析 Cookie → id + secret     │
   │                                   │  ├─ WebSessionByID(id)             │
   │                                   │  │───────────────────────────────>│
   │                                   │  ├─ VerifySecret(secret)           │
   │                                   │  │  └─ subtle.ConstantTimeCompare │
   │                                   │  └─ 验证通过                       │
   │                                   │                                    │
   │                                   │  authenticateWebSession()          │
   │                                   │  ├─ session.SetUser(user)          │
   │                                   │  ├─ session.Rotate()               │
   │                                   │  │  ├─ 生成新 ID                   │
   │                                   │  │  ├─ 生成新 Secret               │
   │                                   │  │  └─ 返回 (oldID, newSecret)    │
   │                                   │  ├─ RotateWebSession(oldID, sess)  │
   │                                   │  │  │  UPDATE web_sessions         │
   │                                   │  │  │  SET id=新, secret_hash=新   │
   │                                   │  │  │  WHERE id=旧                │
   │                                   │  │  │─────────────────────────────>│
   │                                   │  └─ setSessionCookie(新)           │
   │  Set-Cookie: MinifluxSessionID    │                                    │
   │    = {新id}.{新secret}            │                                    │
   │<──────────────────────────────────│                                    │
   │                                   │                                    │
   │  后续请求 (带新 Cookie)           │                                    │
   │──────────────────────────────────>│                                    │
   │                                   │  loadWebSessionFromCookie()        │
   │                                   │  └─ 用新 id 查询 → 验证通过       │
   │                                   │                                    │
   │  ... N 天后 ...                   │                                    │
   │                                   │                                    │
   │  [定时调度器]                     │  CleanOldWebSessions()              │
   │                                   │  │  DELETE FROM web_sessions       │
   │                                   │  │  WHERE created_at < now()-N天   │
   │                                   │  │─────────────────────────────────>│
   │                                   │                                    │
   │  请求 (Session 已被清理)          │                                    │
   │──────────────────────────────────>│                                    │
   │                                   │  WebSessionByID(id) → nil          │
   │                                   │  NewWebSession() → 创建新 session  │
   │  Set-Cookie: 新 session           │                                    │
   │  302 → /login                     │                                    │
   │<──────────────────────────────────│                                    │
```

---

## 七、深度拆解：refresh_token 过期后的降级路径

### 7.1 代码事实：Miniflux 中不存在 refresh_token 的使用

在整个代码库中搜索 `refresh_token`/`refreshToken`/`RefreshToken`，结果仅有 3 处命中：

| 文件 | 内容 |
|---|---|
| `internal/integration/wallabag/wallabag.go:148` | `tokenResponse` 结构体中声明了 `RefreshToken string` 字段 |
| `internal/integration/wallabag/wallabag_test.go:58` | 测试用例 mock 响应中包含 `"refresh_token": "token"` |
| `integration-token-rotation.md` | 本文档自身 |

**Wallabag 是唯一触及 refresh_token 的集成，但它仅解析、未使用。** 具体代码路径：

```go
// wallabag.go:103-142 — getAccessToken() 完整实现
func (c *Client) getAccessToken() (string, error) {
    values := url.Values{}
    values.Add("grant_type", "password")           // ← 硬编码为 password grant
    values.Add("client_id", c.clientID)
    values.Add("client_secret", c.clientSecret)
    values.Add("username", c.username)
    values.Add("password", c.password)
    // ...发送请求到 /oauth/v2/token...
    var responseBody tokenResponse
    json.NewDecoder(response.Body).Decode(&responseBody)
    return responseBody.AccessToken, nil           // ← 只返回 access_token
}
```

**不存在降级路径的原因：** `getAccessToken()` 的认证策略是 **"每次用密码重取"** 而非 "先用缓存 token → 401 则用 refresh_token → 再 401 则用密码"。这是一种单层扁平策略——无论 token 处于什么状态（有效、过期、被撤销），都直接走 `grant_type=password`。

### 7.2 假设性推演：如果 refresh_token 被使用，过期后的降级应怎样？

按照 OAuth2 标准流程（RFC 6749 §6），一个完整的降级路径应为：

```
请求业务 API (access_token)
  ├── 200 OK → 完成
  ├── 401 Unauthorized → access_token 可能过期
  │   ├── 尝试 refresh_token grant
  │   │   ├── POST /oauth/v2/token grant_type=refresh_token
  │   │   ├── 200 OK → 获得新 access_token + refresh_token → 重试业务请求
  │   │   └── 400/401 → refresh_token 也过期/被撤销
  │   │       └── 降级到 grant_type=password（如果可用）
  │   │           ├── 200 OK → 获得全新 token 对 → 重试业务请求
  │   │           └── 401 → 密码也已变更 → 标记集成失效 → 通知用户
  │   └── 其他错误 → 网络问题 → 可重试
  └── 其他错误 → 服务端问题 → 可重试
```

**Miniflux 的实际代码直接跳到了降级链的中间**——它绕过了 access_token 缓存和 refresh_token 两个环节，每次都执行 `grant_type=password`。这等效于降级链的"最终回退"操作变成了常规操作。

### 7.3 Wallabag 的 password grant 失败后的完整路径

```
CreateEntry() → getAccessToken()
  │
  ├── POST /oauth/v2/token (grant_type=password) 失败
  │   ├── HTTP 层面错误 (DNS/连接超时)
  │   │   └── fmt.Errorf("wallabag: unable to send request: %v", err)
  │   │       → 返回到 CreateEntry() → 返回到 SendEntry()
  │   │       → slog.Error("Unable to send entry to Wallabag")
  │   │       → 流程结束，无重试
  │   │
  │   └── HTTP 成功但状态码 >= 400
  │       └── fmt.Errorf("wallabag: unable to get access token: url=%s status=%d")
  │           → 同上，直接返回错误
  │
  └── POST /oauth/v2/token 成功 → 获得 access_token
      → createEntry(accessToken, ...)
        ├── HTTP 层面错误 → 同上
        ├── 状态码 >= 401 → access_token 无效（理论上不应发生，因为是刚获取的）
        │   └── fmt.Errorf("wallabag: unable to save entry: url=%s status=%d")
        │       → 返回错误，**不会回到 getAccessToken() 重试**
        └── 200 OK → 成功
```

**关键缺陷：** 当 `createEntry()` 返回 401 时（比如 token 在获取和使用之间被撤销，或者 Wallabag 服务端时钟偏移导致 token 立即过期），代码不会重新获取 token 后重试，而是直接失败。这是因为 `getAccessToken()` 和 `createEntry()` 是两个串行的独立步骤，中间没有重试循环。

### 7.4 其他"类型 B"集成的等效分析

| 集成 | 认证方式 | refresh_token 概念 | 认证失败后的回退 |
|---|---|---|---|
| Wallabag | OAuth2 password grant | 服务端返回但客户端忽略 | 无回退，直接报错 |
| Shiori | POST /api/v1/auth/login | 服务端返回 token+session 但不适用 | 无回退，直接报错 |
| Matrix | POST /_matrix/client/v3/login | 不适用（Matrix 使用 access_token） | 无回退，直接报错 |

三者都是**单层认证策略**——认证失败即流程终止，不存在分层降级。

---

## 八、深度拆解：多个第三方 Provider 串行调用时的失败隔离机制

### 8.1 Pocket 集成的历史与现状

Pocket 集成**已从代码库中移除**。数据库迁移记录了它的生命周期：

```go
// migrations.go:252-258 — 添加 Pocket 列（早期版本）
func(tx *sql.Tx) (err error) {
    sql := `
        ALTER TABLE integrations
            ADD COLUMN pocket_enabled bool default 'f',
            ADD COLUMN pocket_access_token text default '',
            ADD COLUMN pocket_consumer_key text default '';
    `
    // ...
}

// migrations.go:1142-1150 — 删除 Pocket 列
func(tx *sql.Tx) (err error) {
    sql := `
        ALTER TABLE integrations
            DROP COLUMN pocket_enabled,
            DROP COLUMN pocket_access_token,
            DROP COLUMN pocket_consumer_key;
    `
    // ...
}
```

Pocket 曾使用 OAuth2 流程获取 `pocket_access_token`（通过 `pocket_consumer_key`），但这个集成已经被完全移除。当前的 `model.Integration` 结构体和 `integration.go` 的 `SendEntry()`/`PushEntries()` 中均不包含任何 Pocket 相关代码。

### 8.2 当前代码中的串行调用拓扑

`SendEntry()` 和 `PushEntries()` 的调用拓扑是完全串行的、if-guard 隔离的：

```
SendEntry(entry, userIntegrations)
  │
  ├── if BetulaEnabled     → betula.CreateBookmark()      → if err: slog.Error → 继续
  ├── if PinboardEnabled   → pinboard.CreateBookmark()    → if err: slog.Error → 继续
  ├── if InstapaperEnabled → instapaper.AddURL()          → if err: slog.Error → 继续
  ├── if WallabagEnabled   → wallabag.CreateEntry()       → if err: slog.Error → 继续
  ├── if NotionEnabled     → notion.UpdateDocument()      → if err: slog.Error → 继续
  ├── ... (共 22 个集成) ...
  └── if RaindropEnabled   → raindrop.CreateRaindrop()   → if err: slog.Error → 继续

PushEntries(feed, entries, userIntegrations)
  │
  ├── if MatrixBotEnabled  → matrixbot.PushEntries()      → if err: slog.Error → 继续
  ├── if WebhookEnabled    → webhook.SendNewEntries...()  → if err: slog.Warn  → 继续
  ├── if NtfyEnabled       → ntfy.SendMessages()          → if err: slog.Warn  → 继续
  ├── if AppriseEnabled    → apprise.SendNotification()   → if err: slog.Warn  → 继续
  ├── if DiscordEnabled    → discord.SendDiscordMsg()     → if err: slog.Warn  → 继续
  ├── if SlackEnabled      → slack.SendSlackMsg()         → if err: slog.Warn  → 继续
  ├── if PushoverEnabled   → pushover.SendMessages()      → if err: slog.Warn  → 继续
  ├── if TelegramBotEnabled → for each entry:             → if err: slog.Error → 继续
  │     telegrambot.PushEntry()
  └── if ReadeckPushEnabled → for each entry:             → if err: slog.Error → 继续
        readeck.CreateBookmark()
```

### 8.3 失败隔离机制的代码层分析

**隔离靠的是 if-guard + 独立 client 实例，不是 try-catch 或 circuit breaker。**

以 Telegram 和其他集成为例，逐行分析隔离实现：

```go
// integration.go:666-692 — Telegram 推送
if userIntegrations.TelegramBotEnabled {           // ← 隔离点1: 仅在启用时进入
    for _, entry := range entries {
        slog.Debug(...)
        if err := telegrambot.PushEntry(           // ← 每个条目独立调用
            feed, entry,
            userIntegrations.TelegramBotToken,     // ← 参数来自结构体，无共享可变状态
            userIntegrations.TelegramBotChatID,
            // ...
        ); err != nil {
            slog.Error("Unable to send entry to Telegram", ...)
            // ← 错误仅被日志记录
            // ← 没有 break/continue/return
            // ← 循环继续处理下一个 entry
        }
    }
}
// ← 无论 Telegram 是否失败，下面的 Readeck 代码都会执行
if userIntegrations.ReadeckPushEnabled { ... }
```

**隔离机制的关键特征：**

1. **错误不跨集成传播**：每个 `if XxxEnabled` 块中的错误处理都是独立的 `slog.Error`/`slog.Warn`，**不会影响后续集成块的执行**。这是因为 Go 中 `if err != nil { slog.Error(...) }` 之后没有 `return` 或 `panic`，控制流自然落入下一个 `if` 块。

2. **无共享可变状态**：每个集成创建独立的 client 实例（如 `telegrambot.PushEntry()` 内部 `NewClient()`），不存在共享的 token 缓存或连接池。一个集成的 panic 不会污染另一个集成的状态。

3. **goroutine 级别的隔离**：调用方以 `go integration.SendEntry(...)` 和 `go integration.PushEntries(...)` 启动 goroutine：

   ```go
   // entry_save.go:36
   go integration.SendEntry(entry, userIntegrations)

   // handler.go:338
   go integration.PushEntries(originalFeed, newEntries, userIntegrations)
   ```

   这意味着即使某个集成的 HTTP 请求永久阻塞（10 秒超时），也**不会阻塞主请求的处理**。但 goroutine 内部的串行集成调用之间**没有超时隔离**——如果 Wallabag 请求超时（10s），后续所有启用的集成都会延迟 10 秒。

4. **Telegram 内部循环的隔离**：Telegram 在 `PushEntries()` 中按 entry 逐条发送，一条发送失败不会中断其余条目的发送。但错误级别是 `slog.Error`（非 Warn），说明设计者认为单条推送失败值得记录。

### 8.4 缺失的隔离机制

| 机制 | 是否存在 | 影响 |
|---|---|---|
| 集成间短路（一个失败跳过后续） | ❌ 不存在 | 正确——不应该因 Telegram 失败而跳过 Readeck |
| 单集成超时隔离 | ❌ 不存在 | Wallabag 超时 10s 会拖慢后续所有集成 |
| Circuit Breaker（连续失败自动禁用） | ❌ 不存在 | Token 失效后每次都会尝试并失败 |
| 失败计数与自动禁用 | ❌ 不存在 | 不会因为 N 次失败而临时关闭集成 |
| 集成间并发执行 | ❌ 不存在 | 所有启用的集成严格串行执行 |
| 错误聚合与上报 | ❌ 不存在 | 错误仅分散在日志中，不回传给调用者 |

### 8.5 以 Telegram 为例的具体失败场景

Telegram Bot API 使用 `botToken` 直接嵌入 URL：`https://api.telegram.org/bot{token}/sendMessage`

```go
// telegrambot/client.go:74-111
func (c *Client) SendMessage(message *MessageRequest) (*Message, error) {
    endpointURL, _ := url.JoinPath(telegramAPIEndpoint, "/bot"+c.botToken, "/sendMessage")
    // ...发送请求...
    var messageResponse MessageResponse
    json.NewDecoder(response.Body).Decode(&messageResponse)

    if !messageResponse.Ok {
        return nil, fmt.Errorf("telegram: unable to send message: %s (error code is %d)",
            messageResponse.Description, messageResponse.ErrorCode)
    }
    return &messageResponse.Result, nil
}
```

Telegram 的 token 不会过期（bot token 是永久的），但如果 token 被撤销：
1. API 返回 `{"ok": false, "error_code": 401, "description": "Unauthorized"}`
2. `SendMessage()` 返回 error
3. `PushEntry()` 返回 error
4. `PushEntries()` 中 `slog.Error()` 记录日志
5. 继续下一个 entry，每个 entry 都会重复失败并记录一条 Error 日志
6. **不会通知用户、不会自动禁用集成、不会有任何自动恢复操作**

---

## 九、深度拆解：Token 持久化——加密密钥来源与轮换周期

### 9.1 代码事实：Miniflux 不对存储的 Token 做应用层加密

逐层排查整个加密与存储链路：

**1) crypto 包不提供对称加密能力：**

```go
// internal/crypto/crypto.go — 完整内容
func HashFromBytes(value []byte) string      // FNV-1a 非加密哈希
func SHA256(value string) string             // SHA-256 哈希
func GenerateRandomBytes(size int) []byte    // 随机字节
func GenerateRandomStringHex(size int) string // 随机 hex 字符串
func HashPassword(password string) (string, error)  // bcrypt
func GenerateSHA256Hmac(secret string, data []byte) string // HMAC-SHA256
func GenerateUUID() string                   // UUID
func ConstantTimeCmp(a, b string) bool       // 常量时间比较
```

**没有 AES、没有 cipher、没有 encrypt/decrypt 函数。** crypto 包仅提供哈希、随机数生成、HMAC 和 bcrypt——这些都是单向或签名操作，不是对称加密。

**2) 集成 Token 以明文存储在 PostgreSQL：**

`storage/integration.go` 中的 `Integration()` 和 `UpdateIntegration()` 函数直接将 token 字段读写到数据库的 `integrations` 表，**无任何加密/解密层**：

```go
// storage/integration.go:376-636 — UpdateIntegration()
query := `UPDATE integrations SET
    pinboard_token=$2,
    wallabag_client_secret=$15,
    wallabag_password=$17,
    notion_token=$53,
    telegram_bot_token=$26,
    shaarli_api_secret=$71,
    webhook_secret=$74,
    ...
WHERE user_id=$122`
_, err := s.db.Exec(query,
    integration.PinboardToken,       // ← 明文直接传入
    integration.WallabagClientSecret,
    integration.WallabagPassword,
    integration.NotionToken,
    integration.TelegramBotToken,
    integration.ShaarliAPISecret,
    integration.WebhookSecret,
    ...
)
```

所有敏感字段（API Key、Token、Secret、Password）都是 `string` 类型，从 Go 结构体到 SQL 参数**全程明文**，无加密中间层。

**3) model.Integration 中的 Token 字段无加密标记：**

```go
// model/integration.go — 所有 token/secret 字段均为 plain string
type Integration struct {
    PinboardToken      string  // ← 明文
    NotionToken        string  // ← 明文
    TelegramBotToken   string  // ← 明文
    ShaarliAPISecret   string  // ← 明文
    WebhookSecret      string  // ← 明文
    WallabagPassword   string  // ← 明文
    // ...
}
```

没有 `Encrypted` 标签、没有自定义 `Scan()/Value()` 方法、没有 `sql.Scanner` 接口实现来做透明的加解密。

### 9.2 唯一做了哈希处理的秘密：Web Session Secret

```go
// model/web_session.go:70-76
func (s *WebSession) Rotate() (oldID, newSecret string) {
    oldID = s.ID
    newSecret = rand.Text()
    s.ID = rand.Text()
    s.SecretHash = hashWebSessionSecret(newSecret)  // ← SHA-256 哈希
    return oldID, newSecret
}

// model/web_session.go:87-90
func hashWebSessionSecret(secret string) []byte {
    sum := sha256.Sum256([]byte(secret))
    return sum[:]
}
```

Web Session 的 secret 用 SHA-256 哈希存储（`secret_hash` 列），这是**单向哈希**而非加密——不需要解密，只需验证。这与集成 token 的使用场景完全不同：集成 token 需要在每次调用时还原出明文传给第三方 API，所以不能做单向哈希。

### 9.3 Google Reader 密码的存储方式：bcrypt

```go
// storage/integration.go:56-81
func (s *Storage) GoogleReaderUserCheckPassword(username, password string) error {
    var hash string
    s.db.QueryRow(query, username).Scan(&hash)
    if err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)); err != nil {
        return fmt.Errorf(`store: invalid password for "%s" (%v)`, username, err)
    }
    return nil
}
```

Google Reader 密码存储为 **bcrypt 哈希**——同样是单向的，只做验证不做还原。这是因为 Google Reader 认证是"用户提供密码，Miniflux 验证"模式，不需要还原明文。

### 9.4 安全边界：依赖 PostgreSQL 和传输层

Miniflux 的 token 安全策略是**不在应用层做加密，而是依赖底层基础设施**：

| 安全层 | 机制 | 代码位置 |
|---|---|---|
| 传输层加密 | `config.Opts.HTTPS()` → HSTS Header | `http/server/middleware.go:41-43` |
| 数据库连接 | 依赖 `DATABASE_URL` 中的 `sslmode` 参数 | `config/options.go:185-195` |
| Cookie 安全 | `Secure` flag (HTTPS only), `HttpOnly`, `SameSite=Lax` | `ui/auth.go:44-52` |
| Secret 日志脱敏 | `config.ConfigMap(redactSecret=true)` | `config/options.go:1007-1013` |
| 应用层加密 | ❌ **不存在** | — |
| 密钥管理 | ❌ **不存在**（无加密密钥就没有密钥管理） | — |

`config/options.go:1007-1013` 中的日志脱敏仅作用于配置项的打印输出：

```go
func (c *configOptions) ConfigMap(redactSecret bool) []*optionPair {
    for _, value := range c.options {
        if displayValue != "" && redactSecret && value.secret {
            displayValue = "***"    // ← 配置项打印时脱敏
        }
    }
}
```

这只是防止 `ADMIN_PASSWORD`、`OAUTH2_CLIENT_SECRET` 等配置项在日志/调试输出中泄露，**不影响数据库中集成 token 的存储方式**。

### 9.5 假如要实现 Token 加密，需要什么？

当前架构下，集成 token 必须以明文形式使用（调用第三方 API 时需要原始值），所以如果要做应用层加密，需要一个**对称加密方案**。然而代码中不存在任何这样的基础设施：

| 需要的组件 | 当前状态 |
|---|---|
| 对称加密函数 (AES-GCM 等) | ❌ 不存在 |
| 加密密钥 (KEK) | ❌ 不存在 |
| 密钥来源 (env/var/file/KMS) | ❌ 不存在 |
| 密钥轮换机制 | ❌ 不存在 |
| 透明的 SQL Scanner/Valuer | ❌ 不存在 |
| 数据迁移 (明文→密文) | ❌ 不存在 |

Miniflux 选择了**不在应用层加密 token** 的设计决策，把数据安全完全交给 PostgreSQL 的访问控制和传输加密（`sslmode`）。这意味着：
- 有数据库读权限的人可以直接看到所有用户的集成 token
- 没有 key rotation 的概念，因为没有 key
- 如果需要加密，需要从零构建整套对称加密基础设施

## 十、设计决策总结与潜在问题

### 10.1 为什么没有 Token 缓存和刷新？

Miniflux 的第三方集成采用了一种**"无状态"的认证策略**：每次请求都重新获取 token（类型 B）或本地生成 token（类型 C）。这带来了：

**优点：**
- 实现简单，无需维护 token 状态
- 不存在 token 过期导致请求失败的问题
- 无需处理 refresh token 的并发竞态

**缺点：**
- 每次操作多一次 HTTP 往返（Wallabag、Shiori、Matrix）
- 对第三方服务造成不必要的认证负载（特别是 Matrix 的设备注册问题）
- Wallabag 集成忽略了 refresh_token，无法利用 OAuth2 的标准刷新机制

### 10.2 潜在问题

1. **Wallabag 的 `grant_type=password`**：OAuth2 规范中，Resource Owner Password Grant 已被废弃（RFC 6819），且每次都走密码授权而非 refresh_token，既不安全也不高效
2. **Matrix 设备累积**：每次推送都通过 `m.login.password` 登录，Matrix 服务端会为每次登录创建一个新的 device session，长期运行可能产生大量设备
3. **无重试机制**：网络抖动导致的瞬时失败会被直接丢弃，没有指数退避重试
4. **无集成健康状态**：Token 失效不会反馈到 UI，用户无法感知集成是否正常工作
5. **Ntfy 的认证优先级**：当 API Token 和用户名密码同时设置时，两者都会被加到请求头，而非互斥回退

### 10.3 与 Web Session 轮换的对比

Web Session 的轮换是 Miniflux 中唯一实现了"认证后替换标识符"防 session fixation 的机制，但它也缺少：
- 滚动续期（每次活跃使用时延长有效期）
- 非活跃超时（长时间不使用则提前失效）
- 并发 session 的冲突解决

---

## 十一、代码文件索引

| 文件路径 | 作用 |
|---|---|
| `internal/model/web_session.go` | Web Session 模型，Rotate()/VerifySecret() |
| `internal/model/integration.go` | 集成配置模型（存储所有第三方 token） |
| `internal/model/api_key.go` | API Key 模型 |
| `internal/ui/auth.go` | Web Session 认证入口，Cookie 设置 |
| `internal/ui/web_session_middleware.go` | Session 加载/验证/创建中间件 |
| `internal/storage/web_session.go` | Session 持久化、轮换、清理 |
| `internal/ui/oauth2_callback.go` | OAuth2 回调处理 |
| `internal/ui/oauth2_redirect.go` | OAuth2 重定向，PKCE 生成 |
| `internal/oauth2/provider.go` | OAuth2 Provider 接口 |
| `internal/oauth2/authorization.go` | PKCE 授权生成器 |
| `internal/oauth2/oidc.go` | OIDC Provider 实现 |
| `internal/oauth2/google.go` | Google Provider 实现 |
| `internal/oauth2/manager.go` | Provider 管理器 |
| `internal/integration/integration.go` | 集成调度入口 SendEntry()/PushEntries() |
| `internal/integration/wallabag/wallabag.go` | Wallabag OAuth2 客户端（忽略 refresh_token） |
| `internal/integration/shaarli/shaarli.go` | Shaarli JWT 本地签名 |
| `internal/integration/shiori/shiori.go` | Shiori 每次登录换 token |
| `internal/integration/matrixbot/client.go` | Matrix 端点发现、登录、发消息 |
| `internal/integration/matrixbot/matrixbot.go` | Matrix 推送入口 |
| `internal/integration/ntfy/ntfy.go` | Ntfy 双认证模式 |
| `internal/integration/telegrambot/client.go` | Telegram Bot API 客户端，token 嵌入 URL |
| `internal/integration/telegrambot/telegrambot.go` | Telegram 推送入口 |
| `internal/fever/middleware.go` | Fever API Token 认证 |
| `internal/googlereader/middleware.go` | Google Reader HMAC Token 认证 |
| `internal/api/middleware.go` | API Key + Basic Auth 认证 |
| `internal/cli/scheduler.go` | 定时任务调度（session 清理等） |
| `internal/cli/cleanup_tasks.go` | 清理任务执行 |
| `internal/crypto/crypto.go` | 加密工具包（仅哈希/HMAC/bcrypt，无对称加密） |
| `internal/storage/integration.go` | 集成配置的数据库读写（明文存储） |
| `internal/database/migrations.go` | 数据库迁移（含 Pocket 集成的添加与删除） |
| `internal/ui/entry_save.go` | 手动保存条目入口（go SendEntry） |
| `internal/reader/handler/handler.go` | Feed 刷新时推送入口（go PushEntries） |
