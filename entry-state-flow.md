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

## 十五、总结

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
