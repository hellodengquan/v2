# 条目状态（已读/收藏）同步更新与未读计数刷新流程分析

## 一、概述

Miniflux 是一个 Go 语言实现的 RSS 阅读器，其条目状态（已读/未读、收藏/取消收藏）的同步更新涉及三个入口：

1. **Web 界面** (`internal/ui/`)：用户通过浏览器操作
2. **REST API v1** (`internal/api/`)：通过 `/v1/` 前缀的 JSON API
3. **兼容接口**：
   - Fever API (`internal/fever/`)：兼容 Fever 阅读器协议
   - Google Reader API (`internal/googlereader/`)：兼容 Google Reader 协议

所有入口最终调用同一套 `Storage` 层的数据库操作，通过"共享存储层"实现多入口状态同步。未读计数采用"即时计算"策略，在每次页面渲染或 API 响应时实时查询。

---

## 二、路由总入口：请求分发机制

### 2.1 顶层路由注册

**文件**: `internal/http/server/routes.go:18-73`

所有 HTTP 请求在 `newRouter()` 函数中统一注册分发：

```
根路由 (rootMux)
├── 健康检查 (无 base path)
│   ├── /liveness, /healthz
│   ├── /readiness, /readyz
│
└── 应用路由 (appMux，带 base path)
    ├── /healthcheck
    ├── /fever/          → Fever API Handler
    ├── /accounts/ClientLogin → Google Reader (登录)
    ├── /reader/api/0/   → Google Reader API Handler
    ├── /v1/             → REST API v1 Handler (需 HasAPI() 开启)
    ├── /metrics         → 指标收集
    └── /                → Web UI Handler (catch-all)
```

关键代码 (`routes.go:26-46`)：
```go
// Fever API
feverHandler := fever.Middleware(store)(fever.NewHandler(store))
appMux.Handle("/fever/", feverHandler)

// Google Reader API
googleReaderHandler := googlereader.NewHandler(store)
appMux.HandleFunc("POST /accounts/ClientLogin", googleReaderHandler.ServeHTTP)
appMux.Handle("/reader/api/0/", googleReaderHandler)

// REST API
if config.Opts.HasAPI() {
    appMux.Handle("/v1/", api.NewHandler(store, pool))
}

// Web UI (catch-all)
appMux.Handle("/", ui.Serve(store, pool))
```

**核心设计**：四个入口共享同一个 `*storage.Storage` 实例，所有状态变更都通过这同一个对象操作数据库，天然实现数据一致性。

---

## 三、数据模型：状态字段定义

### 3.1 Entry 结构体

**文件**: `internal/model/entry.go:11-47`

```go
const (
    EntryStatusUnread = "unread"  // 未读
    EntryStatusRead   = "read"    // 已读
)

type Entry struct {
    ID          int64     `json:"id"`
    UserID      int64     `json:"user_id"`
    FeedID      int64     `json:"feed_id"`
    Status      string    `json:"status"`      // 已读/未读状态
    Starred     bool      `json:"starred"`     // 收藏状态
    ChangedAt   time.Time `json:"changed_at"`  // 状态变更时间戳
    // ... 其他字段
}
```

### 3.2 状态变更请求模型

```go
type EntriesStatusUpdateRequest struct {
    EntryIDs []int64 `json:"entry_ids"`
    Status   string  `json:"status"`
}
```

---

## 四、Storage 层：状态写入的核心实现

所有入口的状态写入最终都通过 `internal/storage/entry.go` 中的函数操作数据库。

### 4.1 设置条目状态（批量）

**函数**: `SetEntriesStatus` — `storage/entry.go:407-423`

```go
func (s *Storage) SetEntriesStatus(userID int64, entryIDs []int64, status string) error {
    query := `
        UPDATE entries
        SET status=$1, changed_at=now()
        WHERE user_id=$2 AND id=ANY($3)
    `
    _, err := s.db.Exec(query, status, userID, pq.Array(entryIDs))
    return err
}
```

**特点**：
- 使用 PostgreSQL `ANY()` 语法，支持批量 ID 列表
- 同步更新 `changed_at` 时间戳
- 使用 `user_id` 条件确保数据隔离

### 4.2 设置条目状态并返回可见数量

**函数**: `SetEntriesStatusAndCountVisible` — `storage/entry.go:426-449`

```go
func (s *Storage) SetEntriesStatusAndCountVisible(userID int64, entryIDs []int64, status string) (int, error) {
    query := `
        WITH updated AS (
            UPDATE entries
            SET status=$1, changed_at=now()
            WHERE user_id=$2 AND id=ANY($3)
            RETURNING feed_id
        )
        SELECT count(*)
        FROM updated u
            JOIN feeds f ON (f.id = u.feed_id)
            JOIN categories c ON (c.id = f.category_id)
        WHERE NOT f.hide_globally AND NOT c.hide_globally
    `
    var visible int
    err := s.db.QueryRow(query, status, userID, pq.Array(entryIDs)).Scan(&visible)
    return visible, err
}
```

**特点**：
- 使用 CTE (`WITH updated AS ...`) 在**一次 SQL 调用**中完成更新+计数
- 过滤掉被标记为 `hide_globally` 的 Feed 和 Category
- Web UI 前端可据此更新未读计数显示（避免 off-by-one）

### 4.3 设置收藏状态（批量）

**函数**: `SetEntriesStarredState` — `storage/entry.go:452-469`

```go
func (s *Storage) SetEntriesStarredState(userID int64, entryIDs []int64, starred bool) error {
    query := `UPDATE entries SET starred=$1, changed_at=now() WHERE user_id=$2 AND id=ANY($3)`
    result, err := s.db.Exec(query, starred, userID, pq.Array(entryIDs))
    count, _ := result.RowsAffected()
    if count == 0 {
        return errors.New(`store: nothing has been updated`)
    }
    return nil
}
```

### 4.4 切换收藏状态（单条）

**函数**: `ToggleStarred` — `storage/entry.go:472-489`

```go
func (s *Storage) ToggleStarred(userID int64, entryID int64) error {
    query := `UPDATE entries SET starred = NOT starred, changed_at=now() WHERE user_id=$1 AND id=$2`
    // ... 检查 RowsAffected
}
```

**设计要点**：使用 `starred = NOT starred` 在数据库层原子切换，无需先 SELECT 再 UPDATE。

### 4.5 批量标记已读（按范围）

| 函数 | 位置 | 作用范围 | SQL 条件 |
|------|------|---------|---------|
| `MarkAllAsRead` | `entry.go:511-525` | 用户全部未读 | `status=$3 (unread)` |
| `MarkAllAsReadBeforeDate` | `entry.go:528-549` | 用户指定日期前 | `status=$3 AND published_at < $4` |
| `MarkGloballyVisibleFeedsAsRead` | `entry.go:552-579` | 全局可见的 Feed | `feeds.hide_globally=$4 (false)` |
| `MarkFeedAsRead` | `entry.go:582-606` | 指定 Feed | `feed_id=$3 AND status=$4` |
| `MarkCategoryAsRead` | `entry.go:609-643` | 指定分类 | 通过 JOIN feeds 关联 category_id |

---

## 五、未读计数的刷新机制

Miniflux **不使用缓存或计数器表**，每次需要时直接查询数据库实时计算。主要有以下两种计数查询方式：

### 5.1 导航栏全局未读计数

**函数**: `GetNavMetadata` — `storage/nav_metadata.go:20-97`

Web UI 的每个页面渲染时都会调用此函数获取导航栏元数据，包括全局未读计数。

```go
type NavMetadata struct {
    CountUnread     int   // 全局未读计数
    CountErrorFeeds int   // 错误 Feed 计数
    HasSaveEntry    bool  // 是否配置了保存集成
}

func (s *Storage) GetNavMetadata(userID int64) (NavMetadata, error) {
    query := `
        SELECT
            (SELECT count(*)
               FROM entries e
               JOIN feeds f ON f.id = e.feed_id
               JOIN categories c ON c.id = f.category_id
              WHERE e.user_id = $1
                AND e.status = 'unread'
                AND f.hide_globally IS FALSE
                AND c.hide_globally IS FALSE
            ) AS count_unread,
            (SELECT EXISTS( ... integrations 查询 ... )) AS has_save_entry,
            (SELECT count(*) FROM feeds WHERE ... parsing_error_count >= $2) AS count_error_feeds
    `
    // 单次查询通过子查询获取三个值
}
```

**关键特性**：
- **单一 SQL 查询**：使用子查询在一次 round-trip 中获取三个值
- **过滤隐藏项**：排除 `hide_globally` 的 Feed 和 Category（Web UI 才需要）
- **调用时机**：每个页面渲染时调用（见各 entry handler 末尾的 `navMetadata, _ := h.store.GetNavMetadata(user.ID)`）

### 5.2 按 Feed 粒度的读/未读计数

**函数**: `fetchFeedCounter` — `storage/feed_query_builder.go:302-350`

```go
func (f *feedQueryBuilder) fetchFeedCounter() (readCounters, unreadCounters map[int64]int, err error) {
    query := `
        SELECT e.feed_id, e.status, count(*)
        FROM entries e
        [INNER JOIN feeds f ON f.id=e.feed_id]  -- 仅当按 category 过滤时
        WHERE e.user_id = $1 AND e.status IN ($2, $3)
        GROUP BY e.feed_id, e.status
    `
    // 结果组装到两个 map: feed_id → count
}
```

**调用时机**：
- `FeedsWithCounters()` — 侧边栏 Feed 列表
- `FeedsByCategoryWithCounters()` — 分类内 Feed 列表
- `FetchCounters()` — API 的 `/v1/feeds/counters` 端点

### 5.3 Web UI 的计数刷新：避免 off-by-one

**文件**: `internal/ui/entry_unread.go:65-95`

在展示未读条目详情页时，存在精心设计的时序：

```go
// 步骤1: 先将条目临时设为 unread，确保分页查询正确（因为分页基于状态）
if entry.Status == model.EntryStatusRead {
    h.store.SetEntriesStatus(user.ID, []int64{entry.ID}, model.EntryStatusUnread)
}

// 步骤2: 基于未读状态查询前后导航条目
prevEntry, nextEntry, _ := h.store.NewEntryPaginationBuilder(...).WithStatus(EntryStatusUnread)...

// 步骤3: 根据用户设置决定是否标记为已读
if entry.ShouldMarkAsReadOnView(user) {
    entry.Status = model.EntryStatusRead
}
if entry.Status == model.EntryStatusRead {
    h.store.SetEntriesStatus(user.ID, []int64{entry.ID}, model.EntryStatusRead)
}

// 步骤4: 最后再获取计数，保证准确性（注释明确说明 "Fetching the counters here avoids being off by one"）
navMetadata, _ := h.store.GetNavMetadata(user.ID)
view.Set("countUnread", navMetadata.CountUnread)
```

---

## 六、Web 界面入口的状态写入

### 6.1 Web UI 路由注册

**文件**: `internal/ui/ui.go:28-184`

与条目状态相关的路由：

| 方法 | 路径 | 处理函数 | 作用 |
|------|------|---------|------|
| POST | `/mark-all-as-read` | `markAllAsRead` | 全局标记已读（仅全局可见 Feed） |
| POST | `/entry/status` | `updateEntriesStatus` | 批量更新条目状态 |
| POST | `/entry/star/{entryID}` | `toggleStarred` | 切换收藏 |
| POST | `/entry/save/{entryID}` | `saveEntry` | 发送到第三方集成 |
| POST | `/feed/{feedID}/mark-all-as-read` | `markFeedAsRead` | 标记单个 Feed 已读 |
| POST | `/category/{categoryID}/mark-all-as-read` | `markCategoryAsRead` | 标记分类已读 |
| POST | `/category/{categoryID}/feed/{feedID}/mark-all-as-read` | `markCategoryFeedAsRead` | 分类内单个 Feed 已读 |

### 6.2 批量更新条目状态

**文件**: `internal/ui/entry_update_status.go:16-35`

```go
func (h *handler) updateEntriesStatus(w http.ResponseWriter, r *http.Request) {
    var req model.EntriesStatusUpdateRequest
    json_parser.NewDecoder(r.Body).Decode(&req)
    validator.ValidateEntriesStatusUpdateRequest(&req)

    // 调用带可见计数的 Storage 函数
    count, err := h.store.SetEntriesStatusAndCountVisible(
        request.UserID(r),
        req.EntryIDs,
        req.Status,
    )
    response.JSON(w, r, count)  // 返回可见条目数量，前端用于刷新计数
}
```

### 6.3 全局标记已读（Web 专用）

**文件**: `internal/ui/unread_mark_all_read.go:13-20`

```go
func (h *handler) markAllAsRead(w http.ResponseWriter, r *http.Request) {
    // 注意：Web UI 使用的是 MarkGloballyVisibleFeedsAsRead，
    // 而非 MarkAllAsRead，只标记非隐藏的 Feed
    err := h.store.MarkGloballyVisibleFeedsAsRead(request.UserID(r))
    response.JSON(w, r, "OK")
}
```

**与 API 的区别**：API `PUT /v1/users/{userID}/mark-all-as-read` 使用 `MarkAllAsRead()`，标记**所有**未读条目（包括隐藏的）。

### 6.4 切换收藏

**文件**: `internal/ui/entry_toggle_starred.go:13-21`

```go
func (h *handler) toggleStarred(w http.ResponseWriter, r *http.Request) {
    entryID := request.RouteInt64Param(r, "entryID")
    err := h.store.ToggleStarred(request.UserID(r), entryID)
    response.JSON(w, r, "OK")
}
```

---

## 七、REST API v1 入口的状态写入

### 7.1 API v1 路由注册

**文件**: `internal/api/api.go:20-78`

与条目状态相关的路由：

| 方法 | 路径 | 处理函数 | 作用 |
|------|------|---------|------|
| PUT | `/v1/entries` | `setEntryStatusHandler` | 批量更新状态 |
| PUT | `/v1/entries/{entryID}/bookmark` | `toggleStarredHandler` | 切换收藏（别名1） |
| PUT | `/v1/entries/{entryID}/star` | `toggleStarredHandler` | 切换收藏（别名2） |
| POST | `/v1/entries/{entryID}/save` | `saveEntryHandler` | 发送到第三方集成 |
| PUT | `/v1/users/{userID}/mark-all-as-read` | `markUserAsReadHandler` | 用户全部标记已读 |
| PUT | `/v1/feeds/{feedID}/mark-all-as-read` | `markFeedAsReadHandler` | 单 Feed 标记已读 |
| PUT | `/v1/categories/{categoryID}/mark-all-as-read` | `markCategoryAsReadHandler` | 分类标记已读 |
| GET | `/v1/feeds/counters` | `fetchCountersHandler` | 获取各 Feed 计数 |

### 7.2 批量更新状态

**文件**: `internal/api/entry_handlers.go:197-215`

```go
func (h *handler) setEntryStatusHandler(w http.ResponseWriter, r *http.Request) {
    var req model.EntriesStatusUpdateRequest
    json_parser.NewDecoder(r.Body).Decode(&req)
    validator.ValidateEntriesStatusUpdateRequest(&req)

    // 注意：API 版本使用普通的 SetEntriesStatus，不返回可见计数
    err := h.store.SetEntriesStatus(request.UserID(r), req.EntryIDs, req.Status)
    response.NoContent(w, r)
}
```

**与 Web UI 的区别**：Web 使用 `SetEntriesStatusAndCountVisible` 返回计数，API 使用 `SetEntriesStatus` 返回 `204 No Content`。

### 7.3 切换收藏

**文件**: `internal/api/entry_handlers.go:217-230`

```go
func (h *handler) toggleStarredHandler(w http.ResponseWriter, r *http.Request) {
    entryID := request.RouteInt64Param(r, "entryID")
    err := h.store.ToggleStarred(request.UserID(r), entryID)
    response.NoContent(w, r)
}
```

### 7.4 导入条目时的状态设置

**文件**: `internal/api/entry_handlers.go:330-436` (`importFeedEntryHandler`)

导入单条条目时，依次调用：
1. `InsertEntryForFeed()` — 创建条目（默认 unread）
2. `SetEntriesStatus()` — 设置导入时指定的状态（默认为 read）
3. `SetEntriesStarredState()` — 如果指定 starred，则设为 true

---

## 八、Fever 兼容接口的状态写入

### 8.1 Fever Handler 分发

**文件**: `internal/fever/handler.go:31-54`

Fever API 使用查询参数而非 REST 路径来区分操作：

```go
switch {
case request.HasQueryParam(r, "groups"):       h.handleGroups(w, r)
case request.HasQueryParam(r, "feeds"):        h.handleFeeds(w, r)
case request.HasQueryParam(r, "unread_item_ids"): h.handleUnreadItems(w, r)
case request.HasQueryParam(r, "saved_item_ids"):  h.handleSavedItems(w, r)
case request.HasQueryParam(r, "items"):        h.handleItems(w, r)
case r.FormValue("mark") == "item":            h.handleWriteItems(w, r)  // 状态写入
case r.FormValue("mark") == "feed":            h.handleWriteFeeds(w, r)
case r.FormValue("mark") == "group":           h.handleWriteGroups(w, r)
}
```

### 8.2 单条目写操作（mark=item）

**文件**: `internal/fever/handler.go:401-473`

```go
func (h *feverHandler) handleWriteItems(w http.ResponseWriter, r *http.Request) {
    entryID := request.FormInt64Value(r, "id")
    // 先查询条目（用于后续 save 集成推送）
    entry, _ := h.store.NewEntryQueryBuilder(userID).WithEntryIDs(entryID).GetEntry()

    switch r.FormValue("as") {
    case "read":
        h.store.SetEntriesStatus(userID, []int64{entryID}, model.EntryStatusRead)
    case "unread":
        h.store.SetEntriesStatus(userID, []int64{entryID}, model.EntryStatusUnread)
    case "saved":
        // ★ 注意：Fever 的 saved 操作使用 ToggleStarred
        // 如果当前未收藏 → 收藏；如果已收藏 → 无操作？（ToggleStarred 会切换）
        h.store.ToggleStarred(userID, entryID)
        // 同时触发第三方集成推送（异步 goroutine）
        settings, _ := h.store.Integration(userID)
        go integration.SendEntry(entry, settings)
    case "unsaved":
        // ★ 同样使用 ToggleStarred 切换收藏状态
        h.store.ToggleStarred(userID, entryID)
    }
}
```

**潜在不一致**：Fever 的 `saved`/`unsaved` 使用 `ToggleStarred` 而非明确设置 `true`/`false`。如果客户端重复调用 `saved`，可能会导致收藏状态被意外切换。这是 Fever 协议与 Miniflux 内部模型的语义映射问题。

### 8.3 Feed 范围写操作（mark=feed）

**文件**: `internal/fever/handler.go:481-502`

```go
func (h *feverHandler) handleWriteFeeds(w http.ResponseWriter, r *http.Request) {
    feedID := request.FormInt64Value(r, "id")
    before := time.Unix(request.FormInt64Value(r, "before"), 0)
    h.store.MarkFeedAsRead(userID, feedID, before)
}
```

### 8.4 Group 范围写操作（mark=group）

**文件**: `internal/fever/handler.go:510-541`

```go
if groupID == 0 {
    h.store.MarkAllAsRead(userID)  // groupID=0 表示全部
} else {
    h.store.MarkCategoryAsRead(userID, groupID, before)
}
```

---

## 九、Google Reader 兼容接口的状态写入

### 9.1 标签编辑接口（edit-tag）

**文件**: `internal/googlereader/handler.go:187-320`

Google Reader 通过"标签"(tag) 系统表示状态：
- `user/-/state/com.google/read` → 已读
- `user/-/state/com.google/starred` → 收藏
- `user/-/state/com.google/kept-unread` → 保持未读（等价于加读标签的反向）

**处理流程**：

```go
func (h *greaderHandler) editTagHandler(w http.ResponseWriter, r *http.Request) {
    // 步骤1: 解析 add/remove tags，转换为内部 StreamType
    addTags, _ := getStreams(r.PostForm[paramTagsAdd], userID)
    removeTags, _ := getStreams(r.PostForm[paramTagsRemove], userID)
    tags, _ := checkAndSimplifyTags(addTags, removeTags)
    // tags: map[StreamType]bool
    //   ReadStream → true=标记已读, false=标记未读
    //   StarredStream → true=添加收藏, false=取消收藏

    // 步骤2: 查询所有目标条目（用于状态判断和集成推送）
    entries, _ := h.store.NewEntryQueryBuilder(userID).WithEntryIDs(itemIDs...).GetEntries()

    // 步骤3: 按当前状态分类，分批处理
    var readEntryIDs, unreadEntryIDs, starredEntryIDs, unstarredEntryIDs []int64
    for _, entry := range entries {
        if read, exists := tags[ReadStream]; exists {
            if read && entry.Status == Unread {
                readEntryIDs = append(readEntryIDs, entry.ID)
            } else if !read && entry.Status == Read {
                unreadEntryIDs = append(unreadEntryIDs, entry.ID)
            }
        }
        if starred, exists := tags[StarredStream]; exists {
            if starred && !entry.Starred {
                starredEntryIDs = append(starredEntryIDs, entry.ID)
                entries[n] = entry  // 仅保留新收藏的，用于集成推送
                n++
            } else if !starred && entry.Starred {
                unstarredEntryIDs = append(unstarredEntryIDs, entry.ID)
            }
        }
    }
    entries = entries[:n]

    // 步骤4: 分批调用 Storage（最多4次批量更新）
    if len(readEntryIDs) > 0 {
        h.store.SetEntriesStatus(userID, readEntryIDs, model.EntryStatusRead)
    }
    if len(unreadEntryIDs) > 0 {
        h.store.SetEntriesStatus(userID, unreadEntryIDs, model.EntryStatusUnread)
    }
    if len(unstarredEntryIDs) > 0 {
        h.store.SetEntriesStarredState(userID, unstarredEntryIDs, false)
    }
    if len(starredEntryIDs) > 0 {
        h.store.SetEntriesStarredState(userID, starredEntryIDs, true)
    }

    // 步骤5: 新收藏的条目异步推送到第三方集成
    if len(entries) > 0 {
        settings, _ := h.store.Integration(userID)
        for _, entry := range entries {
            go integration.SendEntry(entry, settings)
        }
    }
}
```

**StreamType 解析**: `internal/googlereader/stream.go:73-107`

```
user/123/state/com.google/read         → ReadStream
user/123/state/com.google/starred      → StarredStream
user/123/state/com.google/reading-list → ReadingListStream
user/123/state/com.google/kept-unread  → KeptUnreadStream
user/123/label/MyFolder                 → LabelStream (Category)
feed/456                                → FeedStream
```

### 9.2 Mark-all-as-read 接口

**文件**: `internal/googlereader/handler.go:1155-1232`

```go
switch stream.Type {
case FeedStream:
    h.store.MarkFeedAsRead(userID, feedID, before)
case LabelStream:
    h.store.MarkCategoryAsRead(userID, category.ID, before)
case ReadingListStream:
    h.store.MarkAllAsReadBeforeDate(userID, before)
}
```

### 9.3 响应中的状态映射

在返回条目内容时 (`streamItemContentsHandler`, `handler.go:674-692`)，将内部状态翻译为 Google Reader 标签：

```go
for i, entry := range entries {
    categories := make([]string, 0, 4)
    categories = append(categories, userReadingList)  // 始终包含 reading-list
    if entry.Feed.Category.Title != "" {
        categories = append(categories, labelPrefix+entry.Feed.Category.Title)
    }
    if entry.Status == model.EntryStatusRead {
        categories = append(categories, userRead)  // 已读标签
    }
    if entry.Starred {
        categories = append(categories, userStarred)  // 收藏标签
    }
}
```

---

## 十、多入口同步实现：架构总结

### 10.1 同步策略：共享存储层 + 无缓存设计

```
┌─────────────────────────────────────────────────────────────┐
│                      HTTP Server                            │
│  internal/http/server/routes.go                             │
└──────────────┬──────────────┬──────────────┬────────────────┘
               │              │              │
    ┌──────────▼───┐  ┌───────▼─────┐  ┌─────▼────────┐  ┌───────▼────────┐
    │   Web UI     │  │ REST API v1 │  │  Fever API   │  │ Google Reader  │
    │ internal/ui/ │  │ internal/api│  │internal/fever│  │internal/google-│
    └──────────┬───┘  └───────┬─────┘  └─────┬────────┘  │reader/         │
               │              │              │            └───────┬────────┘
               └──────────────┴──────┬───────┴────────────────────┘
                                     │
                          ┌──────────▼──────────┐
                          │   Storage 层        │
                          │ internal/storage/   │
                          │  entry.go           │
                          │  - SetEntriesStatus │
                          │  - ToggleStarred    │
                          │  - MarkFeedAsRead   │
                          │  - ...              │
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │   PostgreSQL DB      │
                          │   entries 表        │
                          │   status 字段       │
                          │   starred 字段      │
                          └─────────────────────┘
```

**核心原理**：
1. **所有入口共享同一个 `*storage.Storage` 实例** → 操作同一张 `entries` 表
2. **不使用内存缓存** → 避免缓存失效/不一致问题
3. **每次读取即时查询计数** → 永远获取最新值

### 10.2 状态写入函数对照

| 操作 | Web UI | REST API v1 | Fever API | Google Reader | Storage 函数 |
|------|--------|------------|-----------|--------------|-------------|
| 单条改状态 | (页面查看时自动) | `PUT /v1/entries/{entryID}` | `mark=item&as=read/unread` | `edit-tag` (add/remove read) | `SetEntriesStatus` |
| 批量改状态 | `POST /entry/status` | `PUT /v1/entries` | 不支持 | `edit-tag` (批量 item IDs) | `SetEntriesStatus` / `SetEntriesStatusAndCountVisible` |
| 切换收藏 | `POST /entry/star/{id}` | `PUT /v1/entries/{id}/bookmark` | `mark=item&as=saved/unsaved` | `edit-tag` (add/remove starred) | `ToggleStarred` / `SetEntriesStarredState` |
| 用户全部已读 | `POST /mark-all-as-read` (仅全局可见) | `PUT /v1/users/{id}/mark-all-as-read` (全部) | `mark=group&id=0` | `mark-all-as-read` (reading-list) | `MarkGloballyVisibleFeedsAsRead` / `MarkAllAsRead` |
| 单 Feed 已读 | `POST /feed/{id}/mark-all-as-read` | `PUT /v1/feeds/{id}/mark-all-as-read` | `mark=feed&id=..` | `mark-all-as-read` (feed stream) | `MarkFeedAsRead` |
| 分类已读 | `POST /category/{id}/mark-all-as-read` | `PUT /v1/categories/{id}/mark-all-as-read` | `mark=group&id=..` | `mark-all-as-read` (label stream) | `MarkCategoryAsRead` |

### 10.3 未读计数的一致性保证

| 场景 | 计数获取方式 | 保证一致性的机制 |
|------|------------|----------------|
| Web 页面导航栏 | `GetNavMetadata()` | 每次页面渲染前实时查询，放在状态写入后最后一步执行 (`entry_unread.go:91-93`) |
| Web 前端列表页 | `SetEntriesStatusAndCountVisible()` 返回值 | SQL CTE 在更新的同一事务中计算变化量 |
| Web Feed 侧栏 | `fetchFeedCounter()` | `FeedsWithCounters()` 每次获取 Feed 列表时查询 |
| API 列表响应 | 调用方自行请求 `/v1/feeds/counters` | 实时 `GROUP BY feed_id, status` |
| Fever 同步 | `handleUnreadItems()` → 全部未读 ID 列表 | 每次请求时 `WHERE status='unread'` |
| Google Reader 同步 | `streamItemIDsHandler` → 带排除 filter | `WithStatuses(Unread)` 过滤 |

### 10.4 Changed_at 时间戳

所有状态修改（`SetEntriesStatus`、`ToggleStarred`、`SetEntriesStarredState`、`MarkFeedAsRead` 等）都会同步更新 `changed_at = now()` 字段。此字段用于：

1. **增量同步**：API 的 `changed_after` / `changed_before` 参数过滤
2. **Google Reader 响应**：`Updated: entry.ChangedAt.Unix()`
3. **调试与追踪**：记录状态最后修改时间

---

## 十一、特殊场景处理

### 11.1 查看页面时自动标记已读

**逻辑**: `model/entry.go:61-74` → `ShouldMarkAsReadOnView()`

```go
func (e *Entry) ShouldMarkAsReadOnView(user *User) bool {
    if e.Status != EntryStatusUnread {
        return false  // 已是已读，无需再标记
    }
    if user.MarkReadOnMediaPlayerCompletion && e.Enclosures.ContainsAudioOrVideo() {
        return false  // 有音视频附件，等播放完成再标记（由 enclosure handler 处理）
    }
    return user.MarkReadOnView  // 用户设置
}
```

**调用位置**：各 entry 展示 handler（`entry_unread.go`, `entry_starred.go` 等）。

### 11.2 归档时的保护机制

**函数**: `ArchiveEntries` (`storage/entry.go:362-404`) 和 `FlushHistory` (`storage/entry.go:492-508`)

删除已读条目时，SQL 条件包含 `starred is false AND share_code=''`，保护：
- **已收藏的条目** 不会被清理
- **已分享的条目**（有 share_code）不会被清理

### 11.3 第三方集成推送触发

触发第三方集成（Wallabag / Pinboard / Notion / Readwise 等）的路径：

| 入口 | 触发函数 | 时机 |
|------|---------|------|
| Web UI `POST /entry/save/{id}` | `saveEntry` | 用户手动点击保存 |
| API `POST /v1/entries/{id}/save` | `saveEntryHandler` | 客户端调用保存端点 |
| Fever `mark=item&as=saved` | `handleWriteItems` | 收藏时自动触发 |
| Google Reader `edit-tag` (add starred) | `editTagHandler` | 收藏标签时自动触发 |

均使用 `go integration.SendEntry(entry, settings)` 异步推送（不阻塞 HTTP 请求）。

---

## 十二、并发更新控制：乐观锁 / 悲观锁与冲突解决

### 12.1 总体设计：以 PostgreSQL MVCC + 唯一约束为核心，不使用显式版本号乐观锁

Miniflux **没有采用传统的"version/revision"字段 + `WHERE version = N` 的乐观锁模式**，也没有对条目状态更新做显式 `FOR UPDATE` 行锁。其并发一致性依赖以下几层机制的组合：

#### 12.1.1 基础层：PostgreSQL MVCC（多版本并发控制）

所有状态变更 SQL 都是单条 `UPDATE`，在 PostgreSQL 的 MVCC 下天然具有以下特性：

- 两个并发事务同时 `UPDATE entries SET status='read' WHERE id=123 AND user_id=456`：后提交者不会产生冲突，只是 `RowsAffected` 可能为 0（若前者已修改）。
- 写-写冲突不会产生"丢失更新"，但可能产生"最后写入者胜"（Last-Writer-Wins）语义。

#### 12.1.2 `changed_at = now()` 的副作用

所有状态更新 SQL 都包含 `changed_at=now()`，这意味着：

- **changed_at 不是乐观锁条件**：SQL 中**没有** `WHERE changed_at = $old_changed_at` 这样的比较子句，因此它不参与冲突检测。
- changed_at 仅作为"状态变更时间戳"用于增量同步过滤（API 的 `changed_after` / `changed_before` 参数，见 `api/entry_handlers.go:589-594`）。

```go
// api/entry_handlers.go:589-594
if beforeChangedTimestamp := request.QueryInt64Param(r, "changed_before", 0); beforeChangedTimestamp > 0 {
    builder = builder.BeforeChangedDate(time.Unix(beforeChangedTimestamp, 0))
}
if afterChangedTimestamp := request.QueryInt64Param(r, "changed_after", 0); afterChangedTimestamp > 0 {
    builder = builder.AfterChangedDate(time.Unix(afterChangedTimestamp, 0))
}
```

### 12.2 唯一约束作为幂等性保障：`entries_feed_id_hash_key`

**Schema 定义**（`database/migrations.go:74-91`）：

```sql
CREATE TABLE entries (
    ...
    PRIMARY KEY (id),
    UNIQUE (feed_id, hash),   -- entries_feed_id_hash_key
    ...
);
```

在 `RefreshFeedEntries`（爬虫刷新）中，该唯一约束与显式事务配合实现幂等导入：

**代码**（`storage/entry.go:314-360`）：

```go
func (s *Storage) RefreshFeedEntries(userID, feedID int64, entries model.Entries, ...) (...) {
    for _, entry := range entries {
        tx, err := s.db.Begin()       // 每条 entry 一个独立事务

        entryExists, err := s.entryExists(tx, entry)   // SELECT ... WHERE feed_id=$1 AND hash=$2
        if entryExists {
            if updateExistingEntries {
                err = s.updateEntry(tx, entry)         // UPDATE ... WHERE user_id=$9 AND feed_id=$10 AND hash=$11
            }
        } else {
            err = s.createEntry(tx, entry)             // INSERT ...
        }
        tx.Commit()
    }
}
```

**并发竞态分析**：

若两个爬虫 worker 同时刷新同一个 feed 的同一条 entry：

| 时序 | Worker A | Worker B |
|------|---------|---------|
| T1 | `BEGIN` |  |
| T2 | `entryExists()` → false |  |
| T3 |  | `BEGIN` |
| T4 |  | `entryExists()` → false |
| T5 | `createEntry()` → INSERT 成功 |  |
| T6 | `COMMIT` |  |
| T7 |  | `createEntry()` → 违反 UNIQUE(feed_id, hash)，报错 |
| T8 |  | `ROLLBACK` |

此时 Worker B 的 `createEntry` 将返回 `pq: duplicate key value violates unique constraint "entries_feed_id_hash_key"`，被上层捕获为错误，不影响数据一致性。

### 12.3 `createEntry` 中的原子防墓碑检查：`WHERE NOT EXISTS` 子查询

**代码**（`storage/entry.go:81-161`，注释行 83-85）：

```go
func (s *Storage) createEntry(tx *sql.Tx, entry *model.Entry) error {
    query := `
        INSERT INTO entries (...)
        SELECT $1, $2, ...
        WHERE NOT EXISTS (
            SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
        )
        RETURNING id, status, created_at, changed_at
    `
    err := tx.QueryRow(...).Scan(...)
    if errors.Is(err, sql.ErrNoRows) {
        return ErrEntryTombstoned   // 墓碑存在，拒绝插入
    }
}
```

**注释原文**（第 83-85 行）：

> The WHERE NOT EXISTS guard makes the tombstone check atomic with the insert, so a concurrent archive committing between an earlier existence check and this statement cannot bring a deleted entry back as unread.

这是一个**针对"归档清理 vs 爬虫重新导入"竞态**的专用乐观机制：

1. 归档操作 `ArchiveEntries` 删除 entries 并写入 `entry_tombstones`（带 `ON CONFLICT DO NOTHING`）
2. 如果没有 `WHERE NOT EXISTS`，时序如下会出错：
   - T1: `entryExists()` 查询发现 entry 已被删除（返回 false）
   - T2: 归档事务提交，写入 tombstone
   - T3: `createEntry()` 直接 INSERT → **已被删除的 entry 复活为 unread**
3. 加上 `WHERE NOT EXISTS` 后，INSERT 与 tombstone 检查在**同一条 SQL 的同一快照**中执行，由 PostgreSQL 保证原子性。

### 12.4 `ArchiveEntries` 中的悲观锁：`FOR UPDATE SKIP LOCKED`

**代码**（`storage/entry.go:362-404`）：

```go
func (s *Storage) ArchiveEntries(status string, interval time.Duration, limit int) (int64, error) {
    query := `
        WITH to_delete AS (
            SELECT id, feed_id, hash
            FROM entries
            WHERE ...
            ORDER BY created_at ASC
            FOR UPDATE SKIP LOCKED   -- ★ 悲观锁：跳过已被锁住的行
            LIMIT $3
        ), deleted AS (
            DELETE FROM entries USING to_delete WHERE entries.id = to_delete.id
            RETURNING entries.feed_id, entries.hash
        )
        INSERT INTO entry_tombstones (feed_id, hash)
        SELECT feed_id, hash FROM deleted WHERE hash <> ''
        ON CONFLICT (feed_id, hash) DO NOTHING
    `
}
```

**设计意图**：

- `FOR UPDATE` 对选中行加**排他行锁**，防止同一行被两个并发归档 worker 同时删除
- `SKIP LOCKED` 跳过已被其他事务锁定的行，避免阻塞等待 → 这是典型的"任务队列"模式
- 该锁**仅在归档/清理路径使用**，不影响正常的状态更新（SetEntriesStatus/ToggleStarred 等不走这条路径）

### 12.5 入口 Pagination 的事务一致性：`entryPaginationBuilder`

**代码**（`storage/entry_pagination_builder.go:110-142`）：

```go
func (e *entryPaginationBuilder) Entries() (*model.Entry, *model.Entry, error) {
    tx, err := e.db.Begin()

    prevID, nextID, err := e.getPrevNextID(tx)   // CTE: lag()/lead() 窗口函数
    prevEntry, err := e.getEntry(tx, prevID)
    nextEntry, err := e.getEntry(tx, nextID)

    tx.Commit()   // 三个查询在同一事务中
}
```

该显式事务保证：prev/next ID 计算和条目详情读取在**同一数据库快照**下执行，避免"翻页时条目状态改变导致上下条丢失或重复"。

### 12.6 并发冲突处理总结

| 场景 | 机制 | 类型 | 位置 |
|------|------|------|------|
| 爬虫并发导入同一条目 | `UNIQUE(feed_id, hash)` + 单条事务 | 唯一约束（事后检测） | `storage/entry.go:314-360` |
| 归档 vs 爬虫复活竞态 | `INSERT ... WHERE NOT EXISTS (tombstone)` | 原子子查询（事前防护） | `storage/entry.go:86-119` |
| 并发归档清理竞争 | `FOR UPDATE SKIP LOCKED` | 悲观行锁 + 跳过 | `storage/entry.go:368-389` |
| 翻页时状态漂移 | 显式事务包裹 prev/next 查询 + 详情读取 | 事务快照一致性 | `storage/entry_pagination_builder.go:110-142` |
| 两个客户端同时改状态 | PostgreSQL MVCC + Last-Writer-Wins | 数据库原生 | 所有 UPDATE 语句 |
| 状态 vs 收藏交叉更新 | 无显式冲突检测（最后写入者胜） | 无 | - |

**注意缺失点**：不存在 `WHERE changed_at = ?` 或 `WHERE status = ? AND starred = ?` 的 CAS（Compare-And-Swap）更新。若客户端 A 将条目 123 设为 read，同时客户端 B 设为 starred，两条 UPDATE 都能成功，最终状态是 `status='read', starred=true`，两者互不覆盖——因为 UPDATE 的 SET 子句只修改各自负责的字段，这是 PostgreSQL 的列级更新特性，天然安全。

---

## 十三、未读计数：无缓存设计下的 Fast-Count 索引与一致性保障

### 13.1 核心设计决策：无计数器表 / 无 Redis / 无 Materialized View

通过代码搜索确认：
- 没有 `CREATE TRIGGER` / `CREATE RULE`（自动维护计数的触发器）
- 没有独立的计数器表（如 `feed_counters` / `category_counters`）
- 没有 Materialized View（物化视图预计算）
- 没有应用层内存缓存（`sync.Map` / `lru` / Redis 等）

所有未读计数均为**实时 SQL 查询**，这是 Miniflux 计数一致性的根本保证。

### 13.2 Fast-Count 支撑：覆盖式 B-Tree 索引体系

为了让实时 count 足够快，Miniflux 通过**一系列精心设计的前缀 B-Tree 索引**，使大部分计数查询可以走**Index Only Scan**（仅扫描索引，不回表）。

**索引演进（migrations.go 中按时间顺序）**：

| 版本 | 索引 | 用途 |
|------|------|------|
| v0 | `entries_feed_idx (feed_id)` | 最基础 feed 查询（后被删除） |
| v? | `entries_user_status_idx (user_id, status)` | 全局未读计数（后被删除，被更宽索引覆盖） |
| v? | `entries_user_id_status_starred_idx (user_id, status, starred)` | 收藏筛选 + 计数 |
| v? | `entries_user_feed_idx (user_id, feed_id)` | Feed 内查询 |
| v? | `entries_id_user_status_idx (id, user_id, status)` | 按 ID 定位并顺带检查状态 |
| v? | `entries_feed_id_status_hash_idx (feed_id, status, hash)` | Feed 内状态筛选 + hash 去重 |
| v? | `entries_user_status_feed_idx (user_id, status, feed_id)` | Feed 粒度计数（核心） |
| v? | `entries_user_status_changed_idx (user_id, status, changed_at)` | 增量同步过滤 |
| v? | `entries_user_status_published_idx (user_id, status, published_at)` | 按发布时间排序+筛选 |
| v? | `entries_user_status_created_idx (user_id, status, created_at)` | 按创建时间排序+筛选 |
| v? | `entries_user_status_changed_published_idx (user_id, status, changed_at, published_at)` | 增量排序 |

**优化证据**（`migrations.go:1522-1535`）——后期迁移显式删除冗余索引：

```go
// entries_feed_idx is redundant: the unique constraint
// entries_feed_id_hash_key(feed_id, hash) and the explicit
// entries_feed_id_status_hash_idx(feed_id, status, hash) both
// cover feed_id-leading lookups, including FK cascade deletes.
//
// entries_user_status_idx is redundant: five three-column indexes
// share the same (user_id, status) prefix and serve every query
// that the two-column index could.
_, err = tx.Exec(`
    DROP INDEX IF EXISTS entries_feed_idx;
    DROP INDEX IF EXISTS entries_user_status_idx;
`)
```

### 13.3 三类计数查询与索引命中分析

#### 13.3.1 全局未读计数：`GetNavMetadata()`

**SQL**（`storage/nav_metadata.go:20-97`）：

```sql
SELECT
    (SELECT count(*)
       FROM entries e
       JOIN feeds f ON f.id = e.feed_id
       JOIN categories c ON c.id = f.category_id
      WHERE e.user_id = $1
        AND e.status = 'unread'
        AND f.hide_globally IS FALSE
        AND c.hide_globally IS FALSE
    ) AS count_unread,
    ...
```

**索引利用**：
- `e.user_id = $1 AND e.status = 'unread'` → 命中 `entries_user_status_*` 系列索引（都以 `(user_id, status)` 为前缀）
- 但因为还要 JOIN `feeds` 和 `categories` 过滤 `hide_globally`，**无法走 Index Only Scan**，必须回表取 `feed_id` 再做 JOIN
- 这是最慢的计数查询，但只在 Web UI 页面渲染时执行（每个页面一次）

#### 13.3.2 Feed 粒度读/未读计数：`fetchFeedCounter()`

**SQL**（`storage/feed_query_builder.go:302-350`）：

```sql
SELECT e.feed_id, e.status, count(*)
FROM entries e
[INNER JOIN feeds f ON f.id=e.feed_id]   -- 仅当按 category 过滤时
WHERE e.user_id = $1 AND e.status IN ($2, $3)
GROUP BY e.feed_id, e.status
```

**索引利用**：
- `entries_user_status_feed_idx (user_id, status, feed_id)` 是 **完美覆盖索引**
- PostgreSQL 可直接对该索引做 Index Only Scan + Group Aggregate，无需回表
- 按 category 过滤时需 JOIN feeds，但索引仍然先把 `(user_id, status)` 的行集大幅缩小

#### 13.3.3 Category 粒度未读计数：`CategoriesWithFeedCount()`

**SQL**（`storage/category.go:112-170`）：

```sql
SELECT
    c.id, c.user_id, c.title, c.hide_globally,
    coalesce(fc.feed_count, 0),
    coalesce(uc.unread_count, 0)
FROM categories c
LEFT JOIN (
    SELECT category_id, count(*) AS feed_count FROM feeds ... GROUP BY category_id
) fc ON fc.category_id = c.id
LEFT JOIN (
    SELECT f.category_id, count(*) AS unread_count
    FROM entries e INNER JOIN feeds f ON f.id = e.feed_id
    WHERE e.user_id = $2 AND e.status = $1   -- $1 = 'unread'
    GROUP BY f.category_id
) uc ON uc.category_id = c.id
```

**索引利用**：
- 子查询 `uc` 的 `WHERE e.user_id = $2 AND e.status = $1` → 命中 `entries_user_status_feed_idx`
- 但需要 `GROUP BY f.category_id`，必须回表 JOIN feeds，无法完全覆盖
- 使用 `LEFT JOIN + coalesce` 保证"无未读条目时显示 0"而非 NULL

### 13.4 "对账机制"的本质：无缓存 = 对账 = 查询

由于完全不使用缓存，Miniflux **不存在"缓存值 vs 真实值"的对账需求**。每次获取计数就是一次从数据库原始数据的实时聚合。

唯一接近"对账"的是：

1. **Web UI 中的时序控制**（`ui/entry_unread.go:85-93`）：先执行状态写入，再执行 `GetNavMetadata()` 读取计数。代码注释明确说明：

```go
// Fetching the counters here avoids being off by one.
navMetadata, _ := h.store.GetNavMetadata(user.ID)
view.Set("countUnread", navMetadata.CountUnread)
```

这不是对账，而是**同一请求内的写入后读取（Read-Your-Writes）顺序保证**。

2. **`SetEntriesStatusAndCountVisible` 的 CTE 原子性**（`storage/entry.go:425-449`）：

```sql
WITH updated AS (
    UPDATE entries SET status=$1, changed_at=now()
    WHERE user_id=$2 AND id=ANY($3)
    RETURNING feed_id
)
SELECT count(*) FROM updated u
    JOIN feeds f ON f.id = u.feed_id
    JOIN categories c ON c.id = f.category_id
WHERE NOT f.hide_globally AND NOT c.hide_globally
```

UPDATE 和 COUNT 在**同一 SQL 语句（隐式事务）**中执行，CTE 的 `updated` 行集就是刚刚被修改的行，不存在时间差，天然一致。

### 13.5 `CountAllEntries`：管理员总览计数

**代码**（`storage/entry.go:23-48`）：

```go
func (s *Storage) CountAllEntries() (map[string]int64, error) {
    rows, err := s.db.Query(`SELECT status, count(*) FROM entries GROUP BY status`)
    // ...
}
```

这是管理员仪表盘使用的全库计数，不带 user_id 过滤，走 `entries` 表的全表扫描或主键索引。

---

## 十四、批量标记已读的事务边界分析

### 14.1 事务边界总览：两种模式

Miniflux 中批量标记已读的 Storage 函数可分为两大类：

| 模式 | 代表函数 | 事务类型 | 边界 |
|------|---------|---------|------|
| **模式 A：单条 SQL 隐式事务** | `SetEntriesStatus`、`MarkAllAsRead`、`MarkFeedAsRead` 等 | 自动提交（auto-commit） | 整个 UPDATE 是一个原子事务 |
| **模式 B：显式 BEGIN/COMMIT** | `RefreshFeedEntries`、`InsertEntryForFeed`、`entryPaginationBuilder.Entries()` | 手动控制 | 多条 SQL 在同一事务中 |

### 14.2 模式 A：状态标记函数——全部使用单条 SQL（隐式事务）

所有面向用户入口的状态更新函数**都不使用显式 `BEGIN/COMMIT`**，而是依赖 PostgreSQL 的"单条 SQL = 一个原子事务"：

#### 14.2.1 `SetEntriesStatus` — 批量按 ID 更新

```go
// storage/entry.go:407-423
func (s *Storage) SetEntriesStatus(userID int64, entryIDs []int64, status string) error {
    query := `UPDATE entries SET status=$1, changed_at=now() WHERE user_id=$2 AND id=ANY($3)`
    _, err := s.db.Exec(query, status, userID, pq.Array(entryIDs))
    return err
}
```

- **事务边界**：单个 `Exec()` 调用 = 一个隐式事务
- **原子性**：要么所有 `id=ANY($3)` 的行都被更新，要么都不更新
- **部分失败**：PostgreSQL 中 UPDATE 是原子的，不存在"更新了一部分"的情况
- **调用方**：API v1 `PUT /v1/entries`、Fever `mark=item`、Google Reader `edit-tag`

#### 14.2.2 `SetEntriesStatusAndCountVisible` — 批量更新 + 返回可见计数

```go
// storage/entry.go:425-449
func (s *Storage) SetEntriesStatusAndCountVisible(...) (int, error) {
    query := `
        WITH updated AS (
            UPDATE entries SET status=$1, changed_at=now()
            WHERE user_id=$2 AND id=ANY($3)
            RETURNING feed_id
        )
        SELECT count(*) FROM updated u JOIN feeds f ... JOIN categories c ...
        WHERE NOT f.hide_globally AND NOT c.hide_globally
    `
    var visible int
    err := s.db.QueryRow(query, ...).Scan(&visible)
    return visible, err
}
```

- **事务边界**：CTE + SELECT 在**单条 SQL**中，是一个隐式事务
- **原子性**：UPDATE 和 COUNT 之间不存在时间窗口，不可能出现"计数少算/多算已更新的行"
- **调用方**：Web UI `POST /entry/status`（`ui/entry_update_status.go:16-35`）

#### 14.2.3 `SetEntriesStarredState` — 批量设置收藏

```go
// storage/entry.go:451-469
func (s *Storage) SetEntriesStarredState(userID int64, entryIDs []int64, starred bool) error {
    query := `UPDATE entries SET starred=$1, changed_at=now() WHERE user_id=$2 AND id=ANY($3)`
    result, _ := s.db.Exec(query, starred, userID, pq.Array(entryIDs))
    count, _ := result.RowsAffected()
    if count == 0 {
        return errors.New(`store: nothing has been updated`)
    }
    return nil
}
```

- **事务边界**：单条 `Exec()`
- **特殊点**：检查 `RowsAffected == 0` 返回错误——这是幂等性检查（所有 ID 都不存在或已经是目标状态）
- **调用方**：Google Reader `edit-tag`（`googlereader/handler.go:283-287`）

#### 14.2.4 `ToggleStarred` — 单条切换收藏

```go
// storage/entry.go:472-489
func (s *Storage) ToggleStarred(userID int64, entryID int64) error {
    query := `UPDATE entries SET starred = NOT starred, changed_at=now() WHERE user_id=$1 AND id=$2`
    result, _ := s.db.Exec(query, userID, entryID)
    count, _ := result.RowsAffected()
    if count == 0 {
        return errors.New(`store: unable to toggle bookmark for this entry`)
    }
    return nil
}
```

- **事务边界**：单条 `Exec()`
- **原子切换**：`starred = NOT starred` 在数据库层完成，不需要先 SELECT 再 UPDATE，避免了"读-改-写"竞态
- **调用方**：Web UI `POST /entry/star/{id}`、API v1 `PUT /v1/entries/{id}/bookmark`、Fever `mark=item&as=saved/unsaved`

#### 14.2.5 范围型批量标记已读

```go
// MarkAllAsRead — storage/entry.go:510-525
UPDATE entries SET status=$1, changed_at=now() WHERE user_id=$2 AND status=$3

// MarkAllAsReadBeforeDate — storage/entry.go:527-549
UPDATE entries SET status=$1, changed_at=now()
WHERE user_id=$2 AND status=$3 AND published_at < $4

// MarkGloballyVisibleFeedsAsRead — storage/entry.go:551-579
UPDATE entries SET status=$1, changed_at=now()
FROM feeds
WHERE entries.feed_id = feeds.id
  AND entries.user_id=$2 AND entries.status=$3
  AND feeds.hide_globally=$4

// MarkFeedAsRead — storage/entry.go:581-606
UPDATE entries SET status=$1, changed_at=now()
WHERE user_id=$2 AND feed_id=$3 AND status=$4 AND published_at < $5

// MarkCategoryAsRead — storage/entry.go:608-643
UPDATE entries SET status=$1, changed_at=now()
FROM feeds
WHERE feed_id=feeds.id
  AND feeds.user_id=$2
  AND status=$3 AND published_at < $4 AND feeds.category_id=$5
```

全部都是**单条 UPDATE = 一个隐式事务**。特点：
- `MarkCategoryAsRead` 和 `MarkGloballyVisibleFeedsAsRead` 使用 `FROM feeds` 做跨表 UPDATE，依然是单条 SQL
- 所有函数都只检查错误，不做重试（PostgreSQL 单条 UPDATE 不会因为并发冲突而回滚，最多 `RowsAffected` 较少）

### 14.3 模式 B：爬虫导入路径——显式事务，每条 entry 一个事务

#### 14.3.1 `RefreshFeedEntries`

```go
// storage/entry.go:314-360
func (s *Storage) RefreshFeedEntries(userID, feedID int64, entries model.Entries, ...) (...) {
    for _, entry := range entries {   // ★ 外层 for 循环
        tx, err := s.db.Begin()       // 每条 entry 一个 BEGIN
        entryExists, err := s.entryExists(tx, entry)
        if entryExists {
            err = s.updateEntry(tx, entry)    // UPDATE
        } else {
            err = s.createEntry(tx, entry)    // INSERT + INSERT enclosures
        }
        if err != nil {
            tx.Rollback()
            return nil, err
        }
        tx.Commit()
    }
}
```

**事务边界设计决策分析**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **当前方案：每条 entry 一个事务** | 失败时只回滚单条，不影响已成功的；单事务短小减少锁持有时间 | 大量 entry 时事务开销大；无整体原子性（前几条成功后中间失败时部分导入） |
| 备选 A：所有 entry 一个大事务 | 原子性——要么全导入要么全不回滚 | 大事务持有锁时间长，可能阻塞其他操作；feed 有 1000 条 entry 时事务日志巨大 |
| 备选 B：每 N 条一个批量事务 | 平衡开销和原子性 | 实现复杂，部分失败语义不清晰 |

Miniflux 选择了**方案 1（每条 entry 独立事务）**，因为 RSS 刷新是**幂等操作**，下次刷新会重试失败的条目。

#### 14.3.2 `InsertEntryForFeed` — 单条导入

```go
// storage/entry.go:245-276
func (s *Storage) InsertEntryForFeed(userID, feedID int64, entry *model.Entry) (bool, error) {
    tx, err := s.db.Begin()
    defer tx.Rollback()

    entryID, err := s.getEntryIDByHash(tx, entry.FeedID, entry.Hash)
    alreadyExistingEntry := entryID > 0
    if alreadyExistingEntry {
        entry.ID = entryID
    } else {
        s.createEntry(tx, entry)
    }
    tx.Commit()
    return !alreadyExistingEntry, nil
}
```

- 单条 entry 的 `SELECT (getEntryIDByHash) + [INSERT (createEntry)]` 需要显式事务包裹，保证 SELECT 和 INSERT 之间不会有并发插入（否则 `createEntry` 的 UNIQUE 约束仍会兜底报错，但 `defer tx.Rollback()` 确保不会泄漏事务）

### 14.4 为什么状态更新不走显式事务？

与爬虫导入不同，所有面向用户的状态更新（SetEntriesStatus / MarkFeedAsRead / ToggleStarred 等）都刻意避免显式事务：

1. **单 SQL 足够**：UPDATE 本身就是原子的，不需要 BEGIN/COMMIT 包装
2. **减少往返**：显式事务需要 3 次网络往返（BEGIN → UPDATE → COMMIT），单 SQL 只需 1 次
3. **锁持有时间最短**：单条 UPDATE 的锁只在语句执行期间持有，不跨网络往返
4. **避免长事务**：HTTP handler 中若 BEGIN 后在计算/日志处阻塞，会导致事务长时间不提交

### 14.5 异常路径：Rollback 触发条件

代码中显式事务的 Rollback 触发场景：

| 函数 | Rollback 时机 |
|------|--------------|
| `RefreshFeedEntries` | `entryExists` 报错、`updateEntry` 报错、`createEntry` 报错、`tx.Commit()` 之前任何错误 |
| `InsertEntryForFeed` | 通过 `defer tx.Rollback()` 兜底——只要没走到 Commit，函数返回时自动回滚 |
| `entryPaginationBuilder.Entries` | `getPrevNextID` 报错、`getEntry` 报错 |
| 其他事务（`user.go` / `icon.go` / `category.go` / `feed.go`） | 任何中间步骤出错 |

### 14.6 批量事务边界与未读计数的一致性

由于所有状态更新 SQL：
1. 要么是**单条隐式事务**（UPDATE 自包含）
2. 要么 UPDATE + COUNT 是**同一条 SQL 的 CTE**（SetEntriesStatusAndCountVisible）

不存在"UPDATE 已提交但 COUNT 还没跟上"的不一致窗口。未读计数查询在 UPDATE 之后执行时，必然能看到最新状态——这是 PostgreSQL 的事务隔离（Read Committed）保证的。

---

## 十五、Entry 状态迁移历史的审计机制

### 15.1 核心结论：无状态历史审计表，只保留当前状态

通过全代码库搜索确认：
- 不存在 `entry_status_log`、`entry_status_history`、`entry_audit` 等历史表
- 没有 `CREATE TABLE ... LOG` / `CREATE TABLE ... HISTORY` 的数据库迁移
- 没有触发器（`CREATE TRIGGER`）用于记录状态变更
- 没有应用层的历史记录代码（如写入历史表的 hook）

Miniflux 的审计设计是**极简主义**：仅保留条目当前状态，不记录状态迁移的时间线。

### 15.2 已废弃的 `removed` 状态

**Schema 定义**（`database/migrations.go:74`）：

```sql
CREATE TYPE entry_status as enum('unread', 'read', 'removed');
```

最初设计了三种状态，但 `removed` 状态在后期迁移中被废弃：

**迁移代码**（`database/migrations.go:1470-1498`）：

```go
_, err = tx.Exec(`
    CREATE TABLE entry_tombstones (
        feed_id bigint not null references feeds(id) on delete cascade,
        hash text not null check (hash <> ''),
        deleted_at timestamp with time zone not null default now(),
        primary key (feed_id, hash)
    );

    INSERT INTO entry_tombstones (feed_id, hash, deleted_at)
        SELECT feed_id, hash, changed_at
        FROM entries
        WHERE status = 'removed' AND hash <> ''
        ON CONFLICT (feed_id, hash) DO NOTHING;

    DELETE FROM entries WHERE status = 'removed';
`)
```

**原因**：`removed` 状态本质上是"软删除"，但会污染 entries 表的查询（所有查询都需要 `WHERE status != 'removed'`），且没有实际的恢复流程。改为 `entry_tombstones` 表后：
- 已删除条目的 hash 被记录，防止爬虫重新导入时"复活"
- entries 表只包含 `unread` 和 `read` 两种状态，查询更简单
- 但 `entry_status` enum 类型仍然保留 `removed` 值（PostgreSQL 不能安全移除 enum 值）

### 15.3 仅有的"审计"信息：`changed_at` 时间戳

所有状态变更都会更新 `changed_at = now()`，但这只是**最后一次变更的时间戳**，不包含历史：

| 字段 | 含义 | 粒度 |
|------|------|------|
| `created_at` | 条目首次导入时间 | 一次性，创建后不变 |
| `published_at` | RSS 源中条目的发布时间 | 来自 RSS，不随状态变更 |
| `changed_at` | **最后一次**状态/收藏变更时间 | 每次 `SetEntriesStatus` / `ToggleStarred` 等都会更新 |

### 15.4 审计信息的使用场景

`changed_at` 唯一的"类审计"用途是**增量同步过滤**：

**代码**（`api/entry_handlers.go:564-612`）：

```go
func configureFilters(builder *storage.EntryQueryBuilder, r *http.Request) *storage.EntryQueryBuilder {
    // ...
    if beforeChangedTimestamp := request.QueryInt64Param(r, "changed_before", 0); beforeChangedTimestamp > 0 {
        builder = builder.BeforeChangedDate(time.Unix(beforeChangedTimestamp, 0))
    }
    if afterChangedTimestamp := request.QueryInt64Param(r, "changed_after", 0); afterChangedTimestamp > 0 {
        builder = builder.AfterChangedDate(time.Unix(afterChangedTimestamp, 0))
    }
    // ...
}
```

这允许 API 客户端"只拉取某个时间点之后有状态变更的条目"，但无法知道具体变更是从什么状态变成什么状态。

### 15.5 设计权衡分析

| 方案 | 优点 | 缺点 |
|------|------|------|
| **当前方案：无历史** | 实现简单、存储开销为 0、查询性能最优 | 无法审计"谁在什么时候把条目从 unread 变成 read 再变成 starred" |
| 历史表方案 | 完整审计、可回溯 | 写入放大（每次状态变更多一次 INSERT）、存储随时间增长、查询复杂 |
| 触发器方案 | 透明无侵入 | 数据库端逻辑、难以调试、迁移复杂 |

Miniflux 作为个人 RSS 阅读器，选择了**极简方案**——因为单用户场景下，状态变更的审计需求极低，用户几乎不会关心"三个月前我哪天把这篇文章标为已读"。

---

## 十六、跨设备同步：Last-Writer-Wins 策略与客户端拉取模型

### 16.1 核心设计：无服务器端合并逻辑，全靠数据库原生 LWW

Miniflux **没有实现显式的最后写赢逻辑**（如 `WHERE changed_at < $new_changed_at` 的 CAS 更新）。其跨设备同步一致性依赖以下几层机制的组合：

#### 16.1.1 数据库层：PostgreSQL 原生 Last-Writer-Wins

所有状态更新都是单条 SQL：

```sql
UPDATE entries SET status=$1, changed_at=now() WHERE user_id=$2 AND id=ANY($3)
```

当两个设备同时更新同一条目时：
- 两个 UPDATE 都会执行成功
- 后提交的 UPDATE 会覆盖先提交的 `status` 和 `changed_at`
- 没有冲突检测、没有合并、没有版本向量

**这是最朴素的 LWW**——完全依赖 PostgreSQL 的事务调度，"后到达者胜"。

#### 16.1.2 `changed_at` 不是乐观锁条件

关键细节：SQL 中 **没有** `WHERE changed_at = $old_changed_at` 或 `WHERE status = $old_status` 这样的比较条件。

如果设备 A 在 T1 将条目 123 设为 read，设备 B 在 T2（T2 > T1）将同一条目设为 unread：
- 最终状态是 `unread`（后写入者胜）
- `changed_at` 是 T2
- 设备 A 下次拉取时会看到 `changed_at = T2, status = 'unread'`，知道自己的变更被覆盖

但 Miniflux 不会告诉设备 A"你的变更被 B 覆盖了"——这需要设备端自行对比本地记录与服务器状态。

### 16.2 同步模型：客户端拉取（Pull）而非服务器推送（Push）

Miniflux 没有 WebSocket 或长连接推送。所有同步都是**客户端主动轮询**，分为两种模式：

#### 16.2.1 Fever API 同步模式：全量 ID 列表对比

**代码**（`fever/handler.go:331-370`）：

```go
// unread_item_ids — 返回所有未读条目的 ID，逗号分隔字符串
func (h *feverHandler) handleUnreadItems(w http.ResponseWriter, r *http.Request) {
    rawEntryIDs, _ := h.store.NewEntryQueryBuilder(userID).
        WithStatuses(model.EntryStatusUnread).
        GetEntryIDs()
    // 序列化为 "123,456,789" 字符串
}

// saved_item_ids — 返回所有收藏条目的 ID
func (h *feverHandler) handleSavedItems(w http.ResponseWriter, r *http.Request) {
    rawEntryIDs, _ := h.store.NewEntryQueryBuilder(userID).
        WithStarred(true).
        GetEntryIDs()
}
```

**客户端同步流程**：
1. 客户端缓存本地的 `unread_item_ids` 和 `saved_item_ids` 列表
2. 定期调用 `?unread_item_ids` 和 `?saved_item_ids` 获取服务器端全量列表
3. 客户端对比两个列表的差异：
   - 服务器有但本地没有 → 拉取条目详情 + 应用状态
   - 本地有但服务器没有 → 向服务器发送 `mark=item&as=read` / `mark=item&as=unsaved`
4. 条目详情通过 `?items&since_id=XXX` 分页拉取（每次 50 条）

**特点**：
- 协议简单，ID 列表传输量小
- 客户端负责冲突解决（发现差异时客户端决定如何处理）
- 服务器完全无状态

#### 16.2.2 Google Reader API 同步模式：带时间窗口的增量拉取

**代码**（`googlereader/handler.go:1005-1047` + `980-1002`）：

```go
func (h *greaderHandler) handleReadingListStreamHandler(w http.ResponseWriter, r *http.Request, rm requestModifiers) {
    builder := h.store.NewEntryQueryBuilder(rm.UserID).
        WithLimit(rm.Count).
        WithOffset(rm.Offset).
        WithSorting(model.DefaultSortingOrder, rm.SortDirection)

    // 排除已读条目（xt = exclude tag）
    for _, s := range rm.ExcludeTargets {
        switch s.Type {
        case ReadStream:
            builder = builder.WithStatuses(model.EntryStatusUnread)
        }
    }

    // 时间窗口过滤（ot = start time, st = stop time）
    if rm.StartTime > 0 {
        builder = builder.AfterPublishedDate(time.Unix(rm.StartTime, 0))
    }
    if rm.StopTime > 0 {
        builder = builder.BeforePublishedDate(time.Unix(rm.StopTime, 0))
    }
}
```

**状态回传中的时间戳**（`googlereader/handler.go:696-730`）：

```go
result.Items[i] = contentItem{
    // ...
    TimestampUsec: strconv.FormatInt(entry.Date.UnixMicro(), 10),    // 发布时间（微秒）
    CrawlTimeMsec: strconv.FormatInt(entry.CreatedAt.UnixMilli(), 10), // 导入时间（毫秒）
    Published:     entry.Date.Unix(),                              // 发布时间（秒）
    Updated:       entry.ChangedAt.Unix(),                         // ★ 状态最后变更时间
}
```

**客户端同步流程**：
1. 首次全量拉取：`/reader/api/0/stream/items/ids?s=user/-/state/com.google/reading-list&n=1000`
2. 记录最大的 `Updated` 时间戳
3. 增量拉取：带上 `ot=<last_updated_timestamp>` 只拉取发布时间在该时间之后的条目
4. 对于状态变更，客户端通过 `/reader/api/0/stream/items/contents` 拉取条目详情，检查 `Updated` 字段是否大于本地记录
5. 发现冲突时（本地 `Updated` < 服务器 `Updated`），以服务器状态为准

**关键缺失**：Google Reader 协议原生支持 `changed_after` 过滤（按状态变更时间拉取），但 Miniflux 实现中**只支持按 `published_at` 过滤**，不支持按 `changed_at` 过滤增量同步。客户端必须拉取条目详情才能发现状态变更。

#### 16.2.3 REST API v1 同步模式：明确的 `changed_after` 支持

**代码**（`api/entry_handlers.go:589-594`）：

```go
if afterChangedTimestamp := request.QueryInt64Param(r, "changed_after", 0); afterChangedTimestamp > 0 {
    builder = builder.AfterChangedDate(time.Unix(afterChangedTimestamp, 0))
}
```

这是最先进的同步方式——客户端可以直接"只拉取上次同步之后有状态变更的条目"，无需拉取全量 ID 列表。

### 16.3 冲突解决策略总览

| 同步入口 | 冲突检测方式 | 解决策略 | 位置 |
|---------|-------------|---------|------|
| Fever API | 客户端对比全量 ID 列表 | 客户端决定（通常服务器胜） | `fever/handler.go:331-370` |
| Google Reader API | 客户端对比条目的 `Updated` 字段 | `Updated` 大者胜 | `googlereader/handler.go:703` |
| REST API v1 | 客户端使用 `changed_after` 过滤 | 无冲突（只拉增量，客户端应用） | `api/entry_handlers.go:589-594` |
| Web UI | 无（每次重新加载页面） | 无冲突（实时查询数据库） | 所有 UI handler |

### 16.4 缺陷与边界情况

1. **`changed_at` 精度问题**：`now()` 的精度是微秒级，但两个请求在同一微秒内到达时，`changed_at` 相同，客户端无法判断哪个更新。
2. **状态 vs 收藏的独立更新**：设备 A 更新 `status`，设备 B 同时更新 `starred`，两者都能成功，最终状态是两者的组合（PostgreSQL 列级更新）。这实际上是"无冲突"，因为两个字段独立。
3. **Fever 的 `saved` 翻转问题**：`ToggleStarred` 在并发时可能导致收藏状态意外翻转（见第八章分析）。
4. **无删除同步**：条目被归档删除后，客户端不会收到"已删除"通知，只会在下次拉取时发现 ID 不在列表中。

---

## 十七、第三方阅读器协议差异下的状态映射边界

### 17.1 三种协议的状态模型对比

| 维度 | Miniflux 内部 | Fever API | Google Reader API |
|------|-------------|-----------|------------------|
| **已读状态** | `status` 字段（enum: unread/read） | `is_read`（0/1 整数） | `user/-/state/com.google/read` 标签（存在=已读，不存在=未读） |
| **收藏状态** | `starred` 字段（boolean） | `is_saved`（0/1 整数） | `user/-/state/com.google/starred` 标签（存在=收藏） |
| **保持未读** | 无对应字段 | 无对应字段 | `user/-/state/com.google/kept-unread` 标签（特殊语义） |
| **状态变更时间** | `changed_at`（timestamp） | 无对应字段，条目返回 `created_on_time` | `Updated` 字段（Unix 时间戳） |
| **批量标记已读** | 按 user_id / feed_id / category_id | `mark=feed` / `mark=group` + `before` 时间戳 | `mark-all-as-read` + `ts` 时间戳 |
| **同步粒度** | 无（全量查询） | 全量 ID 列表（`unread_item_ids`） | 条目详情带状态标签 |

### 17.2 Fever API 状态映射的边界与缺陷

#### 17.2.1 读状态映射

**Miniflux → Fever**（`fever/handler.go:303-325`）：

```go
for _, entry := range entries {
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
        IsSaved:   isSaved,
        IsRead:    isRead,
        CreatedAt: entry.Date.Unix(),  // ★ 注意：用的是 published_at，不是 changed_at
    })
}
```

**边界 1：`created_on_time` 的语义偏差**

Fever 协议规定 `created_on_time` 是条目创建时间，但 Miniflux 返回的是 `entry.Date.Unix()`（即 RSS 的 `published_at`）。对于客户端同步来说，这意味着：
- 无法通过 `created_on_time` 判断状态是否变更
- 只能通过 `is_read` / `is_saved` 字段的当前值判断

**Fever → Miniflux**（`fever/handler.go:429-470`）：

```go
switch r.FormValue("as") {
case "read":
    h.store.SetEntriesStatus(userID, []int64{entryID}, model.EntryStatusRead)
case "unread":
    h.store.SetEntriesStatus(userID, []int64{entryID}, model.EntryStatusUnread)
case "saved":
    h.store.ToggleStarred(userID, entryID)  // ★ 缺陷 1：用 Toggle 而非显式设置 true
case "unsaved":
    h.store.ToggleStarred(userID, entryID)  // ★ 缺陷 2：同样用 Toggle 而非显式设置 false
}
```

#### 17.2.2 收藏状态的 Toggle 缺陷

这是最严重的映射边界问题：

**问题场景**：
1. 条目初始状态：未收藏（`starred = false`）
2. 客户端 A 调用 `mark=item&as=saved` → `ToggleStarred` → `starred = true` ✓
3. 同步延迟下，客户端 B 本地状态仍是未收藏
4. 客户端 B 也调用 `mark=item&as=saved` → `ToggleStarred` → `starred = false` ✗（预期应该保持 true）

**正确实现应该是**：

```go
// 伪代码——当前代码没有这样做
case "saved":
    if !entry.Starred {
        h.store.SetEntriesStarredState(userID, []int64{entryID}, true)
    }
case "unsaved":
    if entry.Starred {
        h.store.SetEntriesStarredState(userID, []int64{entryID}, false)
    }
```

当前代码（`fever/handler.go:447,466`）在写操作前**已经 SELECT 了 entry**，但没有利用该信息做条件判断，直接 `ToggleStarred`。

#### 17.2.3 范围标记已读的时间戳映射

**Fever → Miniflux**（`fever/handler.go:481-541`）：

```go
// mark=feed
func (h *feverHandler) handleWriteFeeds(w http.ResponseWriter, r *http.Request) {
    before := time.Unix(request.FormInt64Value(r, "before"), 0)
    h.store.MarkFeedAsRead(userID, feedID, before)  // ★ 传递 before 时间戳
}

// mark=group&id=0
case groupID == 0:
    h.store.MarkAllAsRead(userID)  // ★ 注意：没有 before 参数，标记全部
```

**边界 2：`mark=group&id=0` 忽略 `before` 参数**

Fever 协议允许 `mark=group` 时带上 `before` 参数，但 Miniflux 实现中 `groupID == 0`（表示全部）时调用 `MarkAllAsRead`，该函数**不接受时间戳参数**，会标记**所有**未读条目。如果客户端本意是"标记全部中在某个时间点之前的条目"，会导致超出预期的标记范围。

### 17.3 Google Reader API 状态映射的边界与缺陷

#### 17.3.1 标签系统到内部状态的映射

**标签解析**（`googlereader/stream.go:73-107`）：

```
标签 → 内部 StreamType
user/123/state/com.google/read           → ReadStream
user/123/state/com.google/starred        → StarredStream
user/123/state/com.google/reading-list   → ReadingListStream
user/123/state/com.google/kept-unread    → KeptUnreadStream
user/123/label/MyFolder                 → LabelStream (Category)
feed/456                                → FeedStream
```

**标签简化与冲突检测**（`googlereader/handler.go:1234-1281`）：

```go
func checkAndSimplifyTags(addTags []Stream, removeTags []Stream) (map[StreamType]bool, error) {
    tags := make(map[StreamType]bool)

    // add 处理
    for _, s := range addTags {
        switch s.Type {
        case ReadStream:
            if _, ok := tags[KeptUnreadStream]; ok {
                return nil, errSimultaneously  // ★ 互斥检测
            }
            tags[ReadStream] = true
        case KeptUnreadStream:
            if _, ok := tags[ReadStream]; ok {
                return nil, errSimultaneously  // ★ 互斥检测
            }
            tags[ReadStream] = false  // kept-unread 映射为"不读"
        case StarredStream:
            tags[StarredStream] = true
        }
    }

    // remove 处理
    for _, s := range removeTags {
        switch s.Type {
        case ReadStream:
            if _, ok := tags[ReadStream]; ok {
                return nil, errSimultaneously  // ★ 同时 add 和 remove 报错
            }
            tags[ReadStream] = false  // remove read = 标记未读
        case KeptUnreadStream:
            if _, ok := tags[ReadStream]; ok {
                return nil, errSimultaneously
            }
            tags[ReadStream] = true   // remove kept-unread = 标记已读
        case StarredStream:
            if _, ok := tags[StarredStream]; ok {
                return nil, fmt.Errorf("...")
            }
            tags[StarredStream] = false
        }
    }
    return tags, nil
}
```

**边界 1：`kept-unread` 语义的二义性**

Google Reader 中 `kept-unread` 标签的语义是"即使条目已读，也保持在未读列表中"。但 Miniflux 没有对应字段，简化映射为：
- `add kept-unread` = `SetEntriesStatus(status=unread)`
- `remove kept-unread` = `SetEntriesStatus(status=read)`

这丢失了"保持未读"的语义（Miniflux 没有"读了但仍显示为未读"的中间状态）。

#### 17.3.2 状态应用前的条件检查

**代码**（`googlereader/handler.go:248-271`）：

```go
var readEntryIDs, unreadEntryIDs, starredEntryIDs, unstarredEntryIDs []int64
for _, entry := range entries {
    if read, exists := tags[ReadStream]; exists {
        if read && entry.Status == model.EntryStatusUnread {
            readEntryIDs = append(readEntryIDs, entry.ID)  // ★ 只有未读的才加入已读列表
        } else if !read && entry.Status == model.EntryStatusRead {
            unreadEntryIDs = append(unreadEntryIDs, entry.ID)  // ★ 只有已读的才加入未读列表
        }
    }
    if starred, exists := tags[StarredStream]; exists {
        if starred && !entry.Starred {
            starredEntryIDs = append(starredEntryIDs, entry.ID)  // ★ 只有未收藏的才加入收藏列表
            entries[n] = entry
            n++
        } else if !starred && entry.Starred {
            unstarredEntryIDs = append(unstarredEntryIDs, entry.ID)
        }
    }
}
```

这是 Google Reader 实现比 Fever 更健壮的地方——**应用状态变更前先检查当前状态**，只对真正需要变更的条目调用 UPDATE。这避免了：
1. 不必要的数据库写入
2. `changed_at` 被无意义地更新
3. `RowsAffected == 0` 错误（`SetEntriesStarredState` 会检查并报错）

#### 17.3.3 状态回传的标签映射

**代码**（`googlereader/handler.go:680-691`）：

```go
categories := make([]string, 0, 4)
categories = append(categories, userReadingList)  // 始终包含
if entry.Feed.Category.Title != "" {
    categories = append(categories, labelPrefix+entry.Feed.Category.Title)
}
if entry.Status == model.EntryStatusRead {
    categories = append(categories, userRead)  // 已读 → 添加 read 标签
}
if entry.Starred {
    categories = append(categories, userStarred)  // 收藏 → 添加 starred 标签
}
```

**边界 2：`kept-unread` 标签永不返回**

因为 Miniflux 没有对应字段，即使客户端之前添加了 `kept-unread` 标签，下次拉取时该标签也不会出现在 `categories` 中。客户端会认为该标签已被移除。

#### 17.3.4 时间戳映射

**代码**（`googlereader/handler.go:700-703`）：

```go
TimestampUsec: strconv.FormatInt(entry.Date.UnixMicro(), 10),    // published_at（微秒）
CrawlTimeMsec: strconv.FormatInt(entry.CreatedAt.UnixMilli(), 10), // created_at（毫秒）
Published:     entry.Date.Unix(),                              // published_at（秒）
Updated:       entry.ChangedAt.Unix(),                         // ★ changed_at（秒）
```

Google Reader 协议返回 4 个时间戳，Miniflux 都正确映射了。其中 `Updated` 字段是客户端判断状态是否变更的关键。

### 17.4 三方协议的状态幂等性对比

| 操作 | Fever API | Google Reader API | REST API v1 |
|------|----------|------------------|------------|
| 标记已读 | 非幂等（重复调用无副作用，但也不报错） | 幂等（先检查当前状态） | 非幂等（直接 UPDATE） |
| 标记未读 | 非幂等 | 幂等 | 非幂等 |
| 添加收藏 | **非幂等且危险**（ToggleStarred 可能翻转） | 幂等（先检查 `!entry.Starred`） | 非幂等 |
| 取消收藏 | **非幂等且危险**（ToggleStarred 可能翻转） | 幂等（先检查 `entry.Starred`） | 非幂等 |
| 批量标记已读 | 非幂等 | 幂等（按 before 时间戳） | 非幂等 |

### 17.5 不支持的标签与静默忽略

两个兼容接口都有一些协议标签被静默忽略：

**Google Reader**（`googlereader/handler.go:1250-1251,1273-1274`）：
```go
case BroadcastStream, LikeStream:
    slog.Debug("Broadcast & Like tags are not implemented!")
```

**Fever**：
- `is_spark` 字段恒为 0（不支持 Sparks 功能）
- `favicon_id` 只有在 Feed 有图标时才设置

这些标签的 add/remove 不会报错，但也不会产生任何效果。

---

## 十八、Entry 内容在 Web 端的离线缓存与状态同步机制

### 18.1 总体设计：极简离线支持

Miniflux 的 Web 端离线功能非常克制——**不缓存条目数据，只缓存一个离线提示页**。状态同步也采用**即时同步 + 前端乐观更新**的简单模式，没有本地队列或延迟同步。

### 18.2 Service Worker 与离线缓存

**代码**（`ui/static/js/service_worker.js:1-44`）：

```javascript
const OFFLINE_VERSION = 1;
const CACHE_NAME = "offline";

self.addEventListener("install", (event) => {
    event.waitUntil((async () => {
        const cache = await caches.open(CACHE_NAME);
        await cache.add(new Request(OFFLINE_URL, { cache: "reload" }));
    })());
    self.skipWaiting();
});

self.addEventListener("fetch", (event) => {
    // 只在离线且是导航请求时才介入
    if (navigator.onLine === false && event.request.mode === "navigate") {
        event.respondWith((async () => {
            try {
                const networkResponse = await fetch(event.request);
                return networkResponse;
            } catch (error) {
                const cache = await caches.open(CACHE_NAME);
                const cachedResponse = await cache.match(OFFLINE_URL);
                return cachedResponse;
            }
        })());
    }
});
```

**离线缓存内容**：

| 缓存项 | 内容 | 缓存策略 |
|--------|------|---------|
| `OFFLINE_URL` ( /offline ) | 纯静态 HTML 离线提示页 | 安装时预缓存，每次 SW 版本更新时重新获取 |

**不缓存的内容**（重要）：
- ❌ 条目列表（HTML 或 JSON）
- ❌ 条目内容/正文
- ❌ Feed 列表
- ❌ 未读计数
- ❌ 收藏状态

**设计意图**：Service Worker 只提供"降级体验"——离线时显示一个友好的"你已离线"页面，而不是完整的离线阅读。这与 Feedly、Inoreader 等商业 RSS 阅读器的完整离线缓存有本质区别。

### 18.3 前端状态更新：乐观 UI + 服务器即时同步

Web 端的条目状态操作（标记已读/未读、切换收藏）采用**乐观更新**模式：

#### 18.3.1 状态变更的前端流程

**代码**（`ui/static/js/app.js:682-692`）：

```javascript
function updateEntriesStatus(entryIDs, status, callback) {
    const url = document.body.dataset.entriesStatusUrl;
    sendPOSTRequest(url, { entry_ids: entryIDs, status: status }).then((resp) => {
        resp.json().then(count => {
            if (callback) {
                callback(resp);
            }
            // 服务器返回可见条目数，用于更新未读计数
            updateUnreadCounterValue(status === "read" ? -count : count);
        });
    });
}
```

**列表页标记当前页为已读**（`app.js:577-596`）：

```javascript
function markPageAsReadAction() {
    const items = getVisibleEntries();
    const entryIDs = items.map(element => parseInt(element.dataset.id, 10));

    // 步骤1：乐观更新 DOM（立即添加已读样式）
    items.forEach(element => element.classList.add("item-status-read"));

    // 步骤2：发送请求到服务器
    updateEntriesStatus(entryIDs, "read", () => {
        // 步骤3：回调中处理页面跳转
        if (showOnlyUnread) {
            window.location.reload();
        } else {
            goToPage("next", true);
        }
    });
}
```

**收藏切换**（`app.js:721-741`）：

```javascript
function handleStarAction(element) {
    // 步骤1：设为 loading 状态
    setButtonToLoadingState(buttonElement);

    // 步骤2：发送请求
    sendPOSTRequest(buttonElement.dataset.starUrl).then(() => {
        // 步骤3：收到响应后更新按钮状态
        const isStarred = currentState === "star";
        const newStarStatus = isStarred ? "unstar" : "star";
        setStarredButtonState(buttonElement, newStarStatus);
    });
}
```

**时序对比**：

| 操作 | DOM 更新时机 | 计数更新时机 |
|------|------------|------------|
| 标记已读/未读 | 请求发送后立即乐观更新 | 响应返回后（使用服务器返回的可见计数） |
| 切换收藏 | 请求完成后更新（非乐观） | 不更新全局未读计数（收藏不影响未读计数） |
| 播放完成标记已读 | 达到阈值时立即更新 DOM + 后台发送请求 | 不更新（媒体页场景） |

#### 18.3.2 未读计数的前端更新

**代码**（`app.js:890-901`）：

```javascript
function updateUnreadCounterValue(delta) {
    // 更新所有页面上的未读计数器元素
    document.querySelectorAll("span.unread-counter").forEach((element) => {
        const oldValue = parseInt(element.textContent, 10);
        element.textContent = oldValue + delta;
    });

    // 如果在未读列表页，同时更新页面标题
    if (window.location.href.endsWith('/unread')) {
        document.title = document.title.replace(/\(\d+\)/, `(${newValue})`);
    }
}
```

**关键设计**：
- 未读计数的 delta 不是客户端计算的，而是**使用服务器返回的可见条目数**（`SetEntriesStatusAndCountVisible` 的返回值）
- 这保证了：即使某些条目属于被隐藏的 Feed（`hide_globally = true`），前端计数也不会"多减"
- 因为 Web UI 的全局未读计数本来就不包含隐藏的 Feed

### 18.4 状态同步的三种触发方式

| 触发方式 | 场景 | 同步行为 |
|---------|------|---------|
| 用户点击 | 列表页/详情页手动切换状态 | 即时 AJAX POST，乐观更新 UI |
| 键盘快捷键 | `m` 切换已读、`s` 切换收藏 | 同点击，调用相同的 JS 函数 |
| 媒体播放完成 | 音频/视频播放到指定百分比 | 后台静默发送标记已读请求 |
| 前进/后退缓存（bfcache） | 从历史记录返回页面 | `pageshow` 事件中重新标记已读（`app.js:1288`） |

### 18.5 离线状态下的行为

由于没有本地状态队列，离线时的状态操作：
1. 用户点击标记已读 → `fetch()` 失败 → `catch` 中没有降级处理
2. 按钮保持 loading 状态或报错
3. 状态变更**不会被暂存**，刷新后恢复为原状态
4. 用户需要手动重试

这是极简设计的代价——Miniflux Web UI 本质上是**在线应用**，Service Worker 只提供降级体验而非完整离线功能。

---

## 十九、过期 Entry 的清理触发与级联状态删除

### 19.1 清理任务的触发机制

#### 19.1.1 定时调度：后台 Scheduler

**代码**（`cli/scheduler.go:27-30, 53-57`）：

```go
func runScheduler(store *storage.Storage, pool *worker.Pool) {
    // ...
    go cleanupScheduler(
        store,
        config.Opts.CleanupFrequency(),
    )
}

func cleanupScheduler(store *storage.Storage, frequency time.Duration) {
    for range time.Tick(frequency) {
        runCleanupTasks(store)
    }
}
```

- 调度器随服务启动自动启动（`-d` / `--daemon` 模式）
- 使用 `time.Tick` 定时触发，**无并发保护**（如果 `runCleanupTasks` 执行时间超过 frequency，会并发执行）
- 实际受 `FOR UPDATE SKIP LOCKED` 保护，不会产生数据冲突

#### 19.1.2 手动触发：CLI 命令

```bash
miniflux --run-cleanup-tasks
```

可通过 cron 或 systemd timer 外部调度。

#### 19.1.3 配置参数

| 参数 | 默认值 | 含义 |
|------|-------|------|
| `CLEANUP_FREQUENCY` | 24 小时 | 清理任务执行频率 |
| `CLEANUP_ARCHIVE_READ_DAYS` | 60 天 | 已读条目的保留天数 |
| `CLEANUP_ARCHIVE_UNREAD_DAYS` | 180 天 | 未读条目的保留天数 |
| `CLEANUP_ARCHIVE_BATCH_SIZE` | 10000 | 每批删除的条目数 |

**代码**（`cli/cleanup_tasks.go:16-58`）：

```go
func runCleanupTasks(store *storage.Storage) {
    // 1. 清理过期 Web Session
    store.CleanOldWebSessions(config.Opts.CleanupRemoveSessionsInterval())

    // 2. 归档已读条目
    store.ArchiveEntries(
        model.EntryStatusRead,
        config.Opts.CleanupArchiveReadInterval(),   // 默认 60 天
        config.Opts.CleanupArchiveBatchSize(),       // 默认 10000
    )

    // 3. 归档未读条目
    store.ArchiveEntries(
        model.EntryStatusUnread,
        config.Opts.CleanupArchiveUnreadInterval(), // 默认 180 天
        config.Opts.CleanupArchiveBatchSize(),       // 默认 10000
    )

    // 4. 清理孤立图标
    store.CleanupOrphanIcons()
}
```

### 19.2 核心清理逻辑：`ArchiveEntries`

**代码**（`storage/entry.go:362-404`）：

```sql
WITH to_delete AS (
    SELECT id, feed_id, hash
    FROM entries
    WHERE
        status=$1 AND                  -- 按状态分别清理
        starred is false AND           -- ★ 保护 1：已收藏的不删
        share_code='' AND              -- ★ 保护 2：已分享的不删
        created_at < now() - $2::interval  -- 超过保留时间
    ORDER BY created_at ASC
    FOR UPDATE SKIP LOCKED              -- 悲观锁 + 跳过被锁的行
    LIMIT $3                            -- 批量限制
),
deleted AS (
    DELETE FROM entries
    USING to_delete
    WHERE entries.id = to_delete.id
    RETURNING entries.feed_id, entries.hash
)
INSERT INTO entry_tombstones (feed_id, hash)
SELECT feed_id, hash FROM deleted WHERE hash <> ''
ON CONFLICT (feed_id, hash) DO NOTHING   -- 幂等插入墓碑
```

**执行流程**（单条 SQL 内）：
1. `to_delete` CTE：选中待删除行，加 `FOR UPDATE SKIP LOCKED` 行锁
2. `deleted` CTE：删除选中行
3. 插入 `entry_tombstones` 墓碑记录

### 19.3 受保护的条目（不被清理）

| 保护条件 | 原因 |
|---------|------|
| `starred = true` | 用户收藏的条目是"重要内容"，不应被自动清理 |
| `share_code <> ''` | 已分享的条目有公开 URL，删除会导致链接失效 |

**注意**：保护条件仅对 `ArchiveEntries` 有效。`FlushHistory`（手动清空历史）只保护收藏的条目，不保护分享的。

### 19.4 `FlushHistory`：用户主动清空历史

**代码**（`storage/entry.go:491-510`）：

```go
func (s *Storage) FlushHistory(userID int64) error {
    query := `
        WITH deleted AS (
            DELETE FROM entries
            WHERE user_id=$1 AND status='read' AND starred is false
            RETURNING feed_id, hash
        )
        INSERT INTO entry_tombstones (feed_id, hash)
        SELECT feed_id, hash FROM deleted WHERE hash <> ''
        ON CONFLICT (feed_id, hash) DO NOTHING
    `
    _, err := s.db.Exec(query, userID)
    return err
}
```

**与 `ArchiveEntries` 的区别**：
- 范围：仅指定用户的已读条目（非全用户）
- 时间：不按时间过滤，删除**所有**已读且未收藏的条目
- 触发：用户主动操作（Web UI "清空历史"按钮、API `POST /v1/me/flush-history`）
- 保护：只保护 `starred = false`，不检查 `share_code`

### 19.5 级联删除与关联数据清理

#### 19.5.1 Feed 删除时的级联

**Schema 定义**（`database/migrations.go:78-91`）：

```sql
CREATE TABLE entries (
    ...
    feed_id bigint not null references feeds(id) on delete cascade,
    ...
);
```

删除 Feed 时，PostgreSQL 自动级联删除该 Feed 的所有条目。但**附属表需要额外处理**：

- `enclosures`：有外键指向 entries，`ON DELETE CASCADE`，自动级联
- `entry_tombstones`：有外键指向 feeds，`ON DELETE CASCADE`，自动级联

#### 19.5.2 手动清理：孤立图标

**代码**（`storage/icon.go:150-162`）：

```go
// feed. Such rows accumulate when feeds are deleted (the cascade only removes
// icons that are still referenced; orphaned rows stay behind).
func (s *Storage) CleanupOrphanIcons() (int64, error) {
    result, err := s.db.Exec(`
        DELETE FROM icons WHERE id NOT IN (SELECT icon_id FROM feeds WHERE icon_id IS NOT NULL)
    `)
    // ...
}
```

图标表没有 `ON DELETE SET NULL` 或级联，Feed 删除后图标变成孤儿，需要定期手动清理。

### 19.6 清理对未读计数的影响

由于未读计数是实时查询的：
- 清理已读条目 → 不影响未读计数
- 清理未读条目 → 未读计数**自动减少**（因为条目被删除了）
- 客户端感知：下次拉取时发现条目不在列表中，或未读计数变少

**注意**：Fever API 的 `unread_item_ids` 和 Google Reader 的 `stream/items/ids` 都是基于当前 entries 表实时查询的，清理后这些列表会自动反映最新状态。

### 19.7 清理的幂等性与并发安全

| 安全机制 | 作用 |
|---------|------|
| `FOR UPDATE SKIP LOCKED` | 并发清理任务不会重复删除同一行，跳过已被锁定的行 |
| `ON CONFLICT DO NOTHING` | 墓碑插入幂等，重复插入不会报错 |
| `created_at < now() - interval` | 时间窗口判断，已删除的不会再被选中 |

即使多个 `cleanupScheduler` 并发运行（如 frequency 过短 + 上轮未执行完），数据也不会出问题。

---

## 二十、Entry 状态对统计聚合（Per-Feed / Per-Category）的反馈链路

### 20.1 总体设计：无缓存实时聚合

Miniflux 的统计聚合**完全不使用缓存或计数器表**，每次请求时从 `entries` 表实时 `GROUP BY` 计算。这是"状态变更 → 聚合反映"链路的根本特征——没有中间层，状态变更直接体现在下次聚合查询中。

### 20.2 Per-Feed 计数：`feedQueryBuilder.fetchFeedCounter()`

#### 20.2.1 核心 SQL

**代码**（`storage/feed_query_builder.go:302-350`）：

```sql
SELECT
    e.feed_id,
    e.status,
    count(*)
FROM entries e
    [INNER JOIN feeds f ON f.id=e.feed_id]  -- 仅当按 category 过滤时
WHERE
    e.user_id = $1
    AND e.status IN ($2, $3)               -- 只统计 unread 和 read
    [AND f.category_id = $N]               -- 可选：按 category 过滤
GROUP BY e.feed_id, e.status
```

**返回结构**：两个 map
- `readCounters[feedID] = count`
- `unreadCounters[feedID] = count`

#### 20.2.2 调用入口

| 调用方 | 函数 | 场景 |
|-------|------|------|
| Web UI 列表页 | `FeedsWithCounters()` | 侧边栏 Feed 列表，显示每个 Feed 的未读数 |
| Web UI 分类页 | `FeedsByCategoryWithCounters()` | 分类内的 Feed 列表 |
| REST API v1 | `FetchCounters()` | `/v1/feeds/counters` 端点，客户端同步使用 |

**Web UI Feed 列表页**（`ui/feed_list.go:14-37`）：

```go
func (h *handler) showFeedsPage(w http.ResponseWriter, r *http.Request) {
    feeds, err := h.store.FeedsWithCounters(user.ID)
    navMetadata, _ := h.store.GetNavMetadata(user.ID)
    view.Set("countUnread", navMetadata.CountUnread)
    // ...
}
```

**API Counters 端点**（`api/feed_handlers.go:209-217`）：

```go
func (h *handler) fetchCountersHandler(w http.ResponseWriter, r *http.Request) {
    counters, err := h.store.FetchCounters(request.UserID(r))
    response.JSON(w, r, counters)
}
```

#### 20.2.3 模型层表示

**代码**（`model/feed.go:74-76, 79-82`）：

```go
type Feed struct {
    // ...
    UnreadCount            int  `json:"-"`  // 内部属性，不序列化到 API
    ReadCount              int  `json:"-"`  // 内部属性，不序列化到 API
    NumberOfVisibleEntries int  `json:"-"`  // ReadCount + UnreadCount
}

type FeedCounters struct {
    ReadCounters   map[int64]int `json:"reads"`
    UnreadCounters map[int64]int `json:"unreads"`
}
```

注意：`Feed` 结构体的 `UnreadCount` / `ReadCount` 标记为 `json:"-"`，**不在 Feed API 中直接返回**。独立的 `/v1/feeds/counters` 端点专门返回计数数据，供客户端同步使用。

### 20.3 Per-Category 计数：`CategoriesWithFeedCount()`

#### 20.3.1 核心 SQL

**代码**（`storage/category.go:112-170`）：

```sql
SELECT
    c.id, c.user_id, c.title, c.hide_globally,
    coalesce(fc.feed_count, 0),       -- 该分类下的 Feed 数量
    coalesce(uc.unread_count, 0)      -- 该分类下的未读条目总数
FROM categories c
LEFT JOIN (
    SELECT category_id, count(*) AS feed_count
    FROM feeds
    WHERE user_id = $2
    GROUP BY category_id
) fc ON fc.category_id = c.id
LEFT JOIN (
    SELECT f.category_id, count(*) AS unread_count
    FROM entries e
    INNER JOIN feeds f ON f.id = e.feed_id
    WHERE e.user_id = $2 AND e.status = $1  -- $1 = 'unread'
    GROUP BY f.category_id
) uc ON uc.category_id = c.id
WHERE c.user_id = $2
ORDER BY
    [c.title ASC] 或 [uc.unread_count DESC, c.title ASC]
```

**特点**：
- 分类计数只统计**未读**（`status = 'unread'`），不统计已读
- 使用 `LEFT JOIN + coalesce` 保证"空分类"显示为 0 而非 NULL
- 可按 `unread_count` 排序（用户偏好：`CategoriesSortingOrder`）

#### 20.3.2 调用入口

| 调用方 | 场景 |
|-------|------|
| Web UI 分类列表 | `/categories` 页面，显示每个分类的 Feed 数和未读数 |
| REST API v1 | `/v1/categories?counts=true` |

**模型层**（`model/category.go:15-16`）：

```go
type Category struct {
    FeedCount   *int `json:"feed_count,omitempty"`   // 用指针 + omitempty 表示可选
    TotalUnread *int `json:"total_unread,omitempty"`
}
```

### 20.4 状态变更到聚合反映的完整链路

以"标记条目 123 为已读"为例，整个反馈链路：

```
时间点 T0:
  feed A: unread=10, read=50
  category X: unread=100

时间点 T1: 用户点击"标记已读"
  → UI 层: updateEntriesStatus([123], "read")
     → HTTP POST /entry/status
        → UI Handler: entry_update_status.go
           → Storage: SetEntriesStatusAndCountVisible(userID, [123], "read")
              → SQL: WITH updated AS (UPDATE ...) SELECT count(*) ...
                 ← 返回可见条目数 count
           ← 返回 count 给前端
        ← 响应: JSON number
     → 前端: updateUnreadCounterValue(-count)  ← 更新导航栏计数
  → 此时 feed A: unread=9, read=51 （数据库中已变更）

时间点 T2: 用户刷新页面或点击下一页
  → 重新查询 FeedsWithCounters()
     → 重新执行 GROUP BY 查询
     → 反映最新状态
```

**关键点**：
1. 状态变更与聚合同步**不是同一请求内完成**的（导航栏计数除外）
2. Feed 列表的计数只有在下次重新获取 Feed 列表时才更新
3. 没有"实时推送"或"WebSocket"机制
4. 前端的未读计数 delta 更新只影响导航栏，不影响侧边栏 Feed 列表的计数

### 20.5 计数查询的性能优化

#### 20.5.1 覆盖索引

`entries_user_status_feed_idx (user_id, status, feed_id)` 是 feed 粒度计数的完美覆盖索引——PostgreSQL 可以直接对索引做 Index Only Scan，无需回表。

#### 20.5.2 一次性获取，避免 N+1

`FeedsWithCounters()` 的执行流程：
1. 一次查询获取所有 Feed 基础信息
2. 一次 `fetchFeedCounter()` 查询获取所有 Feed 的计数（`GROUP BY feed_id`）
3. 在 Go 代码中按 feed_id 关联赋值

总共 **2 次 SQL 查询**，而不是每个 Feed 查一次计数（避免 N+1）。

#### 20.5.3 分类计数的单次查询

`CategoriesWithFeedCount()` 在**单条 SQL** 中用两个 LEFT JOIN 子查询同时获取 feed_count 和 unread_count。

### 20.6 一致性保证：数据库为唯一真相源

由于所有聚合查询都直接从 entries 表实时计算：

| 一致性属性 | 保证方式 |
|-----------|---------|
| 状态变更 → 立即可聚合 | 单 SQL 事务，UPDATE 提交后聚合查询立即可见 |
| Feed 计数 = 所有条目 status 统计之和 | 同一张表，天然一致 |
| 分类计数 = 分类下所有 Feed 计数之和 | 同一张表，天然一致 |
| 全局未读计数 = 所有 Feed 未读数之和 | 同一张表，天然一致 |

不存在"Feed 计数加起来不等于全局计数"的不一致问题——因为所有数字都来自同一张表的实时统计。

### 20.7 特殊聚合：每周条目数预测

**代码**（`storage/feed.go:167-197`）：

```go
func (s *Storage) WeeklyFeedEntryCount(userID, feedID int64) (int, error) {
    query := `
        SELECT COALESCE(CAST(CEIL(
            (EXTRACT(epoch from interval '1 week')) /
            NULLIF(
                (EXTRACT(epoch from (max(published_at)-min(published_at))
                 / NULLIF((count(*)-1), 0)
                )
            ), 0)
        ) AS BIGINT), 0)
        FROM entries
        WHERE entries.user_id=$1 AND entries.feed_id=$2
          AND entries.published_at >= now() - interval '1 week'
    `
}
```

这是一个**虚拟指标**：基于最近一周条目的平均发布间隔，推算每周预计条目数。用于 Feed 列表页展示"更新频率"参考。

---

## 二十一、Entry 状态变更触发 Webhook 与第三方通知的代码链路

### 21.1 核心结论：状态变更本身不触发通知，只有"Save"和"新条目推送"触发

Miniflux 的 entry 状态（read/unread/starred）变更**不会**触发 Webhook 或邮件通知。触发通知的只有两类场景：

| 触发场景 | 触发条件 | 通知类型 |
|---------|---------|---------|
| **Save Entry** | 用户点击"保存到第三方服务"按钮 | `save_entry` Webhook 事件 + 28 种第三方书签/稍后读服务推送 |
| **New Entries Push** | Feed 刷新发现新条目 | `new_entries` Webhook 事件 + Telegram/Discord/Slack/Ntfy/Pushover/Matrix 等即时推送 |

> **重要**：Miniflux **没有内置邮件通知功能**。邮件通知需通过 Apprise、Ntfy、Pushover 等第三方集成间接实现。

### 21.2 Save Entry 的触发链路

`integration.SendEntry()` 函数是 Save 通知的入口，被四个协议入口调用：

**调用入口总览**：

| 协议层 | 文件位置 | 调用方式 |
|--------|---------|---------|
| Web UI | `ui/entry_save.go:36` | `go integration.SendEntry(entry, userIntegrations)` |
| REST API v1 | `api/entry_handlers.go:263` | `go integration.SendEntry(entry, settings)` |
| Fever API | `fever/handler.go:458-460` | `go func() { integration.SendEntry(entry, settings) }()` |
| Google Reader API | `googlereader/handler.go:311-316` | 循环每个 starred entry: `go func() { integration.SendEntry(e, settings) }()` |

**Fever API 的 Save 触发上下文**（`fever/handler.go:442-460`）：

```go
case "saved":
    h.store.ToggleStarred(userID, entryID)  // 先切换收藏状态
    settings, _ := h.store.Integration(userID)
    go func() {
        integration.SendEntry(entry, settings)  // 再异步推送
    }()
```

**Google Reader API 的 Save 触发上下文**（`googlereader/handler.go:296-316`）：

```go
if len(starredEntryIDs) > 0 {
    h.store.SetEntriesStarredState(userID, starredEntryIDs, true)
}
// ...
for _, entry := range entries {
    e := entry
    go func() {
        integration.SendEntry(e, settings)  // 每个被收藏的条目都单独推送
    }()
}
```

### 21.3 New Entries Push 的触发链路

Feed 刷新完成后，在 `reader/handler/handler.go:330-339` 中触发：

```go
userIntegrations, intErr := store.Integration(userID)
if intErr != nil {
    slog.Error("Fetching integrations failed; ...")
} else if userIntegrations != nil && len(newEntries) > 0 {
    go integration.PushEntries(originalFeed, newEntries, userIntegrations)
}
```

关键细节：
- **条件严格**：必须同时满足「集成配置获取成功」+「有新条目」才触发
- **失败不阻断**：集成配置获取失败只打日志，不影响 Feed 刷新主流程
- **异步执行**：所有推送都在 goroutine 中执行，不阻塞 HTTP 响应

### 21.4 Webhook URL 覆盖优先级

在 `integration.SendEntry()` 中（`integration/integration.go:422-428`）和 `integration.PushEntries()` 中（`integration/integration.go:536-542`），Webhook URL 有两级覆盖：

```go
var webhookURL string
if entry.Feed != nil && entry.Feed.WebhookURL != "" {
    webhookURL = entry.Feed.WebhookURL   // Feed 级配置优先
} else {
    webhookURL = userIntegrations.WebhookURL  // 回退到用户级配置
}
```

**优先级链**：`Feed.WebhookURL` (per-Feed) > `Integration.WebhookURL` (per-User)

### 21.5 Webhook 事件格式与签名

两种事件类型定义在 `integration/webhook/webhook.go:24-26`：

```go
const (
    NewEntriesEventType = "new_entries"
    SaveEntryEventType  = "save_entry"
)
```

**请求头**（`webhook.go:132-135`）：

| Header | 含义 |
|--------|------|
| `Content-Type` | `application/json` |
| `User-Agent` | `Miniflux/{version}` |
| `X-Miniflux-Event-Type` | `new_entries` 或 `save_entry` |
| `X-Miniflux-Signature` | SHA256 HMAC 签名，密钥为用户配置的 Webhook Secret |

**安全性**：默认 Block 私有网络请求，除非 `INTEGRATION_ALLOW_PRIVATE_NETWORKS=true`（`webhook.go:137`）。

### 21.6 第三方集成的触发范围

`SendEntry()` 覆盖 28 种第三方服务：
- 书签类：Pinboard、Shaarli、LinkAce、Linkding、LinkTaco、Linkwarden、Raindrop、Shiori、Espial、Cubox、Betula、Karakeep、Omnivore
- 稍后读类：Instapaper、Wallabag、Notion、NunuxKeeper、Readeck、Readwise
- 归档类：Archive.org
- 通用 Webhook：用户自定义 URL

`PushEntries()` 覆盖 8 种即时通知服务：
- 聊天类：Telegram Bot、Discord、Slack、Matrix Bot
- 推送类：Pushover、Ntfy
- 通用通知：Apprise、Webhook

---

## 二十二、Entry Retention 配置覆盖优先级分析

### 22.1 核心结论：没有 User-Level 或 Feed-Level Retention，只有全局配置

经过对 User 模型、Feed 模型和 Storage 层的全面排查，Miniflux **不存在 per-User 或 per-Feed 的 entry retention 配置**。所有 retention 参数都是全局级别的，通过环境变量配置。

### 22.2 全局配置参数

定义在 `config/options.go:651-665`：

| 环境变量 | 默认值 | Go 访问方法 | 含义 |
|---------|-------|------------|------|
| `CLEANUP_ARCHIVE_READ_DAYS` | 60 天 | `CleanupArchiveReadInterval()` | 已读条目的保留天数 |
| `CLEANUP_ARCHIVE_UNREAD_DAYS` | 180 天 | `CleanupArchiveUnreadInterval()` | 未读条目的保留天数 |
| `CLEANUP_ARCHIVE_BATCH_SIZE` | 10000 | `CleanupArchiveBatchSize()` | 每批删除的条目数 |
| `CLEANUP_FREQUENCY_HOURS` | 24 小时 | `CleanupFrequency()` | 清理任务执行频率 |

### 22.3 配置的消费点

在 `cli/cleanup_tasks.go:16-58` 中直接使用全局 config，**不做任何 user/feed 级覆盖**：

```go
func runCleanupTasks(store *storage.Storage) {
    // ...
    store.ArchiveEntries(
        model.EntryStatusRead,
        config.Opts.CleanupArchiveReadInterval(),   // 直接取全局配置
        config.Opts.CleanupArchiveBatchSize(),
    )
    store.ArchiveEntries(
        model.EntryStatusUnread,
        config.Opts.CleanupArchiveUnreadInterval(), // 直接取全局配置
        config.Opts.CleanupArchiveBatchSize(),
    )
    // ...
}
```

`ArchiveEntries` 函数签名（`storage/entry.go:363`）：
```go
func (s *Storage) ArchiveEntries(status string, interval time.Duration, limit int) (int64, error)
```

该函数没有 `userID` 参数，是**全表扫描删除**，不区分用户。

### 22.4 唯一的 User-Level 清理操作：`FlushHistory`

`FlushHistory(userID)`（`storage/entry.go:491-508`）是唯一按 user 区分的清理操作，但它是**用户主动触发的即时删除**，不涉及 retention 天数配置：

```sql
DELETE FROM entries
WHERE user_id=$1 AND status=$2 AND starred is false AND share_code=''
```

它删除某个用户的**全部**非收藏、非分享的已读条目，没有时间窗口参数。

### 22.5 配置优先级总结

```
全局环境变量 (CLEANUP_ARCHIVE_*)
    └──► cli/cleanup_tasks.go 直接消费
            └──► storage.ArchiveEntries() 全表删除
                    └──► 无 per-User / per-Feed 覆盖
```

**User 模型相关字段核查**（`model/user.go:12-46`）：无任何 retention / archive_days 字段。
**Feed 模型相关字段核查**（`model/feed.go:24-77`）：无任何 retention / archive_days 字段。

> 设计意图：Miniflux 将 entry 保留策略视为服务器运维层面的全局配置（类似数据库磁盘配额），而非用户偏好。不同用户如果需要不同保留策略，需部署多个独立实例。

---

## 二十三、Archive Entry 与活跃 Entry 的查询合并路径

### 23.1 核心结论：Miniflux 没有"归档状态"，Archive = 物理删除

这是理解该问题的关键：Miniflux 的 entry **只有两种状态**（`model/entry.go:11-16`）：

```go
const (
    EntryStatusUnread = "unread"
    EntryStatusRead   = "read"
)
```

**不存在 `archived` 状态**。所谓的"归档"（Archive）在代码中就是**物理 DELETE**，不是软删除。被 Archive 的条目从 `entries` 表中永久消失，只在 `entry_tombstones` 表留下 (feed_id, hash) 记录防止爬虫重新摄入。

### 23.2 Entry 查询的唯一来源：`entries` 表

所有 entry 查询都走 `EntryQueryBuilder`，其核心 SQL（`storage/entry_query_builder.go:291-342`）只查一张表：

```sql
SELECT e.id, e.status, e.starred, ...
FROM entries e
INNER JOIN feeds f ON f.id=e.feed_id
INNER JOIN categories c ON c.id=f.category_id
-- JOIN 其他关联表...
WHERE e.user_id = $1 AND [其他条件]
```

没有 `archived_entries` 表、没有 `status='archived'` 过滤条件、也没有 UNION 合并两张表。

### 23.3 用户感知的"归档页"实现方式

用户在 UI 中看到的"历史记录"（History）页面，实际上是**过滤 `status='read'`** 的查询结果，不是从独立的归档存储读取：

查询构建器的状态过滤方法（`entry_query_builder.go:147-157`）：

```go
func (e *EntryQueryBuilder) WithStatuses(statuses ...string) *EntryQueryBuilder {
    if len(statuses) == 1 {
        e.conditions = append(e.conditions, fmt.Sprintf("e.status = $%d", len(e.args)+1))
        e.args = append(e.args, statuses[0])
    } else if len(statuses) > 1 {
        e.conditions = append(e.conditions, fmt.Sprintf("e.status = ANY($%d)", len(e.args)+1))
        e.args = append(e.args, pq.StringArray(statuses))
    }
    return e
}
```

**各视图的状态过滤参数**：

| UI 视图 | WithStatuses 参数 | 实际含义 |
|--------|------------------|---------|
| Unread (未读) | `WithStatuses("unread")` | 未读条目 |
| History (历史) | `WithStatuses("read")` | 已读条目（即用户感知的"归档"） |
| Starred (收藏) | 无状态过滤 + `WithStarred(true)` | 收藏条目（无论读/未读） |
| All (全部) | 无状态过滤（等价于 unread + read） | 所有未删除条目 |

### 23.4 收藏条目对清理的豁免

`ArchiveEntries`（自动清理）和 `FlushHistory`（手动清理）都豁免收藏条目：

```sql
-- ArchiveEntries: storage/entry.go:372-375
WHERE status=$1
  AND starred is false      -- 跳过收藏
  AND share_code=''         -- 跳过已分享
  AND created_at < now() - $2::interval

-- FlushHistory: storage/entry.go:496
WHERE user_id=$1 AND status=$2 AND starred is false AND share_code=''
```

因此**收藏条目永远存在于 `entries` 表中**，直到用户主动取消收藏且过了 retention 周期才会被清理。这就是"收藏夹"视图能查到久远条目的原因——它们从未被归档删除。

### 23.5 聚合查询同样不区分 Archive/Active

Per-Feed 计数（`storage/feed_query_builder.go`）和 Per-Category 计数（`storage/category.go`）同样只查 `entries` 表：

```sql
SELECT e.feed_id, e.status, count(*)
FROM entries e
WHERE e.user_id = $1 AND e.status IN ('unread', 'read')
GROUP BY e.feed_id, e.status
```

被物理删除的条目自然不会出现在计数中。**不存在单独的"已归档计数"**。

### 23.6 查询合并路径全景

```
                        ┌─────────────────────┐
                        │    entries 表        │
                        │  (唯一数据来源)      │
                        └─────────┬───────────┘
                                  │
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
     WithStatuses(unread)  WithStatuses(read)   WithStarred(true)
         (未读视图)           (历史视图)           (收藏视图)
                │                 │                 │
                └─────────────────┴─────────────────┘
                                  │
                                  ▼
                        EntryQueryBuilder.GetEntries()
                     (单次查询，无 UNION，无合并)
                                  │
                                  ▼
                              UI 渲染
```

**关键点**：
- 没有"Archive 表"→"Active 表"的合并，所有数据都在同一张 `entries` 表中
- 归档 = 物理 DELETE，归档后的数据不可查询
- 历史视图本质是 `status='read'` 过滤，不是查询"归档区"
- 收藏条目通过 `starred is false` 条件豁免清理，实现"永久保存"

---

## 二十四、总结

### 状态同步的核心设计原则

1. **统一存储层**：Web、API v1、Fever、Google Reader 四个入口最终全部调用 `internal/storage/entry.go` 中的数据库操作函数，从根源避免不一致。

2. **即时查询无缓存**：未读计数不使用 Redis 或内存缓存，每次需要时执行 `SELECT count(*)` 或 `GROUP BY` 查询，虽然牺牲了部分性能但换取了绝对的一致性。

3. **状态转换的原子性**：
   - `ToggleStarred` 使用 `SET starred = NOT starred` 单 SQL 原子切换
   - `SetEntriesStatusAndCountVisible` 使用 CTE 在同一事务中更新+计数

4. **差异适配而非分叉**：Fever 和 Google Reader 接口通过"协议适配层"将各自的语义（Fever 的 `mark`/`as` 参数，Google Reader 的标签系统）翻译为相同的 Storage 调用，而非各自实现独立的数据库逻辑。

5. **计数获取时序控制**：Web UI 中 `GetNavMetadata` 在状态写入**之后**执行，保证计数准确性；`entry_unread.go` 中的注释 "Fetching the counters here avoids being off by one" 明确说明了这一时序设计。

6. **极简审计设计**：不维护状态迁移历史表，仅通过 `changed_at` 记录最后一次变更时间戳。`removed` 状态已废弃，改为 `entry_tombstones` 表防止删除条目被爬虫复活。

7. **跨设备同步靠客户端**：无服务器端冲突合并逻辑，完全依赖 PostgreSQL 原生 Last-Writer-Wins。同步模型为客户端拉取（Pull），三种协议各有差异：
   - Fever：全量 ID 列表对比 + 分页拉取
   - Google Reader：按 `published_at` 时间窗口增量拉取 + `Updated` 时间戳判断
   - REST API v1：支持 `changed_after` 按状态变更时间过滤

8. **协议适配层的健壮性差异**：
   - Google Reader 实现更健壮：状态应用前先检查当前状态，确保幂等；`kept-unread` 与 `read` 标签互斥检测
   - Fever 实现存在缺陷：`saved`/`unsaved` 使用 `ToggleStarred` 而非显式设置，并发时可能翻转状态；`mark=group&id=0` 忽略 `before` 参数

9. **极简离线策略**：Service Worker 只缓存一个离线提示页，不缓存条目数据。前端状态更新采用"乐观 UI + 即时同步"模式，无本地队列、无延迟同步、无离线暂存。未读计数 delta 使用服务器返回的可见条目数计算，而非客户端本地推算。

10. **两层清理机制**：
    - 自动清理：后台定时调度（默认每天一次），按状态分别归档（已读 60 天、未读 180 天），保护收藏和已分享条目
    - 手动清理：`FlushHistory` 用户主动清空历史，只保护收藏条目
    - 级联删除：Feed 删除时通过 `ON DELETE CASCADE` 自动删除 entries，附属表如 enclosures 同理级联

11. **聚合查询的实时一致性**：Per-Feed 和 Per-Category 计数均为实时 `GROUP BY` 查询，无计数器表、无缓存、无触发器。状态变更提交后，下次聚合查询立即可见。使用覆盖索引（`entries_user_status_feed_idx`）实现 Index Only Scan，保证查询性能。

12. **状态变更不触发通知**：Entry 的 read/unread/starred 状态变更不会触发 Webhook 或第三方通知。只有两类事件会触发通知：用户主动 Save Entry（`save_entry` 事件）和 Feed 刷新发现新条目（`new_entries` 事件）。Webhook URL 支持 per-Feed 覆盖 per-User 配置。

13. **Retention 配置为全局运维参数**：entry 保留天数是服务器级全局配置（环境变量 `CLEANUP_ARCHIVE_*`），不支持 per-User 或 per-Feed 覆盖。`ArchiveEntries` 是全表删除，不区分用户。唯一的用户级清理是 `FlushHistory`，但它是即时手动操作，不涉及 retention 天数。

14. **无归档状态，Archive = 物理删除**：entry 只有 `unread` 和 `read` 两种状态，不存在 `archived` 软删除状态。"归档"就是物理 DELETE，被删条目仅在 `entry_tombstones` 留痕防止复活。UI 中的"历史记录"页面本质是 `status='read'` 过滤，不是查询独立归档存储。收藏条目（`starred=true`）被清理逻辑豁免，实现"永久保存"。

### 注意事项与潜在问题

- **Fever 收藏翻转风险**：`saved`/`unsaved` 操作使用 `ToggleStarred` 而非显式 `SetEntriesStarredState`，存在客户端重复调用时状态意外翻转的风险（`fever/handler.go:447,466`）。代码已经 SELECT 了 entry 但未做条件判断，是可修复的缺陷。

- **Web UI 与 API 范围差异**：Web UI 的 `markAllAsRead` 仅标记全局可见的 Feed（`MarkGloballyVisibleFeedsAsRead`），与 API / Fever / Google Reader 的全局标记语义不同（使用 `MarkAllAsRead`），使用时需注意范围差异。

- **无状态迁移历史**：无法审计"条目 X 在 T1 被设备 A 标为已读，T2 被设备 B 标为未读"的完整时间线，只能知道最后一次变更时间。

- **Google Reader `kept-unread` 语义丢失**：Miniflux 没有"保持未读"中间状态，`add kept-unread` 直接映射为 `SetEntriesStatus(unread)`，与原生协议语义有差异。

- **Google Reader 增量同步限制**：协议实现只支持按 `published_at` 过滤，不支持按 `changed_at` 过滤增量同步，客户端必须拉取条目详情才能发现状态变更。

- **`changed_at` 精度限制**：微秒级精度下，同一微秒内的并发更新无法通过时间戳区分先后。

- **离线状态操作无降级**：Web 端离线时标记已读/收藏会失败且不暂存，刷新后恢复原状态。Service Worker 仅提供离线提示页，不提供完整离线阅读功能。

- **清理调度无并发保护**：`cleanupScheduler` 使用 `time.Tick` 驱动，如果上一轮清理未完成，下一轮会并发执行。虽然数据层面受 `FOR UPDATE SKIP LOCKED` 保护无冲突，但可能产生不必要的资源消耗。

- **聚合查询的非即时性**：标记已读后，导航栏未读计数即时更新（通过返回值），但侧边栏 Feed 列表和分类列表的计数**不会即时更新**，需要刷新页面或重新进入列表页才能看到变化。

- **Save 通知无状态关联**：Fever 的 `saved` 操作使用 `ToggleStarred` 切换收藏后立即触发 `SendEntry`，但发送的 entry 是 Toggle 之前查询的，`entry.Starred` 字段可能与数据库实际状态不一致。同样，Google Reader 批量标记收藏时，SendEntry 使用的是变更前的 entry 对象，状态字段不保证同步。

- **通知推送无重试机制**：所有第三方集成推送（Webhook、Telegram、Pushover 等）都是 fire-and-forget 的 goroutine，无失败重试、无持久化队列、无死信处理。第三方服务临时不可用时，该次推送永久丢失。

- **无 per-User Retention 带来的多租户风险**：`ArchiveEntries` 是全表删除，不区分用户。多用户部署场景下，所有用户共享同一保留周期。重度用户和轻度用户的历史条目保留时间完全相同，无法个性化。如果某用户希望永久保留已读条目，只能通过"全部收藏"这种非预期用法实现。
