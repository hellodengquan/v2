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
