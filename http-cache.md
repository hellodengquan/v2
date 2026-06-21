# HTTP 缓存机制分析

本文档从代码实现角度梳理 Miniflux 在拉取外部 Feed 时的 HTTP 缓存机制，包括缓存命中、ETag 和 Last-Modified 协商、304 响应处理的全链路流程。

## 一、核心概念

### 1.1 HTTP 缓存协商机制

Miniflux 使用 HTTP 标准的条件请求（Conditional Requests）机制进行缓存协商：

- **ETag（Entity Tag）**：服务器为资源生成的唯一标识符，通过 `If-None-Match` 请求头进行协商
- **Last-Modified**：资源最后修改时间，通过 `If-Modified-Since` 请求头进行协商
- **304 Not Modified**：当资源未变化时，服务器返回 304 状态码，响应体为空，节省带宽

### 1.2 缓存相关的 HTTP 响应头

| 响应头 | 作用 |
|--------|------|
| `ETag` | 资源实体标签，用于强缓存验证 |
| `Last-Modified` | 资源最后修改时间，用于弱缓存验证 |
| `Cache-Control: max-age=N` | 指示资源在 N 秒内新鲜 |
| `Expires` | 指示资源过期的绝对时间 |
| `Retry-After` | 限流时指示多久后可以重试 |

## 二、数据模型层

### 2.1 Feed 模型中的缓存字段

在 `internal/model/feed.go` 中，Feed 结构体包含以下缓存相关字段：

```go
type Feed struct {
    // ... 其他字段
    EtagHeader         string    `json:"etag_header"`           // 上次响应的 ETag 值
    LastModifiedHeader string    `json:"last_modified_header"` // 上次响应的 Last-Modified 值
    IgnoreHTTPCache    bool      `json:"ignore_http_cache"`    // 是否忽略 HTTP 缓存
    NextCheckAt        time.Time `json:"next_check_at"`        // 下次检查时间
    CheckedAt          time.Time `json:"checked_at"`           // 上次检查时间
    // ...
}
```

### 2.2 数据库持久化

数据库表 `feeds` 中对应的列（`internal/database/migrations.go`）：

```sql
etag_header text default '',
last_modified_header text default '',
```

在 `internal/storage/feed.go` 中，`CreateFeed` 和 `UpdateFeed` 方法都会持久化这两个字段。

## 三、请求构建层

### 3.1 RequestBuilder 概述

`internal/reader/fetcher/request_builder.go` 中的 `RequestBuilder` 负责构建 HTTP 请求，使用 Builder 模式链式调用。

### 3.2 缓存协商请求头设置

#### 3.2.1 WithETag - 设置 If-None-Match

```go
func (r *RequestBuilder) WithETag(etag string) *RequestBuilder {
    if etag != "" {
        r.headers.Set("If-None-Match", etag)
    }
    return r
}
```

- 当 ETag 非空时，设置 `If-None-Match` 请求头
- 服务器收到后会比较资源的当前 ETag 与该值
- 如果相同，返回 304 Not Modified

#### 3.2.2 WithLastModified - 设置 If-Modified-Since

```go
func (r *RequestBuilder) WithLastModified(lastModified string) *RequestBuilder {
    if lastModified != "" {
        r.headers.Set("If-Modified-Since", lastModified)
    }
    return r
}
```

- 当 Last-Modified 非空时，设置 `If-Modified-Since` 请求头
- 服务器收到后会比较资源的最后修改时间与该值
- 如果资源未修改，返回 304 Not Modified

### 3.3 请求执行

`ExecuteRequest` 方法构建 `http.Client` 和 `http.Request`，将所有设置的请求头（包括缓存协商头）附加到请求中，然后执行 `client.Do(req)`。

## 四、响应处理层

### 4.1 ResponseHandler 概述

`internal/reader/fetcher/response_handler.go` 中的 `ResponseHandler` 负责处理 HTTP 响应，封装了 `*http.Response`。

### 4.2 缓存响应头获取

#### 4.2.1 ETag() - 获取 ETag 响应头

```go
func (r *ResponseHandler) ETag() string {
    // Ignore caching headers for feeds that do not want any cache.
    if r.httpResponse.Header.Get("Expires") == "0" {
        return ""
    }
    return r.httpResponse.Header.Get("ETag")
}
```

- 如果 `Expires: 0`，说明服务器明确不希望缓存，返回空字符串
- 否则返回 `ETag` 响应头的值

#### 4.2.2 LastModified() - 获取 Last-Modified 响应头

```go
func (r *ResponseHandler) LastModified() string {
    // Ignore caching headers for feeds that do not want any cache.
    if r.httpResponse.Header.Get("Expires") == "0" {
        return ""
    }
    return r.httpResponse.Header.Get("Last-Modified")
}
```

- 同样，如果 `Expires: 0`，返回空字符串
- 否则返回 `Last-Modified` 响应头的值

### 4.3 内容修改判断

#### IsModified() - 判断资源是否修改

```go
func (r *ResponseHandler) IsModified(lastEtagValue, lastModifiedValue string) bool {
    if r.httpResponse.StatusCode == http.StatusNotModified {
        return false
    }

    if r.ETag() != "" {
        return r.ETag() != lastEtagValue
    }

    if r.LastModified() != "" {
        return r.LastModified() != lastModifiedValue
    }

    return true
}
```

**判断逻辑：**

1. **304 状态码**：如果响应状态码是 304 Not Modified，直接返回 `false`（未修改）
2. **ETag 优先**：如果响应中有 ETag，优先比较 ETag 是否变化（符合 RFC 9110 第 8.8.1 节）
3. **Last-Modified 兜底**：如果没有 ETag 但有 Last-Modified，比较最后修改时间
4. **默认修改**：如果两者都没有，默认认为内容已修改

**ETag 优先原则**（来自测试用例 `TestIsModified`）：

| 场景 | ETag 变化 | Last-Modified 变化 | IsModified 结果 |
|------|-----------|-------------------|----------------|
| 304 响应 | - | - | false |
| 都未变 | 否 | 否 | false |
| 仅 Last-Modified 变 | 否 | 是 | false（ETag 优先） |
| 仅 ETag 变 | 是 | 否 | true |
| 都变了 | 是 | 是 | true |

### 4.4 缓存有效期解析

#### 4.4.1 CacheControlMaxAge() - 解析 Cache-Control: max-age

```go
func (r *ResponseHandler) CacheControlMaxAge() time.Duration {
    cacheControlHeaderValue := r.httpResponse.Header.Get("Cache-Control")
    if cacheControlHeaderValue != "" {
        for directive := range strings.SplitSeq(cacheControlHeaderValue, ",") {
            if after, ok := strings.CutPrefix(strings.TrimSpace(directive), "max-age="); ok {
                if maxAge, err := strconv.Atoi(after); err == nil {
                    return time.Duration(maxAge) * time.Second
                }
            }
        }
    }
    return 0
}
```

- 解析 `Cache-Control` 响应头中的 `max-age` 指令
- 返回值为 `time.Duration` 类型，表示资源的新鲜期
- 如果解析失败或没有该指令，返回 0

#### 4.4.2 Expires() - 解析 Expires 响应头

```go
func (r *ResponseHandler) Expires() time.Duration {
    expiresHeaderValue := r.httpResponse.Header.Get("Expires")
    if expiresHeaderValue != "" {
        t, err := time.Parse(time.RFC1123, expiresHeaderValue)
        if err == nil {
            // This rounds up to the next minute by rounding down and just adding a minute.
            return time.Until(t).Truncate(time.Minute) + time.Minute
        }
    }
    return 0
}
```

- 解析 `Expires` 响应头（RFC1123 格式）
- 计算当前时间到过期时间的差值，向上取整到分钟
- 如果解析失败或没有该头，返回 0

### 4.5 限流重试解析

#### ParseRetryDelay() - 解析 Retry-After 响应头

```go
func (r *ResponseHandler) ParseRetryDelay() time.Duration {
    retryAfterHeaderValue := r.httpResponse.Header.Get("Retry-After")
    if retryAfterHeaderValue != "" {
        // First, try to parse as an integer (number of seconds)
        if seconds, err := strconv.Atoi(retryAfterHeaderValue); err == nil {
            return time.Duration(seconds) * time.Second
        }
        // If not an integer, try to parse as an HTTP-date
        if t, err := time.Parse(time.RFC1123, retryAfterHeaderValue); err == nil {
            return time.Until(t).Truncate(time.Second)
        }
    }
    return 0
}
```

- 当服务器返回 429 Too Many Requests 时，解析 `Retry-After` 头
- 支持两种格式：整数（秒数）和 HTTP-date（绝对时间）
- 用于调整下次检查时间

## 五、业务逻辑层

### 5.1 RefreshFeed 主流程

`internal/reader/handler/handler.go` 中的 `RefreshFeed` 函数是 feed 刷新的核心入口。

#### 5.1.1 完整流程

```
1. 从数据库获取原始 Feed 信息
2. 设置 CheckedAt 和初始 NextCheckAt
3. 构建 HTTP 请求（决定是否带上缓存头）
4. 执行 HTTP 请求
5. 处理限流（429）- 调整下次检查时间
6. 检查响应错误
7. 检查 URL 是否重复
8. 判断内容是否修改
   ├─ 未修改（304 或缓存头相同）
   │  └─ 更新 Last-Modified（如果有新值）
   └─ 已修改
      ├─ 读取响应体
      ├─ 解析 Feed
      ├─ 计算刷新延迟（TTL、Cache-Control、Expires 的最大值）
      ├─ 重新计算 NextCheckAt
      ├─ 处理条目（processor + storage）
      ├─ 触发集成推送
      └─ 更新 ETag 和 Last-Modified
9. 重置错误计数器
10. 更新 Feed 到数据库
```

#### 5.1.2 缓存头的条件使用

```go
ignoreHTTPCache := originalFeed.IgnoreHTTPCache || forceRefresh
if !ignoreHTTPCache {
    requestBuilder = requestBuilder.
        WithETag(originalFeed.EtagHeader).
        WithLastModified(originalFeed.LastModifiedHeader)
}
```

- 当 `IgnoreHTTPCache` 为 `true` 或 `forceRefresh` 为 `true` 时，不使用缓存
- 否则，在请求中带上 `If-None-Match` 和 `If-Modified-Since` 头

#### 5.1.3 内容修改判断与处理

```go
if ignoreHTTPCache || responseHandler.IsModified(originalFeed.EtagHeader, originalFeed.LastModifiedHeader) {
    // 内容已修改：读取并解析响应体，更新条目
    responseBody, _ := responseHandler.ReadBody(...)
    updatedFeed, _ := parser.ParseFeed(...)
    
    // 计算刷新延迟：取 TTL、Cache-Control、Expires 的最大值
    feedTTLValue := updatedFeed.TTL
    cacheControlMaxAgeValue := responseHandler.CacheControlMaxAge()
    expiresValue := responseHandler.Expires()
    refreshDelay := max(feedTTLValue, cacheControlMaxAgeValue, expiresValue)
    
    // 重新计算下次检查时间
    originalFeed.ScheduleNextCheck(weeklyEntryCount, refreshDelay)
    
    // 处理条目
    processor.ProcessFeedEntries(store, originalFeed, userID, forceRefresh)
    store.RefreshFeedEntries(...)
    
    // 更新缓存头
    originalFeed.EtagHeader = responseHandler.ETag()
    originalFeed.LastModifiedHeader = responseHandler.LastModified()
} else {
    // 内容未修改（304）
    // 按 RFC9111 sections 3.2 和 4.3.4 更新 Last-Modified
    if responseHandler.LastModified() != "" {
        originalFeed.LastModifiedHeader = responseHandler.LastModified()
    }
}
```

**关键点：**

1. **修改时的刷新延迟计算**：取 RSS TTL、`Cache-Control: max-age`、`Expires` 三者的最大值作为刷新延迟
2. **未修改时的更新**：即使返回 304，如果响应中有新的 `Last-Modified` 值，也需要更新（符合 RFC 9111 规范）
3. **强制刷新**：`forceRefresh` 为 true 时，忽略缓存，强制重新拉取

### 5.2 下次检查时间调度

`ScheduleNextCheck` 方法（`internal/model/feed.go`）负责计算 `NextCheckAt`：

```go
func (f *Feed) ScheduleNextCheck(weeklyCount int, refreshDelay time.Duration) time.Duration {
    // 1. 根据调度策略确定基础间隔
    interval := config.Opts.SchedulerRoundRobinMinInterval()
    if config.Opts.PollingScheduler() == SchedulerEntryFrequency {
        // 根据条目频率动态调整
    }
    
    // 2. 使用 HTTP 缓存头或 RSS TTL 调整间隔（取较大值）
    interval = max(interval, refreshDelay)
    
    // 3. 限制最大间隔（防止配置错误的 feed）
    interval = min(interval, config.Opts.Scheduler...MaxInterval())
    
    f.NextCheckAt = time.Now().Add(interval)
    return interval
}
```

## 六、调度与执行层

### 6.1 批量刷新流程

`internal/cli/refresh_feeds.go` 中的 `refreshFeeds` 函数负责批量刷新 feeds：

```
1. 从数据库获取需要刷新的 feed 批次
   - 排除已禁用的 feed
   - 只取 next_check_at 已过期的
   - 考虑错误次数限制
   - 按主机限制并发
2. 创建 worker 池
3. 将任务分发到 worker 队列
4. 每个 worker 调用 feedHandler.RefreshFeed()
5. 等待所有任务完成
```

### 6.2 并发刷新

- 使用 `sync.WaitGroup` 管理 worker 生命周期
- 使用 channel 作为任务队列
- worker 数量由配置项 `WorkerPoolSize` 决定

## 七、全链路流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     调度层 (Scheduler)                       │
│  - refreshFeeds() 批量获取待刷新 feeds                       │
│  - Worker 池并发执行 RefreshFeed()                          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  业务逻辑层 (Handler)                        │
│  - RefreshFeed() 主流程                                     │
│  - 判断是否使用缓存 (IgnoreHTTPCache / forceRefresh)        │
│  - 调用 RequestBuilder 构建请求                              │
│  - 调用 ResponseHandler 处理响应                              │
│  - 判断内容是否修改                                          │
│  - 更新缓存字段 (ETag, Last-Modified)                        │
│  - 计算 NextCheckAt                                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
┌─────────────────────┐    ┌─────────────────────┐
│  请求构建层          │    │  响应处理层          │
│  (RequestBuilder)   │    │  (ResponseHandler)  │
│                     │    │                     │
│  - WithETag()       │    │  - ETag()           │
│    → If-None-Match  │    │  - LastModified()   │
│  - WithLastModified()│   │  - IsModified()     │
│    → If-Modified-Since│  │  - CacheControlMaxAge()│
│  - ExecuteRequest() │    │  - Expires()        │
└─────────────────────┘    │  - ParseRetryDelay()│
                           └──────────┬──────────┘
                                      │
                                      ▼
                           ┌─────────────────────┐
                           │   存储层 (Storage)   │
                           │                     │
                           │  - CreateFeed()     │
                           │    保存 ETag/LastMod│
                           │  - UpdateFeed()     │
                           │    更新缓存字段      │
                           │  - FeedByID()       │
                           │    读取缓存字段      │
                           └─────────────────────┘
                                      │
                                      ▼
                           ┌─────────────────────┐
                           │    数据库 (feeds)    │
                           │                     │
                           │  - etag_header      │
                           │  - last_modified_header│
                           │  - next_check_at    │
                           └─────────────────────┘
```

## 八、关键设计要点

### 8.1 ETag 优先原则

根据 RFC 9110 第 8.8.1 节，当响应中同时存在 ETag 和 Last-Modified 时，ETag 具有更高的优先级。Miniflux 在 `IsModified()` 方法中严格遵循这一原则。

### 8.2 Expires: 0 的特殊处理

当响应头 `Expires` 值为 `"0"` 时，Miniflux 认为服务器明确表示不希望被缓存，因此忽略 ETag 和 Last-Modified。

### 8.3 304 响应的 Last-Modified 更新

即使返回 304 Not Modified，如果响应头中有新的 Last-Modified 值，Miniflux 也会更新存储的 Last-Modified。这符合 RFC 9111 第 3.2 节和第 4.3.4 节的规范。

### 8.4 多重刷新延迟来源

刷新延迟（即下次检查时间的最小间隔）取以下三者的最大值：
1. RSS feed 的 `<ttl>` 字段值
2. HTTP `Cache-Control: max-age` 指令值
3. HTTP `Expires` 响应头值

### 8.5 强制刷新机制

通过 `forceRefresh` 参数可以绕过缓存协商，强制重新拉取完整内容。常用于用户手动刷新 feed 的场景。

### 8.6 限流处理

当遇到 429 Too Many Requests 时，解析 `Retry-After` 头并调整下次检查时间，避免被服务器封禁。

## 十、缓存失效触发条件

缓存失效是指系统决定需要重新拉取 feed 内容（即使 HTTP 缓存协商可能返回 304）。以下是所有触发缓存失效的条件及其代码实现。

### 10.1 Feed TTL 到期

RSS 2.0 规范中的 `<ttl>`（time to live）元素指示 feed 可以缓存多久。

**解析链路：**

1. **XML 解析层** (`internal/reader/rss/rss.go:72-73`)
```go
// TTL is a number of minutes that indicates how long a channel can be cached before refreshing from the source.
TTL string `xml:"rss ttl"`
```

2. **适配器转换层** (`internal/reader/rss/adapter.go:59-64`)
```go
// Get TTL if defined.
if r.rss.Channel.TTL != "" {
    if ttl, err := strconv.Atoi(r.rss.Channel.TTL); err == nil {
        feed.TTL = time.Duration(ttl) * time.Minute
    }
}
```
- 将 XML 中的字符串值转换为整数分钟
- 转换为 `time.Duration` 类型存储在 `feed.TTL` 中
- 解析失败时忽略该字段（TTL 保持 0）

3. **业务逻辑层** (`internal/reader/handler/handler.go:295-300`)
```go
// Use the RSS TTL value, or the Cache-Control or Expires HTTP headers if available.
// Otherwise, we use the default value from the configuration (min interval parameter).
feedTTLValue := updatedFeed.TTL
cacheControlMaxAgeValue := responseHandler.CacheControlMaxAge()
expiresValue := responseHandler.Expires()
refreshDelay := max(feedTTLValue, cacheControlMaxAgeValue, expiresValue)
```
- 取 RSS TTL、`Cache-Control: max-age`、`Expires` 三者的最大值作为刷新延迟

### 10.2 HTTP 缓存头过期

#### 10.2.1 Cache-Control: max-age

`internal/reader/fetcher/response_handler.go:68-80`
```go
func (r *ResponseHandler) CacheControlMaxAge() time.Duration {
    cacheControlHeaderValue := r.httpResponse.Header.Get("Cache-Control")
    if cacheControlHeaderValue != "" {
        for directive := range strings.SplitSeq(cacheControlHeaderValue, ",") {
            if after, ok := strings.CutPrefix(strings.TrimSpace(directive), "max-age="); ok {
                if maxAge, err := strconv.Atoi(after); err == nil {
                    return time.Duration(maxAge) * time.Second
                }
            }
        }
    }
    return 0
}
```
- 解析 `Cache-Control` 响应头中的 `max-age` 指令
- 单位为秒，转换为 `time.Duration`

#### 10.2.2 Expires 头

`internal/reader/fetcher/response_handler.go:56-66`
```go
func (r *ResponseHandler) Expires() time.Duration {
    expiresHeaderValue := r.httpResponse.Header.Get("Expires")
    if expiresHeaderValue != "" {
        t, err := time.Parse(time.RFC1123, expiresHeaderValue)
        if err == nil {
            return time.Until(t).Truncate(time.Minute) + time.Minute
        }
    }
    return 0
}
```
- 解析 RFC1123 格式的过期时间
- 计算当前时间到过期时间的差值，向上取整到分钟

### 10.3 NextCheckAt 调度过期

`next_check_at` 是数据库中记录的下次检查时间，是最主要的缓存失效触发点。

**调度查询条件** (`internal/storage/batch.go:56-59`)
```go
func (b *batchBuilder) WithNextCheckExpired() *batchBuilder {
    b.conditions = append(b.conditions, "next_check_at < now()")
    return b
}
```
- SQL 查询条件：`next_check_at < now()`
- 只有下次检查时间已过的 feed 才会被加入刷新批次

**NextCheckAt 计算** (`internal/model/feed.go:121-149`)
```go
func (f *Feed) ScheduleNextCheck(weeklyCount int, refreshDelay time.Duration) time.Duration {
    // 1. 根据调度策略确定基础间隔
    interval := config.Opts.SchedulerRoundRobinMinInterval()
    if config.Opts.PollingScheduler() == SchedulerEntryFrequency {
        // 根据条目频率动态调整
        if weeklyCount <= 0 {
            interval = config.Opts.SchedulerEntryFrequencyMaxInterval()
        } else {
            interval = (7 * 24 * time.Hour) / time.Duration(weeklyCount*config.Opts.SchedulerEntryFrequencyFactor())
            interval = min(interval, config.Opts.SchedulerEntryFrequencyMaxInterval())
            interval = max(interval, config.Opts.SchedulerEntryFrequencyMinInterval())
        }
    }

    // 2. 使用 HTTP 缓存头或 RSS TTL 调整间隔（取较大值）
    interval = max(interval, refreshDelay)

    // 3. 限制最大间隔（防止配置错误的 feed）
    interval = min(interval, config.Opts.Scheduler...MaxInterval())

    f.NextCheckAt = time.Now().Add(interval)
    return interval
}
```

**调度策略：**

| 调度策略 | 说明 |
|---------|------|
| `round_robin` | 固定间隔，受 `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL` 和 `SCHEDULER_ROUND_ROBIN_MAX_INTERVAL` 限制 |
| `entry_frequency` | 根据过去一周的条目更新频率动态调整间隔，更新越频繁的 feed 检查越频繁 |

### 10.4 强制刷新（Force Refresh）

`forceRefresh` 参数可以绕过所有缓存机制，强制重新拉取完整内容。

**代码位置** (`internal/reader/handler/handler.go:235-240`)
```go
ignoreHTTPCache := originalFeed.IgnoreHTTPCache || forceRefresh
if !ignoreHTTPCache {
    requestBuilder = requestBuilder.
        WithETag(originalFeed.EtagHeader).
        WithLastModified(originalFeed.LastModifiedHeader)
}
```

当 `forceRefresh = true` 时：
1. 不设置 `If-None-Match` 和 `If-Modified-Since` 请求头
2. 跳过 `IsModified()` 判断，直接认为内容已修改
3. 读取完整响应体并重新解析
4. 更新 ETag 和 Last-Modified

### 10.5 用户操作触发刷新

#### 10.5.1 Web UI 操作

**单个 feed 刷新** (`internal/ui/feed_refresh.go:18-31`)
```go
func (h *handler) refreshFeed(w http.ResponseWriter, r *http.Request) {
    feedID := request.RouteInt64Param(r, "feedID")
    forceRefresh := request.QueryBoolParam(r, "forceRefresh", false)
    if localizedError := feedHandler.RefreshFeed(h.store, request.UserID(r), feedID, forceRefresh); localizedError != nil {
        // ... 错误处理
    }
    response.HTMLRedirect(w, r, h.routePath("/feed/%d/entries", feedID))
}
```
- 支持通过 `?forceRefresh=true` 查询参数强制刷新

**全部 feed 刷新** (`internal/ui/feed_refresh.go:33-68`)
```go
func (h *handler) refreshAllFeeds(w http.ResponseWriter, r *http.Request) {
    sess := request.WebSession(r)
    printer := locale.NewPrinter(sess.Language())

    // 防止频繁刷新：检查距离上次强制刷新的时间间隔
    if time.Since(sess.LastForceRefresh()) < config.Opts.ForceRefreshInterval() {
        interval := int(config.Opts.ForceRefreshInterval().Minutes())
        sess.SetErrorMessage(printer.Plural("alert.too_many_feeds_refresh", interval, interval))
    } else {
        userID := request.UserID(r)
        jobs, err := h.store.NewBatchBuilder().
            WithoutDisabledFeeds().
            WithUserID(userID).
            WithLimitPerHost(config.Opts.PollingLimitPerHost()).
            FetchJobs()
        // ... 异步推送到 worker 池
        go h.pool.Push(jobs)
        sess.MarkForceRefreshed()
    }
    response.HTMLRedirect(w, r, h.routePath("/feeds"))
}
```

**防刷机制** (`internal/model/web_session.go:183-267`)
```go
// LastForceRefresh returns the last force refresh timestamp, or zero time if unset.
func (s *WebSession) LastForceRefresh() time.Time {
    if s.state.LastForceRefreshAt != nil {
        return *s.state.LastForceRefreshAt
    }
    return time.Time{}
}

// MarkForceRefreshed records the current time as the last force refresh.
func (s *WebSession) MarkForceRefreshed() {
    s.dirty = true
    now := time.Now().UTC()
    s.state.LastForceRefreshAt = &now
}
```
- `FORCE_REFRESH_INTERVAL` 配置项控制最小刷新间隔（默认 1 分钟）
- 防止用户意外或恶意频繁触发刷新

**分类刷新** (`internal/ui/category_refresh.go:27-65`)
- 与全部刷新类似，但限制在特定分类下的 feed

#### 10.5.2 API 操作

**单个 feed 刷新** (`internal/api/feed_handlers.go:54-74`)
```go
func (h *handler) refreshFeedHandler(w http.ResponseWriter, r *http.Request) {
    feedID := request.RouteInt64Param(r, "feedID")
    userID := request.UserID(r)
    if !h.store.FeedExists(userID, feedID) {
        response.JSONNotFound(w, r)
        return
    }
    localizedError := feedHandler.RefreshFeed(h.store, userID, feedID, false)
    // ...
}
```

**全部 feed 刷新** (`internal/api/feed_handlers.go:76-100`)
```go
func (h *handler) refreshAllFeedsHandler(w http.ResponseWriter, r *http.Request) {
    userID := request.UserID(r)
    jobs, err := h.store.NewBatchBuilder().
        WithErrorLimit(config.Opts.PollingParsingErrorLimit()).
        WithoutDisabledFeeds().
        WithNextCheckExpired().
        WithUserID(userID).
        WithLimitPerHost(config.Opts.PollingLimitPerHost()).
        FetchJobs()
    go h.pool.Push(jobs)
    response.NoContent(w, r)
}
```

### 10.6 IgnoreHTTPCache 标志

每个 feed 可以单独设置 `IgnoreHTTPCache` 标志，永久忽略 HTTP 缓存协商。

**数据模型** (`internal/model/feed.go:50`)
```go
IgnoreHTTPCache bool `json:"ignore_http_cache"`
```

**使用位置** (`internal/reader/handler/handler.go:235`)
```go
ignoreHTTPCache := originalFeed.IgnoreHTTPCache || forceRefresh
```

当 `IgnoreHTTPCache = true` 时：
- 每次刷新都不携带缓存协商头
- 总是从源站拉取完整内容
- 但仍然更新数据库中的 ETag 和 Last-Modified 字段（以备将来启用缓存时使用）

### 10.7 缓存失效触发条件总结

| 触发条件 | 触发时机 | 代码位置 |
|---------|---------|----------|
| RSS TTL 到期 | 下次检查时间计算时 | `internal/reader/rss/adapter.go:62` |
| Cache-Control 过期 | 下次检查时间计算时 | `internal/reader/fetcher/response_handler.go:74` |
| Expires 过期 | 下次检查时间计算时 | `internal/reader/fetcher/response_handler.go:62` |
| NextCheckAt 到期 | 调度批次查询时 | `internal/storage/batch.go:57` |
| forceRefresh 参数 | 用户手动刷新时 | `internal/reader/handler/handler.go:235` |
| IgnoreHTTPCache 标志 | 每次刷新时 | `internal/model/feed.go:50` |
| 用户 UI 操作 | 点击刷新按钮时 | `internal/ui/feed_refresh.go:20` |
| API 调用 | 调用刷新接口时 | `internal/api/feed_handlers.go:67` |

## 十一、并发拉取时的缓存读写竞争与锁路径

在多 worker 并发环境下，缓存字段（ETag、Last-Modified、NextCheckAt）的读写可能存在竞争条件。本节分析实际的并发场景和锁机制。

### 11.1 并发场景分析

#### 11.1.1 潜在竞争场景

**场景 1：同一 feed 被多个 worker 同时刷新**

```
  时间线 →
  │
  ├─ Worker A 读取 Feed (Etag="abc123")
  │    └─ 发送 HTTP 请求，携带 If-None-Match: "abc123"
  │
  ├─ Worker B 读取同一 Feed (Etag="abc123")
  │    └─ 发送 HTTP 请求，携带 If-None-Match: "abc123"
  │
  ├─ Worker A 收到 200 响应，新 Etag="xyz789"
  │    └─ 更新数据库 Etag="xyz789"
  │
  └─ Worker B 收到 200 响应（内容与 A 相同），新 Etag="xyz789"
       └─ 更新数据库 Etag="xyz789"（重复更新）
```

**后果：**
- 重复的 HTTP 请求，浪费带宽和服务器资源
- 重复解析相同的 feed 内容
- 但最终结果是一致的，因为两者都会更新到相同的新 ETag 值

**场景 2：缓存字段的读写竞态**

```
  Worker A: 读取 EtagHeader = "v1"
  Worker B: 读取 EtagHeader = "v1"
  Worker A: 收到响应，更新 EtagHeader = "v2" （写入数据库）
  Worker B: 基于旧的 "v1" 发送请求（本可以用 "v2"）
```

**后果：**
- Worker B 的请求可能获得 200 而不是 304
- 浪费带宽，但不会导致数据不一致

### 11.2 实际的锁路径

#### 11.2.1 应用层：无显式锁

Miniflux 在 feed 刷新流程中**没有使用应用级别的锁**（如 `sync.Mutex` 或分布式锁）。

**Worker Pool 实现** (`internal/worker/pool.go:13-45`)
```go
type Pool struct {
    queue chan model.Job
    wg    sync.WaitGroup
}

func NewPool(store *storage.Storage, nbWorkers int) *Pool {
    workerPool := &Pool{
        queue: make(chan model.Job),
    }

    for i := range nbWorkers {
        workerPool.wg.Add(1)
        worker := &worker{id: i, store: store}
        go worker.Run(workerPool.queue, &workerPool.wg)
    }

    return workerPool
}
```

- 使用无缓冲 channel 作为任务队列
- `sync.WaitGroup` 仅用于管理 worker 生命周期，不用于保护 feed 数据

**Worker 实现** (`internal/worker/worker.go:24-49`)
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

- 每个 worker 独立调用 `RefreshFeed`
- 没有检查其他 worker 是否正在处理同一个 feed

#### 11.2.2 数据库层：无行锁（Feed 表）

`FetchJobs` 查询没有使用 `SELECT ... FOR UPDATE` 或 `SKIP LOCKED` 来锁定行。

**批次查询** (`internal/storage/batch.go:74-137`)
```go
func (b *batchBuilder) FetchJobs() (model.JobList, error) {
    query := `SELECT id, user_id, feed_url FROM feeds`

    if len(b.conditions) > 0 {
        query += " WHERE " + strings.Join(b.conditions, " AND ")
    }

    query += " ORDER BY next_check_at ASC"

    if b.batchSize > 0 {
        query += " LIMIT " + strconv.Itoa(b.batchSize)
    }

    rows, err := b.db.Query(query, b.args...)
    // ...
}
```

- 普通 `SELECT` 查询，无 `FOR UPDATE` 子句
- 无法防止多个调度批次获取同一个 feed

#### 11.2.3 数据库层：有行锁（Entry 表）

虽然 feed 表没有行锁，但在条目归档功能中使用了 `FOR UPDATE SKIP LOCKED`。

**条目归档** (`internal/storage/entry.go:368-389`)
```sql
WITH to_delete AS (
    SELECT id, feed_id, hash
    FROM entries
    WHERE
        status=$1 AND
        starred is false AND
        share_code='' AND
        created_at < now() - $2::interval
    ORDER BY created_at ASC
    FOR UPDATE SKIP LOCKED
    LIMIT $3
), deleted AS (
    DELETE FROM entries
    USING to_delete
    WHERE entries.id = to_delete.id
    RETURNING entries.feed_id, entries.hash
)
INSERT INTO entry_tombstones (feed_id, hash)
SELECT feed_id, hash FROM deleted WHERE hash <> ''
ON CONFLICT (feed_id, hash) DO NOTHING
```

- `FOR UPDATE SKIP LOCKED` 确保多个并发归档任务不会处理相同的条目
- 已被其他事务锁定的行会被跳过（`SKIP LOCKED`）

> **注意**：这种行锁机制**没有应用**到 feed 刷新流程中。

#### 11.2.4 隐性并发控制：NextCheckAt 更新

虽然没有显式锁，但 `NextCheckAt` 的更新提供了一定程度的并发控制。

**刷新流程开始时** (`internal/reader/handler/handler.go:220-221`)
```go
originalFeed.CheckedNow()
originalFeed.ScheduleNextCheck(weeklyEntryCount, time.Duration(0))
```
- 在刷新流程**开始时**就更新 `next_check_at`
- 这意味着在刷新完成前，该 feed 不会被下一批次选中（因为 `next_check_at < now()` 不再成立）

**但存在竞态窗口**：
```
  T0: Batch 查询 next_check_at < now() → 选中 Feed X
  T1: Batch 查询（另一个调度周期）→ 可能也选中 Feed X （如果 T0 的更新还未完成）
  T2: Worker A 处理 Feed X，更新 next_check_at = now() + 1h
  T3: Worker B 也处理 Feed X（重复工作）
```

#### 11.2.5 主机级并发限制

`WithLimitPerHost` 提供了主机级别的并发限制，但不是 feed 级别的锁。

**实现** (`internal/storage/batch.go:107-120`)
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
- 限制同一批次中同一主机的 feed 数量
- 防止对单一主机发起过多并发请求
- 但不防止同一 feed 被多个批次选中

### 11.3 并发安全分析

#### 11.3.1 数据一致性保证

即使存在并发刷新，数据最终是一致的：

1. **ETag/Last-Modified 更新**：最后完成的更新会覆盖之前的，但值是相同的（因为响应来自同一服务器）
2. **NextCheckAt 更新**：最后完成的更新会设置一个更晚的时间，这是合理的
3. **条目处理**：`RefreshFeedEntries` 内部会检查条目是否已存在，不会重复创建

#### 11.3.2 资源浪费问题

并发刷新的主要问题是资源浪费，而不是数据不一致：

- **重复 HTTP 请求**：多个 worker 请求同一个 feed URL
- **重复解析**：多个 worker 解析相同的 XML/JSON 内容
- **数据库重复查询**：多次读取和更新相同的 feed 行

#### 11.3.3 实际影响评估

在实践中，并发刷新同一 feed 的概率相对较低：

1. **调度频率**：`POLLING_FREQUENCY` 通常是分钟级别（默认 1 分钟）
2. **批次大小**：`BATCH_SIZE` 限制了每批处理的 feed 数量
3. **NextCheckAt 前置更新**：在刷新开始时就更新了下次检查时间
4. **主机级限制**：减少了同一主机的并发

只有在以下情况才可能发生：
- 批次处理时间超过了调度间隔
- feed 数量巨大，单个 feed 的刷新时间很长
- 用户手动刷新和后台调度同时触发

### 11.4 锁路径总结

| 层级 | 锁机制 | 应用于 Feed 刷新 | 说明 |
|------|--------|----------------|------|
| 应用层 | `sync.Mutex` | ❌ 无 | 无 feed 级别的互斥锁 |
| 应用层 | Channel 队列 | ✅ 部分 | 同一批次内的任务不会重复，但跨批次可能重复 |
| 数据库层 | `SELECT ... FOR UPDATE` | ❌ 无 | `FetchJobs` 没有使用行锁 |
| 数据库层 | `FOR UPDATE SKIP LOCKED` | ❌ 无 | 仅用于条目归档，不用于 feed 刷新 |
| 业务逻辑 | `NextCheckAt` 前置更新 | ✅ 部分 | 减少但不消除跨批次竞态 |
| 业务逻辑 | `ForceRefreshInterval` | ✅ 部分 | 防止用户频繁手动刷新 |
| 业务逻辑 | `LimitPerHost` | ✅ 部分 | 主机级并发限制，非 feed 级别 |

### 11.5 可能的改进方向

如果需要更强的并发控制，可以考虑以下方案：

1. **数据库行锁**：在 `FetchJobs` 中使用 `SELECT ... FOR UPDATE SKIP LOCKED`
2. **应用级锁表**：使用内存 map 或 Redis 记录正在刷新的 feed
3. **乐观锁**：使用 `checked_at` 作为版本号，更新时检查版本是否匹配
4. **唯一约束**：利用数据库唯一索引防止重复条目（已在 entry 表实现）

## 十二、相关文件清单

| 文件路径 | 作用 |
|----------|------|
| `internal/model/feed.go` | Feed 数据模型，包含缓存字段和调度逻辑 |
| `internal/model/job.go` | Job 数据模型，用于刷新任务队列 |
| `internal/model/web_session.go` | Web 会话管理，包含强制刷新防刷机制 |
| `internal/reader/fetcher/request_builder.go` | HTTP 请求构建，设置缓存协商头 |
| `internal/reader/fetcher/response_handler.go` | HTTP 响应处理，解析缓存头、判断修改 |
| `internal/reader/handler/handler.go` | Feed 刷新业务逻辑 |
| `internal/reader/rss/rss.go` | RSS XML 结构定义，包含 TTL 字段 |
| `internal/reader/rss/adapter.go` | RSS 适配器，TTL 字段解析转换 |
| `internal/reader/processor/processor.go` | Feed 条目处理逻辑 |
| `internal/storage/feed.go` | Feed 数据持久化 |
| `internal/storage/batch.go` | 批次构建，生成刷新任务 |
| `internal/storage/entry.go` | 条目数据操作，包含 FOR UPDATE SKIP LOCKED 示例 |
| `internal/database/migrations.go` | 数据库表结构定义 |
| `internal/cli/refresh_feeds.go` | CLI 批量刷新调度 |
| `internal/cli/scheduler.go` | 后台调度器，定时触发刷新 |
| `internal/worker/pool.go` | Worker 池管理 |
| `internal/worker/worker.go` | Worker 执行逻辑 |
| `internal/ui/feed_refresh.go` | Web UI 刷新接口 |
| `internal/ui/category_refresh.go` | Web UI 分类刷新接口 |
| `internal/api/feed_handlers.go` | API 刷新接口 |
| `internal/config/options.go` | 配置选项，包含调度和刷新相关配置 |
