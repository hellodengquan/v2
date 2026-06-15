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

### 1.3 Hash 算法选型与抗冲突边界

#### 为什么选 SHA-256 而不是其他哈希算法？

Miniflux 的条目 Hash 统一使用 SHA-256（`crypto.SHA256`），而非包内另一个可用的 `HashFromBytes()`（FNV-128a）。选择依据：

| 维度 | SHA-256（条目用） | FNV-128a（HashFromBytes 用） |
|------|:----------------:|:---------------------------:|
| 类型 | 加密哈希函数 | 非加密哈希函数 |
| 输出长度 | 256 bit（64 hex 字符） | 128 bit（32 hex 字符） |
| 抗碰撞性 | 强（密码学安全） | 弱（易构造碰撞） |
| 计算速度 | 较慢 | 极快 |
| 适用场景 | 身份标识、去重键 | 非安全场景的快速哈希 |

**选型理由**：
- **身份标识需要抗碰撞**：条目 Hash 是数据库唯一约束的组成部分，一旦发生碰撞会导致数据错乱（新条目无法入库、错误地覆盖旧条目）。
- **计算量可忽略**：每个 feed 刷新一次只计算几十个 Hash，SHA-256 的开销完全可以忽略。
- **统一输出长度**：64 字符十六进制输出，便于索引存储和比较。

#### 抗冲突边界分析

根据**生日悖论**，在 n 个独立样本中发生至少一次碰撞的概率约为：

```
P(n) ≈ 1 - e^(-n² / (2 * 2^256))
```

**关键数值参考**：

| 条目数量（单 feed） | 碰撞概率 | 说明 |
|:------------------:|:--------:|------|
| 1 万 | ≈ 0 | 可忽略 |
| 100 万 | ≈ 0 | 可忽略 |
| 1 亿 | ≈ 10^-32 | 宇宙级小概率 |
| 10^18 | ≈ 10^-12 | 仍然极低 |
| 2^128（约 3.4×10^38） | ≈ 50% | 生日边界 |

**实际场景评估**：
- 假设一个 Feed 有 10,000 条历史条目，碰撞概率约为 4×10^-70，远低于"服务器被陨石击中"的概率。
- 即使 Miniflux 实例有 10 万个 Feed，每个 Feed 10 万条条目，全局碰撞概率仍然可以忽略不计。
- **结论**：SHA-256 对于 Miniflux 的条目去重场景来说，抗冲突性有极大的安全余量。

#### 与 FNV-128a 的对比（为什么不用 FNV）

如果使用 FNV-128a（128 位输出），生日边界约为 2^64 ≈ 1.8×10^19 条，看似也够用，但问题在于：
- FNV 是**非加密哈希**，攻击者可以**故意构造碰撞**（尽管在 RSS 阅读器场景中威胁不大）
- FNV 的雪崩效应不如 SHA-256 好，相似输入可能产生相似输出
- 对于几千条/几万条的规模，两者速度差异可以忽略

因此选用 SHA-256 是一个"宁滥勿缺"的保守设计选择。

### 1.4 Hash 计算的时机与注意事项

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
2. **唯一约束兜底**：`(feed_id, hash)` 联合唯一索引，数据库层面确保不会重复插入
3. **先查后插**：通过 `entryExists` 找到已有条目，复用其 ID（常规路径）
4. **事务批量处理**：逐条事务提交，配合唯一索引保证最终一致性（并发场景下的兜底）

---

## 五、高并发场景下的 Race 行为分析

### 5.1 事务隔离级别

Miniflux 使用 Go 标准库 `database/sql` + `github.com/lib/pq` 驱动连接 PostgreSQL，**没有显式设置事务隔离级别**，因此使用 PostgreSQL 默认的 **`READ COMMITTED`** 隔离级别。

位置：`internal/database/postgresql.go:14-24`（连接池创建，无隔离级别设置）

```go
db, err := sql.Open("postgres", dsn)
// ...
db.SetMaxOpenConns(maxConnections)
db.SetMaxIdleConns(minConnections)
db.SetConnMaxLifetime(connectionLifetime)
```

**READ COMMITTED 的关键特性**：
- 事务中的每条 SQL 语句看到的是语句开始时的数据库快照
- 同一事务内，不同语句可能看到不同的数据（因为其他事务在中间提交）
- 不会出现脏读（读取未提交数据）
- 可能出现不可重复读和幻读

这意味着：`entryExists()` 和后续的 `createEntry()`/`updateEntry()` 虽然在同一事务中，但它们看到的数据可能不一致，因为中间可能有其他事务提交。

### 5.2 同一 Feed 并发刷新的可能性

首先需要回答：**同一 Feed 会被多个 Worker 同时刷新吗？**

#### 场景分析

| 场景 | 是否会并发刷新同一 Feed | 原因 |
|------|:---------------------:|------|
| 单实例 + 后台调度器 | ❌ 不会（默认） | `feedScheduler` 是单 goroutine，每次生成一批 jobs 推入队列，同一 feed 在一批中最多出现一次 |
| 单实例 + 用户手动刷新 | ⚠️ 可能 | 用户在 Web UI 点击刷新时，直接调用 `RefreshFeed()`，可能与后台 Worker 同时刷新同一 Feed |
| 单实例 + API 调用 | ⚠️ 可能 | API 直接调用 `RefreshFeed()`，可能与后台 Worker 并发 |
| 多实例部署 | ✅ 很可能 | 多个实例各有自己的调度器，可能同时选中同一 Feed |

**关键代码证据**：

位置：`internal/cli/scheduler.go:33-51`

```go
func feedScheduler(...) {
    for range time.Tick(frequency) {
        jobs, err := store.NewBatchBuilder().
            WithBatchSize(batchSize).
            // ...
            FetchJobs()    // 仅 SELECT，不锁定行
        
        if len(jobs) > 0 {
            pool.Push(jobs)  // 推入 Worker 队列
        }
    }
}
```

`FetchJobs()` 只是 `SELECT ... WHERE next_check_at < now()`，**没有 `FOR UPDATE` 或 `SKIP LOCKED`**，因此多实例场景下同一 Feed 可能被多个实例同时选中。

### 5.3 单条目并发插入的 Race 场景

即使同一 Feed 被并发刷新，每个条目也有独立的唯一约束保护。但 `entryExists` + `createEntry` 的模式存在 **TOCTOU（Time-Of-Check, Time-Of-Use）** 竞态。

#### Race 时序图

```
   Worker A 事务                     Worker B 事务
───────────────────────────────   ───────────────────────────────
1. entryExists() → false
2.                                   entryExists() → false
3. createEntry() 
   INSERT ... WHERE NOT EXISTS
   (检查墓碑，不检查 entries)
4.                                   createEntry()
                                       INSERT ... WHERE NOT EXISTS
                                       (检查墓碑，不检查 entries)
5. 唯一约束检测 → 成功
6.                                   唯一约束检测 → 失败！
                                       (23505 unique_violation)
7. COMMIT
                                       回滚/报错
```

#### 代码中的处理

位置：`internal/storage/entry.go:144-149`

```go
if errors.Is(err, sql.ErrNoRows) {
    return ErrEntryTombstoned
}
if err != nil {
    return fmt.Errorf(`store: unable to create entry %q (feed #%d): %v`, entry.URL, entry.FeedID, err)
}
```

**注意**：代码只处理了 `sql.ErrNoRows`（墓碑拦截），**没有处理 `unique_violation`（SQLSTATE 23505）**。当并发插入导致唯一约束冲突时，会直接返回错误，而不是静默处理为"已存在"。

#### `INSERT ... WHERE NOT EXISTS` 的原子性边界

`createEntry` 中的 `INSERT ... WHERE NOT EXISTS` **只检查墓碑表**，不检查 entries 表本身：

```sql
INSERT INTO entries (...)
SELECT ...
WHERE NOT EXISTS (
    SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
)
```

**为什么不检查 entries 表？**
- 因为有 `entries_feed_id_hash_key` 唯一索引兜底，重复插入会被数据库拒绝
- 检查 entries 表会增加额外开销
- 假设并发冲突概率低，错误由上层处理

**实际影响**：
- 在单实例、单调度器的部署中，冲突概率极低
- 在多实例部署中，冲突概率增加，但仍然是"异常路径"
- 冲突时 `RefreshFeedEntries` 会返回错误，导致本次 Feed 刷新失败，下次调度会重试

### 5.4 多 Worker 刷新同一 Feed 的数据一致性

当两个 Worker 同时刷新同一 Feed 时，除了单条目的插入冲突，还需要考虑整体数据一致性。

#### 各操作的原子性

| 操作 | 原子性保证 | 并发风险 |
|------|-----------|----------|
| 单条 `createEntry` | 由唯一索引保证原子性 | 冲突时返回错误 |
| 单条 `updateEntry` | 单条 UPDATE 原子 | 后提交的覆盖先提交的（lost update） |
| `UpdateFeed`（更新 next_check_at 等） | 单条 UPDATE 原子 | 后提交的覆盖先提交的 |
| `RefreshFeedEntries` 整体 | ❌ 非原子（逐条事务） | 部分成功部分失败 |

#### Lost Update 问题

`updateEntry` 通过 `(feed_id, hash)` 定位条目，直接更新所有字段：

位置：`internal/storage/entry.go:166-210`

```sql
UPDATE entries
SET title=$1, url=$2, ..., tags=$12
WHERE user_id=$9 AND feed_id=$10 AND hash=$11
RETURNING id
```

如果两个 Worker 同时更新同一条目，**后提交的会覆盖先提交的**。但由于两边都是从 Feed 源拉取的最新数据，内容差异通常不大，属于"可以接受的 lost update"。

`UpdateFeed` 也有类似问题：后提交的 `next_check_at` 会覆盖先提交的，可能导致调度时间不准确，但影响有限。

### 5.5 墓碑检查的原子性

位置：`internal/storage/entry.go:83-85`（代码注释）

```go
// The WHERE NOT EXISTS guard makes the tombstone check atomic with the insert, so a
// concurrent archive committing between an earlier existence check and this statement
// cannot bring a deleted entry back as unread.
```

这段注释点明了设计意图：
- `entryExists()` 只检查 entries 表，不检查墓碑
- `createEntry()` 的 `WHERE NOT EXISTS` 原子地检查墓碑
- 即使在 `entryExists()` 和 `createEntry()` 之间有归档事务提交（写入墓碑），也不会把已删除的条目重新"复活"

**这是设计上精心考虑的一点**：墓碑检查在 INSERT 语句内原子完成，而 entries 表的存在性检查由唯一约束兜底。

### 5.6 抗并发的架构级保障

虽然单条 SQL 层面存在一些理论上的 race，但系统在架构层面有多层保障：

#### 保障 1：Feed 级串行化（隐式）

同一 Feed 即使被并发刷新，由于：
- 数据库的唯一索引保证条目不会重复插入
- 更新操作是幂等的（同样的数据覆盖多次结果一样）
- 刷新失败会在下一个调度周期重试

因此不会出现数据损坏，最多是"浪费了一次刷新"。

#### 保障 2：单实例部署的实际串行化

绝大多数 Miniflux 部署是单实例的，且后台调度器是单 goroutine 的：
- `feedScheduler` 每次生成一批 jobs 推入队列
- Worker pool 虽然并发工作，但同一 feed 同一批中只出现一次
- 除非用户手动刷新，否则不会并发

#### 保障 3：错误重试机制

刷新失败的 Feed 不会被标记为"已检查"（或错误计数增加），下次调度会重试，最终一致性有保障。

### 5.7 如果要进一步增强并发安全性

如果需要在多实例高并发场景下进一步提升安全性，可以考虑：

1. **使用 PostgreSQL advisory lock**：刷新 Feed 前获取 `pg_try_advisory_xact_lock(feed_id)`，拿不到就跳过
2. **`SELECT ... FOR UPDATE SKIP LOCKED`**：在 `FetchJobs` 时锁定行，避免多实例抢同批
3. **处理 `unique_violation` 错误**：在 `createEntry` 中捕获 23505 错误，返回"已存在"而不是报错
4. **乐观锁**：在 entries 表增加 version 字段，更新时检查版本

但以 Miniflux 的应用场景（单用户/小团队 RSS 阅读器）来看，当前设计的并发安全性已经足够，过度设计反而增加复杂度。

---

## 六、从 Hash 计算到入库的完整衔接

### 6.1 调用链路总览

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

### 6.2 关键衔接点

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

## 七、设计要点总结

### 7.1 为什么用 Hash 而不是直接用 GUID？

1. **长度固定**：SHA-256 始终是 64 字符十六进制，便于索引和存储
2. **统一格式**：不同 Feed 格式的唯一标识格式各异，Hash 后统一
3. **组合标识**：RSS 重复 GUID 场景需要组合多个字段，Hash 后变成单一值
4. **避免注入**：原始 GUID 可能包含特殊字符，Hash 后安全

### 7.2 为什么 Hash 不包含内容？

1. **稳定性优先**：内容经常变化（修复错别字、更新文章），Hash 应保持不变
2. **身份 vs 内容**：Hash 是**身份标识**，不是**内容摘要**
3. **更新机制**：内容变化通过 `updateEntry` 更新，不需要改变 Hash

### 7.3 为什么是 feed_id + hash 而不是全局唯一？

1. **Feed 内去重**：去重范围限定在单个 Feed 内，不同 Feed 的条目天然独立
2. **隔离性**：每个用户/Feed 的条目相互独立，同一文章在多个 Feed 中各自有独立记录
3. **性能**：联合索引查询效率高，范围可控
4. **跨 Feed 不去重**：同一篇文章被多个 Feed 转载时，在各 Feed 中各有一条独立条目（详见第九章）

### 7.4 墓碑机制的必要性

1. **防止回退**：已删除的条目不应因为 Feed 刷新又"回来"
2. **归档策略**：配合 `ArchiveEntries` 实现自动清理旧条目
3. **原子操作**：`INSERT ... WHERE NOT EXISTS` 保证并发安全

---

## 八、Schema 升级后历史 Entries 的 Hash 复算与回填

### 8.1 核心结论：Hash 从不复算

**代码中没有任何 entry hash 回填/复算逻辑。**

在整个 `migrations.go`（当前 59 个迁移版本）中，与 `entries` 表相关的变更都是**加列、加索引、修改无关字段**，从未出现过 `UPDATE entries SET hash = ...` 语句。

这意味着：**一旦条目入库，其 Hash 值就永久不变**。无论后续 Schema 如何升级，历史条目的 Hash 保持写入时的值。

### 8.2 迁移调用链

```
程序启动
    ↓
database.Migrate(db)
    ↓
读取当前 schema_version → currentVersion
    ↓
for version := currentVersion; version < schemaVersion:
    tx, _ := db.Begin()
    ↓
    migrations[version](tx)     ← 逐版本执行迁移函数
    ↓
    TRUNCATE schema_version
    INSERT schema_version (version) VALUES (newVersion)
    ↓
    tx.Commit()
```

**关键特性**：
- 每个迁移在独立事务中执行
- 迁移失败会回滚，版本号不变
- 迁移按版本号严格递增，不可跳过

### 8.3 影响 Hash 语义的潜在变更点

虽然 Hash 值本身不会在迁移中被修改，但以下几类 Schema 变更可能**间接影响 Hash 的语义**：

#### 类型 A：适配器 Hash 计算逻辑变更

如果代码修改了某个适配器中 Hash 的计算方式（如 RSS 适配器对重复 GUID 的处理策略），旧条目的 Hash 与新计算出的 Hash 不一致。

**后果**：
- 旧条目继续以旧 Hash 存在于数据库中
- 下次 Feed 刷新时，同一逻辑条目被计算出新 Hash
- `entryExists()` 查不到旧条目（因为用新 Hash 查旧 Hash）
- 系统认为这是新条目，**导致重复入库**

**代码中没有任何防护措施**：不检测 Hash 变更、不做 Hash 复算、不做旧数据清理。

**实际风险评估**：在 Miniflux 的版本历史中，Hash 计算逻辑在早期版本确立后未发生过变更。如果未来需要变更，需要：
1. 编写迁移脚本，按新逻辑重算所有条目的 Hash
2. 更新墓碑表中对应的 Hash
3. 处理 Hash 冲突（新逻辑下可能出现的重复）

#### 类型 B：URL 清洗/重写规则变更

URL 清洗在 Hash 计算之后执行，因此规则的变更不影响 Hash。

#### 类型 C：Feed 格式解析器变更

如果解析器对同一 Feed 产出的 GUID/ID 发生变化（如修复解析 bug），会导致 Hash 变化，效果同类型 A。

### 8.4 历次迁移中与 Entries 相关的操作清单

| 迁移版本 | 操作 | 是否影响 Hash |
|:-------:|------|:------------:|
| v1 | 创建 entries 表，含 `unique(feed_id, hash)` | — 基线 |
| v4 | 添加 `starred` 列 | ❌ |
| v8 | 添加 `comments_url` 列 | ❌ |
| v10 | 添加 `document_vectors` 列，回填全文搜索向量 | ❌ |
| v11 | 更新 `document_vectors` 权重算法 | ❌ |
| v14 | 添加 `changed_at` 列，用 `published_at` 回填 | ❌ |
| v18 | 添加 `share_code` 列 + 唯一索引 | ❌ |
| v19 | 添加 `next_check_at` 到 feeds 表 | ❌ |
| v21 | 添加 `reading_time` 列 | ❌ |
| v22 | 修正 `created_at = published_at` | ❌ |
| v23 | 添加 `entries_user_feed_idx` 索引 | ❌ |
| v25 | 添加 `entries_id_user_status_idx` 索引 | ❌ |
| v27 | 添加 `entries_feed_id_status_hash_idx` 索引 | ❌ |
| v28 | 添加 `entries_user_id_status_starred_idx` 索引 | ❌ |
| v30 | 添加 `tags` 列 | ❌ |
| v34 | 添加 `entries_feed_url_idx` 索引 | ❌ |
| v41 | 清理空 tags | ❌ |
| v44 | 删除 `entries_feed_url_idx` 索引（URL 可能超 btree 上限） | ❌ |
| v47 | 创建 `entry_tombstones` 表，迁移 `removed` 状态条目 | ❌（Hash 值不变，只是搬迁） |
| v53 | 删除冗余索引 `entries_feed_idx`、`entries_user_status_idx` | ❌ |

**结论**：全部 59 个迁移版本中，**没有任何一个修改过 entries 表的 hash 值**。

### 8.5 Hash 不可变性的设计代价

**优点**：
- 简单可靠，不需要复杂的回填逻辑
- 避免回填过程中的数据一致性风险
- 唯一索引无需重建

**代价**：
- Hash 计算逻辑一旦确定就不能随意修改
- 如果必须修改，需要编写一次性迁移脚本
- 旧 Hash 和新 Hash 可能对应同一条逻辑条目，但系统无法识别

### 8.6 Enclosure Hash 的迁移先例

虽然 entry hash 从未迁移过，但 `enclosures` 表的 URL 唯一索引有过一次 Hash 算法变更的先例：

**问题**：PostgreSQL 18 在 FIPS 模式下禁用 MD5，导致基于 `md5(url)` 的唯一索引不可用。

**迁移 v59**：
```sql
DROP INDEX IF EXISTS enclosures_user_entry_url_unique_idx;
CREATE UNIQUE INDEX enclosures_user_entry_url_unique_idx
    ON enclosures (user_id, entry_id, encode(sha256(url::bytea), 'hex'));
```

**关键差异**：enclosure 的唯一索引是**表达式索引**（基于列值的函数计算），不需要回填数据——只需重建索引。而 entry 的 hash 是**存储列**，如果要变更算法，必须回填所有行的 hash 值，代价完全不同。

---

## 九、跨 Feed 同一 Article 的 Dedup 行为

### 9.1 核心结论：跨 Feed 不去重

Miniflux 的去重机制**严格限定在单个 Feed 内**。同一篇文章出现在多个 Feed 中时，每个 Feed 都会独立存储一条条目，不会被合并或去重。

### 9.2 唯一约束的作用范围

```sql
-- entries 表的唯一约束
UNIQUE (feed_id, hash)
```

**含义**：只有当 `feed_id` 和 `hash` 都相同时才视为重复。

| 场景 | feed_id | hash | 是否重复 | 结果 |
|------|:-------:|:----:|:--------:|------|
| 同一 Feed 内同一条目 | 相同 | 相同 | ✅ 重复 | 不重复插入 |
| 不同 Feed，同一文章（相同 GUID/URL） | 不同 | 相同 | ❌ 不重复 | 各自独立存储 |
| 同一 Feed 内，不同条目 | 相同 | 不同 | ❌ 不重复 | 正常插入 |
| 不同 Feed，不同条目 | 不同 | 不同 | ❌ 不重复 | 正常插入 |

### 9.3 跨 Feed 重复的具体场景

#### 场景 A：同一 RSS 源被多次订阅（不同 URL）

即使两个 Feed URL 指向同一内容源（如 `https://example.com/feed` 和 `https://example.com/feed?cat=tech`），只要 Feed URL 不同，它们就是不同的 Feed（`feeds` 表有 `unique(user_id, feed_url)` 约束），条目互不影响。

#### 场景 B：不同来源转载同一文章

同一篇文章被多个独立 Feed 转载：
- Feed A（原文站点）：GUID = `https://original.com/article-1`
- Feed B（转载站点）：GUID = `https://repost.com/article-1`

两者 GUID 不同，Hash 自然不同，互不干扰。

#### 场景 C：不同来源但 GUID 恰好相同

某些聚合类 Feed 可能直接透传原文的 GUID：
- Feed A：GUID = `tag:example.com,2024:article-1`
- Feed B：GUID = `tag:example.com,2024:article-1`

**此时 Hash 完全相同**，但因为 feed_id 不同，唯一约束 `(feed_id, hash)` 不冲突，两条条目**仍然独立存储**。

### 9.4 Feed 层面的唯一性保证

`feeds` 表的唯一约束确保**同一用户不会订阅相同 URL 的 Feed**：

```sql
-- feeds 表
UNIQUE (user_id, feed_url)
```

位置：`internal/storage/feed.go:62-68`

```go
func (s *Storage) FeedURLExists(userID int64, feedURL string) bool {
    var result bool
    query := `SELECT true FROM feeds WHERE user_id=$1 AND feed_url=$2 LIMIT 1`
    s.db.QueryRow(query, userID, feedURL).Scan(&result)
    return result
}
```

位置：`internal/storage/feed.go:70-76`

```go
func (s *Storage) AnotherFeedURLExists(userID, feedID int64, feedURL string) bool {
    var result bool
    query := `SELECT true FROM feeds WHERE id <> $1 AND user_id=$2 AND feed_url=$3 LIMIT 1`
    s.db.QueryRow(query, feedID, userID, feedURL).Scan(&result)
    return result
}
```

**调用时机**：`handler.RefreshFeed()` 在解析 Feed 内容之前（`internal/reader/handler/handler.go:267-270`）。

**作用**：防止同一用户订阅了重复 Feed URL 后产生混淆。这**不是条目级去重**，而是 **Feed 级去重**——阻止同一 URL 的 Feed 被订阅两次。

### 9.5 跨 Feed 条目在各查询路径中的表现

#### 条目列表页

用户在"未读"页面看到的条目列表是**跨 Feed 的**：

```sql
SELECT ... FROM entries
WHERE user_id = $1 AND status = 'unread'
ORDER BY published_at DESC
```

同一文章在不同 Feed 中的条目**都会显示**，用户会看到重复内容。

#### 搜索

全文搜索 `document_vectors` 也是跨 Feed 的，同一文章的不同 Feed 条目**都会被搜索到**。

#### 已读/未读状态

每个条目有独立的 `status` 字段。同一文章在 Feed A 中标为已读，不影响 Feed B 中的未读状态。

### 9.6 跨 Feed 去重为什么没有实现？

1. **语义模糊**：什么算"同一文章"？GUID 相同？URL 相同？内容相似？没有统一标准。
2. **Hash 不跨 Feed 可比**：不同 Feed 格式的 Hash 计算策略不同，同一文章在不同格式中可能产出不同 Hash。
3. **URL 不可靠**：同一文章的 URL 可能因 UTM 参数、重定向等原因不同。
4. **用户预期**：用户可能主动订阅多个来源以获取不同视角的评论/标注。
5. **复杂度高**：实现跨 Feed 去重需要模糊匹配或语义分析，代价远超收益。

### 9.7 跨 Feed 重复的缓解手段

虽然代码层面没有跨 Feed 去重，但用户可以通过以下方式缓解：

1. **不订阅重复源**：`FeedURLExists` 在订阅时阻止同一 URL 重复添加
2. **Feed 过滤规则**：使用 `blocklist_rules` / `keeplist_rules` 过滤不需要的条目
3. **手动标记已读**：在 UI 中批量标记重复条目为已读
4. **RSS 合并服务**：在上游使用 RSS 合并工具，将多个源合并后再订阅

---

## 十、相关代码文件速查表

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
| Enclosure 索引 MD5→SHA256 | `internal/database/migrations.go` | 迁移 v35/v59 |
| 数据库迁移执行入口 | `internal/database/database.go` | 13-51 |
| 全部迁移定义 | `internal/database/migrations.go` | 16-1548 |
| Feed URL 存在性检查 | `internal/storage/feed.go` | 62-68 |
| Feed URL 跨 Feed 重复检查 | `internal/storage/feed.go` | 70-76 |
| 数据库连接池 | `internal/database/postgresql.go` | 14-24 |
| Feed 调度器（批生成） | `internal/cli/scheduler.go` | 33-51 |
| Worker Pool | `internal/worker/pool.go` | 13-45 |
| Worker 执行体 | `internal/worker/worker.go` | 22-42 |
| 批任务构建（无行锁） | `internal/storage/batch.go` | 75-137 |
