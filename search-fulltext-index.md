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

### 1.4 文本搜索配置（Text Search Config）

Miniflux 使用 PostgreSQL 默认的文本搜索配置，未显式指定配置名称。

**代码特征**：所有 `to_tsvector()` 和 `websearch_to_tsquery()` 调用均省略了 `config` 参数。

```sql
-- 实际调用（省略 config 参数）
to_tsvector($1)
websearch_to_tsquery($1)

-- 等价于使用默认配置
to_tsvector(current_setting('default_text_search_config'), $1)
```

**PostgreSQL 默认配置**：`pg_catalog.english`

| 配置类型 | 说明 | 适用场景 |
|---------|------|---------|
| `pg_catalog.english` | 英语词干提取 + 停用词过滤 | 英文内容，支持时态/单复数归一化 |
| `pg_catalog.simple` | 仅按空格和标点分词，无归一化 | 多语言混合场景，不做词干处理 |

**`english` vs `simple` 的差异**：
- `english`：`"running"` → `"run"`，`"the"` 被过滤，`"don't"` → `"don't"` + `"not"`
- `simple`：`"running"` → `"running"`，`"the"` → `"the"`，保留原始形态

### 1.5 多语言支持的局限性

#### 1.5.1 中文与日文的 Token 化问题

PostgreSQL 内置的 `english` 和 `simple` 配置对中文和日文的分词效果**极差**：

**根本原因**：
- 中文和日文没有空格作为词边界
- 默认分词器按字符切分，导致索引颗粒度为单字符

**示例对比**：

| 输入文本 | `english` 配置分词结果 | 理想分词结果 |
|---------|----------------------|------------|
| `Miniflux 是一个开源的 RSS 阅读器` | `'miniflux':1 '一':2 '个':3 '开':4 '源':5 '的':6 '阅':7 '读':8 '器':9` | `'miniflux':1 '一个':2 '开源':3 'rss':4 '阅读器':5` |
| `Miniflux はオープンソースのRSSリーダーです` | `'miniflux':1 'は':2 'オ':3 'ー':4 'プ':5 'ン':6 'ソ':7 'ー':8 'ス':9 ...` | `'miniflux':1 'オープンソース':2 'rss':3 'リーダー':4` |

**实际影响**：
1. **搜索准确率低**：搜索 `"开源"` 可能匹配到 `"开"` 或 `"源"` 单独出现的内容
2. **索引膨胀**：单字符索引导致 GIN 索引体积增大 5-10 倍
3. **查询性能下降**：高频单字符（如 `"的"`、`"是"`）匹配文档过多，查询变慢

#### 1.5.2 改进方案（需手动配置）

PostgreSQL 本身不内置中文/日文分词，需要安装第三方扩展：

| 语言 | 推荐扩展 | 说明 |
|-----|---------|------|
| 中文 | `pg_jieba` / `zhparser` / `pg_bigm` | 基于结巴分词/中文分词库 |
| 日文 | `mecab` / `pg_jieba` | 基于 MeCab 日语分词器 |

**Miniflux 目前未提供**这些扩展的自动配置支持。

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

### 3.3.1 查询边界处理

#### 空查询处理

**代码位置**：`internal/storage/entry_query_builder.go:47`

```go
func (e *EntryQueryBuilder) WithSearchQuery(query string) *EntryQueryBuilder {
    if query != "" {  // ← 空字符串直接跳过
        // ... 添加搜索条件
    }
    return e
}
```

**行为**：
- 空字符串 `""`、仅空白字符的查询不会触发全文搜索
- 返回结果等同于无搜索条件的普通条目列表
- **不会抛错**，静默降级为常规查询

#### 超长查询处理

**前端限制**：搜索表单未设置 `maxlength` 属性（`internal/template/templates/views/search.html:13`）

**后端限制**：
- 代码中**无显式长度校验**
- 依赖 PostgreSQL `websearch_to_tsquery()` 的内部限制
- PostgreSQL 对 tsquery 长度有隐含限制（约几百 KB）
- 极端超长查询可能导致 PostgreSQL 返回语法错误

#### 特殊字符与 SQL 关键字处理

**安全机制**：参数化查询

```go
// 安全：使用 $N 占位符，由 lib/pq 驱动处理转义
e.conditions = append(e.conditions, 
    fmt.Sprintf("e.document_vectors @@ websearch_to_tsquery($%d)", nArgs))
e.args = append(e.args, query)  // query 作为参数传递，非字符串拼接
```

**处理方式**：
- **SQL 关键字**（如 `SELECT`、`DROP`）：作为普通文本处理，不会触发 SQL 注入
- **特殊字符**（如 `'`、`"`、`\`、`;`）：由 PostgreSQL 驱动自动转义
- **`websearch_to_tsquery` 语法字符**（`"`、`-`、`OR`、`(`、`)`）：具有特殊语义，不转义

**语法错误处理**：
- 非法语法（如未闭合引号 `'"unclosed`）会导致 PostgreSQL 返回 `syntax error in tsquery`
- 错误会向上传播，最终由 `response.HTMLServerError` 返回 500 错误页面
- **无专门的捕获和友好提示**

### 3.4 排序算法

搜索结果采用**相关度与时间衰减结合**的排序策略：

```
排序值 = ts_rank(文档向量, 查询向量) - (当前时间 - 发布时间) * 0.0000001
```

公式说明：
- `ts_rank()` 计算文本匹配相关度，值越大越相关
- 时间衰减系数 `0.0000001` ≈ 0.1 / 86400（秒/天），即每天约衰减 0.1 分
- 新文章因时间近而排名靠前，平衡了相关性和时效性

#### 3.4.1 时间衰减系数硬编码位置

**硬编码位置**：
- 主查询：`internal/storage/entry_query_builder.go:52-55`
- 注释说明：`// 0.0000001 = 0.1 / (seconds_in_a_day)`

```go
// internal/storage/entry_query_builder.go:52-55
// 0.0000001 = 0.1 / (seconds_in_a_day)

e.sortExpressions = append(e.sortExpressions,
    fmt.Sprintf("ts_rank(document_vectors, websearch_to_tsquery($%d)) - extract (epoch from now() - published_at)::float * 0.0000001 DESC", nArgs),
)
```

**相同硬编码的两处位置**：
1. `entry_query_builder.go:55` - 主搜索排序
2. `entry_pagination_builder.go` - 分页导航（仅用于过滤，不涉及排序）

#### 3.4.2 衰减系数的含义

| 系数值 | 每天衰减量 | 物理意义 |
|-------|-----------|---------|
| `0.0000001` | 0.1 分/天 | 相关性得分每天降低 0.1 |
| `0.000001` | 1 分/天 | 相关性得分每天降低 1.0 |
| `0` | 0 分/天 | 纯相关度排序，无时间衰减 |

**业务含义**：
- 30 天前的文章需要比新文章相关度高 3 分才能排到相同位置
- 对于高时效性内容（如新闻），当前系数可能偏保守
- 对于技术文档类内容，当前系数可能偏激进

#### 3.4.3 运营调整方式

**当前状态**：**完全硬编码，无可配置项**

配置文件 `internal/config/options.go` 中无搜索相关配置项。

**调整方法（需修改源码）**：

```go
// 需修改 internal/storage/entry_query_builder.go
// 方案1：直接修改常量
const timeDecayFactor = "0.000001"  // 改为 1 分/天

// 方案2：改为可配置参数（需新增配置项）
e.sortExpressions = append(e.sortExpressions,
    fmt.Sprintf("ts_rank(document_vectors, websearch_to_tsquery($%d)) - extract (epoch from now() - published_at)::float * %s DESC", 
        nArgs, config.Opts.SearchTimeDecayFactor()),
)
```

**场景化调整建议**：

| 运营场景 | 推荐系数 | 理由 |
|---------|---------|------|
| 新闻资讯类 | `0.000001` ~ `0.00001` | 强调时效性，新内容优先 |
| 技术博客类 | `0.0000001` | 平衡质量与时效 |
| 知识库/文档 | `0` ~ `0.00000001` | 质量优先，时效权重极低 |

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

### 5.3 查询边界与安全

**空查询**：
- `query != ""` 校验，空字符串静默降级为常规查询，不抛错

**超长查询**：
- 前端无 `maxlength` 限制，后端无长度校验
- 依赖 PostgreSQL 内部限制，极端超长可能触发语法错误

**特殊字符与 SQL 注入防护**：
- 使用参数化查询 `$N` 占位符，由 lib/pq 驱动自动转义
- SQL 关键字（`SELECT`、`DROP`）作为普通文本处理，无注入风险
- `websearch_to_tsquery` 语法字符（`"`、`-`、`OR`）保留特殊语义
- 语法错误（如未闭合引号）返回 500 错误，无友好提示

### 5.4 时间衰减系数

**硬编码位置**：`internal/storage/entry_query_builder.go:55`
- 系数值：`0.0000001`，注释说明 `0.1 / (seconds_in_a_day)`
- 衰减速率：每天约 0.1 分

**运营调整**：
- 当前无可配置项，需修改源码
- 建议按场景调整：
  - 新闻资讯：`0.000001` ~ `0.00001`（强调时效性）
  - 技术博客：`0.0000001`（默认值，平衡质量与时效）
  - 知识库：`0` ~ `0.00000001`（质量优先）

### 5.5 多语言支持现状

**文本搜索配置**：
- 使用 PostgreSQL 默认配置 `pg_catalog.english`
- 支持英语词干提取（`running` → `run`）和停用词过滤
- 未使用 `simple` 配置（无归一化）

**中文/日文局限性**：
- 默认分词器按字符切分，搜索准确率极低
- 索引体积膨胀 5-10 倍，查询性能下降
- 需安装第三方扩展：`pg_jieba`（中/日）、`zhparser`（中）、`mecab`（日）
- Miniflux 未提供自动配置支持

### 5.6 查询语法

使用 PostgreSQL 的 `websearch_to_tsquery`，支持：
- 简单关键词搜索
- 引号精确匹配
- OR 操作符
- 排除（减号前缀）

### 5.7 相关文件清单

| 文件路径 | 主要职责 |
|---------|---------|
| `internal/database/migrations.go` | 数据库迁移，定义索引结构 |
| `internal/storage/entry.go` | 索引写入逻辑（创建/更新） |
| `internal/storage/entry_query_builder.go` | 查询构建，搜索条件与排序 |
| `internal/storage/entry_pagination_builder.go` | 搜索结果分页导航 |
| `internal/ui/search.go` | Web UI 搜索页面 |
| `internal/ui/entry_search.go` | Web UI 搜索条目详情 |
| `internal/api/entry_handlers.go` | REST API 搜索接口 |
