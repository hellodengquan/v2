# Miniflux API Key 鉴权链路深度分析

## 一、整体架构概览

Miniflux 是一个 Go 语言编写的 RSS 阅读器，其 API Key 鉴权体系采用了**分层中间件 + 上下文传递 + Handler 内权限检查**的三级防护架构。

### 1.1 模块职责划分

| 层级 | 模块 | 核心职责 | 关键文件 |
|------|------|----------|----------|
| 路由层 | HTTP Server | 请求入口、通用中间件、路由分发 | `internal/http/server/routes.go` |
| 鉴权层 | API Middleware | API Key 解析、用户认证、上下文注入 | `internal/api/middleware.go` |
| 存储层 | Storage | API Key 校验、用户查询、审计更新 | `internal/storage/user.go`, `internal/storage/api_key.go` |
| 业务层 | API Handlers | 权限边界判定、数据隔离、业务逻辑 | `internal/api/*_handlers.go` |
| 响应层 | HTTP Response | 标准化错误响应、兜底页面 | `internal/http/response/json.go`, `internal/http/response/html.go` |

### 1.2 完整执行流程图

```
第三方脚本请求
    ↓
[Server 层] server.middleware()
    ├─ 提取 Client IP 注入上下文
    ├─ HSTS 头处理
    └─ 请求日志记录
    ↓
[路由匹配] newRouter()
    ├─ 匹配 /v1/* 路径 → API Handler
    ├─ 匹配 /fever/* → Fever API
    ├─ 匹配 /reader/api/0/* → Google Reader API
    └─ 其他 → UI Handler (兜底登录页)
    ↓
[API 层中间件链] api.NewHandler()
    ├─ withCORSHeaders() → CORS 跨域处理
    ├─ validateAPIKeyAuth() → API Key 鉴权 ←【核心】
    └─ validateBasicAuth() → HTTP Basic Auth 兜底
    ↓
[业务 Handler]
    ├─ 从 Context 读取 UserID / IsAdmin
    ├─ 权限边界检查 (IsAdminUser / 数据归属)
    ├─ 业务逻辑处理
    └─ 返回 JSON 响应
```

---

## 二、各环节深度解析

### 2.1 环节一：请求识别与路由分发

#### 入口点
`internal/http/server/routes.go:18` - `newRouter()` 函数

```go
func newRouter(store *storage.Storage, pool *worker.Pool) http.Handler {
    rootMux := http.NewServeMux()
    
    // API 路由: /v1/ 前缀
    if config.Opts.HasAPI() {
        appMux.Handle("/v1/", api.NewHandler(store, pool))
    }
    
    // UI 兜底路由 (catch-all)
    appMux.Handle("/", ui.Serve(store, pool))
    
    // ...
}
```

**关键逻辑**：
- API 请求通过 `/v1/` 路径前缀识别
- 当 `config.Opts.HasAPI()` 为 false 时，API 功能完全关闭
- 未匹配到 API 路由的请求会落入 UI 层的 catch-all 处理器

**失败退路**：
- 若 API 功能被禁用，请求直接落入 UI 层，UI 中间件会重定向到登录页

---

### 2.2 环节二：API Key 提取与解析

#### 入口点
`internal/api/middleware.go:37` - `validateAPIKeyAuth()` 函数

```go
func (m *middleware) validateAPIKeyAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        clientIP := request.ClientIP(r)
        token := r.Header.Get("X-Auth-Token")  // 【关键】从 Header 提取

        if token == "" {
            // 未提供 API Key，跳过并交由下一个中间件 (Basic Auth) 处理
            slog.Debug("[API] Skipped API token authentication because no API Key has been provided", ...)
            next.ServeHTTP(w, r)
            return
        }
        
        // ... 后续验证逻辑
    })
}
```

**关键细节**：
- **Token 位置**：仅从 HTTP Header `X-Auth-Token` 提取，不支持 Query Parameter
- **双鉴权机制**：API Key 鉴权失败（未提供）时，不会直接拒绝，而是**穿透**到下一个中间件 `validateBasicAuth()` 尝试 HTTP Basic Auth
- **日志记录**：每次鉴权尝试都会记录 `client_ip`、`user_agent`、`request_uri`

**失败退路**：
- 未提供 API Key → 穿透到 Basic Auth 中间件
- Basic Auth 也失败 → 返回 `401 Unauthorized`

---

### 2.3 环节三：用户匹配与数据库校验

#### 存储层实现
`internal/storage/user.go:482` - `UserByAPIKey()` 函数

```go
func (s *Storage) UserByAPIKey(token string) (*model.User, error) {
    query := `
        SELECT u.id, u.username, u.is_admin, u.timezone, ...
        FROM users u
        INNER JOIN api_keys ON api_keys.user_id = u.id
        WHERE api_keys.token = $1
    `
    return s.fetchUser(query, token)
}
```

**关键设计**：
- 使用 `INNER JOIN` 关联 `users` 和 `api_keys` 表
- 一次性返回完整的用户信息（ID、用户名、管理员权限、时区等）
- Token 字段上应有唯一索引（数据库层面保证）

#### 鉴权判定
回到 `internal/api/middleware.go:52-67`：

```go
user, err := m.store.UserByAPIKey(token)
if err != nil {
    // 数据库查询错误 → 500 Server Error
    response.JSONServerError(w, r, err)
    return
}

if user == nil {
    // Token 无效或已被删除 → 401 Unauthorized
    slog.Warn("[API] No user found with the provided API key", ...)
    response.JSONUnauthorized(w, r)
    return
}
```

**失败退路**：
| 场景 | 响应 | 日志级别 |
|------|------|----------|
| 数据库查询错误 | `500 Internal Server Error` | Error |
| Token 不存在 | `401 Unauthorized` | Warn |
| Token 存在但用户已删除 | `401 Unauthorized` | Warn |

---

### 2.4 环节四：认证成功 - 上下文注入与审计

#### 上下文注入
`internal/api/middleware.go:77-86`

```go
// 审计日志
slog.Info("[API] User authenticated successfully with the API Token Authentication",
    slog.String("username", user.Username),
    ...
)

// 更新登录时间和 API Key 使用时间
m.store.SetLastLogin(user.ID)
m.store.SetAPIKeyUsedTimestamp(user.ID, token)

// 注入用户信息到 Request Context
ctx := r.Context()
ctx = context.WithValue(ctx, request.UserIDContextKey, user.ID)
ctx = context.WithValue(ctx, request.UserTimezoneContextKey, user.Timezone)
ctx = context.WithValue(ctx, request.IsAdminUserContextKey, user.IsAdmin)
ctx = context.WithValue(ctx, request.IsAuthenticatedContextKey, true)

next.ServeHTTP(w, r.WithContext(ctx))
```

**关键数据结构**：
`internal/http/request/context.go:16-25` 定义了上下文键：

```go
const (
    UserIDContextKey ContextKey = iota
    UserTimezoneContextKey
    IsAdminUserContextKey
    IsAuthenticatedContextKey
    // ...
)
```

**审计更新**：
`internal/storage/api_key.go:25` - `SetAPIKeyUsedTimestamp()`

```go
func (s *Storage) SetAPIKeyUsedTimestamp(userID int64, token string) error {
    query := `UPDATE api_keys SET last_used_at=now() WHERE user_id=$1 and token=$2`
    _, err := s.db.Exec(query, userID, token)
    return err
}
```

---

### 2.5 环节五：权限边界判定

权限检查发生在**各个业务 Handler 内部**，而非中间件层。这是因为不同接口有不同的权限要求。

#### 类型一：管理员权限检查
`internal/api/user_handlers.go:28-32`

```go
func (h *handler) createUserHandler(w http.ResponseWriter, r *http.Request) {
    if !request.IsAdminUser(r) {  // 从 Context 读取 IsAdmin
        response.JSONForbidden(w, r)  // 403 Forbidden
        return
    }
    // ... 业务逻辑
}
```

#### 类型二：数据归属检查（水平权限控制）
`internal/api/user_handlers.go:78-82`

```go
func (h *handler) updateUserHandler(w http.ResponseWriter, r *http.Request) {
    userID := request.RouteInt64Param(r, "userID")
    
    if !request.IsAdminUser(r) {
        // 非管理员只能修改自己的信息
        if originalUser.ID != request.UserID(r) {
            response.JSONForbidden(w, r)  // 403 Forbidden
            return
        }
    }
    // ...
}
```

#### 类型三：数据隔离（隐式权限）
几乎所有非管理员接口都会使用 `request.UserID(r)` 进行数据过滤：

```go
func (h *handler) getFeedsHandler(w http.ResponseWriter, r *http.Request) {
    userID := request.UserID(r)  // 从 Context 读取，确保只能看自己的数据
    feeds, err := h.store.Feeds(userID)
    // ...
}
```

**权限边界总结**：

| 权限级别 | 检查方式 | 适用接口 | 失败响应 |
|----------|----------|----------|----------|
| 公开接口 | 无 | 登录、OAuth 回调 | - |
| 已认证 | `request.IsAuthenticated(r)` | 大部分接口 | 401 |
| 个人数据 | `request.UserID(r)` 过滤查询 | 我的订阅、我的文章 | 数据天然隔离 |
| 自我管理 | `userID == request.UserID(r)` | 修改个人信息 | 403 |
| 管理员 | `request.IsAdminUser(r)` | 用户管理、系统配置 | 403 |

---

### 2.6 环节六：错误响应与兜底

#### API 层错误响应
`internal/http/response/json.go` 定义了标准化的 JSON 错误响应：

| 函数 | HTTP 状态码 | 场景 |
|------|------------|------|
| `JSONUnauthorized()` | 401 | Token 无效、未认证 |
| `JSONForbidden()` | 403 | 权限不足 |
| `JSONBadRequest()` | 400 | 参数错误 |
| `JSONNotFound()` | 404 | 资源不存在 |
| `JSONServerError()` | 500 | 服务器内部错误 |

统一响应格式：
```json
{
    "error_message": "access unauthorized"
}
```

#### UI 层兜底（当 API 路由未匹配时）
如果请求路径不匹配 `/v1/*`，会落入 UI 层处理：

`internal/ui/web_session_middleware.go:52-55`

```go
if !request.IsAuthenticated(r) && !isPublicRoute(r) {
    // 未认证且非公开路由 → 重定向到登录页
    response.HTMLRedirect(w, r, loginRedirectURL(m.basePath, r.RequestURI))
    return
}
```

`internal/ui/routes.go:31-48` - `isPublicRoute()` 定义了无需认证的公开路径：
```go
func isPublicRoute(r *http.Request) bool {
    switch path {
    case "/", "/login", "/manifest.json", "/healthcheck", "/offline":
        return true
    }
    // OAuth 回调、分享页面、代理路径也公开
    // ...
}
```

**兜底机制**：
- 第三方脚本误访问非 API 路径 → 302 重定向到登录页
- 登录页会保留 `redirect_url` 参数，登录成功后可跳转回来

---

## 三、完整执行时序（成功场景）

```
第三方脚本
    │
    │ 1. GET /v1/feeds
    │    Header: X-Auth-Token: abc123...
    ▼
server.middleware()
    │ 2. 提取 Client IP，注入 Context
    │ 3. 记录请求开始时间
    ▼
newRouter() 路由匹配
    │ 4. 匹配 /v1/ → api.NewHandler()
    ▼
api.middleware.withCORSHeaders()
    │ 5. 添加 CORS 响应头
    ▼
api.middleware.validateAPIKeyAuth()
    │ 6. 提取 X-Auth-Token: abc123...
    │ 7. 调用 store.UserByAPIKey("abc123...")
    │    └─ DB: INNER JOIN users + api_keys
    │ 8. 匹配到用户 user{ID: 1, IsAdmin: false}
    │ 9. 更新 last_login_at 和 last_used_at
    │ 10. 注入 UserID=1, IsAdmin=false 到 Context
    ▼
api.middleware.validateBasicAuth()
    │ 11. 检测到已认证，直接穿透
    ▼
handler.getFeedsHandler()
    │ 12. userID := request.UserID(r) → 1
    │ 13. 调用 store.Feeds(1) → 仅返回用户1的订阅
    │ 14. 返回 200 OK + JSON 数据
    ▼
server.middleware() defer
    │ 15. 记录请求执行时间到日志
    ▼
响应返回
```

---

## 四、失败场景时序（Token 无效）

```
第三方脚本
    │
    │ 1. GET /v1/feeds
    │    Header: X-Auth-Token: invalid-token
    ▼
... (前序步骤相同) ...
    ▼
api.middleware.validateAPIKeyAuth()
    │ 6. 提取 X-Auth-Token: invalid-token
    │ 7. 调用 store.UserByAPIKey("invalid-token")
    │    └─ DB 查询返回 nil (无匹配)
    │ 8. slog.Warn("No user found with the provided API key")
    │ 9. 调用 response.JSONUnauthorized(w, r)
    │    ├─ 设置 401 状态码
    │    ├─ Content-Type: application/json
    │    └─ Body: {"error_message": "access unauthorized"}
    ▼
响应返回 (401 Unauthorized)
```

---

## 五、关键设计要点与安全考虑

### 5.1 Token 安全
- Token 生成：`crypto.GenerateRandomStringHex(32)` 生成 64 字符十六进制随机字符串
- Token 存储：数据库明文存储（注意：非哈希存储，泄露即失效）
- Token 传输：仅通过 Header 传输，避免 URL 泄漏

### 5.2 审计与可观测性
- 每次认证成功/失败都有结构化日志（slog）
- 记录 `client_ip`、`user_agent`、`username`、`request_uri`
- API Key 每次使用都会更新 `last_used_at` 时间戳

### 5.3 防御性设计
- **中间件顺序**：CORS → API Key Auth → Basic Auth，确保 CORS 头始终存在
- **优雅降级**：API Key 失败不直接拒绝，而是尝试 Basic Auth
- **数据隔离**：所有查询都通过 `request.UserID(r)` 过滤，从源头防止越权
- **上下文不可篡改**：使用 `context.WithValue` 传递，Handler 无法修改

### 5.4 失败退路设计原则
1. **越早失败越好**：中间件层失败直接返回，不进入业务逻辑
2. **错误信息最小化**：不暴露数据库结构、不区分"用户不存在"和"密码错误"
3. **统一错误格式**：所有 API 错误都是 `{"error_message": "..."}`
4. **审计日志完整**：所有失败尝试都记录，便于安全分析

---

## 六、代码引用速查

| 环节 | 文件位置 | 行号 |
|------|----------|------|
| 路由分发 | `internal/http/server/routes.go` | 18-73 |
| API Key 鉴权中间件 | `internal/api/middleware.go` | 37-88 |
| Basic Auth 兜底 | `internal/api/middleware.go` | 90-171 |
| 用户匹配查询 | `internal/storage/user.go` | 482-524 |
| API Key 使用时间更新 | `internal/storage/api_key.go` | 25-33 |
| 上下文键定义 | `internal/http/request/context.go` | 12-25 |
| 权限检查 (Admin) | `internal/api/user_handlers.go` | 29-32 |
| 数据归属检查 | `internal/api/user_handlers.go` | 78-82 |
| JSON 401 响应 | `internal/http/response/json.go` | 98-117 |
| JSON 403 响应 | `internal/http/response/json.go` | 119-138 |
| UI 登录兜底 | `internal/ui/web_session_middleware.go` | 52-55 |
| UI 公开路由定义 | `internal/ui/routes.go` | 31-48 |
