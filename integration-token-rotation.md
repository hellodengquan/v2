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

## 五、设计决策总结与潜在问题

### 5.1 为什么没有 Token 缓存和刷新？

Miniflux 的第三方集成采用了一种**"无状态"的认证策略**：每次请求都重新获取 token（类型 B）或本地生成 token（类型 C）。这带来了：

**优点：**
- 实现简单，无需维护 token 状态
- 不存在 token 过期导致请求失败的问题
- 无需处理 refresh token 的并发竞态

**缺点：**
- 每次操作多一次 HTTP 往返（Wallabag、Shiori、Matrix）
- 对第三方服务造成不必要的认证负载（特别是 Matrix 的设备注册问题）
- Wallabag 集成忽略了 refresh_token，无法利用 OAuth2 的标准刷新机制

### 5.2 潜在问题

1. **Wallabag 的 `grant_type=password`**：OAuth2 规范中，Resource Owner Password Grant 已被废弃（RFC 6819），且每次都走密码授权而非 refresh_token，既不安全也不高效
2. **Matrix 设备累积**：每次推送都通过 `m.login.password` 登录，Matrix 服务端会为每次登录创建一个新的 device session，长期运行可能产生大量设备
3. **无重试机制**：网络抖动导致的瞬时失败会被直接丢弃，没有指数退避重试
4. **无集成健康状态**：Token 失效不会反馈到 UI，用户无法感知集成是否正常工作
5. **Ntfy 的认证优先级**：当 API Token 和用户名密码同时设置时，两者都会被加到请求头，而非互斥回退

### 5.3 与 Web Session 轮换的对比

Web Session 的轮换是 Miniflux 中唯一实现了"认证后替换标识符"防 session fixation 的机制，但它也缺少：
- 滚动续期（每次活跃使用时延长有效期）
- 非活跃超时（长时间不使用则提前失效）
- 并发 session 的冲突解决

---

## 六、代码文件索引

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
| `internal/fever/middleware.go` | Fever API Token 认证 |
| `internal/googlereader/middleware.go` | Google Reader HMAC Token 认证 |
| `internal/api/middleware.go` | API Key + Basic Auth 认证 |
| `internal/cli/scheduler.go` | 定时任务调度（session 清理等） |
| `internal/cli/cleanup_tasks.go` | 清理任务执行 |
