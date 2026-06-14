# 全文搜索索引机制分析

## 概述

Miniflux 使用 PostgreSQL 内置的全文搜索功能，通过 `tsvector` 类型和 GIN 索引实现高效的全文检索。本文档分析全文索引的写入流程和查询过程。

---

## 一、数据库索引定义

### 1.1 索引列定义

在 `entries` 表中，`document_vectors` 列存储全文索引向量：

```sql
document_vectors tsvector
```

该列在第 20 次数据库迁移中添加（`internal/database/migrations.go:280`）。

### 1.2 GIN 索引

使用 GIN（Generalized Inverted Index）索引加速全文搜索：

```sql
CREATE INDEX document_vectors_idx ON entries USING gin(document_vectors);
```

索引的演变：
- 初始创建时为全量索引
- 第 75 次迁移曾改为部分索引（排除 `removed` 状态的条目）
- 第 77 次迁移恢复为全量索引，因为 `removed` 状态已被弃用

### 1.3 权重设置

索引向量采用权重分级策略：
- **标题（Title）**：权重 `A`（最高权重）
- **内容（Content）**：权重 `B`（次高权重）

权重设置在第 21 次迁移中引入（`internal/database/migrations.go:297`）。

---

## 二、索引写入流程

全文索引在条目创建和更新时自动写入，涉及三个主要函数。

### 2.1 创建条目时写入

**函数**：`createEntry`（`internal/storage/entry.go:81`）

```sql
INSERT INTO entries (
    title, content, ..., document_vectors
) VALUES (
    $1, $6, ...,
    setweight(to_tsvector($11), 'A') || setweight(to_tsvector($12), 'B')
)
```

关键参数：
- `$11` - 截断后的标题（最大 200,000 字节）
- `$12` - 截断后的内容（最大 500,000 字节）

### 2.2 更新条目时写入

**函数**：`updateEntry`（`internal/storage/entry.go:166`）

```sql
UPDATE entries SET
    title=$1,
    content=$4,
    ...
    document_vectors = setweight(to_tsvector($7), 'A') || setweight(to_tsvector($8), 'B'),
    ...
WHERE user_id=$9 AND feed_id=$10 AND hash=$11
```

### 2.3 更新标题和内容时写入

**函数**：`UpdateEntryTitleAndContent`（`internal/storage/entry.go:51`）

当用户手动编辑条目内容或通过爬虫获取全文时调用。

### 2.4 文本截断处理

**函数**：`truncateTitleAndContentForTSVectorField`（`internal/storage/entry.go:679`）

由于 PostgreSQL 的 `tsvector` 大小限制为 1MB，需要对输入文本进行截断：

| 字段   | 最大长度     | 说明                                   |
|--------|-------------|----------------------------------------|
| 标题   | 200,000 字节 | 约 200KB，为位置信息预留空间           |
| 内容   | 500,000 字节 | 约 500KB，为位置信息预留空间           |

截断时确保 UTF-8 字符边界完整，不会截断多字节字符的中间部分（`internal/storage/entry.go:686-707`）。

### 2.5 触发索引写入的场景

1. **Feed 刷新**：`RefreshFeedEntries` → `createEntry` / `updateEntry`
2. **单条插入**：`InsertEntryForFeed` → `createEntry`
3. **内容更新**：`UpdateEntryTitleAndContent` → 直接更新
4. **API 导入**：`importFeedEntryHandler` → `InsertEntryForFeed`
5. **网页抓取**：`fetchContentHandler` → `UpdateEntryTitleAndContent`

---

## 三、全文查询过程

### 3.1 查询构建器

**类**：`EntryQueryBuilder`（`internal/storage/entry_query_builder.go:20`）

全文搜索通过 `WithSearchQuery` 方法添加查询条件。

### 3.2 搜索条件构造

**方法**：`WithSearchQuery`（`internal/storage/entry_query_builder.go:46`）

```go
func (e *EntryQueryBuilder) WithSearchQuery(query string) *EntryQueryBuilder {
    if query != "" {
        nArgs := len(e.args) + 1
        e.conditions = append(e.conditions, 
            fmt.Sprintf("e.document_vectors @@ websearch_to_tsquery($%d)", nArgs))
        e.args = append(e.args, query)
        
        // 排序：相关度 - 时间衰减
        e.sortExpressions = append(e.sortExpressions,
            fmt.Sprintf("ts_rank(document_vectors, websearch_to_tsquery($%d)) - extract (epoch from now() - published_at)::float * 0.0000001 DESC", nArgs),
        )
    }
    return e
}
```

### 3.3 核心查询操作符

使用 PostgreSQL 的全文搜索操作符：

| 操作符/函数 | 说明 |
|------------|------|
| `@@` | tsvector 与 tsquery 的匹配操作符 |
| `websearch_to_tsquery()` | 将 Web 风格的搜索查询转换为 tsquery |
| `ts_rank()` | 计算匹配相关度得分 |

### 3.4 排序算法

搜索结果采用**相关度与时间衰减结合**的排序策略：

```
排序值 = ts_rank(文档向量, 查询向量) - (当前时间 - 发布时间) * 0.0000001
```

公式说明：
- `ts_rank()` 计算文本匹配相关度，值越大越相关
- 时间衰减系数 `0.0000001` ≈ 0.1 / 86400（秒/天），即每天约衰减 0.1 分
- 新文章因时间近而排名靠前，平衡了相关性和时效性

### 3.5 分页构建器中的搜索

**类**：`entryPaginationBuilder`（`internal/storage/entry_pagination_builder.go:18`）

在搜索结果中浏览上一条/下一条时，同样使用全文索引进行过滤：

```go
func (e *entryPaginationBuilder) WithSearchQuery(query string) *entryPaginationBuilder {
    if query != "" {
        e.conditions = append(e.conditions, 
            fmt.Sprintf("e.document_vectors @@ websearch_to_tsquery($%d)", len(e.args)+1))
        e.args = append(e.args, query)
    }
    return e
}
```

---

## 四、搜索入口点

### 4.1 Web UI 搜索

**文件**：`internal/ui/search.go:15`

处理 `/search` 页面请求，支持：
- 搜索关键词（`q` 参数）
- 仅显示未读（`unread` 参数）
- 分页偏移（`offset` 参数）

调用链：
```
showSearchPage → NewEntryQueryBuilder → WithSearchQuery → GetEntriesWithCount
```

### 4.2 搜索结果中的条目详情

**文件**：`internal/ui/entry_search.go:15`

处理 `/search/entry/{entryID}` 页面，保持搜索上下文用于翻页。

### 4.3 REST API 搜索

**文件**：`internal/api/entry_handlers.go:608`

API 通过 `search` 查询参数支持全文搜索：

```go
func configureFilters(builder *storage.EntryQueryBuilder, r *http.Request) *storage.EntryBuilder {
    // ...
    if searchQuery := request.QueryStringParam(r, "search", ""); searchQuery != "" {
        builder = builder.WithSearchQuery(searchQuery)
    }
    return builder
}
```

---

## 五、技术要点总结

### 5.1 索引写入时机

- **同步写入**：索引在条目创建/更新时同步计算并写入
- **事务保证**：索引写入与数据写入在同一事务中，保证一致性

### 5.2 性能优化

1. **GIN 索引**：使用倒排索引加速全文匹配
2. **文本截断**：限制索引大小，避免超出 PostgreSQL 限制
3. **权重分级**：标题权重高于内容，提升搜索准确性
4. **时间衰减排序**：平衡相关性和时效性

### 5.3 查询语法

使用 PostgreSQL 的 `websearch_to_tsquery`，支持：
- 简单关键词搜索
- 引号精确匹配
- OR 操作符
- 排除（减号前缀）

### 5.4 相关文件清单

| 文件路径 | 主要职责 |
|---------|---------|
| `internal/database/migrations.go` | 数据库迁移，定义索引结构 |
| `internal/storage/entry.go` | 索引写入逻辑（创建/更新） |
| `internal/storage/entry_query_builder.go` | 查询构建，搜索条件与排序 |
| `internal/storage/entry_pagination_builder.go` | 搜索结果分页导航 |
| `internal/ui/search.go` | Web UI 搜索页面 |
| `internal/ui/entry_search.go` | Web UI 搜索条目详情 |
| `internal/api/entry_handlers.go` | REST API 搜索接口 |
