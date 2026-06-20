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

## 五、深度拆解：refresh_token 过期后的降级路径

### 5.1 代码事实：Miniflux 中不存在 refresh_token 的使用

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

### 5.2 假设性推演：如果 refresh_token 被使用，过期后的降级应怎样？

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

### 5.3 Wallabag 的 password grant 失败后的完整路径

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

### 5.4 其他"类型 B"集成的等效分析

| 集成 | 认证方式 | refresh_token 概念 | 认证失败后的回退 |
|---|---|---|---|
| Wallabag | OAuth2 password grant | 服务端返回但客户端忽略 | 无回退，直接报错 |
| Shiori | POST /api/v1/auth/login | 服务端返回 token+session 但不适用 | 无回退，直接报错 |
| Matrix | POST /_matrix/client/v3/login | 不适用（Matrix 使用 access_token） | 无回退，直接报错 |

三者都是**单层认证策略**——认证失败即流程终止，不存在分层降级。

---

## 六、深度拆解：多个第三方 Provider 串行调用时的失败隔离机制

### 6.1 Pocket 集成的历史与现状

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

### 6.2 当前代码中的串行调用拓扑

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

### 6.3 失败隔离机制的代码层分析

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

### 6.4 缺失的隔离机制

| 机制 | 是否存在 | 影响 |
|---|---|---|
| 集成间短路（一个失败跳过后续） | ❌ 不存在 | 正确——不应该因 Telegram 失败而跳过 Readeck |
| 单集成超时隔离 | ❌ 不存在 | Wallabag 超时 10s 会拖慢后续所有集成 |
| Circuit Breaker（连续失败自动禁用） | ❌ 不存在 | Token 失效后每次都会尝试并失败 |
| 失败计数与自动禁用 | ❌ 不存在 | 不会因为 N 次失败而临时关闭集成 |
| 集成间并发执行 | ❌ 不存在 | 所有启用的集成严格串行执行 |
| 错误聚合与上报 | ❌ 不存在 | 错误仅分散在日志中，不回传给调用者 |

### 6.5 以 Telegram 为例的具体失败场景

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

## 七、深度拆解：Token 持久化——加密密钥来源与轮换周期

### 7.1 代码事实：Miniflux 不对存储的 Token 做应用层加密

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

### 7.2 唯一做了哈希处理的秘密：Web Session Secret

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

### 7.3 Google Reader 密码的存储方式：bcrypt

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

### 7.4 安全边界：依赖 PostgreSQL 和传输层

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

### 7.5 假如要实现 Token 加密，需要什么？

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

---

## 八、深度补全：AES-256 加密密钥的轮换周期——代码层面的最终确认

### 8.1 彻底排查：代码库中不存在任何对称加密基础设施

这是上一章结论的强化验证。从代码库的多个维度交叉确认：

**维度一：Go 标准库 crypto/cipher 的使用——零命中**

全库搜索 `cipher.NewGCM`、`aes.NewCipher`、`crypto/cipher`、`crypto/aes` 均返回零结果。没有任何文件 `import` 了这些包。

**维度二：crypto 包的能力边界**

`internal/crypto/crypto.go` 暴露的函数全部是单向操作或随机数生成：

| 函数 | 类型 | 是否可逆 |
|---|---|---|
| `HashFromBytes` | FNV-1a 哈希 | ❌ 单向 |
| `SHA256` | SHA-256 哈希 | ❌ 单向 |
| `HashPassword` | bcrypt | ❌ 单向 |
| `GenerateSHA256Hmac` | HMAC 签名 | ❌ 单向（但可验证） |
| `GenerateRandomBytes` / `GenerateRandomStringHex` | 随机数生成 | N/A |
| `GenerateUUID` | UUID v4 生成 | N/A |
| `ConstantTimeCmp` | 常量时间比较 | N/A |

**维度三：配置项中不存在任何加密密钥相关选项**

`internal/config/options.go` 中所有 `secret: true` 的配置项如下：

| 配置项 | 用途 |
|---|---|
| `ADMIN_PASSWORD` | 初始管理员密码 |
| `DATABASE_URL` | 数据库连接串（可能含密码） |
| `AUTH_PROXY_HEADER_SECRET` | Auth Proxy 的 Header 签名密钥 |
| `OAUTH2_CLIENT_SECRET` | OAuth2 客户端密钥 |
| `HTTPS_CERT_FILE` / `HTTPS_KEY_FILE` | TLS 证书和私钥路径 |
| `WEB_PASSPHRASE` | WebAuthn 相关 |

**没有任何 `ENCRYPTION_KEY`/`DATA_KEY`/`SECRET_KEY` 之类的配置项。** 连配置的基础设施都不存在，更不用说密钥轮换了。

**维度四：数据库存储层的明文证据**

`storage/integration.go:376-636` 的 `UpdateIntegration()` 中，22 个集成的所有 token/secret/password 字段都是直接以 Go `string` 类型传入 `db.Exec()` 的参数，中间没有任何 `encrypt()` / `cipher.Encrypt()` 包装。对应的 `Integration()` 读取函数也对称地直接 `Scan()` 为 `string`。

### 8.2 结论：密钥轮换周期 = 不存在（无密钥则无轮换）

从代码层面可以 100% 确认：

1. **没有加密密钥** — 不存在任何对称加密的主密钥（KEK）或数据密钥（DEK）
2. **没有加密函数** — 没有 AES-256-GCM、AES-CBC 等对称加密的调用
3. **没有密钥来源** — 没有从环境变量、文件、HSM、KMS 读取密钥的逻辑
4. **没有轮换机制** — 没有调度任务、没有 API 端点、没有手动命令来轮换任何加密密钥
5. **没有加密版本号或盐值字段** — 数据库 schema 中不存在 `encryption_version`、`key_id`、`salt` 等字段

**因此"密钥轮换周期"这个问题的前提就不成立**——Miniflux 在应用层不加密任何数据，也就不存在密钥及其轮换。所有敏感 token 以明文形式存储在 PostgreSQL 的 `integrations` 表中，安全边界完全由 PostgreSQL 的访问控制（`pg_hba.conf`、角色权限）和连接加密（`sslmode`）提供。

---

## 九、深度补全：Mastodon / Pleroma（ActivityPub）Provider 的存在性与隔离路径

### 9.1 代码事实：Mastodon、Pleroma、ActivityPub 均不存在

全库搜索 `mastodon`、`pleroma`、`activitypub` 三个关键词，**结果为零命中**（不区分大小写）。

进一步交叉验证：

- `model/integration.go` 的 `Integration` 结构体中没有 Mastodon/Pleroma 相关字段
- `storage/integration.go` 的 SQL 读写中没有 Mastodon/Pleroma 列
- `integration/integration.go` 的 `SendEntry()`/`PushEntries()` 中没有 Mastodon/Pleroma 的调用分支
- `database/migrations.go` 中没有 Mastodon/Pleroma 相关的 DDL 迁移记录
- `internal/integration/` 目录下没有 `mastodon/` 或 `pleroma/` 子目录
- `internal/oauth2/manager.go` 的 `NewManager()` 只接受 `"oidc"` 和 `"google"` 两个 provider

**Mastodon 和 Pleroma 在这个版本的 Miniflux 中完全不存在。**

### 9.2 关于 errgroup 隔离路径的澄清

用户提到"errgroup 隔离路径"——但代码库中**同样不存在 `golang.org/x/sync/errgroup` 或任何并发隔离框架**。全库搜索 `errgroup`、`ErrGroup`、`semaphore` 均返回零结果。

实际的并发模型非常简单：

```go
// entry_save.go:36 — 手动保存
go integration.SendEntry(entry, userIntegrations)

// handler.go:338 — 自动推送
go integration.PushEntries(originalFeed, newEntries, userIntegrations)
```

这是**裸 goroutine**，没有 WaitGroup、没有 errgroup、没有 context 取消、没有 panic recovery（goroutine 内部 panic 会导致整个进程崩溃）。每个 `SendEntry`/`PushEntries` 调用启动一个独立的 goroutine，goroutine 内部的多个 provider 调用仍是**纯串行**：

```
goroutine (SendEntry)
  ├── Betula.CreateBookmark()   — 串行 1
  ├── Pinboard.CreateBookmark() — 串行 2
  ├── Instapaper.AddURL()       — 串行 3
  ├── Wallabag.CreateEntry()    — 串行 4
  └── ... (共 22 个)
```

**失败隔离完全靠 if-guard + 吞掉错误**，不靠任何并发框架。这一点在第六章已有详细分析。

### 9.3 如果未来要接入 Mastodon/Pleroma，会是什么路径？

虽然代码中不存在，但可以基于现有模式推演：

Mastodon/Pleroma 都是 ActivityPub 协议实现，其 API 认证通常使用 **OAuth2 Authorization Code + Bearer Token**。如果接入，大概率会走"类型 A"（静态 Bearer Token，用户在 UI 中填入 `access_token`）或"类型 B"（每次用 client credentials 换取 token）的模式。其隔离路径会和现有 22 个集成完全一致：

```go
// 假设未来的实现
if userIntegrations.MastodonEnabled {
    if err := mastodon.CreatePost(entry.URL, entry.Title); err != nil {
        slog.Error("Unable to send entry to Mastodon", ...)
        // 错误被吞掉，继续下一个集成
    }
}
// 后续 Readeck、Telegram 等不受影响
```

---

## 十、深度补全：refresh_token 过期后登录页跳转在 Provider 禁用时的触发条件

### 10.1 问题拆解：两个独立的概念

需要明确区分：

1. **refresh_token 过期** — 属于第三方 OAuth2 provider（如 Google、OIDC）管理的 token 生命周期
2. **Miniflux 登录页跳转** — 属于 Miniflux 自身 Web Session 失效后的行为

这两者在 Miniflux 中**没有任何关联**，因为 Miniflux **不保存 OAuth2 的 access_token 和 refresh_token**（见第一章第 6 节和第五章）。OAuth2 仅用于"首次身份验证"——用户通过 OAuth2 provider 授权后，Miniflux 获取用户 profile，然后创建/绑定本地用户，之后认证完全依赖 Miniflux 自己的 Web Session。

### 10.2 登录页跳转的真实触发条件

登录页跳转的唯一触发点在 `web_session_middleware.go:52-55`：

```go
if !request.IsAuthenticated(r) && !isPublicRoute(r) {
    response.HTMLRedirect(w, r, loginRedirectURL(m.basePath, r.RequestURI))
    return
}
```

展开这个条件：

```
触发跳转 = (未认证) AND (非公开路由)
```

其中 `request.IsAuthenticated()` 的判断链路是：

```go
// http/request/context.go:48-58
func IsAuthenticated(r *http.Request) bool {
    if getContextBoolValue(r, IsAuthenticatedContextKey) {
        return true
    }
    if session := WebSession(r); session != nil {
        return session.IsAuthenticated()  // session.UserID() 是否非零
    }
    return false
}
```

**登录页跳转与 OAuth2 provider 的状态完全无关。** 只看 Web Session 是否绑定了用户。

### 10.3 Web Session 失效导致跳转的场景

Session 失效有以下几种路径：

| 场景 | 触发点 | 结果 |
|---|---|---|
| **Cookie 不存在** | 首次访问 / 手动清除 Cookie | `loadWebSessionFromCookie()` → nil → 创建新 session → 未认证 → 跳转登录页 |
| **Cookie 格式错误** | Cookie 被篡改，不含 `.` 分隔符 | `strings.Cut()` 失败 → nil → 同上 |
| **Session ID 数据库查不到** | 定时清理任务删除了过期 session | `WebSessionByID()` → nil → 创建新 session → 跳转 |
| **Secret 验证失败** | Secret 被篡改 / 数据库中 SecretHash 不匹配 | `!session.VerifySecret(secret)` → nil → 同上 |
| **Session 未绑定用户** | session 存在但 UserID 为 0（匿名 session） | `session.IsAuthenticated()` → false → 跳转 |

**没有任何一条路径会检查 OAuth2 provider 是否禁用。**

### 10.4 Provider 禁用时的行为分析

**OAuth2 Provider 禁用**指的是将 `OAUTH2_PROVIDER` 环境变量从 `"google"` 或 `"oidc"` 改为空值（或删除该变量）。

禁用后，系统的行为分两层：

**第一层：登录页 UI 的变化**

```html
<!-- templates/views/login.html:47-55 -->
{{ if hasOAuth2Provider "google" }}
    <a class="oauth2-login-button" href="{{ route "oauth2Redirect" "provider" "google" }}">
        Sign in with Google
    </a>
{{ else if hasOAuth2Provider "oidc" }}
    <a class="oauth2-login-button" href="{{ route "oauth2Redirect" "provider" "oidc" }}">
        Sign in with {{ oauth2ProviderName }}
    </a>
{{ end }}
```

`hasOAuth2Provider` 模板函数检查的是 `config.Opts.OAuth2Provider()` 的值。provider 被禁用后，登录页的"Sign in with Google/OIDC"按钮消失。

**第二层：OAuth2 路由的显式拒绝**

即使按钮消失，用户仍可能手动构造 URL 访问 `/oauth2/{provider}/redirect` 和 `/oauth2/{provider}/callback`。这两个路由的处理逻辑是：

```go
// oauth2_redirect.go:23-31
authProvider, err := getOAuth2Manager(r.Context()).FindProvider(provider)
if err != nil {
    slog.Error("Unable to initialize OAuth2 provider", ...)
    response.HTMLRedirect(w, r, h.routePath("/"))   // ← 重定向到首页
    return
}

// oauth2_callback.go:49-57 — 同样的逻辑
authProvider, err := getOAuth2Manager(r.Context()).FindProvider(provider)
if err != nil {
    slog.Error("Unable to initialize OAuth2 provider", ...)
    response.HTMLRedirect(w, r, h.routePath("/"))   // ← 重定向到首页
    return
}
```

关键代码在 `oauth2/manager.go:34-56` 的 `NewManager()`：

```go
func NewManager(ctx context.Context, provider, ...) *Manager {
    m := &Manager{providers: make(map[string]Provider)}
    switch provider {
    case "oidc":
        m.AddProvider("oidc", oidcProvider)
    case "google":
        m.AddProvider("google", NewGoogleProvider(...))
    default:
        slog.Error("Unsupported OAuth2 provider", ...)
        // ← provider 为空或不支持时，providers map 是空的
    }
    return m
}
```

当 `OAUTH2_PROVIDER` 为空时，`providers` map 中没有任何条目，`FindProvider()` 返回 `"oauth2 provider not found"` 错误，然后 `/redirect` 和 `/callback` 都会重定向到首页（`/`）。

**但注意：首页 `/` 是公开路由**（见 `routes.go:39`），不会触发登录页跳转。

### 10.5 综合场景推演：Session 过期 + Provider 已禁用

| 步骤 | 行为 | 代码依据 |
|---|---|---|
| 1. 用户有有效 session，Provider 被禁用 | 不受影响，正常使用 | session 认证不依赖 OAuth2 |
| 2. 用户 session 被清理（过期/手动 flush） | 用户访问 `/unread` → WebSessionByID() → nil → 创建新 session → 未认证且非公开路由 → 302 到 `/`?redirect_url=/unread | `web_session_middleware.go:52-55` |
| 3. 浏览器加载 `/`（登录页） | 展示用户名密码表单，**不展示 OAuth2 登录按钮** | `login.html:47-55` |
| 4. 用户尝试手动访问 `/oauth2/google/redirect` | Manager.FindProvider() → error → 302 到 `/` | `oauth2_redirect.go:23-31` |
| 5. 用户之前 bookmark 了 OAuth2 callback URL，直接访问 `/oauth2/google/callback?code=xxx&state=xxx` | Manager.FindProvider() → error → 302 到 `/` | `oauth2_callback.go:49-57` |
| 6. 如果同时禁用了本地登录（`DISABLE_LOCAL_AUTH=1`） | 登录页只有 WebAuthn 选项；如果用户也没设 WebAuthn，将无法登录 | `login.html` 模板逻辑 |

**关键结论：Provider 禁用不会触发登录页跳转**——触发跳转的是 Session 失效。Provider 禁用只影响登录页上 OAuth2 按钮的显示和 OAuth2 路由的可用性。Session 失效后，用户会被重定向到登录页，然后发现 OAuth2 登录方式不可用，只能用用户名密码或 WebAuthn（如果启用）。

### 10.6 关于 OAuth2 Unlink 的保护

还有一个相关场景：用户已经通过 OAuth2 绑定了账号，现在想在 Settings 页面 Unlink。

```go
// oauth2_unlink.go:17-23
if config.Opts.DisableLocalAuth() {
    slog.Warn("blocking oauth2 unlink attempt, local auth is disabled", ...)
    response.HTMLRedirect(w, r, h.routePath("/"))
    return
}
```

此外还检查用户是否有密码（`hasPassword`），没有密码则不允许 unlink。这防止了用户在唯一的登录方式是 OAuth2 的情况下解绑后无法登录。但这个保护与"登录页跳转"是两个独立的安全机制。

---

## 十一、深度补全：多 Provider 串行场景下已成功写入的回滚清理路径

### 11.1 问题拆解："已成功 Provider 的 token 写入"具体指什么？

在分析回滚路径之前，必须明确：Miniflux 的代码架构中，不存在"多个 Provider 串行 refresh token 并写入数据库"的操作场景。这个问题需要拆分为两个独立场景：

- **场景 A：OAuth2 用户登录/绑定流程** —— 涉及多个数据库写入步骤（创建用户、创建分类、创建 integration 行、更新用户 profile ID、写入 session、更新 last_login），可能有部分成功部分失败
- **场景 B：PushEntries/SendEntry 向多个第三方集成推送** —— 22 个集成串行调用，部分成功部分失败，但**没有任何数据库 token 写入**（集成 token 是预先静态存储的）

两者的回滚机制完全不同。

### 11.2 场景 A：OAuth2 Callback 多步骤写入的事务/回滚分析

**代码路径：** `oauth2_callback.go:19-155` 的 `oauth2Callback()` 函数。

这个流程包含多个独立的数据库写入操作，**逐一检查每个操作失败后的回滚：**

**子场景 A1：新建用户（未登录状态下 OAuth2 首次登录）**

```
oauth2Callback()
  ├── authProvider.Profile()                     // 远程 API 调用（不可回滚）
  ├── h.store.UserByField()                      // 只读查询
  ├── h.store.UserExists()                       // 只读查询
  ├── h.store.CreateUser(userCreationRequest)    // 第一个写操作
  │   └── storage/user.go:58-170
  │       ├── tx = s.db.Begin()                  // ← 开启事务
  │       ├── INSERT INTO users (...)            // 步骤1
  │       │   └── 失败 → tx.Rollback() → return error
  │       ├── INSERT INTO categories (All)       // 步骤2
  │       │   └── 失败 → tx.Rollback() → return error
  │       ├── INSERT INTO integrations (空行)    // 步骤3
  │       │   └── 失败 → tx.Rollback() → return error
  │       └── tx.Commit()                        // 三操作原子提交
  │
  ├── h.store.SetLastLogin(user.ID)              // 第二个写操作（独立 Exec！）
  │   └── UPDATE users SET last_login_at=now()
  │       └── 失败 → 500 HTMLServerError
  │           ↑ **用户已被创建，但 last_login 未更新，无回滚**
  │
  └── authenticateWebSession()                   // 第三个写操作（独立 Exec！）
      ├── session.Rotate()
      ├── store.RotateWebSession(oldID, session) // UPDATE web_sessions
      │   └── 失败 → 500 HTMLServerError
      │       ↑ **用户已创建 + last_login 已更新，但 session 未轮换，无回滚**
      └── setSessionCookie()
```

**关键发现：** `CreateUser()` 内部是有事务的（三个 INSERT 原子执行），但**事务边界只限于 CreateUser 内部**。后续的 `SetLastLogin()` 和 `authenticateWebSession()` 是独立的 `db.Exec` 调用，不在事务内。

如果 `SetLastLogin()` 失败：
- `users` 行已存在（已提交）
- `categories` 行已存在（已提交）
- `integrations` 行已存在（已提交）
- `last_login_at` 仍为 NULL
- **没有自动清理回滚用户**——用户账户"孤儿化"存在于数据库中

如果 `authenticateWebSession()` 失败（`RotateWebSession` 失败）：
- 用户已创建、last_login 已更新
- Web session 中的 `user_id` 尚未绑定（因为 Rotate 中包含 UPDATE ... user_id）
- 用户被重定向到 500 错误页
- **下次重试 OAuth2 登录时，`UserByField()` 能查到已创建的用户，继续走已有用户的路径**

**子场景 A2：绑定已登录用户（Settings 页面链接 OAuth2 账户）**

```
oauth2Callback()
  ├── h.store.UserByID(request.UserID(r))        // 只读
  ├── h.store.AnotherUserWithFieldExists()       // 只读
  ├── authProvider.PopulateUserWithProfileID()   // 修改内存中 user 对象
  └── h.store.UpdateUser(loggedUser)             // 单一 UPDATE
      └── UPDATE users SET google_id=$2, openid_connect_id=$3, ... WHERE id=$1
          └── 失败 → 500 HTMLServerError
              ↑ **只是单条 UPDATE，事务性由数据库保证**
```

绑定场景只有一个写操作，不存在"部分成功部分失败"的多步骤问题。

### 11.3 场景 B：PushEntries/SendEntry 部分成功的回滚——不存在"token 写入撤销"

**核心事实：PushEntries/SendEntry 不写入任何集成 token。**

集成 token 是用户在 Settings 页面提交时通过 `UpdateIntegration()` 一次性写入数据库的。`PushEntries/SendEntry` 只是**读取**这些 token，用于发起 HTTP 请求。

因此，"已成功 Provider 的 token 写入撤销"这个场景在代码中**根本不存在**：

| 概念 | 代码中的事实 |
|---|---|
| 串行 refresh | 22 个集成串行调用，但没有任何一个做 token refresh（都是静态 token 或每次重取） |
| token 写入 | PushEntries/SendEntry 中没有数据库写入 |
| 部分成功 | 只是部分第三方 API 调用成功、部分失败 |
| 回滚清理 | 不需要——没有产生本地状态变更，也就没有需要撤销的写入 |

```go
// integration.go — 典型模式
if userIntegrations.WallabagEnabled {
    if err := wallabag.CreateEntry(...); err != nil {
        slog.Error(...)  // 只记录日志，不写数据库
    }
}
if userIntegrations.NotionEnabled {
    if err := notion.UpdateDocument(...); err != nil {
        slog.Error(...)  // 即使 Wallabag 成功 Notion 失败，也不需要撤销 Wallabag
    }
}
```

**值得注意的边界情况：** 如果 Wallabag 推送成功（条目已保存到 Wallabag 服务器），Notion 推送失败，代码中**不会回滚 Wallabag 端已创建的条目**——因为这需要调用 Wallabag 的 DELETE API，而所有集成客户端都没有实现删除/回滚操作。

### 11.4 storage 层事务使用范围

代码库中使用事务（`db.Begin()` / `tx.Rollback()` / `tx.Commit()`）的地方仅有以下四处：

| 位置 | 事务范围 | 作用 |
|---|---|---|
| `database/database.go:22-48` | 单次迁移 + schema_version 更新 | 数据库迁移的原子执行 |
| `storage/user.go:105-166` | INSERT users + INSERT categories + INSERT integrations | **CreateUser 内部的三行原子创建** |
| `storage/icon.go:101-142` | SELECT/INSERT icons + DELETE/INSERT feed_icons | Feed 图标去重与关联 |
| `storage/entry.go` / `feed.go` / `category.go` / `enclosure.go` | 各有独立事务 | Feed 刷新、条目创建等批量操作 |

**没有任何事务跨越"集成 token 写入"与其他操作。**

### 11.5 小结：回滚清理路径的真实情况

| 场景 | 回滚机制 | 部分成功的后果 |
|---|---|---|
| CreateUser 内部三操作 | **有事务回滚**（tx.Rollback） | 三操作要么全成功要么全失败 |
| OAuth2 callback 中 CreateUser → SetLastLogin → authenticateWebSession | **无跨步骤回滚** | 用户可能被创建但未绑定 session（需重新登录） |
| UpdateUser（绑定 OAuth2） | **单条 UPDATE，数据库自身原子** | 无部分成功问题 |
| PushEntries/SendEntry 多集成调用 | **不涉及数据库写入，无需回滚** | 部分第三方 API 调用永久成功，无回滚 |
| UpdateIntegration（用户保存配置） | **单条 UPDATE，数据库自身原子** | 无部分成功问题 |

---

## 十二、深度补全：ActivityPub Provider 的 Token 失效检测路径与 OAuth Provider 的对比

### 12.1 代码事实：ActivityPub 相关实现完全不存在

全库搜索 `mastodon`、`pleroma`、`activitypub` 三个关键词**零命中**。进一步确认：

- `internal/integration/` 目录下有 20+ 个子目录（wallabag、shaarli、matrixbot、telegrambot 等），**没有 mastodon/pleroma/activitypub 目录**
- `model/integration.go` 的结构体中没有 `MastodonEnabled` / `PleromaToken` / `ActivityPubEndpoint` 等字段
- `storage/integration.go` 的 SQL 中没有 Mastodon/Pleroma 列
- `database/migrations.go` 中没有 Mastodon/Pleroma 相关 DDL
- `oauth2/manager.go` 的 `NewManager()` switch 只处理 `"oidc"` 和 `"google"`

**因此，不存在"ActivityPub provider 的 token 失效检测路径"。** 代码中没有这个 Provider，也就谈不上与 OAuth Provider 的一致性对比。

### 12.2 对比分析：如果接入 ActivityPub，失效检测路径会怎样？

虽然代码中不存在，但可以基于 OAuth Provider（Google/OIDC）的现有失效检测模式来推演对比：

| 失效检测维度 | 当前 OAuth Provider 的模式 | 如果接入 ActivityPub 会走什么模式 |
|---|---|---|
| **Web Session 层面** | `WebSessionByID()` → nil → 视为失效 → 创建新 session → 跳转登录 | **相同**：Session 失效检测与 Provider 类型无关，完全是 Miniflux 内部机制（见第十章） |
| **OAuth2 access_token 层面** | ❌ **不检测**：Miniflux 只在 callback 时用一次 code 换 profile，之后不持有 access_token/refresh_token | **大概率相同**：如果 Mastodon 走用户自行填入 `access_token`（像 Notion 那样的模式），则不做 token 有效性检测，失败时只看 HTTP 状态码 |
| **第三方 API 调用层面** | OAuth2 provider 本身不参与 Miniflux 对第三方服务的推送调用 | 如果 Mastodon 作为**推送目标**（类似 Telegram），则失效检测看 HTTP 401 → slog.Error → 无回退 |
| **OAuth2 callback 失败** | `authProvider.Profile()` 失败 → 重定向到 `/`（首页，公开路由） | **相同**：如果实现 ActivityPub OAuth2，callback 失败也会走同样的重定向路径 |
| **instance domain 不可达** | ❌ **不存在此场景**：Google/OIDC 是中心化服务，不涉及用户自定义的 instance domain | 如果有 Mastodon，instance 不可达会在 `client.Do()` 时返回连接错误 → 记录 slog.Error → 流程终止（无重试） |

### 12.3 instance domain 不可达场景的具体推演

对于支持自定义 instance domain 的 ActivityPub 客户端（如 Mastodon/Pleroma），domain 不可达的典型失败路径如下（基于现有 HTTP 客户端模式）：

```
Mastodon.CreatePost(entry.URL, entry.Title)
  ├── client.NewClientWithOptions({Timeout: 10s, BlockPrivateNetworks: true})
  │   └── http.Client { Timeout: 10 * time.Second }
  ├── http.NewRequest("POST", "https://{mastodon_instance}/api/v1/statuses", body)
  └── httpClient.Do(request)
      ├── DNS 解析失败
      │   └── &net.DNSError → fmt.Errorf("mastodon: unable to send request: %v") → slog.Error
      ├── TCP 连接超时（10s）
      │   └── context.DeadlineExceeded → 同上
      ├── TLS 握手失败
      │   └── tls.RecordHeaderError / x509.UnknownAuthorityError → 同上
      └── HTTP 401 Unauthorized（instance 返回但 token 已失效）
          └── fmt.Errorf("mastodon: unable to send request, status=%d") → slog.Error
```

与现有 OAuth Provider（Google/OIDC）的区别：

- **Google/OIDC**：domain 是硬编码的（`https://accounts.google.com` / OIDC discovery endpoint 配置的），Miniflux 管理员配置；**不可达时是系统级故障**——所有用户都无法使用 OAuth2 登录
- **ActivityPub**：domain 是每个用户单独配置的 instance（`mastodon.social`、`hachyderm.io` 等）；**不可达时是用户级故障**——只影响该用户的推送

但两者的**错误处理模式完全一致**：无论是系统级的 Google 服务不可达，还是用户级的 Mastodon instance 不可达，代码都是：
1. HTTP 客户端返回 error
2. 封装成 `fmt.Errorf("xxx: unable to ...")`
3. 调用方 `slog.Error/Warn`
4. 流程终止，无重试、无标记失效、无用户通知

### 12.4 与现有所有集成的失效检测路径完全一致

总结：虽然 ActivityPub provider 不存在于代码中，但其 token 失效和 instance 不可达的处理模式，如果按照代码库的现有设计风格实现，**会与 OAuth Provider 和所有 22 个集成的失效检测路径保持完全一致**——即仅通过 HTTP 客户端的返回值判断，没有专门的 token 健康检查、没有 instance reachability probe、没有预检测、没有回退重试。

---

## 十三、深度补全：AES 加密密钥来源是否支持 KMS/Vault 等 Secret Manager

### 13.1 代码事实：不存在任何 KMS/Vault/Secrets Manager 集成

从多个维度交叉验证：

**维度一：直接关键词搜索——零有效命中**

| 关键词 | 搜索结果 |
|---|---|
| `kms` / `KMS` | 零命中 |
| `vault` / `Vault` / `hashicorp` | 零命中（排除文档本身） |
| `secretsmanager` / `secret_manager` | 零命中 |
| `aws.*secret` / `aws_ssm` | 零命中（go.sum 中的 transitive dependency 不代表使用） |
| `gcp.*secret` / `azure.*vault` | 零命中 |

**维度二：go.mod / go.sum 中没有 KMS/Vault 相关依赖**

Go module 的直接依赖中不包含：
- `github.com/hashicorp/vault/api`
- `github.com/aws/aws-sdk-go-v2/service/secretsmanager`
- `github.com/aws/aws-sdk-go-v2/service/kms`
- `cloud.google.com/go/secretmanager`
- `github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets`

这些包在 go.sum 中的可能出现只能是 transitive dependency（通过其他库间接引入），**不代表 Miniflux 实际使用了它们**。

**维度三：配置系统没有 Secret Provider 抽象**

`internal/config/` 包的配置解析只支持三种来源：
1. **环境变量**（优先）
2. **命令行 flag**（其次）
3. **配置文件**（最后，通过 `--config` flag 指定）

配置项的解析流程在 `config/parser.go:48-68`：

```go
// 伪代码还原
func NewConfigParser() *configParser {
    cp := &configParser{options: newDefaultConfigOptions()}
    cp.parseFlags()        // 解析命令行参数
    cp.parseConfigFile()   // 读取配置文件
    cp.parseEnvVariables() // 覆盖环境变量
    cp.validate()          // 校验（如 OAUTH2_PROVIDER=oidc 需要 discovery endpoint）
    return cp
}
```

**没有 `lookupSecret` / `fetchFromKMS` / `resolveVaultPath` 这类 hook。** 配置值就是字符串字面量，不做任何二次解析。

### 13.2 当前所有 secret 类配置项的来源方式

以下是代码中所有 `secret: true` 的配置项及其实际存储/读取方式：

| 配置项 | 来源支持 | 加密/解密 |
|---|---|---|
| `ADMIN_PASSWORD` | env / flag / file | 明文 → 首次运行时 bcrypt 哈希存入 DB |
| `DATABASE_URL` | env / flag / file | 明文 → 直接传给 `sql.Open()` |
| `AUTH_PROXY_HEADER_SECRET` | env / flag / file | 明文 → 用于 HMAC 验证 |
| `OAUTH2_CLIENT_SECRET` | env / flag / file | 明文 → 传给 `oauth2.Config{ClientSecret}` |
| `HTTPS_CERT_FILE` | env / flag / file | 文件路径（非 secret 内容） |
| `HTTPS_KEY_FILE` | env / flag / file | 文件路径（非 secret 内容） |
| `WEB_PASSPHRASE` | env / flag / file | 明文 → WebAuthn 相关 |

所有配置项都只支持三种原生来源（env/flag/file），**没有任何 secret:/// arn:aws:kms:/// vault:/// 之类的 URI scheme 解析**。

### 13.3 加密密钥不存在，KMS/Vault 集成也就无意义

回到 AES-256 加密密钥的话题——第九章和第八章已经 100% 确认：
- 代码库中**没有对称加密基础设施**（没有 AES、没有 cipher、没有 encrypt/decrypt 函数）
- 集成 token 以明文形式存储在 PostgreSQL 的 `integrations` 表中
- 不存在任何加密密钥（KEK/DEK）的配置项

因此，"KMS/Vault 作为加密密钥来源"这个问题的**前提不成立**——没有加密密钥，就不存在"密钥从哪里来"的问题，更谈不上"从 KMS/Vault 获取密钥"。

### 13.4 如果未来要支持 KMS/Vault，需要构建的基础设施

假设 Miniflux 未来要实现：
1. 集成 token 应用层加密（AES-256-GCM）
2. 从 KMS/Vault 获取加密密钥

需要实现的组件清单：

| 层级 | 需要的组件 | 当前状态 |
|---|---|---|
| **Secret Provider 接口** | `type SecretProvider interface { Get(key string) ([]byte, error) }` | ❌ 不存在 |
| **Vault Provider** | 实现 `hashicorp/vault/api` 的逻辑逻辑读取 | ❌ 不存在 |
| **AWS KMS Provider** | 实现 `aws-sdk-go-v2/service/kms` 的加密切钥解包 | ❌ 不存在 |
| **GCP Secret Manager** | 实现 `secretmanager` 客户端 | ❌ 不存在 |
| **配置 URI 解析** | 解析 `vault://secret/data/miniflux` 格式的配置值 | ❌ 不存在 |
| **对称加密层** | `Encrypt(plaintext, key)` / `Decrypt(ciphertext, key)`（AES-GCM） | ❌ 不存在 |
| **SQL Transparent Layer** | `sql.Scanner` / `driver.Valuer` 自动加解密 | ❌ 不存在 |
| **密钥版本管理** | `key_id` / `encryption_version` 字段 + 数据迁移 | ❌ 不存在 |
| **密钥轮换调度** | 定时任务重加密旧数据 | ❌ 不存在 |

这些组件在当前代码库中**全部缺失**，需要从零构建。

### 13.5 间接方式：通过部署环境实现 KMS/Vault

虽然 Miniflux 应用层不支持 KMS/Vault，但部署层面有两种"间接"方式：

**方式一：Vault Agent / Consul Template**

通过 sidecar 模式运行 Vault Agent，将 secret 渲染到本地文件，Miniflux 通过 `--config` 读取该文件：
```
Vault → Vault Agent → /etc/miniflux/secrets.conf → Miniflux --config /etc/miniflux/secrets.conf
```
这是部署层面的方案，Miniflux 代码层面完全无感知，仍走原生的"配置文件解析"路径。

**方式二：PostgreSQL pgcrypto + TDE**

在 PostgreSQL 层面启用：
- `pgcrypto` 扩展：`pgp_sym_encrypt()` / `pgp_sym_decrypt()` 列级加密（需要修改 SQL，但代码不需要了解 KMS）
- Transparent Data Encryption（TDE）：文件系统级磁盘加密（PostgreSQL 16+ 或云厂商 RDS 自带）

这些都是数据库层面的方案，Miniflux 应用层同样无感知。

**但这两种方式都属于"绕过"应用层加密的外部方案，不是代码层面的实现。**

---

## 十四、深度补全：deferred provider.revoke 的兜底重试与人工告警机制

### 14.1 代码事实：Miniflux 不实现任何 OAuth2 Token Revoke 功能

全库搜索 `revoke`、`defer.*revoke`、`deferred.*revoke` 关键词**零命中**。这包括：

- `internal/oauth2/` 包（`manager.go`、`provider.go`、`google.go`、`oidc.go`）：没有任何 `RevokeToken()` / `RevokeAccessToken()` / `CallRevokeEndpoint()` 方法
- `internal/ui/oauth2_unlink.go`：用户解绑 OAuth2 账号时，只更新数据库中的 `google_id`/`openid_connect_id` 字段，不调用 provider 的 revoke 端点
- `internal/ui/oauth2_callback.go`：没有 `defer revoke(...)` 来保证 token 在流程失败时被撤销

**OAuth2 登录流程的 token 生命周期：**

```
oauth2Callback()
  ├── state 校验
  ├── oauth2.Config.Exchange(code)  // ← 用 code 换 access_token + (可能有) refresh_token
  ├── authProvider.Profile(accessToken)  // ← 用 access_token 取用户 profile
  ├── 绑定到本地用户（更新 DB）
  └── 认证 Web Session
  
  // ↑ 全程没有 revoke 调用
  // access_token 和 refresh_token 在 Profile() 调用后即被丢弃，不保存在任何地方
```

**OAuth2 Unlink 流程：**

```go
// oauth2_unlink.go:17-28 — 解绑只清空数据库字段
if config.Opts.DisableLocalAuth() {
    slog.Warn("blocking oauth2 unlink attempt, local auth is disabled", ...)
    response.HTMLRedirect(w, r, h.routePath("/"))
    return
}
if !user.HasPassword() {
    slog.Warn("blocking oauth2 unlink attempt, user has no password", ...)
    response.HTMLRedirect(w, r, h.routePath("/settings"))
    return
}
h.store.UnlinkOAuth2Account(user.ID, provider)  // ← UPDATE users SET google_id=''
// ↑ 没有调用 provider.revoke(token)
```

### 14.2 兜底重试机制：完全不存在

不仅没有 revoke 功能，代码库中也不存在任何"兜底重试"框架：

| 重试机制 | 是否存在 | 代码位置 |
|---|---|---|
| HTTP 请求重试（指数退避） | ❌ 不存在 | — |
| Dead Letter Queue（DLQ） | ❌ 不存在 | — |
| Scheduler 重试任务 | ❌ 不存在 | — |
| `golang.org/x/time/rate` 限流 | ❌ 不存在 | — |
| `cenkalti/backoff` / `jpillow/backoff` | ❌ 不存在（go.mod 无依赖） | — |
| `Retry-After` 解析 | ✅ 存在（仅 Feed 刷新使用） | `response_handler.go:82-96` |
| 集成推送的自动重试 | ❌ 不存在 | — |

`Retry-After` 的解析仅用于** Feed 刷新调度**，用于调整 `next_check_at`（下次刷新时间），不是"失败重试"，而是"延迟下次尝试"。这不适用于集成推送。

**唯一近似的重试概念：** `parsing_error_count` 计数器（`model/feed.go:36`）在 Feed 刷新失败时递增，但这只是统计用，不触发任何重试调度。集成推送连这个计数器都没有。

### 14.3 人工告警机制：仅依赖结构化日志

代码库中不存在任何主动告警机制：

| 告警机制 | 是否存在 |
|---|---|
| 内置 Alertmanager/PagerDuty/Opsgenie 集成 | ❌ 不存在 |
| Admin Email/SMS 通知 | ❌ 不存在（代码库甚至没有 SMTP 客户端） |
| Admin Webhook（独立于用户 Webhook） | ❌ 不存在 |
| 错误阈值告警（连续 N 次失败后自动禁用集成） | ❌ 不存在 |
| 集成健康状态 API | ❌ 不存在 |

**唯一的"告警"方式**是 `slog.Error` / `slog.Warn` 输出的结构化日志。运维侧需要通过外部日志收集系统（ELK、Promtail + Loki、Datadog 等）配置告警规则，例如：

```
# 示例 Promtail/Loki 告警规则（伪代码）
- alert: IntegrationPushFailure
  expr: count_over_time({app="miniflux"} |= "Unable to send entry to"[1h]) > 10
  labels:
    severity: warning
  annotations:
    summary: "Integration push failures detected"
    description: "{{ $value }} push failures in last hour"
```

但这完全是运维侧的配置，Miniflux 应用层不提供任何内建支持。

### 14.4 如果未来要实现 deferred revoke + 兜底重试 + 告警，需要什么？

假设 Miniflux 未来要实现 OAuth2 token revoke + 兜底重试 + 人工告警，需要添加的组件：

```go
// 需要新增的接口（概念性设计）
type TokenRevoker interface {
    RevokeToken(ctx context.Context, token string) error
}

type RetryQueue interface {
    Enqueue(task RetryTask) error
    ProcessNext(ctx context.Context) error
    Stats() RetryStats
}

type AlertNotifier interface {
    NotifyAdmin(ctx context.Context, alert Alert) error
}
```

这些组件在当前代码库中全部缺失。

---

## 十五、深度补全：ActivityPub instance domain 错误分类（DNS 失败 vs TCP RST）的失效桶路径

### 15.1 前置说明：代码库中不存在 ActivityPub 集成

如第九章所述，全库搜索 `mastodon`、`pleroma`、`activitypub` 零命中。但代码库中存在完整的**网络错误分类框架**（`response_handler.go:175-275` 的 `LocalizedError()` 和相关辅助函数），可以基于此分析"如果有 ActivityPub 集成"时 DNS 失败 vs TCP RST 的失效路径。

### 15.2 现有错误分类框架分析

`response_handler.go:175-229` 的 `LocalizedError()` 是全库最完整的错误分类器。它的分类逻辑是：

```
LocalizedError(clientErr, httpResponse)
  ├── clientErr != nil （连接层错误）
  │   ├── isSSLError → error.tls_error 桶
  │   ├── isNetworkError → error.network_operation 桶
  │   ├── os.IsTimeout → error.network_timeout 桶
  │   ├── errors.Is(err, io.EOF) → error.http_empty_response 桶
  │   └── 其他 → error.http_client_error 桶
  │
  ├── clientErr == nil （HTTP 响应层错误）
  │   ├── Cloudflare Challenge → error.http_cloudflare_challenge 桶
  │   ├── 401 → error.http_not_authorized 桶
  │   ├── 403 → error.http_forbidden 桶
  │   ├── 429 → error.http_too_many_requests 桶
  │   ├── 404/410 → error.http_resource_not_found 桶
  │   ├── 500 → error.http_internal_server_error 桶
  │   ├── 502 → error.http_bad_gateway 桶
  │   ├── 503 → error.http_service_unavailable 桶
  │   ├── 504 → error.http_gateway_timeout 桶
  │   ├── >= 400 → error.http_unexpected_status_code 桶
  │   └── ContentLength == 0 → error.http_empty_response_body 桶
  │
  └── 无错误 → nil
```

**`isNetworkError` 的判定逻辑**（`response_handler.go:245-259`）：

```go
func isNetworkError(err error) bool {
    if _, ok := errors.AsType[*url.Error](err); ok { return true }
    if errors.Is(err, io.EOF) { return true }
    if _, ok := errors.AsType[*net.OpError](err); ok { return true }
    return false
}
```

**`os.IsTimeout` 的判定逻辑**（Go 标准库）：检查是否实现了 `Timeout() bool` 方法并返回 true。

### 15.3 DNS 解析失败的失效桶路径

DNS 解析失败时，Go net 包返回的错误链是：

```
*url.Error {
    Op: "Get",
    URL: "https://mastodon.example/api/v1/statuses",
    Err: *net.OpError {
        Op: "dial",
        Net: "tcp",
        Source: nil,
        Addr: &net.TCPAddr{IP: nil, Port: 443, Zone: ""},
        Err: *net.DNSError {
            Err:        "no such host",
            Name:       "mastodon.example",
            Server:     "8.8.8.8:53",
            IsTimeout:  false,
            IsTemporary: false,
            IsNotFound: true,
        },
    },
}
```

**分类路径：**
1. `clientErr != nil` → 进入连接层错误分支
2. `isSSLError()` → false（还没到 TLS 握手）
3. `isNetworkError()` → true（`*url.Error` 匹配）
4. **跳过 `os.IsTimeout()`**（因为 `*net.DNSError{IsTimeout: false}`）
5. **落入 `error.network_operation` 桶**
6. 封装为 `LocalizedErrorWrapper{originalErr: err, translationKey: "error.network_operation"}`

### 15.4 TCP RST 的失效桶路径

TCP RST（连接被重置）发生在 TCP 三次握手或数据传输阶段，Go net 包返回的错误链是：

```
*url.Error {
    Op: "Get",
    URL: "https://mastodon.example/api/v1/statuses",
    Err: *net.OpError {
        Op: "read",
        Net: "tcp",
        Source: &net.TCPAddr{IP: 192.168.1.1, Port: 54321, Zone: ""},
        Addr: &net.TCPAddr{IP: 93.184.216.34, Port: 443, Zone: ""},
        Err: syscall.ECONNRESET,  // "connection reset by peer"
    },
}
```

**分类路径：**
1. `clientErr != nil` → 进入连接层错误分支
2. `isSSLError()` → false（ECONNRESET 不是 x509 错误）
3. `isNetworkError()` → true（`*net.OpError` 匹配）
4. `os.IsTimeout()` → false（ECONNRESET 不实现 Timeout() 方法或返回 false）
5. **落入 `error.network_operation` 桶**
6. 封装为 `LocalizedErrorWrapper{originalErr: err, translationKey: "error.network_operation"}`

### 15.5 结论：DNS 失败和 TCP RST 走**同一失效桶**

两者的分类路径完全相同：
- 都匹配 `*url.Error` 或 `*net.OpError` → `isNetworkError()` 返回 true
- 都不是 SSL 错误
- 都不是 timeout 错误
- 都落入 `error.network_operation` 桶

**区分度为零。** 代码库的错误分类框架没有设计"按错误原因细分"的桶——所有网络层错误（DNS 失败、TCP RST、连接被拒绝、主机不可达、TLS 证书过期之外的所有错误）都进入同一个 `error.network_operation` 桶。

### 15.6 与 OAuth Provider 的对比

OAuth2 provider（Google/OIDC）的 token 获取失败路径同样使用这个错误分类框架。例如 Google token endpoint 不可达时：
- 如果是 DNS 失败 → `error.network_operation` 桶
- 如果是 TCP RST → `error.network_operation` 桶
- 如果是 TLS 证书过期 → `error.tls_error` 桶
- 如果是 401 → `error.http_not_authorized` 桶

**因此，ActivityPub 和 OAuth Provider 的错误分类机制是一致的**——如果 ActivityPub 接入，它会复用同样的 `LocalizedError()` 分类器，DNS 失败和 TCP RST 会走与 OAuth Provider 完全相同的桶路径。

---

## 十六、深度补全：KMS Hook 接入的 Reference 实现示例

### 16.1 总体架构设计

基于 Miniflux 现有的 `config/` 包结构，KMS hook 应该设计为**配置值解析器**，在 `parseEnvVariables()` 之后、`validate()` 之前插入。架构图：

```
┌───────────────────────────────────────────────────────────────┐
│                    config.Options                              │
│                                                               │
│  parseFlags() → parseConfigFile() → parseEnvVariables() →     │
│                                              ↓                │
│  [NEW] resolveSecretURIs()  ←  SecretProvider 注册表          │
│              ↓                                                │
│  validate()                                                   │
└───────────────────────────────────────────────────────────────┘
                               ↑
                               │ 注册 Provider
                    ┌──────────┴──────────┐
                    │                     │
              VaultProvider        AWSKMSProvider
                    │                     │
              vault:///...          kms:///...
```

### 16.2 Reference 实现示例

#### 步骤 1：定义 Secret Provider 接口

```go
// internal/config/secret_provider.go — 新增文件
package config

import (
    "context"
    "fmt"
    "strings"
    "sync"
)

// SecretProvider 定义了从外部密钥管理系统获取密钥的接口
type SecretProvider interface {
    // Scheme 返回支持的 URI scheme（如 "vault", "kms", "gcp-sm"）
    Scheme() string
    // Get 从密钥管理系统获取指定路径的密钥值
    Get(ctx context.Context, uri string) ([]byte, error)
    // Close 清理资源（关闭连接等）
    Close(ctx context.Context) error
}

var (
    secretProvidersMu sync.RWMutex
    secretProviders   = make(map[string]SecretProvider)
)

// RegisterSecretProvider 注册一个密钥提供者
func RegisterSecretProvider(provider SecretProvider) {
    secretProvidersMu.Lock()
    defer secretProvidersMu.Unlock()
    secretProviders[provider.Scheme()] = provider
}

// IsSecretURI 判断一个配置值是否是密钥 URI
func IsSecretURI(value string) bool {
    return strings.HasPrefix(value, "vault://") ||
           strings.HasPrefix(value, "kms://") ||
           strings.HasPrefix(value, "gcp-sm://") ||
           strings.HasPrefix(value, "azure-kv://")
}

// resolveSecretURI 解析密钥 URI，返回实际的密钥值
func resolveSecretURI(ctx context.Context, uri string) (string, error) {
    parts := strings.SplitN(uri, "://", 2)
    if len(parts) != 2 {
        return "", fmt.Errorf("config: invalid secret URI format: %s", uri)
    }
    scheme := parts[0]

    secretProvidersMu.RLock()
    provider, ok := secretProviders[scheme]
    secretProvidersMu.RUnlock()

    if !ok {
        return "", fmt.Errorf("config: no secret provider registered for scheme: %s", scheme)
    }

    secret, err := provider.Get(ctx, uri)
    if err != nil {
        return "", fmt.Errorf("config: failed to fetch secret from %s: %w", scheme, err)
    }

    return string(secret), nil
}

// resolveAllSecretURIs 遍历所有配置项，解析其中的密钥 URI
func (c *configParser) resolveAllSecretURIs(ctx context.Context) error {
    for name, opt := range c.options {
        if opt.secret && IsSecretURI(opt.value) {
            resolved, err := resolveSecretURI(ctx, opt.value)
            if err != nil {
                return fmt.Errorf("config: failed to resolve %s: %w", name, err)
            }
            c.options[name] = configOption{
                value:       resolved,
                description: opt.description,
                secret:      opt.secret,
            }
            slog.Debug("Resolved secret URI for config option",
                slog.String("option", name),
                slog.String("scheme", strings.SplitN(opt.value, "://", 2)[0]))
        }
    }
    return nil
}
```

#### 步骤 2：实现 HashiCorp Vault Provider

```go
// internal/config/vault_provider.go — 新增文件
package config

import (
    "context"
    "fmt"
    "os"
    "path/filepath"
    "strings"

    vault "github.com/hashicorp/vault/api"
)

// VaultProvider 实现从 HashiCorp Vault 获取密钥
type VaultProvider struct {
    client *vault.Client
}

// NewVaultProvider 从环境变量创建 Vault 客户端
func NewVaultProvider() (*VaultProvider, error) {
    config := vault.DefaultConfig()
    if addr := os.Getenv("VAULT_ADDR"); addr != "" {
        config.Address = addr
    }

    client, err := vault.NewClient(config)
    if err != nil {
        return nil, fmt.Errorf("vault: failed to create client: %w", err)
    }

    // 支持多种认证方式
    if token := os.Getenv("VAULT_TOKEN"); token != "" {
        client.SetToken(token)
    } else if roleID := os.Getenv("VAULT_ROLE_ID"); roleID != "" {
        // AppRole 认证
        secretID := os.Getenv("VAULT_SECRET_ID")
        resp, err := client.Auth().AppRoleLogin(roleID, secretID)
        if err != nil {
            return nil, fmt.Errorf("vault: approle login failed: %w", err)
        }
        client.SetToken(resp.Auth.ClientToken)
    } else if k8sSA := os.Getenv("VAULT_K8S_SA_PATH"); k8sSA != "" {
        // Kubernetes 认证（简化实现）
        role := os.Getenv("VAULT_K8S_ROLE")
        jwt, _ := os.ReadFile(filepath.Clean(k8sSA))
        resp, err := client.Auth().Kubernetes().Login(role, string(jwt))
        if err != nil {
            return nil, fmt.Errorf("vault: k8s login failed: %w", err)
        }
        client.SetToken(resp.Auth.ClientToken)
    }

    return &VaultProvider{client: client}, nil
}

func (p *VaultProvider) Scheme() string { return "vault" }

func (p *VaultProvider) Get(ctx context.Context, uri string) ([]byte, error) {
    // URI 格式: vault://secret/data/miniflux/encryption_key?field=key
    path := strings.TrimPrefix(uri, "vault://")
    
    // 分离路径和字段查询参数
    var field string
    if idx := strings.Index(path, "?field="); idx != -1 {
        field = path[idx+len("?field="):]
        path = path[:idx]
    }

    secret, err := p.client.Logical().ReadWithContext(ctx, path)
    if err != nil {
        return nil, fmt.Errorf("vault: read failed: %w", err)
    }
    if secret == nil || secret.Data == nil {
        return nil, fmt.Errorf("vault: secret not found at path: %s", path)
    }

    // 从 data 中提取字段（KV v2 返回 {data: {key: value}, metadata: {...}}）
    data, ok := secret.Data["data"].(map[string]interface{})
    if !ok {
        // 尝试 KV v1 格式
        data = secret.Data
    }

    if field == "" {
        // 没有指定字段，尝试常见字段名
        for _, candidate := range []string{"key", "value", "secret", "password", "token"} {
            if val, ok := data[candidate]; ok {
                return []byte(fmt.Sprint(val)), nil
            }
        }
        return nil, fmt.Errorf("vault: no field specified and no default field found")
    }

    val, ok := data[field]
    if !ok {
        return nil, fmt.Errorf("vault: field %q not found in secret", field)
    }

    return []byte(fmt.Sprint(val)), nil
}

func (p *VaultProvider) Close(ctx context.Context) error {
    // Vault HTTP 客户端不需要显式关闭
    return nil
}
```

#### 步骤 3：实现 AWS KMS Provider（用于 envelope encryption）

```go
// internal/config/aws_kms_provider.go — 新增文件
package config

import (
    "context"
    "encoding/base64"
    "fmt"
    "strings"

    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/kms"
    "github.com/aws/aws-sdk-go-v2/service/kms/types"
)

// AWSKMSProvider 实现用 AWS KMS 解密切钥
type AWSKMSProvider struct {
    client *kms.Client
}

func NewAWSKMSProvider(ctx context.Context, region string) (*AWSKMSProvider, error) {
    cfg, err := config.LoadDefaultConfig(ctx, config.WithRegion(region))
    if err != nil {
        return nil, fmt.Errorf("aws: failed to load config: %w", err)
    }
    return &AWSKMSProvider{client: kms.NewFromConfig(cfg)}, nil
}

func (p *AWSKMSProvider) Scheme() string { return "kms" }

func (p *AWSKMSProvider) Get(ctx context.Context, uri string) ([]byte, error) {
    // URI 格式: kms://alias/miniflux-key?ciphertext=AQICAHj...
    // 或者:    kms://arn:aws:kms:us-east-1:123456789:key/abcd-1234?ciphertext=...
    path := strings.TrimPrefix(uri, "kms://")
    
    var ciphertextBlob string
    if idx := strings.Index(path, "?ciphertext="); idx != -1 {
        ciphertextBlob = path[idx+len("?ciphertext="):]
        path = path[:idx]
    }

    if ciphertextBlob == "" {
        return nil, fmt.Errorf("aws-kms: ciphertext parameter required, format: kms://<key-id>?ciphertext=<base64>")
    }

    encrypted, err := base64.StdEncoding.DecodeString(ciphertextBlob)
    if err != nil {
        return nil, fmt.Errorf("aws-kms: invalid base64 ciphertext: %w", err)
    }

    output, err := p.client.Decrypt(ctx, &kms.DecryptInput{
        KeyId:          &path,
        CiphertextBlob: encrypted,
        EncryptionAlgorithm: types.EncryptionAlgorithmSpecSymmetricDefault,
    })
    if err != nil {
        return nil, fmt.Errorf("aws-kms: decrypt failed: %w", err)
    }

    return output.Plaintext, nil
}

func (p *AWSKMSProvider) Close(ctx context.Context) error { return nil }
```

#### 步骤 4：修改 parser.go 集成到现有流程

```go
// internal/config/parser.go — 修改 NewConfigParser()
func NewConfigParser() *configParser {
    cp := &configParser{options: newDefaultConfigOptions()}
    
    cp.parseFlags()
    cp.parseConfigFile()
    cp.parseEnvVariables()
    
    // [NEW] 解析密钥 URI（放在 validate 之前）
    if err := cp.resolveAllSecretURIs(context.Background()); err != nil {
        slog.Error("Failed to resolve secret URIs", slog.Error(err))
        // 注意：这里可以选择继续启动（用默认值/空值）或者直接 os.Exit(1)
    }
    
    cp.validate()
    return cp
}
```

#### 步骤 5：在 main.go 中注册 Provider

```go
// main.go — 在 init() 或 main() 开头注册
func main() {
    // 注册可用的密钥提供者
    if vaultProvider, err := config.NewVaultProvider(); err == nil {
        config.RegisterSecretProvider(vaultProvider)
        slog.Info("Vault secret provider registered")
    }
    if kmsProvider, err := config.NewAWSKMSProvider(
        context.Background(), 
        os.Getenv("AWS_REGION"),
    ); err == nil {
        config.RegisterSecretProvider(kmsProvider)
        slog.Info("AWS KMS secret provider registered")
    }
    
    // ... 原有的启动流程 ...
}
```

### 16.3 运维侧使用示例

#### 示例 1：用 Vault 存储加密密钥

```bash
# 1. 在 Vault 中写入密钥
vault kv put secret/miniflux/encryption key=$(openssl rand -hex 32)

# 2. 配置 Miniflux 使用 Vault 中的密钥
export ENCRYPTION_KEY="vault://secret/data/miniflux/encryption?field=key"
export VAULT_ADDR="https://vault.example.com:8200"
export VAULT_TOKEN="hvs.xxx..."  # 或用 AppRole/K8s auth

# 3. 启动 Miniflux
miniflux
```

#### 示例 2：用 AWS KMS 解密切钥（Envelope Encryption 模式）

```bash
# 1. 生成数据密钥并用 KMS 加密
aws kms generate-data-key \
  --key-id alias/miniflux-key \
  --key-spec AES_256 \
  --query CiphertextBlob \
  --output text > encrypted_dek.b64

# 2. 配置 Miniflux
export ENCRYPTION_KEY="kms://alias/miniflux-key?ciphertext=$(cat encrypted_dek.b64)"
export AWS_REGION="us-east-1"
export AWS_ACCESS_KEY_ID="xxx"
export AWS_SECRET_ACCESS_KEY="xxx"
# 或使用 IAM Role（EC2/EKS 推荐）

# 3. 启动 Miniflux
miniflux
```

### 16.4 配套的加密层示例（用于 token 列加密）

```go
// internal/crypto/encryption.go — 新增文件（扩展现有 crypto 包）
package crypto

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "encoding/base64"
    "fmt"
    "io"
)

// Encrypt 使用 AES-256-GCM 加密明文
func Encrypt(plaintext, key []byte) (string, error) {
    block, err := aes.NewCipher(key)
    if err != nil {
        return "", fmt.Errorf("crypto: invalid key: %w", err)
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", fmt.Errorf("crypto: GCM init failed: %w", err)
    }

    nonce := make([]byte, gcm.NonceSize())
    if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
        return "", fmt.Errorf("crypto: nonce generation failed: %w", err)
    }

    ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

// Decrypt 解密 AES-256-GCM 密文
func Decrypt(ciphertext string, key []byte) ([]byte, error) {
    data, err := base64.StdEncoding.DecodeString(ciphertext)
    if err != nil {
        return nil, fmt.Errorf("crypto: invalid base64: %w", err)
    }

    block, err := aes.NewCipher(key)
    if err != nil {
        return nil, fmt.Errorf("crypto: invalid key: %w", err)
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return nil, fmt.Errorf("crypto: GCM init failed: %w", err)
    }

    nonceSize := gcm.NonceSize()
    if len(data) < nonceSize {
        return nil, fmt.Errorf("crypto: ciphertext too short")
    }

    nonce, ciphertextBytes := data[:nonceSize], data[nonceSize:]
    plaintext, err := gcm.Open(nil, nonce, ciphertextBytes, nil)
    if err != nil {
        return nil, fmt.Errorf("crypto: decryption failed: %w", err)
    }

    return plaintext, nil
}
```

### 16.5 依赖项

```bash
# go.mod 需要添加的新依赖
go get github.com/hashicorp/vault/api
go get github.com/aws/aws-sdk-go-v2/config
go get github.com/aws/aws-sdk-go-v2/service/kms
```

**注意：** 以上是完整的 reference 实现，但如第九章所述，这些代码**当前不存在于 Miniflux 代码库中**，需要运维/开发团队从零实现。

## 十七、设计决策总结与潜在问题

### 17.1 为什么没有 Token 缓存和刷新？

Miniflux 的第三方集成采用了一种**"无状态"的认证策略**：每次请求都重新获取 token（类型 B）或本地生成 token（类型 C）。这带来了：

**优点：**
- 实现简单，无需维护 token 状态
- 不存在 token 过期导致请求失败的问题
- 无需处理 refresh token 的并发竞态

**缺点：**
- 每次操作多一次 HTTP 往返（Wallabag、Shiori、Matrix）
- 对第三方服务造成不必要的认证负载（特别是 Matrix 的设备注册问题）
- Wallabag 集成忽略了 refresh_token，无法利用 OAuth2 的标准刷新机制

### 17.2 潜在问题

1. **Wallabag 的 `grant_type=password`**：OAuth2 规范中，Resource Owner Password Grant 已被废弃（RFC 6819），且每次都走密码授权而非 refresh_token，既不安全也不高效
2. **Matrix 设备累积**：每次推送都通过 `m.login.password` 登录，Matrix 服务端会为每次登录创建一个新的 device session，长期运行可能产生大量设备
3. **无重试机制**：网络抖动导致的瞬时失败会被直接丢弃，没有指数退避重试
4. **无集成健康状态**：Token 失效不会反馈到 UI，用户无法感知集成是否正常工作
5. **Ntfy 的认证优先级**：当 API Token 和用户名密码同时设置时，两者都会被加到请求头，而非互斥回退

### 17.3 与 Web Session 轮换的对比

Web Session 的轮换是 Miniflux 中唯一实现了"认证后替换标识符"防 session fixation 的机制，但它也缺少：
- 滚动续期（每次活跃使用时延长有效期）
- 非活跃超时（长时间不使用则提前失效）
- 并发 session 的冲突解决

---

## 十八、代码文件索引

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
| `internal/ui/routes.go` | 路由定义、公开路由判定、登录跳转 URL 构建 |
| `internal/ui/login_show.go` | 登录页渲染（已认证用户重定向） |
| `internal/ui/oauth2_unlink.go` | OAuth2 账号解绑（含禁用本地登录保护） |
| `internal/http/request/context.go` | IsAuthenticated()、UserID() 等上下文辅助函数 |
| `internal/config/options.go` | 配置项（含所有 secret 标记的配置项列表） |
| `internal/config/parser.go` | 配置解析（含 OAUTH2_PROVIDER 校验） |
| `internal/template/functions.go` | 模板函数 hasOAuth2Provider |
| `internal/template/templates/views/login.html` | 登录页模板（OAuth2 按钮显隐逻辑） |
| `internal/storage/user.go` | 用户存储（CreateUser 内部事务：users+categories+integrations） |
| `internal/storage/icon.go` | Feed 图标存储（事务使用示例） |
| `internal/database/database.go` | 数据库迁移（唯一显式使用 tx.Rollback 的业务层） |
| `internal/config/parser.go` | 配置解析（env/flag/file 三来源解析流程） |
| `internal/reader/fetcher/response_handler.go` | HTTP 响应错误分类器（LocalizedError、isSSLError、isNetworkError、错误桶划分） |
| `internal/locale/error.go` | LocalizedErrorWrapper 定义与实现 |
| `internal/model/feed.go` | Feed 模型（parsing_error_count 计数器、ScheduleNextCheck） |
