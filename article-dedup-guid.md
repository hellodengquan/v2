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

## 三、墓碑的写入时机与生命周期

### 3.1 墓碑表结构

```sql
-- internal/database/migrations.go (迁移 v47)
CREATE TABLE entry_tombstones (
    feed_id   bigint NOT NULL REFERENCES feeds(id) ON DELETE CASCADE,
    hash      text    NOT NULL CHECK (hash <> ''),
    deleted_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (feed_id, hash)
);

CREATE INDEX entry_tombstones_deleted_at_idx ON entry_tombstones (deleted_at);
```

**关键约束**：
- **联合主键** `(feed_id, hash)`：同一 Feed 下同一 Hash 只保留一条墓碑
- **外键级联删除** `REFERENCES feeds(id) ON DELETE CASCADE`：Feed 被删除时，该 Feed 下所有墓碑自动清除
- **`deleted_at` 索引**：便于按时间查询墓碑（但目前代码中未使用此索引做定期清理）

### 3.2 墓碑写入的三条路径

#### 路径 A：定时归档 — ArchiveEntries

位置：`internal/storage/entry.go:362-404`

**触发时机**：后台清理调度器周期性调用

**调用链**：
```
daemon.startDaemon()
    └─ runScheduler(store, pool)
           └─ cleanupScheduler(store, cleanupFrequency)       # 每 CLEANUP_FREQUENCY_HOURS 小时触发
                  └─ runCleanupTasks(store)
                         ├─ store.ArchiveEntries("read",   CleanupArchiveReadInterval(),   CleanupArchiveBatchSize())
                         └─ store.ArchiveEntries("unread", CleanupArchiveUnreadInterval(), CleanupArchiveBatchSize())
```

**归档条件**：

| 状态 | 配置项 | 默认值 | 含义 |
|------|--------|--------|------|
| `read` | `CLEANUP_ARCHIVE_READ_DAYS` | 60 天 | 已读条目保留 60 天 |
| `unread` | `CLEANUP_ARCHIVE_UNREAD_DAYS` | 180 天 | 未读条目保留 180 天 |

**批量控制**：

| 配置项 | 默认值 | 含义 |
|--------|--------|------|
| `CLEANUP_ARCHIVE_BATCH_SIZE` | 10000 | 每轮最多归档条目数 |
| `CLEANUP_FREQUENCY_HOURS` | 24 小时 | 清理任务执行周期 |

**归档 SQL 的筛选条件**：
```sql
WHERE
    status = $1                           -- 指定状态（read 或 unread）
    AND starred IS FALSE                  -- 星标条目不归档
    AND share_code = ''                   -- 已分享条目不归档
    AND created_at < now() - $2::interval -- 超过保留期
ORDER BY created_at ASC
FOR UPDATE SKIP LOCKED                   -- 跳过被锁定的行，避免阻塞
LIMIT $3                                 -- 批量上限
```

**设计要点**：
- `FOR UPDATE SKIP LOCKED`：多个清理进程不会冲突，跳过正在被其他事务处理的行
- 批量限制：避免单次删除过多行导致长事务和锁争用
- 星标和分享的条目永不归档

#### 路径 B：手动清空历史 — FlushHistory

位置：`internal/storage/entry.go:491-508`

**触发时机**：用户主动操作

**两个调用入口**：
1. **Web UI**：`internal/ui/history_flush.go:13-21` — `flushHistory()` 同步执行
2. **API**：`internal/api/entry_handlers.go:558-562` — `flushHistoryHandler()` 异步执行（`go h.store.FlushHistory(...)`）

**清除条件**：
```sql
WHERE user_id = $1 AND status = 'read' AND starred IS FALSE AND share_code = ''
```

**与归档的区别**：
- 归档按时间窗口删除，FlushHistory 删除用户**所有**已读条目
- 归档是系统自动行为，FlushHistory 是用户主动操作
- 两者都会写入墓碑

#### 路径 C：迁移初始化 — 从旧状态迁移

位置：`internal/database/migrations.go` (迁移 v47)

```sql
-- 将旧的 "removed" 状态条目转化为墓碑
INSERT INTO entry_tombstones (feed_id, hash, deleted_at)
    SELECT feed_id, hash, changed_at
    FROM entries
    WHERE status = 'removed' AND hash <> ''
    ON CONFLICT (feed_id, hash) DO NOTHING;

-- 然后删除所有 "removed" 条目
DELETE FROM entries WHERE status = 'removed';
```

这是一次性迁移，将旧版 `removed` 状态条目转为墓碑记录，之后 `removed` 状态不再使用。

### 3.3 墓碑的清除

**当前代码中没有墓碑的定期清除机制**。墓碑一旦写入就会永久存在，除非：

1. **Feed 被删除**：外键 `ON DELETE CASCADE` 自动清除对应墓碑
2. **用户被删除**：Feed 被级联删除后，墓碑随之清除

**`deleted_at` 索引的存在意义**：为未来可能的墓碑定期清理预留索引支持，但当前版本未实现自动清理。

### 3.4 墓碑在写入路径上的拦截

```
                         墓碑拦截点
                             │
    ┌────────────────────────┤
    │                        │
    ▼                        ▼
IsNewEntry()           createEntry()
(预检查，轻量)          (入库，原子操作)
    │                        │
    │  查 entries            │  INSERT ... WHERE NOT EXISTS
    │  查 entry_tombstones   │    (SELECT 1 FROM entry_tombstones)
    │                        │
    ▼                        ▼
  用于爬虫决策              若被拦截 → ErrEntryTombstoned
  跳过全文抓取              → 静默丢弃
```

**两层检查的职责分工**：

| 函数 | 检查范围 | 调用者 | 目的 |
|------|----------|--------|------|
| `IsNewEntry()` | entries + tombstones | processor | 避免对已知条目做昂贵爬虫操作 |
| `createEntry()` | tombstones (原子) | storage | 入库时的最终防线，防止并发写入 |

---

## 四、完整入库流程

### 4.1 RefreshFeedEntries：Feed 刷新主流程

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

### 4.2 updateExistingEntries 在不同刷新场景下的取值

位置：`internal/reader/handler/handler.go:323`

```go
updateExistingEntries := forceRefresh || (!originalFeed.Crawler && !originalFeed.IgnoreEntryUpdates)
```

此值由三个因子决定：`forceRefresh`（调用方传入）、`Crawler`（Feed 配置）、`IgnoreEntryUpdates`（Feed 配置）。

#### 所有调用方及其传入的 forceRefresh 值

| 调用方 | 文件 | forceRefresh | 典型场景 |
|--------|------|:----------:|----------|
| Worker 后台消费 | `internal/worker/worker.go:40` | `false` | 定时调度刷新 |
| CLI `refresh-feeds` | `internal/cli/refresh_feeds.go:55` | `false` | 命令行手动批量刷新 |
| API 单个刷新 | `internal/api/feed_handlers.go:67` | `false` | REST API 调用 |
| Web UI 单个刷新 | `internal/ui/feed_refresh.go:21` | URL 参数 `?forceRefresh=true` | 用户在界面上点击刷新 |
| Web UI 全部刷新 | `internal/ui/feed_refresh.go:61` | N/A (走 worker pool) | 用户点击"刷新所有"，受 `ForceRefreshInterval` 限流 |

#### updateExistingEntries 真值表

| 场景 | forceRefresh | Crawler | IgnoreEntryUpdates | updateExistingEntries | 说明 |
|------|:----------:|:-------:|:------------------:|:-------------------:|------|
| 后台定时刷新 (普通 Feed) | F | F | F | **T** | 最常见场景，正常更新 |
| 后台定时刷新 (爬虫模式 Feed) | F | T | F | **F** | 爬虫模式只抓新条目，不覆盖已有 |
| 后台定时刷新 (忽略更新 Feed) | F | F | T | **F** | 用户选择不更新已有条目 |
| 后台定时刷新 (爬虫+忽略) | F | T | T | **F** | 双重跳过 |
| API 刷新 (同上 Feed 配置) | F | * | * | 同上 | 与后台刷新逻辑一致 |
| UI 强制刷新 (forceRefresh=true) | T | * | * | **T** | 无论 Feed 配置如何，总是更新 |
| UI 普通刷新 (forceRefresh=false) | F | * | * | 同后台刷新 | 无 query 参数时走默认逻辑 |

**`forceRefresh=true` 的额外效果**（不仅影响 updateExistingEntries）：

位置：`internal/reader/handler/handler.go:235-236`

```go
ignoreHTTPCache := originalFeed.IgnoreHTTPCache || forceRefresh
```

| 效果 | forceRefresh=false | forceRefresh=true |
|------|:-:|:-:|
| 跳过 HTTP 缓存 (ETag/Last-Modified) | 取决于 Feed 配置 | 强制跳过 |
| 触发全文爬虫（对已有条目） | 否（仅新条目） | 是 |
| 更新已有条目内容 | 取决于 Feed 配置 | 强制更新 |
| 强制刷新 Feed 图标 | 否 | 是 |
| 更新 Feed 图标 | 仅缺失时创建 | 强制更新或创建 |

#### Web UI 中 forceRefresh 的传递方式

位置：`internal/ui/feed_refresh.go:20`

```go
forceRefresh := request.QueryBoolParam(r, "forceRefresh", false)
```

用户在 Web UI 点击"刷新"按钮时，URL 中可以携带 `?forceRefresh=true` 参数。不带此参数时等同于普通刷新。

#### 全部刷新的限流保护

位置：`internal/ui/feed_refresh.go:38`

```go
if time.Since(sess.LastForceRefresh()) < config.Opts.ForceRefreshInterval() {
    // 拒绝请求，提示"刷新过于频繁"
}
```

用户点击"刷新所有 Feed"时受 `ForceRefreshInterval` 限流保护，避免误操作导致大量请求。

### 4.3 稳定 ID 的保证机制

**稳定 ID 的含义**：同一逻辑条目在数据库中的主键 ID 保持不变。

**保证手段**：

1. **Hash 稳定性**：基于 Feed 原始数据的稳定标识（GUID/ID）计算 Hash，不受内容变化影响
2. **唯一约束**：`(feed_id, hash)` 联合唯一索引，确保不会重复插入
3. **先查后插**：通过 `entryExists` 找到已有条目，复用其 ID
4. **事务保护**：检查和插入在同一事务中，避免并发竞态

---

## 五、从 Hash 计算到入库的完整衔接

### 5.1 调用链路总览

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

### 5.2 关键衔接点

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

## 六、设计要点总结

### 6.1 为什么用 Hash 而不是直接用 GUID？

1. **长度固定**：SHA-256 始终是 64 字符十六进制，便于索引和存储
2. **统一格式**：不同 Feed 格式的唯一标识格式各异，Hash 后统一
3. **组合标识**：RSS 重复 GUID 场景需要组合多个字段，Hash 后变成单一值
4. **避免注入**：原始 GUID 可能包含特殊字符，Hash 后安全

### 6.2 为什么 Hash 不包含内容？

1. **稳定性优先**：内容经常变化（修复错别字、更新文章），Hash 应保持不变
2. **身份 vs 内容**：Hash 是**身份标识**，不是**内容摘要**
3. **更新机制**：内容变化通过 `updateEntry` 更新，不需要改变 Hash

### 6.3 为什么是 feed_id + hash 而不是全局唯一？

1. **同源去重**：用户订阅的不同 Feed 可能有相同内容（如转载），这是正常的
2. **隔离性**：每个用户/Feed 的条目相互独立
3. **性能**：联合索引查询效率高，范围可控

### 6.4 墓碑机制的必要性

1. **防止回退**：已删除的条目不应因为 Feed 刷新又"回来"
2. **归档策略**：配合 `ArchiveEntries` 实现自动清理旧条目
3. **原子操作**：`INSERT ... WHERE NOT EXISTS` 保证并发安全

---

## 七、相关代码文件速查表

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
| 清空历史+墓碑 | `internal/storage/entry.go` | 491-508 |
| 清理任务入口 | `internal/cli/cleanup_tasks.go` | 16-58 |
| 清理调度器 | `internal/cli/scheduler.go` | 53-56 |
| 守护进程启动 | `internal/cli/daemon.go` | 32-34 |
| Handler 刷新流程 | `internal/reader/handler/handler.go` | 195-372 |
| Worker 后台刷新 | `internal/worker/worker.go` | 40 |
| CLI 批量刷新 | `internal/cli/refresh_feeds.go` | 55 |
| API 单个刷新 | `internal/api/feed_handlers.go` | 67 |
| Web UI 单个刷新 | `internal/ui/feed_refresh.go` | 18-31 |
| Web UI 全部刷新 | `internal/ui/feed_refresh.go` | 33-68 |
| API 清空历史 | `internal/api/entry_handlers.go` | 558-562 |
| Web UI 清空历史 | `internal/ui/history_flush.go` | 13-21 |
| 墓碑表迁移 | `internal/database/migrations.go` | 迁移 v47 |
