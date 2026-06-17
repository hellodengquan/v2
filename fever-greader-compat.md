# Miniflux Fever & Google Reader 接口兼容实现分析

## 1. 代码路径总览

### 核心代码文件

| 模块 | 路径 | 说明 |
|------|------|------|
| Fever Handler | `internal/fever/handler.go` | Fever API 请求分发与业务处理 |
| Fever Middleware | `internal/fever/middleware.go` | Fever API 鉴权中间件 |
| Fever Response | `internal/fever/response.go` | Fever API 响应结构体定义 |
| GReader Handler | `internal/googlereader/handler.go` | Google Reader API 请求处理 |
| GReader Middleware | `internal/googlereader/middleware.go` | Google Reader API 鉴权中间件 |
| GReader Response | `internal/googlereader/response.go` | Google Reader API 响应结构体 |
| GReader Stream | `internal/googlereader/stream.go` | Stream 类型解析与类型枚举 |
| GReader Prefix/Suffix | `internal/googlereader/prefix_suffix.go` | Stream ID 前缀/后缀常量定义 |
| GReader Item | `internal/googlereader/item.go` | Item ID 格式转换 |
| GReader Parameters | `internal/googlereader/parameters.go` | 请求参数名称常量 |
| GReader Request Modifier | `internal/googlereader/request_modifier.go` | 请求参数解析与过滤修饰 |
| HTTP Routes | `internal/http/server/routes.go` | 全局路由注册入口 |
| Request Context | `internal/http/request/context.go` | HTTP 请求上下文键定义与读取 |
| Model Entry | `internal/model/entry.go` | 内部文章（Entry）数据模型 |
| Model Feed | `internal/model/feed.go` | 内部订阅源（Feed）数据模型 |
| Model Integration | `internal/model/integration.go` | 用户集成配置（含 Fever/GReader 凭证） |

### 路由注册入口

在 `internal/http/server/routes.go:18-73` 中注册两个兼容 API：

```go
// Fever API: 所有 /fever/ 路径
feverHandler := fever.Middleware(store)(fever.NewHandler(store))
appMux.Handle("/fever/", feverHandler)

// Google Reader API
googleReaderHandler := googlereader.NewHandler(store)
appMux.HandleFunc("POST /accounts/ClientLogin", googleReaderHandler.ServeHTTP)
appMux.Handle("/reader/api/0/", googleReaderHandler)
```

---

## 2. 请求识别机制

### 2.1 Fever API 请求分发

Fever 使用**单一端点 `/fever/`**，通过查询参数区分操作类型。在 `internal/fever/handler.go:31-54` 中按以下优先级匹配：

| 查询参数 / 表单值 | 处理函数 | 说明 |
|-------------------|----------|------|
| `?groups` | `handleGroups` | 获取分类列表 |
| `?feeds` | `handleFeeds` | 获取订阅源列表 |
| `?favicons` | `handleFavicons` | 获取图标列表 |
| `?unread_item_ids` | `handleUnreadItems` | 获取未读文章 ID 列表 |
| `?saved_item_ids` | `handleSavedItems` | 获取已收藏文章 ID 列表 |
| `?items` | `handleItems` | 获取文章详情（支持分页过滤） |
| `mark=item` | `handleWriteItems` | 标记单篇文章（读/未读/收藏/取消收藏） |
| `mark=feed` | `handleWriteFeeds` | 标记整个订阅源为已读 |
| `mark=group` | `handleWriteGroups` | 标记整个分类为已读 |
| 无匹配 | 默认 | 返回基础认证响应 |

### 2.2 Google Reader API 请求分发

Google Reader 使用**多端点路径**，在 `internal/googlereader/handler.go:48-63` 中通过 `http.NewServeMux()` 注册：

| 方法 | 路径 | 处理函数 | 说明 |
|------|------|----------|------|
| POST | `/accounts/ClientLogin` | `clientLoginHandler` | 用户名密码登录获取 Token |
| GET | `/reader/api/0/token` | `tokenHandler` | 获取编辑 Token |
| POST | `/reader/api/0/edit-tag` | `editTagHandler` | 编辑文章标签（标记读/星标） |
| POST | `/reader/api/0/rename-tag` | `renameTagHandler` | 重命名标签（分类） |
| POST | `/reader/api/0/disable-tag` | `disableTagHandler` | 删除标签（分类） |
| GET | `/reader/api/0/tag/list` | `tagListHandler` | 获取标签列表 |
| GET | `/reader/api/0/user-info` | `userInfoHandler` | 获取当前用户信息 |
| GET | `/reader/api/0/subscription/list` | `subscriptionListHandler` | 获取订阅列表 |
| POST | `/reader/api/0/subscription/edit` | `editSubscriptionHandler` | 编辑/增删订阅 |
| POST | `/reader/api/0/subscription/quickadd` | `quickAddHandler` | 快速添加订阅 |
| GET | `/reader/api/0/stream/items/ids` | `streamItemIDsHandler` | 获取流内文章 ID 列表 |
| POST | `/reader/api/0/stream/items/contents` | `streamItemContentsHandler` | 获取文章详情内容 |
| POST | `/reader/api/0/mark-all-as-read` | `markAllAsReadHandler` | 批量标记已读 |
| GET/POST | `/reader/api/0/` | `fallbackHandler` | 未实现端点兜底返回 `[]` |

---

## 3. 鉴权与会话机制

### 3.1 Fever 鉴权

**代码位置**：`internal/fever/middleware.go:16-73`

#### 鉴权流程

1. **Token 获取**：从表单参数 `api_key` 读取
   ```go
   apiKey := r.FormValue("api_key")
   ```

2. **Token 格式**：MD5 哈希值，由 `FeverUsername:FeverPassword` 计算
   ```
   api_key = md5("fever_username:fever_password")
   ```

3. **用户查询**：调用 `store.UserByFeverToken(apiKey)` 通过 Token 查找用户

4. **鉴权失败响应**：返回 HTTP 200 + JSON `{"api_version":3,"auth":0}`，**不返回 401**

5. **鉴权成功**：将用户信息写入 Context 后转发请求

#### Context 注入字段

```go
ctx = context.WithValue(ctx, request.UserIDContextKey, user.ID)
ctx = context.WithValue(ctx, request.UserTimezoneContextKey, user.Timezone)
ctx = context.WithValue(ctx, request.IsAdminUserContextKey, user.IsAdmin)
ctx = context.WithValue(ctx, request.IsAuthenticatedContextKey, true)
```

#### 凭证存储

在 `internal/model/integration.go:19-24` 中，每个用户的集成配置包含：
- `FeverEnabled bool` - 是否启用
- `FeverUsername string` - Fever 用户名
- `FeverToken string` - 存储的 MD5 令牌（= md5(username:password)）

---

### 3.2 Google Reader 鉴权

**代码位置**：`internal/googlereader/middleware.go:21-186`

#### 两步鉴权流程

**第一步：ClientLogin 登录换取 Token**

路径 `POST /accounts/ClientLogin`，参数：
- `Email` - Google Reader 用户名
- `Passwd` - Google Reader 密码
- `output` - 可选 `json` 指定 JSON 响应

验证流程：
1. 调用 `store.GoogleReaderUserCheckPassword(username, password)` 验证密码（bcrypt 比对）
2. 调用 `store.GoogleReaderUserGetIntegration(username)` 获取集成配置
3. 调用 `getAuthToken()` 生成 Token 返回

Token 生成算法（`middleware.go:182-186`）：
```go
func getAuthToken(username, password string) string {
    token := hex.EncodeToString(hmac.New(sha256.New, []byte(username+password)).Sum(nil))
    token = username + "/" + token
    return token
}
```
Token 格式：`<username>/<hmac_sha256_hex>`

响应格式：
- 默认：纯文本 `SID=xxx\nLSID=xxx\nAuth=xxx\n`
- JSON：`{"SID":"xxx","LSID":"xxx","Auth":"xxx"}`

**第二步：API 请求鉴权**

根据 HTTP 方法不同，Token 传递方式不同：

| 方法 | Token 位置 | 格式 |
|------|-----------|------|
| GET | HTTP Header | `Authorization: GoogleLogin auth=<token>` |
| POST | 表单参数 | `T=<token>` |

GET 请求解析 Header：
```go
authorization := r.Header.Get("Authorization")
// 期望格式: "GoogleLogin auth=readeruser/abcd1234..."
fields := strings.Fields(authorization)  // ["GoogleLogin", "auth=readeruser/abcd1234..."]
auths := strings.Split(fields[1], "=")   // ["auth", "readeruser/abcd1234..."]
token = auths[1]
```

POST 请求解析表单：
```go
token = r.Form.Get("T")
```

Token 验证：
1. 按 `/` 分割为 `[username, hash]` 两部分
2. `store.GoogleReaderUserGetIntegration(username)` 查找集成配置
3. 重新计算 `expectedToken = getAuthToken(integration.GoogleReaderUsername, integration.GoogleReaderPassword)`
4. 使用 `crypto.ConstantTimeCmp(expectedToken, token)` 恒定时间比较防时序攻击

鉴权失败响应：
- HTTP 401
- Header: `X-Reader-Google-Bad-Token: true`
- Body: 纯文本 `Unauthorized`

#### Context 注入字段

```go
ctx = context.WithValue(ctx, request.UserIDContextKey, user.ID)
ctx = context.WithValue(ctx, request.UserNameContextKey, user.Username)
ctx = context.WithValue(ctx, request.UserTimezoneContextKey, user.Timezone)
ctx = context.WithValue(ctx, request.IsAdminUserContextKey, user.IsAdmin)
ctx = context.WithValue(ctx, request.IsAuthenticatedContextKey, true)
ctx = context.WithValue(ctx, request.GoogleReaderTokenKey, token)
```

相比 Fever，多注入了 `UserNameContextKey` 和 `GoogleReaderTokenKey`。

#### 凭证存储

在 `internal/model/integration.go:22-24`：
- `GoogleReaderEnabled bool` - 是否启用
- `GoogleReaderUsername string` - Google Reader 用户名（全局唯一）
- `GoogleReaderPassword string` - bcrypt 哈希后的密码

---

### 3.3 Context 键定义与访问

**代码位置**：`internal/http/request/context.go:12-25`

```go
const (
    UserIDContextKey          ContextKey = iota  // int64
    UserNameContextKey                            // string
    UserTimezoneContextKey                        // string
    IsAdminUserContextKey                         // bool
    IsAuthenticatedContextKey                     // bool
    WebSessionContextKey                          // *model.WebSession
    ClientIPContextKey                            // string
    GoogleReaderTokenKey                          // string (GReader 专用)
)
```

各 Handler 通过辅助函数从 Request Context 读取：
- `request.UserID(r)` - 获取用户 ID
- `request.UserName(r)` - 获取用户名（GReader 用）
- `request.IsAuthenticated(r)` - 判断是否已认证
- `request.GoogleReaderToken(r)` - 获取 GReader Token

---

## 4. 数据结构映射

### 4.1 Fever 响应结构与内部模型映射

**Fever 响应定义**：`internal/fever/response.go`

#### 基础响应（所有响应共用）

```go
type baseResponse struct {
    Version       int   `json:"api_version"`       // 固定 3
    Authenticated int   `json:"auth"`              // 1=已认证, 0=失败
    LastRefresh   int64 `json:"last_refreshed_on_time"` // 当前 Unix 时间戳
}
```

#### Feed 映射（`response.go:93-101` ↔ `model/feed.go:24-77`）

| Fever 字段 (`feed`) | 类型 | 内部 Feed 字段 | 说明 |
|---------------------|------|----------------|------|
| `id` | int64 | `Feed.ID` | 订阅源 ID |
| `favicon_id` | int64 | `Feed.Icon.IconID` | 图标 ID，无图标时为 0 |
| `title` | string | `Feed.Title` | 标题 |
| `url` | string | `Feed.FeedURL` | Feed URL |
| `site_url` | string | `Feed.SiteURL` | 网站 URL |
| `is_spark` | int | 固定 0 | Miniflux 不支持 Sparks |
| `last_updated_on_time` | int64 | `Feed.CheckedAt.Unix()` | 上次检查时间戳 |

映射代码在 `handler.go:141-155`：
```go
subscription := feed{
    ID:          f.ID,
    Title:       f.Title,
    URL:         f.FeedURL,
    SiteURL:     f.SiteURL,
    IsSpark:     0,
    LastUpdated: f.CheckedAt.Unix(),
}
if f.Icon != nil {
    subscription.FaviconID = f.Icon.IconID
}
```

#### Group（分类）映射

| Fever 字段 (`group`) | 类型 | 内部 Category 字段 |
|----------------------|------|-------------------|
| `id` | int64 | `Category.ID` |
| `title` | string | `Category.Title` |

#### FeedsGroups 映射

| Fever 字段 (`feedsGroups`) | 类型 | 说明 |
|---------------------------|------|------|
| `group_id` | int64 | Category ID |
| `feed_ids` | string | 逗号分隔的 Feed ID 列表 |

由 `buildFeedGroups()` (`handler.go:549-564`) 按 Category 聚合生成。

#### Item（文章）映射（`response.go:103-113` ↔ `model/entry.go:27-47`）

| Fever 字段 (`item`) | 类型 | 内部 Entry 字段 | 转换逻辑 |
|---------------------|------|-----------------|----------|
| `id` | int64 | `Entry.ID` | 直接映射 |
| `feed_id` | int64 | `Entry.FeedID` | 直接映射 |
| `title` | string | `Entry.Title` | 直接映射 |
| `author` | string | `Entry.Author` | 直接映射 |
| `html` | string | `Entry.Content` | 经 `mediaproxy.RewriteDocumentWithAbsoluteProxyURL()` 重写媒体 URL |
| `url` | string | `Entry.URL` | 直接映射 |
| `is_saved` | int | `Entry.Starred` | bool→int：true=1, false=0 |
| `is_read` | int | `Entry.Status` | `EntryStatusRead`=1, 其他=0 |
| `created_on_time` | int64 | `Entry.Date.Unix()` | 发布时间转 Unix 时间戳 |

映射代码在 `handler.go:303-325`：
```go
isRead := 0
if entry.Status == model.EntryStatusRead {
    isRead = 1
}
isSaved := 0
if entry.Starred {
    isSaved = 1
}
result.Items = append(result.Items, item{
    ID:        entry.ID,
    FeedID:    entry.FeedID,
    Title:     entry.Title,
    Author:    entry.Author,
    HTML:      mediaproxy.RewriteDocumentWithAbsoluteProxyURL(entry.Content),
    URL:       entry.URL,
    IsSaved:   isSaved,
    IsRead:    isRead,
    CreatedAt: entry.Date.Unix(),
})
```

---

### 4.2 Google Reader 响应结构与内部模型映射

**GReader 响应定义**：`internal/googlereader/response.go`

#### Stream ID 前缀体系

**代码位置**：`internal/googlereader/prefix_suffix.go`

| 常量 | 值 | 含义 |
|------|----|------|
| `feedPrefix` | `"feed/"` | 订阅源前缀 |
| `streamPrefix` | `"user/-/state/com.google/"` | 通用状态流前缀 |
| `userStreamPrefix` | `"user/%d/state/com.google/"` | 用户特定状态流前缀 |
| `labelPrefix` | `"user/-/label/"` | 通用标签前缀 |
| `userLabelPrefix` | `"user/%d/label/"` | 用户特定标签前缀 |

状态流后缀：

| 后缀常量 | 值 | StreamType |
|----------|----|------------|
| `readStreamSuffix` | `"read"` | `ReadStream` |
| `starredStreamSuffix` | `"starred"` | `StarredStream` |
| `readingListStreamSuffix` | `"reading-list"` | `ReadingListStream` |
| `keptUnreadStreamSuffix` | `"kept-unread"` | `KeptUnreadStream` |

#### Stream 类型枚举

**代码位置**：`internal/googlereader/stream.go:11-34`

```go
const (
    NoStream              StreamType = iota
    ReadStream                       // 已读
    StarredStream                    // 已星标
    ReadingListStream                // 阅读列表（全部）
    KeptUnreadStream                 // 保持未读
    BroadcastStream                  // 广播（未实现）
    BroadcastFriendsStream           // 好友广播（未实现）
    LabelStream                      // 标签/分类
    FeedStream                       // 订阅源
    LikeStream                       // 点赞（未实现）
)
```

Stream 解析由 `getStream()` 函数 (`stream.go:73-107`) 根据前缀匹配完成。

#### Subscription（订阅）映射

| GReader 字段 (`subscriptionResponse`) | 类型 | 内部 Feed/Category 字段 | 说明 |
|---------------------------------------|------|------------------------|------|
| `id` | string | `feedPrefix + strconv.FormatInt(Feed.ID, 10)` | 如 `"feed/42"` |
| `title` | string | `Feed.Title` | 标题 |
| `url` | string | `Feed.FeedURL` | Feed URL |
| `htmlUrl` | string | `Feed.SiteURL` | 网站 URL |
| `iconUrl` | string | 由 `feedIconURL()` 生成 | `/feed-icon/<ExternalIconID>` |
| `categories[].id` | string | `userLabelPrefix + Category.Title` | 如 `"user/1/label/Tech"` |
| `categories[].label` | string | `Category.Title` | 分类名称 |
| `categories[].type` | string | 固定 `"folder"` | 分类类型 |

映射代码在 `handler.go:904-913`。

#### Tag（标签）映射

`tagListHandler` 返回 Starred 流 + 所有分类：

```go
// Starred 固定项
result.Tags = append(result.Tags, subscriptionCategoryResponse{
    ID: fmt.Sprintf(userStreamPrefix, userID) + starredStreamSuffix,
})
// 每个分类
result.Tags = append(result.Tags, subscriptionCategoryResponse{
    ID:    labelPrefix + category.Title,
    Label: category.Title,
    Type:  "folder",
})
```

#### Item ID 格式转换

**代码位置**：`internal/googlereader/item.go:14-57`

输出格式（长格式，用于 `stream/items/contents`）：
```go
// tag:google.com,2005:reader/item/%016x
func convertEntryIDToLongFormItemID(entryID int64) string {
    return fmt.Sprintf(ItemIDFormat, entryID)
}
// 例：entryID=123 → "tag:google.com,2005:reader/item/000000000000007b"
```

输入解析（兼容多种客户端格式）：
| 格式 | 示例 | 解析方式 |
|------|------|----------|
| 长格式 | `tag:google.com,2005:reader/item/00000000148b9369` | Sscanf 十六进制 |
| 短格式 | `tag:google.com,2005:reader/item/2f2` | Sscanf 十六进制 |
| 16位十六进制 | `000000000000048c` | Sscanf 十六进制 |
| 十进制 | `12345` | ParseInt 十进制 |

#### Content Item（文章详情）映射

| GReader 字段 (`contentItem`) | 类型 | 内部 Entry 字段 | 说明 |
|------------------------------|------|-----------------|------|
| `id` | string | `Entry.ID` | 转长格式 Google Reader ID |
| `title` | string | `Entry.Title` | 标题 |
| `author` | string | `Entry.Author` | 作者 |
| `published` | int64 | `Entry.Date.Unix()` | 发布时间戳 |
| `updated` | int64 | `Entry.ChangedAt.Unix()` | 修改时间戳 |
| `timestampUsec` | string | `Entry.Date.UnixMicro()` | 微秒时间戳字符串 |
| `crawlTimeMsec` | string | `Entry.CreatedAt.UnixMilli()` | 爬取毫秒时间戳 |
| `canonical[].href` | string | `Entry.URL` | 规范链接 |
| `alternate[].href` | string | `Entry.URL` | 替代链接 |
| `alternate[].type` | string | 固定 `"text/html"` | 内容类型 |
| `content.content` | string | `Entry.Content` | 经媒体代理重写 |
| `summary.content` | string | `Entry.Content` | 同上（摘要和内容相同） |
| `origin.streamId` | string | `feedPrefix + FeedID` | 所属源 Stream ID |
| `origin.title` | string | `Feed.Title` | 所属源标题 |
| `origin.htmlUrl` | string | `Feed.SiteURL` | 所属源网站 URL |
| `enclosure[]` | array | `Entry.Enclosures` | 附件列表 |
| `categories[]` | []string | 组合生成 | 见下表 |

categories 数组构成逻辑（`handler.go:680-691`）：

| 值来源 | 示例 | 条件 |
|--------|------|------|
| `user/<id>/state/com.google/reading-list` | 固定添加 | 始终存在 |
| `user/<id>/label/<CategoryTitle>` | `"user/1/label/Tech"` | Feed 有分类时 |
| `user/<id>/state/com.google/read` | 已读状态标记 | `Entry.Status == EntryStatusRead` |
| `user/<id>/state/com.google/starred` | 星标状态标记 | `Entry.Starred == true` |

---

### 4.3 写操作状态映射

#### Fever 写操作（`mark=item`）

| Fever 参数 `as` | 内部操作 |
|-----------------|----------|
| `"read"` | `SetEntriesStatus(userID, [id], EntryStatusRead)` |
| `"unread"` | `SetEntriesStatus(userID, [id], EntryStatusUnread)` |
| `"saved"` | `ToggleStarred(userID, id)`（切换，非设置） |
| `"unsaved"` | `ToggleStarred(userID, id)`（同上，切换） |

#### Google Reader 写操作（`edit-tag`）

标签通过 `addTags`/`removeTags` 解析后映射：

| 标签 Stream | add | remove |
|-------------|-----|--------|
| `ReadStream` | 标记已读 | 标记未读 |
| `KeptUnreadStream` | 标记未读 | 标记已读 |
| `StarredStream` | 设置 Starred=true | 设置 Starred=false |

冲突检测：`ReadStream` 和 `KeptUnreadStream` 不能矛盾同时出现。

---

## 5. 分页与过滤参数映射

### Fever Items 过滤

`handler.go:244-287`：

| Fever 参数 | EntryQueryBuilder 调用 |
|-----------|------------------------|
| `since_id` > 0 | `AfterEntryID(since_id)` + 排序 `id ASC` |
| `max_id` == 0 | 排序 `id DESC`（最新条目） |
| `max_id` > 0 | `BeforeEntryID(max_id)` + 排序 `id DESC` |
| `with_ids` (逗号分隔) | `WithEntryIDs(ids...)` |
| 无参数 | 无排序（依赖 SQL 默认顺序） |

固定限制 `WithLimit(50)`。

### Google Reader Stream 过滤

`request_modifier.go:60-90` 解析参数：

| GReader 参数 | 含义 | requestModifiers 字段 |
|-------------|------|----------------------|
| `s` (重复) | 包含流 | `Streams []Stream` |
| `xt` (重复) | 排除流 | `ExcludeTargets []Stream` |
| `it` (重复) | 过滤流 | `FilterTargets []Stream`（已解析但未使用） |
| `n` | 数量上限 | `Count int` |
| `c` | 偏移续传 | `Offset int` |
| `r` | 排序方向 | `SortDirection`: `"o"`→`"asc"`, 其他→`"desc"` |
| `ot` | 开始时间戳 | `StartTime int64` |
| `nt` | 结束时间戳 | `StopTime int64` |

各 Stream 处理器构建不同的 QueryBuilder：
- `ReadingListStream` → 全部条目 + 排除 `ReadStream` 时仅未读
- `StarredStream` → `WithStarred(true)`
- `ReadStream` → `WithStatuses(EntryStatusRead)`
- `FeedStream` → `WithFeedID(feedID)` + 排除 `ReadStream` 时 `WithoutStatus(EntryStatusRead)`

时间过滤：
- `StartTime` → `AfterPublishedDate(time.Unix(StartTime, 0))`
- `StopTime` → `BeforePublishedDate(time.Unix(StopTime, 0))`

Continuation 计算：`offset + len(results) < total ? offset + len(results) : 0`

---

## 6. 架构要点总结

### 两层中间件模式

```
请求到达
  ↓
server 级中间件（日志、CORS 等）
  ↓
┌─────────────────────────────────────────┐
│ Fever: Middleware(store)(Handler)       │  ← 全局一个中间件包裹所有 Fever 请求
│   鉴权 → Context 注入 → 按参数分发       │
└─────────────────────────────────────────┘
  或
┌─────────────────────────────────────────┐
│ GReader: NewHandler 内部 mux            │  ← 内部细粒度路由，部分端点包裹鉴权
│   ClientLogin (无鉴权)                   │
│   /reader/api/0/* (withApiKeyAuth 包裹) │
└─────────────────────────────────────────┘
  ↓
业务 Handler 使用 Context 中的 UserID 查询存储
  ↓
返回映射后的兼容格式响应
```

### 凭证与用户分离设计

Fever 和 GReader 均使用独立于主账号的凭证，存储在 `model.Integration` 中，通过以下方式关联用户：

```
Integration.UserID ──────────→ User.ID
     │
     ├─ FeverEnabled / FeverUsername / FeverToken
     └─ GoogleReaderEnabled / GoogleReaderUsername / GoogleReaderPassword
```

### 响应映射的核心原则

1. **ID 格式转换**：内部 `int64` ID ↔ 外部字符串（Fever 保持数字、GReader 加前缀或十六进制编码）
2. **状态枚举压缩**：内部多状态字符串（`"unread"`/`"read"`）↔ 外部整型布尔（0/1）或 Stream 标签
3. **内容代理重写**：`mediaproxy.RewriteDocumentWithAbsoluteProxyURL()` 统一处理 HTML 中的媒体 URL，保护源站并支持代理策略
4. **嵌套结构展开**：内部关联对象（如 `Entry.Feed`、`Feed.Category`）在响应中展开或转为引用 ID

---

## 7. 异常输入的识别与拒绝路径

### 7.1 Fever 异常输入拒绝

#### 7.1.1 无效 api_key

Fever 中间件（`middleware.go:16-73`）对 `api_key` 的校验分为三层，每一层都返回 **HTTP 200** + `{"api_version":3,"auth":0}`：

| 拒绝阶段 | 条件 | 代码行 | 日志级别 |
|----------|------|--------|----------|
| 1. 空值检查 | `api_key == ""` | `middleware.go:22` | Warn |
| 2. 数据库查询错误 | `store.UserByFeverToken()` 返回 err | `middleware.go:32-42` | Error |
| 3. 无匹配用户 | `user == nil` | `middleware.go:44-52` | Warn |

**关键点**：Fever 不对 `api_key` 做格式校验。无论传入任意字符串（如 `"abc"`、`"123"`），都直接走数据库查询路径。格式错误不会被提前拦截——查询结果返回 `user == nil` 时才拒绝。这意味着无效 Token 会导致一次无意义的数据库查询。

**Fever 协议设计原因**：Fever API 规范要求鉴权失败时仍然返回 HTTP 200，仅通过 JSON 体中 `auth` 字段区分成败（0=失败，1=成功）。客户端通过解析 `auth` 值判断而非 HTTP 状态码。

#### 7.1.2 无效写操作参数

**无效 entry ID**（`mark=item`，`handler.go:407-409`）：

```go
entryID := request.FormInt64Value(r, "id")
if entryID <= 0 {
    return  // 直接 return，不写任何响应
}
```

当 `id` 为空、非数字或 ≤0 时，Handler **静默返回空响应**，不返回任何 JSON 或错误。客户端收到一个空 body 的 HTTP 200 响应。

**Entry 不存在**（`handler.go:420-427`）：

```go
if entry == nil {
    response.JSON(w, r, newBaseResponse())
    return
}
```

返回正常的 `{"api_version":3,"auth":1,...}` 基础响应，仿佛操作成功。这是 Fever 协议的设计——写操作无失败语义。

**无效 feed/group ID**（`mark=feed`、`mark=group`）：

- `mark=feed` 中 `feedID <= 0` 时静默返回（`handler.go:492-494`）
- `mark=group` 中 `groupID < 0` 时静默返回（`handler.go:514-516`）
- `groupID == 0` 表示「标记全部已读」，是合法值

**无效 `as` 值**（`mark=item`，`handler.go:429`）：

`switch r.FormValue("as")` 不匹配任何 case 时，跳过状态修改，直接执行到 `handler.go:472` 返回 `newBaseResponse()`。客户端无法区分操作是否生效。

#### 7.1.3 Fever 异常处理总结

Fever 的设计哲学是**尽量不报错**——所有异常路径要么返回 `auth:0`（鉴权层），要么返回 `auth:1` 的基础响应（业务层），要么返回空响应（静默丢弃）。这与 Fever 原始协议规范一致，客户端需自行通过后续读操作验证变更是否生效。

---

### 7.2 Google Reader 异常输入拒绝

GReader 的异常处理比 Fever 细致得多，不同层级的校验产生不同的 HTTP 状态码和错误格式。

#### 7.2.1 ClientLogin 阶段的异常

| 异常条件 | 代码行 | 响应 |
|----------|--------|------|
| 表单解析失败 | `handler.go:81-90` | HTTP 401 + JSON `{"error_message":"access unauthorized"}` |
| `Email` 或 `Passwd` 为空 | `handler.go:96-104` | HTTP 401 + JSON `{"error_message":"access unauthorized"}` |
| 密码校验失败 | `handler.go:106-116` | HTTP 401 + JSON `{"error_message":"access unauthorized"}` |

所有登录失败统一返回 `response.JSONUnauthorized()`（`json.go:99-117`），HTTP 401 + JSON `{"error_message":"access unauthorized"}`。不泄露具体失败原因。

#### 7.2.2 Authorization 头格式校验（GET 请求）

GReader 中间件对 GET 请求的 `Authorization` 头执行**五层逐步校验**（`middleware.go:62-112`），任何一步失败都调用 `sendUnauthorizedResponse()`：

| 校验步骤 | 条件 | 代码行 | 错误日志描述 |
|---------|------|--------|-------------|
| 1. 缺失 | `authorization == ""` | `middleware.go:64-71` | "No token provided" |
| 2. 结构错误 | `strings.Fields()` 结果 `len != 2` | `middleware.go:73-82` | "Authorization header does not have the expected GoogleLogin format" |
| 3. Scheme 错误 | `fields[0] != "GoogleLogin"` | `middleware.go:83-91` | "Authorization header does not begin with GoogleLogin" |
| 4. auth 字段结构错误 | `strings.Split(fields[1], "=")` 结果 `len != 2` | `middleware.go:92-101` | "Authorization header does not have the expected GoogleLogin format" |
| 5. auth 字段名错误 | `auths[0] != "auth"` | `middleware.go:102-110` | "Authorization header does not have the expected GoogleLogin format" |

每步失败后的响应一致：
- HTTP 401
- Header: `X-Reader-Google-Bad-Token: true`
- Content-Type: `text/plain; charset=utf-8`
- Body: `Unauthorized`

**典型异常输入示例**：
- `Authorization: Bearer xxx` → 第 3 步拒绝（Scheme 不是 `GoogleLogin`）
- `Authorization: GoogleLogin token=xxx` → 第 5 步拒绝（字段名不是 `auth`）
- `Authorization: GoogleLogin` → 第 2 步拒绝（缺少第二部分）
- `Authorization: GoogleLogin auth=xxx yyy` → 第 2 步拒绝（`Fields` 分割后超过 2 部分）

#### 7.2.3 POST 请求 Token 校验

| 校验步骤 | 条件 | 代码行 |
|---------|------|--------|
| 1. 表单解析失败 | `r.ParseForm()` 返回 err | `middleware.go:40-49` |
| 2. T 字段为空 | `token == ""` | `middleware.go:52-60` |

两种失败均调用 `sendUnauthorizedResponse()`。

#### 7.2.4 Token 结构与内容校验

通过前两步后，Token 值本身还要经过三层校验（`middleware.go:114-167`）：

| 校验步骤 | 条件 | 代码行 | 日志描述 |
|---------|------|--------|---------|
| 1. 结构不符 | `strings.Split(token, "/")` 结果 `len != 2` | `middleware.go:114-124` | "Auth token does not have the expected structure username/hash" |
| 2. 用户不存在 | `GoogleReaderUserGetIntegration()` 返回 err | `middleware.go:128-137` | "No user found with the given Google Reader username" |
| 3. 哈希不匹配 | `!crypto.ConstantTimeCmp(expectedToken, token)` | `middleware.go:138-147` | "Token does not match" |

此外还有两层后置校验：

| 校验步骤 | 条件 | 代码行 | 日志描述 |
|---------|------|--------|---------|
| 4. 用户查询失败 | `store.UserByID()` 返回 err | `middleware.go:148-157` | "Unable to fetch user from database"（日志级别 Error） |
| 5. 用户不存在 | `user == nil` | `middleware.go:159-167` | "No user found with the given Google Reader credentials" |

**Token 格式异常示例**：
- `"abc"`（无 `/` 分隔）→ 第 1 步拒绝
- `"a/b/c"`（多个 `/`）→ 第 1 步拒绝（`Split` 结果 `len == 3`）
- `"nonexistent_user/abcd1234..."` → 第 2 步拒绝（数据库查不到该 GReader 用户名）
- `"readeruser/wrong_hash_value"` → 第 3 步拒绝（恒定时间比较失败）

**安全要点**：第 3 步使用 `crypto.ConstantTimeCmp()` 进行恒定时间比较，防止通过响应时间差异推断 Token 片段的正确性（时序攻击）。

#### 7.2.5 Stream ID 格式校验

`getStream()` 函数（`stream.go:73-107`）按前缀逐级匹配，不合法时返回 error：

| 异常输入 | 匹配路径 | 返回的 error |
|---------|---------|-------------|
| `user/-/state/com.google/unknown` | 进入 `streamPrefix` 分支，后缀不匹配任何 case | `"googlereader: unknown stream with id: unknown"` |
| `user/999/state/com.google/fresh` | 进入 `userStreamPrefix` 分支，后缀不匹配 | `"googlereader: unknown stream with id: fresh"` |
| `tag/something` | 不匹配任何前缀 | `"googlereader: unknown stream type: tag/something"` |
| `""` (空字符串) | 单独 case | 返回 `Stream{NoStream, ""}` + `nil` error |

**`getStreams()` 的错误传播**（`stream.go:109-119`）：

```go
for _, streamID := range streamIDs {
    stream, err := getStream(streamID, userID)
    if err != nil {
        return []Stream{}, err  // 任何一个解析失败则整个列表失败
    }
    streams = append(streams, stream)
}
```

只要列表中有一个 Stream ID 不合法，整个请求就被拒绝。这是**快速失败**策略。

**Stream ID 错误在不同端点的传播**：

| 端点 | 调用位置 | 错误响应 |
|------|---------|---------|
| `stream/items/ids` | `parseStreamFilterFromRequest()` → `getStreams(s)` | HTTP 500 + JSON `{"error_message":"..."}` |
| `stream/items/ids` | `parseStreamFilterFromRequest()` → `getStreams(xt)` | HTTP 500 + JSON `{"error_message":"..."}` |
| `stream/items/ids` | `parseStreamFilterFromRequest()` → `getStreams(it)` | HTTP 500 + JSON `{"error_message":"..."}` |
| `stream/items/contents` | 同上 | HTTP 500 + JSON `{"error_message":"..."}` |
| `edit-tag` | `getStreams(r.PostForm["a"])` / `getStreams(r.PostForm["r"])` | HTTP 500 + JSON `{"error_message":"..."}` |
| `subscription/edit` | `getStreams(r.Form["s"])` | HTTP 400 + JSON `{"error_message":"..."}` |
| `mark-all-as-read` | `getStream(r.Form.Get("s"))` | HTTP 400 + JSON `{"error_message":"..."}` |
| `disable-tag` | `getStreams(r.Form["s"])` | HTTP 400 + JSON `{"error_message":"..."}` |
| `rename-tag` | `getStream(r.Form.Get("s"))` / `getStream(r.Form.Get("dest"))` | HTTP 400 + JSON `{"error_message":"..."}` |

**注意**：同一个 `getStream()` 错误在不同端点会产生不同的 HTTP 状态码。在流查询端点（`stream/items/ids`、`stream/items/contents`）中，`parseStreamFilterFromRequest` 返回的 error 被 `JSONServerError`（500）处理；在写操作端点中，同样的 error 被 `JSONBadRequest`（400）处理。这是代码中的不一致之处——Stream ID 格式错误本质上是客户端错误（4xx），但流查询端点返回了 500。

#### 7.2.6 Item ID 格式校验

`parseItemID()`（`item.go:29-57`）按优先级尝试三种解析方式：

| 尝试顺序 | 格式 | 失败条件 |
|---------|------|---------|
| 1 | `tag:google.com,2005:reader/item/<hex>` | Sscanf 失败、扫描数量 != 1、结果 == 0 |
| 2 | 16 字符裸十六进制 | Sscanf 失败或扫描数量 != 1 |
| 3 | 十进制字符串 | ParseInt 失败（含溢出） |

全部失败时返回 error，由调用方 `parseItemIDsFromRequest()`（`item.go:59-75`）包装为 `"googlereader: failed to parse item ID ..."` 向上传播。

**Item ID 错误传播**：

| 端点 | 代码行 | 响应 |
|------|--------|------|
| `edit-tag` | `handler.go:224-228` | HTTP 400 + `JSONBadRequest` |
| `stream/items/contents` | `handler.go:638-642` | HTTP 400 + `JSONBadRequest` |

此外 `parseItemIDsFromRequest` 还有前置检查：当 `r.Form["i"]` 为空时，返回 `"googlereader: no items requested"`。

#### 7.2.7 edit-tag 标签语义冲突校验

`checkAndSimplifyTags()`（`handler.go:1234-1281`）检测以下语义冲突：

| 冲突类型 | 错误 |
|---------|------|
| add `read` + add `kept-unread` | `errSimultaneously`：`"googlereader: read and kept-unread should not be supplied simultaneously"` |
| add `read` + remove `read` | `"googlereader: read should not be supplied for add and remove simultaneously"` |
| add `starred` + remove `starred` | `"googlereader: starred should not be supplied for add and remove simultaneously"` |
| add/remove 不支持的 StreamType（如 `NoStream`） | `"googlereader: unsupported tag type: ..."` |

`BroadcastStream` 和 `LikeStream` 被识别但**静默忽略**（仅 Debug 日志），不触发错误。

#### 7.2.8 subscription/edit Stream 类型校验

`editSubscriptionHandler`（`handler.go:525-602`）对 `ac=edit` 时的标签做了额外类型校验：

```go
if newLabel.Type != LabelStream {
    response.JSONBadRequest(w, r, errors.New("destination must be a label"))
    return
}
```

即 `a` 参数必须是 Label 流（如 `user/1/label/Tech`），传入 `user/1/state/com.google/read` 等状态流会被 400 拒绝。

`disableTagHandler`（`handler.go:759-764`）也有类似校验：

```go
if stream.Type != LabelStream {
    response.JSONBadRequest(w, r, errors.New("googlereader: only labels are supported"))
    return
}
```

#### 7.2.9 FeedStream 中 ID 非数字的处理

`handleFeedStreamHandler`（`handler.go:1119-1124`）在从 `Stream.ID` 解析 feedID 时：

```go
feedID, err := strconv.ParseInt(rm.Streams[0].ID, 10, 64)
if err != nil {
    response.JSONServerError(w, r, err)
    return
}
```

由于 `getStream()` 对 `feed/` 前缀的处理只做 `TrimPrefix`（`stream.go:76`），不验证剩余部分是否为有效数字，所以 `feed/abc` 这种 Stream ID 能通过 `getStream()` 但在 `handleFeedStreamHandler` 中 `ParseInt` 失败，返回 HTTP 500。

`markAllAsReadHandler`（`handler.go:1199-1204`）对 `FeedStream` 有相同问题，但返回 HTTP 400。

#### 7.2.10 output 参数校验

`checkOutputFormat()`（`handler.go:1283-1298`）要求 `output` 参数必须为 `"json"`：

```go
if output != "json" {
    return errors.New("googlereader: only json output is supported")
}
```

适用端点：`tag/list`、`subscription/list`、`stream/items/ids`、`stream/items/contents`。缺失或非 `json` 值均被拒绝为 HTTP 400。

---

### 7.3 过滤目标参数 `it` 解析后未使用的原因

`parseStreamFilterFromRequest()`（`request_modifier.go:81-84`）：

```go
result.FilterTargets, err = getStreams(request.QueryStringParamList(r, paramStreamFilters), userID)
if err != nil {
    return requestModifiers{}, err
}
```

`FilterTargets` 被解析并存储在 `requestModifiers` 中，但后续所有 Stream Handler 中均未读取 `rm.FilterTargets`：

| Handler | 代码行 | 是否使用 `FilterTargets` |
|---------|--------|------------------------|
| `handleReadingListStreamHandler` | `handler.go:1005-1047` | ❌ 只遍历 `ExcludeTargets` |
| `handleStarredStreamHandler` | `handler.go:1049-1071` | ❌ 不使用任何过滤 |
| `handleReadStreamHandler` | `handler.go:1073-1095` | ❌ 不使用任何过滤 |
| `handleFeedStreamHandler` | `handler.go:1119-1153` | ❌ 只遍历 `ExcludeTargets` |

**未使用的原因**：

1. **`it` 参数语义的复杂性**：在 Google Reader 原始协议中，`it`（include targets）用于将结果限定为同时属于指定流的项目。例如 `it=user/1/label/Tech` 表示只返回 Tech 分类的文章。实现此功能需要将 Stream 映射为 SQL JOIN 或子查询条件（如 LabelStream → Category 过滤、FeedStream → FeedID 过滤），而当前 `EntryQueryBuilder` 不直接支持这种多 Stream 交集过滤。

2. **最小可用实现策略**：Miniflux 的 GReader 兼容层采用「先解析、后按需实现」的方式。将 `it` 解析为结构化 `[]Stream` 确保：
   - 参数格式错误能在解析阶段就被捕获（返回 500 错误而非静默忽略）
   - 未来添加过滤逻辑时代码改动最小（只需在 Handler 中遍历 `rm.FilterTargets`）

3. **与 `xt`（排除目标）的不对称**：`xt` 已实现且仅支持 `ReadStream`（排除已读），因为「排除已读」可通过简单的 `WithStatuses(EntryStatusUnread)` 实现，无需复杂 JOIN。而 `it` 的 LabelStream 过滤需要 `Entry.Feed.Category` 关联查询，实现成本更高。

4. **对客户端的兼容影响**：主流 GReader 客户端（Reeder、NetNewsWire 等）很少使用 `it` 参数，缺失此功能不影响基本使用场景。

---

### 7.4 鉴权失败返回 200 与 401 的差异分析

#### Fever 返回 HTTP 200 的设计

```go
// middleware.go:28
response.JSON(w, r, newAuthFailureResponse())
// 等价于：HTTP 200 + {"api_version":3,"auth":0}
```

**协议原因**：

Fever API 规范明确规定鉴权失败通过 JSON 体中的 `auth` 字段表达，而非 HTTP 状态码。这是 Fever 原始服务端的协议设计——所有请求都返回 200，客户端通过解析 `auth` 值判断是否认证成功。

**客户端行为假设**：

Fever 客户端实现通常先发一个无操作的请求（如 `?api_key=xxx`），检查 `auth` 是否为 1 来确认凭证有效。如果收到 HTTP 401，某些简单客户端可能直接抛出网络错误而不解析响应体，导致用户看到的是"网络连接失败"而非"密码错误"。

**安全影响**：

- 正面：不暴露 HTTP 层面的鉴权失败，对简单探测器有一定混淆
- 负面：HTTP 缓存层和 CDN 无法基于状态码区分已认证与未认证响应，可能缓存 `auth:0` 的响应

#### GReader 返回 HTTP 401 的设计

**两个鉴权失败响应路径**：

路径 A — `ClientLogin` 端点（`handler.go:88`）：
```
HTTP 401 + JSON {"error_message":"access unauthorized"}
Content-Type: application/json
```

路径 B — `/reader/api/0/*` 端点（`response.go:122-129`）：
```
HTTP 401 + 纯文本 "Unauthorized"
Content-Type: text/plain; charset=utf-8
X-Reader-Google-Bad-Token: true
```

**协议原因**：

Google Reader 原始 API 使用标准 HTTP 鉴权语义。`ClientLogin` 返回 JSON 格式的 401 是因为该端点本就返回 JSON（或纯文本）格式。而 `/reader/api/0/*` 端点的 401 响应格式遵循 Google Reader 的特定约定：

- `X-Reader-Google-Bad-Token` Header 让客户端可以区分「Token 过期需刷新」与「凭证错误需重新登录」。客户端收到此 Header 后通常会尝试 `GET /reader/api/0/token` 获取新 Token，若仍失败则跳转到登录页。
- 纯文本 Body（而非 JSON）是因为 Google Reader 原始实现就是如此，客户端对此有硬编码匹配。

**两种鉴权失败响应的差异总结**：

| 维度 | Fever（HTTP 200） | GReader ClientLogin（HTTP 401） | GReader API（HTTP 401） |
|------|-------------------|-------------------------------|----------------------|
| HTTP 状态码 | 200 | 401 | 401 |
| Content-Type | application/json | application/json | text/plain |
| Body 格式 | `{"api_version":3,"auth":0}` | `{"error_message":"access unauthorized"}` | `Unauthorized` |
| 客户端检测方式 | 解析 JSON 的 `auth` 字段 | 解析 JSON 的 `error_message` | 检查 HTTP 状态码 + `X-Reader-Google-Bad-Token` |
| 可恢复性 | 不区分原因 | 不区分原因 | `X-Reader-Google-Bad-Token` 允许客户端尝试刷新 Token |
| 缓存友好性 | 差（200 可能被缓存） | 好（401 不被缓存） | 好（401 不被缓存） |
| 错误信息详细度 | 无（仅 `auth:0`） | 低（固定消息） | 低（固定消息 + 特殊 Header） |

**设计哲学对比**：

- **Fever**：鉴权是「查询的一部分」——`auth` 字段和其他数据字段同级，客户端统一解析。这与 Fever 把所有操作合并到单一端点的设计一致。
- **GReader**：鉴权是「请求的前提」——先验失败直接中断请求，使用标准 HTTP 语义拒绝。这与 GReader 多端点、REST 风格的设计一致。
