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
