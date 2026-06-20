# Miniflux 全文搜索功能深度分析

## 概述

Miniflux 的全文搜索基于 PostgreSQL 原生的全文检索能力（tsvector + tsquery + GIN 索引），实现了高性能的条目内搜索。搜索功能可以与订阅源（Feed）、分类（Category）、标签（Tag）、状态（Status）等过滤条件组合使用，通过 Builder 模式灵活构建 SQL 查询，同时采用窗口函数和 CTE（公用表表达式）分别实现列表分页和条目内前后导航。索引数据在条目创建、更新和抓取时同步刷新，无需额外的后台任务。

---

## 一、核心数据模型与索引结构

### 1.1 数据库表结构

`entries` 表中存储全文搜索所需的向量字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `document_vectors` | `tsvector` | 预计算的全文搜索向量，由标题和内容加权生成 |
| `title` | `text` | 条目标题（权重 A） |
| `content` | `text` | 条目正文（权重 B） |
| `tags` | `text[]` | 标签数组（PostgreSQL 数组） |
| `feed_id` | `bigint` | 关联订阅源 |
| `user_id` | `int` | 所属用户（隔离数据） |

### 1.2 索引创建（迁移脚本）

索引创建位于 `internal/database/migrations.go`：

```sql
-- 迁移步骤 1: 添加 tsvector 列并生成初始数据
ALTER TABLE entries ADD COLUMN document_vectors tsvector;
UPDATE entries SET document_vectors = to_tsvector(substring(title || ' ' || coalesce(content, '') for 1000000));

-- 迁移步骤 2: 创建 GIN 索引（倒排索引）
CREATE INDEX document_vectors_idx ON entries USING gin(document_vectors);

-- 迁移步骤 3: 引入权重系统（标题 A / 内容 B）
UPDATE entries SET document_vectors = 
    setweight(to_tsvector(substring(coalesce(title, '') for 1000000)), 'A') 
    || setweight(to_tsvector(substring(coalesce(content, '') for 1000000)), 'B')
```

**权重策略**：
- 标题（title）→ 权重 **A**（最高，匹配时排名靠前）
- 正文（content）→ 权重 **B**（次高）
- PostgreSQL 默认提供 A/B/C/D 四级权重，数值比例为 1.0 / 0.4 / 0.2 / 0.1

---

## 二、查询构建器（EntryQueryBuilder）

### 2.1 架构设计

`internal/storage/entry_query_builder.go` 中采用 **Builder 模式** 链式调用构建查询。

```
EntryQueryBuilder 结构:
├── store: *Storage           // 数据库连接
├── args: []any               // 参数化查询参数（防 SQL 注入）
├── conditions: []string      // WHERE 条件片段
├── sortExpressions: []string // ORDER BY 片段
├── limit: int                // LIMIT
├── offset: int               // OFFSET
├── fetchEnclosures: bool     // 是否抓取附件
└── excludeContent: bool      // 是否排除 content 列（列表页优化）
```

**初始化**：始终强制用户隔离（`e.user_id = $1`）
```go
func (s *Storage) NewEntryQueryBuilder(userID int64) *EntryQueryBuilder {
    return &EntryQueryBuilder{
        store:      s,
        args:       []any{userID},
        conditions: []string{"e.user_id = $1"},
    }
}
```

### 2.2 全文搜索条件构建

`WithSearchQuery` 方法（`entry_query_builder.go:45-59`）：

```go
func (e *EntryQueryBuilder) WithSearchQuery(query string) *EntryQueryBuilder {
    if query != "" {
        nArgs := len(e.args) + 1
        // 条件: 使用 websearch_to_tsquery 解析用户输入
        e.conditions = append(e.conditions, 
            fmt.Sprintf("e.document_vectors @@ websearch_to_tsquery($%d)", nArgs))
        e.args = append(e.args, query)

        // 排序: ts_rank 相关度 - 时间衰减因子
        e.sortExpressions = append(e.sortExpressions,
            fmt.Sprintf("ts_rank(document_vectors, websearch_to_tsquery($%d)) - extract (epoch from now() - published_at)::float * 0.0000001 DESC", nArgs),
        )
    }
    return e
}
```

**关键技术点**：

1. **`websearch_to_tsquery()`**：PostgreSQL 内置函数，支持类 Google 的搜索语法：
   - `"exact phrase"` → 精确短语匹配
   - `word1 OR word2` → 或逻辑
   - `-word` → 排除词
   - 普通空格 → 隐式 AND

2. **`@@` 操作符**：tsvector 与 tsquery 的匹配运算符，使用 GIN 索引

3. **排序算法（BM25-like 混合排序）**：
   ```
   最终得分 = ts_rank(向量匹配相关度) - 时间衰减
   ```
   - `ts_rank()`：根据词频、权重、逆文档频率计算相关度
   - 时间衰减系数：`0.0000001 ≈ 0.1 / 86400`（每天衰减 0.1 分）
   - 新文章排名天然靠前，旧文章需要更高相关度才能排名靠前

### 2.3 各过滤条件与全文搜索的组合

所有过滤条件均通过 `AND` 连接（`buildCondition()` 使用 `strings.Join(conditions, " AND ")`），形成复合查询。

#### 2.3.1 订阅源过滤（FeedID）

**代码位置**：`entry_query_builder.go:129-136`

```go
func (e *EntryQueryBuilder) WithFeedID(feedID int64) *EntryQueryBuilder {
    if feedID > 0 {
        e.conditions = append(e.conditions, "e.feed_id = $"+strconv.Itoa(len(e.args)+1))
        e.args = append(e.args, feedID)
    }
    return e
}
```

**示例组合**：`WithSearchQuery("golang").WithFeedID(42)`
```sql
WHERE e.user_id = $1 
  AND e.document_vectors @@ websearch_to_tsquery($2)
  AND e.feed_id = $3
```

#### 2.3.2 分类过滤（CategoryID）

**代码位置**：`entry_query_builder.go:138-145`

```go
func (e *EntryQueryBuilder) WithCategoryID(categoryID int64) *EntryQueryBuilder {
    if categoryID > 0 {
        e.conditions = append(e.conditions, "f.category_id = $"+strconv.Itoa(len(e.args)+1))
        e.args = append(e.args, categoryID)
    }
    return e
}
```

注意：分类过滤依赖 `feeds f` 表，查询中已包含 `JOIN feeds f ON f.id=e.feed_id`。

#### 2.3.3 标签过滤（Tags）

**代码位置**：`entry_query_builder.go:159-166`

```go
func (e *EntryQueryBuilder) WithTags(tags ...string) *EntryQueryBuilder {
    if len(tags) > 0 {
        e.conditions = append(e.conditions, 
            fmt.Sprintf("LOWER(e.tags::text)::text[] @> LOWER($%d::text)::text[]", len(e.args)+1))
        e.args = append(e.args, pq.Array(tags))
    }
    return e
}
```

**实现原理**：
- `@>` 运算符：PostgreSQL 数组包含运算符（左侧数组是否包含右侧所有元素）
- `LOWER()` 转换：实现大小写不敏感的标签匹配
- `tags::text` 序列化为字符串后再转数组，规避直接对 `text[]` 用 `LOWER()` 的类型问题

#### 2.3.4 其他常用过滤条件

| 方法 | SQL 条件 | 用途 |
|------|----------|------|
| `WithStatuses("unread")` | `e.status = $X` / `ANY($X)` | 按状态筛选（可多值） |
| `WithStarred(true)` | `e.starred is true` | 仅收藏 |
| `WithGloballyVisible()` | `c.hide_globally IS FALSE AND f.hide_globally IS FALSE` | 全局可见过滤 |
| `BeforeEntryID(id)` | `e.id < $X` | ID 游标分页 |
| `AfterPublishedDate(t)` | `e.published_at > $X` | 时间范围过滤 |

### 2.5 最终 SQL 生成示例

UI 搜索页（`internal/ui/search.go`）的典型调用链：

```go
builder := h.store.NewEntryQueryBuilder(user.ID).
    WithSearchQuery(searchQuery).     // 全文搜索
    WithoutContent().                  // 列表页排除 content，减少数据传输
    WithOffset(offset).                // 分页偏移
    WithLimit(user.EntriesPerPage).    // 每页数量
    WithStatuses(model.EntryStatusUnread)  // 可选：仅未读
```

**生成的完整 SQL**：
```sql
SELECT
    count(*) OVER(),                    -- 窗口函数: 总匹配数（忽略 LIMIT/OFFSET）
    e.id, e.user_id, e.feed_id, ...
    '' AS content,                      -- WithoutContent() 优化
    f.title as feed_title,
    c.title as category_title, ...
FROM entries e
INNER JOIN feeds f ON f.id=e.feed_id
INNER JOIN categories c ON c.id=f.category_id
LEFT JOIN feed_icons fi ON fi.feed_id=f.id
LEFT JOIN icons i ON i.id=fi.icon_id
INNER JOIN users u ON u.id=e.user_id
WHERE e.user_id = $1
  AND e.document_vectors @@ websearch_to_tsquery($2)
  AND e.status = $3
ORDER BY ts_rank(document_vectors, websearch_to_tsquery($2)) 
         - extract(epoch from now() - published_at)::float * 0.0000001 DESC
LIMIT $4 OFFSET $5
```

**关联表的必要性**：
- `feeds f`：分类过滤、订阅源标题显示
- `categories c`：分类信息、全局可见性过滤
- `feed_icons / icons`：订阅源图标
- `users u`：时区转换

---

## 三、分页机制

### 3.1 列表分页：窗口函数一步到位

`GetEntriesWithCount()`（`entry_query_builder.go:278-280`）通过一次查询同时获取结果和总数：

```go
func (e *EntryQueryBuilder) fetchEntries(withCount bool) (model.Entries, int, error) {
    countColumn := ""
    if withCount {
        countColumn = "count(*) OVER(),"  -- 窗口函数
    }
    // ... 执行查询，扫描 totalCount
}
```

**传统方式 vs Miniflux 方式对比**：

| 方式 | 查询次数 | 优点 | 缺点 |
|------|----------|------|------|
| 传统 COUNT + LIMIT/OFFSET | 2 次 | 直观 | 两次查询，一致性问题（并发写入时不一致） |
| **窗口函数 count(*) OVER()** | **1 次** | 原子一致、性能优 | 需要数据库计算所有匹配行（大数据量下有开销，靠 GIN 索引缓解） |

**分页参数传递**（`internal/ui/pagination.go` 配合）：
- `offset`：当前页起始偏移
- `limit = user.EntriesPerPage`：用户配置的每页数量
- `SearchQuery` / `UnreadOnly` 等过滤参数在分页链接中保留（编码到 URL query）

### 3.2 条目详情页的前后导航：CTE + 窗口函数

用户阅读搜索结果中的某条时，需要在**相同过滤条件下**找到上一条/下一条。由 `entryPaginationBuilder`（`internal/storage/entry_pagination_builder.go`）实现。

```go
type entryPaginationBuilder struct {
    db         *sql.DB
    conditions []string  // 与列表查询完全一致的条件
    args       []any
    entryID    int64     // 当前条目 ID
    order      string    // 用户排序字段
    direction  string    // 升/降序
}
```

#### 3.2.1 CTE 查询核心

`getPrevNextID()`（`entry_pagination_builder.go:144-183`）使用 CTE + `lag()/lead()` 窗口函数：

```sql
WITH entry_pagination AS (
    SELECT
        e.id,
        lag(e.id) over (order by e."published_at" asc, e.created_at asc, e.id desc)  as prev_id,
        lead(e.id) over (order by e."published_at" asc, e.created_at asc, e.id desc) as next_id
    FROM entries AS e
    JOIN feeds AS f ON f.id=e.feed_id
    JOIN categories c ON c.id = f.category_id
    WHERE e.user_id = $1                -- 用户隔离
      AND e.document_vectors @@ websearch_to_tsquery($2)  -- 全文搜索
      AND f.category_id = $3            -- 分类过滤（如果有）
      -- ... 其他条件（feed_id、status、tags 等）
    ORDER BY e."published_at" asc, e.created_at asc, e.id desc
)
SELECT prev_id, next_id FROM entry_pagination AS ep WHERE ep.id = $4;
```

**关键点**：
1. `lag(id) OVER (ORDER BY ...)`：取排序后前一行的 id
2. `lead(id) OVER (ORDER BY ...)`：取排序后后一行的 id
3. CTE 必须与原列表**使用完全相同的 WHERE 条件和 ORDER BY**，确保前后条目一致
4. `entryPaginationBuilder` 提供了完整的条件镜像：`WithSearchQuery`、`WithFeedID`、`WithCategoryID`、`WithTags`、`WithStatus` 等，与 `EntryQueryBuilder` 对称

#### 3.2.2 全文搜索 + 分类 + 仅未读的完整调用链

以 `internal/ui/entry_search.go` 为例：

```go
// 1. 构建分页 builder（镜像过滤条件）
entryPaginationBuilder := h.store.NewEntryPaginationBuilder(user.ID, entry.ID, user.EntryOrder, user.EntryDirection).
    WithSearchQuery(searchQuery)      // ← 全文搜索

// 2. 叠加状态过滤（未读模式）
if unreadOnly {
    if entry.Status == model.EntryStatusRead {
        // 当前条目已读时，允许它出现在列表中（避免丢失上下文）
        entryPaginationBuilder = entryPaginationBuilder.WithStatusOrEntryID(
            model.EntryStatusUnread, entry.ID)
    } else {
        entryPaginationBuilder = entryPaginationBuilder.WithStatus(model.EntryStatusUnread)
    }
}

// 3. 查询前后条目
prevEntry, nextEntry, err := entryPaginationBuilder.Entries()
```

**排序注意事项**：搜索结果列表页按 `ts_rank - 时间衰减` 排序，但条目详情的前后导航使用用户配置的 `user.EntryOrder`（如 `published_at`），因为 `ts_rank` 需要额外参数且在分页时不够稳定。

---

## 四、索引维护机制（动态刷新）

Miniflux 不使用 PostgreSQL 的触发器自动更新 `document_vectors`，而是在**应用层写入时显式计算**，便于控制截断策略和兼容不同 PostgreSQL 版本。

### 4.1 新条目创建时的索引生成

`createEntry()`（`internal/storage/entry.go:80-161`）：

```go
func (s *Storage) createEntry(tx *sql.Tx, entry *model.Entry) error {
    // 1. 截断（PostgreSQL tsvector < 1MB 限制）
    truncatedTitle, truncatedContent := truncateTitleAndContentForTSVectorField(
        entry.Title, entry.Content)

    // 2. INSERT 时同步计算 document_vectors
    query := `
        INSERT INTO entries (..., document_vectors, ...)
        SELECT ...,
            setweight(to_tsvector($11), 'A') || setweight(to_tsvector($12), 'B'),
            --        ↑ 截断后的标题(权重A)        ↑ 截断后的正文(权重B)
            $13 ...
    `
    // 参数: $11=truncatedTitle, $12=truncatedContent
}
```

### 4.2 订阅源刷新时更新索引

`updateEntry()`（`internal/storage/entry.go:166-210`）：订阅源重新抓取到内容变化时：

```go
func (s *Storage) updateEntry(tx *sql.Tx, entry *model.Entry) error {
    truncatedTitle, truncatedContent := truncateTitleAndContentForTSVectorField(
        entry.Title, entry.Content)
    query := `
        UPDATE entries SET
            title=$1, content=$4, ...,
            document_vectors = setweight(to_tsvector($7), 'A') || setweight(to_tsvector($8), 'B'),
            --              ↑ 重新计算
            tags=$12
        WHERE user_id=$9 AND feed_id=$10 AND hash=$11
    `
}
```

触发入口：`RefreshFeedEntries()` → `updateEntry()`

### 4.3 用户手动更新条目内容

`UpdateEntryTitleAndContent()`（`internal/storage/entry.go:50-78`）：用户通过 API 修改条目内容时：

```go
func (s *Storage) UpdateEntryTitleAndContent(entry *model.Entry) error {
    truncatedTitle, truncatedContent := truncateTitleAndContentForTSVectorField(
        entry.Title, entry.Content)
    query := `
        UPDATE entries SET
            title=$1, content=$2, reading_time=$3,
            document_vectors = setweight(to_tsvector($4), 'A') || setweight(to_tsvector($5), 'B')
        WHERE id=$6 AND user_id=$7
    `
}
```

触发入口：
- API `updateEntryHandler`（用户手动编辑条目）
- `fetchContentHandler` + `update_content=true`（抓取完整网页后更新）

### 4.4 截断策略（tsvector 1MB 限制）

PostgreSQL 的 `tsvector` 单个值最大为 1MB。长文章需截断：

```go
func truncateTitleAndContentForTSVectorField(title, content string) (string, string) {
    // 标题最多索引前 200,000 字符 ≈ 200KB
    // 正文最多索引前 500,000 字符 ≈ 500KB
    // 合计留足 300KB 给分词后的位置和权重信息
    return truncateStringForTSVectorField(title, 200000),
           truncateStringForTSVectorField(content, 500000)
}

func truncateStringForTSVectorField(s string, maxSize int) string {
    // ... 按 UTF-8 字符边界截断，不破坏多字节字符
}
```

**设计取舍**：
- 不索引全文而是取头部，牺牲尾部词的可搜索性
- 大多数文章核心信息位于前 500KB 文本中
- 避免 PostgreSQL `string is too long for tsvector` 错误

### 4.5 索引维护时序图

```
[订阅源刷新] 或 [API 写入]
        │
        ▼
┌─────────────────────┐
│ RefreshFeedEntries  │  InsertEntryForFeed  updateEntryHandler  fetchContentHandler
│ (每篇条目独立事务)  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────────────────────┐
│ 截断标题+内容 (UTF-8 安全边界)      │
│ truncateTitleAndContentForTSVector  │
└─────────┬───────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────┐
│ SQL: setweight(to_tsvector(title), 'A')              │
│   || setweight(to_tsvector(content), 'B')            │
│   = document_vectors（GIN 索引自动维护）              │
└─────────┬────────────────────────────────────────────┘
          │
          ▼
    PostgreSQL 自动更新
    document_vectors_idx GIN 索引
   （倒排表 + 词位图增量写入）
```

**GIN 索引维护由 PostgreSQL 自动处理**：当 `document_vectors` 列被 UPDATE/INSERT 时，GIN 索引 `document_vectors_idx` 自动维护倒排列表，无需应用层干预。

---

## 五、各入口点调用链汇总

### 5.1 Web UI 搜索流程

```
用户输入搜索词 q="golang tutorial"
        │
        ▼
GET /search?q=golang+tutorial&unread=1
        │
        ▼
internal/ui/search.go → showSearchPage()
  │
  ├── NewEntryQueryBuilder(user.ID)
  │     .WithSearchQuery("golang tutorial")
  │     .WithStatuses("unread")
  │     .WithoutContent()
  │     .WithOffset(0).WithLimit(100)
  │     .GetEntriesWithCount()
  │
  └── 渲染 search.html 模板（带分页链接）
        │
        ▼
  用户点击某条结果
        │
        ▼
GET /search/entry/12345?q=golang+tutorial&unread=1
        │
        ▼
internal/ui/entry_search.go → showSearchEntryPage()
  │
  ├── 查询单条（带 q 参数的 EntryQueryBuilder）
  ├── NewEntryPaginationBuilder(...)
  │     .WithSearchQuery("golang tutorial")
  │     .WithStatusOrEntryID("unread", 12345)
  │     .Entries()  ← CTE + lag/lead
  │
  └── 渲染 entry.html（带上一条/下一条链接）
```

### 5.2 JSON API 搜索流程

```
GET /v1/entries?search=golang&feed_id=42&category_id=7&tags=tech,news
        │
        ▼
internal/api/entry_handlers.go → getEntriesHandler → findEntries()
  │
  ├── 参数解析: search=golang, feed_id=42, category_id=7, tags=[tech,news]
  │
  ├── NewEntryQueryBuilder(user.ID)
  │     .WithFeedID(42)
  │     .WithCategoryID(7)
  │     .WithTags("tech", "news")
  │     .WithSearchQuery("golang")     ← configureFilters() 中调用
  │     .WithOffset(0).WithLimit(100)
  │     .GetEntriesWithCount()
  │
  └── 返回 JSON: { total: 42, entries: [...] }
```

### 5.3 API 中其他可组合的过滤参数

`configureFilters()`（`api/entry_handlers.go:564-613`）中支持：
- `before_entry_id` / `after_entry_id`：ID 游标
- `before` / `after` 或 `published_before` / `published_after`：发布时间范围
- `changed_before` / `changed_after`：变更时间范围
- `category_id`：分类
- `starred`：是否收藏
- `search`：全文搜索 ← 可与任意过滤条件自由叠加

---

## 六、关键文件索引

| 文件 | 职责 |
|------|------|
| `internal/storage/entry_query_builder.go` | 查询构建器：全文搜索、条件组合、排序权重、列表获取 |
| `internal/storage/entry_pagination_builder.go` | 条目前后导航：CTE + lag/lead 窗口函数 |
| `internal/storage/entry.go` | 条目 CRUD：创建/更新时计算并写入 document_vectors |
| `internal/database/migrations.go` | 数据库迁移：document_vectors 列 + GIN 索引创建 |
| `internal/ui/search.go` | UI 搜索列表页入口 |
| `internal/ui/entry_search.go` | UI 搜索结果详情页（含前后导航） |
| `internal/api/entry_handlers.go` | API 入口（findEntries + configureFilters） |
| `internal/ui/tag_entries_all.go` | 标签过滤列表页（WithTags 使用示例） |

---

## 七、性能与设计要点总结

### 7.1 性能优化手段

1. **GIN 倒排索引**：`document_vectors_idx` 使用 PostgreSQL GIN，支持 `@@` 匹配命中
2. **列表页 WithoutContent()**：排除大字段 content，显著减少网络传输
3. **窗口函数 count(*) OVER()**：一次 SQL 同时获取分页数据和总数
4. **参数化查询**：所有条件通过 `$1, $2...` 占位符，防 SQL 注入并启用查询计划缓存
5. **多条件联合时的索引策略**：
   - 全文搜索优先使用 GIN 索引过滤候选集
   - 其余条件（user_id、feed_id、status）在候选集上二次过滤
   - PostgreSQL 查询优化器根据选择性自动选择最优执行路径

### 7.2 设计优点

1. **Builder 模式**：条件正交，可自由组合（搜索+分类+标签+时间范围任意叠加）
2. **应用层计算 tsvector**：显式控制截断和权重，避免触发器带来的调试困难
3. **双查询构建器对称设计**：`EntryQueryBuilder` ↔ `entryPaginationBuilder` 条件一一对应，确保列表与详情导航一致
4. **用户隔离硬编码**：NewEntryQueryBuilder(userID) 始终注入 `e.user_id = $1`，防止多租户数据泄露

### 7.3 权衡与局限

1. **内容截断**：超过 500KB 的正文尾部不可搜索（通常可接受）
2. **前后导航排序差异**：详情导航不用 ts_rank 排序，可能与列表排序顺序不完全一致
3. **无实时高亮**：搜索结果列表仅排序，不提供匹配关键词高亮（需 `ts_headline()` 额外扩展）
4. **无词形还原/同义词配置**：使用 PostgreSQL 默认 `english` 或默认文本搜索配置，不支持多语言精细化配置
