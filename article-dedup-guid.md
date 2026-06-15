# 同源条目去重与稳定 ID 落库指南

## 概述

Miniflux 通过 **"稳定 Hash 标识 + 数据库唯一约束"** 的方式实现同源 Feed 内的条目去重与稳定 ID 落库。整个流程分为两个核心阶段：

1. **Hash 计算阶段**：在 Feed 解析适配器中，基于条目的稳定标识（GUID/ID/URL）生成 SHA-256 哈希值
2. **入库判定阶段**：在存储层，通过 `(feed_id, hash)` 联合唯一索引判断条目是否已存在

---

## 一、Hash 计算：稳定标识的提取与哈希化

Hash 是条目的"指纹"，必须具备 **稳定性**（同一内容多次计算结果相同）和 **唯一性**（不同内容哈希不同）。

### 1.1 哈希算法

所有适配器统一使用 `crypto.SHA256()` 计算哈希：

```go
// internal/crypto/crypto.go:26-29
func SHA256(value string) string {
    h := sha256.Sum256([]byte(value))
    return hex.EncodeToString(h[:])
}
```

> **注意**：`crypto.HashFromBytes()` 使用 FNV-128a（非加密哈希），但**条目 Hash 统一使用 SHA-256**。

### 1.2 各 Feed 格式的 Hash 计算策略

不同 Feed 格式有不同的唯一标识字段，因此 Hash 计算策略各异。

#### RSS 2.0 适配器

位置：`internal/reader/rss/adapter.go:116-139`

**优先级策略**：

| 优先级 | 数据源 | 计算方式 | 适用场景 |
|--------|--------|----------|----------|
| 1 | `<guid>` 元素 | `SHA256(guid)` | 规范情况，首次出现 |
| 2 | `<guid>` + 条目 URL | `SHA256(guid + "\|" + url)` | GUID 重复但有 URL |
| 3 | `<guid>` + 位置序号 | `SHA256(guid + "\|" + n)` | GUID 重复且无 URL |
| 4 | 条目 URL | `SHA256(url)` | 无 GUID 但有 URL |
| 5 | 标题 + 内容 | `SHA256(title + content)` | 兜底方案 |

**重复 GUID 处理**：

```go
seenGUIDs := make(map[string]int)

// 首次出现: n == 0 → 直接用 guid
// 重复出现: n > 0  → 拼接 URL 或位置序号
switch {
case n == 0:
    entry.Hash = crypto.SHA256(item.GUID.Data)
case entry.URL != "":
    entry.Hash = crypto.SHA256(item.GUID.Data + "|" + entry.URL)
default:
    entry.Hash = crypto.SHA256(item.GUID.Data + "|" + strconv.Itoa(n))
}
```

**设计意图**：
- 首次出现保持纯净 GUID 哈希，确保向后兼容
- 重复 GUID 时通过 URL 或位置进行去歧义
- 位置序号保证即使 URL 也相同，也不会哈希冲突

#### Atom 1.0 / 0.3 适配器

位置：`internal/reader/atom/atom_10_adapter.go:149-155`、`internal/reader/atom/atom_03_adapter.go:107-113`

**优先级策略**：

| 优先级 | 数据源 | 计算方式 |
|--------|--------|----------|
| 1 | `<id>` 元素 | `SHA256(id)` |
| 2 | 条目链接 (alternate) | `SHA256(url)` |

```go
for _, value := range []string{atomEntry.ID, atomEntry.Links.originalLink()} {
    if value != "" {
        entry.Hash = crypto.SHA256(value)
        break
    }
}
```

#### JSON Feed 适配器

位置：`internal/reader/json/adapter.go:174-181`

**优先级策略**：

| 优先级 | 数据源 | 计算方式 |
|--------|--------|----------|
| 1 | `item.id` | `SHA256(id)` |
| 2 | `item.url` | `SHA256(url)` |
| 3 | `item.external_url` | `SHA256(external_url)` |
| 4 | 内容拼接 | `SHA256(content_text + content_html + summary)` |

#### RDF (RSS 1.0) 适配器

位置：`internal/reader/rdf/adapter.go:73-79`

**优先级策略**：

| 优先级 | 数据源 | 计算方式 |
|--------|--------|----------|
| 1 | 条目链接 | `SHA256(link)` |
| 2 | 标题 + 描述 | `SHA256(title + description)` |

### 1.3 Hash 计算的时机与注意事项

**关键设计决策**：Hash 计算发生在 **URL 清洗和重写之前**。

处理流程顺序：
```
Feed 解析 → 计算 Hash → URL 清洗 → URL 重写 → 内容抓取 → 入库
```

**原因**：
- Hash 必须基于 Feed 原始数据计算，确保每次刷新结果一致
- URL 清洗/重写可能随规则变化而变化，不应影响 Hash 稳定性
- 处理器 (`processor.ProcessFeedEntries`) 只是消费已有的 Hash，不重新计算

---

## 二、入库判定：基于 (feed_id, hash) 的去重机制

### 2.1 数据库层唯一约束

**唯一索引**：`entries_feed_id_hash_key (feed_id, hash)`

> 位置：`internal/database/migrations.go`（多处迁移定义）

```sql
-- entries 表的唯一约束
UNIQUE (feed_id, hash)
```

**设计要点**：
- 去重是 **Feed 内的**（同源去重），不是全局去重
- `feed_id + hash` 联合唯一，不同 Feed 间相同 Hash 不会冲突
- 使用数据库唯一约束保证并发安全

### 2.2 关键判定函数

#### entryExists：检查条目是否已存在

位置：`internal/storage/entry.go:213-224`

```go
func (s *Storage) entryExists(tx *sql.Tx, entry *model.Entry) (bool, error) {
    var result bool
    err := tx.QueryRow(
        `SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1`,
        entry.FeedID, entry.Hash,
    ).Scan(&result)
    
    if err != nil && err != sql.ErrNoRows {
        return result, fmt.Errorf(...)
    }
    return result, nil
}
```

**特点**：
- 利用 `entries_feed_id_hash_key` 索引，查询高效
- 只检查 `entries` 表，不检查墓碑表
- 在事务内执行，配合 INSERT 保证一致性

#### IsNewEntry：判断是否为全新条目

位置：`internal/storage/entry.go:278-293`

```go
func (s *Storage) IsNewEntry(feedID int64, entryHash string) bool {
    query := `
        SELECT
            EXISTS (SELECT 1 FROM entries WHERE feed_id=$1 AND hash=$2)
            OR EXISTS (SELECT 1 FROM entry_tombstones WHERE feed_id=$1 AND hash=$2)
    `
    var known bool
    s.db.QueryRow(query, feedID, entryHash).Scan(&known)
    return !known
}
```

**用途**：
- 在爬虫抓取前快速判断是否需要抓取
- 同时检查 `entries` 和 `entry_tombstones` 表
- 避免对已知条目做无用功（如全文抓取）

### 2.3 墓碑机制 (Tombstone)

**问题**：如果条目被删除（归档/清空历史），下次刷新时可能又被重新摄入。

**解决方案**：`entry_tombstones` 表记录已删除条目的 `(feed_id, hash)`。

位置：`internal/storage/entry.go:362-404` (ArchiveEntries)、`internal/storage/entry.go:491-508` (FlushHistory)

```sql
-- 删除条目时，同时插入墓碑记录
WITH deleted AS (
    DELETE FROM entries ...
    RETURNING entries.feed_id, entries.hash
)
INSERT INTO entry_tombstones (feed_id, hash)
SELECT feed_id, hash FROM deleted WHERE hash <> ''
ON CONFLICT (feed_id, hash) DO NOTHING
```

**创建新条目时的原子检查**：

位置：`internal/storage/entry.go:86-119`

```sql
INSERT INTO entries (...)
SELECT ...
WHERE NOT EXISTS (
    SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
)
RETURNING id, status, created_at, changed_at
```

**原子性保证**：
- 使用 `INSERT ... WHERE NOT EXISTS` 单条语句
- 避免先查后插的竞态条件
- 若返回 `sql.ErrNoRows`，说明被墓碑拦截

---

## 三、完整入库流程

### 3.1 RefreshFeedEntries：Feed 刷新主流程

位置：`internal/storage/entry.go:315-360`

```
遍历每条条目
    ↓
开启事务
    ↓
entryExists() 检查是否存在
    ├─ 已存在 → updateExistingEntries ? updateEntry() : 跳过
    └─ 不存在 → createEntry()
            ├─ 成功 → 加入 newEntries 列表
            └─ 墓碑拦截 → 静默跳过 (ErrEntryTombstoned)
    ↓
提交事务
```

**关键代码**：

```go
func (s *Storage) RefreshFeedEntries(userID, feedID int64, entries model.Entries, updateExistingEntries bool) (newEntries model.Entries, err error) {
    for _, entry := range entries {
        entry.UserID = userID
        entry.FeedID = feedID

        tx, err := s.db.Begin()
        // ... 错误处理

        entryExists, err := s.entryExists(tx, entry)
        // ... 错误处理

        if entryExists {
            if updateExistingEntries {
                err = s.updateEntry(tx, entry)
            }
        } else {
            err = s.createEntry(tx, entry)
            switch {
            case errors.Is(err, ErrEntryTombstoned):
                err = nil  // 静默跳过墓碑条目
            case err == nil:
                newEntries = append(newEntries, entry)
            }
        }
        // ... 错误处理 & 提交事务
    }
    return newEntries, nil
}
```

### 3.2 updateExistingEntries 参数的控制

位置：`internal/reader/handler/handler.go:323-324`

```go
// 只有在非爬虫模式、非忽略更新、或强制刷新时才更新现有条目
updateExistingEntries := forceRefresh || (!originalFeed.Crawler && !originalFeed.IgnoreEntryUpdates)
```

**设计意图**：
- **爬虫模式**：只抓取新条目，不更新已有条目（避免覆盖用户阅读状态）
- **忽略更新**：Feed 配置了忽略条目更新时不更新
- **强制刷新**：强制刷新时总是更新

### 3.3 稳定 ID 的保证机制

**稳定 ID 的含义**：同一逻辑条目在数据库中的主键 ID 保持不变。

**保证手段**：

1. **Hash 稳定性**：基于 Feed 原始数据的稳定标识（GUID/ID）计算 Hash，不受内容变化影响
2. **唯一约束**：`(feed_id, hash)` 联合唯一索引，确保不会重复插入
3. **先查后插**：通过 `entryExists` 找到已有条目，复用其 ID
4. **事务保护**：检查和插入在同一事务中，避免并发竞态

---

## 四、从 Hash 计算到入库的完整衔接

### 4.1 调用链路总览

```
handler.RefreshFeed()
    ↓
parser.ParseFeed()               # 解析 Feed
    ↓
adapter.buildFeed()             # 各格式适配器
    └─ entry.Hash = SHA256(...)  # ← Hash 在此生成
    ↓
processor.ProcessFeedEntries()   # 处理条目（URL清洗、爬虫、过滤）
    ├─ entry.URL 被修改          # 注意：Hash 不变
    └─ store.IsNewEntry()        # 预检查（用于爬虫决策）
    ↓
storage.RefreshFeedEntries()     # 入库判定
    ├─ entryExists(tx, entry)    # 检查是否存在
    ├─ createEntry(tx, entry)    # 创建新条目
    └─ updateEntry(tx, entry)    # 更新已有条目
```

### 4.2 关键衔接点

| 阶段 | 模块 | 关键动作 | Hash 是否变化 |
|------|------|----------|--------------|
| Feed 解析 | 各 adapter | 生成 Hash | ✅ 首次生成 |
| URL 清洗 | processor | 移除追踪参数 | ❌ 不变 |
| URL 重写 | processor | 应用重写规则 | ❌ 不变 |
| 全文抓取 | processor | 替换正文内容 | ❌ 不变 |
| 内容重写 | processor | 应用内容重写规则 | ❌ 不变 |
| 过滤检查 | processor | 应用过滤规则 | ❌ 不变 |
| 入库判定 | storage | 存在性检查 | ❌ 不变 |
| 新建/更新 | storage | 写入数据库 | ❌ 不变 |

**重要结论**：Hash 在 Feed 解析阶段就已确定，后续所有处理都不会改变 Hash。这保证了即使内容被重写、URL 被清洗，同一条目的身份标识始终一致。

---

## 五、设计要点总结

### 5.1 为什么用 Hash 而不是直接用 GUID？

1. **长度固定**：SHA-256 始终是 64 字符十六进制，便于索引和存储
2. **统一格式**：不同 Feed 格式的唯一标识格式各异，Hash 后统一
3. **组合标识**：RSS 重复 GUID 场景需要组合多个字段，Hash 后变成单一值
4. **避免注入**：原始 GUID 可能包含特殊字符，Hash 后安全

### 5.2 为什么 Hash 不包含内容？

1. **稳定性优先**：内容经常变化（修复错别字、更新文章），Hash 应保持不变
2. **身份 vs 内容**：Hash 是**身份标识**，不是**内容摘要**
3. **更新机制**：内容变化通过 `updateEntry` 更新，不需要改变 Hash

### 5.3 为什么是 feed_id + hash 而不是全局唯一？

1. **同源去重**：用户订阅的不同 Feed 可能有相同内容（如转载），这是正常的
2. **隔离性**：每个用户/Feed 的条目相互独立
3. **性能**：联合索引查询效率高，范围可控

### 5.4 墓碑机制的必要性

1. **防止回退**：已删除的条目不应因为 Feed 刷新又"回来"
2. **归档策略**：配合 `ArchiveEntries` 实现自动清理旧条目
3. **原子操作**：`INSERT ... WHERE NOT EXISTS` 保证并发安全

---

## 六、相关代码文件速查表

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| SHA-256 哈希 | `internal/crypto/crypto.go` | 26-29 |
| RSS Hash 计算 | `internal/reader/rss/adapter.go` | 116-139 |
| Atom 1.0 Hash 计算 | `internal/reader/atom/atom_10_adapter.go` | 149-155 |
| Atom 0.3 Hash 计算 | `internal/reader/atom/atom_03_adapter.go` | 107-113 |
| JSON Hash 计算 | `internal/reader/json/adapter.go` | 174-181 |
| RDF Hash 计算 | `internal/reader/rdf/adapter.go` | 73-79 |
| 条目处理流程 | `internal/reader/processor/processor.go` | 27-177 |
| 条目存在性检查 | `internal/storage/entry.go` | 213-224 |
| 新条目预检查 | `internal/storage/entry.go` | 278-293 |
| 创建条目（含墓碑检查） | `internal/storage/entry.go` | 81-161 |
| 更新条目 | `internal/storage/entry.go` | 166-210 |
| Feed 刷新入库 | `internal/storage/entry.go` | 315-360 |
| 归档+墓碑 | `internal/storage/entry.go` | 362-404 |
| Handler 刷新流程 | `internal/reader/handler/handler.go` | 195-372 |
