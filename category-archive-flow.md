# 类别、标签与归档策略处理路径分析

本文档基于 Miniflux v2 代码库，深入分析分类归属、标签处理、置顶规则、归档策略和清理任务的完整数据流。

---

## 一、分类归属（Category）

### 1.1 数据模型

**Category 结构体** (`internal/model/category.go:9-17`)

| 字段 | 类型 | 说明 |
|------|------|------|
| ID | int64 | 主键 |
| Title | string | 分类名称，同用户下唯一 |
| UserID | int64 | 所属用户 |
| HideGlobally | bool | 是否在全局视图中隐藏 |
| FeedCount | *int | 该分类下 Feed 数量（非持久化，查询时计算） |
| TotalUnread | *int | 该分类下未读数（非持久化，查询时计算） |

**Feed 与 Category 的关联** (`internal/model/feed.go:67`)
- Feed 通过非持久化字段 `Category *Category` 建立对象关联
- 数据库层面通过 `feeds.category_id` 外键关联

### 1.2 数据库表结构

**categories 表** (`internal/database/migrations.go:47-54`)
```sql
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    user_id int NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title text NOT NULL,
    UNIQUE (user_id, title)
);
-- 后续迁移增加: hide_globally boolean NOT NULL DEFAULT false
```

**feeds 表** (`internal/database/migrations.go:56-72`)
```sql
CREATE TABLE feeds (
    id BIGSERIAL PRIMARY KEY,
    user_id int NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    category_id int NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    ...
);
```

**关键约束**：
- 删除用户 → 级联删除用户所有分类 → 级联删除所有 Feed → 级联删除所有条目
- 删除分类 → 级联删除该分类下所有 Feed

### 1.3 分类归属完整路径

#### 创建 Feed 时指定分类

**调用链**：
```
API/UI Handler
  → validator.ValidateCategoryCreation/Modification
  → handler.CreateFeed() (internal/reader/handler/handler.go:104)
    → store.CategoryIDExists(userID, categoryID) 校验 [L111-113]
    → subscription.WithCategoryID(feedCreationRequest.CategoryID) [L174]
    → processor.ProcessFeedEntries() 处理条目
    → store.CreateFeed(subscription) (internal/storage/feed.go:216-326)
      → INSERT INTO feeds (... category_id=$4 ...) VALUES (... feed.Category.ID ...) [L218-261]
      → 逐条创建该 Feed 下的初始 Entries
```

#### 分类归属查询

**按分类查询 Feed**：
- `storage/feed.go:FeedsByCategoryWithCounters()` - NewFeedQueryBuilder + WithCategoryID()

**按分类查询 Entry** (`internal/storage/entry_query_builder.go:139-145`)：
```go
func (e *EntryQueryBuilder) WithCategoryID(categoryID int64) *EntryQueryBuilder {
    // 通过 INNER JOIN feeds f 过滤: f.category_id = $N
}
```

实际 SQL 关联路径 (`entry_query_builder.go:330-335`)：
```sql
FROM entries e
INNER JOIN feeds f ON f.id = e.feed_id
INNER JOIN categories c ON c.id = f.category_id
```

#### 删除分类时的归属迁移

**普通删除** (`storage/category.go:225-242`)
- 直接 `DELETE FROM categories`，依赖外键 `ON DELETE CASCADE` 删除所有 Feed

**OPML 批量删除迁移** (`storage/category.go:246-290`) - `RemoveAndReplaceCategoriesByName`

事务性操作，三步流程：
1. **预检查**：确保删除后至少保留 1 个分类 [L254-263]
2. **迁移 Feed 归属** [L265-280]：
```sql
WITH d_cats AS (SELECT id FROM categories WHERE user_id=$1 AND title = ANY($2))
UPDATE feeds
SET category_id = (
    SELECT id FROM categories
    WHERE user_id = $1 AND id NOT IN (SELECT id FROM d_cats)
    ORDER BY title ASC LIMIT 1  -- 迁移到"第一个"分类（按标题字母序）
)
WHERE user_id = $1 AND category_id IN (SELECT id FROM d_cats)
```
3. **删除分类**：`DELETE FROM categories WHERE user_id=$1 AND title = ANY($2)`

### 1.4 全局隐藏机制

Category 和 Feed 都有 `hide_globally` 字段，形成**双重隐藏过滤**。

**查询过滤** (`entry_query_builder.go:226-230`)：
```go
func (e *EntryQueryBuilder) WithGloballyVisible() *EntryQueryBuilder {
    e.conditions = append(e.conditions, "c.hide_globally IS FALSE")
    e.conditions = append(e.conditions, "f.hide_globally IS FALSE")
}
```

即：条目所在的 Feed **和** Feed 所在的 Category 都必须不隐藏，才会出现在全局未读/历史视图中。

---

## 二、标签（Tag）处理

### 2.1 数据模型与存储

**Entry.Tags** (`internal/model/entry.go:46`)
```go
Tags []string `json:"tags"`
```

**数据库列** (`internal/database/migrations.go:674`)
```sql
ALTER TABLE entries ADD COLUMN tags text[] DEFAULT '{}';
```

PostgreSQL 原生数组类型，支持数组操作符。

### 2.2 标签来源：Feed 解析适配层

标签在 Feed 解析阶段从源数据的 category 字段提取，不同格式适配器逻辑不同：

#### RSS 适配器 (`internal/reader/rss/adapter.go`)

**优先级：条目标签 > Feed 级标签** [L154-156]
```go
entry.Tags = findEntryTags(&item)        // 先尝试从 <item> 提取
if len(entry.Tags) == 0 {
    entry.Tags = findFeedTags(&r.rss.Channel)  // 回退到 <channel> 级
}
```

**findEntryTags** [L290-295]：
- 从 `<category>` 元素提取：`rssItem.Categories`
- 从 `<media:category>` 提取：`rssItem.MediaCategories.LabelsSeq()`
- 去重+排序后存入

**findFeedTags** [L185-189]：
- `<channel>/<category>`：`rssChannel.Categories`
- iTunes 分类：`rssChannel.ItunesCategoriesSeq()`（含父子层级展开）

#### Atom 适配器 (`internal/reader/atom/atom_10_adapter.go:135-137`)
```go
entry.Tags = atomEntry.Categories.CategoryNames()  // <entry>/<category>
if len(entry.Tags) == 0 {
    entry.Tags = a.atomFeed.Categories.CategoryNames()  // <feed>/<category>
}
```

#### JSON Feed 适配器 (`internal/reader/json/adapter.go:171-172`)
```go
entry.Tags = make([]string, 0, len(item.Tags))
entry.Tags = appendSorted(entry.Tags, strings.TrimSpace, item.Tags...)
```
直接使用 JSON Feed 规范的 `_items[].tags` 数组。

### 2.3 标签的持久化

**创建条目** (`storage/entry.go:101,137`)：
```sql
INSERT INTO entries (..., tags, ...) VALUES (..., $13, ...)
-- $13 = pq.Array(entry.Tags)
```

**更新条目** (`storage/entry.go:179,198`)：
```sql
UPDATE entries SET ..., tags=$12 WHERE ...
-- $12 = pq.Array(entry.Tags)
```

### 2.4 标签查询

**查询构建器** (`entry_query_builder.go:160-166`)
```go
func (e *EntryQueryBuilder) WithTags(tags ...string) *EntryQueryBuilder {
    // PostgreSQL 数组包含操作符 @>
    // LOWER 转换实现大小写不敏感
    LOWER(e.tags::text)::text[] @> LOWER($N::text)::text[]
}
```

语义：条目的 tags 数组**必须包含所有**查询标签（AND 语义）。

### 2.5 标签 UI 路由

| 路由 | 处理文件 | 说明 |
|------|----------|------|
| `/tags/{tagName}/entries/all` | `ui/tag_entries_all.go:15-57` | 某标签下的全部条目列表，支持分页 |
| `/tags/{tagName}/entry/{entryID}` | `ui/entry_tag.go:16-85` | 标签上下文内的条目详情，带前后翻页 |

排序规则：`status ASC`（未读在前） → 用户配置的 `EntryOrder/EntryDirection` → `id` 兜底。

---

## 三、置顶规则（Starred）

### 3.1 数据模型

**Entry.Starred** (`internal/model/entry.go:42`)
```go
Starred bool `json:"starred"`
```

**数据库列** (`migrations.go:220`)
```sql
ALTER TABLE entries ADD COLUMN starred bool DEFAULT 'f';
```

**索引优化** (`migrations.go:403`)
```sql
CREATE INDEX entries_user_id_status_starred_idx 
ON entries (user_id, status, starred);
```

### 3.2 置顶/取消置顶操作

**存储层** (`storage/entry.go:452-459`)
```go
func (s *Storage) SetEntriesStarredState(userID int64, entryIDs []int64, starred bool) error {
    // UPDATE entries SET starred=$1, changed_at=now() 
    // WHERE user_id=$2 AND id=ANY($3)
}
```

支持批量操作，同时更新 `changed_at` 时间戳。

### 3.3 置顶保护：归档豁免

**这是置顶规则最关键的设计**。在归档清理逻辑中 (`storage/entry.go:368-379`)：

```sql
WITH to_delete AS (
    SELECT ... FROM entries
    WHERE
        status=$1 AND
        starred IS FALSE     -- ★ 置顶条目永不被归档
        share_code='' AND
        created_at < ...
    ...
)
```

置顶条目会被永久保留，不受任何自动清理策略影响。

### 3.4 置顶条目查询

**查询过滤** (`entry_query_builder.go:62-69`)
```go
func (e *EntryQueryBuilder) WithStarred(starred bool) *EntryQueryBuilder {
    if starred {
        e.conditions = append(e.conditions, "e.starred IS TRUE")
    } else {
        e.conditions = append(e.conditions, "e.starred IS FALSE")
    }
}
```

### 3.5 置顶 UI 路由

| 路由 | 处理文件 | 说明 |
|------|----------|------|
| `/starred` | `ui/starred_entries.go:14-48` | 星标/置顶列表视图 |
| `/starred/entry/{entryID}` | `ui/entry_starred.go:15-84` | 置顶上下文内的条目详情 |

排序：按用户偏好 `EntryOrder + EntryDirection`，再以 `id` 同方向兜底。

**分类内的置顶视图**还支持：`/category/{categoryID}/entries/starred`。

---

## 四、归档策略

### 4.1 归档的本质

归档 ≠ 软删除，归档 = **硬删除 + 墓碑记录**

- **删除**：从 `entries` 表物理移除旧条目，回收存储空间
- **墓碑**：在 `entry_tombstones` 表记录 `(feed_id, hash)` 元组，防止该条目下次刷新时被重新摄入

### 4.2 ArchiveEntries 核心实现

位置：`internal/storage/entry.go:363-404`

**函数签名**：
```go
func (s *Storage) ArchiveEntries(
    status string,        // "read" 或 "unread"
    interval time.Duration,  // 保留多久
    limit int,            // 单次批大小
) (int64, error)
```

#### 筛选条件（四者同时满足）

| 条件 | SQL 表达式 | 说明 |
|------|-----------|------|
| 状态匹配 | `status=$1` | 分别处理已读/未读 |
| 非置顶 | `starred IS FALSE` | 置顶条目豁免 |
| 未分享 | `share_code=''` | 生成过分享链接的条目保留 |
| 超时 | `created_at < now() - $2::interval` | 超过保留天数 |

#### SQL 执行流程（单条原子语句）

```sql
-- Step 1: 选取待删除的候选（行锁+跳过避免并发冲突）
WITH to_delete AS (
    SELECT id, feed_id, hash FROM entries
    WHERE ... ORDER BY created_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT $3
),
-- Step 2: 物理删除
deleted AS (
    DELETE FROM entries USING to_delete
    WHERE entries.id = to_delete.id
    RETURNING entries.feed_id, entries.hash
)
-- Step 3: 写入墓碑防重放
INSERT INTO entry_tombstones (feed_id, hash)
SELECT feed_id, hash FROM deleted WHERE hash <> ''
ON CONFLICT (feed_id, hash) DO NOTHING
```

**并发安全**：`FOR UPDATE SKIP LOCKED` 允许多个清理任务并行执行不冲突。

### 4.3 时间配置项

| 配置项 | 默认值 | Getter 方法 |
|--------|--------|-------------|
| `CLEANUP_ARCHIVE_READ_DAYS` | 60 天 | `CleanupArchiveReadInterval()` |
| `CLEANUP_ARCHIVE_UNREAD_DAYS` | 180 天 | `CleanupArchiveUnreadInterval()` |
| `CLEANUP_ARCHIVE_BATCH_SIZE` | 10000 条 | `CleanupArchiveBatchSize()` |

配置解析：`internal/config/options.go:125-155`

### 4.4 墓碑机制详解

#### 表结构 (`migrations.go:1472-1480`)
```sql
CREATE TABLE entry_tombstones (
    feed_id bigint NOT NULL REFERENCES feeds(id) ON DELETE CASCADE,
    hash text NOT NULL CHECK (hash <> ''),
    deleted_at timestamp with time zone NOT NULL DEFAULT now(),
    PRIMARY KEY (feed_id, hash)
);
CREATE INDEX entry_tombstones_deleted_at_idx ON entry_tombstones (deleted_at);
```

删除 Feed 时通过外键级联清理该 Feed 的所有墓碑。

#### 墓碑生命周期与清理时机

**结论：墓碑记录默认永久保留，不存在 TTL 或定期清理机制。**

经过全代码库检索，没有任何 `DELETE FROM entry_tombstones` 或 `CleanupTombstones` 之类的主动清理代码。`deleted_at` 字段虽然建有索引 `entry_tombstones_deleted_at_idx`，但在业务逻辑中从未以它为条件做过期筛选——该索引为未来扩展预留。

**唯一的墓碑清理触发点：删除 Feed**
```sql
-- entry_tombstones.feed_id 外键定义：
REFERENCES feeds(id) ON DELETE CASCADE
```
当一个 Feed 被删除时，PostgreSQL 级联删除该 Feed 关联的全部墓碑记录。删除 Feed 的路径：
- 用户主动删除单个 Feed（`DELETE FROM feeds WHERE id=?`）
- 删除其所属的 Category，外键级联删除 Feed，再级联删除墓碑
- 删除用户，级联删除全部数据

**墓碑的写入路径（共两处）**：
| 写入场景 | 代码位置 | 触发条件 |
|----------|----------|----------|
| 自动归档 | `storage/entry.go:363-404` `ArchiveEntries()` | 清理调度器或 `--run-cleanup-tasks` 时批量写入 |
| 手动清空历史 | `storage/entry.go:491-508` `FlushHistory()` | UI "清除历史记录"按钮 或 API `PUT /v1/flush-history` |

两者写入逻辑相同：`INSERT INTO entry_tombstones ... ON CONFLICT DO NOTHING`。

> **存储代价提示**：由于墓碑永久累积，对于运行多年、订阅大量 Feed 的实例，`entry_tombstones` 表可能膨胀。若需人工清理，可基于 `deleted_at` 索引执行 SQL（例如删除一年前的墓碑），但这会带来被删条目重新被摄入的副作用。

#### 墓碑检查点

**1. 创建条目时** (`storage/entry.go:117-119`)
```sql
INSERT INTO entries (...) SELECT ... WHERE NOT EXISTS (
    SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
)
```
命中墓碑 → 返回 `ErrEntryTombstoned` 不创建。

**2. 判断条目新旧时** (`storage/entry.go:278-289`)
```sql
SELECT EXISTS(entries...) OR EXISTS(entry_tombstones...)
```
墓碑化的条目视为"非新条目"，避免爬虫重复抓取全文。

---

## 五、清理任务调度

### 5.1 调度器启动

位置：`internal/cli/scheduler.go`

**启动入口** `runScheduler()` [L15-31]：
```go
go feedScheduler(...)       // Feed 刷新调度器
go cleanupScheduler(...)    // 清理任务调度器
```

**清理调度器** [L53-57]：
```go
func cleanupScheduler(store *storage.Storage, frequency time.Duration) {
    for range time.Tick(frequency) {
        runCleanupTasks(store)
    }
}
```

使用 `time.Tick()` 无限循环触发。默认频率 `CLEANUP_FREQUENCY_HOURS = 24` 小时。

### 5.1.1 配置生效时机：修改 max-days 后是否需要重启？

**结论：需要重启进程。配置仅在启动时解析一次，运行中无法动态生效。**

#### 配置加载链路

**启动时一次性解析** (`internal/cli/cli.go:80-96`)
```go
cfg := config.NewConfigParser()

if flagConfigFile != "" {
    config.Opts, err = cfg.ParseFile(flagConfigFile)  // 1. 先读配置文件
}

config.Opts, err = cfg.ParseEnvironmentVariables()     // 2. 再读环境变量（覆盖文件）
```

`ParseFile` 和 `ParseEnvironmentVariables` 都将原始值解析后写入 `configOptions.options` map 中的 `parsedDuration`、`parsedIntValue` 等字段，之后整个进程生命周期内不再重新解析。

**`config.Opts` 是包级全局单例** (`internal/config/config.go:8-9`)：
```go
var Opts *configOptions
```
没有任何 hot-reload 或文件监听机制。

#### 调度器与配置值的关系

**调度器频率**在 `runScheduler()` 启动时就已确定并固化到 `time.Tick()` 中：
```go
// internal/cli/scheduler.go:27-30
go cleanupScheduler(
    store,
    config.Opts.CleanupFrequency(),  // 启动时快照，之后不会重新读取
)
```
`time.Tick(frequency)` 创建后无法修改间隔——即使你能改 `config.Opts`，调度器频率也不会变。

**归档天数**是每次 `runCleanupTasks()` 运行时从 `config.Opts` 读取的：
```go
// internal/cli/cleanup_tasks.go:26,39
config.Opts.CleanupArchiveReadInterval()
config.Opts.CleanupArchiveUnreadInterval()
```

但这只是 getter 返回 `parsedDuration` 的静态值。因为 `config.Opts` 底层 map 没有被重新解析，所以即使环境变量已更新，读出来的仍然是启动时的值。

#### 两条独立的执行路径

| 触发方式 | 配置解析时机 | 配置变更是否生效 |
|----------|-------------|----------------|
| 守护进程定时调度 | daemon 启动时 (`cli.Parse()` 中) | ❌ 不生效，需重启 |
| CLI 手动执行 `--run-cleanup-tasks` | 每次命令执行时重新 `cli.Parse()` | ✅ 每次使用最新配置 |

手动模式每次都是独立进程：
```bash
miniflux --config-file /etc/miniflux.conf --run-cleanup-tasks
```
这条命令每次都会重新读文件和环境变量，所以能拿到最新值。

### 5.1.2 信号处理与配置热重载：为什么 SIGHUP 无效？

**结论：Miniflux 守护进程未实现任何信号驱动的配置热重载。SIGHUP/SIGUSR1/SIGUSR2 均未绑定回调。**

#### 唯一注册的信号：仅用于优雅停机

位置：`internal/cli/daemon.go:26-78`

```go
stop := make(chan os.Signal, 1)
signal.Notify(stop, os.Interrupt)   // Ctrl+C
signal.Notify(stop, syscall.SIGTERM) // systemd stop / container kill
// ← 无 SIGHUP、SIGUSR1、SIGUSR2

// ...启动 worker pool、HTTP 服务、metrics 收集器...

<-stop   // 阻塞在此，直到收到上述任一信号
slog.Debug("Shutting down the process")
// 关 metrics、关 HTTP 服务、关 worker pool 后退出
```

收到 SIGTERM 后走标准优雅关闭流程：关闭 HTTP 连接 → Shutdown worker pool → 退出。

#### 不存在的信号通路

| 常见做法 | Miniflux 是否实现 | 说明 |
|----------|-----------------|------|
| `SIGHUP` → reload 配置 | ❌ 无 | 信号未注册，按 Unix 默认行为**直接终止进程** |
| `SIGUSR1` → reload 配置 | ❌ 无 | 同上，收到即死 |
| `SIGUSR2` → 递增日志级别 | ❌ 无 | 未使用 |
| 文件监听 inotify | ❌ 无 | `config/parser.go` 是纯一次性解析，无监听器 |

> **部署陷阱**：如果你在 systemd unit 或 SysVinit 脚本（见 `contrib/sysvinit/etc/init.d/miniflux:105` 的 `restart|force-reload`）里配置了 `ExecReload` 发 SIGHUP，实际效果等同于 `kill`。请务必将 reload 改成 `systemctl restart`，而不是 `kill -HUP`。

#### 调整配置的标准方式

1. **编辑配置文件** `/etc/miniflux.conf` 或更新环境变量
2. **重启守护进程**：`systemctl restart miniflux`
3. **验证新值**（可选）：执行 `miniflux --config-dump` 看解析后的配置快照

如果不想中断服务，退而求其次是用 CLI 手动跑：
```bash
# 每次都是独立进程，会重新读配置
miniflux --config-file /etc/miniflux.conf --run-cleanup-tasks
```
但这**只影响单次手动执行**，守护进程的调度器频率和默认阈值仍需重启。

---

### 5.2 清理任务完整流程

位置：`internal/cli/cleanup_tasks.go:16-58`

四个子任务**顺序执行**，单个失败不影响其他：

```
runCleanupTasks()
├─ 1. CleanOldWebSessions()      清理过期会话
├─ 2. ArchiveEntries(read)       归档已读条目
├─ 3. ArchiveEntries(unread)     归档未读条目
└─ 4. CleanupOrphanIcons()       清理孤立图标
```

#### 任务 1：清理旧 Web 会话

位置：`storage/web_session.go:230-247`
```sql
DELETE FROM web_sessions WHERE created_at < now() - $1::interval
```
- 默认保留期：`CLEANUP_REMOVE_SESSIONS_DAYS = 30` 天
- 最小强制 1 天：`max(int(interval/(24h)), 1)`
- 返回被删除的会话数写入日志

#### 任务 2：归档已读条目
```go
store.ArchiveEntries(
    model.EntryStatusRead,
    config.Opts.CleanupArchiveReadInterval(),   // 60 天
    config.Opts.CleanupArchiveBatchSize(),      // 10000
)
```
- Prometheus 指标：`metric.ArchiveEntriesDuration` 标签 `read`
- 日志输出：`read_entries_archived` 计数

#### 任务 3：归档未读条目
```go
store.ArchiveEntries(
    model.EntryStatusUnread,
    config.Opts.CleanupArchiveUnreadInterval(), // 180 天
    config.Opts.CleanupArchiveBatchSize(),      // 10000
)
```
- Prometheus 指标：`metric.ArchiveEntriesDuration` 标签 `unread`
- 日志输出：`unread_entries_archived` 计数

> 注意：两项归档独立运行，意味着置顶/分享条目永远不被清理。

#### 任务 4：清理孤立图标

位置：`storage/icon.go:153-166`
```sql
DELETE FROM icons WHERE NOT EXISTS (
    SELECT 1 FROM feed_icons WHERE feed_icons.icon_id = icons.id
)
```

**产生孤立图标的场景**：
- Feed 被删除：外键级联删 `feed_icons` 映射表，但 `icons` 主表是按 hash 去重的共享表，不级联
- Feed 图标被替换：`StoreFeedIcon` 先插新映射再断旧关联，旧图标行悬空

---

### 5.3 并发冲突分析：手动 cleanup 与守护态调度同时执行

**结论：不会产生数据错乱或死锁。PostgreSQL 的默认隔离级别 + 语句级原子性 + SKIP LOCKED 构成了完整的并发安全屏障。**

#### 5.3.1 事务隔离级别

Miniflux 使用 Go 标准库 `database/sql` 配 PostgreSQL 驱动 `github.com/lib/pq`。所有 SQL 通过两条路径执行：

| 执行方式 | 代码模式 | 事务上下文 |
|----------|---------|-----------|
| 单语句（清理任务用） | `s.db.Exec(query, args...)` | 无显式事务 → PostgreSQL **隐式单语句事务** |
| 多语句（Feed 刷新用）| `tx, _ := s.db.Begin()` ... `tx.Commit()` | 显式事务，默认隔离级别 |

两者都没有通过 `BeginTx()` 设置隔离级别，因此使用 PostgreSQL 服务端的 **默认值 `READ COMMITTED`**。

连接池配置位置：`internal/database/postgresql.go:14-24`
```go
db.SetMaxOpenConns(maxConnections)       // 默认 20
db.SetMaxIdleConns(minConnections)       // 默认 1
db.SetConnMaxLifetime(connectionLifetime) // 默认 5 分钟
```
连接回收通过 `ConnMaxLifetime = 5 分钟` 定时轮转，`sql.DB` 层没有任何隔离级别相关的 `SET` 语句。

**READ COMMITTED 特性（与本文档相关的点）**：
- 语句内的快照在语句开始时建立
- 并发提交的 DELETE 对新语句可见
- 不会出现脏读；不做重复读，所以不存在不可重复读问题

#### 5.3.2 ArchiveEntries 的三层并发防护

`internal/storage/entry.go:368-388` 的单条 SQL 本身就是并发安全的核心：

```sql
WITH to_delete AS (
    SELECT id, feed_id, hash
    FROM entries
    WHERE status=$1 AND starred=false AND share_code='' 
      AND created_at < now() - $2::interval
    ORDER BY created_at ASC
    FOR UPDATE SKIP LOCKED   -- ★ 第 1 层：行锁避让
    LIMIT $3
),
deleted AS (
    DELETE FROM entries
    USING to_delete
    WHERE entries.id = to_delete.id   -- ★ 第 2 层：仅删除已锁行
    RETURNING entries.feed_id, entries.hash
)
INSERT INTO entry_tombstones (feed_id, hash)
SELECT ... FROM deleted
ON CONFLICT (feed_id, hash) DO NOTHING  -- ★ 第 3 层：主键去重
```

**第 1 层 — SKIP LOCKED**：如果守护进程已经锁定了一批待删行，手动进程会跳过那些行，选择下一批未锁的。两个进程拿到的候选集**完全不重叠**。结果是两个并行的 cleanup 会"瓜分"旧条目，各删各的，没有竞争也没有死锁。

**第 2 层 — 基于主键的 DELETE**：`USING to_delete` 相当于 `JOIN to_delete USING (id)`，只删除 CTE 中已选出的行，不会误删其他行。

**第 3 层 — ON CONFLICT DO NOTHING**：极端兜底场景。当用户同时点击 "清除历史"（FlushHistory）和调度器跑 ArchiveEntries 时，两者都会写墓碑；由于 SKIP LOCKED 只存在于 ArchiveEntries 中，理论上可能出现同一条 entry 两个路径各自尝试插墓碑。此时主键冲突被静默忽略，不报错。

#### 5.3.3 另外三个子任务的并发情况

| 子任务 | 并发执行的行为 | 是否安全 |
|--------|---------------|---------|
| `CleanOldWebSessions` | 两个并发 DELETE 条件相同。READ COMMITTED 下后开始的语句能看到先提交的删除，剩下的行重新判断条件；最终重复删除 0 行。| ✅ 幂等，无副作用 |
| `CleanupOrphanIcons` | 同上。由于使用 `NOT EXISTS` 子查询，删除范围会因并发而逐步收敛。| ✅ 幂等 |
| `runCleanupTasks` 整体 | 4 个子任务顺序串行；守护态调度也是顺序执行。不存在"归档和清理会话"交叉执行。 | ✅ 两个进程的执行整体也是交织而非冲突 |

#### 5.3.4 潜在的性能影响（非正确性问题）

虽然正确性无问题，但两个 cleanup 并发会造成额外开销：

1. **两倍的 PostgreSQL 短查询压力**：两个进程各自跑 CTE + DELETE + INSERT，CPU/IO 翻倍
2. **SKIP LOCKED 扫描浪费**：两个进程都扫描 `created_at` 索引找到候选，其中一个要跳过部分已锁行再重选
3. **共享池连接争抢**：默认 `DATABASE_MAX_CONNS = 20`（`internal/config/options.go:169-175`），两个 cleanup 各持一个连接不影响，但如果同时有 HTTP 高峰可能加剧等待

**建议**：避免主动并发。若守护进程已运行，不要手动 `--run-cleanup-tasks`；如想立即生效，可临时改小 `CLEANUP_FREQUENCY_HOURS` 后重启守护进程，或等待下一次定时触发。

---

## 附录 A：核心数据表关系图

```
users (1)
  │
  ├─→ categories (N)
  │     │
  │     └ hide_globally  (分类级隐藏)
  │     │
  │     └─→ feeds (N)
  │           │
  │           ├ category_id  (外键 → categories)
  │           ├ hide_globally  (Feed 级隐藏)
  │           │
  │           ├─→ entries (N)
  │           │     ├ status (unread/read)
  │           │     ├ starred bool  ★ 置顶=归档豁免
  │           │     ├ share_code   非空=归档豁免
  │           │     ├ tags text[]   PostgreSQL数组
  │           │     ├ created_at   归档判断依据
  │           │     └ changed_at
  │           │
  │           └─→ feed_icons (N)
  │                 └─→ icons (1)
  │
  ├─→ web_sessions (N)
  │     └ created_at  ← CleanOldWebSessions
  │
  └─→ entry_tombstones (通过 feed 关联)
        └ (feed_id, hash) PK  ← 防重放
```

---

## 附录 B：归档豁免清单

以下条目不参与自动归档：
1. ✅ `starred = TRUE` — 置顶/星标条目（永久保留）
2. ✅ `share_code <> ''` — 生成了公开分享链接的条目
3. ✅ 未超过 `CLEANUP_ARCHIVE_READ_DAYS`（60 天）的已读条目
4. ✅ 未超过 `CLEANUP_ARCHIVE_UNREAD_DAYS`（180 天）的未读条目

**注意**："置顶" 是用户层面唯一能主动触发的永久保留机制。

---

## 附录 C：关键文件索引

| 模块 | 文件路径 | 关键行 |
|------|----------|--------|
| Category 模型 | `internal/model/category.go` | 9-17 |
| Category 存储 | `internal/storage/category.go` | 246-290 删除迁移 |
| Entry 模型 | `internal/model/entry.go` | 27-47 starred/tags |
| Entry 归档 | `internal/storage/entry.go` | 363-404 ArchiveEntries |
| FlushHistory 手动清历史 | `internal/storage/entry.go` | 491-508 |
| 标签查询 | `internal/storage/entry_query_builder.go` | 160-166 WithTags |
| 置顶查询 | `internal/storage/entry_query_builder.go` | 62-69 WithStarred |
| 全局可见性 | `internal/storage/entry_query_builder.go` | 226-230 |
| RSS 标签提取 | `internal/reader/rss/adapter.go` | 154-156, 185-189, 290-295 |
| Feed 创建流程 | `internal/reader/handler/handler.go` | 104-192 |
| 清理任务入口 | `internal/cli/cleanup_tasks.go` | 16-58 |
| 清理调度器 | `internal/cli/scheduler.go` | 27-30, 53-57 |
| 守护进程信号处理 | `internal/cli/daemon.go` | 26-78 |
| CLI 入口与配置解析 | `internal/cli/cli.go` | 80-96 |
| 配置全局单例 | `internal/config/config.go` | 8-9 |
| 配置解析器 | `internal/config/parser.go` | 31-37, 39-51 |
| 配置默认值与 getter | `internal/config/options.go` | 125-155, 655-667 |
| 数据库连接池 | `internal/database/postgresql.go` | 13-24 |
| 数据库初始 schema | `internal/database/migrations.go` | 47-121 |
| Tombstones 迁移 | `internal/database/migrations.go` | 1470-1498 |
| 清理旧会话 | `internal/storage/web_session.go` | 229-247 |
| 清理孤立图标 | `internal/storage/icon.go` | 149-166 |
