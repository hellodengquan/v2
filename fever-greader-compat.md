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

---

## 8. 存储层错误与媒体重写的错误传播

### 8.1 流查询 SQL 出错的响应传播路径

Fever 和 GReader 的流查询都通过 `EntryQueryBuilder` 访问数据库，SQL 出错时的传播路径高度一致——均返回 HTTP 500 + JSON 错误。

#### 8.1.1 Fever 侧错误传播链

| 端点 | 调用链 | 错误出口 | 代码行 |
|------|--------|----------|--------|
| `?items` | `handleItems` → `builder.GetEntries()` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:289-293` |
| `?items` | `handleItems` → `builder.CountEntries()` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:295-300` |
| `?unread_item_ids` | `handleUnreadItems` → `builder.GetEntryIDs()` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:345-349` |
| `?saved_item_ids` | `handleSavedItems` → `builder.GetEntryIDs()` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:375-379` |
| `?feeds` | `handleFeeds` → `store.Feeds(userID)` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:133-137` |
| `?groups` | `handleGroups` → `store.Categories(userID)` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:185-189` |
| `?favicons` | `handleFavicons` → `store.Favicons(userID)` 失败 | `response.JSONServerError(w, r, err)` | `handler.go:225-229` |

**共同点**：所有存储层错误直接通过 `response.JSONServerError()` 输出，HTTP 状态码 **500**，响应体为 `{"error_message": "..."}`。

#### 8.1.2 GReader 侧错误传播链

**ID 列表查询**（`stream/items/ids`）：

```
streamItemIDsHandler
  └─ parseStreamFilterFromRequest()  ← Stream 解析错误也走 500（见 7.2.5）
  └─ rm.Streams[0].Type dispatch:
       ├─ ReadingListStream → handleReadingListStreamHandler
       │    └─ getItemRefsAndContinuation(builder, rm)
       │         ├─ builder.GetEntryIDs() 失败 → JSONServerError (500)
       │         └─ builder.CountEntries() 失败 → JSONServerError (500)
       ├─ StarredStream    → handleStarredStreamHandler  ← 同上
       ├─ ReadStream       → handleReadStreamHandler     ← 同上
       └─ FeedStream       → handleFeedStreamHandler
            ├─ strconv.ParseInt(rm.Streams[0].ID) 失败 → JSONServerError (500)
            └─ getItemRefsAndContinuation 同上
```

**内容详情查询**（`stream/items/contents`）：

```
streamItemContentsHandler
  ├─ checkOutputFormat() 失败 → JSONBadRequest (400)  ← 注意：这是客户端错误
  ├─ r.ParseForm() 失败 → JSONServerError (500)
  ├─ parseStreamFilterFromRequest() 失败 → JSONServerError (500)
  ├─ parseItemIDsFromRequest() 失败 → JSONBadRequest (400)
  └─ builder.GetEntries() 失败 → JSONServerError (500)
```

**写操作的存储错误**：

| 端点 | 存储操作 | 错误响应 | 代码行 |
|------|----------|----------|--------|
| `edit-tag` | `ToggleBookmark` / `SetEntriesStatus` | `JSONServerError` (500) | `handler.go:314-335` |
| `subscription/edit` (ac=subscribe) | `CreateFeed` | `JSONBadRequest` (400) + error 消息 | `handler.go:532-536` |
| `subscription/edit` (ac=unsubscribe) | `RemoveFeed` | `JSONServerError` (500) | `handler.go:565-569` |
| `quickadd` | `CreateFeed` | `JSONBadRequest` (400) + "Quickadd failed" | `handler.go:463-470` |
| `mark-all-as-read` | `UpdateCategoryEntriesStatus` 等 | `JSONServerError` (500) | `handler.go:1199` 等 |

**注意不一致**：`CreateFeed` 失败（如 URL 无效）在 `quickadd` 和 `subscription/edit` 中返回 **400**，但其他存储操作失败返回 **500**。这是因为 `CreateFeed` 的错误可能是客户端参数问题（无效 URL），也可能是数据库问题，没有做细粒度区分。

#### 8.1.3 JSONServerError 的响应格式

`internal/http/response/json.go:88-97`：

```go
func JSONServerError(w http.ResponseWriter, r *http.Request, err error) {
    errorMessage := "Internal Server Error"
    if config.Opts.HasDebugMode() {
        errorMessage = err.Error()
    }
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(http.StatusInternalServerError)
    json.NewEncoder(w).Encode(map[string]string{"error_message": errorMessage})
}
```

- 正常模式：HTTP 500 + `{"error_message":"Internal Server Error"}`，不暴露内部错误细节
- Debug 模式：HTTP 500 + `{"error_message":"<原始错误信息>"}`，包含完整错误栈或 SQL 错误

这是一条**安全边界**——生产环境不向客户端泄露内部错误信息，防止攻击者通过错误消息推断数据库结构或查询逻辑。

---

### 8.2 GoogleReaderUserGetIntegration：用户不存在 vs 连库失败

`GoogleReaderUserGetIntegration()`（`storage/integration.go:84-107`）返回两类 error，但在中间件中被**统一处理**。

#### 8.2.1 存储层的两类错误

```go
err := s.db.QueryRow(query, username).Scan(
    &integration.UserID, &integration.GoogleReaderEnabled,
    &integration.GoogleReaderUsername, &integration.GoogleReaderPassword,
)
if errors.Is(err, sql.ErrNoRows) {
    return &integration, fmt.Errorf(`store: unable to find this user: %s`, username)
} else if err != nil {
    return &integration, fmt.Errorf(`store: unable to fetch user: %v`, err)
}
```

| 错误类型 | 触发条件 | error 消息 | 返回的 integration |
|----------|----------|-----------|-------------------|
| **用户不存在** | 查询返回 0 行（`sql.ErrNoRows`） | `"store: unable to find this user: <username>"` | 零值结构体（字段全空） |
| **连库失败** | 数据库连接错误、SQL 语法错误、超时等 | `"store: unable to fetch user: <底层错误详情>"` | 零值结构体 |

值得注意的是，两类错误都返回了**零值的 integration 结构体**，而不是 `nil`。这意味着调用方可以安全地读取结构体字段（虽然值都是空/零），但必须先检查 error。

#### 8.2.2 中间件层的统一处理

`middleware.go:128-136`：

```go
if integration, err = m.store.GoogleReaderUserGetIntegration(parts[0]); err != nil {
    slog.Warn("[GoogleReader] No user found with the given Google Reader username",
        slog.Bool("authentication_failed", true),
        slog.String("client_ip", clientIP),
        slog.String("user_agent", r.UserAgent()),
        slog.Any("error", err),
    )
    sendUnauthorizedResponse(w, r)
    return
}
```

**关键发现**：无论用户不存在还是数据库故障，`if err != nil` 都进入同一条分支，产生**完全相同的客户端响应**：

- HTTP 401
- `X-Reader-Google-Bad-Token: true`
- Body: `Unauthorized`

**区别仅在服务端日志**：
- 日志级别相同：都是 `Warn`
- 日志消息相同：都是 `"No user found with the given Google Reader username"`
- 但 `slog.Any("error", err)` 会记录不同的 error 详情，运维人员可区分

#### 8.2.3 用户不存在返回 401 的安全理由

这是**防止用户名枚举攻击**的标准安全实践：

- 如果用户不存在返回 404 或不同的错误消息，攻击者可以通过逐个尝试用户名来确认哪些用户名是有效的
- 统一返回 401 + 相同的错误消息，攻击者无法区分「用户名不存在」和「密码错误」
- `X-Reader-Google-Bad-Token` Header 在两种情况下都会设置，进一步确保了不可区分性

**类似设计**：
- `GoogleReaderUserCheckPassword()` 也有同样的模式（`storage/integration.go:57-81`）：用户不存在和密码错误都返回 error，调用方统一返回 `JSONUnauthorized`
- Fever 的 `UserByFeverToken()` 中用户不存在返回 `nil, nil`（零值用户 + nil error），同样在 `user == nil` 分支返回 `auth:0`，不与数据库错误区分

#### 8.2.4 连库失败时的行为问题

当数据库真的挂了的时候，鉴权层返回 401 而不是 500，有几个隐含的影响：

| 影响 | 说明 |
|------|------|
| **监控盲点** | 数据库故障时表现为大量「认证失败」，而不是「服务器错误」，监控告警可能不触发 |
| **客户端误导** | 客户端看到 401 会尝试刷新 Token 或提示用户「密码错误」，而实际上是服务端故障 |
| **安全收益** | 不暴露数据库状态给潜在攻击者，防止通过错误响应推断基础设施状态 |

这是一个**安全优先于可观测性**的设计选择。

---

### 8.3 HTML 媒体代理重写的错误处理

`mediaproxy.RewriteDocumentWithAbsoluteProxyURL()` 是 Fever 和 GReader 共同调用的内容处理函数，其错误处理策略是**容错降级**——解析失败不影响请求，也不影响单条 Entry，最多是不代理媒体 URL。

#### 8.3.1 两个错误点与降级策略

`genericProxyRewriter()`（`mediaproxy/rewriter.go:27-95`）中有两处可能失败：

| 错误点 | 触发条件 | 处理方式 | 代码行 |
|--------|----------|----------|--------|
| HTML 解析失败 | `goquery.NewDocumentFromReader()` 返回 err（如 HTML 格式严重损坏、内存不足） | `return htmlDocument` —— 返回原始 HTML，不做任何重写 | `rewriter.go:33-36` |
| Body 提取失败 | `doc.FindMatcher(goquery.Single("body")).Html()` 返回 err | `return htmlDocument` —— 同上，返回原始 HTML | `rewriter.go:89-92` |

**设计意图**：媒体代理是「增值功能」，核心价值是保护用户隐私和源站带宽。即使代理失效，文章内容本身仍然可读，只是图片/视频直接从源站加载。这是**功能降级（Graceful Degradation）** 模式。

#### 8.3.2 单条媒体 URL 解析失败的粒度处理

`shouldProxifyURL()`（`rewriter.go:110-124`）中每条 URL 单独判断：

```go
parsedURL, err := url.Parse(mediaURL)
if err != nil || !parsedURL.IsAbs() || parsedURL.Host == "" {
    return false  // 不代理，保留原 URL
}
```

- 单张图片的 URL 格式非法 → 这张图片不代理，其他图片正常代理
- 不会因为某一条 src 解析失败导致整篇文章内容出问题
- 粒度精确到**单个媒体元素**

`proxifySourceSet()` 处理 `srcset` 属性时也是同理——每个 image candidate 单独处理，坏的 URL 只是跳过代理，不会导致整个 srcset 属性出错。

#### 8.3.3 对 Entry 和整个请求的影响层级

| 错误层级 | 影响范围 | 结果 |
|----------|----------|------|
| 单条媒体 URL 解析失败 | 该 URL 不代理 | 对应图片/视频直接加载源站 URL，其他内容正常 |
| 整篇 HTML 解析失败 | 该 Entry 内容不重写 | 该 Entry 返回原始 HTML，其他 Entry 正常 |
| 全部 Entry 解析失败 | 所有内容不重写 | 请求成功，只是所有媒体都不代理 |

**不会出现的情况**：
- ❌ 单条 Entry 解析失败导致整条请求 500
- ❌ 单条 Entry 解析失败导致该 Entry 从列表中消失
- ❌ 单条 Entry 解析失败返回空内容（实际上返回原始内容）

这与流查询 SQL 出错的行为形成鲜明对比：
- **存储层错误**：致命错误，整个请求失败（500）
- **内容渲染错误**：非致命错误，局部降级（保留原始内容）

#### 8.3.4 附件（Enclosure）的代理错误处理

GReader 的 `streamItemContentsHandler`（`handler.go:694`）还会调用附件代理：

```go
entry.Enclosures.ProxifyEnclosureURL(config.Opts.MediaProxyMode(), config.Opts.MediaProxyResourceTypes())
```

`ProxifyEnclosureURL` 是 `model.Enclosures` 类型的方法，逻辑同样是容错式——URL 不合法或不需要代理时跳过，不返回 error。Fever 响应中不处理附件代理（Fever 的 item 结构没有 enclosure 字段）。

---

### 8.4 错误处理层级总览

将存储层和媒体重写的错误处理放在一起看，可以归纳出三层错误处理策略：

```
┌─────────────────────────────────────────────────────────────┐
│  第 1 层：请求入口 / 参数解析                                │
│  （Stream ID 格式错误、Item ID 格式错误、output 不对等）     │
│  → HTTP 400 BadRequest 或 HTTP 401 Unauthorized            │
│  → 客户端可修复                                             │
├─────────────────────────────────────────────────────────────┤
│  第 2 层：存储层 / SQL 查询                                 │
│  （数据库连接失败、查询超时、约束冲突等）                    │
│  → HTTP 500 Internal Server Error                          │
│  → 服务端问题，客户端无法修复                               │
│  → Debug 模式暴露详情，生产模式隐藏细节                      │
├─────────────────────────────────────────────────────────────┤
│  第 3 层：内容渲染 / 媒体重写                               │
│  （HTML 解析失败、单条 URL 格式错误等）                     │
│  → 静默降级，返回原始内容                                   │
│  → 非致命错误，不影响请求成功性                             │
└─────────────────────────────────────────────────────────────┘
```

**设计一致性观察**：

1. **越靠近请求入口，错误越明确**（400/401 有明确语义）
2. **越深入业务逻辑，错误越模糊**（500 统一内部错误，生产环境不泄露细节）
3. **纯数据转换类错误不抛出**（媒体重写、ID 格式转换等），走降级或跳过策略
4. **鉴权错误全部收敛为统一响应**，防止信息泄露
5. **Fever 与 GReader 在存储层错误处理上完全一致**，都通过 `response.JSONServerError` 输出 500

---

## 9. Fever HTTP 200 响应的缓存风险与运维影响

### 9.1 代码层面的响应头核对

#### 9.1.1 Fever 响应无任何 Cache-Control

对比 Fever 和 GReader 的所有响应函数经过的 Builder 链路：

| 响应函数 | 调用链 | Cache-Control |
|----------|--------|---------------|
| Fever `response.JSON(w, r, newAuthFailureResponse())` | `JSON()` → `NewBuilder().WithHeader("Content-Type", ...).Write()` | **无** |
| Fever `response.JSON(w, r, result)`（成功响应） | 同上 | **无** |
| GReader `response.JSON(w, r, ...)` | 同上 | **无** |
| GReader `sendUnauthorizedResponse()` | `NewBuilder().WithStatus(401).WithHeader("X-Reader-Google-Bad-Token", ...)` | **无** |
| UI `response.HTML(w, r, ...)`（管理后台） | `NewBuilder().WithHeader("Cache-Control", "no-cache, max-age=0, must-revalidate, no-store")` | `no-cache, max-age=0, must-revalidate, no-store` |
| 静态资源 `WithCaching()` | `ETag` + `Cache-Control: public, max-age=N, immutable` | `public, max-age=N, immutable` |

**代码证据**：

- `internal/http/response/json.go:17-28` — `JSON()` 只设置 `Content-Type`，不设置任何缓存头：
  ```go
  func JSON(w http.ResponseWriter, r *http.Request, body any) {
      responseBody, err := json.Marshal(body)
      ...
      NewBuilder(w, r).
          WithHeader("Content-Type", jsonContentTypeHeader).
          WithBodyAsBytes(responseBody).
          Write()
  }
  ```
- `internal/http/response/builder.go:34-36` — `NewBuilder()` 初始化的 headers 是空 map，不自带任何头：
  ```go
  func NewBuilder(w http.ResponseWriter, r *http.Request) *Builder {
      return &Builder{w: w, r: r, statusCode: http.StatusOK,
          headers: make(http.Header), enableCompression: true}
  }
  ```
- `internal/http/response/builder.go:126-134` — `writeHeaders()` 仅追加安全头，没有缓存控制：
  ```go
  func (b *Builder) writeHeaders() {
      b.headers.Set("X-Content-Type-Options", "nosniff")
      b.headers.Set("X-Frame-Options", "DENY")
      b.headers.Set("Referrer-Policy", "no-referrer")
      ...
  }
  ```
- `internal/http/server/middleware.go:16-47` — 全局中间件只在 HTTPS 时加 `Strict-Transport-Security`，没有任何缓存相关 Header。

**最终结论**：Fever 和 GReader 的所有 JSON 响应（包括鉴权失败的 HTTP 200）**不设置任何 `Cache-Control`、`Expires`、`Pragma`、`ETag`、`Last-Modified` 头**，也**不设置 `Set-Cookie`**（`Set-Cookie` 仅出现在 UI Web 登录 `internal/ui/auth.go:44`）。

#### 9.1.2 仓库内无管理员清缓存接口

搜索整个 `internal/` 目录（见 grep 结果 100 行）：

- 只有 `/history/flush`（`internal/ui/ui.go:51`）用于清空用户的阅读历史，不是清缓存接口
- `FlushAllSessions()`（`internal/storage/web_session.go:249-250`）清空 Web 会话，无 HTTP 路由
- `certificateCache`（`internal/storage/certificate_cache.go`）是 ACME 证书缓存，与 API 响应无关
- `tzCache`（`internal/timezone/timezone.go:14`）是 Go 内存级时区缓存
- `compiledRegexesCache`（`internal/reader/filter/filter.go:49`）是 Go 内存级正则缓存

**结论**：仓库内**没有任何管理员清缓存的 HTTP API**，也没有 PURGE / BAN 相关的处理函数。如果 CDN 缓存了错误响应，只能通过 CDN 控制台手动清除或等待 TTL 过期。

---

### 9.2 主流 CDN / 反向代理默认配置的缓存行为

根据 RFC 7234 与各产品官方文档，没有显式缓存指令时，HTTP 200 响应是否被缓存取决于产品实现。以下是 Fever 鉴权失败响应 `HTTP 200 + {"api_version":3,"auth":0}` 在常见部署场景下的风险评估。

#### 9.2.1 Cloudflare 默认配置

| 条件 | 行为 | 依据 |
|------|------|------|
| Content-Type: application/json | **可能被缓存** | Cloudflare 默认缓存文件类型列表包含 JSON（属于 `text/json`/`application/json` 类别） |
| 无 Cache-Control 头 | **使用默认 TTL** | Cloudflare 默认 TTL：Business/Enterprise 4 小时，Pro 2 小时，Free 计划 4 小时 |
| 无 Set-Cookie 头 | **不阻止缓存** | Cloudflare 仅在有 Set-Cookie 时默认不缓存（Bypass Cache） |
| Request URL: `/fever/` | **命中缓存** | 路径层面默认没有例外，除非配置了「Cache Rules」排除 `/fever/` 和 `/reader/` |

**缓存有效期**：2~4 小时（按计划等级）

**风险等级**：🔴 **高**

一个未授权的首次请求（如攻击者探测）如果先于合法用户请求到达，Cloudflare 会将 `auth:0` 响应缓存 2~4 小时，期间所有正常用户都会被拒绝访问。

#### 9.2.2 nginx 默认配置（`proxy_cache`）

| 条件 | 行为 | 依据 |
|------|------|------|
| `proxy_cache` 启用 + 无显式缓存头 | **不缓存** | nginx `proxy_cache` 默认只在响应含 `Cache-Control: public`、`Expires` 或 `X-Accel-Expires` 时才缓存。无这些头时不缓存。 |
| Content-Type: application/json | 不影响 | nginx 不按 Content-Type 决定是否缓存，只看缓存头 |

**缓存有效期**：不缓存（仅当管理员显式配置 `proxy_cache_valid 200 10m;` 这类指令时才缓存）

**风险等级**：🟢 **低**（默认安全）

**但是**：如果管理员配置了 `proxy_cache_valid any 10m;` 或 `proxy_cache_valid 200 1h;`，则会缓存所有 200 响应，包括 `auth:0`，风险上升到 🔴 高。

#### 9.2.3 Varnish 默认配置（`builtin.vcl`）

| 条件 | 行为 | 依据 |
|------|------|------|
| 无 Cache-Control 头 | **不缓存** | `vcl_fetch` builtin 规则：`set beresp.ttl = 120s;` 仅用于静态资源；但后续 `return(deliver)` 默认不进入 `cacheable` 判断 |
| 无 Set-Cookie 头 | 不阻止缓存 | 但无 `Cache-Control: public` 时不会执行缓存动作 |
| POST 请求 | **永不缓存** | Varnish 默认不缓存 POST 请求 |

**缓存有效期**：GET 请求无缓存头时不缓存（若管理员显式配置 `set beresp.ttl = 1h;` 则缓存 1 小时）。Fever 写操作都是 POST，天然不被缓存。

**风险等级**：🟢 **低**（默认安全）

#### 9.2.4 总结对比表

| 代理/CDN | 默认是否缓存 Fever `auth:0` 200 响应 | 默认 TTL | 触发缓存的条件 |
|----------|-------------------------------------|----------|---------------|
| Cloudflare Free/Pro/Enterprise | ✅ **是** | 2~4 小时 | 默认即启用，无需额外配置 |
| nginx proxy_cache | ❌ 否 | N/A | 需显式 `proxy_cache_valid` |
| Varnish builtin.vcl | ❌ 否 | N/A | 需显式 `set beresp.ttl` |
| Squid 默认 | ❌ 否 | N/A | 需显式配置 refresh_pattern |
| Fastly 默认 | ✅ 是 | 3600 秒 | 默认 TTL 对无缓存头的 200 |
| Akamai 默认 | ✅ 是 | 600 秒 | 无缓存头时按 MIME 类型默认缓存 |

**高危部署组合**：
- Miniflux + Cloudflare 无自定义 Cache Rules → 所有 Fever 用户 2~4 小时内无法登录
- Miniflux + Cloudflare + `/fever/` 路径命中 Page Rules 缓存 → 风险更高（自定义 TTL 可能更长）

---

### 9.3 第三方客户端如何区分真假未授权

由于 Fever 鉴权失败返回 HTTP 200，客户端和代理层无法通过状态码区分。以下从代码和协议层分析客户端可用的区分手段。

#### 9.3.1 Fever 客户端的判断逻辑（代码层面）

根据 `internal/fever/README.md:33-40` 和 `internal/fever/response.go:14-21`，Fever 协议要求客户端解析 JSON body：

**鉴权失败响应体**：
```json
{"api_version": 3, "auth": 0}
```

**鉴权成功响应体**：
```json
{"api_version": 3, "auth": 1, "last_refreshed_on_time": 1710000000, ...}
```

客户端判断伪代码：
```go
resp, _ := http.Get("https://example.com/fever/?api_key=xxx")
var result feverResponse
json.NewDecoder(resp.Body).Decode(&result)
if result.Auth == 0 {
    // 视为未授权
    return ErrAuthFailed
}
```

**CDN 缓存了 `auth:0` 后的表现**：
- 合法用户请求时，CDN 返回缓存的 `{"api_version":3,"auth":0}`
- 客户端解析 `auth==0` → 提示「用户名或密码错误」
- 用户反复检查密码无果 → 运维噩梦

**客户端能用来排除缓存的辅助信号**：

| 信号 | 真未授权 | CDN 缓存的假未授权 | 可靠性 |
|------|---------|-------------------|--------|
| HTTP 状态码 200 | ✅ 是 | ✅ 是 | 无法区分 |
| `auth` 字段 == 0 | ✅ 是 | ✅ 是 | 无法区分 |
| `api_version` 字段存在 | ✅ 是 | ✅ 是 | 无法区分 |
| `last_refreshed_on_time` 字段 | ❌ 不存在（auth:0 响应不含此字段） | ❌ 不存在 | 无法区分 |
| HTTP 响应头 `Age` | 通常不存在 | Cloudflare 会加 `Age:` Header | **部分可用** |
| HTTP 响应头 `CF-Cache-Status` | 不存在 | `HIT` | **Cloudflare 可用** |
| HTTP 响应头 `X-Cache` | 不存在 | `HIT` | **Varnish/nginx 可用** |
| Body 中有 `error_message` 字段 | ❌ 不存在（仅 500 时才出现） | ❌ 不存在 | 无法区分 |

**客户端无法从响应内容层面区分**真假未授权。只能通过 CDN 特有的 Header（`CF-Cache-Status: HIT`、`X-Cache: HIT`、`Age: >0`）间接怀疑是缓存问题，但这些头不是 Fever 协议的一部分，多数通用 Fever 客户端不检查。

#### 9.3.2 GReader 客户端的天然优势

GReader 鉴权失败返回 HTTP 401，代理层天然不缓存（除非管理员强行配置缓存 401，这非常罕见）。此外：

- HTTP 401 不是 RFC 7231 定义的「可缓存」状态码（200/203/204/206/300/301/404/405/410/414/501 才是默认可缓存的）
- `X-Reader-Google-Bad-Token: true` Header 让客户端可以进一步区分是 Token 过期还是完全未授权

**对比总结**：

| 维度 | Fever 200 + auth:0 | GReader 401 + Unauthorized |
|------|-------------------|---------------------------|
| RFC 默认可缓存 | ✅ 是（200 默认可缓存） | ❌ 否（401 默认不缓存） |
| CDN 默认行为 | Cloudflare 等会缓存 2~4 小时 | 不缓存 |
| 客户端区分难度 | 高（需依赖非标准 CDN Header） | 低（HTTP 状态码即可） |
| 对客户端的影响 | 误报「密码错误」 | 正常报错或尝试刷新 Token |
| 受影响的操作 | 所有 Fever 操作（读写都是 200） | 不受影响 |

---

### 9.4 运维缓解建议（基于代码现状）

仓库代码层面不提供缓存防护，运维需要在外层代理层加固：

#### 9.4.1 必须：对 API 路径设置 Cache-Control

在反向代理（nginx / Caddy / Cloudflare Rules）层为以下路径追加响应头：

```
/fever/*        → Cache-Control: no-store, private, no-cache, must-revalidate
/reader/*       → Cache-Control: no-store, private, no-cache, must-revalidate
/accounts/*     → Cache-Control: no-store, private, no-cache, must-revalidate
```

nginx 示例配置：
```nginx
location /fever/ {
    add_header Cache-Control "no-store, private, no-cache, must-revalidate" always;
    proxy_pass http://miniflux;
}
location /reader/ {
    add_header Cache-Control "no-store, private, no-cache, must-revalidate" always;
    proxy_pass http://miniflux;
}
```

Cloudflare 配置路径：
- 规则 → Cache Rules → 新建规则
- 匹配：`URI Path starts with /fever/ OR URI Path starts with /reader/`
- 动作：设置缓存级别 → Bypass

#### 9.4.2 备选：仅对 POST 放行 Cache（效果有限）

由于 Fever 写操作使用 POST，读操作使用 GET。仅对 `/fever/` 的 GET 请求禁用缓存：

```nginx
location /fever/ {
    if ($request_method = GET) {
        add_header Cache-Control "no-store, private, no-cache, must-revalidate" always;
    }
    proxy_pass http://miniflux;
}
```

但这仍无法防止攻击者先用 GET 请求缓存 `auth:0`，所以建议全路径禁用。

#### 9.4.3 监控：区分「真未授权」与「缓存污染」

如果已经发生了疑似缓存污染，排查方法：

```bash
# 检查 Cloudflare 返回头，看是否 HIT
curl -I 'https://your-miniflux.com/fever/?api_key=WRONG_TOKEN' | grep -iE '(cf-cache-status|age|x-cache)'

# 如果返回 CF-Cache-Status: HIT 且 Age > 0，说明已经缓存了未授权响应
# 解决：Cloudflare Dashboard → Caching → Configuration → Purge Cache → Purge Everything
```

#### 9.4.4 代码修复方向（仓库尚未实现）

要从根本上解决，需要在 `response.JSON()` 或 Fever/GReader Handler 中主动追加缓存控制头。参考 UI 层 `response.HTML()` 的做法：

```go
// 建议修改方向：在 JSON() 中增加缓存头（仅针对 API 路径）
func JSON(w http.ResponseWriter, r *http.Request, body any) {
    ...
    builder := NewBuilder(w, r).
        WithHeader("Content-Type", jsonContentTypeHeader).
        WithHeader("Cache-Control", "no-store, private, no-cache, must-revalidate")
    ...
}
```

但当前代码**未做此修改**，运维不能依赖应用层处理。

---

## 10. response.JSON 统一加 Cache-Control 头的影响面评估

### 10.1 Fever 全量响应类型清单

对 `internal/fever/` 目录下所有 `response.` 调用的完整核对（见 grep 30 行结果）：

| 调用函数 | 次数 | 使用场景 | 走 `response.JSON` |
|----------|------|---------|-------------------|
| `response.JSON` + `newBaseResponse()` | 1 | 无参数默认响应 | ✅ 是 |
| `response.JSON` + `groupsResponse` | 1 | `?groups` 分类列表 | ✅ 是 |
| `response.JSON` + `feedsResponse` | 1 | `?feeds` 订阅源列表 + favicons | ✅ 是 |
| `response.JSON` + `faviconsResponse` | 1 | `?favicons` 图标列表（base64 data URI） | ✅ 是 |
| `response.JSON` + `itemsResponse` | 1 | `?items` 文章列表 | ✅ 是 |
| `response.JSON` + `unreadResponse` | 1 | `?unread_item_ids` 未读 ID 列表 | ✅ 是 |
| `response.JSON` + `savedResponse` | 1 | `?saved_item_ids` 已收藏 ID 列表 | ✅ 是 |
| `response.JSON` + `newBaseResponse()` | 5 | 写操作成功返回（mark 三种 + toggle 两次路径） | ✅ 是 |
| `response.JSON` + `newAuthFailureResponse()` | 3 | 鉴权中间件三种失败路径 | ✅ 是 |
| `response.JSONServerError` | 17 | 各类存储层错误 | ✅ 是（共用 Builder） |

**结论**：Fever 的**全部 30 处响应输出**都直接或间接经过 `response.JSON()` 家族函数（`JSON`/`JSONServerError` 等），没有任何例外。Fever 响应中提到的 `favicons` 数据是 base64 编码内嵌在 JSON body 里（`faviconsResponse.Favicons[].Data`），不是独立的二进制 HTTP 端点。

### 10.2 GReader 响应类型清单（含 nonstandard 文本响应）

对 `internal/googlereader/` 目录下所有响应输出的完整核对（见 grep 95 行结果）：

#### 10.2.1 走 `response.JSON` 家族的响应（JSON 格式）

| 调用函数 | 次数 | 典型使用场景 |
|----------|------|------------|
| `response.JSON` | 11 | `subscription/list`、`tag/list`、`stream/items/ids`（4 种 Stream）、`stream/items/contents`、`subscription/quickadd`（成功+失败两种）、`tokenHandler`(JSON 格式)、`user-info`、fallback 返回 `[]` |
| `response.JSONBadRequest` | 21 | `output` 格式错误、`i` 为空、Stream ID 非法、标签冲突、`ac` 不识别、`edit-tag` 中 item ID 解析失败等 |
| `response.JSONServerError` | 41 | 各类存储层错误、`ParseForm` 失败、Stream 解析错误（此处应是 400 但返回 500，见 7.2.5） |
| `response.JSONUnauthorized` | 4 | `ClientLogin` 中表单解析失败、用户空、密码校验失败、token 鉴权失败 |
| `response.JSONNotFound` | 2 | `rename-tag` / `mark-all-as-read` 中找不到 Category/Feed |

JSON 家族合计：**79 处**。

#### 10.2.2 **不走** `response.JSON` 的响应（非 JSON 格式 / 非标准路径）

| 调用方式 | 次数 | 使用场景 | 代码位置 | 有缓存头？ |
|----------|------|---------|---------|-----------|
| `response.Text(w, r, "OK")` | 6 | `edit-tag`、`subscription/edit`、`mark-all-as-read`、`disable-tag`、`rename-tag` 成功返回 | `handler.go:319/601/774/841/1231` | ❌ 无 |
| `response.Text(w, r, loginResponse.String())` | 1 | `ClientLogin` 非 JSON 模式 | `handler.go:146` | ❌ 无 |
| `response.Text(w, r, token)` | 1 | `tokenHandler` 默认路径（非 `output=json`） | `handler.go:184` | ❌ 无 |
| `sendUnauthorizedResponse()` 直接 NewBuilder | N/A | `/reader/api/0/*` 鉴权失败 401 | `response.go:122-129` | ❌ 无 |

**关键代码核对**：

`response.Text()` 实现（`text.go:8-13`）：
```go
func Text(w http.ResponseWriter, r *http.Request, body string) {
    NewBuilder(w, r).
        WithHeader("Content-Type", `text/plain; charset=utf-8`).
        WithBodyAsString(body).
        Write()
}
```
和 `response.JSON()` 一样，`Text()` 也只设置 `Content-Type`，**没有任何缓存头**。

`sendUnauthorizedResponse()` 实现（`response.go:122-129`）：
```go
func sendUnauthorizedResponse(w http.ResponseWriter, r *http.Request) {
    response.NewBuilder(w, r).
        WithStatus(http.StatusUnauthorized).
        WithHeader("X-Reader-Google-Bad-Token", "true").
        WithHeader("Content-Type", "text/plain; charset=utf-8").
        WithBodyAsString("Unauthorized").
        Write()
}
```
同样**没有缓存头**。

**结论**：如果只在 `response.JSON()` 中加 `Cache-Control`，则 **8 处响应（6 个 "OK" 文本 + ClientLogin 纯文本 + token 纯文本）和 `sendUnauthorizedResponse()` 仍然没有缓存头保护**。需要同时修改 `response.Text()` 并在 `sendUnauthorizedResponse()` 中追加缓存头。

### 10.3 图标、静态资源响应与 Fever/GReader API 的路径隔离

#### 10.3.1 图标的响应路径（完全不经过 response.JSON）

**Fever favicon**：不是独立 HTTP 端点，而是内嵌在 `?favicons` JSON 响应中的 base64 data URI：
```go
// fever/response.go:115-118
type favicon struct {
    ID   int64  `json:"id"`
    Data string `json:"data"`  // 形如 "image/png;base64,iVBORw0KGgoAAAA..."
}
```
**走 `response.JSON`**，加 `no-store` 会影响这条响应，但内嵌的 data URI 本身无法被 CDN 单独缓存，只有整体 JSON 会。由于 favicon 数据不常变但也会偶尔变化（用户添加新源时），`no-store` 对客户端体验影响极小。

**GReader subscription 图标**：通过 URL 指向独立端点，不走 `response.JSON`。

GReader `subscriptionResponse.IconURL` 生成（`handler.go:518-523`）：
```go
func (h *greaderHandler) feedIconURL(f *model.Feed) string {
    if f.Icon != nil && f.Icon.ExternalIconID != "" {
        return config.Opts.BaseURL() + "/feed-icon/" + f.Icon.ExternalIconID
    }
    return ""
}
```

实际图标端点（`ui/feed_icon.go:14-36`）**使用 `WithCaching()`，不受 `response.JSON` 修改影响**：
```go
response.NewBuilder(w, r).WithCaching(icon.Hash, 72*time.Hour, func(b *response.Builder) {
    b.WithHeader("Content-Type", icon.MimeType)
    b.WithBodyAsBytes(icon.Content)
    ...
})
```

`WithCaching()` 设置的头（`builder.go:87-92`）：
```go
b.headers.Set("ETag", etag)
b.headers.Set("Cache-Control", fmt.Sprintf("public, max-age=%d, immutable", int64(duration.Seconds())))
b.headers.Set("Expires", ...)
```

#### 10.3.2 所有使用 `WithCaching()` 的静态资源汇总

| 端点 | 缓存策略 | 路径 | 是否走 response.JSON |
|------|---------|------|---------------------|
| `/favicon.ico` | ETag + public 48h immutable | `ui/static_favicon.go:22` | ❌ 否 |
| `/feed-icon/{id}` | ETag + public 72h immutable | `ui/feed_icon.go:27` | ❌ 否 |
| `/static/stylesheets/*.css` | ETag + public 48h immutable | `ui/static_stylesheet.go:22` | ❌ 否 |
| `/static/javascripts/*.js` | ETag + public 48h immutable | `ui/static_javascript.go:28` | ❌ 否 |
| `/static/app-icons/*` | ETag + public 72h immutable | `ui/static_app_icon.go:25` | ❌ 否 |
| `/share/{code}` | ETag + public 72h immutable | `ui/share.go:45` | ❌ 否 |
| `/proxy/{encryptedURL}` | ETag + public 72h immutable | `ui/proxy.go:145` | ❌ 否 |

**关键结论**：所有二进制资源、静态资源、媒体代理都使用 `Builder.WithCaching()` 方法显式设置缓存头，**完全不经过 `response.JSON()` 的代码路径**。在 `response.JSON()` 中加 `no-store` 不会对这些合法缓存的资源产生任何负面影响。

### 10.4 统一加 Cache-Control 头对各响应类型的影响评估

#### 10.4.1 影响矩阵

| 响应类型 | 代表端点 | 修改 `response.JSON` 加 no-store | 额外修改 `Text()` + `sendUnauthorized()` | 对合法缓存的影响 |
|---------|---------|--------------------------------|----------------------------------------|---------------|
| Fever 鉴权失败 200 | `auth:0` 响应 | ✅ 被覆盖（核心目标） | N/A | ✅ 正面：杜绝 CDN 缓存污染 |
| Fever 业务 JSON | `?items`、`?feeds` 等 | ✅ 被覆盖 | N/A | 🟡 中性：客户端每次请求都会拉取，但文章状态实时变化本来就不应该缓存 |
| GReader JSON 成功 | `subscription/list`、`stream/items/ids`、`stream/items/contents` | ✅ 被覆盖 | N/A | 🟡 中性：订阅/标签/文章状态实时变化，缓存反而导致脏数据 |
| GReader JSON 错误 | 400/401/404/500 JSON | ✅ 被覆盖 | N/A | ✅ 正面：错误响应不应缓存 |
| GReader "OK" 文本 | `edit-tag`、`subscription/edit`、`mark-all-as-read` | ❌ 未覆盖 | ✅ 需额外处理 | 🟢 影响极小：body 仅 "OK" 3 字节，不缓存也无性能损失 |
| GReader ClientLogin 纯文本 | `SID=xxx\nAuth=xxx\n` | ❌ 未覆盖 | ✅ 需额外处理 | ✅ 正面：Token 明文绝对不应被缓存 |
| GReader token 纯文本 | 编辑 Token | ❌ 未覆盖 | ✅ 需额外处理 | ✅ 正面：Token 不应被缓存 |
| GReader 401 纯文本 | `/reader/api/0/*` 鉴权失败 | ❌ 未覆盖（走 NewBuilder） | ✅ 需额外处理 | ✅ 正面：401 虽默认可缓存性低，但显式 no-store 更安全 |
| `/feed-icon/{id}` | 订阅源图标二进制 | ❌ 不经过 JSON | ❌ 不经过 Text | 🟢 完全不受影响，72h 缓存保持不变 |
| UI 静态资源（CSS/JS/图片） | `/static/*` | ❌ 不经过 JSON | ❌ 不经过 Text | 🟢 完全不受影响 |
| 媒体代理图片 | `/proxy/*` | ❌ 不经过 JSON | ❌ 不经过 Text | 🟢 完全不受影响，72h 缓存保持不变 |
| Fever favicons JSON（base64） | `?favicons` 中的内嵌 data URI | ✅ 被覆盖 | N/A | 🟡 影响极小：JSON body 中内嵌，CDN 无法单独提取缓存 |
| REST API `/v1/*`（如果启用） | Miniflux 自有 JSON API | ✅ 被覆盖（副作用） | N/A | ✅ 正面：同属用户数据，不应被公共 CDN 缓存 |

#### 10.4.2 对「合法缓存」的客户端表现分析

用户担忧的「订阅列表静态部分被缓存导致客户端刷新变慢」在实际场景中**不成立**：

1. **GReader subscription/list 返回的不是静态数据**：用户随时可能在管理后台增删订阅、移动分类，客户端本地缓存列表会与服务器不一致。客户端实现中通常用 `Continuation` 机制增量同步，而不是依赖 HTTP 层缓存。

2. **Fever feeds + feedsGroups 也会动态变化**：用户添加/删除 Feed、修改分类归属都会影响返回结构。Fever 协议中没有增量同步机制（`unread_item_ids`/`saved_item_ids` 仅针对文章），客户端每次都要全量拉取，no-store 不增加额外负担。

3. **实际性能损失可以忽略**：
   - 订阅列表 JSON 通常 < 50KB
   - 文章列表每次最多 50 条，通常 < 200KB（Fever 固定限制 50 条）
   - 这些 API 的典型调用频率是客户端后台 15~60 分钟一次，不是高频请求
   - 与从 CDN 缓存命中节省的几百毫秒相比，返回脏数据导致的客户端逻辑错误成本高得多

4. **唯一可能有性能收益的缓存已独立保护**：
   - 订阅源图标（/feed-icon/）：72h immutable 缓存，完全独立
   - 内嵌媒体图片（/proxy/）：72h immutable 缓存，完全独立
   - 这些才是体积最大（单图几十 KB 到 MB 级）、请求最多（每篇文章多图）的部分，已经有缓存保护

#### 10.4.3 修复方案的代码粒度建议

从代码改动影响面和效果综合评估，**最优的修复粒度不是直接修改 `response.JSON`，而是在路由级别包裹**：

| 方案 | 改动位置 | 覆盖范围 | 副作用 |
|------|---------|---------|--------|
| A. 改 `response.JSON()` | `json.go:18-28` | Fever + GReader + REST API 所有 JSON 响应 | 漏了 GReader 的 8 处 Text 响应和 sendUnauthorizedResponse()；可能影响未来新增的 JSON 静态资源端点（如果有的话） |
| B. 改路由层 Fever/GReader middleware | `fever/middleware.go`、`googlereader/middleware.go`，在 `next.ServeHTTP` 之后写头 | Fever 所有端点 + GReader `/reader/api/0/*` 所有端点 + `/accounts/ClientLogin` | 最优：**精确覆盖目标路径**；不影响静态资源；不影响 REST API；不遗漏 Text 响应；不遗漏 sendUnauthorizedResponse |
| C. 加全局中间件 | `server/middleware.go` | 所有 JSON/Text 响应 | 过宽：会影响 `/healthcheck`、`/metrics` 等运维端点的缓存能力 |

**推荐方案 B 的伪代码**（Fever middleware 示例）：
```go
func Middleware(store *storage.Storage) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // ... 现有鉴权逻辑 ...
            // 鉴权通过后，先写缓存头，再转发
            w.Header().Set("Cache-Control", "no-store, private, no-cache, must-revalidate")
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

GReader 需额外为 `POST /accounts/ClientLogin` 包裹相同逻辑（当前是 `HandleFunc`，没有中间件包裹）。这样修改可以保证：
- Fever 的所有 30 处响应（含 JSONServerError）全被覆盖
- GReader 的 79 处 JSON + 8 处 Text + `sendUnauthorizedResponse()` 全被覆盖
- `/feed-icon/`、`/static/`、`/proxy/`、`/favicon.ico` 完全不受影响，72h 缓存策略保持不变
- 不会误伤 REST API 之外的 JSON 端点（如 `metrics`）
