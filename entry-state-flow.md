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

## 十二、总结

### 状态同步的核心设计原则

1. **统一存储层**：Web、API v1、Fever、Google Reader 四个入口最终全部调用 `internal/storage/entry.go` 中的数据库操作函数，从根源避免不一致。

2. **即时查询无缓存**：未读计数不使用 Redis 或内存缓存，每次需要时执行 `SELECT count(*)` 或 `GROUP BY` 查询，虽然牺牲了部分性能但换取了绝对的一致性。

3. **状态转换的原子性**：
   - `ToggleStarred` 使用 `SET starred = NOT starred` 单 SQL 原子切换
   - `SetEntriesStatusAndCountVisible` 使用 CTE 在同一事务中更新+计数

4. **差异适配而非分叉**：Fever 和 Google Reader 接口通过"协议适配层"将各自的语义（Fever 的 `mark`/`as` 参数，Google Reader 的标签系统）翻译为相同的 Storage 调用，而非各自实现独立的数据库逻辑。

5. **计数获取时序控制**：Web UI 中 `GetNavMetadata` 在状态写入**之后**执行，保证计数准确性；`entry_unread.go` 中的注释 "Fetching the counters here avoids being off by one" 明确说明了这一时序设计。

### 注意事项与潜在问题

- Fever 的 `saved`/`unsaved` 操作使用 `ToggleStarred` 而非显式 `SetEntriesStarredState`，存在客户端重复调用时状态意外翻转的风险（`fever/handler.go:447,466`）。
- Web UI 的 `markAllAsRead` 仅标记全局可见的 Feed，与 API / Fever / Google Reader 的全局标记语义不同，使用时需注意范围差异。
