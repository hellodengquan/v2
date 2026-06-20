# Miniflux 订阅源周期性刷新机制分析

## 概述

Miniflux 的订阅源刷新系统由三个核心机制协同工作：**抓取队列维护**、**条件请求处理** 和 **失败退避策略**。这三者共同确保了订阅源刷新的高效性、可靠性和对源站的友好性。

---

## 一、抓取队列维护机制

### 1.1 整体架构

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Scheduler      │────▶│  Batch Builder   │────▶│  Worker Pool    │
│  (周期性触发)   │     │  (数据库查询)    │     │  (并发执行)     │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        │                        │                        │
        ▼                        ▼                        ▼
  按 PollingFrequency     按条件筛选 feeds           Worker 协程消费
  触发一次批次生成        生成 Job 列表              Job 并刷新
```

### 1.2 调度器 (Scheduler)

**文件**: `internal/cli/scheduler.go`

调度器是整个刷新流程的入口，使用 `time.Tick` 实现周期性触发：

- **触发频率**: 由 `POLLING_FREQUENCY` 配置，默认 60 分钟
- **核心函数**: `feedScheduler()` (`scheduler.go:33`)
- **工作方式**: 每个周期从数据库拉取一批待刷新的订阅源，推送到 worker 池

```go
func feedScheduler(store *storage.Storage, pool *worker.Pool, frequency time.Duration, batchSize, errorLimit, limitPerHost int) {
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
            slog.Debug("Feed URLs in this batch", slog.Any("feed_urls", jobs.FeedURLs()))
            pool.Push(jobs)
        }
    }
}
```

### 1.3 批量构建器 (Batch Builder)

**文件**: `internal/storage/batch.go`

批量构建器负责从数据库中筛选出需要刷新的订阅源，生成 Job 列表。

#### 筛选条件

| 条件 | 说明 | 默认值 |
|------|------|--------|
| `WithBatchSize` | 每批次最大数量 | 100 |
| `WithErrorLimit` | 错误次数上限，超过则跳过 | 3 |
| `WithoutDisabledFeeds` | 排除已禁用的订阅源 | - |
| `WithNextCheckExpired` | 只选取 `next_check_at < now()` 的 | - |
| `WithLimitPerHost` | 每个主机名的并发限制 | 0 (不限制) |

#### 排序策略

- 按 `next_check_at ASC` 排序，优先刷新最久未检查的订阅源
- 这确保了公平性，避免某些订阅源被长时间遗忘

#### 单主机限制 (Limit Per Host)

当配置了 `POLLING_LIMIT_PER_HOST` 时，批量构建器会在应用层进行二次过滤：

```go
if b.limitPerHost > 0 {
    feedHostname := urllib.Domain(job.FeedURL)
    if hosts[feedHostname] >= b.limitPerHost {
        nbSkippedFeeds++
        continue
    }
    hosts[feedHostname]++
}
```

**设计意图**: 防止对同一主机的订阅源集中发起大量请求，避免给源站造成压力，也避免被源站限流。

### 1.4 Worker 池 (Worker Pool)

**文件**: `internal/worker/pool.go`, `internal/worker/worker.go`

Worker 池采用 Go channel 实现的生产者-消费者模式：

#### Pool 结构

```go
type Pool struct {
    queue chan model.Job
    wg    sync.WaitGroup
}
```

#### Worker 数量

- 由 `WORKER_POOL_SIZE` 配置，默认 16 个 worker
- 每个 worker 是一个独立的 goroutine

#### 工作流程

1. **主调度协程** 调用 `pool.Push(jobs)` 将任务推入 channel
2. **Worker 协程** 从 channel 中消费 Job，调用 `feedHandler.RefreshFeed()`
3. 每个 Worker 顺序处理任务，直到 channel 被关闭

```go
func (w *worker) Run(c <-chan model.Job, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range c {
        startTime := time.Now()
        localizedError := feedHandler.RefreshFeed(w.store, job.UserID, job.FeedID, false)
        // ... metrics
    }
}
```

#### Job 数据结构

```go
type Job struct {
    UserID  int64
    FeedID  int64
    FeedURL string
}
```

---

## 二、条件请求处理机制

### 2.1 概述

条件请求（Conditional Requests）是 HTTP 协议中用于缓存验证的机制。Miniflux 利用 `ETag` 和 `Last-Modified` 头实现条件请求，避免重复下载未变更的内容，节省带宽和处理时间。

### 2.2 请求构建器 (Request Builder)

**文件**: `internal/reader/fetcher/request_builder.go`

#### 设置条件请求头

```go
func (r *RequestBuilder) WithETag(etag string) *RequestBuilder {
    if etag != "" {
        r.headers.Set("If-None-Match", etag)
    }
    return r
}

func (r *RequestBuilder) WithLastModified(lastModified string) *RequestBuilder {
    if lastModified != "" {
        r.headers.Set("If-Modified-Since", lastModified)
    }
    return r
}
```

#### 调用时机

在 `RefreshFeed` 函数中 (`handler.go:235-240`)，如果不忽略 HTTP 缓存，则添加条件请求头：

```go
ignoreHTTPCache := originalFeed.IgnoreHTTPCache || forceRefresh
if !ignoreHTTPCache {
    requestBuilder = requestBuilder.
        WithETag(originalFeed.EtagHeader).
        WithLastModified(originalFeed.LastModifiedHeader)
}
```

### 2.3 响应处理器 (Response Handler)

**文件**: `internal/reader/fetcher/response_handler.go`

#### 是否修改的判断逻辑

```go
func (r *ResponseHandler) IsModified(lastEtagValue, lastModifiedValue string) bool {
    // 304 Not Modified 状态码直接判定为未修改
    if r.httpResponse.StatusCode == http.StatusNotModified {
        return false
    }

    // 优先比较 ETag
    if r.ETag() != "" {
        return r.ETag() != lastEtagValue
    }

    // 其次比较 Last-Modified
    if r.LastModified() != "" {
        return r.LastModified() != lastModifiedValue
    }

    // 没有缓存头则视为已修改
    return true
}
```

判断优先级：**304 状态码 > ETag > Last-Modified > 默认已修改**

#### 特殊处理：Expires: 0

某些服务器设置 `Expires: 0` 表示不希望缓存。Miniflux 会在这种情况下忽略缓存头：

```go
func (r *ResponseHandler) ETag() string {
    if r.httpResponse.Header.Get("Expires") == "0" {
        return ""
    }
    return r.httpResponse.Header.Get("ETag")
}
```

### 2.4 刷新流程中的条件请求处理

**文件**: `internal/reader/handler/handler.go` 中 `RefreshFeed` 函数

#### 未修改的情况 (304)

当订阅源未修改时：

1. 跳过响应体读取和解析
2. 可能更新 `Last-Modified` 头（即使 ETag 不变，Last-Modified 也可能更新）
3. 重置错误计数器
4. 更新 feed 的 `checked_at` 和 `next_check_at`

```go
if ignoreHTTPCache || responseHandler.IsModified(originalFeed.EtagHeader, originalFeed.LastModifiedHeader) {
    // 内容有修改，解析并更新条目
    // ...
} else {
    slog.Debug("Feed not modified", ...)
    
    // 根据 RFC9111，Last-Modified 可能更新，需同步更新
    if responseHandler.LastModified() != "" {
        originalFeed.LastModifiedHeader = responseHandler.LastModified()
    }
}

originalFeed.ResetErrorCounter()
```

#### 已修改的情况 (200)

当订阅源有修改时：

1. 读取响应体
2. 解析 Feed
3. 处理新条目
4. 更新 `ETag` 和 `Last-Modified` 头
5. 重置错误计数器
6. 根据缓存头和 TTL 计算下次检查时间

```go
originalFeed.EtagHeader = responseHandler.ETag()
originalFeed.LastModifiedHeader = responseHandler.LastModified()
```

### 2.5 缓存相关的刷新延迟计算

除了条件请求，Miniflux 还会根据响应头中的缓存信息来调整下次刷新时间：

```go
feedTTLValue := updatedFeed.TTL                    // RSS/Atom 的 TTL 字段
cacheControlMaxAgeValue := responseHandler.CacheControlMaxAge()  // Cache-Control: max-age
expiresValue := responseHandler.Expires()          // Expires 头
refreshDelay := max(feedTTLValue, cacheControlMaxAgeValue, expiresValue)

calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, refreshDelay)
```

这确保了 Miniflux 尊重源站的缓存策略，不会过于频繁地刷新。

---

## 三、失败退避机制

### 3.1 概述

失败退避（Backoff）机制确保当订阅源持续失败时，不会过于频繁地重试，减少对源站的压力，也节省自身资源。

Miniflux 的失败退避采用 **错误计数 + 调度间隔调整** 的组合策略。

### 3.2 错误计数系统

**文件**: `internal/model/feed.go`

#### 相关字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `ParsingErrorCount` | int | 连续解析错误次数 |
| `ParsingErrorMsg` | string | 最后一次错误消息 |

#### 错误累加

```go
func (f *Feed) WithTranslatedErrorMessage(message string) {
    f.ParsingErrorCount++
    f.ParsingErrorMsg = message
}
```

#### 错误重置

```go
func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0
    f.ParsingErrorMsg = ""
}
```

**重置时机**: 每当订阅源成功刷新（无论内容是否修改），错误计数器都会被重置。

### 3.3 错误阈值过滤

**文件**: `internal/storage/batch.go`

批量构建器在生成批次时，会过滤掉错误次数超过阈值的订阅源：

```go
func (b *batchBuilder) WithErrorLimit(limit int) *batchBuilder {
    if limit > 0 {
        b.conditions = append(b.conditions, "parsing_error_count < $"+strconv.Itoa(len(b.args)+1))
        b.args = append(b.args, limit)
    }
    return b
}
```

- 配置项: `POLLING_PARSING_ERROR_LIMIT`，默认 3
- 效果: 错误次数 >= 3 的订阅源不会被加入刷新批次

**注意**: 这是一个"软"退避——订阅源只是暂时不被调度，而不是永久禁用。当用户手动刷新或错误被重置时，订阅源会重新进入调度队列。

### 3.4 调度间隔策略

**文件**: `internal/model/feed.go` 中 `ScheduleNextCheck` 方法

Miniflux 支持两种调度算法，通过 `POLLING_SCHEDULER` 配置选择。

#### 1. 轮询调度 (Round Robin) - 默认

```go
if config.Opts.PollingScheduler() == SchedulerRoundRobin {
    interval = config.Opts.SchedulerRoundRobinMinInterval()  // 默认 60 分钟
}
// ...
interval = min(interval, config.Opts.SchedulerRoundRobinMaxInterval())  // 默认 1440 分钟
```

- **最小间隔**: `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL`，默认 60 分钟
- **最大间隔**: `SCHEDULER_ROUND_ROBIN_MAX_INTERVAL`，默认 1440 分钟 (24 小时)

特点: 所有订阅源使用相同的调度间隔，简单公平。

#### 2. 条目频率调度 (Entry Frequency)

```go
if config.Opts.PollingScheduler() == SchedulerEntryFrequency {
    if weeklyCount <= 0 {
        interval = config.Opts.SchedulerEntryFrequencyMaxInterval()
    } else {
        interval = (7 * 24 * time.Hour) / time.Duration(weeklyCount*config.Opts.SchedulerEntryFrequencyFactor())
        interval = min(interval, config.Opts.SchedulerEntryFrequencyMaxInterval())
        interval = max(interval, config.Opts.SchedulerEntryFrequencyMinInterval())
    }
}
```

- **计算方式**: 根据每周条目数动态计算间隔
- **频率因子**: `SCHEDULER_ENTRY_FREQUENCY_FACTOR`，默认 1
- **最小间隔**: `SCHEDULER_ENTRY_FREQUENCY_MIN_INTERVAL`，默认 5 分钟
- **最大间隔**: `SCHEDULER_ENTRY_FREQUENCY_MAX_INTERVAL`，默认 24 小时

特点: 更新频繁的订阅源被更频繁地检查，更新少的订阅源检查频率低，智能分配资源。

### 3.5 限流退避 (Rate Limited)

**文件**: `internal/reader/handler/handler.go`

当收到 429 Too Many Requests 响应时，Miniflux 会根据 `Retry-After` 头调整下次检查时间：

```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
    
    slog.Warn("Feed is rate limited",
        slog.Int("retry_delay_in_seconds", int(retryDelay.Seconds())),
        ...
    )
}
```

#### Retry-After 解析

```go
func (r *ResponseHandler) ParseRetryDelay() time.Duration {
    retryAfterHeaderValue := r.httpResponse.Header.Get("Retry-After")
    if retryAfterHeaderValue != "" {
        // 尝试解析为秒数
        if seconds, err := strconv.Atoi(retryAfterHeaderValue); err == nil {
            return time.Duration(seconds) * time.Second
        }
        // 尝试解析为 HTTP 日期
        if t, err := time.Parse(time.RFC1123, retryAfterHeaderValue); err == nil {
            return time.Until(t).Truncate(time.Second)
        }
    }
    return 0
}
```

支持两种格式：
1. 整数秒数 (e.g., `Retry-After: 120`)
2. HTTP 日期格式 (e.g., `Retry-After: Fri, 31 Dec 2023 23:59:59 GMT`)

### 3.6 刷新延迟的综合计算

`ScheduleNextCheck` 方法接收一个 `refreshDelay` 参数，该参数综合了多种来源的延迟建议：

```go
func (f *Feed) ScheduleNextCheck(weeklyCount int, refreshDelay time.Duration) time.Duration {
    // 1. 根据调度策略计算基础间隔
    interval := config.Opts.SchedulerRoundRobinMinInterval()
    // ... (entry frequency 计算)
    
    // 2. 使用 RSS TTL、Cache-Control、Expires、Retry-After 中的最大值
    interval = max(interval, refreshDelay)
    
    // 3. 限制最大值，防止配置错误导致间隔过大
    switch config.Opts.PollingScheduler() {
    case SchedulerRoundRobin:
        interval = min(interval, config.Opts.SchedulerRoundRobinMaxInterval())
    case SchedulerEntryFrequency:
        interval = min(interval, config.Opts.SchedulerEntryFrequencyMaxInterval())
    }
    
    f.NextCheckAt = time.Now().Add(interval)
    return interval
}
```

优先级: **调度策略基础间隔 < refreshDelay (TTL/Cache-Control/Expires/Retry-After) < 最大间隔限制**

---

## 四、三者协同工作流程

### 4.1 完整刷新流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        调度循环 (每 PollingFrequency)               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  BatchBuilder 从数据库筛选 feeds                                   │
│  - next_check_at < now()                                           │
│  - parsing_error_count < error_limit                               │
│  - disabled = false                                                │
│  - 每 host 不超过 limit_per_host                                   │
│  - 按 next_check_at ASC 排序                                       │
│  - 限制 batch_size                                                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌──────────────────┐
                    │  生成 Job 列表   │
                    └──────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Worker Pool 消费 Job                                               │
│  - 每个 Worker 顺序执行 RefreshFeed                                 │
│  - 共 WORKER_POOL_SIZE 个 Worker 并发                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  RefreshFeed 执行流程                                               │
│                                                                     │
│  1. 从数据库读取原始 feed 信息                                     │
│     (包含 EtagHeader, LastModifiedHeader, ParsingErrorCount)       │
│                                                                     │
│  2. 构建 HTTP 请求                                                  │
│     ├─ 如果不忽略缓存                                              │
│     │   ├─ 添加 If-None-Match (ETag)                               │
│     │   └─ 添加 If-Modified-Since (Last-Modified)                  │
│     └─ 设置超时、代理、User-Agent 等                                │
│                                                                     │
│  3. 发送请求，获取响应                                              │
│                                                                     │
│  4. 检查是否限流 (429)                                              │
│     ├─ 是: 解析 Retry-After，调整 next_check_at                    │
│     └─ 否: 继续                                                     │
│                                                                     │
│  5. 检查是否有错误                                                  │
│     ├─ 是:                                                          │
│     │   ├─ ParsingErrorCount++                                      │
│     │   ├─ 更新错误信息                                             │
│     │   ├─ 调用 UpdateFeedError 保存                               │
│     │   └─ 返回错误 (退出)                                          │
│     └─ 否: 继续                                                     │
│                                                                     │
│  6. 检查内容是否修改                                                │
│     ├─ 未修改 (304 或 ETag/Last-Modified 相同):                    │
│     │   ├─ 可能更新 Last-Modified                                   │
│     │   └─ 跳转到步骤 8                                             │
│     └─ 已修改:                                                      │
│         ├─ 读取响应体                                               │
│         ├─ 解析 feed                                                │
│         ├─ 处理新条目                                               │
│         ├─ 更新 ETag 和 Last-Modified                               │
│         └─ 继续步骤 7                                               │
│                                                                     │
│  7. 计算刷新延迟                                                    │
│     - max(RSS TTL, Cache-Control max-age, Expires)                 │
│                                                                     │
│  8. 重置错误计数器                                                  │
│                                                                     │
│  9. 调度下次检查                                                    │
│     - ScheduleNextCheck(weeklyCount, refreshDelay)                  │
│     - 考虑调度策略 (round_robin / entry_frequency)                  │
│     - 考虑 refreshDelay                                             │
│     - 限制在 min/max 范围内                                        │
│                                                                     │
│  10. 保存 feed 更新                                                 │
│      - UpdateFeed (成功) 或 UpdateFeedError (失败)                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键协同点

#### 协同点 1: 错误计数 × 队列筛选

**机制**: 失败退避通过错误计数影响队列筛选

- 当 feed 刷新失败时，`ParsingErrorCount` 递增
- BatchBuilder 的 `WithErrorLimit` 条件过滤掉错误过多的 feed
- 当 feed 刷新成功时，`ParsingErrorCount` 重置为 0，重新进入调度

**效果**: 形成一个负反馈循环 —— 失败越多，越不被调度；成功后恢复正常调度。

#### 协同点 2: 条件请求 × 刷新间隔

**机制**: 条件请求的响应头影响下次刷新时间

- 成功的条件请求返回 `Cache-Control`、`Expires` 等缓存头
- 这些头被解析为 `refreshDelay`
- `ScheduleNextCheck` 将 `refreshDelay` 纳入下次检查时间的计算

**效果**: 尊重源站的缓存策略，源站说"可以缓存多久"，Miniflux 就至少等多久再刷新。

#### 协同点 3: 限流响应 × 刷新间隔

**机制**: 429 响应触发特殊的退避

- 当收到 429 Too Many Requests 时
- 解析 `Retry-After` 头获取建议的重试延迟
- 将该延迟作为 `refreshDelay` 传入 `ScheduleNextCheck`

**效果**: 源站明确说"请 X 秒后再来"，Miniflux 就严格遵守。

#### 协同点 4: 调度策略 × Worker 池

**机制**: 调度策略决定产生哪些任务，Worker 池决定如何消费任务

- BatchBuilder 按 `next_check_at ASC` 排序，优先调度最久未检查的
- Worker 池并发执行，`WORKER_POOL_SIZE` 控制并发度
- `limitPer_host` 在批次层面限制单主机并发，避免对同一源站压力过大

**效果**: 既保证了整体的刷新效率，又避免了对单个源站的冲击。

### 4.3 状态转换图

```
                    ┌─────────────────────┐
                    │   初始/正常状态     │
                    │  error_count = 0    │
                    └──────────┬──────────┘
                               │
                               │ 成功刷新
                               │ (重置错误计数)
                               │
┌─────────────────────┐        │        ┌─────────────────────┐
│  被限流 (429)       │        │        │  内容未修改 (304)   │
│  retry-after 延迟   │◄───────┼───────►│  快速返回           │
│  延长 next_check_at │        │        │  重置错误计数       │
└──────────┬──────────┘        │        └──────────┬──────────┘
           │                   │                   │
           │ 等待 Retry-After  │                   │
           │ 后重新进入调度     │                   │
           ▼                   ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│                    调度队列中                            │
│  next_check_at < now() 且 error_count < limit           │
└─────────────────────────────┬───────────────────────────┘
                              │
                              │ Worker 取出执行
                              │
                              ▼
                    ┌──────────────────┐
                    │  执行刷新请求    │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
         ┌────────┐     ┌────────┐     ┌────────┐
         │  成功  │     │  失败  │     │  限流  │
         └───┬────┘     └───┬────┘     └───┬────┘
             │              │              │
             │ error=0      │ error++      │ error++
             │              │              │ + retry-after
             ▼              ▼              ▼
         (回到正常)     error < limit?  (延长调度)
                           /   \
                          是    否
                         /      \
                        ▼        ▼
                  (下次继续)   (被踢出队列)
                               error >= limit
                               不再被调度
                               直到用户手动刷新
```

---

## 五、核心文件索引

| 文件路径 | 模块 | 核心功能 |
|----------|------|----------|
| `internal/cli/scheduler.go` | 调度器 | 周期性触发批次生成 |
| `internal/cli/refresh_feeds.go` | CLI 刷新 | 命令行手动刷新的实现 |
| `internal/worker/pool.go` | Worker 池 | Worker 池管理 |
| `internal/worker/worker.go` | Worker | 单个 Worker 的执行逻辑 |
| `internal/storage/batch.go` | 批量构建 | 从数据库筛选待刷新的 feed |
| `internal/storage/feed.go` | Feed 存储 | Feed 的增删改查 |
| `internal/model/feed.go` | Feed 模型 | Feed 数据结构和调度计算 |
| `internal/model/job.go` | Job 模型 | Job 数据结构 |
| `internal/reader/handler/handler.go` | 刷新处理 | RefreshFeed 主逻辑 |
| `internal/reader/fetcher/request_builder.go` | 请求构建 | HTTP 请求构建，条件请求头设置 |
| `internal/reader/fetcher/response_handler.go` | 响应处理 | HTTP 响应解析，缓存头提取 |
| `internal/config/options.go` | 配置 | 所有配置选项及默认值 |

---

## 六、关键配置项汇总

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `POLLING_FREQUENCY` | 60 分钟 | 调度周期，多久生成一批任务 |
| `BATCH_SIZE` | 100 | 每批次最大 feed 数量 |
| `WORKER_POOL_SIZE` | 16 | Worker 并发数 |
| `POLLING_PARSING_ERROR_LIMIT` | 3 | 错误次数阈值，超过则暂停调度 |
| `POLLING_LIMIT_PER_HOST` | 0 | 每批次同主机名的最大 feed 数 (0 不限制) |
| `POLLING_SCHEDULER` | round_robin | 调度策略: round_robin / entry_frequency |
| `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL` | 60 分钟 | 轮询调度最小间隔 |
| `SCHEDULER_ROUND_ROBIN_MAX_INTERVAL` | 1440 分钟 | 轮询调度最大间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_MIN_INTERVAL` | 5 分钟 | 频率调度最小间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_MAX_INTERVAL` | 1440 分钟 | 频率调度最大间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_FACTOR` | 1 | 频率调度因子 |
| `HTTP_CLIENT_TIMEOUT` | 20 秒 | HTTP 请求超时时间 |

---

## 七、分布式部署下的多 Worker 抓取去重

### 7.1 问题背景

Miniflux 的 feed 数据模型中，同一个 feed URL 在**同一用户**下是唯一的（数据库约束 `unique (user_id, feed_url)`），但**不同用户**可以订阅同一个 feed URL。这意味着当多个用户订阅了同一个源时，调度器可能在同一批次中为不同用户生成多个指向同一源站的 Job。同时，在分布式部署场景下，多个 Miniflux 实例可能同时调度到相同的 feed。

### 7.2 调度层面的去重：next_check_at 的预推进

**核心机制**: `RefreshFeed` 在发出 HTTP 请求**之前**就更新了 `next_check_at`。

```go
// handler.go:220-221 — 在 HTTP 请求之前！
originalFeed.CheckedNow()
originalFeed.ScheduleNextCheck(weeklyEntryCount, time.Duration(0))
```

**工作原理**:

1. 当 Scheduler 周期性触发时，`BatchBuilder` 通过 SQL 条件 `next_check_at < now()` 筛选待刷新 feed
2. Worker 从 Job channel 中取出任务并调用 `RefreshFeed`
3. `RefreshFeed` 在**第一件事**就是把 `next_check_at` 推进到未来（至少 `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL` 之后）
4. 即使另一个调度周期在当前刷新未完成时到来，`next_check_at < now()` 条件也不会再匹配到这个 feed

**但在分布式场景下存在窗口期**: 由于 `next_check_at` 的更新是在内存中修改 `originalFeed` 对象，直到 `UpdateFeed` 或 `UpdateFeedError` 写入数据库才真正生效。在读取 feed 和写回数据库之间有一个时间窗口。如果两个实例几乎同时读取到同一个 feed 的 `next_check_at < now()`，可能会同时开始刷新。

### 7.3 数据库层面：无显式分布式锁

经代码审查，Miniflux **没有**使用 PostgreSQL 的以下分布式协调机制：

- 无 `SELECT ... FOR UPDATE` / `FOR UPDATE SKIP LOCKED` 保护 feed 行的读取
- 无 `pg_advisory_lock` 咨询锁
- 无 `SELECT ... FOR NO KEY UPDATE`

唯一的 `FOR UPDATE SKIP LOCKED` 出现在 `ArchiveEntries` (`entry.go:378`) 中，用于归档旧条目时防止并发归档冲突，而非 feed 刷新去重。

### 7.4 实际的容错设计：幂等性代替互斥

Miniflux 选择了**幂等性**而非**互斥锁**来处理并发刷新问题：

1. **feed 级别**: `UpdateFeed` 写入的是全量字段覆盖，两次并发刷新的结果是"最后写入者赢"（Last Writer Wins），不会导致数据损坏
2. **entry 级别**: 依赖数据库的 `unique (feed_id, hash)` 约束（`migrations.go:88`），即使两个 Worker 同时处理同一 feed，同一 entry 也只会被插入一次

```
entries 表约束:
  unique (feed_id, hash)   ← 防止同一 feed 下重复插入相同 entry

entry_tombstones 表约束:
  primary key (feed_id, hash)  ← 防止已删除的 entry 被重新引入
```

3. **条件请求保护**: 即使两个 Worker 同时请求同一源，第二个请求也会因为携带 `If-None-Match` / `If-Modified-Since` 头而收到 304 响应，节省带宽

### 7.5 单实例内的去重保证

在单个 Miniflux 实例内部，去重是完全可靠的：

- **一个 Scheduler goroutine**: `feedScheduler` 在单个 goroutine 中运行，每轮生成的批次不会重复
- **Worker 从同一 channel 消费**: 每个 Job 只会被一个 Worker 取走
- **`limitPerHost`**: 在批次层面限制同主机的并发请求数

### 7.6 建议与局限

| 场景 | 去重保证 | 说明 |
|------|----------|------|
| 单实例 | ✅ 完全保证 | Scheduler 单 goroutine + channel 单消费 |
| 多实例同 feed | ⚠️ 最终一致 | 可能重复请求，但幂等写入保证数据正确 |
| 多实例同 entry | ✅ 数据库保证 | `unique (feed_id, hash)` 约束防止重复插入 |
| 手动刷新 + 自动调度 | ⚠️ 可能重复 | API/UI 手动刷新不检查 `next_check_at` |

**结论**: Miniflux 的设计哲学是"宁可多请求一次，也不引入复杂的分布式锁"。对于 RSS 刷新这种非关键路径，偶尔的重复请求是可接受的代价。

---

## 八、用户级 vs 全局级 Feed 配置覆盖优先级

### 8.1 配置层级概述

Miniflux 中存在三层配置影响 feed 的刷新行为：

```
┌─────────────────────────────────────────────┐
│  全局级 (Application-Level)                 │
│  - config.Opts 中的环境变量/配置文件参数     │
│  - 适用于所有 feed 的默认值                  │
└──────────────────────┬──────────────────────┘
                       │ 被用户级覆盖
                       ▼
┌─────────────────────────────────────────────┐
│  用户级 (User-Level)                        │
│  - User 模型中的字段                        │
│  - 对该用户的所有 feed 生效                  │
└──────────────────────┬──────────────────────┘
                       │ 被 feed 级覆盖
                       ▼
┌─────────────────────────────────────────────┐
│  Feed 级 (Feed-Level)                       │
│  - Feed 模型/FeedCreationRequest 中的字段   │
│  - 仅对该 feed 生效                         │
└─────────────────────────────────────────────┘
```

### 8.2 各层级的配置项对应关系

#### User-Agent 覆盖

**代码位置**: `request_builder.go:75-81`

```go
func (r *RequestBuilder) WithUserAgent(userAgent string, defaultUserAgent string) *RequestBuilder {
    if userAgent != "" {
        r.headers.Set("User-Agent", userAgent)      // feed 级
    } else {
        r.headers.Set("User-Agent", defaultUserAgent) // 全局级
    }
    return r
}
```

**调用位置**: `handler.go:225`

```go
requestBuilder.WithUserAgent(originalFeed.UserAgent, config.Opts.HTTPClientUserAgent())
```

**优先级**: `feed.UserAgent`（feed 级） > `config.Opts.HTTPClientUserAgent()`（全局级）

逻辑: 如果 feed 上设置了自定义 User-Agent，使用 feed 级的；否则使用全局默认值。

#### 代理 (Proxy) 覆盖

**代码位置**: `request_builder.go:143-157`

```go
func (r *RequestBuilder) ExecuteRequest(requestURL string) (*http.Response, error) {
    var clientProxyURL *url.URL
    switch {
    case r.feedProxyURL != "":
        // 1. Feed 级别的代理（最高优先级）
        clientProxyURL, err = url.Parse(r.feedProxyURL)
    case r.useClientProxy && r.clientProxyURL != nil:
        // 2. 全局级别的代理（通过 FetchViaProxy 启用）
        clientProxyURL = r.clientProxyURL
    case r.proxyRotator != nil && r.proxyRotator.HasProxies():
        // 3. 代理轮换池（最低优先级）
        clientProxyURL = r.proxyRotator.GetNextProxy()
    }
}
```

**优先级**: `feed.ProxyURL`（feed 级） > `config.Opts.HTTPClientProxyURL()`（全局级，需 `feed.FetchViaProxy=true`） > `proxyRotator`（轮换池）

#### 条目过滤规则覆盖

**代码位置**: `processor.go:40-41`, `filter.go:74-87`

```go
// processor.go
blockRules := filter.ParseRules(user.BlockFilterEntryRules, feed.BlockFilterEntryRules)
allowRules := filter.ParseRules(user.KeepFilterEntryRules, feed.KeepFilterEntryRules)
```

```go
// filter.go
func ParseRules(userRules, feedRules string) filterRules {
    rules := make(filterRules, 0)
    // 先解析用户级规则
    for line := range strings.SplitSeq(strings.TrimSpace(userRules), "\n") {
        if valid, filterRule := parseRule(line); valid {
            rules = append(rules, filterRule)
        }
    }
    // 再解析 feed 级规则
    for line := range strings.SplitSeq(strings.TrimSpace(feedRules), "\n") {
        if valid, filterRule := parseRule(line); valid {
            rules = append(rules, filterRule)
        }
    }
    return rules
}
```

**优先级**: 用户级和 feed 级规则**合并**（非覆盖），用户级规则先执行，feed 级规则后追加。

`IsBlockedEntry` 中的判定顺序 (`filter.go:101-127`):

1. **用户级 block filter 规则** (`user.BlockFilterEntryRules` + `feed.BlockFilterEntryRules` 合并)
2. **Feed 级 blocklist 正则** (`feed.BlocklistRules`)
3. **用户级 keep filter 规则** (`user.KeepFilterEntryRules` + `feed.KeepFilterEntryRules` 合并)
4. **Feed 级 keeplist 正则** (`feed.KeeplistRules`)

```
判定流程:
  用户 block filter 规则 → 匹配则阻止
  Feed blocklist 正则    → 匹配则阻止
  用户 keep filter 规则  → 不匹配则阻止
  Feed keeplist 正则     → 不匹配则阻止
  全部通过               → 允许
```

#### HTTP 缓存策略覆盖

```go
// handler.go:235
ignoreHTTPCache := originalFeed.IgnoreHTTPCache || forceRefresh
```

**优先级**: `feed.IgnoreHTTPCache`（feed 级）或 `forceRefresh`（请求级）可以覆盖全局的默认缓存行为。

#### HTTP/2 和 TLS 覆盖

```go
requestBuilder.
    IgnoreTLSErrors(originalFeed.AllowSelfSignedCertificates).
    DisableHTTP2(originalFeed.DisableHTTP2)
```

这些是 **feed 级独占配置**，没有全局级的对应项，只能在每个 feed 上单独设置。

### 8.3 覆盖优先级汇总表

| 配置项 | 全局级 | 用户级 | Feed 级 | 覆盖策略 |
|--------|--------|--------|---------|----------|
| User-Agent | `HTTP_CLIENT_USER_AGENT` | - | `feed.UserAgent` | Feed 级非空则覆盖全局 |
| Proxy URL | `HTTP_CLIENT_PROXY` | - | `feed.ProxyURL` | Feed 级 > 全局(需 FetchViaProxy) > 轮换池 |
| Proxy 轮换池 | `HTTP_CLIENT_PROXIES` | - | - | 无 feed 级覆盖 |
| HTTP 缓存忽略 | - | - | `feed.IgnoreHTTPCache` | Feed 级覆盖默认缓存行为 |
| TLS 证书验证 | - | - | `feed.AllowSelfSignedCertificates` | Feed 级独占 |
| HTTP/2 禁用 | - | - | `feed.DisableHTTP2` | Feed 级独占 |
| Cookie | - | - | `feed.Cookie` | Feed 级独占 |
| 认证信息 | - | - | `feed.Username/Password` | Feed 级独占 |
| Block/Keep 过滤 | - | `user.BlockFilterEntryRules` | `feed.BlockFilterEntryRules` | 合并（非覆盖），用户级先执行 |
| 正则 Blocklist | - | - | `feed.BlocklistRules` | Feed 级独占 |
| 正则 Keeplist | - | - | `feed.KeeplistRules` | Feed 级独占 |
| Crawler (抓取原文) | - | - | `feed.Crawler` | Feed 级独占 |
| 忽略条目更新 | - | - | `feed.IgnoreEntryUpdates` | Feed 级独占 |
| 调度策略 | `POLLING_SCHEDULER` | - | - | 全局级，无 feed 级覆盖 |
| 刷新间隔范围 | `SCHEDULER_*_INTERVAL` | - | - | 全局级，无 feed 级覆盖 |
| 错误限制 | `POLLING_PARSING_ERROR_LIMIT` | - | - | 全局级，无 feed 级覆盖 |
| 强制刷新间隔 | `FORCE_REFRESH_INTERVAL` | - | - | 全局级，会话级别限流 |
| 条目阅读速度 | - | `user.DefaultReadingSpeed` | - | 用户级独占 |

### 8.4 设计特点总结

1. **Feed 级优先**: 几乎所有网络请求相关配置都可以在 feed 级别覆盖，因为不同源站可能需要不同的请求策略
2. **用户级过滤合并**: 过滤规则采用合并而非覆盖策略，用户可以设置全局过滤，feed 可以追加更细粒度的过滤
3. **全局级不可覆盖的**: 调度策略、间隔范围、错误限制等运维级别的配置无法被用户或 feed 级覆盖，确保系统稳定性
4. **无用户级网络配置**: 代理、TLS、HTTP/2 等网络配置没有用户级别的设置，只有全局和 feed 级

---

## 九、Feed 内容增量更新与全量替换的判定逻辑

### 9.1 核心问题

当 Miniflux 从源站获取到 feed 内容后，需要决定对每个 entry 执行**新增**（INSERT）还是**更新**（UPDATE）。这个判定基于 entry 的 **hash** 值。

### 9.2 Entry Hash 的生成

Hash 是 entry 的唯一标识，在 feed 解析阶段生成。不同格式的 feed 有不同的 hash 生成策略：

#### RSS 2.0 (`internal/reader/rss/adapter.go:123-138`)

```go
switch {
case item.GUID.Data != "":
    n := seenGUIDs[item.GUID.Data]
    seenGUIDs[item.GUID.Data] = n + 1
    switch {
    case n == 0:
        entry.Hash = crypto.SHA256(item.GUID.Data)           // 优先: GUID
    case entry.URL != "":
        entry.Hash = crypto.SHA256(item.GUID.Data + "|" + entry.URL)  // GUID 重复: GUID+URL
    default:
        entry.Hash = crypto.SHA256(item.GUID.Data + "|" + strconv.Itoa(n))  // 最后手段: GUID+序号
    }
case entryURL != "":
    entry.Hash = crypto.SHA256(entryURL)                     // 无 GUID: URL
default:
    entry.Hash = crypto.SHA256(entry.Title + entry.Content)  // 无 GUID 无 URL: 标题+内容
}
```

**关键**: RSS 规范要求 `<guid>` 唯一标识 item，但有些 feed 每个 item 使用相同的 GUID。Miniflux 通过 `seenGUIDs` 计数器检测重复 GUID，并用 URL 或序号消歧。

#### Atom 1.0 (`internal/reader/atom/atom_10_adapter.go:149-155`)

```go
for _, value := range []string{atomEntry.ID, atomEntry.Links.originalLink()} {
    if value != "" {
        entry.Hash = crypto.SHA256(value)
        break
    }
}
```

优先级: `atom:entry/id` > `atom:entry/link` (原始链接)

#### JSON Feed (`internal/reader/json/adapter.go:175-181`)

```go
for _, value := range []string{item.ID, item.URL, item.ExternalURL, item.ContentText + item.ContentHTML + item.Summary} {
    value = strings.TrimSpace(value)
    if value != "" {
        entry.Hash = crypto.SHA256(value)
        break
    }
}
```

优先级: `id` > `url` > `external_url` > `content_text + content_html + summary`

#### RDF/RSS 1.0 (`internal/reader/rdf/adapter.go:79`)

```go
entry.Hash = crypto.SHA256(hashValue)  // hashValue 来自 rdf:about 属性
```

### 9.3 增量更新判定流程

**代码位置**: `internal/storage/entry.go:315-360` — `RefreshFeedEntries` 函数

```go
func (s *Storage) RefreshFeedEntries(userID, feedID int64, entries model.Entries, updateExistingEntries bool) (newEntries model.Entries, err error) {
    for _, entry := range entries {
        entry.UserID = userID
        entry.FeedID = feedID

        tx, err := s.db.Begin()

        entryExists, err := s.entryExists(tx, entry)
        // entryExists 查询: SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1

        if entryExists {
            if updateExistingEntries {
                err = s.updateEntry(tx, entry)   // UPDATE: 增量更新
            }
            // 如果 updateExistingEntries == false，直接跳过，不做任何操作
        } else {
            err = s.createEntry(tx, entry)       // INSERT: 新增
            switch {
            case errors.Is(err, ErrEntryTombstoned):
                err = nil                         // 墓碑条目，静默跳过
            case err == nil:
                newEntries = append(newEntries, entry)  // 记录新 entry 用于推送通知
            }
        }

        tx.Commit()
    }
    return newEntries, nil
}
```

#### 判定流程图

```
                         ┌─────────────────┐
                         │  解析 entry      │
                         │  计算 hash       │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │  SELECT FROM entries     │
                    │  WHERE feed_id=? AND     │
                    │        hash=?            │
                    └──────────┬───────────────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
          entryExists=true           entryExists=false
          (entry 已存在)              (entry 不存在)
                 │                           │
                 │                           ▼
                 │                 ┌─────────────────────┐
                 │                 │  INSERT INTO entries │
                 │                 │  检查 entry_tombstones│
                 │                 └──────────┬──────────┘
                 │                            │
                 │               ┌────────────┴────────────┐
                 │               │                         │
                 │               ▼                         ▼
                 │        插入成功                  被 tombstone 阻止
                 │        (新 entry)                (ErrEntryTombstoned)
                 │        加入 newEntries            静默跳过
                 │
                 ▼
        updateExistingEntries?
           /          \
         true        false
          │            │
          ▼            ▼
    updateEntry()   跳过(不做任何操作)
    更新 title,
    content, url,
    author, tags
```

### 9.4 `updateExistingEntries` 标志的判定

**代码位置**: `handler.go:323`

```go
updateExistingEntries := forceRefresh || (!originalFeed.Crawler && !originalFeed.IgnoreEntryUpdates)
```

| forceRefresh | Crawler | IgnoreEntryUpdates | 结果 | 说明 |
|:---:|:---:|:---:|:---:|------|
| true | * | * | **true** | 强制刷新总是更新 |
| false | true | * | **false** | Crawler 只抓新条目的原文 |
| false | false | true | **false** | 显式忽略条目更新 |
| false | false | false | **true** | 默认行为，更新已有条目 |

**设计意图**:
- `Crawler=true` 时，已有条目的内容是通过爬取原始网页获取的，RSS 中的内容可能更旧或不完整，因此不更新
- `IgnoreEntryUpdates=true` 时，用户显式选择不更新已有条目（保留阅读进度等状态）
- `forceRefresh=true` 时，无视以上规则强制更新

### 9.5 updateEntry 更新了哪些字段

**代码位置**: `entry.go:166-210`

```go
func (s *Storage) updateEntry(tx *sql.Tx, entry *model.Entry) error {
    query := `
        UPDATE entries SET
            title=$1,
            url=$2,
            comments_url=$3,
            content=$4,
            author=$5,
            reading_time=$6,
            document_vectors = ...,
            tags=$12
        WHERE user_id=$9 AND feed_id=$10 AND hash=$11
        RETURNING id
    `
}
```

**更新**: title, url, comments_url, content, author, reading_time, tags

**不更新**: `published_at`（发布日期）、`status`（已读/未读状态）、`starred`（收藏）

注释明确说明 (`entry.go:164-165`):

> Note: we do not update the published date because some feeds do not contains any date, it default to time.Now() which could change the order of items on the history page.

### 9.6 Entry Tombstone 机制：防止"僵尸"条目

当条目被归档删除后，Miniflux 使用 `entry_tombstones` 表记录已删除条目的 `(feed_id, hash)` 对。

**代码位置**: `entry.go:81-146`

```go
func (s *Storage) createEntry(tx *sql.Tx, entry *model.Entry) error {
    query := `
        INSERT INTO entries (...)
        SELECT $1, $2, ...
        WHERE NOT EXISTS (
            SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
        )
        RETURNING id, status, created_at, changed_at
    `
    // 如果被 tombstone 阻止，sql.ErrNoRows → ErrEntryTombstoned
}
```

**关键特性**:
1. `INSERT ... WHERE NOT EXISTS` 子查询使 tombstone 检查与插入操作**原子化**，消除了 TOCTOU（Time-of-Check to Time-of-Use）竞态
2. `ArchiveEntries` (`entry.go:362-404`) 在删除旧条目的同时写入 tombstone
3. `IsNewEntry` (`entry.go:278-293`) 同时检查 entries 表和 tombstones 表，确保 Crawler 不会对已删除条目做无用的网页抓取

### 9.7 增量更新 vs 全量替换总结

| 场景 | 行为 | 依据 |
|------|------|------|
| 新 entry (hash 不存在) | INSERT | `entryExists=false` |
| 已有 entry + 默认配置 | UPDATE (title, content 等) | `updateExistingEntries=true` |
| 已有 entry + Crawler=true | 跳过 | `updateExistingEntries=false` |
| 已有 entry + IgnoreEntryUpdates=true | 跳过 | `updateExistingEntries=false` |
| 已有 entry + forceRefresh=true | UPDATE | `updateExistingEntries=true` |
| 已删除 entry (tombstone) | 静默跳过 | `WHERE NOT EXISTS` 原子检查 |
| 条目发布日期 | 永不更新 | 防止排序混乱 |
| 条目阅读状态 | 永不更新 | 保留用户状态 |

---

## 十、User-Agent 与代理设置的完整抓取链路注入

### 10.1 代码注入路径总览

User-Agent 和代理设置的注入发生在抓取链路的**请求构建阶段**，存在**两条独立的调用路径**：

```
┌─────────────────────────────────────────────────────────────────────┐
│  路径 1: RefreshFeed（周期性自动刷新 / 手动刷新）                   │
│  handler.go:223-233 - RefreshFeed 函数中构建请求                    │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  路径 2: CreateFeed（创建新 feed 时预抓取）                         │
│  handler.go:115-125 - CreateFeed 函数中构建请求                     │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌──────────────────────────┐
                    │  fetcher.NewRequestBuilder()
                    │  request_builder.go:49
                    └─────────────┬────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
   WithUserAgent            WithProxy*系列          其他配置（超时、TLS等）
   (User-Agent 头)         (三级代理选择)
```

### 10.2 两条注入路径的代码细节

#### 路径 1: RefreshFeed（自动/手动刷新）

**代码位置**: `handler.go:223-233`

```go
// handler.go:223-233 — RefreshFeed 函数
requestBuilder := fetcher.NewRequestBuilder().
    WithUsernameAndPassword(originalFeed.Username, originalFeed.Password).
    WithUserAgent(originalFeed.UserAgent, config.Opts.HTTPClientUserAgent()).   // ← UA 注入
    WithCookie(originalFeed.Cookie).
    WithTimeout(config.Opts.HTTPClientTimeout()).
    WithProxyRotator(proxyrotator.ProxyRotatorInstance).                           // ← 代理轮换池
    WithCustomFeedProxyURL(originalFeed.ProxyURL).                                 // ← feed 级代理
    WithCustomApplicationProxyURL(config.Opts.HTTPClientProxyURL()).               // ← 全局级代理
    UseCustomApplicationProxyURL(originalFeed.FetchViaProxy).                       // ← 启用全局代理标志
    IgnoreTLSErrors(originalFeed.AllowSelfSignedCertificates).
    DisableHTTP2(originalFeed.DisableHTTP2)
```

#### 路径 2: CreateFeed（创建时预抓取）

**代码位置**: `handler.go:115-125`

```go
// handler.go:115-125 — CreateFeed 函数
requestBuilder := fetcher.NewRequestBuilder().
    WithUsernameAndPassword(feedCreationRequest.Username, feedCreationRequest.Password).
    WithUserAgent(feedCreationRequest.UserAgent, config.Opts.HTTPClientUserAgent()). // ← UA 注入
    WithCookie(feedCreationRequest.Cookie).
    WithTimeout(config.Opts.HTTPClientTimeout()).
    WithProxyRotator(proxyrotator.ProxyRotatorInstance).                           // ← 代理轮换池
    WithCustomFeedProxyURL(feedCreationRequest.ProxyURL).                           // ← feed 级代理
    WithCustomApplicationProxyURL(config.Opts.HTTPClientProxyURL()).               // ← 全局级代理
    UseCustomApplicationProxyURL(feedCreationRequest.FetchViaProxy).               // ← 启用全局代理标志
    IgnoreTLSErrors(feedCreationRequest.AllowSelfSignedCertificates).
    DisableHTTP2(feedCreationRequest.DisableHTTP2)
```

**注意**: 两条路径的**参数来源不同但注入逻辑完全一致**：
- RefreshFeed 从 `originalFeed` 对象读取已保存的配置
- CreateFeed 从 `feedCreationRequest` 读取用户提交的配置

### 10.3 User-Agent 的注入逻辑

**代码位置**: `request_builder.go:75-82`

```go
func (r *RequestBuilder) WithUserAgent(userAgent string, defaultUserAgent string) *RequestBuilder {
    if userAgent != "" {
        r.headers.Set("User-Agent", userAgent)      // feed 级自定义 UA
    } else {
        r.headers.Set("User-Agent", defaultUserAgent) // 全局默认 UA
    }
    return r
}
```

**覆盖优先级**: `feed.UserAgent`（非空） > `config.Opts.HTTPClientUserAgent()`（全局默认）

**全局默认 UA 的构造**: `internal/config/options.go` 中定义为 `"Miniflux/" + version.Version`

### 10.4 代理设置的三级选择逻辑

**代码位置**: `request_builder.go:143-157` — `ExecuteRequest` 方法中实时选择

```go
func (r *RequestBuilder) ExecuteRequest(requestURL string) (*http.Response, error) {
    var clientProxyURL *url.URL

    switch {
    // 优先级 1: Feed 级自定义代理（最高优先级）
    case r.feedProxyURL != "":
        clientProxyURL, err = url.Parse(r.feedProxyURL)
        if err != nil {
            return nil, fmt.Errorf(`fetcher: invalid feed proxy URL %q: %w`, r.feedProxyURL, err)
        }
    // 优先级 2: 全局级应用代理（需 feed.FetchViaProxy=true 启用）
    case r.useClientProxy && r.clientProxyURL != nil:
        clientProxyURL = r.clientProxyURL
    // 优先级 3: 代理轮换池（最低优先级，仅当前面都未设置时使用）
    case r.proxyRotator != nil && r.proxyRotator.HasProxies():
        clientProxyURL = r.proxyRotator.GetNextProxy()
    }
    // ...
    if clientProxyURL != nil {
        transport.Proxy = http.ProxyURL(clientProxyURL)  // 最终注入到 http.Transport
    }
}
```

#### 三级代理的设置方法

| 优先级 | 类型 | 配置注入方法 | 存储字段 |
|:---:|------|-------------|----------|
| 1 (最高) | Feed 级代理 | `WithCustomFeedProxyURL(feed.ProxyURL)` | `feeds.proxy_url` |
| 2 | 全局级代理 | `WithCustomApplicationProxyURL(config.Opts.HTTPClientProxyURL())` + `UseCustomApplicationProxyURL(feed.FetchViaProxy)` | `HTTP_CLIENT_PROXY` 环境变量 + `feeds.fetch_via_proxy` |
| 3 (最低) | 代理轮换池 | `WithProxyRotator(proxyrotator.ProxyRotatorInstance)` | `HTTP_CLIENT_PROXIES` 环境变量（逗号分隔） |

#### 代理轮换池的实现

**代码位置**: `internal/proxyrotator/proxyrotator.go`

```go
type ProxyRotator struct {
    proxies      []*url.URL
    currentIndex int
    mutex        sync.Mutex  // 线程安全，跨 Worker 并发安全
}

func (pr *ProxyRotator) GetNextProxy() *url.URL {
    pr.mutex.Lock()
    proxy := pr.proxies[pr.currentIndex]
    pr.currentIndex = (pr.currentIndex + 1) % len(pr.proxies)  // 轮询算法
    pr.mutex.Unlock()
    return proxy
}
```

- **初始化时机**: 程序启动时在 `main.go` 中初始化 `ProxyRotatorInstance` 单例
- **线程安全**: 通过 `sync.Mutex` 保护，多个 Worker 并发调用时不会冲突
- **轮询算法**: 简单的 `(currentIndex + 1) % len(proxies)`，平均分配请求

### 10.5 特殊网络配置的注入

除了 User-Agent 和代理，还有以下网络相关配置在同一点注入：

```go
requestBuilder.
    WithUsernameAndPassword(...)   // HTTP Basic Auth → Authorization 头 (handler.go:92-96)
    WithCookie(...)                // Cookie → Cookie 头 (handler.go:84-89)
    IgnoreTLSErrors(...)           // TLS 证书跳过 → Transport.TLSClientConfig.InsecureSkipVerify (handler.go:213-224)
    DisableHTTP2(...)              // HTTP/2 禁用 → Transport.ForceAttemptHTTP2=false + TLSNextProto={} (handler.go:226-232)
    WithTimeout(...)               // 请求超时 → http.Client.Timeout (handler.go:118-121)
```

### 10.6 代理注入与私有网络检查的协作

**代码位置**: `request_builder.go:199-211`

当同时启用代理和私有网络拦截时，Miniflux 有一个特殊的旁路设计：

```go
transport.DialContext = directDialer.DialContext  // 直连的 Dialer 有私有网络检查

if !allowPrivateNetworks && proxyDialAddress != "" {
    // 对代理服务器本身的连接绕过私有网络检查
    // 因为代理服务器可能在内网，但我们信任它作为跳点
    transport.DialContext = func(ctx context.Context, network, addr string) (net.Conn, error) {
        if normalizeDialAddress(addr) == proxyDialAddress {
            return proxyDialer.DialContext(ctx, network, addr)  // 连接代理本身，无检查
        }
        return directDialer.DialContext(ctx, network, addr)     // 其他连接有检查
    }
}
```

**设计意图**: 显式配置的代理是可信跳点，允许连接到内网代理服务器，但通过代理访问的目标地址仍受保护（代理端会进行 DNS 解析）。

---

## 十一、HTTP 客户端重试与 Backoff 算法

### 11.1 核心结论：Miniflux 没有内置 HTTP 客户端重试

经过全代码库搜索，确认 **Miniflux 的 HTTP 客户端（`internal/reader/fetcher/request_builder.go`）没有任何内置的重试逻辑**。具体表现为：

- 无 `retryablehttp` 等第三方重试库
- 无自定义的 `http.RoundTripper` 重试包装
- 无循环重试代码（`for` 循环重试）
- 无指数退避（exponential backoff）算法
- 无抖动（jitter）算法

**请求执行是一次性的**: `request_builder.go:283` 中只有一次 `client.Do(req)` 调用，失败即返回错误。

### 11.2 两级 Backoff 策略

虽然没有 HTTP 传输层的重试，但 Miniflux 在**应用层**实现了两层 Backoff 机制：

```
┌─────────────────────────────────────────────────────────────┐
│  层级 1: 单 feed 级错误计数退避                             │
│  - 触发条件: 连续刷新失败                                   │
│  - 实现: ParsingErrorCount 递增 + 阈值过滤                  │
│  - 效果: 错误次数 >= POLLING_PARSING_ERROR_LIMIT 时         │
│          暂不调度（从批次中过滤）                           │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  层级 2: 源站指令退避（Retry-After）                        │
│  - 触发条件: 收到 429 Too Many Requests 响应                │
│  - 实现: 解析 Retry-After 头，调整 next_check_at            │
│  - 效果: 严格遵守源站指定的重试间隔                          │
└─────────────────────────────────────────────────────────────┘
```

### 11.3 层级 1: 错误计数退避的详细参数

**代码位置**: `internal/model/feed.go` + `internal/storage/batch.go`

#### 参数配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `POLLING_PARSING_ERROR_LIMIT` | 3 | 连续错误阈值，超过则从调度批次中过滤 |

#### 退避流程

```
                        ┌─────────────────┐
                        │  刷新 feed 失败  │
                        └────────┬────────┘
                                 │
                                 ▼
                   originalFeed.WithTranslatedErrorMessage(err)
                   ParsingErrorCount++
                                 │
                                 ▼
                        store.UpdateFeedError(feed)
                        保存错误计数到数据库
                                 │
                                 ▼
                   ┌───────────────────────────────┐
                   │  下一轮 BatchBuilder 筛选      │
                   │  parsing_error_count < limit?  │
                   └──────────────┬────────────────┘
                                  │
                ┌─────────────────┴─────────────────┐
                │                                   │
                ▼                                   ▼
        ParsingErrorCount < 3             ParsingErrorCount >= 3
        加入下一批次（继续调度）           不加入批次（暂不调度）
                │                                   │
                │                                   │
                ▼                                   ▼
        下次 POLLING_FREQUENCY               用户手动刷新时恢复
        后再次尝试
```

#### 错误重置条件

```go
// 任何成功刷新都会重置错误计数
originalFeed.ResetErrorCounter()   // handler.go:364
```

重置发生在以下情况：
- 收到 200 OK 且内容解析成功
- 收到 304 Not Modified（内容未变更，也视为成功）

### 11.4 层级 2: Retry-After 退避的详细参数

**代码位置**: `internal/reader/fetcher/response_handler.go:82-96` + `handler.go:245-255`

#### Retry-After 解析算法

```go
func (r *ResponseHandler) ParseRetryDelay() time.Duration {
    retryAfterHeaderValue := r.httpResponse.Header.Get("Retry-After")
    if retryAfterHeaderValue != "" {
        // 分支 1: 整数秒数格式 (e.g., "120")
        if seconds, err := strconv.Atoi(retryAfterHeaderValue); err == nil {
            return time.Duration(seconds) * time.Second
        }
        // 分支 2: HTTP 日期格式 (e.g., "Fri, 31 Dec 2023 23:59:59 GMT")
        if t, err := time.Parse(time.RFC1123, retryAfterHeaderValue); err == nil {
            return time.Until(t).Truncate(time.Second)
        }
    }
    return 0  // 无法解析则不额外退避
}
```

#### 退避应用

```go
// handler.go:245-255
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    // 将 Retry-After 的延迟作为 refreshDelay 传入
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
}
```

在 `ScheduleNextCheck` 中，`refreshDelay` 会覆盖默认的调度间隔：

```go
// model/feed.go:ScheduleNextCheck
interval = max(interval, refreshDelay)  // refreshDelay 是 Retry-After 解析出的值
```

**优先级**: `Retry-After` 退避 > 调度策略基础间隔 > 最小间隔限制

### 11.5 网络错误分类与本地化处理

虽然没有重试，但 Miniflux 对 HTTP 错误进行了精细分类，为不同错误类型提供不同的用户提示。错误分类发生在 `LocalizedError()` 中：

**代码位置**: `response_handler.go:175-229`

```go
func (r *ResponseHandler) LocalizedError() *locale.LocalizedErrorWrapper {
    // 第一类: 客户端/网络层错误 (clientErr != nil)
    if r.clientErr != nil {
        switch {
        case isSSLError(r.clientErr):       // TLS 证书错误
            return locale.NewLocalizedErrorWrapper(err, "error.tls_error", r.clientErr)
        case isNetworkError(r.clientErr):   // 网络操作错误 (DNS、连接等)
            return locale.NewLocalizedErrorWrapper(err, "error.network_operation", r.clientErr)
        case os.IsTimeout(r.clientErr):     // 超时错误
            return locale.NewLocalizedErrorWrapper(err, "error.network_timeout", r.clientErr)
        case errors.Is(r.clientErr, io.EOF): // 空响应
            return locale.NewLocalizedErrorWrapper(err, "error.http_empty_response")
        default:                             // 其他客户端错误
            return locale.NewLocalizedErrorWrapper(err, "error.http_client_error", r.clientErr)
        }
    }

    // 第二类: Cloudflare 特殊检测
    if r.isCloudflareChallenge() {
        return locale.NewLocalizedErrorWrapper(..., "error.http_cloudflare_challenge")
    }

    // 第三类: HTTP 状态码错误
    switch r.httpResponse.StatusCode {
    case http.StatusUnauthorized:      // 401
        return locale.NewLocalizedErrorWrapper(..., "error.http_not_authorized")
    case http.StatusForbidden:         // 403
        return locale.NewLocalizedErrorWrapper(..., "error.http_forbidden")
    case http.StatusTooManyRequests:   // 429 — 会触发 Retry-After 退避
        return locale.NewLocalizedErrorWrapper(..., "error.http_too_many_requests")
    case http.StatusNotFound:          // 404
    case http.StatusGone:              // 410
        return locale.NewLocalizedErrorWrapper(..., "error.http_resource_not_found")
    case http.StatusInternalServerError:  // 500
        return locale.NewLocalizedErrorWrapper(..., "error.http_internal_server_error")
    case http.StatusBadGateway:           // 502
        return locale.NewLocalizedErrorWrapper(..., "error.http_bad_gateway")
    case http.StatusServiceUnavailable:   // 503
        return locale.NewLocalizedErrorWrapper(..., "error.http_service_unavailable")
    case http.StatusGatewayTimeout:       // 504
        return locale.NewLocalizedErrorWrapper(..., "error.http_gateway_timeout")
    }
    // ...
}
```

### 11.6 为何没有传输层重试？

从代码设计看，Miniflux 选择不做 HTTP 传输层重试的原因可能是：

1. **RSS 刷新不是关键路径**: 一次刷新失败不影响整体可用性，等下一轮即可
2. **源站脆弱**: 很多 RSS 源由小服务器或个人博客托管，重试可能加剧源站负担
3. **避免重复条目**: 重试可能导致部分成功的请求产生副作用（虽然幂等性可以避免，但增加复杂度）
4. **应用层退避足够**: 错误计数退避 + Retry-After 退避已能有效处理多数场景

---

## 十二、Entry Hash 计算与重复检测的完整代码路径

### 12.1 完整链路总览

Entry Hash 是 Miniflux 去重体系的核心，从计算到存储经过四个阶段：

```
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 1: Hash 计算（Feed 解析阶段）                                 │
│  - RSS/Atom/JSON/RDF 各自的 adapter 中计算                          │
│  - 针对不同格式选择不同的 hash 源（GUID/ID/URL/Content 等）          │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 2: Processor 预检测（处理阶段）                               │
│  - processor.go:95 调用 IsNewEntry()                                │
│  - 同时检查 entries 表和 entry_tombstones 表                        │
│  - 决定是否需要 Crawler 抓取全文（仅新条目抓取）                     │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 3: 事务内存在性检查（存储阶段）                               │
│  - entry.go:213-224 entryExists() 函数                              │
│  - SELECT true FROM entries WHERE feed_id=? AND hash=? LIMIT 1      │
│  - 使用 entries_feed_id_hash_key 唯一索引                           │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 4: 原子化插入（存储阶段）                                     │
│  - entry.go:81-146 createEntry() 函数                               │
│  - INSERT ... WHERE NOT EXISTS (SELECT FROM entry_tombstones)       │
│  - 利用数据库唯一约束 entries_feed_id_hash_key 防并发重复           │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.2 阶段 1: Hash 计算的具体算法

Hash 计算在各 feed 格式的 adapter 中完成，统一使用 SHA-256 十六进制编码：

**代码位置**: `internal/crypto/crypto.go:26-29`

```go
func SHA256(value string) string {
    h := sha256.Sum256([]byte(value))
    return hex.EncodeToString(h[:])  // 64 字符十六进制字符串
}
```

#### RSS 2.0 的 Hash 计算（最复杂）

**代码位置**: `internal/reader/rss/adapter.go:116-139`

```go
seenGUIDs := make(map[string]int)  // 检测重复 GUID
for _, item := range r.rss.Channel.Items {
    switch {
    case item.GUID.Data != "":
        n := seenGUIDs[item.GUID.Data]
        seenGUIDs[item.GUID.Data] = n + 1
        switch {
        case n == 0:
            // 第一次出现此 GUID: 直接使用 GUID
            entry.Hash = crypto.SHA256(item.GUID.Data)
        case entry.URL != "":
            // GUID 重复，有 URL: 使用 GUID + "|" + URL 消除歧义
            entry.Hash = crypto.SHA256(item.GUID.Data + "|" + entry.URL)
        default:
            // GUID 重复，无 URL: 使用 GUID + "|" + 序号
            entry.Hash = crypto.SHA256(item.GUID.Data + "|" + strconv.Itoa(n))
        }
    case entryURL != "":
        // 无 GUID，有 URL: 使用 URL
        entry.Hash = crypto.SHA256(entryURL)
    default:
        // 无 GUID 无 URL: 使用标题 + 内容
        entry.Hash = crypto.SHA256(entry.Title + entry.Content)
    }
}
```

**Hash 源优先级（RSS）**: `GUID` > `GUID + URL` > `GUID + 序号` > `URL` > `Title + Content`

**重复 GUID 处理的设计意图**: 有些不规范的 feed 为每个 item 使用相同的 GUID，`seenGUIDs` 计数器确保第一次出现保持原 hash 以兼容历史数据，后续出现用 URL 或序号消歧。

#### Atom 1.0 的 Hash 计算

**代码位置**: `internal/reader/atom/atom_10_adapter.go:149-155`

```go
for _, value := range []string{atomEntry.ID, atomEntry.Links.originalLink()} {
    if value != "" {
        entry.Hash = crypto.SHA256(value)
        break
    }
}
```

**Hash 源优先级（Atom）**: `atom:entry/id` > `atom:entry/link`

#### JSON Feed 的 Hash 计算

**代码位置**: `internal/reader/json/adapter.go:175-181`

```go
for _, value := range []string{item.ID, item.URL, item.ExternalURL, item.ContentText + item.ContentHTML + item.Summary} {
    value = strings.TrimSpace(value)
    if value != "" {
        entry.Hash = crypto.SHA256(value)
        break
    }
}
```

**Hash 源优先级（JSON Feed）**: `id` > `url` > `external_url` > `content_text + content_html + summary`

#### RDF / RSS 1.0 的 Hash 计算

**代码位置**: `internal/reader/rdf/adapter.go:74-79`

```go
hashValue := itemLink
if hashValue == "" {
    hashValue = item.Title + item.Description
}
entry.Hash = crypto.SHA256(hashValue)
```

**Hash 源优先级（RDF）**: `rdf:about` (link) > `Title + Description`

### 12.3 阶段 2: Processor 预检测

**代码位置**: `internal/reader/processor/processor.go:95`

```go
entryIsNew := store.IsNewEntry(feed.ID, entry.Hash)
contentExtractedSuccessfully := false
if feed.Crawler && (entryIsNew || forceRefresh) {
    // 仅对新条目抓取原文，避免重复抓取浪费资源
    webpageBaseURL := ""
    // ... 调用 crawler 抓取原文
}
```

`IsNewEntry` 同时检查两张表（`entry.go:278-293`）：

```go
func (s *Storage) IsNewEntry(feedID int64, entryHash string) bool {
    query := `
        SELECT
            EXISTS (SELECT 1 FROM entries WHERE feed_id=$1 AND hash=$2)
            OR
            EXISTS (SELECT 1 FROM entry_tombstones WHERE feed_id=$1 AND hash=$2)
    `
    var known bool
    s.db.QueryRow(query, feedID, entryHash).Scan(&known)
    return !known  // 两张表都不存在才算"新"
}
```

**性能优化**: 这个预检测是对 `Crawler` 功能的重要优化——跳过已有条目的网页抓取，节省大量时间和带宽。

### 12.4 阶段 3: 事务内存在性检查

**代码位置**: `internal/storage/entry.go:213-224` — `entryExists`

```go
func (s *Storage) entryExists(tx *sql.Tx, entry *model.Entry) (bool, error) {
    var result bool
    // Note: This query uses entries_feed_id_hash_key index
    err := tx.QueryRow(
        `SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1`,
        entry.FeedID, entry.Hash,
    ).Scan(&result)
    
    if err != nil && err != sql.ErrNoRows {
        return result, fmt.Errorf(`store: unable to check if entry exists: %v`, err)
    }
    return result, nil
}
```

**数据库索引**: `entries_feed_id_hash_key` 是 `unique (feed_id, hash)` 约束自动创建的索引，查询是 O(1) 复杂度。

**调用位置**: `RefreshFeedEntries` 函数中（`entry.go:325`），每个 entry 在事务内先检查再决定插入或更新。

### 12.5 阶段 4: 原子化插入与双重防重

实际写入时，Miniflux 使用**双重防重机制**确保并发安全：

#### 第一层: 数据库唯一约束

```sql
-- migrations.go:88
CONSTRAINT entries_feed_id_hash_key UNIQUE (feed_id, hash)
```

如果两个 Worker 同时检测到 `entryExists=false` 并尝试插入，PostgreSQL 的唯一约束会拒绝第二个插入，返回 `pq: duplicate key value violates unique constraint "entries_feed_id_hash_key"`。

#### 第二层: Tombstone 原子检查

**代码位置**: `entry.go:86-122` — `createEntry`

```go
func (s *Storage) createEntry(tx *sql.Tx, entry *model.Entry) error {
    query := `
        INSERT INTO entries (...)
        SELECT $1, $2, ...
        WHERE NOT EXISTS (
            SELECT 1 FROM entry_tombstones WHERE feed_id=$9 AND hash=$2
        )
        RETURNING id, status, created_at, changed_at
    `
    err := tx.QueryRow(query, ...).Scan(&entry.ID, ...)
    
    if errors.Is(err, sql.ErrNoRows) {
        return ErrEntryTombstoned  // 被 tombstone 阻止
    }
    if err != nil {
        return fmt.Errorf(...)
    }
    // ...
}
```

**原子性保证**: `INSERT ... WHERE NOT EXISTS` 是一个原子 SQL 语句，在 INSERT 执行的同时检查 tombstone。如果没有这个原子检查，`ArchiveEntries` 并发删除条目时，可能发生以下竞态：

```
TOCTOU 竞态场景（如果没有原子检查）:
  时间1: Worker A 检测 entryExists=false, 未检查 tombstone
  时间2: ArchiveEntries 删除该 entry 并写入 tombstone
  时间3: Worker A 插入 entry → "僵尸"条目复活
```

有了 `WHERE NOT EXISTS` 子查询，时间3的 INSERT 会被阻止，因为子查询检测到 tombstone 已存在。

### 12.6 两次存在性检查的差异和必要性

| 检查点 | 检查范围 | 用途 | 原子性 |
|--------|----------|------|--------|
| `entryExists` (事务内 SELECT) | 仅 `entries` 表 | 决定 INSERT 还是 UPDATE | 非原子（TOCTOU 窗口） |
| `WHERE NOT EXISTS` (INSERT 子查询) | 仅 `entry_tombstones` 表 | 防止已删除条目复活 | 原子 |

**为什么需要两次检查**:
1. `entryExists` 检查用于区分**已有条目**（需要 UPDATE）和**新条目**（需要 INSERT）
2. `WHERE NOT EXISTS` 检查专门用于防止**已删除条目**被重新插入
3. 两者都检查 `unique (feed_id, hash)` 约束，最终由数据库保证不重复

### 12.7 Hash 计算的设计权衡

| 设计决策 | 优点 | 缺点 |
|----------|------|------|
| 使用业务标识（GUID/ID/URL）而非内容 hash | 标题/内容修正不会产生重复条目；hash 稳定 | 真正的内容更新（原 URL 下文章重写）可能漏更 |
| SHA-256 十六进制 | 碰撞概率极低；标准算法 | 64 字符较长，占用存储空间 |
| 对重复 GUID 用 "|" 拼接 URL/序号 | 兼容不规范 feed；保持第一个 GUID 的兼容性 | 使 hash 生成逻辑复杂 |
| 不包含 `user_id` 在 hash 中 | 同 feed 下同一 entry 跨用户共享 hash | 每个用户下的 entry 仍需独立存储（不能跨用户共享） |

### 12.8 重复检测的完整竞态分析

```
高并发场景下的重复检测时序:

Worker 1                         Worker 2
   │                                │
   ├─ entryExists(feed_id, hash) → false
   │                                ├─ entryExists(feed_id, hash) → false
   │                                │
   ├─ BEGIN TRANSACTION             ├─ BEGIN TRANSACTION
   ├─ INSERT INTO entries ...       ├─ INSERT INTO entries ...
   ├─ 成功 (获得唯一约束 lock)      ├─ 阻塞 (等待唯一约束 lock)
   ├─ COMMIT                        │
   │                                ├─ 唯一约束违反错误!
   │                                └─ ROLLBACK + 返回错误
                                    
结果: 只有一个 Worker 成功插入，另一个收到重复键错误。
      这是正常的，上层会捕获并跳过。

Tombstone 并发场景:

Worker A (refresh)                Worker B (archive)
   │                                │
   ├─ IsNewEntry → true            ├─ BEGIN TRANSACTION
   │                                ├─ DELETE FROM entries WHERE ...
   │                                ├─ INSERT INTO entry_tombstones ...
   │                                ├─ COMMIT
   ├─ BEGIN TRANSACTION             │
   ├─ INSERT ... WHERE NOT EXISTS (SELECT FROM entry_tombstones)
   └─ 子查询检测到 tombstone → sql.ErrNoRows → ErrEntryTombstoned
                                    
结果: 原子检查阻止了已删除条目的复活。
```
