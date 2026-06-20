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

### 1.2 索引创建与权重重建（迁移脚本完整历史）

`internal/database/migrations.go` 中的 `migrations` 数组是**按序号递增**的迁移列表（注释"Order is important. Add new migrations at the end of the list."），与全文搜索相关的迁移共有 **4 次**，覆盖了从引入到权重重建、再到索引裁剪的完整生命周期：

| 迁移序号（数组下标） | 所在行号 | 操作类型 | 具体内容 |
|---------------------|---------|---------|---------|
| **#21** | `migrations.go:278` | **首次引入** | 新增 `document_vectors tsvector` 列；全量 UPDATE 生成初始向量；创建全量 GIN 索引 |
| **#23** | `migrations.go:292` | **权重重建** | 全量 UPDATE 重建全部 `document_vectors`，从"标题+空格+正文混为一体（默认权重 D）"改为"标题权重 A + 正文权重 B 分离" |
| **#…** | — | — | 中间多次迁移不涉及全文搜索（users/feeds 等表结构变更） |
| **#N（接近末尾）** | `migrations.go:1400` | **索引裁剪** | DROP 原全量索引，重建 `WHERE status != 'removed'` 的**部分索引**，为已删除条目节省索引空间 |

#### 1.2.1 迁移 #21 — 初始引入（无权重，标题正文混合）

```sql
ALTER TABLE entries ADD COLUMN document_vectors tsvector;

-- ★ 注意: 标题和内容用空格拼接后统一 to_tsvector，未用 setweight
-- 所有词默认落在权重 D（0.1），标题与正文无区分度
UPDATE entries
   SET document_vectors = to_tsvector(
           substring(title || ' ' || coalesce(content, '') for 1000000)
           --        ↑ 硬编码 1,000,000 字符（≈1MB 原文，≈占满 tsvector 上限）
       );

CREATE INDEX document_vectors_idx ON entries USING gin(document_vectors);
```

**此版本的缺陷**：
- `title` 和 `content` 拼接后统一处理，词在标题中出现和在正文中出现权重**完全相同**（都是默认 D = 0.1）
- `substring(... for 1000000)` 是**字符数**不是字节数，对中文（UTF-8 3字节/字）实际约 3MB 原文，可能触发 PostgreSQL `string is too long for tsvector` 报错（当时 1MB 限制尚未在应用层强制截断）

#### 1.2.2 迁移 #23 — 权重重建（标题 A / 正文 B 分离）

这是**最关键的权重迁移**，一次性重写全表所有 `document_vectors`：

```sql
UPDATE entries
   SET document_vectors =
           setweight(to_tsvector(substring(coalesce(title, '') for 1000000)), 'A')
           --        ↑ 标题单独分词，赋权重 A（1.0）
        ||
           setweight(to_tsvector(substring(coalesce(content, '') for 1000000)), 'B')
           --        ↑ 正文单独分词，赋权重 B（0.4）
```

**代码证据**：`migrations.go:294-297` 的 UPDATE 语句直接出现在迁移数组第 23 个元素（紧接 #21 的 `username/password` 列添加之后、`user_agent` 列添加之前）。

**此迁移前后 ts_rank 得分比例变化**（以词 "golang" 为例）：

| 出现位置 | 迁移 #21 前（混合 D） | 迁移 #23 后（分离 A/B） |
|---------|---------------------|------------------------|
| 仅标题出现 1 次 | 词频 × 0.1 | 词频 × 1.0（**提升 10 倍**） |
| 仅正文出现 1 次 | 词频 × 0.1 | 词频 × 0.4（**提升 4 倍**） |
| 标题 1 次 + 正文 1 次 | 2 × 0.1 = 0.2 | 1.0 + 0.4 = 1.4（**提升 7 倍**） |

**注意事项**：
- 此次迁移为**全表 UPDATE**，对大数据库是重量级操作（产生大量 WAL、锁表、触发 GIN 索引全量重建）
- 迁移未使用 `CREATE INDEX CONCURRENTLY`，在 Miniflux 单用户/小团队场景可接受
- 迁移后后续新写入（`createEntry`/`updateEntry`）也同步使用相同的 setweight 公式，保持一致

#### 1.2.3 末尾迁移 — 索引裁剪（部分 GIN 索引）

在引入软删除（`status='removed'`）和墓碑表（`entry_tombstones`）之后，接近末尾的一次迁移优化了索引体积：

```sql
DROP INDEX document_vectors_idx;

CREATE INDEX document_vectors_idx
    ON entries
    USING gin(document_vectors)
    WHERE status != 'removed';   -- ★ 部分索引条件
```

**收益**：
- 已标记为 `removed` 的条目虽然保留行（供墓碑去重），但其全文搜索向量**不再占索引空间**
- PostgreSQL 查询优化器自动过滤 `status='removed'` 的条目（因搜索 WHERE 条件均含用户过滤+状态隐含约束），部分索引完全可用

---

**权重策略（setweight 与 ts_rank 协同）**：
- 标题（title）→ `setweight(..., 'A')` → **权重 A**（最高，匹配时排名靠前）
- 正文（content）→ `setweight(..., 'B')` → **权重 B**（次高）
- PostgreSQL 四级权重在 `ts_rank()` 函数中的默认权重数组为 **`{0.1, 0.2, 0.4, 1.0}`**，索引顺序是 **`{D, C, B, A}`**（注意字母倒序）：
  - **D** = 0.1（默认权重，未显式 setweight 的词）
  - **C** = 0.2
  - **B** = 0.4 ← Miniflux 正文使用此级
  - **A** = 1.0 ← Miniflux 标题使用此级
- **实际比例换算**：同一个词出现在标题中的 ts_rank 贡献是出现在正文中的 **1.0 / 0.4 = 2.5 倍**。例如词 "golang" 在标题出现 1 次，等价于在正文出现 2.5 次（不计词频归一化因子）
- `||` 操作符：`setweight(title_vec, 'A') || setweight(content_vec, 'B')` 会合并两个 tsvector，保留各自的权重标签和位置信息

### 1.3 文本搜索配置（regconfig）与中文条目退化路径

#### 1.3.1 Miniflux 未显式指定 regconfig，走默认配置

**代码证据**：全文所有 `to_tsvector()` / `websearch_to_tsquery()` 调用均只传入一个文本参数，未传入 `regconfig`：

```go
// entry.go createEntry / updateEntry / UpdateEntryTitleAndContent 三处均为：
setweight(to_tsvector($11), 'A') || setweight(to_tsvector($12), 'B')
//                                    ↑ 只传文本，不传 regconfig

// entry_query_builder.go WithSearchQuery:
e.document_vectors @@ websearch_to_tsquery($%d)
//                                  ↑ 同样只传文本
```

根据 PostgreSQL 源码（`src/backend/tsearch/to_tsany.c`），单参数 `to_tsvector(text)` 内部实现为：

```c
// to_tsvector_byid 内部调用 getTSCurrentConfig(true)
// 读取 GUC 参数 default_text_search_config 的当前值
```

即**每次 SQL 执行都会读取当前会话的 `default_text_search_config` 参数**。

#### 1.3.2 default_text_search_config 的实际取值链

| 层级 | 默认值 | 说明 |
|------|--------|------|
| PostgreSQL 编译内置 | **`pg_catalog.simple`** | 绝对最低 fallback |
| initdb 初始化时 | 根据 `lc_ctype` 区域自动匹配：<br>`en_US.UTF-8` → `pg_catalog.english`<br>`zh_CN.UTF-8` → **无对应内置配置，回落至 `pg_catalog.simple`** | PostgreSQL 仅为十几种西欧语言提供预定义 config |
| `postgresql.conf` | 可手动覆盖，Miniflux 安装文档**未要求设置**此参数 | 大多数生产部署沿用 initdb 的值 |
| 会话级 `SET` | Miniflux 代码中 **未发现任何 `SET default_text_search_config` 语句** | 全部沿用实例级默认值 |

**关键代码核查**：在 `internal/database/`、`internal/storage/`、`internal/config/` 三个目录全局搜索，没有任何对 `default_text_search_config`、`regconfig`、`english`（作为 config 名）的直接 SET 或连接池初始化语句。

#### 1.3.3 中文条目的命中退化路径（三层退化）

对于中文 RSS 条目，由于 PostgreSQL 社区版**不内置中文分词配置**（zhparser、pg_jieba 均为第三方扩展，Miniflux 未集成），必然发生以下退化：

```
退化第一层：default_text_search_config 取不到中文配置
        ↓
退化第二层：实际使用 pg_catalog.simple 或 pg_catalog.english
        ↓
退化第三层：分词器无法识别中文字符边界，退化为 "逐字符" 或 "连续字符串" 建索引
```

**两种退化场景的具体行为**：

**场景 A — 使用 `pg_catalog.simple`（常见于 zh_CN 部署）**
- `simple` 配置的解析器：default parser 按 Unicode 类别分词
- 对连续中文字符串 "这是一个中文测试文本"：
  - parser 识别出 token 类型为 `word`，但**不会按语义拆分**
  - 整个连续中文序列被当作**单个巨型 token**（例如 10 个汉字 = 1 个 lexeme）
  - `to_tsvector()` 直接存储该 token，不做词干还原
- **查询时的致命问题**：用户搜 "中文" 只有当 "中文" 这两个字**恰好作为完整条目被收录**时才能命中。如果条目是 "学习中文编程"，建索引时被当成一个 token，则搜 "中文"**无法匹配**（`@@` 操作符要求 lexeme 完全相等）

**场景 B — 使用 `pg_catalog.english`（常见于 Docker 镜像、en_US 部署）**
- `english` 配置使用 Snowball stemmer + 英文停用词表
- 中文字符逐个进入英文 Snowball stemmer：
  - stemmer 对非 ASCII 字符**不做任何变换**（无规则匹配）
  - 但分词边界仍以标点、空格为准
- **行为与场景 A 等价**：连续无标点中文 → 单个 lexeme → 无法分词搜索

**用户侧表现**：
- 搜 "docker" 这类英文单词可正常命中（有空格分词边界）
- 搜 "Kubernetes 入门" 中 "Kubernetes" 可命中，"入门" 大概率无法命中
- 中文**标题命中概率 > 正文**，因为标题中常常夹杂英文/数字/标点，无意中制造了分词边界
- 标签搜索（`WithTags`）不受影响，因为标签使用 PostgreSQL 数组 `@>` 运算符，与全文索引完全独立

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

#### 2.2.1 ts_rank vs ts_rank_cd：Miniflux 为何选择 ts_rank 以及邻近性排序的影响

PostgreSQL 提供两种全文相关度排名函数，**Miniflux 明确使用 `ts_rank()` 而非 `ts_rank_cd()`**（代码全局 grep `ts_rank_cd` / `cover_density` 无任何匹配）。两者在多关键字搜索下行为差异显著：

| 维度 | `ts_rank()`（Miniflux 使用） | `ts_rank_cd()`（Cover Density，未使用） |
|------|------------------------------|----------------------------------------|
| **核心算法** | 词频 × 权重 × 逆文档频率（TF-IDF 变体） | Clarke-Cormack-Tudhope 覆盖密度算法 |
| **多关键字邻近性** | ❌ **完全忽略**。词 A 在第 1 段、词 B 在末段 vs 词 A 词 B 相邻，得分相同 | ✅ **核心考虑**。相邻匹配得分远高于分散匹配 |
| **对短语匹配的敏感度** | "postgresql performance" 两词分两处出现 = 两词紧邻出现 | 两词紧邻（距离 1） >> 两词相隔 100 词 >> 两词分处两段 |
| **位置信息依赖** | 需要位置（计算权重）但不计算距离 | **强依赖位置**，无位置信息的 tsvector 返回 0 |
| **归一化参数** | 支持 1/2/4/8/16/32 六种归一化（文档长度、日志等） | 同样归一化参数集，但默认值都是 `0`（不归一化） |

**Miniflux 选择 ts_rank 的原因（代码事实推导）**：

1. **RSS 条目长度差异极大**：从几十字的微推到几万字的长文，`ts_rank_cd` 的"覆盖密度"对长文不友好——长文即使包含所有关键字，但因整体长度大，覆盖密度分数反而低；`ts_rank` 通过词频累积对长文更公平
2. **排序目标是"新且相关"而非"精确匹配"**：Miniflux 的排序公式 `ts_rank - 时间衰减` 本质上是让新文章占优势；如果改用 `ts_rank_cd`，会让"关键词紧密相邻"的旧文章排名反超"关键词分散"的新文章，与 RSS 阅读器的时效性期望冲突
3. **`websearch_to_tsquery` 本身已提供短语匹配**：用户可输入 `"postgresql performance"`（带引号）强制短语匹配，`websearch_to_tsquery` 会生成带 `<->` 邻近操作符的 tsquery，`@@` 匹配阶段即过滤非短语结果，不需要 ranking 阶段用 `ts_rank_cd` 再次强化
4. **性能**：`ts_rank_cd` 需要对匹配位置做区间覆盖计算，复杂度高于纯词频统计的 `ts_rank`

**多关键字场景的具体行为差异对比**（示例：查询 `postgresql fulltext search`，三词 AND）：

假设文档 A：`"PostgreSQL fulltext search engine..."`（三词紧邻出现在开头）
假设文档 B：长文，"PostgreSQL" 在第 1 段，"fulltext" 在第 10 段，"search" 在最末段

| 函数 | 文档 A 得分 | 文档 B 得分 | 排名结果 |
|------|-----------|-----------|---------|
| **`ts_rank`**（Miniflux） | ≈1.0+1.0+1.0 = **3.0**（词频相同，权重相同） | ≈1.0+1.0+1.0 = **3.0**（三词各出现 1 次，频率相同） | **A 与 B 并列**，靠时间衰减分胜负 |
| `ts_rank_cd`（未使用） | **高**（三词紧邻，覆盖区间长度 ≈ 词距=2，密度 = 3/2 = 1.5） | **低**（三词覆盖长度 ≈ 2000 词，密度 = 3/2000 = 0.0015） | **A 远高于 B**（可能差 1000 倍） |

**如果未来 Miniflux 想增强多关键字邻近性**：
```sql
-- 可选方案：混合 ts_rank（词频）+ ts_rank_cd（邻近）+ 时间衰减
ts_rank(document_vectors, query) * 0.6
+ ts_rank_cd(document_vectors, query) * 0.4
- extract(epoch from now() - published_at)::float * 0.0000001 DESC
```
但当前代码（`entry_query_builder.go:169-171`）仅使用 `ts_rank`，不涉及 `ts_rank_cd`。

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

```sql
INSERT INTO entries
    (title, hash, url, comments_url, published_at, content, author,
     user_id, feed_id, reading_time, changed_at, document_vectors, tags)
SELECT
    $1,       -- entry.Title（原始标题，存入 title 列供显示）
    $2,       -- entry.Hash（内容哈希，用于去重）
    $3,       -- entry.URL
    $4,       -- entry.CommentsURL
    $5,       -- entry.Date（published_at）
    $6,       -- entry.Content（原始正文，存入 content 列供显示）
    $7,       -- entry.Author
    $8,       -- entry.UserID
    $9,       -- entry.FeedID
    $10,      -- entry.ReadingTime
    now(),    -- changed_at 自动填充
    -- ★ 关键：document_vectors 列实时计算（不存储原始文本，存分词+权重向量）
    setweight(to_tsvector($11), 'A') || setweight(to_tsvector($12), 'B'),
    --          ↑ $11=truncatedTitle  ↑ $12=truncatedContent
    $13       -- pq.Array(entry.Tags)
WHERE NOT EXISTS (
    SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
)
RETURNING id, status, created_at, changed_at
```

**参数映射表（与 tx.QueryRow 调用顺序一一对应）**：

| 占位符 | Go 变量 | 用途 | 是否参与索引 |
|--------|---------|------|-------------|
| `$1` | `entry.Title` | 原始标题列存储，供 UI 显示 | 否（截断版本见 `$11`） |
| `$2` | `entry.Hash` | SHA256 内容哈希，去重+墓碑检查 | 否 |
| `$3` | `entry.URL` | 条目链接 | 否 |
| `$4` | `entry.CommentsURL` | 评论链接 | 否 |
| `$5` | `entry.Date` | 发布时间（published_at） | 否（但影响排序时间衰减） |
| `$6` | `entry.Content` | 原始正文列存储，供阅读 | 否（截断版本见 `$12`） |
| `$7` | `entry.Author` | 作者 | 否（作者字段**未被索引**，搜作者名无法命中） |
| `$8` | `entry.UserID` | 用户隔离 | 否 |
| `$9` | `entry.FeedID` | 订阅源关联 | 否 |
| `$10` | `entry.ReadingTime` | 估算阅读时长 | 否 |
| **`$11`** | **`truncatedTitle`** | **标题截断版本（≤200KB，UTF-8 边界安全）** | **是，权重 A（1.0）** |
| **`$12`** | **`truncatedContent`** | **正文截断版本（≤500KB，UTF-8 边界安全）** | **是，权重 B（0.4）** |
| `$13` | `pq.Array(entry.Tags)` | 标签数组 | 否（标签使用 PostgreSQL `@>` 独立过滤） |

**重要区分**：`$1/$6` 存完整文本供显示，`$11/$12` 存截断文本供分词建索引。两者是**不同参数**，避免为了索引截断而丢失用户看到的完整内容。

### 4.2 订阅源刷新时更新索引

`updateEntry()`（`internal/storage/entry.go:166-210`）：订阅源重新抓取到内容变化时：

```sql
UPDATE entries
SET
    title            = $1,     -- 更新标题显示列
    url              = $2,
    comments_url     = $3,
    content          = $4,     -- 更新正文显示列
    author           = $5,
    reading_time     = $6,
    -- ★ document_vectors 随内容变更而全量重算（增量无意义，分词不可叠加）
    document_vectors = setweight(to_tsvector($7), 'A') || setweight(to_tsvector($8), 'B'),
    --                    ↑ $7=truncatedTitle    ↑ $8=truncatedContent
    tags             = $12
WHERE
    user_id = $9   AND
    feed_id = $10  AND
    hash    = $11     -- 按 (user_id, feed_id, hash) 三元组定位旧条目
RETURNING id
```

**参数映射表**：

| 占位符 | Go 变量 | 参与索引 |
|--------|---------|---------|
| `$1` | `entry.Title` | 否 |
| `$2` | `entry.URL` | 否 |
| `$3` | `entry.CommentsURL` | 否 |
| `$4` | `entry.Content` | 否 |
| `$5` | `entry.Author` | 否 |
| `$6` | `entry.ReadingTime` | 否 |
| **`$7`** | **`truncatedTitle`** | **是，权重 A** |
| **`$8`** | **`truncatedContent`** | **是，权重 B** |
| `$9` | `entry.UserID` | 否（WHERE 条件） |
| `$10` | `entry.FeedID` | 否（WHERE 条件） |
| `$11` | `entry.Hash` | 否（WHERE 条件） |
| `$12` | `pq.Array(entry.Tags)` | 否 |

**索引刷新原子性**：UPDATE 是单条 SQL 原子操作，`document_vectors` 与 `title/content` 列在同一事务中一起变更。GIN 索引随 `document_vectors` 列的 UPDATE 在同一 WAL 记录中增量维护，不存在"内容更新了但索引未刷新"的窗口期。

触发入口：`RefreshFeedEntries()` → `updateEntry()`（每篇条目独立事务）

### 4.3 用户手动更新条目内容

`UpdateEntryTitleAndContent()`（`internal/storage/entry.go:50-78`）：用户通过 API 修改条目内容或抓取完整网页时：

```sql
UPDATE entries
SET
    title            = $1,
    content          = $2,
    reading_time     = $3,
    -- ★ 用户编辑 / 网页抓取重算索引
    document_vectors = setweight(to_tsvector($4), 'A') || setweight(to_tsvector($5), 'B')
    --                    ↑ $4=truncatedTitle  ↑ $5=truncatedContent
WHERE
    id      = $6   AND   -- 按 (id, user_id) 精确定位单条
    user_id = $7
```

**参数映射表**：

| 占位符 | Go 变量 | 参与索引 |
|--------|---------|---------|
| `$1` | `entry.Title` | 否 |
| `$2` | `entry.Content` | 否 |
| `$3` | `entry.ReadingTime` | 否 |
| **`$4`** | **`truncatedTitle`** | **是，权重 A** |
| **`$5`** | **`truncatedContent`** | **是，权重 B** |
| `$6` | `entry.ID` | 否（WHERE 条件） |
| `$7` | `entry.UserID` | 否（WHERE 条件） |

**与 4.2 的差异**：
- 此函数不更新 URL、Author、Tags，只更新标题/正文/阅读时长（因此参数更少）
- WHERE 条件用主键 `id + user_id`，而非 `feed_id + hash`，适用单条 API 修改
- **不返回 `id`**（无需 RETURNING 子句），使用 `db.Exec` 而非 `QueryRow`

触发入口：
- API `updateEntryHandler`（用户通过 JSON API PATCH 条目内容）
- `fetchContentHandler` + `update_content=true`（readability 抓取完整网页后持久化）

### 4.4 截断策略与尾部内容命中丢失分析

PostgreSQL 的 `tsvector` 单个值最大为 1MB（PostgreSQL 源码 `src/include/tsearch/ts_type.h` 中定义的限制）。Miniflux 通过**三层截断机制**保证不越界，但代价是**超长文章尾部不可搜索**。

#### 4.4.1 三层截断的完整代码路径

```
原始 RSS 条目正文（可能 MB 级）
        │
        ▼ 第 1 层：迁移脚本 SQL 层（历史遗留，当前写入路径已不再使用）
│    substring(coalesce(content, '') for 1000000)
│    -- PostgreSQL substring(... for N) 按 "字符数" 截断，非字节数
│    -- 仅用于迁移 #21、#23 的历史回刷
        │
        ▼ 第 2 层：Go 应用层标题/正文独立字符数截断（当前主路径）
│    truncateTitleAndContentForTSVectorField(entry.Title, entry.Content)
│    ├── 标题: truncateStringForTSVectorField(title, 200000)   ← 最多 20 万字符
│    └── 正文: truncateStringForTSVectorField(content, 500000) ← 最多 50 万字符
│    -- 合计 ≤ 70 万字符；UTF-8 中文约 3 字节/字 → ≈ 2.1MB 原文
│    -- 留出约 800KB 给分词后的 lexeme 列表 + 位置数组 + 权重标签
        │
        ▼ 第 3 层：Go UTF-8 字符边界安全截断（truncateStringForTSVectorField 内部）
     truncateStringForTSVectorField(s string, maxSize int) string
     ├── if len(s) < maxSize: return s   ← 快速路径，不足上限直接返回
     ├── truncated := s[:maxSize-1]      ← 按字节切，可能切到多字节汉字中间
     └── 从末尾向前回退:
         ├── 遇到 0xxxxxxx (ASCII): 截断在此
         ├── 遇到 11xxxxxx (多字节起始): 截断在此之前，保留完整字符
         └── 遇到 10xxxxxx (continuation byte): 继续回退
     -- 回退逻辑保证不会出现无效 UTF-8 序列传给 to_tsvector
     -- 特殊 fallback: 全是 continuation byte（病理输入）→ 返回空串
```

#### 4.4.2 truncateStringForTSVectorField 关键实现（`storage/entry.go:686-708`）

```go
func truncateStringForTSVectorField(s string, maxSize int) string {
    if len(s) < maxSize {     // ★ 注意: 此处比较的是 len(s) = 字节数，不是 rune 数
        return s              // 快速路径: 字节长度 < maxSize 直接返回
    }

    truncated := s[:maxSize-1]
    // 从末尾字节向前回溯找 UTF-8 边界
    for i := len(truncated) - 1; i >= 0; i-- {
        if (truncated[i] & 0x80) == 0 {
            return truncated[:i+1]       // ASCII 起始: 保留到该字节
        }
        if (truncated[i] & 0xC0) == 0xC0 {
            return truncated[:i]         // 多字节起始: 保留到该字符之前
        }
    }
    return ""  // Fallback: 找不到合法边界 → 空串（保证 to_tsvector 不报错）
}
```

**重要代码事实**：
- `maxSize` 参数单位是**字节**（`len(s)` 返回字节数），不是字符数
- 正文 `maxSize = 500,000` ≈ **488 KB**，标题 `maxSize = 200,000` ≈ **195 KB**
- 对英文（1 字节/字）：正文可索引约 **50 万词**（足够任何 RSS 条目）
- 对中文（3 字节/字）：正文可索引约 **16.6 万汉字**（约 500~1000 段正文）

**单元测试覆盖**（`entry_test.go:11-97` 的 7 个测试用例）：

| 测试用例 | 输入 | 验证点 |
|---------|------|-------|
| Test case 1 | 短中文 "这是一个简短的中文测试文本" | 不截断，原样返回 |
| Test case 2 | `strings.Repeat("汉", 350K+)`（> 1MB） | 截断后 < 1MB，是原串前缀，UTF-8 合法 |
| Test case 3 | 恰好等于上限 - 1 的 ASCII | 不截断 |
| Test case 4 | 中英文混合 "测试Test汉字" 重复放大 | 截断后 UTF-8 合法，是前缀 |
| Test case 5 | 接近末尾是中文 + ASCII 后缀 | 截断到 ASCII 边界 |
| Test case 6 | 纯 ASCII 超大文本 | 截断后仍是前缀 |
| Test case 7 | 病理输入：全是 `0x80`（continuation byte） | 返回空串 fallback |

#### 4.4.3 尾部内容命中丢失的量化影响

**正文截断的实际影响**（以中文 RSS 条目为例）：

| 文章长度（汉字） | 正文索引量 | 尾部被截断的比例 | 命中丢失风险 |
|----------------|-----------|----------------|------------|
| < 16,666 字 | 100% | 0% | 无风险 |
| 50,000 字（≈标准中篇） | 100% | 0% | 无风险（5 万字 × 3 = 150KB < 488KB） |
| 166,666 字 | 100% | 0% | 临界（≈ 499KB，接近 50 万字节上限） |
| 200,000 字 | **83%** | 约 3.3 万字（尾部 17%） | 中风险 |
| 500,000 字（长篇连载） | **33%** | 约 33 万字（尾部 67%） | **高风险** |

**具体丢失场景示例**：

1. **法律/技术长文**：条款、结论、附录常出现在文末 → "版权所有"、"MIT License"、"总结" 等词若出现在 33 万字之后，完全无法命中
2. **教程系列**：实战代码块、习题答案位于文末 → 搜索代码中的函数名、配置项不命中
3. **长书评/影评**：最终评分、推荐语在末尾 → 搜 "五星推荐" 不命中（出现在尾部截断区）
4. **多页文章抓取**：readability 抓取的全文拼接后可能超过 50 万字节 → 后面几页的内容完全不可搜索

**标题截断的影响极小**：
- 标题上限 20 万字节（中文 ≈ 6.6 万字），远超任何 RSS 条目标题长度（实际 RSS 标题通常 < 200 字符），几乎不会触发截断
- 唯一风险：部分畸形 RSS 将正文塞入标题字段，此时标题截断才会生效

#### 4.4.4 迁移脚本 substring(... for 1000000) vs 应用层截断的关系

两处截断**同时存在但作用于不同时间点**：

| 来源 | 位置 | 截断方式 | 单位 | 当前是否仍在使用 |
|------|------|---------|------|---------------|
| 迁移脚本 #21/#23 | `migrations.go:281/297` | `substring(... for 1000000)` | **字符数**（PostgreSQL char length） | ❌ 仅历史迁移时执行过一次，当前写入路径不走 |
| `createEntry` / `updateEntry` / `UpdateEntryTitleAndContent` | `entry.go:679-708` | `truncateStringForTSVectorField` | **字节数**（Go len(s)） | ✅ 当前所有新写入、更新都走此路径 |

**迁移历史遗留的不一致**：
- 迁移 #21/#23 用 SQL `substring(... for 1000000)` 按**字符数**截断（中文 100 万字符 ≈ 3MB），而当前应用层按**字节数**截断（中文 50 万字节 ≈ 16.6 万字符）
- 含义：**极旧条目**（迁移 #23 之前创建、之后从未被刷新过）可能索引了比新条目更长的正文（最多 100 万字符 vs 16.6 万汉字）
- 实际影响极低：RSS 条目通常在被抓取后短期内会被 `RefreshFeedEntries` 重刷几次，触发 `updateEntry` 重新走应用层截断，旧数据会被逐步覆盖
- Miniflux 没有专门的"重索引所有历史条目"迁移来统一此差异

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
