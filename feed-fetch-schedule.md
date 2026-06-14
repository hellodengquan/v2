# 订阅源拉取调度与并发刷新机制深度分析

## 整体架构概览

订阅源的拉取调度与并发刷新由三个核心组件协同完成：

1. **调度器（Scheduler）** — 按固定节奏周期性地从数据库中筛选到期需要刷新的 Feed，生成 Job 列表
2. **工作池（Worker Pool）** — 长驻的后台 goroutine 池，并发消费 Job 队列
3. **批量构建器（BatchBuilder）** — 负责从数据库中按条件筛选并构建 Job 列表，是调度器与手动触发之间的共享查询逻辑

数据流向：

```
定时器 / 手动触发 → BatchBuilder.FetchJobs() → JobList → Pool.Push() → Worker.Run() → RefreshFeed()
```

---

## 1. 启动入口：一切从 daemon 开始

文件：`internal/cli/daemon.go:23`

```go
func startDaemon(store *storage.Storage) {
    pool := worker.NewPool(store, config.Opts.WorkerPoolSize())

    if config.Opts.HasSchedulerService() && !config.Opts.HasMaintenanceMode() {
        runScheduler(store, pool)
    }

    // HTTP 服务启动...
    server.StartWebServer(store, pool)
}
```

关键点：
- **Worker Pool 先于 Scheduler 创建**，Pool 是全局唯一的，所有刷新任务（定时 + 手动）共用同一个 Pool
- 只有启用了 Scheduler 服务且非维护模式时，才启动定时调度
- Pool 同时传给了 Web Server，使得 HTTP handler 也能向池中推送任务

---

## 2. round_robin 与 entry_frequency 调度策略的完整触发链路

### 2.1 调度器总控逻辑

文件：`internal/cli/scheduler.go:33`

```go
func feedScheduler(store *storage.Storage, pool *worker.Pool,
    frequency time.Duration, batchSize, errorLimit, limitPerHost int) {
    for range time.Tick(frequency) {
        jobs, err := store.NewBatchBuilder().
            WithBatchSize(batchSize).
            WithErrorLimit(errorLimit).
            WithoutDisabledFeeds().
            WithNextCheckExpired().
            WithLimitPerHost(limitPerHost).
            FetchJobs()

        if err != nil {
            slog.Error("Unable to fetch jobs from database", slog.Any("error", err))
        } else if len(jobs) > 0 {
            pool.Push(jobs)
        }
    }
}
```

**调度节奏由 `POLLING_FREQUENCY` 驱动**（默认 60 分钟一轮）。每轮的核心是 `BatchBuilder.FetchJobs()`，它只选择满足 `next_check_at < now()` 的 Feed。

### 2.2 round_robin 策略触发链路

| 阶段 | 操作 | 文件/行号 |
|------|------|----------|
| 1. 调度触发 | `time.Tick(POLLING_FREQUENCY)` 定时器触发 | `scheduler.go:34` |
| 2. 筛选 Job | `BatchBuilder.FetchJobs()` 选择 `next_check_at < now()` 的 Feed | `batch.go:75` |
| 3. 推送任务 | `pool.Push(jobs)` 将 Job 推送到无缓冲 channel | `pool.go:20` |
| 4. Worker 消费 | Worker 从 channel 取 Job，调用 `RefreshFeed()` | `worker.go:31` |
| 5. 计算周发布量 | `store.WeeklyFeedEntryCount()` — **但 round_robin 不使用此值** | `handler.go:214` |
| 6. 预更新 next_check_at | `originalFeed.ScheduleNextCheck(0, 0)` 以 min_interval 为基础 | `handler.go:221` |
| 7. HTTP 请求 | 发送请求，支持 ETag/Last-Modified 条件请求 | `handler.go:242` |
| 8. 重算 next_check_at | 成功后用 `refreshDelay = max(TTL, Cache-Control, Expires)` 再算一次 | `handler.go:303` |
| 9. 持久化 | `store.UpdateFeed(originalFeed)` 写入数据库 | `handler.go:366` |

**round_robin 核心公式**（`model/feed.go:122`）：

```go
interval = SchedulerRoundRobinMinInterval()           // 默认 60 min
interval = max(interval, refreshDelay)                // 不小于 HTTP 提示的延迟
interval = min(interval, SchedulerRoundRobinMaxInterval()) // 不超过 24h
NextCheckAt = time.Now().Add(interval)
```

### 2.3 entry_frequency 策略触发链路

| 阶段 | 操作 | 文件/行号 |
|------|------|----------|
| 1-4. 同 round_robin | 调度触发 → 筛选 → 推送 → 消费 | 同上 |
| 5. 计算周发布量 | `store.WeeklyFeedEntryCount()` — **entry_frequency 必须使用** | `handler.go:214` |
| 6. 预更新 next_check_at | `originalFeed.ScheduleNextCheck(weeklyCount, 0)` | `handler.go:221` |
| 7-9. 同 round_robin | HTTP 请求 → 重算 → 持久化 | 同上 |

**entry_frequency 核心公式**（`model/feed.go:126`）：

```go
if weeklyCount <= 0 {
    interval = SchedulerEntryFrequencyMaxInterval()     // 无数据用最大间隔（24h）
} else {
    // 核心：一周总时间 / (周发布量 × 因子)
    interval = (7 * 24 * time.Hour) / time.Duration(weeklyCount * factor)
    interval = min(interval, maxInterval)              // 不超过 24h
    interval = max(interval, minInterval)              // 不低于 5min
}
interval = max(interval, refreshDelay)                 // 不小于 HTTP 提示
NextCheckAt = time.Now().Add(interval)
```

### 2.4 周发布量计算（仅 entry_frequency 使用）

文件：`internal/storage/feed.go:168`

```go
func (s *Storage) WeeklyFeedEntryCount(userID, feedID int64) (int, error) {
    query := `
        SELECT
            COALESCE(CAST(CEIL(
                (EXTRACT(epoch from interval '1 week')) /
                NULLIF((EXTRACT(epoch from (max(published_at)-min(published_at))/NULLIF((count(*)-1), 0) )), 0)
            ) AS BIGINT), 0)
        FROM entries
        WHERE
            entries.user_id=$1 AND
            entries.feed_id=$2 AND
            entries.published_at >= now() - interval '1 week';
    `
```

**计算逻辑解读**：
1. 只统计最近一周发布的条目
2. 用 `(max(published_at) - min(published_at)) / (count - 1)` 计算平均发布间隔
3. 用 `一周总秒数 / 平均间隔秒数` 估算周发布量
4. `NULLIF` 处理除零情况（只有 1 条或 0 条时，`count-1=0` 返回 0）
5. `CEIL` 向上取整，`COALESCE` 将 NULL 转为 0

### 2.5 refreshDelay 的来源

在 `RefreshFeed` 执行过程中，会从 HTTP 响应中提取多种刷新提示（`handler.go:297`）：

```go
feedTTLValue := updatedFeed.TTL                     // RSS <ttl> 字段
cacheControlMaxAgeValue := responseHandler.CacheControlMaxAge() // Cache-Control: max-age=N
expiresValue := responseHandler.Expires()           // Expires 头
refreshDelay := max(feedTTLValue, cacheControlMaxAgeValue, expiresValue)
```

如果遇到 429 限流，直接使用 `Retry-After` 头（`handler.go:246`）：

```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
}
```

`ParseRetryDelay` 支持两种格式（`response_handler.go:82`）：
- 整数秒数：`Retry-After: 120` → 120 秒
- HTTP 日期：`Retry-After: Wed, 21 Oct 2015 07:28:00 GMT` → 计算到该时间的差值

---

## 3. next_check_at 的事务边界与更新时机

这是系统中**最微妙**的部分之一。`next_check_at` 在一次刷新过程中可能被多次更新，且事务边界分散。

### 3.1 next_check_at 的三次内存更新

在 `RefreshFeed` 函数中，`next_check_at` 在内存中被修改了 **3 次**，但只有最后一次被持久化：

| 时机 | 代码位置 | 调用参数 | 目的 |
|------|---------|---------|------|
| 1. HTTP 请求前 | `handler.go:221` | `ScheduleNextCheck(weeklyCount, 0)` | **乐观预更新**，假设这次刷新会成功，防止其他调度周期重复选中 |
| 2. 遇到 429 限流 | `handler.go:247` | `ScheduleNextCheck(weeklyCount, retryDelay)` | 用服务器建议的 `Retry-After` 更新 |
| 3. 解析成功后 | `handler.go:303` | `ScheduleNextCheck(weeklyCount, refreshDelay)` | 用 TTL/Cache-Control/Expires 更新最终间隔 |

**重要设计缺陷**：第 1 次更新在 HTTP 请求前就已发生，且**此时该值尚未写入数据库**。如果调度周期短于 HTTP 请求耗时，可能导致：
- 调度周期 T1 选中 Feed F，开始刷新
- 调度周期 T2 在 T1 的 HTTP 请求完成前运行，`next_check_at < now()` 仍然成立，F 再次被选中
- Feed F 被并发刷新两次

### 3.2 持久化的事务边界

所有数据库操作都使用**自动提交的独立事务**（没有 `BEGIN` / `COMMIT` 包裹整个刷新流程）：

| 操作 | 函数 | 事务范围 | 是否更新 next_check_at |
|------|------|---------|-----------------------|
| 读取 Feed | `FeedByID()` | 单条 SELECT | ❌ |
| 查询周发布量 | `WeeklyFeedEntryCount()` | 单条 SELECT | ❌ |
| 刷新条目 | `RefreshFeedEntries()` | **每条 entry 一个独立事务** | ❌ |
| 更新 Feed 元数据 | `UpdateFeed()` | 单条 UPDATE | ✅ |
| 更新错误信息 | `UpdateFeedError()` | 单条 UPDATE | ✅ |

**UpdateFeed 的 SQL**（`storage/feed.go:329`）：

```go
func (s *Storage) UpdateFeed(feed *model.Feed) (err error) {
    query := `
        UPDATE feeds SET
            feed_url=$1, ...,
            checked_at=$7,
            next_check_at=$22,  -- 持久化内存中的 NextCheckAt
            ...
        WHERE id=$40 AND user_id=$41
    `
    _, err = s.db.Exec(query, ..., feed.NextCheckAt, ...)
}
```

**关键结论**：
1. **没有使用 `SELECT ... FOR UPDATE` 行锁** 来防止并发修改
2. **没有将整个刷新过程包裹在一个事务中**
3. `next_check_at` 的更新是**最后一条语句**，如果 HTTP 请求耗时很长，期间可能被重复调度
4. 每次数据库调用都是独立的自动提交事务，这意味着中间任何一步失败都不会回滚之前的操作

### 3.3 失败路径的 next_check_at 更新

如果刷新过程中出现错误（HTTP 错误、解析错误等），通过 `getTranslatedLocalizedError()` 调用 `UpdateFeedError()`：

```go
func getTranslatedLocalizedError(...) *locale.LocalizedErrorWrapper {
    originalFeed.WithTranslatedErrorMessage(message)  // 递增 ParsingErrorCount
    store.UpdateFeedError(originalFeed)              // 持久化错误信息和 next_check_at
    return localizedError
}
```

`UpdateFeedError` 也会更新 `next_check_at`（`storage/feed.go:427`）：

```go
UPDATE feeds SET
    parsing_error_msg=$1,
    parsing_error_count=$2,
    checked_at=$3,
    next_check_at=$4    -- 错误时也更新，使用第 1 次 ScheduleNextCheck 计算的值
WHERE id=$5 AND user_id=$6
```

这意味着：
- 失败时 `next_check_at` 使用的是**第 1 次预更新**的值（HTTP 请求前那次）
- 成功时使用的是**第 3 次更新**的值（考虑了 TTL 等）

---

## 4. 单主机限制（limitPerHost）防止同源打爆的实现细节

### 4.1 实现位置与机制

文件：`internal/storage/batch.go:107`

```go
func (b *batchBuilder) FetchJobs() (model.JobList, error) {
    // SQL 查询...
    query := `SELECT id, user_id, feed_url FROM feeds`
    // ... WHERE ... ORDER BY next_check_at ASC LIMIT batchSize
    rows, err := b.db.Query(query, b.args...)

    jobs := make(model.JobList, 0, b.batchSize)
    hosts := make(map[string]int)  // 内存中的 host 计数器

    for rows.Next() {
        // 扫描 job...
        if b.limitPerHost > 0 {
            feedHostname := urllib.Domain(job.FeedURL)
            if hosts[feedHostname] >= b.limitPerHost {
                nbSkippedFeeds++
                continue  // 超过同一 host 限制，跳过
            }
            hosts[feedHostname]++
        }
        jobs = append(jobs, job)
    }
}
```

### 4.2 域名提取规则

`urllib.Domain()` 的实现（`urllib/url.go:132`）：

```go
func Domain(websiteURL string) string {
    parsedURL, err := url.Parse(websiteURL)
    if err != nil {
        return websiteURL
    }
    return parsedURL.Host  // 包含端口号！
}
```

**重要细节**：
- 返回 `parsedURL.Host`，包含端口号。例如 `https://example.com:8080/feed` → `example.com:8080`
- `www.example.com` 和 `example.com` 被视为**不同的 host**
- 解析失败时返回原始 URL 字符串（可能导致误判）

### 4.3 限制的作用范围

| 特性 | 说明 |
|------|------|
| **限制粒度** | 每批次（每轮调度）的限制，不是全局并发限制 |
| **限制位置** | 内存过滤，SQL 仍然返回所有匹配行 |
| **排序影响** | SQL 按 `next_check_at ASC` 排序，同一 host 下最久未刷新的优先被选中 |
| **默认值** | `POLLING_LIMIT_PER_HOST=0`（不限） |
| **共享范围** | 定时调度、Web UI 刷新、API 刷新、CLI 刷新都使用此限制 |

### 4.4 实际效果示例

假设 `POLLING_LIMIT_PER_HOST=2`，`BATCH_SIZE=100`，且有以下 Feed：

| Feed | Host | next_check_at |
|------|------|---------------|
| F1 | example.com | 09:00 |
| F2 | example.com | 09:05 |
| F3 | example.com | 09:10 |
| F4 | other.com | 09:15 |

查询结果按 `next_check_at ASC` 返回 F1, F2, F3, F4...

内存过滤过程：
1. F1 → `hosts["example.com"] = 1` → 加入 jobs
2. F2 → `hosts["example.com"] = 2` → 加入 jobs
3. F3 → `hosts["example.com"] >= 2` → **跳过**
4. F4 → `hosts["other.com"] = 1` → 加入 jobs

最终结果：同一 host 每批最多 2 个 Job，即使该 host 还有更多到期的 Feed。

---

## 5. Worker Pool 的锁与重试机制

### 5.1 Pool 结构与同步机制

文件：`internal/worker/pool.go`

```go
type Pool struct {
    queue chan model.Job   // 无缓冲 channel
    wg    sync.WaitGroup
}
```

**关键发现**：
- **没有使用 `sync.Mutex` 或 `sync.RWMutex`**
- 所有同步都通过**无缓冲 channel** 完成

### 5.2 无缓冲 channel 的同步语义

```go
func NewPool(store *storage.Storage, nbWorkers int) *Pool {
    workerPool := &Pool{
        queue: make(chan model.Job),  // 无缓冲！
    }
    for i := range nbWorkers {
        workerPool.wg.Add(1)
        worker := &worker{id: i, store: store}
        go worker.Run(workerPool.queue, &workerPool.wg)
    }
    return workerPool
}
```

无缓冲 channel 的特性：
- **发送操作 (`p.queue <- job`) 会阻塞**，直到有接收方（Worker）执行接收操作
- **接收操作 (`job := <-c`) 也会阻塞**，直到有发送方执行发送操作
- 这形成了**天然的同步点**和**背压机制**

### 5.3 Push 方法的阻塞行为

```go
func (p *Pool) Push(jobs model.JobList) {
    for _, job := range jobs {
        p.queue <- job  // 逐个同步发送
    }
}
```

**背压效果**：
- 如果 16 个 Worker 都在忙，第 17 个 Job 的 `p.queue <- job` 会阻塞
- 调度器 goroutine 会被阻塞，直到某个 Worker 完成当前任务并接收新的 Job
- 这防止了任务无限堆积，但也意味着调度周期可能被拉长

### 5.4 没有内置的重试机制

**整个刷新流程中没有任何重试循环**：

1. **Worker 层面**：`worker.Run()` 收到 Job 后只调用一次 `RefreshFeed()`，失败就结束
2. **Handler 层面**：`RefreshFeed()` 没有 `for retry := 0; retry < 3; retry++` 这类循环
3. **HTTP 客户端层面**：标准库 `http.Client` 默认不重试，也没有自定义重试逻辑

**"重试"的唯一方式**：
- 失败时 `UpdateFeedError()` 更新 `parsing_error_count` 和 `next_check_at`
- 下一轮调度时，如果 `parsing_error_count < POLLING_PARSING_ERROR_LIMIT`，Feed 会被再次选中
- 这是一种**延迟重试**，而非即时重试

错误计数机制（`model/feed.go:100`）：
```go
func (f *Feed) WithTranslatedErrorMessage(message string) {
    f.ParsingErrorCount++          // 每次失败递增
    f.ParsingErrorMsg = message
}

func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0        // 成功时清零
    f.ParsingErrorMsg = ""
}
```

当 `ParsingErrorCount >= POLLING_PARSING_ERROR_LIMIT`（默认 3）时，`BatchBuilder.WithErrorLimit()` 会过滤掉该 Feed，直到用户手动修复或重置错误计数。

### 5.5 并发安全分析

| 组件 | 并发安全 | 说明 |
|------|---------|------|
| **Worker Pool** | ✅ 安全 | 通过无缓冲 channel 同步，没有共享可变状态 |
| **BatchBuilder** | ✅ 安全 | 每次调用 `NewBatchBuilder()` 创建新实例，无共享 |
| **RefreshFeed** | ⚠️ 有风险 | 没有行锁，同一 Feed 可能被并发刷新 |
| **RefreshFeedEntries** | ✅ 安全 | 每条 entry 独立事务，使用 `entryExists()` 检查 |
| **数据库连接** | ✅ 安全 | `*sql.DB` 本身是并发安全的 |

**潜在竞争条件**：
1. 调度周期 T1 选中 Feed F，开始 HTTP 请求
2. T1 的 `ScheduleNextCheck` 更新了内存中的 `NextCheckAt`，但**尚未写入数据库**
3. 调度周期 T2 开始，查询 `next_check_at < now()` 仍然命中 F
4. T2 也开始刷新 F
5. T1 完成，`UpdateFeed` 写入 `next_check_at`
6. T2 完成，`UpdateFeed` 再次写入，可能覆盖 T1 的结果

**缓解措施**：
- `UpdateFeed` 的 WHERE 条件是 `id=$40 AND user_id=$41`，没有版本号或乐观锁
- 实际影响有限，最多是重复刷新，不会造成数据损坏

---

## 6. 批量构建器：Job 筛选的核心逻辑

文件：`internal/storage/batch.go`

`BatchBuilder` 采用链式调用模式，逐步添加 SQL WHERE 条件，最终调用 `FetchJobs()` 执行查询。

### 6.1 筛选条件

| 方法 | SQL 条件 | 说明 |
|------|---------|------|
| `WithBatchSize(n)` | `LIMIT n` | 限制每批最多取 n 条 |
| `WithUserID(id)` | `user_id = $x` | 指定用户（手动刷新时使用） |
| `WithCategoryID(id)` | `category_id = $x` | 指定分类（分类刷新时使用） |
| `WithErrorLimit(n)` | `parsing_error_count < $x` | 跳过错误过多的 Feed |
| `WithNextCheckExpired()` | `next_check_at < now()` | 只取到期需刷新的 Feed |
| `WithoutDisabledFeeds()` | `disabled IS false` | 跳过被禁用的 Feed |
| `WithLimitPerHost(n)` | 内存过滤 | 同一域名不超过 n 个 Job |

### 6.2 不同触发场景下的 Builder 使用差异

| 触发方式 | WithBatchSize | WithErrorLimit | WithNextCheckExpired | WithUserID | WithCategoryID |
|---------|:---:|:---:|:---:|:---:|:---:|
| **定时调度** | ✅ | ✅ | ✅ | ❌ | ❌ |
| **Web UI 全部刷新** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Web UI 分类刷新** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **API 全部刷新** | ❌ | ✅ | ✅ | ✅ | ❌ |
| **CLI 手动刷新** | ✅ | ✅ | ✅ | ❌ | ❌ |

关键差异：
- **手动刷新（Web UI）** 不设置 `WithErrorLimit` 和 `WithNextCheckExpired`，意味着强制刷新所有未禁用的 Feed，不论错误次数和到期时间
- **定时调度** 则严格遵守到期和错误限制
- **API 刷新** 介于两者之间：有错误限制和到期检查，但限定了用户范围

---

## 7. 手动刷新与限流

### 7.1 Web UI 手动刷新的限流

文件：`internal/ui/feed_refresh.go:38`

```go
if time.Since(sess.LastForceRefresh()) < config.Opts.ForceRefreshInterval() {
    // 拒绝，提示刷新过于频繁
}
```

- `FORCE_REFRESH_INTERVAL` 默认 **30 分钟**
- 基于会话（Session）级别限流，`sess.MarkForceRefreshed()` 记录上次强制刷新时间
- 全部刷新和分类刷新都受此限制

### 7.2 单个 Feed 手动刷新

文件：`internal/ui/feed_refresh.go:18`

```go
func (h *handler) refreshFeed(w http.ResponseWriter, r *http.Request) {
    feedHandler.RefreshFeed(h.store, request.UserID(r), feedID, forceRefresh)
}
```

- 单个刷新是**同步执行**的（不走 Pool），直接调用 `RefreshFeed`
- 支持 `forceRefresh` 参数，跳过 HTTP 缓存（ETag / Last-Modified）
- 不受 `ForceRefreshInterval` 限制

---

## 8. 配置参数速查表

| 环境变量 | 默认值 | 说明 |
|---------|--------|------|
| `POLLING_FREQUENCY` | 60 min | 调度器轮询间隔 |
| `BATCH_SIZE` | 100 | 每轮最多选取的 Feed 数 |
| `WORKER_POOL_SIZE` | 16 | 并发 Worker 数量 |
| `POLLING_SCHEDULER` | `round_robin` | 调度策略：`round_robin` 或 `entry_frequency` |
| `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL` | 60 min | Round Robin 最小间隔 |
| `SCHEDULER_ROUND_ROBIN_MAX_INTERVAL` | 1440 min (24h) | Round Robin 最大间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_MIN_INTERVAL` | 5 min | Entry Frequency 最小间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_MAX_INTERVAL` | 1440 min (24h) | Entry Frequency 最大间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_FACTOR` | 1 | Entry Frequency 因子（越大间隔越长） |
| `POLLING_PARSING_ERROR_LIMIT` | 3 | 解析错误次数上限 |
| `POLLING_LIMIT_PER_HOST` | 0（不限） | 每轮每域名最大 Job 数 |
| `FORCE_REFRESH_INTERVAL` | 30 min | 手动全部刷新的会话级限流间隔 |

---

## 9. 完整流程串联

### 9.1 定时自动刷新流程

```
1. Daemon 启动
   ├── 创建 Worker Pool（16 个 goroutine 监听无缓冲 channel）
   └── 启动 feedScheduler goroutine

2. 每 60 分钟（POLLING_FREQUENCY）
   └── BatchBuilder 查询数据库
       ├── WHERE next_check_at < now()     （到期检查）
       ├── AND parsing_error_count < 3     （错误限制）
       ├── AND disabled IS false           （跳过禁用）
       ├── ORDER BY next_check_at ASC      （优先最过期的）
       ├── LIMIT 100                       （批量大小）
       └── 内存中按 host 去重（limitPerHost）
   └── pool.Push(jobs)
       └── 逐个发送到无缓冲 channel
           └── Worker 取走 Job → 执行 RefreshFeed()
               ├── 预更新 next_check_at（HTTP 请求前！）
               ├── 发送 HTTP 请求（支持 ETag / Last-Modified 条件请求）
               ├── 若 304 Not Modified → 跳过，不解析
               ├── 若 429 Too Many Requests → 用 Retry-After 更新 next_check_at
               ├── 若 200 OK → 解析 Feed 内容 → 逐条 entry 独立事务入库
               ├── 用 TTL/Cache-Control/Expires 重算 next_check_at
               └── UpdateFeed 持久化 next_check_at → 影响下一轮调度

3. 下一轮调度时，已刷新的 Feed 因 next_check_at 尚未到期，不再被选中
```

### 9.2 手动 Web 全部刷新流程

```
1. 用户点击"刷新全部"
   ├── 检查 Session 级限流（30 分钟内不允许重复）
   └── BatchBuilder 查询（仅限当前用户，无到期/错误限制）
       └── go pool.Push(jobs)  // 异步推送
   └── 立即返回页面，后台执行
```

---

## 11. Webhook 与 WebSub/PubSubHubbub 推送订阅

### 11.1 Webhook 推送调度链路

Webhook 是**拉取模式的补充**，而非替代。它在拉取完成后，将新条目异步推送到第三方系统。

**触发时机**（`internal/reader/handler/handler.go:338`）：

```go
if userIntegrations != nil && len(newEntries) > 0 {
    go integration.PushEntries(originalFeed, newEntries, userIntegrations)
}
```

**关键特性**：
- **异步执行**：使用 `go` 关键字在独立 goroutine 中推送，不阻塞主刷新流程
- **触发条件**：只有当拉取到**新条目**（`len(newEntries) > 0`）且用户启用了 webhook 时才触发
- **两种事件类型**（`internal/integration/webhook/webhook.go:24`）：
  - `new_entries`：批量推送本次刷新发现的所有新条目
  - `save_entry`：用户点击"保存"按钮时单独推送某个条目

**完整推送链路**：

```
1. Worker 执行 RefreshFeed()
   ├── 拉取 Feed 内容
   ├── 解析并对比，找出 newEntries
   ├── 持久化条目到数据库
   └── go integration.PushEntries()  ← 异步触发
       └── webhook.NewClient().SendNewEntriesWebhookEvent()
           ├── 构造 JSON payload（Feed + Entries）
           ├── 计算 X-Miniflux-Signature（HMAC-SHA256，用 webhookSecret）
           ├── POST 到配置的 webhook URL
           └── 超时 10 秒，无重试
```

**与拉取模式的关系**：
- **互补关系**，而非覆盖
- 拉取模式负责**获取**内容，Webhook 负责**分发**内容
- 即使启用了 Webhook，拉取调度仍然正常运行
- Webhook 推送失败**不影响**拉取流程，只是打日志记录

### 11.2 WebSub/PubSubHubbub 支持现状

**重要结论**：Miniflux v2 **不支持 WebSub/PubSubHubbub 推送订阅**。

代码中仅有的 WebSub 痕迹在 JSON Feed 解析的结构体定义中（`internal/reader/json/json.go:55`）：

```go
type JSONFeed struct {
    // ...
    // Hubs  describes endpoints that can be used to subscribe to real-time notifications
    // from the publisher of this feed.
    Hubs []JSONHub `json:"hubs"`
}

type JSONHub struct {
    // Type defines the protocol used to talk with the hub: "rssCloud" or "WebSub".
    Type string `json:"type"`
    // URL is the location of the hub.
    URL string `json:"url"`
}
```

**缺失的 WebSub 功能**：
1. ❌ 没有订阅流程（向 Hub 发送 `hub.mode=subscribe` 请求）
2. ❌ 没有回调接口（`/websub/callback` 路由不存在）
3. ❌ 没有 Hub 订阅状态管理（订阅有效期、续租等）
4. ❌ 没有签名验证（`X-Hub-Signature`）
5. ❌ 没有将解析到的 Hub URL 存入数据库（`feeds` 表无相关字段）

**实际业务建议**：如果需要实时推送，目前只能依赖 Webhook + 缩短拉取间隔的组合方案。

---

## 12. host-limit 命中时的处理策略与 retry 机制

### 12.1 命中时的行为：丢弃而非排队

当 `limitPerHost` 命中时，Feed 会被**直接丢弃**（从当前批次中排除），不会排队等待。

**代码逻辑**（`internal/storage/batch.go:107`）：

```go
if b.limitPerHost > 0 {
    feedHostname := urllib.Domain(job.FeedURL)
    if hosts[feedHostname] >= b.limitPerHost {
        slog.Debug("Feed host limit reached for this batch", ...)
        nbSkippedFeeds++
        continue  // ← 直接跳过，不加入 jobs
    }
    hosts[feedHostname]++
}
```

**关键细节**：
- 被跳过的 Feed 仍然满足 `next_check_at < now()`，但**不会更新 `next_check_at`**
- 这些 Feed 会**在下一轮调度中重新被考虑**
- `nbSkippedFeeds` 会记录到日志中（`batch.go:132`），方便排查

**示例场景**：

假设 `POLLING_LIMIT_PER_HOST=2`，`BATCH_SIZE=100`，某域名下有 5 个到期 Feed：

| 调度轮次 | 选中的 Feed | 被跳过的 Feed |
|---------|------------|-------------|
| 第 1 轮（T0） | F1, F2 | F3, F4, F5 |
| 第 2 轮（T0+60min） | F3, F4 | F5 |
| 第 3 轮（T0+120min） | F5 | - |

### 12.2 Retry 策略：依赖下一轮调度

**没有专门的 retry 队列**。被 `limitPerHost` 过滤掉的 Feed，其重试完全依赖下一轮调度的自然重选。

**重试周期**：
- 最快在下一个 `POLLING_FREQUENCY`（默认 60 分钟）后被重新选中
- 如果某域名下到期 Feed 很多，可能需要多轮才能全部刷完

**与错误重试的区别**：

| 场景 | 处理方式 | next_check_at 变化 |
|------|---------|-------------------|
| `limitPerHost` 过滤 | 跳过，下轮重试 | **不变**，仍满足 `< now()` |
| HTTP 4xx/5xx 错误 | 跳过，下轮重试 | **更新**，`parsing_error_count++` |
| 刷新成功 | 正常入库 | **更新**，按策略计算新值 |

**风险**：如果某个域名下有大量 Feed，且 `limitPerHost` 设置较小，可能导致部分 Feed 长期"饿肚子"，因为每次都被排在后面的 Feed 顶替。

---

## 13. HTTP 429 服务端限速与本地调度间隔的协调处理

### 13.1 429 检测与 Retry-After 解析

**检测逻辑**（`internal/reader/fetcher/response_handler.go:98`）：

```go
func (r *ResponseHandler) IsRateLimited() bool {
    return r.httpResponse != nil && r.httpResponse.StatusCode == http.StatusTooManyRequests
}
```

**Retry-After 解析**（`response_handler.go:82`）支持两种格式：

```go
func (r *ResponseHandler) ParseRetryDelay() time.Duration {
    // 1) 整数秒数：Retry-After: 120 → 120 秒
    if seconds, err := strconv.Atoi(retryAfterHeaderValue); err == nil {
        return time.Duration(seconds) * time.Second
    }
    // 2) HTTP 日期：Retry-After: Wed, 21 Oct 2015 07:28:00 GMT → 计算差值
    if t, err := time.Parse(time.RFC1123, retryAfterHeaderValue); err == nil {
        return time.Until(t).Truncate(time.Second)
    }
    return 0
}
```

### 13.2 与本地调度间隔的协调机制

**处理流程**（`internal/reader/handler/handler.go:245`）：

```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
    // 记录日志后直接返回，不解析内容
}
```

**核心协调逻辑在 `ScheduleNextCheck`** 中（`model/feed.go:150`）：

```go
interval = max(interval, refreshDelay)  // refreshDelay 就是 retryDelay
```

**优先级规则**：`Retry-After` 作为**下限**，确保不会比服务器建议的更频繁。

**不同场景下的最终间隔**：

| 场景 | 本地计算间隔 | Retry-After | 最终间隔 |
|------|-------------|-------------|---------|
| 正常刷新 | 60 min | 0 | 60 min |
| 429，服务器建议 5 分钟 | 60 min | 5 min | 60 min（本地间隔更大） |
| 429，服务器建议 2 小时 | 60 min | 120 min | 120 min（服务器建议更大） |
| entry_frequency，高频源 | 15 min | 30 min | 30 min（服务器建议更大） |

**边界情况处理**：
- 如果 `Retry-After` 解析失败（格式不合法），`ParseRetryDelay()` 返回 0，此时完全使用本地计算的间隔
- 如果 `Retry-After` 是过去的时间（HTTP 日期格式），`time.Until(t)` 返回负值，`max()` 会忽略它
- 429 状态下仍然会执行 `UpdateFeedError()`，持久化新的 `next_check_at`

---

## 14. 多实例 HA 部署下的 Feed 去重机制

### 14.1 现状：无显式分布式锁，存在竞态风险

**重要结论**：Miniflux v2 的 `FetchJobs()` 查询**没有使用 `SELECT ... FOR UPDATE SKIP LOCKED`** 来防止多实例并发选中同一 Feed。

**BatchBuilder 的 SQL 查询**（`internal/storage/batch.go:76`）：

```sql
SELECT id, user_id, feed_url FROM feeds
WHERE ...
ORDER BY next_check_at ASC
LIMIT 100
```

**对比有锁的查询**（仅在 `ArchiveEntries` 中使用，`entry.go:378`）：

```sql
SELECT id, feed_id, hash FROM entries
WHERE ...
ORDER BY created_at ASC
FOR UPDATE SKIP LOCKED  -- ← 防止并发修改
LIMIT $3
```

### 14.2 竞态场景分析

部署两个实例 A 和 B，连接同一个 PostgreSQL 数据库：

```
时间线：
T0: 实例 A 的调度器运行 → 执行 SELECT，读取 Feed F（next_check_at = T0-10min）
T0 + 1ms: 实例 B 的调度器运行 → 执行 SELECT，也读取 Feed F（next_check_at 仍为 T0-10min）
T0 + 100ms: 实例 A 的 Worker 开始刷新 F，预更新内存中的 next_check_at = T0 + 60min
T0 + 200ms: 实例 B 的 Worker 也开始刷新 F，预更新内存中的 next_check_at = T0 + 60min
T0 + 500ms: 实例 A 完成刷新，执行 UPDATE feeds SET next_check_at = ...
T0 + 600ms: 实例 B 完成刷新，执行 UPDATE feeds SET next_check_at = ...（覆盖 A 的结果）
```

**后果**：
- Feed F 在同一调度周期内被刷新两次
- 第二次刷新会收到 304 Not Modified（如果源站支持 ETag），但仍浪费了一次 HTTP 请求
- `next_check_at` 被覆盖，但值相同（因为计算逻辑相同），所以最终结果一致
- 没有数据损坏，只是浪费了资源

### 14.3 隐性的去重/防护机制

虽然没有显式分布式锁，但有一些机制降低了冲突概率：

1. **`next_check_at` 乐观预更新**：
   - HTTP 请求前就在内存中更新 `next_check_at`（虽然还没写入数据库）
   - 但这个更新是实例本地的，对其他实例不可见

2. **ETag / Last-Modified 条件请求**：
   - 即使同一 Feed 被并发刷新，第二次请求会带上第一次请求获得的 ETag
   - 源站如果支持，会返回 304 Not Modified，节省了解析和入库的开销

3. **`AnotherFeedURLExists` 检查**（`handler.go:267`）：
   - 防止同一用户订阅重复的 Feed URL
   - 但这不防止同一 Feed 被并发刷新

4. **`RefreshFeedEntries` 的条目去重**（`entry.go:329`）：
   - 每条 entry 入库前检查 `entryExists()`，防止重复插入
   - 即使 Feed 被并发刷新，条目也不会重复

### 14.4 HA 部署的实际建议

**官方推荐**：单实例运行调度器，多实例运行 Web 服务。

**可行的 HA 方案**：

1. **主备模式**：
   - 只有一个实例启用调度器（`SCHEDULER_SERVICE=true`）
   - 其他实例只提供 Web UI 和 API（`SCHEDULER_SERVICE=false`）
   - 用外部机制（如 Kubernetes livenessProbe + leader election）实现故障转移

2. **时间偏移**：
   - 多个实例都启用调度器，但设置不同的 `POLLING_FREQUENCY` 偏移
   - 简单但不严谨，仍有竞态可能

3. **外部分布式锁**（需自行实现）：
   - 在 `FetchJobs()` 前获取 Redis / etcd 分布式锁
   - 或修改 SQL 添加 `FOR UPDATE SKIP LOCKED`

### 14.5 为什么不启用 `SKIP LOCKED`？

从代码来看，`ArchiveEntries` 已经在使用 `FOR UPDATE SKIP LOCKED`，说明开发团队了解这个特性。`FetchJobs` 不使用可能的原因：

1. **事务边界问题**：`FOR UPDATE` 需要在事务中使用，但 `FetchJobs` 是只读查询，没有开启事务
2. **性能权衡**：加锁会增加数据库开销，对于大多数单实例部署是不必要的
3. **竞态后果轻微**：如前所述，并发刷新只会浪费 HTTP 请求，不会造成数据损坏

---

## 15. 关键设计要点总结（补充版）

1. **调度与执行解耦**：Scheduler 只负责生产 Job，Worker Pool 只负责消费 Job，通过 channel 连接
2. **无缓冲 channel 背压**：Pool 的 channel 无缓冲，生产速度受限于消费速度，防止任务堆积
3. **双层限流**：调度层有 `limitPerHost` 限制同主机并发；应用层有 `ForceRefreshInterval` 限制手动操作频率
4. **自适应调度**：`entry_frequency` 模式根据 Feed 的实际发布频率动态调整检查间隔，高频源检查更频繁
5. **HTTP 协议尊重**：`refreshDelay` 将 TTL / Cache-Control / Expires / Retry-After 纳入调度间隔的下限，避免对源站过度请求
6. **手动刷新绕过部分限制**：Web UI 的全部刷新不受 `errorLimit` 和 `nextCheckExpired` 限制，确保用户可以强制拉取
7. **next_check_at 预更新**：HTTP 请求前就更新内存中的 `next_check_at`，是一种乐观并发控制，但存在竞态风险
8. **无行锁无大事务**：整个刷新流程没有使用 `SELECT ... FOR UPDATE`，也没有全局事务，每次数据库操作独立提交
9. **无即时重试机制**：刷新失败后不立刻重试，而是通过 `parsing_error_count` 和下一轮调度实现延迟重试
10. **域名包含端口**：`limitPerHost` 使用完整 Host（含端口），`www.example.com` 和 `example.com` 视为不同主机
11. **Webhook 是拉取的补充**：异步推送新条目到第三方，不阻塞主流程，失败不影响拉取
12. **WebSub 不支持**：仅在 JSON Feed 解析中定义了 Hub 结构体，但无实际订阅/回调机制
13. **host-limit 命中即丢弃**：被过滤的 Feed 不排队，依赖下一轮调度重试，可能导致长尾 Feed 饥饿
14. **429 协调策略**：`Retry-After` 作为调度间隔下限，与本地计算间隔取最大值
15. **HA 部署无分布式锁**：多实例部署时可能并发刷新同一 Feed，但 ETag 和条目去重机制降低了负面影响
