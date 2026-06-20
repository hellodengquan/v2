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

### 4.3 边界问题

1. **身份类型硬编码**：每增加一种 OAuth2 提供商，就需要在 User 结构体和数据库表中新增字段。
2. **单一外部身份限制**：每个用户只能绑定一个 Google ID 和一个 OIDC ID，无法绑定多个同类型身份。
3. **解绑不彻底**：解绑（`oauth2_unlink.go`）只是清空对应字段，没有审计记录。
4. **会话与 OAuth2 状态耦合**：OAuth2 流程状态存储在 WebSession 的 state JSON 中，与会话生命周期绑定。

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

### 6.4 鉴权边界问题

1. **多种 API 认证系统并存**：REST API Key、Fever Token、Google Reader Token 三套独立的凭证体系，都关联到同一个用户。
2. **上下文键混用**：所有认证方式都写入同一组上下文键（`UserIDContextKey`, `IsAuthenticatedContextKey` 等），但 Google Reader API 额外有 `GoogleReaderTokenKey`。
3. **认证失败处理不一致**：
   - REST API：无 API Key 时放行到下一层（Basic Auth）
   - Fever API：无 API Key 直接返回失败
   - Web UI：未认证重定向到登录页
4. **权限粒度粗**：只有"是否管理员"二元权限，没有针对 API Key 的细粒度权限控制。

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

### 8.2 会话边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 会话既是认证载体又是状态存储 | OAuth2 流程状态、消息、偏好都塞在 session state 里 | `internal/model/web_session.go:37` |
| 无滑动续期 | 会话创建时间不随活动刷新，可能导致用户活跃中被登出 | `internal/storage/web_session.go:183` |
| 认证状态双重来源 | 上下文直存 + 会话对象，两种方式并存 | `internal/http/request/context.go:48` |

### 8.3 API 鉴权边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 三套 API 凭证体系 | REST API Key、Fever Token、Google Reader Token 各自独立 | `internal/storage/api_key.go`, `internal/fever/middleware.go`, `internal/googlereader/middleware.go` |
| 认证中间件行为不一致 | 有的放行、有的直接拒绝 | `internal/api/middleware.go:37` vs `internal/fever/middleware.go:17` |
| Token 与 Session 混用 | REST API v1 理论上也可能带上 Session Cookie | - |

### 8.4 OAuth2 边界模糊

| 问题 | 表现 | 位置 |
|------|------|------|
| 外部身份直接写入用户表 | 没有独立的 user_identities 关联表 | `internal/model/user.go:26-27` |
| OAuth2 状态与会话耦合 | state 和 code_verifier 存在会话 state JSON 中 | `internal/model/web_session.go:49` |
| 自动创建用户权限边界不清 | OAuth2 登录可能绕过本地用户创建策略 | `internal/ui/oauth2_callback.go:120` |

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

| 功能模块 | 文件路径 | 关键行号 |
|----------|----------|----------|
| 用户模型 | `internal/model/user.go` | 13 |
| Web 会话模型 | `internal/model/web_session.go` | 25 |
| API Key 模型 | `internal/model/api_key.go` | 13 |
| OAuth2 重定向 | `internal/ui/oauth2_redirect.go` | 15 |
| OAuth2 回调 | `internal/ui/oauth2_callback.go` | 19 |
| 本地登录 | `internal/ui/login_check.go` | 20 |
| Web 会话认证 | `internal/ui/auth.go` | 22 |
| Web 会话中间件 | `internal/ui/web_session_middleware.go` | 27 |
| REST API 中间件 | `internal/api/middleware.go` | 37, 90 |
| Fever API 中间件 | `internal/fever/middleware.go` | 17 |
| Google Reader 中间件 | `internal/googlereader/middleware.go` | 29 |
| 请求上下文 | `internal/http/request/context.go` | 16 |
| 用户存储 | `internal/storage/user.go` | 58, 481 |
| 会话存储 | `internal/storage/web_session.go` | 16, 97 |
| API Key 存储 | `internal/storage/api_key.go` | 73 |
| OAuth2 提供者接口 | `internal/oauth2/provider.go` | 15 |
