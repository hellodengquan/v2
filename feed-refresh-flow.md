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

---

## 十三、Feed 抓取的 Timeout 分级策略与分支

### 13.1 Timeout 层级总览

Miniflux 的 HTTP 超时并非单一值，而是在三个不同层级上分别控制的分级策略：

```
┌─────────────────────────────────────────────────────────────────────┐
│  层级 1: http.Client.Timeout（全局请求超时）                        │
│  - 从 DNS 解析到响应读取完毕的总时间上限                            │
│  - 包含所有 TCP/TLS 握手 + 服务器处理 + 响应读取                    │
│  - Feed 抓取: 20s (HTTP_CLIENT_TIMEOUT)                             │
│  - Media 代理: 120s (MEDIA_PROXY_HTTP_CLIENT_TIMEOUT)               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  层级 2: net.Dialer.Timeout（连接建立超时）                         │
│  - 仅控制 TCP 连接建立阶段                                         │
│  - 直连/代理均使用 10s                                             │
│  - 硬编码在 request_builder.go:159-167                              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  层级 3: http.Transport.IdleConnTimeout（空闲连接超时）             │
│  - 控制 Transport 连接池中空闲连接的保活时间                        │
│  - 固定 10s（默认 90s）                                            │
│  - 硬编码在 request_builder.go:197                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 13.2 层级 1: http.Client.Timeout 的分级

**代码位置**: `request_builder.go:240-242`

```go
client := &http.Client{
    Timeout: r.clientTimeout,
}
```

`http.Client.Timeout` 是 Go 的"总超时"——从拨号开始到响应体读取完毕。它覆盖了整个请求生命周期。

#### 两档配置

| 场景 | 配置项 | 默认值 | 说明 |
|------|--------|--------|------|
| Feed 抓取 / Scraper / WatchTime | `HTTP_CLIENT_TIMEOUT` | 20s | 所有 reader 和 processor 内的 HTTP 请求 |
| Media 代理 | `MEDIA_PROXY_HTTP_CLIENT_TIMEOUT` | 120s | UI proxy 代理多媒体资源（图片/视频） |

**设计意图**: Media 代理需要更长超时，因为它代理的是图片、视频等大文件，传输时间远超 RSS XML 文件。

#### 调用点对照

```
HTTP_CLIENT_TIMEOUT (20s) 使用场景:
├── handler.go:119          — CreateFeed (创建 feed 预抓取)
├── handler.go:227          — RefreshFeed (周期性刷新)
├── processor.go:56         — ProcessFeedEntries (scraper 请求构建器)
├── processor.go:187        — ProcessEntryWebPage (单条目网页抓取)
├── reading_time.go:23      — fetchWatchTime (YouTube/Nebula/Odysee/Bilibili)
├── youtube.go:96           — fetchYouTubeWatchTimeInBulk (批量 YouTube)
├── bilibili.go:47          — fetchBilibiliWatchTime (Bilibili)
├── icon/checker.go:32      — NewIconChecker (feed icon 抓取)
├── subscription_submit.go:61 — 订阅发现
└── opml_upload.go:94       — OPML 导入

MEDIA_PROXY_HTTP_CLIENT_TIMEOUT (120s) 使用场景:
└── ui/proxy.go:90          — WebUI 媒体代理 (图片/视频代理)
```

### 13.3 层级 2: net.Dialer.Timeout

**代码位置**: `request_builder.go:159-167`

```go
directDialer := &net.Dialer{
    Timeout:   10 * time.Second, // Default is 30s.
    KeepAlive: 15 * time.Second, // Default is 30s.
}

proxyDialer := &net.Dialer{
    Timeout:   10 * time.Second, // Default is 30s.
    KeepAlive: 15 * time.Second, // Default is 30s.
}
```

| 参数 | 值 | Go 默认值 | 说明 |
|------|-----|-----------|------|
| `Dialer.Timeout` | 10s | 30s | TCP 连接建立超时 |
| `Dialer.KeepAlive` | 15s | 30s | TCP keepalive 探测间隔 |

**设计意图**: 将连接超时从 30s 缩短到 10s，避免对不可达源站长时间阻塞。这对 RSS 刷新场景尤为重要——快速失败可以让 Worker 尽快处理下一个 Job。

### 13.4 层级 3: http.Transport 连接池参数

**代码位置**: `request_builder.go:192-198`

```go
transport := &http.Transport{
    Proxy:             http.ProxyFromEnvironment,
    ForceAttemptHTTP2: true,
    MaxIdleConns:      50,               // Default is 100.
    IdleConnTimeout:   10 * time.Second, // Default is 90s.
}
```

| 参数 | 值 | Go 默认值 | 说明 |
|------|-----|-----------|------|
| `MaxIdleConns` | 50 | 100 | 最大空闲连接数 |
| `IdleConnTimeout` | 10s | 90s | 空闲连接保活时间 |
| `ForceAttemptHTTP2` | true | false | 尝试 HTTP/2（即使自定义了 DialContext） |

**注意**: 每次 `ExecuteRequest` 都创建新的 `http.Transport` 和 `http.Client`，这意味着**连接池不会被复用**。`MaxIdleConns` 和 `IdleConnTimeout` 主要作用于同一请求内的 redirect 链，而非跨请求复用。

### 13.5 超时触发的错误处理路径

当超时发生时，Go 标准库返回的错误会被 `ResponseHandler.LocalizedError()` 分类：

```go
// response_handler.go:183
case os.IsTimeout(r.clientErr):
    return locale.NewLocalizedErrorWrapper(err, "error.network_timeout", r.clientErr)
```

在 RefreshFeed 中，超时错误的处理链路：

```
http.Client.Timeout 触发
    │
    ▼
ResponseHandler.clientErr = context.DeadlineExceeded
    │
    ▼
LocalizedError() → error.network_timeout
    │
    ▼
getTranslatedLocalizedError()
    │
    ├── originalFeed.WithTranslatedErrorMessage(err)  → ParsingErrorCount++
    ├── store.UpdateFeedError(originalFeed)            → 保存错误到数据库
    └── return localizedError                          → Worker 记录指标，继续下一个 Job
```

### 13.6 一次 Feed 刷新请求的超时预算

对于一次完整的 RefreshFeed，涉及的 HTTP 请求及其超时预算：

```
RefreshFeed 总耗时预算（无上限，但受 Worker 串行处理约束）
│
├── 1. 抓取 Feed XML: 1 次 HTTP 请求
│   └── http.Client.Timeout = 20s
│
├── 2. ProcessFeedEntries: N 次 HTTP 请求（每个 entry 一次）
│   ├── Scraper 抓取原文: 每个 entry 1 次 HTTP 请求 × 20s
│   ├── YouTube WatchTime: 可能 1 次 HTTP 请求 × 20s
│   ├── Bilibili WatchTime: 可能 1 次 HTTP 请求 × 20s
│   └── ... 其他平台
│
└── 3. Icon 抓取: 1 次 HTTP 请求
    └── http.Client.Timeout = 20s

理论最大耗时 = (1 + N + 1) × 20s
一个 feed 有 50 个 entry 时的最大耗时 ≈ 52 × 20s ≈ 17 分钟
```

**关键问题**: 单次 RefreshFeed 的总耗时没有上限。当一个 feed 配置了 Crawler 且有大量新 entry 时，Worker 可能被阻塞很长时间。这就是为什么 `WORKER_POOL_SIZE` 需要根据订阅源的 Crawler 配置合理设置。

---

## 十四、规则/重写过滤器在抓取后的执行链路

### 14.1 处理管线总览

Entry 在被抓取后经过一个**严格有序的处理管线**，每个阶段都在前一个阶段完成后执行：

```
Feed XML 解析完成
       │
       ▼
┌──────────────────────────────────────────────────────────────┐
│  ProcessFeedEntries (processor.go:27-177)                    │
│                                                              │
│  对每个 entry，按以下顺序执行:                                │
│                                                              │
│  1. 过滤（第一次） ──── before_scrape                        │
│  2. URL 清洗 ──── 移除跟踪参数                               │
│  3. URL 重写 ──── RewriteEntryURL                            │
│  4. 新旧检测 ──── IsNewEntry                                 │
│  5. 内容抓取 ──── Crawler/Scraper（仅新条目）                 │
│  6. 内容重写 ──── ApplyContentRewriteRules                    │
│  7. 过滤（第二次）── after_scrape（仅 Crawler 成功时）        │
│  8. HTML 消毒 ──── SanitizeHTML                              │
│  9. 阅读时间 ──── updateEntryReadingTime                     │
│  10. 加入 filteredEntries                                    │
└──────────────────────────────────────────────────────────────┘
```

### 14.2 阶段 1: 第一次过滤（before_scrape）

**代码位置**: `processor.go:75-86`

```go
if filter.IsBlockedEntry(blockRules, allowRules, feed, entry) {
    slog.Debug("Entry is blocked by filter rules", ..., slog.String("filter_stage", "before_scrape"))
    continue  // 直接跳过，不执行后续任何步骤
}
```

**目的**: 在抓取前就过滤掉不需要的条目，节省后续 Scraper 的 HTTP 请求。

**过滤规则执行顺序**（`filter.go:101-127`）：

```
1. Block filter rules (user + feed 合并)  → 匹配则阻止
2. Feed blocklist regex                   → 匹配则阻止
3. Keep filter rules (user + feed 合并)   → 不匹配则阻止
4. Feed keeplist regex                    → 不匹配则阻止
5. 全部通过 → 允许
```

**可用的 filter rule 类型**（`filter.go:179-205`）：

| 规则类型 | 匹配目标 | 说明 |
|----------|----------|------|
| `EntryTitle` | `entry.Title` | 标题正则匹配 |
| `EntryURL` | `entry.URL` | URL 正则匹配 |
| `EntryCommentsURL` | `entry.CommentsURL` | 评论 URL 正则匹配 |
| `EntryContent` | `entry.Content` | 内容正则匹配 |
| `EntryAuthor` | `entry.Author` | 作者正则匹配 |
| `EntryTag` | `entry.Tags` | 标签正则匹配 |
| `EntryDate` | `entry.Date` | 日期匹配（before/after/between/future/max-age） |

**正则缓存优化**（`filter.go:48-72`）：

```go
var compiledRegexesCache sync.Map
const maxCachedRegexes = 1024

func cachedRegex(pattern string) *regexp.Regexp {
    if v, ok := compiledRegexesCache.Load(pattern); ok {
        return v.(*regexp.Regexp)  // 缓存命中
    }
    re, _ := regexp.Compile(pattern)
    compiledRegexesCache.Store(pattern, re)
    if compiledRegexesCacheSize.Add(1) >= maxCachedRegexes {
        compiledRegexesCache.Clear()  // 缓存满时清空
        compiledRegexesCacheSize.Store(0)
    }
    return re
}
```

### 14.3 阶段 2: URL 清洗（移除跟踪参数）

**代码位置**: `processor.go:88-91`

```go
parsedInputUrl, _ := url.Parse(entry.URL)
if cleanedURL, err := urlcleaner.RemoveTrackingParameters(parsedFeedURL, parsedSiteURL, parsedInputUrl); err == nil {
    entry.URL = cleanedURL
}
```

**代码位置**: `internal/reader/urlcleaner/urlcleaner.go`

移除的跟踪参数包括：

| 类别 | 参数 |
|------|------|
| Facebook | `fbclid`, `_openstat`, `fb_action_ids`, `fb_action_types`, `fb_ref`, `fb_source`, `fb_comment_id` |
| Google | `gclid`, `dclid`, `gbraid`, `wbraid`, `gclsrc`, `srsltid` |
| Google Analytics | `campaign_id`, `campaign_medium`, `campaign_name`, `campaign_source`, `campaign_term`, `campaign_content` |
| Yandex | `yclid`, `ysclid` |
| Twitter | `twclid` |
| Microsoft | `msclkid` |
| Mailchimp | `mc_cid`, `mc_eid`, `mc_tc` |
| HubSpot | `hsa_cam`, `_hsenc`, `__hssc`, `__hstc`, `__hsfp`, `_hsmi`, `hsctatracking` |
| UTM 系列 | 所有 `utm_` 前缀 |
| Matomo | 所有 `mtm_` 前缀 |
| Outbound | `ref`（仅当值匹配 feed/site 域名时） |

### 14.4 阶段 3: URL 重写

**代码位置**: `processor.go:94`

```go
entry.URL = rewrite.RewriteEntryURL(feed, entry)
```

**代码位置**: `internal/reader/rewrite/url_rewrite.go`

```go
func RewriteEntryURL(feed *model.Feed, entry *model.Entry) string {
    if feed.UrlRewriteRules == "" {
        return entry.URL  // 无重写规则，直接返回
    }

    // 格式: rewrite("正则表达式"|"替换字符串")
    parts := customReplaceRuleRegex.FindStringSubmatch(feed.UrlRewriteRules)
    if len(parts) == 3 {
        re, _ := regexp.Compile(parts[1])
        return re.ReplaceAllString(entry.URL, parts[2])
    }
    return entry.URL
}
```

**用途**: 某些 feed 的 entry URL 不直接指向原文，而是经过中间跳转页。URL 重写规则可以修正这些 URL。

### 14.5 阶段 4: 新旧检测 + 阶段 5: 内容抓取

**代码位置**: `processor.go:95-142`

```go
entryIsNew := store.IsNewEntry(feed.ID, entry.Hash)
contentExtractedSuccessfully := false
if feed.Crawler && (entryIsNew || forceRefresh) {
    scrapedPageBaseURL, extractedContent, scraperErr := scraper.ScrapeWebsite(
        requestBuilder,
        entry.URL,
        feed.ScraperRules,
    )
    // ...
    if scraperErr != nil {
        // 抓取失败，保留原始 feed 内容
    } else if extractedContent != "" {
        entry.Content = minifyContent(extractedContent)
        contentExtractedSuccessfully = true  // 标记抓取成功
    }
}
```

**Scraper 的内容提取策略**（`scraper.go:21-71`）：

```
ScrapeWebsite
    │
    ├── 1. HTTP 请求抓取网页
    │   └── 使用与 feed 相同的 RequestBuilder（超时/代理/TLS 等配置共享）
    │
    ├── 2. 检查 Content-Type
    │   └── 仅接受 text/html 和 application/xhtml+xml
    │
    ├── 3. 判断是否同站
    │   └── 比较 entry URL 和有效 URL 的域名
    │
    ├── 4. 选择提取策略
    │   ├── 同站 + 有自定义规则 → findContentUsingCustomRules (goquery CSS 选择器)
    │   ├── 同站 + 无自定义规则 → getPredefinedScraperRules → findContentUsingCustomRules
    │   └── 不同站 → readability.ExtractContent (Readability 算法)
    │
    └── 5. 返回 baseURL + extractedContent
```

**HTML Minify**（`utils.go:69-72`）:

Scraper 提取的内容会经过 `minifyContent` 压缩，移除注释、多余空白、默认属性等。

### 14.6 阶段 6: 内容重写规则

**代码位置**: `processor.go:144`

```go
rewrite.ApplyContentRewriteRules(entry, feed.RewriteRules)
```

**代码位置**: `internal/reader/rewrite/content_rewrite.go:104-121`

```go
func ApplyContentRewriteRules(entry *model.Entry, customRewriteRules string) {
    rulesList := getPredefinedRewriteRules(entry.URL)
    if customRewriteRules != "" {
        rulesList = customRewriteRules  // 自定义规则覆盖预定义规则
    }

    rules := parseRules(rulesList)
    rules = append(rules, rule{name: "add_pdf_download_link"})  // 始终追加

    for _, rule := range rules {
        rule.applyRule(entry.URL, entry)
    }
}
```

**优先级**: `feed.RewriteRules`（非空） > `getPredefinedRewriteRules(entry.URL)`（按域名匹配）

**可用的内容重写规则**：

| 规则名 | 作用 |
|--------|------|
| `add_image_title` | 将 img alt 属性添加为图片标题 |
| `add_dynamic_image` | 替换 data-src 为 src（懒加载图片） |
| `add_dynamic_iframe` | 替换 data-src 为 src（懒加载 iframe） |
| `add_youtube_video` | 嵌入 YouTube 视频播放器 |
| `add_invidious_video` | 嵌入 Invidious 视频播放器 |
| `add_youtube_video_using_invidious_player` | YouTube 链接用 Invidious 播放器 |
| `add_youtube_video_from_id` | 从 YouTube video ID 嵌入播放器 |
| `add_pdf_download_link` | 添加 PDF 下载链接（始终追加） |
| `add_mailto_subject` | 将 mailto 链接主题添加为文本 |
| `add_enclosure_links` | 将附件链接添加到内容中 |
| `add_castopod_episode` | 嵌入 Castopod 播客播放器 |
| `nl2br` | 换行符转 `<br>` |
| `convert_text_link` / `convert_text_links` | 纯文本链接转 HTML 超链接 |
| `fix_medium_images` | 修复 Medium 图片加载 |
| `use_noscript_figure_images` | 使用 noscript 中的图片 |
| `replace("search"|"replace")` | 自定义正则搜索替换（内容） |
| `replace_title("search"|"replace")` | 自定义正则搜索替换（标题） |
| `remove("selector")` | CSS 选择器移除元素 |
| `base64_decode("selector")` | Base64 解码指定元素内容 |
| `add_hn_links_using_hack` | Hacker News 链接转换（hack） |
| `add_hn_links_using_opener` | Hacker News 链接转换（opener） |
| `remove_tables` | 移除 HTML 表格 |
| `remove_clickbait` | 标题去标题党 |
| `fix_ghost_cards` | 修复 Ghost 博客卡片 |
| `remove_img_blur_params` | 移除图片模糊参数 |

**预定义规则**（`content_rewrite_rules.go`）：

为特定网站内置了重写规则，如 `xkcd.com` → `add_image_title`、`youtube.com` → `add_youtube_video`、`medium.com` → `fix_medium_images` 等。

### 14.7 阶段 7: 第二次过滤（after_scrape）

**代码位置**: `processor.go:147-158`

```go
// Re-run filters only when extracted content replaced entry.Content.
if contentExtractedSuccessfully && filter.IsBlockedEntry(blockRules, allowRules, feed, entry) {
    slog.Debug("Entry is blocked by filter rules", ..., slog.String("filter_stage", "after_scrape"))
    continue
}
```

**设计意图**: Crawler 抓取到的全文内容可能与 RSS 中的摘要不同。用户可能希望基于全文内容进行过滤（例如 `EntryContent` 规则匹配抓取后的完整正文）。

**条件**: 只有当 Scraper 成功提取了内容（`contentExtractedSuccessfully=true`）时才重新过滤。如果 Scraper 失败，保留的是原始 RSS 内容，不需要重新过滤（已在第一次过滤中检查过）。

### 14.8 阶段 8: HTML 消毒

**代码位置**: `processor.go:164-165`

```go
// The sanitizer should always run at the end of the process to make sure unsafe HTML is filtered out.
entry.Content = sanitizer.SanitizeHTML(webpageBaseURL, entry.Content, &sanitizer.SanitizerOptions{OpenLinksInNewTab: user.OpenExternalLinksInNewTab})
```

**作为最后一道防线**: 无论内容经过多少次变换，HTML 消毒始终最后执行，确保输出安全。

### 14.9 阶段 9: 阅读时间计算

**代码位置**: `processor.go:167`

```go
updateEntryReadingTime(store, feed, entry, entryIsNew, user)
```

**代码位置**: `reading_time.go:61-107`

```
阅读时间计算优先级:

1. 视频 WatchTime（仅新条目）
   ├── YouTube → fetchYouTubeWatchTimeForSingleEntry
   ├── Nebula → fetchNebulaWatchTime
   ├── Odysee → fetchOdyseeWatchTime
   └── Bilibili → fetchBilibiliWatchTime

2. 已有条目 → store.GetReadTime(feed.ID, entry.Hash)  // 从数据库读取

3. 文本估算 → readingtime.EstimateReadingTime
   └── 基于 content 字数 / 用户配置的阅读速度
```

### 14.10 处理顺序：旧条目优先

**代码位置**: `processor.go:64-65`

```go
// Processing older entries first ensures that their creation timestamp is lower than newer entries.
for _, entry := range slices.Backward(feed.Entries) {
```

使用 `slices.Backward` 从列表末尾向前遍历（即先处理最旧的条目），确保旧条目的 `created_at` 时间戳小于新条目。这在条目没有发布日期时尤为重要——此时 `created_at` 就是排序依据。

### 14.11 完整处理管线数据流图

```
entry (来自 feed 解析)
  │
  ├─ ① IsBlockedEntry? ──── 是 ──→ 丢弃
  │   (block filter + blocklist + keep filter + keeplist)
  │
  ├─ ② RemoveTrackingParameters ──→ entry.URL (移除 utm_*, fbclid 等)
  │
  ├─ ③ RewriteEntryURL ─────────→ entry.URL (正则替换)
  │
  ├─ ④ IsNewEntry? ─────────────→ entryIsNew (决定是否 Crawler)
  │
  ├─ ⑤ Crawler? && (isNew || force)?
  │   ├── 是 → ScrapeWebsite ──→ entry.Content (全文替换)
  │   │         └── minifyContent
  │   └── 否 → 保留原始 feed 内容
  │
  ├─ ⑥ ApplyContentRewriteRules → entry.Content / entry.Title (规则变换)
  │
  ├─ ⑦ contentExtractedSuccessfully? && IsBlockedEntry?
  │   └── 是 ──→ 丢弃 (二次过滤)
  │
  ├─ ⑧ SanitizeHTML ────────────→ entry.Content (安全消毒)
  │
  ├─ ⑨ updateEntryReadingTime ──→ entry.ReadingTime
  │   ├── Video WatchTime (YouTube/Nebula/Odysee/Bilibili)
  │   ├── 数据库已有值
  │   └── 文本估算
  │
  └─ ⑩ 加入 filteredEntries ───→ 最终写入数据库
```

---

## 十五、Scheduler Ticker 与 Worker 信号通信代码

### 15.1 通信架构总览

```
┌──────────────┐     channel     ┌──────────────┐
│  Scheduler   │ ──── Push ────▶ │  Worker Pool │
│  (1 goroutine)│               │  (N goroutines)│
│              │                │              │
│  time.Tick   │                │  for range c │
│  (ticker)    │                │              │
└──────────────┘                └──────────────┘
       │                              │
       │ 每轮生成 Job 列表             │ 每个 Worker 串行消费
       │ 推入 channel                 │ 调用 RefreshFeed
       ▼                              ▼
  BatchBuilder                    handler.RefreshFeed
  (数据库查询)                    (HTTP + 解析 + 存储)
```

### 15.2 信号通信的原语：Go Channel

**代码位置**: `internal/worker/pool.go`

```go
type Pool struct {
    queue chan model.Job   // 无缓冲 channel
    wg    sync.WaitGroup
}
```

**关键设计决策**: 使用**无缓冲 channel** (`make(chan model.Job)`)，而非带缓冲的 channel。

#### 无缓冲 Channel 的语义

- 每个 `p.queue <- job` 调用会**阻塞**，直到有 Worker 从 channel 中取走这个 Job
- 这意味着 `pool.Push(jobs)` 的发送速率受限于 Worker 的消费速率
- 如果所有 Worker 都在忙，`Push` 会阻塞整个 Scheduler goroutine

#### 为什么选择无缓冲？

1. **天然限流**: Scheduler 不会堆积大量未处理的 Job，避免内存压力
2. **背压传导**: 当 Worker 处理慢时，Scheduler 自动减速，不会无限生成 Job
3. **简化设计**: 不需要额外的 Job 队列管理和超时机制

### 15.3 Scheduler 端：Ticker 驱动的生产循环

**代码位置**: `internal/cli/scheduler.go:33-51`

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
            pool.Push(jobs)
        }
    }
}
```

#### 时序分析

```
时间轴:
  T+0s        T+60s       T+120s      T+180s
    │           │            │           │
    ▼           ▼            ▼           ▼
  Tick①      Tick②       Tick③       Tick④
    │           │            │           │
    ├─ FetchJobs             ├─ FetchJobs
    ├─ Push(jobs)            ├─ Push(jobs)
    │  (可能阻塞)            │
    │                        │
  如果 Push 阻塞:         如果上一轮 Push
  Scheduler 等待          还没完成:
  Worker 消费             Tick② 的 FetchJobs
                          被延迟执行

关键: time.Tick 是固定间隔的 ticker，
不管上一轮 Push 是否完成，下一轮 Tick 都会准时到来。
但由于 Push 可能阻塞，实际执行频率可能低于 ticker 频率。
```

**注意**: `time.Tick` 返回的是一个只读 channel，`for range time.Tick(frequency)` 等价于每次从 ticker channel 中接收一个时间信号。如果 `Push` 阻塞时间超过 `frequency`，下一个 tick 会被"吞掉"（Go ticker channel 在没人接收时会丢弃信号）。

### 15.4 Push 的阻塞行为

**代码位置**: `internal/worker/pool.go:20-24`

```go
func (p *Pool) Push(jobs model.JobList) {
    for _, job := range jobs {
        p.queue <- job  // 阻塞直到有 Worker 接收
    }
}
```

**行为分析**:

| 场景 | Push 行为 | 耗时 |
|------|-----------|------|
| Worker 空闲 | 每个 Job 立即被接收 | ≈ 0 |
| Worker 忙，但队列可消化 | 每个 Job 短暂等待 | 毫秒级 |
| Worker 长时间忙（Crawler feed） | Push 严重阻塞 | 可能数分钟 |
| 所有 Worker 在 Crawler，50 个 entry | 50 个 Job 排队 | 可能 > 10 分钟 |

**无超时保护**: `Push` 没有设置超时。如果 Worker 全部阻塞在长时间的 RefreshFeed 上，Scheduler goroutine 也会被阻塞。

### 15.5 Worker 端：消费循环

**代码位置**: `internal/worker/worker.go:24-49`

```go
func (w *worker) Run(c <-chan model.Job, wg *sync.WaitGroup) {
    defer wg.Done()

    for job := range c {
        startTime := time.Now()
        localizedError := feedHandler.RefreshFeed(w.store, job.UserID, job.FeedID, false)

        if config.Opts.HasMetricsCollector() {
            status := metric.StatusSuccess
            if localizedError != nil {
                status = metric.StatusError
            }
            metric.BackgroundFeedRefreshDuration.WithLabelValues(status).Observe(time.Since(startTime).Seconds())
        }
    }
}
```

**`for job := range c` 的语义**:

- 从 channel 中取出一个 Job，**同时解除 Scheduler 的 `Push` 阻塞**
- 如果 channel 被关闭（`close(p.queue)`），循环自动退出
- 每个 Worker 串行处理：取出一个 Job → 执行 RefreshFeed → 取下一个 Job

### 15.6 信号流完整时序图

```
Scheduler goroutine                    Worker 0              Worker 1
      │                                   │                      │
  time.Tick 触发                           │                      │
      │                                   │                      │
  FetchJobs()                              │                      │
  ┌─── 耗时: DB 查询 ───┐                  │                      │
      │                                   │                      │
  jobs = [J0, J1, J2, J3, J4]             │                      │
      │                                   │                      │
  Push(jobs)                               │                      │
      │                                   │                      │
  p.queue <- J0 ──────────────────────▶  收到 J0                  │
      │                               RefreshFeed(J0)             │
  p.queue <- J1 ─────────────────────────────────────────────▶  收到 J1
      │                                                       RefreshFeed(J1)
  p.queue <- J2 ──── 阻塞，等 Worker 空闲 ────                   │
      │                                   │                      │
      │                             (J0 完成)                     │
      │                                   │                      │
  p.queue <- J2 ──────────────────────▶  收到 J2                  │
      │                               RefreshFeed(J2)             │
  p.queue <- J3 ──── 阻塞...              │                      │
      │                                                          │
      │                                                    (J1 完成)
      │                                                          │
  p.queue <- J3 ─────────────────────────────────────────────▶  收到 J3
      │                                                       RefreshFeed(J3)
  p.queue <- J4 ──── 阻塞...              │                      │
      │                                   │                      │
      │                             (J2 完成)                     │
      │                                   │                      │
  p.queue <- J4 ──────────────────────▶  收到 J4                  │
      │                               RefreshFeed(J4)             │
  Push 完成，等待下一个 Tick               │                      │
```

### 15.7 优雅关闭

**代码位置**: `internal/worker/pool.go:27-30` + `internal/cli/daemon.go:77-99`

```go
// Pool.Shutdown 关闭 channel 并等待所有 Worker 完成
func (p *Pool) Shutdown() {
    close(p.queue)    // 关闭 channel → Worker 的 for-range 循环退出
    p.wg.Wait()       // 等待所有 Worker 的 goroutine 结束
}
```

```
关闭时序:

主 goroutine
    │
    ├── <-stop (收到 SIGTERM/SIGINT)
    │
    ├── 关闭 HTTP 服务器 (5s 超时)
    │
    ├── pool.Shutdown()
    │   ├── close(p.queue)     → Worker 的 for-range 退出循环
    │   └── p.wg.Wait()        → 等待 Worker 完成当前正在处理的 Job
    │
    └── 进程退出
```

**注意**: Worker 不会中断正在执行的 `RefreshFeed`。如果一个 Worker 正在处理一个需要 10 分钟的 Crawler feed，`Shutdown()` 会等待它完成。没有强制的 Job 超时或取消机制。

### 15.8 设计特点与局限

| 特点 | 说明 |
|------|------|
| **无缓冲 channel** | 天然背压，但可能导致 Scheduler 阻塞 |
| **Scheduler 单 goroutine** | 简单可靠，但一次只能处理一个批次 |
| **无 Job 优先级** | 所有 Job 平等，手动刷新和自动刷新在同一个队列 |
| **无 Job 超时** | Worker 不取消长时间运行的 Job |
| **无 Job 去重** | 同一 feed 可能在连续两轮 Tick 中被调度（如果上一轮 Push 还在阻塞，下一轮 FetchJobs 可能再次选到同一个 feed） |
| **优雅关闭** | 通过 channel close + WaitGroup 实现，但可能等待时间较长 |
| **Ticker 信号丢失** | 如果 Push 阻塞超过 PollingFrequency，中间的 tick 信号会被丢弃 |

### 15.9 手动刷新如何进入同一队列

手动刷新（UI/API）**不经过** Scheduler 的 channel，而是直接调用 `RefreshFeed`：

```
自动刷新路径:
  Scheduler → BatchBuilder → FetchJobs → pool.Push → Worker → RefreshFeed

手动刷新路径 (UI):
  ui/feed_refresh.go → 直接调用 feedHandler.RefreshFeed (在 HTTP handler goroutine 中)

手动刷新路径 (API):
  api/feed_handlers.go → 直接调用 feedHandler.RefreshFeed (在 HTTP handler goroutine 中)
```

**关键区别**:
- 自动刷新经过 Worker Pool 的 channel，受 Worker 池并发度限制
- 手动刷新在 HTTP handler 的 goroutine 中直接执行，不受 Worker 池限制
- 这意味着在 Worker 全部忙碌时，手动刷新仍可立即执行（但会增加并发压力）

---

## 十六、Feed Icon / Favicon 抓取完整链路

### 16.1 抓取触发的三种时机

Icon 抓取由 `iconChecker` 驱动，在以下三种时机触发：

| 触发时机 | 代码位置 | 调用方式 | 说明 |
|----------|----------|----------|------|
| 1. 创建新 Feed | `handler.go:98` | `UpdateOrCreateFeedIcon()` | 强制更新，无论是否已存在 |
| 2. 创建订阅 (Discover) | `handler.go:189` | `UpdateOrCreateFeedIcon()` | 同上 |
| 3. 周期性刷新 | `handler.go:347-349` | 根据响应状态分支 | 200 OK 时 `UpdateOrCreateFeedIcon()`；304 Not Modified 时 `CreateFeedIconIfMissing()` |

**调用代码**:

```go
// handler.go:345-350
iconChecker := icon.NewIconChecker(store, originalFeed)
if responseHandler.Changed() {
    // 源站内容有变化 → 重新抓取图标（网站可能也换了）
    iconChecker.UpdateOrCreateFeedIcon()
} else {
    // 304 未变化 → 仅在缺失时抓取（可能是旧数据升级时）
    iconChecker.CreateFeedIconIfMissing()
}
```

### 16.2 iconChecker 的两种模式

**代码位置**: `internal/reader/icon/checker.go`

#### 模式 1: CreateFeedIconIfMissing（保守）

```go
func (c *iconChecker) CreateFeedIconIfMissing() {
    if c.store.HasFeedIcon(c.feed.ID) {  // 先检查 feed_icons 关联表
        slog.Debug("Feed icon already exists", ...)
        return
    }
    c.UpdateOrCreateFeedIcon()
}
```

- 用途: 304 Not Modified 时避免重复抓取
- 检查方式: `SELECT true FROM feed_icons WHERE feed_id=$1 LIMIT 1`

#### 模式 2: UpdateOrCreateFeedIcon（强制）

```go
func (c *iconChecker) UpdateOrCreateFeedIcon() {
    // 1. 构建与 feed 相同配置的请求（共享 UA / 代理 / TLS 等）
    requestBuilder := fetcher.NewRequestBuilder().
        WithUserAgent(c.feed.UserAgent, config.Opts.HTTPClientUserAgent()).
        WithCookie(c.feed.Cookie).
        WithTimeout(config.Opts.HTTPClientTimeout()).
        WithProxyRotator(proxyrotator.ProxyRotatorInstance).
        WithCustomFeedProxyURL(c.feed.ProxyURL).
        WithCustomApplicationProxyURL(config.Opts.HTTPClientProxyURL()).
        UseCustomApplicationProxyURL(c.feed.FetchViaProxy).
        IgnoreTLSErrors(c.feed.AllowSelfSignedCertificates).
        DisableHTTP2(c.feed.DisableHTTP2)

    // 2. 查找图标
    iconFinder := newIconFinder(requestBuilder, c.feed.SiteURL, c.feed.IconURL)
    icon, err := iconFinder.findIcon()

    // 3. 存储
    c.store.StoreFeedIcon(c.feed.ID, icon)
}
```

### 16.3 findIcon 的四级回退查找策略

**代码位置**: `internal/reader/icon/finder.go:49-85`

```
findIcon() 的查找顺序:
  │
  ├── ① feed.IconURL（RSS/Atom feed 中显式指定的图标 URL）
  │     └── 优先使用 feed 中 <icon> 或 <logo> 元素
  │
  ├── ② feed.SiteURL（网站首页）HTML 文档中的 link 标签
  │     ├── 搜索: <link rel="icon" href="...">
  │     ├── 搜索: <link rel="shortcut icon" href="...">
  │     ├── 搜索: <link rel="apple-touch-icon" href="...">
  │     └── 支持 data: URL（内联 base64 图标）
  │
  ├── ③ RootURL（网站根目录）HTML 文档中的 link 标签
  │     └── 当 SiteURL 是子目录时，额外检查根目录
  │
  └── ④ /favicon.ico（根目录默认图标）
        └── 尝试 RootURL + "/favicon.ico"
```

**HTML 文档解析**（`finder.go:273-312`）:

```go
query := `link[rel='icon' i][href],
    link[rel='shortcut icon' i][href],
    link[rel='icon shortcut' i][href],
    link[rel='apple-touch-icon'][href]`

for _, s := range doc.Find(query).EachIter() {
    href, _ := s.Attr("href")
    // 转换为绝对 URL，加入 iconURLs 列表
}
```

**Data URL 支持**（`finder.go:318-356`）:

支持 `data:image/...;base64,...` 和 `data:image/...;utf8,...` 格式的内联图标。

### 16.4 图标下载与后处理

**代码位置**: `internal/reader/icon/finder.go:157-271`

#### downloadIcon 流程

```
downloadIcon(iconURL)
    │
    ├── HTTP 请求（与 feed 抓取共享 RequestBuilder 配置）
    │
    ├── 读取响应体（受 HTTP_CLIENT_MAX_BODY_SIZE 限制）
    │
    ├── 计算 hash: crypto.HashFromBytes(responseBody)
    │
    └── resizeIcon(icon)
          │
          ├── SVG: minify 压缩（不改变尺寸）
          │
          ├── 位图格式检查（JPEG/PNG/GIF/WebP）:
          │   ├── 尺寸 > 4096 × 4096 或总像素 > 4096² → 拒绝（返回 nil）
          │   ├── 尺寸 ≤ 32 × 32 → 无需调整
          │   └── 尺寸 > 32 → 使用 draw.BiLinear 缩放到 32×32 PNG
          │
          └── 不支持格式 → 原样返回
```

**关键参数**:
- 最大尺寸: 4096 × 4096 像素
- 输出尺寸: 32 × 32 像素 PNG
- 缩放算法: `draw.BiLinear`（双线性插值）

### 16.5 StoreFeedIcon 的原子存储与去重

**代码位置**: `internal/storage/icon.go:99-147`

```go
func (s *Storage) StoreFeedIcon(feedID int64, icon *model.Icon) error {
    tx, _ := s.db.Begin()

    // 1. 按 hash 查找是否已有相同图标（跨 feed 去重）
    err := tx.QueryRow(`SELECT id FROM icons WHERE hash=$1`, icon.Hash).Scan(&icon.ID)
    if errors.Is(err, sql.ErrNoRows) {
        // 2. 新图标: 插入 icons 表
        tx.QueryRow(`INSERT INTO icons (hash, mime_type, content, external_id)
            VALUES ($1, $2, $3, $4) RETURNING id`,
            icon.Hash, normalizeMimeType(icon.MimeType),
            icon.Content, crypto.GenerateRandomStringHex(20),
        ).Scan(&icon.ID)
    }

    // 3. 先删除该 feed 原有关联（支持图标更新替换）
    tx.Exec(`DELETE FROM feed_icons WHERE feed_id=$1`, feedID)

    // 4. 建立 feed ↔ icon 关联
    tx.Exec(`INSERT INTO feed_icons (feed_id, icon_id) VALUES ($1, $2)`, feedID, icon.ID)

    tx.Commit()
}
```

**数据库表结构**:

```
icons 表:
  id           (主键)
  hash         (唯一索引 — 跨 feed 去重)
  mime_type
  content      (BLOB)
  external_id  (用于 URL 访问，随机字符串)

feed_icons 关联表:
  feed_id      (外键 → feeds.id, ON DELETE CASCADE)
  icon_id      (外键 → icons.id)
  primary key (feed_id, icon_id)
```

**跨 feed 图标共享**: 如果两个 feed 来自同一网站，它们的 icon hash 相同，`StoreFeedIcon` 会复用同一个 `icons` 行，通过不同的 `feed_icons` 行关联。

**孤立图标清理**（`CleanupOrphanIcons`）: 定期删除没有任何 feed 引用的 `icons` 行。

### 16.6 图标访问 URL

**代码位置**: `internal/ui/feed_icon.go`

```
GET /feed-icon/{externalIconID}
    │
    ├── storage.IconByExternalID(externalIconID)
    │
    ├── 缓存 72 小时（Cache-Control: public, max-age=259200）
    │     └── 基于 icon.Hash 做 ETag
    │
    └── 返回:
          ├── Content-Type: icon.MimeType
          └── Body: icon.Content（非 SVG 时附带 Content-Length 和 Last-Modified）
```

### 16.7 完整链路数据流图

```
RefreshFeed 完成
    │
    ├── 响应 Changed?
    │     ├── true  → UpdateOrCreateFeedIcon()   (强制更新)
    │     └── false → CreateFeedIconIfMissing()  (仅缺失时抓取)
    │
    └── iconChecker
          │
          └── iconFinder.findIcon()
                │
                ├── ① feed.IconURL? → downloadIcon
                ├── ② SiteURL HTML → <link rel=icon> → downloadIcon
                ├── ③ RootURL HTML → <link rel=icon> → downloadIcon
                └── ④ RootURL/favicon.ico → downloadIcon
                      │
                      ├── resizeIcon (32×32 PNG)
                      └── StoreFeedIcon
                            │
                            ├── icons 表（按 hash 去重，跨 feed 共享）
                            └── feed_icons 关联表
```

---

## 十七、用户阅读状态与未读计数维护

### 17.1 Entry 状态模型

**代码位置**: `internal/model/entry.go`

```go
const (
    EntryStatusUnread = "unread"
    EntryStatusRead   = "read"
    EntryStatusRemoved = "removed"  // 逻辑删除状态（代码中存在但较少使用）
)
```

**状态持久化字段**（entries 表）:

| 字段 | 类型 | 说明 |
|------|------|------|
| `status` | VARCHAR | `unread` / `read` / `removed` |
| `starred` | BOOLEAN | 收藏标记（独立于阅读状态） |
| `share_code` | VARCHAR | 分享代码（非空表示已分享，FlushHistory 时保留） |
| `created_at` | TIMESTAMP | 创建时间（归档时判断是否超过阈值） |
| `changed_at` | TIMESTAMP | 状态变更时间（每次 SET status 时更新为 now()） |

### 17.2 新 Entry 的默认状态

**代码位置**: `internal/storage/entry.go:86-122` — `createEntry`

```sql
INSERT INTO entries (..., status, ...)
SELECT ..., $6, ...   -- $6 = entry.Status
RETURNING id, status, created_at, changed_at
```

**代码位置**: `internal/model/entry.go:50` — `NewEntry()` 默认值

```go
func NewEntry() *Entry {
    return &Entry{
        Enclosures: make(EnclosureList, 0),
        Tags:       make([]string, 0),
        // Status 字段在模型中无默认值，由解析器或存储层决定
    }
}
```

**实际默认值**: 新 entry 在进入 `RefreshFeedEntries` 时，如果没有显式设置 status，**默认为 `unread`**（由 SQL 默认值 `DEFAULT 'unread'` 保证）。

### 17.3 自动标读: ShouldMarkAsReadOnView

**代码位置**: `internal/model/entry.go:60-74`

```go
func (e *Entry) ShouldMarkAsReadOnView(user *User) bool {
    // 1. 已经不是 unread，无需再次标记
    if e.Status != EntryStatusUnread {
        return false
    }

    // 2. 有音视频附件 + 用户开启"播放完成才标读" → 不立即标读
    if user.MarkReadOnMediaPlayerCompletion && e.Enclosures.ContainsAudioOrVideo() {
        return false
    }

    // 3. 取决于用户"查看即标读"设置
    return user.MarkReadOnView
}
```

**用户配置项**（User 模型）:

| 配置 | 默认值 | 说明 |
|------|--------|------|
| `MarkReadOnView` | true | 查看条目时立即标记为已读 |
| `MarkReadOnMediaPlayerCompletion` | false | 音视频播放完毕才标读 |
| `NoAutoMarkAsRead` | false | 完全不自动标读（全局关闭） |

**调用位置**（UI Handler）:

```go
// ui/unread_entry_feed.go:39-45
if entry.ShouldMarkAsReadOnView(user) {
    err = h.store.SetEntriesStatus(user.ID, []int64{entry.ID}, model.EntryStatusRead)
}
```

### 17.4 状态变更的数据库操作

#### 批量状态更新

**代码位置**: `internal/storage/entry.go:407-423`

```go
func (s *Storage) SetEntriesStatus(userID int64, entryIDs []int64, status string) error {
    query := `
        UPDATE entries
        SET status=$1, changed_at=now()
        WHERE user_id=$2 AND id=ANY($3)
    `
    s.db.Exec(query, status, userID, pq.Array(entryIDs))
}
```

使用 `id=ANY($3)` + `pq.Array(entryIDs)` 实现**单 SQL 批量更新**，避免 N+1 查询。

#### 可见性计数版本

**代码位置**: `internal/storage/entry.go:426-449`

```go
func (s *Storage) SetEntriesStatusAndCountVisible(userID int64, entryIDs []int64, status string) (int, error) {
    // 单条 CTE: UPDATE → JOIN feeds/categories → 统计 hide_globally=false 的数量
    query := `
        WITH updated AS (
            UPDATE entries SET status=$1, changed_at=now()
            WHERE user_id=$2 AND id=ANY($3)
            RETURNING feed_id
        )
        SELECT count(*)
        FROM updated u
          JOIN feeds f ON (f.id = u.feed_id)
          JOIN categories c ON (c.id = f.category_id)
        WHERE NOT f.hide_globally AND NOT c.hide_globally
    `
    // 返回变更的条目中"在全局视图中可见"的数量
}
```

**设计意图**: 用于 Fever/Google Reader API 中，需要返回被标记条目的"可见计数"（排除用户隐藏的 feed 和 category）。

#### 全部标读

**代码位置**: `internal/storage/entry.go:511-523`

```go
func (s *Storage) MarkAllAsRead(userID int64) error {
    query := `UPDATE entries SET status=$1, changed_at=now()
              WHERE user_id=$2 AND status=$3`
    s.db.Exec(query, model.EntryStatusRead, userID, model.EntryStatusUnread)
}
```

#### 收藏切换

**代码位置**: `internal/storage/entry.go:471-489`

```go
func (s *Storage) ToggleStarred(userID int64, entryID int64) error {
    query := `UPDATE entries SET starred = NOT starred, changed_at=now()
              WHERE user_id=$1 AND id=$2`
}
```

### 17.5 未读计数: GetNavMetadata

**代码位置**: `internal/storage/nav_metadata.go`

导航栏的未读计数、错误 feed 计数、是否启用保存功能在**单条 SQL** 中一次性查询：

```go
func (s *Storage) GetNavMetadata(userID int64) (NavMetadata, error) {
    query := `
        SELECT
            -- 未读条目计数（排除隐藏的 feed / category）
            (SELECT count(*)
               FROM entries e
               JOIN feeds f ON f.id = e.feed_id
               JOIN categories c ON c.id = f.category_id
              WHERE e.user_id = $1
                AND e.status = 'unread'
                AND f.hide_globally IS FALSE
                AND c.hide_globally IS FALSE
            ) AS count_unread,

            -- 是否启用了任何第三方保存集成
            (SELECT EXISTS(
                SELECT 1 FROM integrations
                 WHERE user_id = $1
                   AND (pinboard_enabled='t' OR instapaper_enabled='t' OR ...)
            )) AS has_save_entry,

            -- 错误 feed 计数（超过 POLLING_PARSING_ERROR_LIMIT 阈值）
            (SELECT count(*) FROM feeds
              WHERE user_id = $1 AND parsing_error_count >= $2
            ) AS count_error_feeds
    `
}
```

**调用频率**: 几乎每个 UI 页面渲染时都会调用（未读页、历史页、feed 列表、设置页等共 50+ 处）。

**性能影响**: 每次页面加载执行 3 个子查询。未读计数需要扫描 entries + feeds + categories 三表 JOIN，在大型部署中可能是性能热点。

### 17.6 已读条目归档与 Tombstone

**代码位置**: `internal/storage/entry.go:362-404` — `ArchiveEntries`

```go
func (s *Storage) ArchiveEntries(userID int64, markingDate time.Time) (int64, error) {
    query := `
        -- 子查询 1: 加锁选择待删除条目（防止并发归档冲突）
        WITH deleted AS (
            DELETE FROM entries
            WHERE user_id=$1
              AND starred is false
              AND share_code=''
              AND status='read'
              AND changed_at < $2
              AND id IN (
                  SELECT id FROM entries
                  WHERE user_id=$1 AND status='read'
                  ORDER BY changed_at ASC
                  FOR UPDATE SKIP LOCKED  -- ← 并发安全：跳过被其他事务锁定的行
                  LIMIT $3
              )
            RETURNING feed_id, hash
        )
        -- 子查询 2: 写入 tombstone 防止重新摄入
        INSERT INTO entry_tombstones (feed_id, hash)
        SELECT feed_id, hash FROM deleted WHERE hash <> ''
        ON CONFLICT (feed_id, hash) DO NOTHING
    `
}
```

**归档条件**（AND 全部满足）:
- `starred = false`（非收藏）
- `share_code = ''`（未分享）
- `status = 'read'`（已读）
- `changed_at < markingDate`（变更早于指定时间）

**参数**:
- `markingDate`: 由 CLEANUP_ARCHIVE_READ_DAYS_AFTER 配置（默认 60 天前）
- `LIMIT $3`: 由 CLEANUP_ARCHIVE_BATCH_SIZE 配置（默认 10000 条/次）

**并发安全**: 使用 `FOR UPDATE SKIP LOCKED` — 如果其他归档进程已锁定某些行，就跳过它们，避免死锁。

### 17.7 历史清空: FlushHistory

**代码位置**: `internal/storage/entry.go:491-508`

```go
func (s *Storage) FlushHistory(userID int64) error {
    query := `
        WITH deleted AS (
            DELETE FROM entries
            WHERE user_id=$1 AND status=$2 AND starred is false AND share_code=''
            RETURNING feed_id, hash
        )
        INSERT INTO entry_tombstones (feed_id, hash)
        SELECT feed_id, hash FROM deleted WHERE hash <> ''
        ON CONFLICT (feed_id, hash) DO NOTHING
    `
    s.db.Exec(query, userID, model.EntryStatusRead)
}
```

与 `ArchiveEntries` 的区别：
- 不按时间限制，删除**所有**非收藏/非分享的已读条目
- 不使用 `SKIP LOCKED`（用户主动触发，通常不会并发执行）

### 17.8 状态维护数据流图

```
新 Entry 入库 (createEntry)
    │
    └── 默认 status = 'unread'
          │
          ▼
    用户查看条目 (ShouldMarkAsReadOnView)
    ├── status == unread?
    ├── 有音视频 + MarkReadOnMediaPlayerCompletion? → 等待播放完成
    └── MarkReadOnView == true?
          │
          └── 是 → SetEntriesStatus([entryID], 'read')
                │
                └── changed_at = now()
                      │
                      ▼
              条目存在 N 天后
              (CLEANUP_ARCHIVE_READ_DAYS_AFTER)
                      │
                      ▼
              ArchiveEntries 后台清理
              ├── FOR UPDATE SKIP LOCKED (并发安全)
              ├── DELETE FROM entries
              └── INSERT INTO entry_tombstones (防止重复摄入)

用户交互:
  ├── 手动 ToggleStarred → starred 翻转
  ├── 手动 MarkAllAsRead → 所有 unread → read
  ├── 手动 FlushHistory  → 删除所有已读非收藏非分享
  └── SetEntriesStatus批量操作 → 任意 entryIDs 列表状态变更

未读计数:
  └── GetNavMetadata (几乎每个页面调用)
      ├── COUNT unread entries (JOIN feeds + categories, hide_globally 过滤)
      ├── EXISTS integrations (是否有保存集成)
      └── COUNT error feeds (parsing_error_count >= 阈值)
```

---

## 十八、第三方集成推送（Pocket / Instapaper 等）完整链路

### 18.1 两种推送模式

Miniflux 的第三方集成分两种模式，触发时机完全不同：

| 模式 | 触发时机 | 推送对象 | 入口函数 | 说明 |
|------|----------|----------|----------|------|
| **自动推送** | Feed 刷新检测到新 Entry 时 | 本次刷新的所有新条目 | `integration.PushEntries()` | 通知类集成（Webhook / Matrix / Discord / Slack / Ntfy / Pushover / Telegram / Apprise）+ 部分"稍后读"类自动推送 |
| **手动保存** | 用户点击"保存"按钮时 | 单个 Entry | `integration.SendEntry()` | 稍后读/书签类集成（Pinboard / Instapaper / Wallabag / Notion / Webhook 等 20+ 种） |

**调用位置**:

```go
// 自动推送 — handler.go:339-342
if len(newEntries) > 0 {
    userIntegrations, _ := store.Integration(userID)
    integration.PushEntries(originalFeed, newEntries, userIntegrations)
}

// 手动保存 — 由 UI "保存条目" handler 触发 (api / ui entry_save handler)
integration.SendEntry(entry, userIntegrations)
```

### 18.2 自动推送: PushEntries 的集成列表

**代码位置**: `internal/integration/integration.go:511-719`

自动推送支持的集成分为"批量通知"和"逐条推送"两类：

#### 批量通知类（一次性推送整个 feed 的新条目列表）

| 集成 | 类型 | 推送方式 | Feed 级开关 |
|------|------|----------|:---:|
| Matrix Bot | 即时通讯 | `matrixbot.PushEntries(feed, entries, ...)` | 无 |
| Webhook | 通用 | `webhook.SendNewEntriesWebhookEvent(feed, entries)` | 无（Feed 可覆盖 URL） |
| Ntfy | 推送通知 | `ntfy.SendMessages(feed, entries)` | `feed.NtfyEnabled` |
| Apprise | 多平台通知 | `apprise.SendNotification(feed, entries)` | 无（Feed 可覆盖 ServiceURLs） |
| Discord | 即时通讯 | `discord.SendDiscordMsg(feed, entries)` | 无 |
| Slack | 即时通讯 | `slack.SendSlackMsg(feed, entries)` | 无 |
| Pushover | 推送通知 | `pushover.SendMessages(feed, entries)` | `feed.PushoverEnabled` |

#### 逐条推送类（遍历新条目逐个推送）

| 集成 | 类型 | 推送方式 |
|------|------|----------|
| Telegram Bot | 即时通讯 | `telegrambot.PushEntry(feed, entry, ...)` — 每条目一次 HTTP 请求 |
| Readeck Push | 稍后读 | `readeck.CreateBookmark(entry.URL, ...)` — 每条目一次 |

#### Webhook URL 覆盖优先级

```go
// integration.go:536-542
var webhookURL string
if feed.WebhookURL != "" {
    webhookURL = feed.WebhookURL        // Feed 级自定义 URL（最高优先级）
} else {
    webhookURL = userIntegrations.WebhookURL  // 用户级全局 URL
}
```

同理 Apprise 也有 feed 级覆盖 (`feed.AppriseServiceURLs`)。

### 18.3 手动保存: SendEntry 的集成列表

**代码位置**: `internal/integration/integration.go:41-508`

手动保存支持的集成全部是"逐条"模式，覆盖 20+ 种稍后读/书签服务：

#### 稍后读 / 文档管理类

| 集成 | API 调用 | 推送内容 |
|------|----------|----------|
| **Pocket** → 实际为 `Pinboard`（Miniflux 无原生 Pocket） | `pinboard.NewClient(token).CreateBookmark(url, title, tags, toread)` | URL + 标题 + 标签 |
| **Instapaper** | `instapaper.NewClient(username, password).AddURL(url, title)` | URL + 标题 |
| Wallabag | `wallabag.NewClient(...).CreateEntry(url, title, content)` | URL + 标题 + 全文 |
| Notion | `notion.NewClient(token, pageID).UpdateDocument(url, title)` | URL + 标题（追加到指定页面） |
| Readeck | `readeck.NewClient(...).CreateBookmark(url, title, content)` | URL + 标题 + 全文 |
| Readwise | `readwise.NewClient(key).CreateDocument(url)` | URL |
| Omnivore | `omnivore.NewClient(key, url).SaveURL(url)` | URL |
| Karakeep | `karakeep.NewClient(key, url, tags).SaveURL(url)` | URL |

#### 书签 / 链接管理类

| 集成 | API 调用 |
|------|----------|
| Pinboard | `CreateBookmark(url, title, tags, toread)` |
| LinkAce | `AddURL(url, title)` |
| Linkding | `CreateBookmark(url, title)` |
| LinkTaco | `CreateBookmark(url, title, content)` |
| Linkwarden | `CreateBookmark(url, title)` |
| Raindrop | `CreateRaindrop(url, title)` |
| Shaarli | `CreateLink(url, title)` |
| Shiori | `CreateBookmark(url, title)` |
| Espial | `CreateLink(url, title, tags)` |
| Cubox | `SaveLink(url)` |
| Betula | `CreateBookmark(url, title, tags)` |
| archive.org | `SendURL(url)`（Wayback Machine 归档） |

#### 手动保存的 Webhook

```go
// integration.go:422-447 — 手动保存的 Webhook（SendEntry 内）
var webhookURL string
if entry.Feed != nil && entry.Feed.WebhookURL != "" {
    webhookURL = entry.Feed.WebhookURL
} else {
    webhookURL = userIntegrations.WebhookURL
}
webhookClient.SendSaveEntryWebhookEvent(entry)  // 事件类型不同: save_entry vs new_entries
```

### 18.4 典型集成实现: Pinboard / Instapaper / Wallabag

#### Pinboard（书签）

**代码位置**: `internal/integration/pinboard/pinboard.go` + `post.go`

```go
type Client struct { token string }

func (c *Client) CreateBookmark(url, title, tags string, markAsUnread bool) error {
    values := url.Values{}
    values.Set("url", url)
    values.Set("description", title)
    values.Set("tags", tags)
    values.Set("toread", boolToString(markAsUnread))  // "yes" / "no"

    apiEndpoint := "https://api.pinboard.in/v1/posts/add?auth_token=" + c.token + "&" + values.Encode()

    resp, _ := http.Get(apiEndpoint)  // Pinboard 使用 HTTP GET（非 RESTful）
    // 解析 XML 响应 <result code="done" />
}
```

#### Instapaper（稍后读）

**代码位置**: `internal/integration/instapaper/instapaper.go`

```go
type Client struct { username, password string }

func (c *Client) AddURL(url, title string) error {
    // 使用 XAuth 认证 (HTTP Basic)
    apiEndpoint := "https://www.instapaper.com/api/add"
    values := url.Values{}
    values.Set("url", url)
    values.Set("title", title)

    req, _ := http.NewRequest("POST", apiEndpoint, strings.NewReader(values.Encode()))
    req.SetBasicAuth(c.username, c.password)
    req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
    // 检查 201 Created
}
```

#### Wallabag（自托管稍后读）

**代码位置**: `internal/integration/wallabag/wallabag.go`

```go
type Client struct {
    baseURL, clientID, clientSecret, username, password, tags string
    onlyURL bool
}

func (c *Client) CreateEntry(url, title, content string) error {
    // 1. OAuth2 获取 access_token
    accessToken, err := c.getAccessToken()  // password grant

    // 2. POST /api/entries.json
    values := url.Values{}
    values.Set("url", url)
    if !c.onlyURL {  // WallabagOnlyURL 开关控制是否发送全文
        values.Set("title", title)
        values.Set("content", content)
        values.Set("tags", c.tags)
    }

    req, _ := http.NewRequest("POST", c.baseURL+"/api/entries.json", ...)
    req.Header.Set("Authorization", "Bearer "+accessToken)
}
```

### 18.5 两种 Webhook 事件类型

**代码位置**: `internal/integration/webhook/webhook.go`

Webhook 集成区分两种事件，payload 不同：

| 事件 | 触发 | Webhook 方法 | Payload |
|------|------|-------------|---------|
| `new_entries` | 自动推送 (PushEntries) | `SendNewEntriesWebhookEvent(feed, entries)` | 包含 feed 信息 + 多个 entry |
| `save_entry` | 手动保存 (SendEntry) | `SendSaveEntryWebhookEvent(entry)` | 仅单个 entry 信息 |

**安全认证**:

```go
type Client struct { webhookURL, secret string }

func (c *Client) sendWebhook(payload any, eventType string) error {
    body, _ := json.Marshal(payload)

    // HMAC-SHA256 签名
    mac := hmac.New(sha256.New(), []byte(c.secret))
    mac.Write(body)
    signature := "sha256=" + hex.EncodeToString(mac.Sum(nil))

    req.Header.Set("X-Miniflux-Event-Type", eventType)
    req.Header.Set("X-Miniflux-Signature", signature)
}
```

接收方可以用共享密钥 (`WebhookSecret`) 验证 `X-Miniflux-Signature` 确保请求来自 Miniflux。

### 18.6 集成配置模型

**代码位置**: `internal/model/integration.go`（对应数据库 `integrations` 表）

```
integrations 表 (~100 列):
  user_id (主键)

  ├── 通知类 (PushEntries 自动触发):
  │   ├── matrix_bot_enabled, matrix_bot_url, matrix_bot_user, matrix_bot_password, matrix_bot_chat_id
  │   ├── webhook_enabled, webhook_url, webhook_secret
  │   ├── ntfy_enabled, ntfy_url, ntfy_topic, ntfy_api_token, ntfy_username, ntfy_password, ntfy_icon_url
  │   ├── apprise_enabled, apprise_url, apprise_services_url
  │   ├── discord_enabled, discord_webhook_link
  │   ├── slack_enabled, slack_webhook_link
  │   ├── pushover_enabled, pushover_user, pushover_token, pushover_device, pushover_prefix
  │   ├── telegram_bot_enabled, telegram_bot_token, telegram_bot_chat_id, telegram_bot_topic_id
  │   └── telegram_bot_disable_web_page_preview, telegram_bot_disable_notification, telegram_bot_disable_buttons
  │
  ├── 稍后读/书签类 (SendEntry 手动触发 + 部分 PushEntries 自动):
  │   ├── betula_enabled, betula_url, betula_token
  │   ├── pinboard_enabled, pinboard_token, pinboard_tags, pinboard_mark_as_unread
  │   ├── instapaper_enabled, instapaper_username, instapaper_password
  │   ├── wallabag_enabled, wallabag_url, wallabag_client_id, wallabag_client_secret
  │   │   └── wallabag_username, wallabag_password, wallabag_tags, wallabag_only_url
  │   ├── notion_enabled, notion_token, notion_page_id
  │   ├── readeck_enabled, readeck_url, readeck_api_key, readeck_labels, readeck_only_url
  │   │   └── readeck_push_enabled (同时参与自动推送)
  │   ├── readwise_enabled, readwise_api_key
  │   ├── linkace_enabled, linkace_url, linkace_api_key, linkace_tags, linkace_private, linkace_check_disabled
  │   ├── linkding_enabled, linkding_url, linkding_api_key, linkding_tags, linkding_mark_as_unread
  │   ├── linktaco_enabled, linktaco_api_token, linktaco_org_slug, linktaco_tags, linktaco_visibility
  │   ├── linkwarden_enabled, linkwarden_url, linkwarden_api_key, linkwarden_collection_id
  │   ├── nunux_keeper_enabled, nunux_keeper_url, nunux_keeper_api_key
  │   ├── omnivore_enabled, omnivore_api_key, omnivore_url
  │   ├── karakeep_enabled, karakeep_api_key, karakeep_url, karakeep_tags
  │   ├── raindrop_enabled, raindrop_token, raindrop_collection_id, raindrop_tags
  │   ├── espial_enabled, espial_url, espial_api_key, espial_tags
  │   ├── cubox_enabled, cubox_api_link
  │   ├── shiori_enabled, shiori_url, shiori_username, shiori_password
  │   ├── shaarli_enabled, shaarli_url, shaarli_api_secret
  │   └── archiveorg_enabled
  │
  └── API 兼容类 (独立协议实现):
      ├── fever_enabled, fever_username, fever_token (md5(username:password))
      └── googlereader_enabled, googlereader_username, googlereader_password (bcrypt hash)
```

### 18.7 Feed 级 vs 用户级集成覆盖

部分集成分成两级配置，Feed 级可覆盖用户级：

| 集成 | 用户级配置 | Feed 级覆盖 | 覆盖字段 |
|------|----------|-------------|----------|
| Webhook | `WebhookURL` | `feed.WebhookURL` | URL |
| Apprise | `AppriseServicesURL` | `feed.AppriseServiceURLs` | 服务 URL 列表 |
| Ntfy | `NtfyTopic` | `feed.NtfyTopic` | Topic |
| Ntfy | `NtfyEnabled` | `feed.NtfyEnabled` | 启用开关 |
| Pushover | (全局配置) | `feed.PushoverEnabled`, `feed.PushoverPriority` | 开关 + 优先级 |

### 18.8 推送数据流图

```
RefreshFeed 完成
    │
    ├── newEntries 非空?
    │
    └── integration.PushEntries(feed, newEntries, userIntegrations)
          │
          ├── 批量通知类 (一条 HTTP 请求)
          │   ├── Matrix Bot → matrixbot.PushEntries
          │   ├── Webhook     → webhook.SendNewEntriesWebhookEvent (HMAC-SHA256 签名)
          │   ├── Discord     → discord.SendDiscordMsg
          │   ├── Slack       → slack.SendSlackMsg
          │   ├── Ntfy        → ntfy.SendMessages (需 feed.NtfyEnabled)
          │   ├── Apprise     → apprise.SendNotification
          │   └── Pushover    → pushover.SendMessages (需 feed.PushoverEnabled)
          │
          └── 逐条推送类 (每条目一条 HTTP 请求)
              ├── TelegramBot → telegrambot.PushEntry (N 条)
              └── Readeck Push → readeck.CreateBookmark (N 条, 需 ReadeckPushEnabled)

用户点击"保存条目"按钮
    │
    └── integration.SendEntry(entry, userIntegrations)
          │
          ├── 稍后读类
          │   ├── Instapaper  → instapaper.AddURL (HTTP Basic Auth)
          │   ├── Wallabag    → wallabag.CreateEntry (OAuth2 password grant)
          │   ├── Notion      → notion.UpdateDocument
          │   ├── Readeck     → readeck.CreateBookmark
          │   ├── Readwise    → readwise.CreateDocument
          │   ├── Omnivore    → omnivore.SaveURL
          │   └── Karakeep    → karakeep.SaveURL
          │
          ├── 书签类
          │   ├── Pinboard    → pinboard.CreateBookmark (HTTP GET)
          │   ├── LinkAce / Linkding / LinkTaco / Linkwarden
          │   ├── Shaarli / Shiori / Espial / Raindrop
          │   └── Cubox / Betula
          │
          ├── 归档类
          │   └── archive.org → archiveorg.SendURL
          │
          └── Webhook (save_entry 事件)
              └── webhook.SendSaveEntryWebhookEvent (HMAC-SHA256 签名)
```
