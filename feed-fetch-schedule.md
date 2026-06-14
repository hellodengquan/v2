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

### 11.3 关于 PubSubHubbub Hub 不可达的 fallback 机制

**重要结论**：由于 Miniflux v2 根本不支持 WebSub/PubSubHubbub 订阅，因此也**不存在 Hub 不可达时的 fallback 回退机制**。

但从架构设计角度分析，如果未来实现 WebSub 支持，合理的 fallback 策略应该是怎样的？结合现有拉取调度的设计，可以推断出以下模式：

#### 理论上的 fallback 分层架构

```
第一层：WebSub 推送（实时）
   ↓ Hub 不可达 / 订阅过期 / 推送失败
第二层：拉取轮询（兜底）
```

#### 现有代码中隐含的"类 fallback"模式

虽然没有 WebSub，但代码中已经存在类似的"协议降级"思路，可以参考：

1. **HTTP 缓存协议的降级**（`response_handler.go:102`）：
   - 优先使用 ETag（强校验）
   - ETag 不存在时降级使用 Last-Modified（弱校验）
   - 都没有时每次全量拉取

2. **TTL/Cache-Control/Expires 的多源取 max**（`handler.go:297`）：
   - 多个刷新提示同时存在时取最大值
   - 这不是 fallback，而是叠加，但体现了"多信号融合"的设计思想

#### 如果实现 WebSub，可能的 fallback 设计推断

基于 Miniflux 当前的代码风格，可能会这样实现：

| 场景 | 处理方式 |
|------|---------|
| Hub 订阅成功 | 将该 Feed 的轮询间隔拉长（如从 60min 改为 24h），推送为主、轮询兜底 |
| Hub 返回错误 / 不可达 | 立即回退到正常轮询间隔，不等待订阅过期 |
| 长时间未收到推送（超过 N 个周期） | 触发一次主动拉取，验证 Feed 是否仍然有效 |
| 退避期间重新订阅成功 | 恢复拉长的轮询间隔 |

**为什么当前版本不实现 WebSub**：
- WebSub 需要公网可达的回调 URL，很多私有化部署的 Miniflux 不满足
- Hub 生态碎片化，不是所有 Feed 源都提供 Hub
- 拉取模式已经能满足绝大多数场景，实现成本与收益不匹配

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

### 13.3 429 退避对跨 host 调度公平性的影响

当多个 host 同时存在时，429 退避会改变各 host 之间的调度资源分配，对公平性产生复杂影响。

#### 13.3.1 429 的副作用：parsing_error_count 递增

429 不仅会推远 `next_check_at`，还会导致**解析错误计数递增**。

**原因分析**（`handler.go:245` → `handler.go:286`）：

```
1. IsRateLimited() == true → 更新内存中的 next_check_at
2. LocalizedError() → 返回 nil（429 是有效 HTTP 响应，不是连接错误）
3. IsModified() → 返回 true（429 不是 304）
4. ReadBody() → 读取错误响应体（通常是 HTML 错误页）
5. ParseFeed() → 解析失败（错误页不是有效的 Feed 格式）
6. getTranslatedLocalizedError() → parsing_error_count++，持久化
```

**后果**：
- 连续 3 次 429 后，`parsing_error_count >= POLLING_PARSING_ERROR_LIMIT`（默认 3）
- 该 Feed 会被 `WithErrorLimit()` 过滤掉，即使 429 解除了也不会被调度
- 需要用户手动刷新或重置错误计数才能恢复

这是一个**设计缺陷**：429 是服务端限流，不是 Feed 格式错误，不应该计入解析错误计数。

#### 13.3.2 对跨 host 公平性的影响

**公平性定义**：各 host 获取的调度配额（Worker 时间）与其拥有的到期 Feed 数量成正比。

**场景一：单个 host 遭遇 429**

假设 `POLLING_LIMIT_PER_HOST=5`，两个 host 各有 10 个到期 Feed：

| 阶段 | hostA（正常） | hostB（429） | 说明 |
|------|-------------|-------------|------|
| 第 1 轮 | 5 个被选中 | 5 个被选中 → 全部 429 | hostB 的 5 个全部失败 |
| 第 2 轮（60min 后） | 另外 5 个被选中 | 0 个被选中（next_check_at 被推远） | hostB 完全让出配额 |
| 第 N 轮（Retry-After 后） | 10 个已全部刷新 | 5 个被选中 | hostB 才恢复 |

**结论**：遭遇 429 的 host 会**主动让出**调度配额给其他 host，从资源分配角度看是"公平的"——因为这个 host 暂时不可用，把配额让给能用的 host 更高效。

**场景二：host 内部分 Feed 遭遇 429**

同一个 host 下，部分 Feed 返回 429，部分正常：

- 由于 `limitPerHost` 是**域名级别的配额**，不是 Feed 级别的
- 429 的 Feed 的 `next_check_at` 被推远，下一轮不会被选中
- 同 host 下其他正常的 Feed 会填补这些配额空位
- 结果是：**host 总配额不变，但内部流量向正常 Feed 倾斜**

这可能导致一个问题：如果某 host 下有 100 个 Feed，其中 95 个被 429，剩下 5 个正常的会每轮都被选中，**相当于该 host 的实际刷新频率提高了**（因为配额没减少但候选池变小了）。

#### 13.3.3 公平性失衡的极端情况

**级联失效风险**：

```
1. hostA 的一个 Feed 返回 429，Retry-After = 2 小时
2. 下一轮，这个 Feed 的 next_check_at 还没到，不被选中
3. 同 host 的另一个 Feed 顶替它的位置被选中
4. 这个也返回 429
5. ...
6. 最终 hostA 下的所有 Feed 都 429 了，parsing_error_count 都超标
7. hostA 完全从调度中消失，配额全部让给其他 host
8. 即使 429 解除了，由于 error_count 超标，这些 Feed 仍然无法恢复
9. 形成"死区"，需要人工干预
```

#### 13.3.4 现有机制对公平性的补偿

虽然没有显式的公平性调度，但有一些机制间接缓解了问题：

1. **`next_check_at` 的随机化效果**：
   - 每个 Feed 的 `next_check_at` 基于其上次刷新时间独立计算
   - 自然形成了错峰，不会所有 Feed 同时到期

2. **`ORDER BY next_check_at ASC`**：
   - 最久未刷新的 Feed 优先被选中
   - 确保每个 Feed 最终都能被刷到，防止完全饥饿

3. **多轮迭代的抹平效应**：
   - 虽然单轮可能有失配，但多轮下来，刷新频率由 `ScheduleNextCheck` 决定
   - 长期来看，每个 Feed 的平均刷新频率趋近于其计算间隔

#### 13.3.5 公平性改进思路

如果需要更强的公平性保证，可以考虑：

| 改进方向 | 具体方案 |
|---------|---------|
| 错误分类 | 将 429 限流错误与解析错误分开计数，限流错误不触发永久禁用 |
| 配额动态调整 | 根据 host 的健康状况动态调整 `limitPerHost`（健康的 host 给更多配额） |
| 轮间补偿 | 如果某 host 上一轮有大量 Feed 被 429 跳过，下一轮适当增加其配额 |
| 指数退避 | 对 429 的 Feed 采用指数退避，而不是固定的 Retry-After |

#### 13.3.6 429 误计入 parsing_error_count 的缺陷补丁方案

**缺陷根因**：429 是服务端限流（可恢复的瞬时故障），但当前代码将其与 Feed 格式错误（不可恢复的永久故障）混为一谈，共用同一个 `parsing_error_count` 计数器。

**推荐补丁方案**：引入 `transient_error_count`（瞬时错误计数）与 `parsing_error_count`（持久错误计数）分开。

##### 方案设计

| 错误类型 | 计数器 | 阈值 | 触发行为 |
|---------|-------|-----|---------|
| 429 限流、5xx 错误、网络超时、DNS 失败 | `transient_error_count` | 可配置（如 10 次） | 达到阈值后拉长间隔，但不永久禁用 |
| 解析失败、格式错误、4xx 错误 | `parsing_error_count` | 3（当前行为） | 达到阈值后永久禁用（从调度中排除） |

##### 代码修改点清单

**1. 数据模型层（`internal/model/feed.go`）**

```go
type Feed struct {
    // ... 现有字段 ...
    ParsingErrorCount   int    `json:"parsing_error_count"`
    ParsingErrorMsg     string `json:"parsing_error_message"`
+   TransientErrorCount int    `json:"transient_error_count"`  // 新增
+   TransientErrorMsg   string `json:"transient_error_message"` // 新增
}

// 新增：瞬时错误计数
func (f *Feed) WithTransientErrorMessage(message string) {
    f.TransientErrorCount++
    f.TransientErrorMsg = message
}

// 修改：重置所有错误计数器
func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0
    f.ParsingErrorMsg = ""
+   f.TransientErrorCount = 0
+   f.TransientErrorMsg = ""
}
```

**2. 数据库层（`internal/database/migrations.go`）**

```go
// 新增迁移脚本
{
    id: 100,  // 下一个可用的迁移 ID
    migrate: func(db *sql.DB) error {
        _, err := db.Exec(`
            ALTER TABLE feeds
            ADD COLUMN transient_error_count int default 0,
            ADD COLUMN transient_error_msg text default ''
        `)
        return err
    },
},
```

**3. 调度筛选层（`internal/storage/batch.go`）**

```go
// 修改 WithErrorLimit，同时考虑两种错误
func (b *batchBuilder) WithErrorLimit(parsingLimit, transientLimit int) *batchBuilder {
    if parsingLimit > 0 {
        b.conditions = append(b.conditions, "parsing_error_count < $"+strconv.Itoa(len(b.args)+1))
        b.args = append(b.args, parsingLimit)
    }
    if transientLimit > 0 {
        b.conditions = append(b.conditions, "transient_error_count < $"+strconv.Itoa(len(b.args)+1))
        b.args = append(b.args, transientLimit)
    }
    return b
}
```

**4. 刷新处理层（`internal/reader/handler/handler.go`）** — 核心修改点

```go
// 在 handler.go:245 处，检测到 429 时提前返回
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
    // 使用瞬时错误计数，而不是解析错误计数
    originalFeed.WithTransientErrorMessage(
        locale.NewLocalizedErrorWrapper(
            fmt.Errorf("rate limited by server"),
            "error.http_too_many_requests",
        ).Translate(user.Language),
    )
    store.UpdateFeedError(originalFeed)  // 需要 UpdateFeedError 也处理 transient 字段
    return nil  // 提前返回，不走后续解析流程
}
```

**5. 错误处理函数（`internal/reader/handler/handler.go:30`）**

```go
func getTranslatedLocalizedError(store *storage.Storage, userID int64, originalFeed *model.Feed, localizedError *locale.LocalizedErrorWrapper) *locale.LocalizedErrorWrapper {
    user, _ := store.UserByID(userID)
    message := localizedError.Translate(user.Language)

    // 根据错误类型选择不同的计数器
    switch localizedError.ErrorKey {
    case "error.http_too_many_requests",
         "error.network_operation",
         "error.network_timeout",
         "error.http_client_error":
        originalFeed.WithTransientErrorMessage(message)  // 瞬时错误
    default:
        originalFeed.WithTranslatedErrorMessage(message) // 持久错误
    }

    store.UpdateFeedError(originalFeed)
    return localizedError
}
```

**6. UpdateFeedError（`internal/storage/feed.go:427`）**

```go
func (s *Storage) UpdateFeedError(feed *model.Feed) (err error) {
    query := `
        UPDATE feeds SET
            parsing_error_msg=$1,
            parsing_error_count=$2,
+           transient_error_msg=$3,
+           transient_error_count=$4,
            checked_at=$5,
            next_check_at=$6
        WHERE id=$7 AND user_id=$8
    `
    _, err = s.db.Exec(query,
        feed.ParsingErrorMsg,
        feed.ParsingErrorCount,
+       feed.TransientErrorMsg,
+       feed.TransientErrorCount,
        time.Now(),
        feed.NextCheckAt,
        feed.ID,
        feed.UserID,
    )
    return err
}
```

**7. 配置项（`internal/config/options.go`）**

```go
"POLLING_TRANSIENT_ERROR_LIMIT": {
    parsedIntValue: 10,  // 默认 10 次
    rawValue:       "10",
    valueType:      intType,
},
```

##### 回滚方案

如果补丁引入问题，可以快速回滚：
- 将 `POLLING_TRANSIENT_ERROR_LIMIT` 设为 0，瞬时错误不做限制
- 或在 `WithErrorLimit` 中不传入 transientLimit 参数，退化为原有行为

---

## 14. 10K subscribers 规模下的调度延迟与容量推算

### 14.1 基准假设

基于 Miniflux 单实例典型部署和公开的性能数据，我们假设：

| 参数 | 典型值 | 说明 |
|------|-------|------|
| Feed 总数 | 10,000 | 10K subscribers 规模 |
| 活跃用户 | 1,000 | 10% 的订阅付费用户 |
| 每用户平均订阅数 | 50 | 10K Feed / 200 活跃用户 |
| 单 Feed 平均刷新耗时 | 3 秒 | 含 DNS、TCP、HTTP 请求、解析、入库 |
| Worker 并发数 | 16 | 默认 `WORKER_POOL_SIZE=16` |
| 调度频率 | 60 分钟 | 默认 `POLLING_FREQUENCY=60` |
| 304 命中率 | 40% | 源站支持条件请求的比例 |
| 429/错误率 | 5% | 正常网络环境下的失败率 |
| 单 Feed 平均新条目数 | 0.2 | 每次刷新平均 0.2 条新内容 |

### 14.2 理论容量计算

**单轮调度总耗时**：
```
并行耗时 = (Feed 总数 / Worker 数) × 单 Feed 平均耗时
         = (10,000 / 16) × 3s
         = 625 × 3s
         = 1,875 秒 ≈ 31 分钟
```

**关键结论**：
- 10K Feed 规模下，单轮调度需要约 **31 分钟** 完成
- 调度频率为 60 分钟，因此单实例**完全可以支撑** 10K Feed
- 预留约 29 分钟的空闲时间（~48% 利用率），用于应对峰值和突发流量

**考虑 304 命中率后的实际耗时**：
```
304 快速路径耗时：~0.5 秒（无需解析和入库）
200 正常路径耗时：~4 秒（完整流程）

加权平均耗时 = 40% × 0.5s + 60% × 4s
             = 0.2s + 2.4s
             = 2.6 秒/Feed

实际总耗时 = 625 × 2.6s = 1,625 秒 ≈ 27 分钟
```

考虑 304 后，实际耗时降低到约 **27 分钟**，利用率约 45%。

### 14.3 调度延迟分析

**调度延迟**指的是 Feed 到期到实际被刷新的等待时间。

**最坏情况延迟**：
```
如果所有 Feed 同时到期，最后一个 Feed 需要等待：
(Feed 总数 / Worker 数 - 1) × 单 Feed 耗时
= 624 × 3s = 1,872 秒 ≈ 31 分钟
```

**平均延迟**（基于 `ORDER BY next_check_at ASC`）：
```
由于 Feed 的 next_check_at 均匀分布，平均等待时间约为单轮耗时的 50%
= 27 分钟 × 50% ≈ 13.5 分钟
```

**延迟与调度频率的关系**：
| 调度频率 | 理论最小平均延迟 | 10K Feed 实际平均延迟 |
|---------|-----------------|----------------------|
| 15 分钟 | ~3.75 分钟 | ~13.5 分钟（受限于处理能力） |
| 30 分钟 | ~7.5 分钟 | ~13.5 分钟 |
| 60 分钟 | ~15 分钟 | ~13.5 分钟（处理能力足够） |
| 120 分钟 | ~30 分钟 | ~27 分钟 |

**重要发现**：
- 当调度频率 ≤ 60 分钟时，10K Feed 规模下**平均延迟由处理能力决定**，不是调度频率
- 继续缩短调度频率（如从 60 分钟降到 15 分钟）不会显著降低延迟
- 要降低延迟，需要增加 Worker 并发数或优化单 Feed 处理耗时

### 14.4 资源需求估算

**CPU 需求**：
- 16 个 Worker 满载时约占用 4-6 个 CPU 核心（解析和哈希计算是 CPU 密集型）
- 推荐配置：8 核 CPU

**内存需求**：
- 每个 Worker 处理 Feed 时约占用 10-20MB（主要是 HTTP 响应体和解析树）
- 16 个 Worker 峰值约 320MB
- 加上数据库缓存和系统开销，推荐配置：4GB 内存

**网络带宽**：
- 假设平均每个 Feed 响应体 50KB
- 10K Feed × 60KB = ~600MB/轮 = ~0.2Mbps 持续流量
- 峰值（刷新开始时）可能达到 16 × 50KB/s = ~800KB/s = ~6.4Mbps
- 推荐带宽：10Mbps 上行

**数据库压力**：
- 每轮调度：10,000 次 Feed 更新 + 平均 2,000 条新条目入库
- ~每秒 10-15 次数据库写入
- PostgreSQL 单实例完全可以承受

### 14.5 横向扩展容量模型

| Feed 规模 | 单轮耗时 | 推荐 Worker 数 | 推荐 CPU | 推荐内存 |
|----------|---------|--------------|---------|---------|
| 1K | ~3 分钟 | 8 | 4 核 | 2GB |
| 5K | ~14 分钟 | 16 | 8 核 | 4GB |
| **10K** | **~27 分钟** | **16** | **8 核** | **4GB** |
| 20K | ~54 分钟 | 24 | 16 核 | 8GB |
| 50K | 单实例瓶颈 | 需要多实例分片 | - | - |

**10K 是单实例的舒适区上限**：
- 超过 20K Feed，单轮耗时将接近或超过 60 分钟的调度间隔
- 超过 50K Feed，需要采用分片架构（按用户/Feed ID 哈希分片到不同实例）

### 14.6 运营预算参考

| 项目 | 单实例 10K 规模月成本（AWS） |
|------|---------------------------|
| EC2（c5.2xlarge，8 核 16GB） | ~$280 |
| RDS（db.t3.large，2 核 8GB） | ~$120 |
| EBS 存储（100GB SSD） | ~$10 |
| 数据传输（100GB/月） | ~$10 |
| **合计** | **~$420/月** |

**自助式部署（裸金属/私有云）**：
- 硬件成本：一次性投入 ~$2,000（主流服务器）
- 电费+带宽：~$50/月
- 折旧期 3 年：约 **$105/月**

---

## 15. Feed TTL 与本地 next_check_at 的优先级规则

### 15.1 TTL 的来源与提取

**TTL（Time To Live）** 是 RSS 2.0 规范中定义的可选字段，指示 Feed 源希望订阅者缓存多长时间。

**提取位置**（`internal/reader/rss/adapter.go:59`）：

```go
// Get TTL if defined.
if r.rss.Channel.TTL != "" {
    if ttl, err := strconv.Atoi(r.rss.Channel.TTL); err == nil {
        feed.TTL = time.Duration(ttl) * time.Minute
    }
}
```

**TTL 只在 RSS 中存在**：
- Atom 1.0 和 JSON Feed 规范中**没有 TTL 字段**
- 但可以通过 HTTP 头 `Cache-Control` / `Expires` 达到类似效果
- 解析失败时 `TTL` 为 0，表示未设置

### 15.2 优先级决策链

**核心公式**在 `ScheduleNextCheck`（`model/feed.go:137`）：

```go
interval = max(interval, refreshDelay)
```

其中 `refreshDelay` 是多源取最大值（`handler.go:300`）：

```go
refreshDelay := max(feedTTLValue, cacheControlMaxAgeValue, expiresValue)
```

**完整优先级决策链**：

```
1. 先根据调度策略计算本地间隔 interval：
   - round_robin: interval = SchedulerRoundRobinMinInterval()  默认 60min
   - entry_frequency: interval = (7*24h) / (weeklyCount * factor) （限幅在 5min~24h）

2. 取 refreshDelay = max(Feed TTL, Cache-Control: max-age, Expires - now, Retry-After)
   - 4 个来源都可能为 0（未设置）
   - max() 自动忽略 0 值

3. 取 interval = max(interval, refreshDelay)
   - ↓ 谁更大谁说了算

4. 最后根据策略限幅：
   - round_robin: interval = min(interval, 24h)
   - entry_frequency: interval = min(interval, 24h)
```

### 15.3 谁说了算？场景分析

| 场景 | 本地计算间隔 | Feed TTL | Cache-Control | 最终间隔 | 谁主导 |
|------|-------------|----------|--------------|---------|-------|
| 默认配置，无任何缓存提示 | 60 min | 0 | 0 | 60 min | **本地配置** |
| Feed 设置了短 TTL（15 min） | 60 min | 15 min | 0 | 60 min | **本地配置**（更大） |
| Feed 设置了长 TTL（120 min） | 60 min | 120 min | 0 | 120 min | **Feed TTL**（更大） |
| HTTP 长缓存（max-age=3h） | 60 min | 0 | 180 min | 180 min | **HTTP 头**（更大） |
| entry_frequency 高频源 | 10 min | 0 | 0 | 10 min | **本地自适应** |
| entry_frequency + 长 TTL | 10 min | 60 min | 0 | 60 min | **Feed TTL** |
| 429 Retry-After=2h | 60 min | 0 | 0 | 120 min | **服务器指令** |

**关键结论**：
- **"谁更长谁说了算"** —— `max()` 策略确保永远不会比任何一方建议的更频繁
- 本地间隔是**下限保障**，确保不会因为 TTL 为 0 或过短而过载
- Feed TTL / HTTP 缓存头是**上限建议**，源站可以要求更长的刷新间隔
- 429 Retry-After 是**强制指令**，优先级最高（因为通常是最大的）

### 15.4 边界情况处理

**TTL 格式不合法**（`rss/adapter.go:61`）：
- `<ttl>invalid</ttl>` → 解析失败 → TTL = 0 → 被 `max()` 忽略
- 不会导致错误，静默降级

**TTL 为负值或极大值**：
- TTL 是 `strconv.Atoi` 解析的整数，负值会被转为负的 `time.Duration`
- `max()` 会忽略负值，仍使用本地间隔
- 极大值（如 `<ttl>99999999</ttl>`）会被最后的 `min(interval, 24h)` 限幅

**同时设置 TTL 和 Cache-Control**：
- 取两者中的较大值
- 这符合"尊重所有缓存提示中的最保守者"的原则

**成功后清零 TTL**：
- `updatedFeed.TTL` 是每次从 Feed 内容中重新解析的
- 如果某次刷新时 TTL 从 120min 变为 0，会立即生效，下一轮使用本地间隔

### 15.5 与 SkipHours / SkipDays 的关系

RSS 2.0 还定义了 `<skipHours>` 和 `<skipDays>` 字段，提示聚合器在某些时段不要拉取。

**Miniflux 的处理**：
- 解析了这两个字段（`rss.go:79, 83`），但**完全没有在调度逻辑中使用**
- 没有代码在 `ScheduleNextCheck` 或 `FetchJobs` 中检查 SkipHours/SkipDays
- 属于"解析但不处理"的字段

---

## 16. 多实例 HA 部署下的 Feed 去重机制

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

### 14.4 SCHEDULER_SERVICE 的默认值与配置

**默认值**：`SCHEDULER_SERVICE` 默认为 **启用**（`true`）。

配置项实际通过 `DISABLE_SCHEDULER_SERVICE` 反义控制（`config/options.go:217`）：

```go
"DISABLE_SCHEDULER_SERVICE": {
    parsedBoolValue: false,  // 默认不禁用 → 调度器默认开启
    rawValue:        "0",
    valueType:       boolType,
}

func (c *configOptions) HasSchedulerService() bool {
    return !c.options["DISABLE_SCHEDULER_SERVICE"].parsedBoolValue
}
```

| 配置 | 值 | 调度器状态 |
|------|-----|-----------|
| `DISABLE_SCHEDULER_SERVICE` | `0`（默认） | ✅ 启用 |
| `DISABLE_SCHEDULER_SERVICE` | `1` / `true` | ❌ 禁用 |

**启动判断逻辑**（`daemon.go:32`）：

```go
if config.Opts.HasSchedulerService() && !config.Opts.HasMaintenanceMode() {
    runScheduler(store, pool)  // 两个条件都满足才启动调度器
}
```

**维护模式的影响**：
- 即使 `DISABLE_SCHEDULER_SERVICE=0`，如果开启了维护模式（`MAINTENANCE_MODE=true`），调度器也不会启动
- 维护模式常用于数据库迁移、版本升级等场景

### 14.5 运营如何识别调度仅有一个节点

Miniflux v2 **没有内建**的 leader election、节点注册或调度协调机制。运营需要通过外部手段确保只有一个实例运行调度器。

#### 可观测性手段

1. **Prometheus Metrics**（`internal/metric/metric.go:24`）：
   - `miniflux_background_feed_refresh_duration_seconds` — 后台刷新耗时直方图
   - 可以通过这个指标判断哪些实例在执行调度任务
   - 如果多个实例都有这个指标输出，说明多个实例都在运行调度器

2. **日志关键字**：
   - 调度器启动日志：`Starting background scheduler...`（`scheduler.go:16`）
   - 批次创建日志：`Created a batch of feeds`（`batch.go:129`）
   - 通过日志聚合系统（如 ELK、Loki）可以统计有多少实例输出了这些日志

3. **数据库间接推断**：
   - 查询 `feeds` 表中 `checked_at` 的更新频率
   - 如果同一 Feed 在短时间内被多次更新（间隔远小于 `POLLING_FREQUENCY`），可能说明多实例都在调度

#### 部署层面的保障方案

**方案一：环境变量区分角色（推荐）**

```yaml
# 实例 A — 调度角色
env:
  - name: DISABLE_SCHEDULER_SERVICE
    value: "0"

# 实例 B、C — Web 角色
env:
  - name: DISABLE_SCHEDULER_SERVICE
    value: "1"
```

**方案二：Kubernetes + 单副本 StatefulSet**

- 调度器单独部署为 1 副本的 StatefulSet
- Web 服务部署为多副本的 Deployment
- 通过 Service 暴露 Web 端口，调度器不对外暴露

**方案三：数据库行锁（需二次开发）**

修改 `FetchJobs()` 使用 `SELECT ... FOR UPDATE SKIP LOCKED`：

```sql
BEGIN;
SELECT id, user_id, feed_url FROM feeds
WHERE ...
ORDER BY next_check_at ASC
LIMIT 100
FOR UPDATE SKIP LOCKED;  -- 跳过被其他事务锁定的行
-- 执行刷新...
COMMIT;
```

但这种方案需要在事务中完成整个刷新流程，改造量较大。

#### 多实例都开调度器的风险量化

如果不小心在两个实例上都启用了调度器，影响有多大？

| 风险 | 严重程度 | 说明 |
|------|---------|------|
| 重复 HTTP 请求 | ⚠️ 中 | 浪费带宽和源站资源，但有 ETag/304 缓解 |
| 重复解析入库 | ✅ 低 | 有 entry 去重（hash 检查），不会产生重复条目 |
| 数据损坏 | ✅ 低 | 都是幂等操作，不会造成数据不一致 |
| 调度器整体变慢 | ⚠️ 中 | 两个实例同时查询数据库，增加数据库负载 |

**结论**：多实例同时调度不会造成严重故障，主要是资源浪费。但在大规模部署（Feed 数量 > 10,000）时，建议严格控制只有一个调度器实例。

### 14.6 HA 部署的实际建议

**官方推荐**：单实例运行调度器，多实例运行 Web 服务。

**可行的 HA 方案**：

1. **主备模式**：
   - 只有一个实例启用调度器（`DISABLE_SCHEDULER_SERVICE=0`）
   - 其他实例只提供 Web UI 和 API（`DISABLE_SCHEDULER_SERVICE=1`）
   - 用外部机制（如 Kubernetes livenessProbe + leader election）实现故障转移

2. **时间偏移**：
   - 多个实例都启用调度器，但设置不同的 `POLLING_FREQUENCY` 偏移
   - 简单但不严谨，仍有竞态可能

3. **外部分布式锁**（需自行实现）：
   - 在 `FetchJobs()` 前获取 Redis / etcd 分布式锁
   - 或修改 SQL 添加 `FOR UPDATE SKIP LOCKED`

### 14.7 为什么不启用 `SKIP LOCKED`？

从代码来看，`ArchiveEntries` 已经在使用 `FOR UPDATE SKIP LOCKED`，说明开发团队了解这个特性。`FetchJobs` 不使用可能的原因：

1. **事务边界问题**：`FOR UPDATE` 需要在事务中使用，但 `FetchJobs` 是只读查询，没有开启事务
2. **性能权衡**：加锁会增加数据库开销，对于大多数单实例部署是不必要的
3. **竞态后果轻微**：如前所述，并发刷新只会浪费 HTTP 请求，不会造成数据损坏

---

## 17. 关键设计要点总结（完整版）

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
16. **调度器默认启用**：`DISABLE_SCHEDULER_SERVICE` 默认 `false`，即调度器默认开启；维护模式下自动禁用
17. **429 的副作用**：429 不仅会推远 `next_check_at`，还会导致 `parsing_error_count` 递增，可能永久禁用该 Feed（设计缺陷）
18. **跨 host 公平性**：遭遇 429 的 host 会主动让出配额，长期看由 `ScheduleNextCheck` 保证各 Feed 的刷新频率
19. **级联失效风险**：同一 host 下大量 Feed 连续 429 可能导致 error_count 全部超标，形成"死区"需人工干预
20. **运营识别手段**：通过 Prometheus metrics、日志关键字、数据库更新频率可以判断是否有多实例同时调度
21. **transient_error_count 补丁方案**：引入瞬时错误与持久错误分开计数，7 处代码修改点可修复 429 误禁用缺陷
22. **10K Feed 容量**：单实例 16 Worker 可在 ~27 分钟内刷完 10K Feed，平均延迟 ~13.5 分钟，月成本约 $420（AWS）
23. **TTL 优先级**：谁更长谁说了算，`max(本地间隔, Feed TTL, Cache-Control, Expires, Retry-After)`，最终限幅 24h
24. **SkipHours/SkipDays 未使用**：RSS 规范定义的时段跳过字段已解析但未在调度中生效
25. **单实例舒适区上限**：10K Feed 是单实例舒适区，超过 20K 需增加 Worker，超过 50K 需多实例分片
