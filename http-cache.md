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

## 十二、4xx/5xx 错误响应的缓存策略与错误雪崩避免

当 feed 拉取遇到客户端错误（4xx）或服务器错误（5xx）时，Miniflux 实现了一套完整的错误处理和退避策略，以避免错误雪崩。

### 12.1 错误分类与处理流程

#### 12.1.1 错误检测层次

`RefreshFeed` 函数的错误检测按以下顺序逐层进行（`internal/reader/handler/handler.go:242-292`）：

```
1. 连接层错误（clientErr）
   ├─ TLS 错误（证书问题）
   ├─ 网络错误（连接失败、DNS 解析失败）
   ├─ 超时错误
   └─ EOF（空响应）
   
2. Cloudflare 挑战检测（特殊 403）
   
3. HTTP 状态码错误
   ├─ 401 Unauthorized
   ├─ 403 Forbidden（非 Cloudflare 挑战）
   ├─ 429 Too Many Requests（限流，特殊处理）
   ├─ 404 / 410 资源不存在
   ├─ 500 内部服务器错误
   ├─ 502 Bad Gateway
   ├─ 503 Service Unavailable
   ├─ 504 Gateway Timeout
   └─ 其他 >= 400 状态码
   
4. 响应体错误
   ├─ Content-Length = 0（非 304）
   ├─ 响应体过大（超过 HTTP_CLIENT_MAX_BODY_SIZE）
   ├─ 响应体读取失败
   └─ 空响应体
   
5. 解析错误
   ├─ 格式检测失败（ErrFeedFormatNotDetected）
   └─ XML/JSON 解析异常
   
6. 数据一致性错误
   ├─ 重复 Feed URL
   └─ 数据库操作错误
```

#### 12.1.2 LocalizedError 详细分类

`internal/reader/fetcher/response_handler.go:175-229` 中 `LocalizedError()` 方法负责将底层错误映射为用户友好的本地化错误：

```go
func (r *ResponseHandler) LocalizedError() *locale.LocalizedErrorWrapper {
    // 第一层：客户端/网络错误（clientErr != nil）
    if r.clientErr != nil {
        switch {
        case isSSLError(r.clientErr):        // → error.tls_error
        case isNetworkError(r.clientErr):    // → error.network_operation
        case os.IsTimeout(r.clientErr):      // → error.network_timeout
        case errors.Is(r.clientErr, io.EOF): // → error.http_empty_response
        default:                             // → error.http_client_error
        }
    }

    // 第二层：Cloudflare 挑战（CAPTCHA）
    if r.isCloudflareChallenge() {            // → error.http_cloudflare_challenge
        // ...
    }

    // 第三层：HTTP 状态码分类
    switch r.httpResponse.StatusCode {
    case 401: // → error.http_not_authorized
    case 403: // → error.http_forbidden（排除 Cloudflare 挑战后）
    case 429: // → error.http_too_many_requests
    case 404, 410: // → error.http_resource_not_found
    case 500: // → error.http_internal_server_error
    case 502: // → error.http_bad_gateway
    case 503: // → error.http_service_unavailable
    case 504: // → error.http_gateway_timeout
    }

    // 第四层：兜底错误
    if r.httpResponse.StatusCode >= 400 {
        // → error.http_unexpected_status_code
    }

    // 第五层：空响应体检查（非 304 状态）
    if r.httpResponse.StatusCode != 304 && r.httpResponse.ContentLength == 0 {
        // → error.http_empty_response_body
    }

    return nil
}
```

### 12.2 错误计数与退避机制

#### 12.2.1 错误计数器

`internal/model/feed.go:100-110` 提供了错误计数的基础方法：

```go
// WithTranslatedErrorMessage adds a new error message and increment the error counter.
func (f *Feed) WithTranslatedErrorMessage(message string) {
    f.ParsingErrorCount++
    f.ParsingErrorMsg = message
}

// ResetErrorCounter removes all previous errors.
func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0
    f.ParsingErrorMsg = ""
}
```

**计数规则：**
- 每次发生任何类型的错误（HTTP、解析、数据库等），`ParsingErrorCount++`
- 每次成功刷新（包括 304 Not Modified），调用 `ResetErrorCounter()` 清零
- 持久化到 `feeds.parsing_error_count` 和 `feeds.parsing_error_msg` 列

#### 12.2.2 错误上限过滤（批次层面）

`internal/storage/batch.go:61-69` 中的 `WithErrorLimit` 方法在批次查询时过滤掉错误次数过多的 feed：

```go
func (b *batchBuilder) WithErrorLimit(errorLimit int) *batchBuilder {
    if errorLimit > 0 {
        b.conditions = append(b.conditions, "parsing_error_count < $"+strconv.Itoa(b.argsIndex))
        b.args = append(b.args, errorLimit)
        b.argsIndex++
    }
    return b
}
```

**配置项** `POLLING_PARSING_ERROR_LIMIT`（默认值 3）：
- 当 `parsing_error_count >= 3` 时，该 feed **不再被自动调度刷新**
- 用户仍可通过 UI/API 手动触发刷新（手动刷新不经过此过滤）
- 设为 0 表示无限制（不过滤任何 feed）

#### 12.2.3 429 限流特殊处理（响应层面）

`internal/reader/handler/handler.go:245-255` 中对 429 Too Many Requests 做了独立于常规错误流程的特殊处理：

```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)

    slog.Warn("Feed is rate limited",
        slog.String("feed_url", originalFeed.FeedURL),
        slog.Int("retry_delay_in_seconds", int(retryDelay.Seconds())),
        slog.Int("calculated_next_check_interval_in_minutes", int(calculatedNextCheckInterval.Minutes())),
        slog.Time("new_next_check_at", originalFeed.NextCheckAt),
    )
}
```

**处理逻辑：**
1. 在 `LocalizedError()` 检测之前**先检测限流状态**（即使限流也继续走完错误流程）
2. 解析 `Retry-After` 响应头（支持秒数或 HTTP-date 两种格式）
3. 以 `Retry-After` 值作为 `refreshDelay` 重新计算 `NextCheckAt`
4. 确保不会早于服务器要求的时间重试

> **关键设计**：429 处理在错误检测**之前**执行，即使最终抛出错误并提前返回，`NextCheckAt` 的调整也已经在 `UpdateFeedError` 中被持久化。

### 12.3 错误时的缓存字段策略

#### 12.3.1 错误持久化路径

`internal/reader/handler/handler.go:30-38` 中的 `getTranslatedLocalizedError()` 是所有错误的统一出口：

```go
func getTranslatedLocalizedError(store *storage.Storage, userID int64, originalFeed *model.Feed, localizedError *locale.LocalizedErrorWrapper) *locale.LocalizedErrorWrapper {
    user, storeErr := store.UserByID(userID)
    if storeErr != nil {
        return locale.NewLocalizedErrorWrapper(storeErr, "error.database_error", storeErr)
    }
    originalFeed.WithTranslatedErrorMessage(localizedError.Translate(user.Language))
    store.UpdateFeedError(originalFeed)
    return localizedError
}
```

**调用时机**：所有 `return getTranslatedLocalizedError(...)` 的分支。

#### 12.3.2 UpdateFeedError 持久化字段

`internal/storage/feed.go:427-452` 中的 `UpdateFeedError` 只更新**与错误相关的字段**：

```go
func (s *Storage) UpdateFeedError(feed *model.Feed) (err error) {
    query := `
        UPDATE feeds
        SET
            parsing_error_msg=$1,
            parsing_error_count=$2,
            checked_at=$3,
            next_check_at=$4
        WHERE
            id=$5 AND user_id=$6
    `
    _, err = s.db.Exec(query,
        feed.ParsingErrorMsg,    // 新增：错误信息
        feed.ParsingErrorCount,  // 新增：错误次数 +1
        feed.CheckedAt,          // 更新：刷新开始时间
        feed.NextCheckAt,        // 更新：下次检查时间（已调整）
        feed.ID, feed.UserID,
    )
    return nil
}
```

**缓存字段策略（错误时）：**

| 字段 | 错误时是否更新 | 说明 |
|------|--------------|------|
| `etag_header` | ❌ **不更新** | 保留上次成功响应的 ETag |
| `last_modified_header` | ❌ **不更新** | 保留上次成功响应的 Last-Modified |
| `parsing_error_msg` | ✅ 更新 | 记录最新的本地化错误信息 |
| `parsing_error_count` | ✅ 更新 | 计数 +1 |
| `checked_at` | ✅ 更新 | 记录本次检查尝试时间 |
| `next_check_at` | ✅ 更新 | 根据调度策略计算（可能包含 Retry-After） |

#### 12.3.3 缓存字段保留的意义

**为什么错误时不清除 ETag 和 Last-Modified？**

1. **服务器恢复后仍可协商**：当服务器从故障中恢复时，下次请求可以继续使用之前的 ETag/Last-Modified 进行条件请求。如果服务器端内容未变化，仍可获得 304 响应，节省带宽。

2. **渐进式错误恢复**：假设服务器出现 5 分钟故障后恢复：
   - 故障期间：每次请求返回 5xx，ETag 保持不变，`parsing_error_count` 递增
   - 服务恢复后：请求携带原 ETag，内容未变化 → 304，`parsing_error_count` 清零
   - 如果清除了 ETag：恢复后的首次请求必然下载完整内容 → 200，浪费带宽

**考虑场景**：CDN 或源站短暂故障
```
T0: 正常刷新，Etag="v3", error_count=0
T1: 源站故障，返回 502
    → ETag 保持 "v3"，error_count=1，next_check_at = now() + 1h
T2: 源站仍故障，返回 502
    → ETag 保持 "v3"，error_count=2，next_check_at = now() + 1h  
T3: 源站仍故障，返回 502
    → ETag 保持 "v3"，error_count=3 → 达到上限，停止自动调度
T4: 用户手动刷新，源站已恢复
    → 携带 If-None-Match: "v3"，内容未变
    → 304 Not Modified，error_count=0（清零）
```

### 12.4 错误雪崩防护机制总结

错误雪崩是指大量 feed 同时失败导致系统资源耗尽的场景。Miniflux 通过以下多层机制防止雪崩：

#### 12.4.1 多层防线总结

| 防线层级 | 机制 | 作用 | 代码位置 |
|---------|------|------|---------|
| **第一层：调度过滤** | `WithErrorLimit()` | 错误次数超标的 feed 不再进入刷新批次 | `internal/storage/batch.go:61-69` |
| **第二层：主机限流** | `WithLimitPerHost()` | 限制同一主机的并发请求数 | `internal/storage/batch.go:107-120` |
| **第三层：请求超时** | `WithTimeout()` | 单个请求超时限制（默认 20 秒），避免 worker 永久阻塞 | `internal/config/options.go` |
| **第四层：响应体限制** | `ReadBody(maxBodySize)` | 响应体大小限制，避免大响应耗尽内存 | `internal/reader/fetcher/response_handler.go:156-173` |
| **第五层：429 退避** | `IsRateLimited() + ParseRetryDelay()` | 严格遵循服务器的 Retry-After 指示 | `internal/reader/handler/handler.go:245-255` |
| **第六层：退避调度** | `ScheduleNextCheck()` | 下次检查时间按调度间隔指数级（实际线性）推后 | `internal/model/feed.go:121-149` |
| **第七层：错误计数** | `UpdateFeedError()` | 错误计数递增，配合第一层过滤 | `internal/model/feed.go:100-110` |
| **第八层：缓存字段保留** | ETag/Last-Modified 不清除 | 故障恢复后可继续缓存协商 | `internal/storage/feed.go:427-452` |

#### 12.4.2 雪崩场景模拟分析

**场景：某大型 CDN 故障，1000 个 feed 同时失败**

| 时间点 | 事件 | 防护机制 |
|--------|------|---------|
| T0 | 故障开始，大量 feed 进入刷新批次 | 主机级并发限制 → 同一主机的 feed 分批处理 |
| T0+30s | 每个 feed 请求超时或返回 5xx | 超时限制 → worker 在 20s 内释放，不被永久占用 |
| T0+60s | 下一轮调度周期，生成新批次 | 错误计数已递增，部分 feed 被过滤；next_check_at 已推远，不在下次批次 |
| T0+180s | 继续调度 | 大部分 feed 的 error_count >= 3，被 `WithErrorLimit()` 完全过滤，系统负载显著下降 |
| TN | 用户反馈，手动刷新单个 feed | 手动刷新绕过 `WithErrorLimit()` 过滤，正常重试 |
| TN+5min | CDN 恢复，故障解除 | 恢复后的首次请求使用旧 ETag → 304，节省带宽，error_count 清零 |

**无此机制的后果对比：**

| 维度 | 有防护机制 | 无防护机制 |
|------|-----------|-----------|
| 每秒请求数 | 快速收敛到接近 0 | 持续维持高位（所有 feed 每分钟重试） |
| Worker 占用率 | 错误计数达上限后下降 | 持续 100% 占用 |
| 带宽消耗 | 429+ETag 保留 → 最小化 | 5xx 响应 + 完整请求 → 持续浪费 |
| 恢复后首小时带宽 | 304 为主，几乎无额外开销 | 全量下载 1000 个 feed → 带宽峰值 |

### 12.5 Cloudflare 挑战的特殊处理

Cloudflare 的 bot 保护机制会返回 403 状态码，但语义上不同于普通的 403 Forbidden。Miniflux 在 `internal/reader/fetcher/response_handler.go:231-243` 中专门识别这种场景：

```go
func (r *ResponseHandler) isCloudflareChallenge() bool {
    if r.httpResponse == nil {
        return false
    }

    return r.httpResponse.StatusCode == http.StatusForbidden &&
        strings.EqualFold(r.httpResponse.Header.Get("cf-mitigated"), "challenge") &&
        strings.HasPrefix(strings.ToLower(r.ContentType()), "text/html")
}
```

**三要素判定（AND 条件）：**
1. **状态码 403**：Cloudflare 拦截返回的标准状态码
2. **`cf-mitigated: challenge` 响应头**：Cloudflare 官方提供的机器检测信号头（最可靠）
3. **Content-Type 以 `text/html` 开头**：挑战页面是 HTML，不是预期的 RSS/Atom/JSON feed

**检测后的处理：**
- 返回独立的 `error.http_cloudflare_challenge` 错误类型
- UI 层会向用户展示明确提示：网站受 Cloudflare bot 验证保护，Miniflux 无法自动通过
- 但该错误在计数层面与其他错误相同（`parsing_error_count++`），不会被区别对待

## 十三、HTTP 代理与 CDN 中间层的缓存协商语义影响

当 feed 请求经过 HTTP 代理或 CDN 时，缓存协商的语义可能被中间层改写。本节分析 Miniflux 的代理配置及其对缓存协商的实际影响。

### 13.1 代理层级与配置

Miniflux 支持三级代理配置，优先级从高到低（`internal/reader/fetcher/request_builder.go:143-157`）：

```go
switch {
case r.feedProxyURL != "":
    // 第一优先级：单个 feed 专属代理（Feed.ProxyURL）
    clientProxyURL, err = url.Parse(r.feedProxyURL)
case r.useClientProxy && r.clientProxyURL != nil:
    // 第二优先级：应用级全局代理 + Feed.FetchViaProxy 标志
    clientProxyURL = r.clientProxyURL
case r.proxyRotator != nil && r.proxyRotator.HasProxies():
    // 第三优先级：代理轮换池（HTTP_CLIENT_PROXIES）
    clientProxyURL = r.proxyRotator.GetNextProxy()
}
```

#### 13.1.1 三级代理配置详解

| 优先级 | 配置来源 | 配置项/字段 | 作用域 | 适用场景 |
|--------|---------|-----------|--------|---------|
| 1（最高） | Feed 级 | `Feed.ProxyURL` (`proxy_url` JSON 字段) | 单个 feed | 为特定 feed 指定独立代理（绕过封锁） |
| 2 | 混合级 | `HTTP_CLIENT_PROXY` + `Feed.FetchViaProxy` | 标记 feed | 需要代理的 feed（如国外站点）才走全局代理 |
| 3（最低） | 轮换池 | `HTTP_CLIENT_PROXIES` | 所有 feed | 多个代理随机轮换，分布式爬取 |

#### 13.1.2 请求构建中的代理配置

`internal/reader/handler/handler.go:223-233` 在每个 feed 刷新时完整配置代理：

```go
requestBuilder := fetcher.NewRequestBuilder().
    // ...
    WithProxyRotator(proxyrotator.ProxyRotatorInstance).       // 第三优先级
    WithCustomFeedProxyURL(originalFeed.ProxyURL).              // 第一优先级
    WithCustomApplicationProxyURL(config.Opts.HTTPClientProxyURL()). // 第二优先级基础
    UseCustomApplicationProxyURL(originalFeed.FetchViaProxy).   // 第二优先级开关
    // ...
```

### 13.2 HTTP 客户端与中间层交互

#### 13.2.1 http.Client 的缓存语义

**Go `http.Client` 本身不做任何缓存。** 所有缓存协商完全由 Miniflux 代码控制，具体体现在：

1. **请求头由代码显式设置**：`WithETag()` 和 `WithLastModified()` 手动添加 `If-None-Match` 和 `If-Modified-Since`
2. **响应头由代码解析**：`ResponseHandler.ETag()` 和 `LastModified()` 手动读取并存储
3. **304 判断由代码完成**：`IsModified()` 方法手动判断状态码并比较缓存头

**这意味着中间层无法改变缓存协商的逻辑正确性，只能改变最终收到的响应内容。**

#### 13.2.2 透明代理的潜在影响

当用户网络环境存在**透明 HTTP 代理**（并非由 Miniflux 配置，而是网络层面的强制代理，如企业网关、ISP 缓存、运营商劫持等）时，可能出现以下语义改写：

| 中间层行为 | 对缓存协商的影响 | Miniflux 是否有对应处理 |
|-----------|---------------|----------------------|
| **剥离条件请求头**：移除 `If-None-Match`/`If-Modified-Since` | 源站始终返回 200 完整内容，无法获得 304 | ❌ 无特殊处理（Miniflux 会认为内容每次都"修改"，但不会出错） |
| **剥离缓存响应头**：移除 `ETag`/`Last-Modified`/`Cache-Control` | Miniflux 无法保存缓存头，下一次请求无法协商 | ❌ 无特殊处理（退化为无缓存模式，按最小间隔刷新） |
| **伪造 304 响应**：中间层自行决定返回 304 | Miniflux 按正常 304 处理，不会读取新内容 | ⚠️ 风险：中间层判断错误可能导致用户看不到新内容 |
| **篡改 ETag**：中间层重新生成 ETag（如将弱 ETag `W/"abc"` 改为强 ETag `"abc"`） | 字符串比较失败，导致误判为内容修改 | ⚠️ 会增加不必要的完整响应下载 |
| **CDN 边缘缓存过期**：CDN 在源站未修改时也返回 200 | 重复拉取相同内容，浪费带宽 | ✅ 最终通过 ETag 或 Last-Modified 比较可判断内容实际上未修改（如果 CDN 保留了原头） |

### 13.3 Vary 头的处理缺失

**RFC 9110 规范要求**：当响应包含 `Vary` 头时，缓存必须记录请求中对应头的值，后续请求只有在这些头的值都匹配时才能使用缓存。

**Miniflux 的现状**：

1. **Miniflux 作为客户端（请求 feed）时**：
   - ❌ **完全忽略 `Vary` 响应头**
   - `ResponseHandler` 没有 `Vary()` 方法，也不存储任何 Vary 相关信息
   - 缓存键实际上只有 `feed_id`，不区分任何请求头

2. **潜在风险场景**：

   **场景 A：基于 Accept-Encoding 的 Vary**
   ```
   请求 1: Accept-Encoding: gzip → 响应: Vary: Accept-Encoding, ETag: "abc"
   请求 2: Accept-Encoding: identity → 携带 If-None-Match: "abc"
     → 中间层可能错误返回 304（本应返回 200 + 未压缩内容）
     → Miniflux 认为未修改，不会尝试读取不同编码的内容
   ```
   
   **场景 B：基于 User-Agent 的 Vary**（某些 CDN 根据 UA 返回不同内容格式）
   ```
   请求 1: User-Agent: "Miniflux/2.x" → 响应: Vary: User-Agent, ETag: "v1"
   用户修改自定义 User-Agent 后
   请求 2: User-Agent: "MyCustomUA/1.0" → 携带 If-None-Match: "v1"
     → 实际上 CDN 会返回不同内容，但 ETag 可能被认为匹配
     → 或者 CDN 返回新的 ETag "v2"，Miniflux 正确检测到修改
   ```

3. **实际影响评估**：

   对于 feed 拉取场景，Vary 头的实际影响非常有限：
   - **Accept-Encoding**：Miniflux 让 Go `http.Transport` 自动处理压缩（除非 `WithoutCompression()`），相同请求的 Accept-Encoding 是稳定的
   - **User-Agent**：除非用户主动修改 feed 的自定义 UA，否则 UA 不变
   - **Accept**：Miniflux 请求的 Accept 头固定为 `"application/xml,application/atom+xml,..."`，不会变化
   - **Accept-Language**：Miniflux 不发送此头，无影响

   **结论**：Vary 头处理缺失在实际使用中几乎不会造成问题。

### 13.4 CDN 边缘节点的 ETag 一致性问题

当 feed 通过多层 CDN（如 Cloudflare + Fastly + 源站）或多层代理时，可能出现 ETag 不一致问题。

#### 13.4.1 ETag 场景分析

**场景 1：CDN 保留源站 ETag（最理想）**
```
源站 → ETag: "abc123"
  ↓ Cloudflare 边缘节点（原样转发）
Miniflux 收到 → ETag: "abc123"
下次请求 → If-None-Match: "abc123"
  ↓
Cloudflare 校验 → 匹配 → 304 ✓
```

**场景 2：CDN 重新生成 ETag**
```
源站 → ETag: "abc123"
  ↓ Cloudflare 自动压缩（或修改响应体）→ 重新计算 ETag: "xyz789"
Miniflux 收到 → ETag: "xyz789"
下次请求 → If-None-Match: "xyz789"
  ↓
Cloudflare 边缘校验 → 匹配 → 304 ✓（CDN 内部处理，不回源）
  但如果 CDN 边缘缓存失效，需要回源：
  → Cloudflare 将 "xyz789" 转换为源站的 "abc123" → 304 ✓（CDN 做 ETag 映射）
```
✅ 即使 ETag 被重写，只要 CDN 内部维护映射关系，缓存协商仍然有效。

**场景 3：多层代理 ETag 不兼容（最坏情况）**
```
源站 → ETag: "abc123"（强 ETag）
  ↓ 代理 1 转为弱 ETag: W/"abc123"
  ↓ 代理 2 再次转为强 ETag: "def456"
Miniflux 收到 → ETag: "def456"
下次请求 → If-None-Match: "def456"
  ↓
代理 2 缓存命中 → 304 ✓
  但如果代理 2 缓存失效，需要回源到代理 1：
  → "def456" 与代理 1 的 W/"abc123" 不匹配
  → 代理 1 回源，携带源站的 "abc123"
  → 源站返回 200 完整内容（因为代理 1 无法映射 ETag）
  → 完整内容重新经过两层代理 → 浪费带宽
```
⚠️ 但数据仍然**一致**，只是退化为完整下载模式。

#### 13.4.2 Miniflux 的 ETag 比较策略

`IsModified()` 方法使用**简单字符串相等比较**（`internal/reader/fetcher/response_handler.go:102-116`）：

```go
if r.ETag() != "" {
    return r.ETag() != lastEtagValue
}
```

**没有处理的 RFC 规范细节：**

1. **强弱 ETag 区分**：RFC 9110 第 8.8.1.1 节规定弱 ETag（`W/"..."`）只能用于 `If-None-Match` 的缓存协商，不能用于 `If-Match`。Miniflux 只使用 `If-None-Match`，因此即使是弱 ETag 也能正确工作。

2. **多个 ETag 值**：`If-None-Match` 可以包含多个 ETag 值，用逗号分隔。但 Miniflux 每次只发送一个 ETag，且只存储一个，这符合单一资源缓存的场景。

3. **`*` 通配符**：`If-None-Match: *` 表示"只要资源存在就算匹配"。Miniflux 不使用此功能。

**实际上，简单字符串比较对于 feed 场景是足够的**，因为：
- Miniflux 是 HTTP/1.1 客户端，ETag 的往返路径只要一致就能工作
- 即使中间层对 ETag 做了转换，只要保持来回一致，字符串比较就成立
- 不一致时最坏情况只是退化为全量下载，不会数据错误

### 13.5 代理级缓存（HTTP_CLIENT_PROXY 指向缓存代理）

当用户配置 Miniflux 使用本地缓存代理（如 Squid、Varnish、Nginx proxy_cache 等）时，会形成两级缓存：

```
┌─────────────┐    ETag/Last-Modified     ┌─────────────┐    If-None-Match      ┌──────────┐
│  Miniflux   │ ────────────────────────▶ │  本地缓存   │ ────────────────────▶ │  源站    │
│  客户端缓存  │ ◀──────────────────────── │  代理服务器 │ ◀──────────────────── │  服务器  │
│ (数据库存储) │    304 / 200 + 新ETag     │  (内存/磁盘) │    304 / 200 + 新ETag │          │
└─────────────┘                           └─────────────┘                        └──────────┘
```

**两级缓存的协作语义：**

| 场景 | Miniflux 判断 | 本地代理行为 | 最终效果 |
|------|-------------|-----------|---------|
| Miniflux 有有效 ETag，代理也缓存了 | 发送 If-None-Match → 304 | 代理命中本地缓存，直接返回 304，不回源 | ⚡ 最快：完全本地 |
| Miniflux 有有效 ETag，代理缓存过期 | 发送 If-None-Match | 代理向源站转发条件请求，源站 304 → 代理更新缓存 → 返回 304 | ✅ 节省带宽 |
| Miniflux 无 ETag，代理缓存有效 | 发送无条件请求 | 代理直接返回缓存 200，携带原缓存头 | ✅ 节省源站带宽，但 Miniflux 需处理完整内容 |
| 两者都无缓存 | 发送无条件请求 | 代理向源站转发 → 200 → 双方都更新缓存 | 📥 正常全量下载 |

**潜在语义分歧**：当本地缓存代理**剥离了条件请求头**直接返回缓存内容时：
- Miniflux 会收到 200 响应（而非 304），即使内容未变
- 通过 `IsModified()` 比较新响应的 ETag 与存储值
- 如果代理保留了源站的 ETag，`r.ETag() == lastEtagValue` → `IsModified()` 返回 false，**语义上等价于 304**
- Miniflux **不会处理条目**，直接进入 304 分支 ✅

> **关键代码验证** (`internal/reader/handler/handler.go:272`)：
> ```go
> if ignoreHTTPCache || responseHandler.IsModified(originalFeed.EtagHeader, originalFeed.LastModifiedHeader) {
>     // 处理内容变更
> } else {
>     // 未修改分支（304 或 ETag/Last-Modified 未变的 200）
> }
> ```
> 
> 即使代理剥离开条件请求头导致收到 200，只要 ETag 相同，`IsModified()` 会返回 false，逻辑正确。

### 13.6 代理与缓存协商语义总结

| 场景 | 缓存协商语义是否被改写 | 数据一致性 | 带宽效率 | 代码适配 |
|------|---------------------|-----------|---------|---------|
| 无代理，直连源站 | ❌ 不改变 | ✅ 完全一致 | ✅ 最优 | ✅ 标准处理 |
| Feed 专属代理（ProxyURL） | ⚠️ 取决于代理实现 | ✅ 一致 | ⚠️ 取决于代理 | ✅ 正常工作 |
| 全局代理（FetchViaProxy） | ⚠️ 取决于代理实现 | ✅ 一致 | ⚠️ 取决于代理 | ✅ 正常工作 |
| 代理轮换池（ProxyRotator） | ⚠️⚠️ 可能不一致 | ✅ 一致 | ❌ 可能较差（每次不同 IP，CDN 边缘未命中） | ⚠️ 可能降低缓存命中率 |
| 透明代理（非 Miniflux 配置） | ⚠️⚠️ 完全不可控 | ✅ 一致 | ❌ 可能较差（可能剥离缓存头） | ❌ 无处理 |
| 反向 CDN（源站自行配置） | ⚠️ CDN 内部 ETag 映射 | ✅ 一致 | ✅ 通常良好（CDN 会保留缓存头语义） | ✅ 正常工作 |
| 本地缓存代理（多级缓存） | ⚠️ 可能收到 200 而非 304 | ✅ 一致 | ✅ 良好（`IsModified()` 兜底） | ✅ ETag 比较提供等价语义 |
| Cloudflare 挑战（bot 保护） | ❌ 完全不缓存 | ✅ N/A（错误） | ❌ 每次都是 403 错误页 | ✅ 专门识别并提示用户 |

**核心设计思想**：
- Miniflux 的缓存协商逻辑不依赖 Go `http.Client` 的内置缓存（实际上没有）
- 所有缓存判断在 Miniflux 代码层完成，语义可控
- 即使中间层改变了响应状态码（如 200 → 304 或 304 → 200），通过 ETag/Last-Modified 的**事后比较**仍能维持正确的业务语义
- 最坏情况是**效率降低**（多下载完整内容），但不会出现**数据错误**（漏看新内容或显示旧内容）

## 十四、高并发下 Thundering Herd（惊群效应）防护

Thundering Herd（惊群效应）是指大量 feed 同时到期需要刷新时，在同一时刻触发大规模并发请求，导致：
- 内部 worker 池瞬间打满，正常用户请求被阻塞
- 对源站/CDN 发起瞬时流量洪峰，可能触发 429 限流或封禁
- 数据库连接池耗尽，读写事务阻塞

### 14.1 惊群效应的触发场景

#### 14.1.1 典型触发条件

```
场景 1：大量 feed 使用相同的 min_interval（如默认 1 小时）
  → 在某一整点时刻，成百上千个 feed 的 next_check_at 同时到期
  → 同一批次中集中出现大量相同主机的 feed

场景 2：CDN/源站从故障中恢复
  → 之前因 4xx/5xx 被 WithErrorLimit 过滤的 feed
  → 用户手动点击"全部刷新"按钮
  → 所有 feed 的 next_check_at 被重置为 now()（ResetNextCheckAt）
  → 瞬间产生海量待处理任务

场景 3：应用重启后首次调度
  → 重启前大量 feed 的 next_check_at 已经过期
  → 调度器启动后第一批次包含最多的待刷新 feed
  → 形成"冷启动峰值"

场景 4：热门内容源批量更新
  → 某新闻平台、博客聚合站在固定时间发布内容
  → 订阅该站的所有 feed 同时检测到内容修改
  → 虽然 HTTP 层面各自独立，但业务层（条目标识、集成推送）同时处理
```

### 14.2 Miniflux 的实际防护机制

Miniflux **没有专门的惊群效应防护模块**（如显式 jitter、sliding window、漏桶/令牌桶等），但通过多种设计的组合效应实现了一定程度的缓解。

#### 14.2.1 防线一：批次大小限制

**机制**：`BATCH_SIZE` 配置项（默认值视部署而定）限制每个调度周期内进入 worker 池的任务数上限。

**代码位置** (`internal/storage/batch.go:61-69, 134`)：
```go
func (b *batchBuilder) WithBatchSize(batchSize int) *batchBuilder {
    if batchSize > 0 {
        b.batchSize = batchSize
    }
    return b
}

// FetchJobs 中：
query += " ORDER BY next_check_at ASC"
if b.batchSize > 0 {
    query += " LIMIT " + strconv.Itoa(b.batchSize)
}
```

**防护效果**：
- 即使有 10,000 个 feed 同时到期，每个调度周期也只取前 `BATCH_SIZE` 个进入队列
- 剩余 feed 顺延到下个周期（`POLLING_FREQUENCY` 间隔后）
- 相当于一个天然的**固定窗口限流**

**局限性**：
- 批次内的 feed 仍然在同一时刻被处理
- 不能缓解批次内部的峰值

#### 14.2.2 防线二：NextCheckAt 前置更新 + 排序

**机制**：
1. **刷新开始时就更新** (`internal/reader/handler/handler.go:220-221`)：
```go
originalFeed.CheckedNow()
originalFeed.ScheduleNextCheck(weeklyEntryCount, time.Duration(0))
```
- 在发起 HTTP 请求前就将 `next_check_at` 推远到下次
- 即使当前刷新还未完成，下一批次查询也不会重复选中此 feed
- 避免了批次间的任务重复

2. **按 next_check_at 升序排列** (`internal/storage/batch.go:134`)：
```go
query += " ORDER BY next_check_at ASC"
```
- 最紧急（过期最久）的 feed 优先被处理
- 避免"饿死效应"——某些 feed 永远排不到

#### 14.2.3 防线三：主机级并发限制

**机制**：`POLLING_LIMIT_PER_HOST`（默认 5）限制同一批次中相同主机（域名）的 feed 数量。

**代码位置** (`internal/storage/batch.go:107-120`)：
```go
hosts := make(map[string]int)
for rows.Next() {
    // ...
    if b.limitPerHost > 0 {
        feedHostname := urllib.Domain(job.FeedURL)
        if hosts[feedHostname] >= b.limitPerHost {
            nbSkippedFeeds++
            continue
        }
        hosts[feedHostname]++
    }
    jobs = append(jobs, job)
}
```

**防护效果**：
- 假设 1,000 个 feed 都来自 `blogger.google.com`
- 同一批次中只处理 5 个，其余 995 个被跳过（顺延到后续批次）
- **保护了源站**：避免对单一域名发起瞬时上百次请求
- **保护了内部 worker**：避免 worker 被同一主机的慢响应阻塞

#### 14.2.4 防线四：Worker 池大小限制

**机制**：`WORKER_POOL_SIZE` 限制同时执行 HTTP 请求的最大 goroutine 数。

**代码位置** (`internal/worker/pool.go:35-43`)：
```go
for i := range nbWorkers {
    workerPool.wg.Add(1)
    worker := &worker{id: i, store: store}
    go worker.Run(workerPool.queue, &workerPool.wg)
}
```

**防护效果**：
- 即使批次有 500 个任务，也只有 `WORKER_POOL_SIZE` 个在同时请求
- 其余任务在 channel 队列中排队，等待空闲 worker
- 本质上是**并发度的硬上限**

#### 14.2.5 防线五：差异化调度间隔（Entry Frequency 策略）

**机制**：`entry_frequency` 调度策略根据每个 feed 的更新频率动态计算间隔，使各 feed 的刷新时间**天然错开**。

**代码位置** (`internal/model/feed.go:126-134`)：
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

**原理**：
- 假设 A 类 feed 每周更新 168 次（每小时 1 次），B 类 feed 每周只更新 1 次
- A 类间隔 ≈ (7×24h) / 168 = 1 小时
- B 类间隔 = MaxInterval（可能是数小时甚至一天）
- 不同 feed 的 next_check_at 自然分布在不同时间点，不会集中

#### 14.2.6 防线六：HTTP 超时 + 响应体限制

**机制**：防止单个 feed 长时间占用 worker。

- **`HTTP_CLIENT_TIMEOUT`**（默认 20 秒）：单个请求的最大执行时间
- **`HTTP_CLIENT_MAX_BODY_SIZE`**：限制响应体大小，防止大文件下载阻塞 worker

**效果**：即使惊群发生，每个 worker 的占用时间有上界，队列会逐渐清空。

### 14.3 缺少的防护机制与潜在风险

#### 14.3.1 缺失：调度间隔的随机抖动（Jitter）

**现状**：`ScheduleNextCheck` 完全是确定性的，没有任何随机化：
```go
// internal/model/feed.go:147
f.NextCheckAt = time.Now().Add(interval)
```

**问题**：假设 1,000 个 feed 使用相同的 `min_interval=1h`，且在同一秒内创建或刷新：
```
T0:    所有 feed 的 next_check_at = T0 + 1h = T0+3600s（完全相同）
T0+60m: 1,000 个 feed 同时到期 → 惊群
T0+120m: 又是 1,000 个 feed 同时到期 → 周期性惊群
```
**因为所有 feed 的 interval 相同且起点相同，它们的 next_check_at 永远对齐。**

**业界标准做法**：添加 ±5%~25% 的随机抖动：
```go
// 伪代码
jitter := time.Duration(rand.Int63n(int64(interval / 4))) // ±12.5%
if rand.Intn(2) == 0 {
    interval = interval + jitter
} else {
    interval = interval - jitter
}
f.NextCheckAt = time.Now().Add(interval)
```
添加后，原本对齐的 1,000 个 feed 的 next_check_at 会均匀分散在 52.5 分钟 ~ 67.5 分钟的窗口内，峰值被削平。

#### 14.3.2 缺失：Sliding Window 或 Token Bucket

`BATCH_SIZE` + `POLLING_FREQUENCY` 本质上是**固定窗口限流**，存在"窗口边缘问题"：
```
T=0:   批次 A 取 500 个 feed
T=0.5m: 全部处理完成，系统空闲
T=1m:   批次 B 又取 500 个 feed → 在 T=1m 附近瞬时 1000 请求的"双批次重叠"
```
如果使用 Sliding Window，每分钟的请求数会被更精确地限制。

#### 14.3.3 缺失：手动刷新的全局限流

`ForceRefreshInterval` 是**单会话级**（per-user per-web-session）的，不是全局的：
```go
// internal/model/web_session.go
if time.Since(sess.LastForceRefresh()) < config.Opts.ForceRefreshInterval() {
    // 阻止此用户刷新
}
```
如果 100 个用户同时点击"全部刷新"，即使每个用户都通过了会话级检查，**系统仍然会被 100×N 个任务淹没**。

### 14.4 惊群效应防护总览

| 防线 | 机制 | 作用域 | 缓解程度 | 代码位置 |
|------|------|--------|---------|---------|
| 1 | `BATCH_SIZE` | 批次级 | ⭐⭐⭐ | `internal/storage/batch.go` |
| 2 | `NextCheckAt` 前置更新 + 排序 | 跨批次 | ⭐⭐ | `internal/reader/handler/handler.go:220` |
| 3 | `POLLING_LIMIT_PER_HOST` | 主机级 | ⭐⭐⭐⭐ | `internal/storage/batch.go:107-120` |
| 4 | `WORKER_POOL_SIZE` | 应用全局 | ⭐⭐⭐⭐ | `internal/worker/pool.go` |
| 5 | `entry_frequency` 差异化间隔 | feed 级 | ⭐⭐⭐ | `internal/model/feed.go:126-134` |
| 6 | 超时/响应体限制 | 请求级 | ⭐⭐ | `internal/reader/fetcher/request_builder.go` |
| ❌ 缺失 | 调度间隔随机抖动 (Jitter) | feed 级 | - | N/A |
| ❌ 缺失 | 手动刷新全局限流 | 应用全局 | - | N/A |
| ❌ 缺失 | 分布式锁/单飞模式 (single-flight) | feed 级 | - | N/A |

### 14.5 Single-Flight 模式建议（潜在改进）

对于并发拉取同一 feed（第 11 章分析的竞态场景与惊群效应的交叉点），业界标准做法是 **single-flight 模式**：

**实现思路**：
```go
// 伪代码：应用级 in-flight 跟踪
type FeedRefreshCoordinator struct {
    mu      sync.Mutex
    inFlight map[int64]chan struct{} // feed_id → 正在执行的信号
}

func (c *FeedRefreshCoordinator) Do(feedID int64, refreshFn func()) {
    c.mu.Lock()
    if ch, ok := c.inFlight[feedID]; ok {
        c.mu.Unlock()
        <-ch // 已有刷新在执行，等待其完成
        return
    }
    // 第一个请求，登记并执行
    ch := make(chan struct{})
    c.inFlight[feedID] = ch
    c.mu.Unlock()

    defer func() {
        c.mu.Lock()
        delete(c.inFlight, feedID)
        close(ch) // 通知等待者
        c.mu.Unlock()
    }()

    refreshFn() // 执行实际刷新
}
```

**效果**：10 个 worker 同时触发同一 feed 的刷新，最终只有 1 个真正执行 HTTP 请求，其余 9 个等待其结果。

## 十五、响应内容编码（gzip / br）切换对缓存命中的影响

HTTP 协议支持通过 `Accept-Encoding` 协商内容编码（压缩方式），常见的有 `gzip` 和 `br` (Brotli)。编码切换可能影响缓存协商的有效性。

### 15.1 Miniflux 的编码请求策略

#### 15.1.1 Accept-Encoding 请求头设置

**代码位置** (`internal/reader/fetcher/request_builder.go:257-262`)：
```go
req.Header = r.headers
if r.disableCompression {
    req.Header.Set("Accept-Encoding", "identity")
} else {
    req.Header.Set("Accept-Encoding", "br,gzip")
}
```

**策略说明**：

| 配置 | Accept-Encoding 头 | 说明 |
|------|-------------------|------|
| 默认（压缩启用） | `br,gzip` | 优先使用 Brotli (br)，回退到 gzip |
| `WithoutCompression()` 调用后 | `identity` | 明确要求不压缩（原始字节） |

**编码优先级**：`br` > `gzip` > `identity`。服务器会根据自身支持情况选择最优编码，并在 `Content-Encoding` 响应头中声明实际使用的编码。

#### 15.1.2 Go http.Transport 的自动解压与 Miniflux 的选择

**关键点**：Go 标准库的 `http.Transport` 有一个特性——当请求**没有手动设置 `Accept-Encoding`** 时，Transport 会：
1. 自动添加 `Accept-Encoding: gzip`
2. 对 `Content-Encoding: gzip` 的响应自动解压
3. 自动从响应头中移除 `Content-Length`（因为解压后长度改变）

但 Miniflux **手动设置了 `Accept-Encoding` 头**（`br,gzip` 或 `identity`），这触发了 Go 的**"用户接管编码处理"**语义：
- Transport **不会**自动解压任何内容
- `Content-Length` 和 `Content-Encoding` 头按原样保留
- 解压责任完全交给 Miniflux 代码

### 15.2 响应解码实现

#### 15.2.1 按需解码器

**代码位置** (`internal/reader/fetcher/response_handler.go:133-150`)：
```go
func (r *ResponseHandler) getReader(maxBodySize int64) io.ReadCloser {
    contentEncoding := strings.ToLower(r.httpResponse.Header.Get("Content-Encoding"))
    slog.Debug("Request response",
        slog.String("content_encoding", contentEncoding),
        // ...
    )

    reader := r.httpResponse.Body
    switch contentEncoding {
    case "br":
        reader = NewBrotliReadCloser(reader)
    case "gzip":
        reader = NewGzipReadCloser(reader)
    }
    return http.MaxBytesReader(nil, reader, maxBodySize)
}
```

**解码流程**：
1. 读取 `Content-Encoding` 响应头（小写化）
2. 根据值选择解码器：
   - `br` → Brotli 解压流
   - `gzip` → gzip 解压流
   - 其他（包括空、`identity`、`deflate` 等）→ 不解压，原样读取
3. 外层再包一层 `MaxBytesReader` 限制体积（防止压缩炸弹）

> **注意**：只支持 `br` 和 `gzip` 两种。如果服务器返回 `Content-Encoding: deflate` 或 `zstd`，Miniflux 不会解压，解析器会收到压缩后的二进制数据并**报解析错误**。

### 15.3 缓存协商与编码的交互

**核心问题**：当服务器切换编码（例如第一次返回 gzip，第二次返回 br）时，ETag 和 Last-Modified 是否会变化？这直接影响缓存命中率。

#### 15.3.1 理想情况：ETag 基于原始内容（不区分编码）

**符合 RFC 9110 的正确实现**：
- 服务器对**压缩前的原始内容**计算哈希生成 ETag
- 不同编码（gzip / br / identity）共享同一个 ETag
- `Last-Modified` 也是内容的修改时间，与编码无关

**时间线示例**：
```
T0: 第一次请求
   Accept-Encoding: br,gzip
   ↓
   响应: Content-Encoding: br, ETag: "abc123"
   Miniflux 保存: EtagHeader = "abc123"

T1: 第二次请求（编码协商保持 br,gzip 不变）
   If-None-Match: "abc123"
   Accept-Encoding: br,gzip
   ↓
   服务器比较 ETag → 匹配 → 304 Not Modified ✅
   带宽节省成功
```

**编码切换但 ETag 不变的场景**（服务器调整了内部编码优先级）：
```
T0: Accept-Encoding: br,gzip
   响应: Content-Encoding: gzip, ETag: "abc123"
         （服务器暂时不支持 br，或负载均衡路由到了不同后端）
   Miniflux 保存: EtagHeader = "abc123"

T1: Accept-Encoding: br,gzip
   响应: Content-Encoding: br, ETag: "abc123"
         （这次路由到支持 br 的后端）
   
   IsModified() 比较:
     r.ETag() = "abc123" == lastEtagValue = "abc123"
     → 返回 false（内容未修改）
   
   Miniflux: 走 304 分支，不读取响应体，不处理条目 ✅
   即使编码从 gzip 切到 br，业务逻辑正确！
```

> **结论**：只要服务器正确实现 ETag（基于原始内容），编码切换**完全不影响缓存命中**。这是 Miniflux 设计上的关键假设。

#### 15.3.2 糟糕情况：ETag 基于压缩后的字节（区分编码）

**某些服务器/CDN 的错误实现**：
- 对压缩后的实际传输字节计算哈希
- gzip、br、identity 各自有独立的 ETag 值
- 相同内容不同编码 → 不同 ETag

**时间线示例**：
```
T0: Accept-Encoding: br,gzip
   服务器选 gzip
   ETag: SHA256(gzip_bytes) = "gzip_v1_hash"
   Miniflux 保存: EtagHeader = "gzip_v1_hash"

T1: 服务器配置变更，优先 br
   Accept-Encoding: br,gzip
   If-None-Match: "gzip_v1_hash"
   服务器选 br
   ETag: SHA256(br_bytes) = "br_v1_hash"
   
   304 判定失败（ETag 不匹配）→ 返回 200
   
   IsModified() 比较:
     r.ETag() = "br_v1_hash" != lastEtagValue = "gzip_v1_hash"
     → 返回 true（认为内容修改）
   
   后果:
     - Miniflux 解压响应体（br → 原始内容）
     - parser.ParseFeed 解析（XML 内容实际上与 T0 相同）
     - 所有条目比较后发现没有新内容
     - 浪费了完整内容下载的带宽 ❌
     - 但数据仍然一致（不会出错）
```

#### 15.3.3 Last-Modified 的编码无关性

`Last-Modified` 头的语义是"资源的最后修改时间"，**天然与编码无关**。因此：

- 切换编码不会改变 `Last-Modified` 的值
- 在 ETag 不可用时（某些服务器仅提供 Last-Modified），编码切换不影响缓存协商

**代码验证** (`internal/reader/fetcher/response_handler.go:48-54`)：
```go
func (r *ResponseHandler) LastModified() string {
    if r.httpResponse.Header.Get("Expires") == "0" {
        return ""
    }
    return r.httpResponse.Header.Get("Last-Modified")
}
```
Last-Modified 的读取和比较完全不涉及 Content-Encoding。

### 15.4 Vary: Accept-Encoding 的影响（再讨论）

在第 13.3 节中，我们分析了 Miniflux **完全忽略 Vary 头**。对于内容编码的场景，这实际上可能导致缓存不命中。

**正确但被忽略的语义**：
```
T0: 请求 1
   Accept-Encoding: br,gzip
   → 响应: Vary: Accept-Encoding, ETag: "abc123", Content-Encoding: br
   → 缓存键应为: (URL, Accept-Encoding="br,gzip") → ETag="abc123"

T1: 用户修改了 DisableHTTP2（某些服务器根据 HTTP/2 调整编码支持）
   或 DisableHTTP2 引发行为变化（间接影响传输路径）
   
   新请求:
   Accept-Encoding: br,gzip（实际上没变）
   If-None-Match: "abc123"
   → 304 ✅ （因为 Accept-Encoding 本身没变化）
```

**实际风险场景（极小概率）**：

只有当 Miniflux 在两次请求之间**主动改变了 Accept-Encoding** 时，忽略 Vary 才会有问题。但实际上：
- Miniflux 的 `Accept-Encoding` 在 feed 生命周期内是**固定的**（始终是 `br,gzip` 或 `identity`）
- `disableCompression` 选项在创建 feed 时确定，运行时不改变
- 用户无法通过 UI 修改压缩偏好

**因此，Vary: Accept-Encoding 的缺失在实际部署中几乎不会造成影响。**

### 15.5 压缩编码的完整链路总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                       请求构建 (RequestBuilder)                     │
│                                                                     │
│  disableCompression = false → Accept-Encoding: "br,gzip"           │
│  disableCompression = true  → Accept-Encoding: "identity"          │
│  （注意：手动设置后 Go Transport 不自动解压）                        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       网络传输（可能经过 CDN/代理）                 │
│                                                                     │
│  CDN/源站根据 Accept-Encoding 选择编码:                             │
│  - 支持 br → Content-Encoding: br + Brotli 压缩字节                │
│  - 只支持 gzip → Content-Encoding: gzip + gzip 压缩字节            │
│  - 返回 ETag / Last-Modified / Vary: Accept-Encoding               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       缓存协商判断 (IsModified)                    │
│                                                                     │
│  1. 状态码 == 304 → 未修改（跳过编码处理）                          │
│  2. 否则，比较 ETag:                                                │
│     ├─ 理想服务器 → ETag 不区分编码 → 切换编码也命中 304 ✅         │
│     └─ 糟糕服务器 → ETag 基于压缩字节 → 切换编码可能失配 ❌ → 200   │
│  3. 无 ETag 时比较 Last-Modified → 天然不区分编码 ✅                │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
             ┌─────────────────┴─────────────────┐
             ▼                                   ▼
   内容未修改（304 / ETag 相同）          内容已修改（需要处理）
   - 不读取响应体                        - ReadBody → getReader 解码:
   - 不处理条目                            Content-Encoding: br → Brotli 解压器
   - 仅更新 NextCheckAt / 清零错误计数     Content-Encoding: gzip → gzip 解压器
   - ✅ 带宽最省                           其他 → 原样读取
                                           - parser.ParseFeed 解析 XML/JSON
                                           - 比较条目哈希，只处理更新
                                           - ⚠️ 可能浪费带宽（编码切换误判）
```

### 15.6 编码切换对缓存命中的影响矩阵

| 场景 | ETag 策略 | Accept-Encoding 是否变化 | Last-Modified 是否可用 | 缓存命中结果 | 业务正确性 |
|------|----------|------------------------|---------------------|------------|----------|
| 编码保持不变 | 任何策略 | 否 | 任何 | ✅ 命中 304 | ✅ 正确 |
| gzip ↔ br 切换 | 基于原始内容（RFC 合规） | 否（始终 br,gzip） | 任何 | ✅ ETag 相同 → 未修改 | ✅ 正确 |
| gzip ↔ br 切换 | 基于压缩字节（错误实现） | 否 | 是 | ⚠️ ETag 失配，但 Last-Modified 兜底 | ✅ 正确 |
| gzip ↔ br 切换 | 基于压缩字节（错误实现） | 否 | 否 | ❌ 误判为修改，下载完整内容 | ✅ 数据一致，浪费带宽 |
| 启用 → 禁用压缩（`WithoutCompression`） | 任何策略 | 是（`br,gzip` → `identity`） | 任何 | ❌ Vary 被忽略，但实际上 Feed 级选项不变化 | N/A（不发生） |
| 禁用 → 启用压缩 | 任何策略 | 是（`identity` → `br,gzip`） | 任何 | ❌ 同上 | N/A（不发生） |

### 15.7 潜在改进点

#### 15.7.1 添加编码感知的缓存键（如果需要支持 Vary）

```go
// 伪代码：扩展 feed 表或缓存结构
type FeedCacheKey struct {
    FeedID         int64
    AcceptEncoding string // 记录协商时的 Accept-Encoding 值
    ETag           string
    LastModified   string
}
```

#### 15.7.2 检测误判并记录指标

在解析完内容后，比较条目数量和哈希值与上次是否相同，如果完全相同但 ETag 不同，记录一条警告，提示用户该 feed 的服务器可能存在编码相关的 ETag 问题。

---

## 十六、ETag / Cache Header 与持久化路径的结合

本节从代码层面详细跟踪 ETag、Last-Modified、Cache-Control 等 HTTP 缓存头在"抓取 → 判断 → 写入数据库"全链路中与持久化的结合方式。

### 16.1 三条持久化路径

Miniflux 的 feed 缓存数据通过三条不同的 SQL 路径写入数据库，每条路径更新的字段集合不同：

| 路径 | 函数 | 触发时机 | 更新的缓存相关字段 |
|------|------|---------|-----------------|
| **完整更新** | `UpdateFeed()` | 刷新成功（内容修改或未修改） | `etag_header`, `last_modified_header`, `checked_at`, `next_check_at`, `parsing_error_count=0`, `parsing_error_msg=""` |
| **错误更新** | `UpdateFeedError()` | 刷新失败（任何错误） | `parsing_error_msg`, `parsing_error_count++`, `checked_at`, `next_check_at`（**不更新** etag_header / last_modified_header） |
| **创建写入** | `CreateFeed()` | 首次订阅 feed | `etag_header`, `last_modified_header`（从初始响应获取）|

### 16.2 创建 Feed 时的缓存头初始化

#### 16.2.1 路径一：从 HTTP 响应创建（CreateFeed）

`internal/reader/handler/handler.go:103-192`

```
用户提交订阅 URL
    │
    ▼
1. 构建 HTTP 请求（CreateFeed 不带缓存协商头，因为是首次请求）
    requestBuilder := fetcher.NewRequestBuilder().
        WithUsernameAndPassword(...).
        WithUserAgent(...).
        // 注意：没有 WithETag() / WithLastModified()
        // 首次请求没有已知的缓存值
    
2. 执行请求，获取响应
    responseHandler := fetcher.NewResponseHandler(requestBuilder.ExecuteRequest(feedURL))
    
3. 检查重复
    store.FeedURLExists(userID, responseHandler.EffectiveURL())
    → SQL: SELECT true FROM feeds WHERE user_id=$1 AND feed_url=$2 LIMIT 1
    
4. 从响应头提取缓存值
    subscription.EtagHeader = responseHandler.ETag()           ← 写入
    subscription.LastModifiedHeader = responseHandler.LastModified() ← 写入
    subscription.FeedURL = responseHandler.EffectiveURL()
    
5. 设置初始时间戳
    subscription.CheckedNow()  → CheckedAt = now()
    // ScheduleNextCheck 未在 CreateFeed 中调用
    // NextCheckAt 使用数据库默认值 now()
    
6. 持久化到数据库
    store.CreateFeed(subscription)
    → SQL INSERT 包含 etag_header ($6) 和 last_modified_header ($7)
```

**关键代码** (`internal/reader/handler/handler.go:170-175`)：
```go
subscription.EtagHeader = responseHandler.ETag()
subscription.LastModifiedHeader = responseHandler.LastModified()
subscription.FeedURL = responseHandler.EffectiveURL()
subscription.ProxyURL = feedCreationRequest.ProxyURL
subscription.WithCategoryID(feedCreationRequest.CategoryID)
subscription.CheckedNow()
```

#### 16.2.2 路径二：从订阅发现创建（CreateFeedFromSubscriptionDiscovery）

`internal/reader/handler/handler.go:40-101`

此路径用于 OPML 导入或浏览器扩展推送等场景，**不发送 HTTP 请求**，而是直接使用上游已获取的缓存头：

```go
subscription.EtagHeader = feedCreationRequest.ETag
subscription.LastModifiedHeader = feedCreationRequest.LastModified
```

`FeedCreationRequestFromSubscriptionDiscovery` 结构体携带了上游传递的缓存值：
```go
type FeedCreationRequestFromSubscriptionDiscovery struct {
    Content      io.ReadSeeker
    ETag         string           // ← 上游已获取的 ETag
    LastModified string           // ← 上游已获取的 Last-Modified
    FeedCreationRequest
}
```

### 16.3 刷新 Feed 时的缓存头更新路径

#### 16.3.1 成功刷新 → UpdateFeed

**代码位置** (`internal/reader/handler/handler.go:272-371`)

```
RefreshFeed 开始
    │
    ├─ originalFeed.CheckedNow()  → CheckedAt = now()
    ├─ originalFeed.ScheduleNextCheck(weeklyEntryCount, 0)  → NextCheckAt 推远
    │
    ├─ 构建 HTTP 请求（带缓存协商头）
    │   requestBuilder.WithETag(originalFeed.EtagHeader)
    │   requestBuilder.WithLastModified(originalFeed.LastModifiedHeader)
    │
    ├─ 执行请求，获取响应
    │
    ├─ 判断 IsModified()
    │   │
    │   ├─ 内容已修改 (200)
    │   │   ├─ ReadBody → 解码 (gzip/br)
    │   │   ├─ ParseFeed → 解析 XML/JSON
    │   │   ├─ ScheduleNextCheck(refreshDelay)  → NextCheckAt 基于 TTL/Cache-Control/Expires
    │   │   ├─ ProcessFeedEntries → 处理条目
    │   │   ├─ RefreshFeedEntries → 写入条目
    │   │   │
    │   │   ├─ ★ originalFeed.EtagHeader = responseHandler.ETag()
    │   │   ├─ ★ originalFeed.LastModifiedHeader = responseHandler.LastModified()
    │   │   │
    │   │   └─ originalFeed.ResetErrorCounter()
    │   │       → ParsingErrorCount = 0
    │   │       → ParsingErrorMsg = ""
    │   │
    │   └─ 内容未修改 (304 或 ETag/Last-Modified 相同)
    │       ├─ ScheduleNextCheck 未再调用（使用刷新开始时设置的值）
    │       │
    │       ├─ ★ if responseHandler.LastModified() != "" {
    │       │      originalFeed.LastModifiedHeader = responseHandler.LastModified()
    │       │  }
    │       │  // 注意：304 时不更新 ETag（因为 304 响应通常不携带新 ETag）
    │       │
    │       └─ originalFeed.ResetErrorCounter()
    │           → ParsingErrorCount = 0
    │           → ParsingErrorMsg = ""
    │
    └─ ★ store.UpdateFeed(originalFeed)
        → SQL UPDATE 包含 etag_header=$5, last_modified_header=$6
        → WHERE id=$40 AND user_id=$41
```

**UpdateFeed 的完整字段映射** (`internal/storage/feed.go:329-418`)：

```sql
UPDATE feeds SET
    feed_url=$1,
    site_url=$2,
    title=$3,
    category_id=$4,
    etag_header=$5,              ← 缓存头
    last_modified_header=$6,     ← 缓存头
    checked_at=$7,               ← 时间戳
    parsing_error_msg=$8,        ← 错误信息（成功时为空）
    parsing_error_count=$9,      ← 错误计数（成功时为 0）
    ...
    next_check_at=$22,           ← 调度时间
    ignore_http_cache=$23,       ← 缓存开关
    ...
WHERE id=$40 AND user_id=$41
```

#### 16.3.2 刷新失败 → UpdateFeedError

**代码位置** (`internal/reader/handler/handler.go:30-38` → `internal/storage/feed.go:427-452`)

```
RefreshFeed 遇到错误
    │
    ├─ getTranslatedLocalizedError()
    │   ├─ originalFeed.WithTranslatedErrorMessage(message)
    │   │   → ParsingErrorCount++
    │   │   → ParsingErrorMsg = 翻译后的错误信息
    │   │
    │   └─ store.UpdateFeedError(originalFeed)
    │       → SQL UPDATE 只更新 4 个字段:
    │           parsing_error_msg=$1
    │           parsing_error_count=$2
    │           checked_at=$3
    │           next_check_at=$4
    │       → ★ etag_header 和 last_modified_header 不在 UPDATE 列表中
    │       → 保留上次成功的缓存值
```

**UpdateFeedError 的 SQL** (`internal/storage/feed.go:428-438`)：
```sql
UPDATE feeds SET
    parsing_error_msg=$1,
    parsing_error_count=$2,
    checked_at=$3,
    next_check_at=$4
WHERE id=$5 AND user_id=$6
```

#### 16.3.3 429 限流时的特殊路径

429 处理在 `LocalizedError()` 之前执行，同时影响 `NextCheckAt`：

```
RefreshFeed 遇到 429
    │
    ├─ responseHandler.IsRateLimited() → true
    │   ├─ retryDelay = responseHandler.ParseRetryDelay()
    │   └─ originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
    │       → NextCheckAt = now() + max(基础间隔, retryDelay)
    │
    ├─ responseHandler.LocalizedError() → 返回 429 错误
    │
    └─ getTranslatedLocalizedError()
        ├─ ParsingErrorCount++
        └─ store.UpdateFeedError(originalFeed)
            → NextCheckAt 已包含 Retry-After 延迟
            → etag_header / last_modified_header 保留不变
```

### 16.4 缓存头持久化的完整字段追踪

| 阶段 | etag_header | last_modified_header | checked_at | next_check_at | parsing_error_count | 触发的 SQL |
|------|------------|---------------------|-----------|--------------|-------------------|-----------|
| 首次创建 | 响应的 ETag | 响应的 Last-Modified | now() | now()（默认） | 0 | `INSERT` |
| 刷新 - 200 修改 | 新 ETag | 新 Last-Modified | now() | now()+interval | 0（清零） | `UpdateFeed` |
| 刷新 - 304 未修改 | 保持不变 | 更新（如有新值） | now() | 刷新开始时已设置 | 0（清零） | `UpdateFeed` |
| 刷新 - 4xx/5xx 错误 | 保持不变 | 保持不变 | now() | now()+interval | +1 | `UpdateFeedError` |
| 刷新 - 429 限流 | 保持不变 | 保持不变 | now() | now()+max(interval, Retry-After) | +1 | `UpdateFeedError` |
| 刷新 - 网络错误 | 保持不变 | 保持不变 | now() | 刷新开始时已设置 | +1 | `UpdateFeedError` |

### 16.5 304 与 200 的 ETag/Last-Modified 更新差异

#### 16.5.1 内容修改时（200）

```go
// internal/reader/handler/handler.go:341-342
originalFeed.EtagHeader = responseHandler.ETag()           // 总是更新
originalFeed.LastModifiedHeader = responseHandler.LastModified() // 总是更新
```

200 响应通常携带完整的缓存头，两者都会更新。

#### 16.5.2 内容未修改时（304）

```go
// internal/reader/handler/handler.go:357-361
if responseHandler.LastModified() != "" {
    originalFeed.LastModifiedHeader = responseHandler.LastModified()
}
// 注意：没有对 ETag 做类似的更新
```

**为什么 304 时更新 Last-Modified 但不更新 ETag？**

1. **RFC 9111 第 3.2 节和第 4.3.4 节**：304 响应可以携带更新的缓存头，用于"新鲜度验证"后更新已缓存的响应元数据
2. **Last-Modified 可能更新**：即使内容未变，服务器可能返回更精确的修改时间（如时间戳精度从秒提升到亚秒）
3. **ETag 通常不变**：304 响应中的 ETag 应与请求中的 `If-None-Match` 值相同（服务器只在内容变化时改变 ETag）。但 304 响应**可能不携带 ETag 头**（合法行为），此时 `responseHandler.ETag()` 返回空字符串。如果强行更新，会把有效的 ETag 清空，导致下次请求无法协商

**设计逻辑**：
- Last-Modified 即使被更新，也只是时间精度的变化，不影响"内容是否修改"的判断
- ETag 如果被清空，后果严重——退化为无缓存模式，浪费带宽

### 16.6 数据库约束对缓存数据完整性的保障

#### 16.6.1 feeds 表约束

```sql
-- internal/database/migrations.go:56-72
CREATE TABLE feeds (
    id BIGSERIAL,
    user_id int not null,
    category_id int not null,
    feed_url text not null,
    site_url text not null,
    etag_header text default '',           -- 缓存字段
    last_modified_header text default '',   -- 缓存字段
    parsing_error_count int default 0,
    primary key (id),
    unique (user_id, feed_url),            -- ★ 同一用户不能有重复 URL
    foreign key (user_id) references users(id) on delete cascade,
    foreign key (category_id) references categories(id) on delete cascade
);
```

**关键约束**：`UNIQUE (user_id, feed_url)` — 同一用户下 feed URL 唯一。但**不同用户可以订阅相同的 feed URL**，各自拥有独立的 feed 记录和缓存值。

#### 16.6.2 entries 表约束

```sql
-- internal/database/migrations.go:76-91
CREATE TABLE entries (
    id BIGSERIAL,
    user_id int not null,
    feed_id bigint not null,
    hash text not null,
    ...
    primary key (id),
    unique (feed_id, hash),                -- ★ 同一 feed 下条目哈希唯一
    foreign key (user_id) references users(id) on delete cascade,
    foreign key (feed_id) references feeds(id) on delete cascade
);
```

**关键约束**：`UNIQUE (feed_id, hash)` — 条目去重基于 `(feed_id, hash)` 组合，而不是 `(user_id, hash)`。这意味着同一用户的不同 feed 即使有相同 hash 的条目，也会各自存储。

## 十七、多用户共享同一 Feed 时的缓存数据隔离机制

当多个用户订阅同一个外部 feed URL 时（如 `https://example.com/feed.xml`），Miniflux 为每个用户创建**完全独立的 feed 记录**。本节分析这种设计对缓存数据隔离的影响。

### 17.1 数据模型：每用户一份 Feed 记录

#### 17.1.1 核心设计

```
外部 Feed URL: https://example.com/feed.xml

┌─────────────────────────────────────┐
│            feeds 表                  │
├──────┬─────────┬────────────────────┤
│  id  │ user_id │ feed_url           │ etag_header │ last_modified_header │
├──────┼─────────┼────────────────────┼─────────────┼──────────────────────┤
│  1   │   100   │ https://example... │ "abc123"    │ "Wed, 01 Jun..."     │
│  2   │   200   │ https://example... │ ""          │ ""                   │
│  3   │   300   │ https://example... │ "abc123"    │ "Wed, 01 Jun..."     │
└──────┴─────────┴────────────────────┴─────────────┴──────────────────────┘
```

- 用户 100 和 300 的 ETag 相同（都已成功刷新过）
- 用户 200 的 ETag 为空（刚订阅，尚未首次刷新）
- 三条记录**完全独立**，互不影响

#### 17.1.2 唯一约束的作用域

```sql
UNIQUE (user_id, feed_url)
```

此约束**只保证同一用户下 feed_url 不重复**，不阻止不同用户订阅相同 URL。这意味着：
- 用户 A 订阅 `https://example.com/feed.xml` → feed id=1
- 用户 B 也订阅 `https://example.com/feed.xml` → feed id=2
- 两条记录**独立存储** ETag、Last-Modified、NextCheckAt 等所有字段

### 17.2 缓存隔离的完整影响

#### 17.2.1 ETag / Last-Modified 隔离

每个用户的 feed 记录有**独立的缓存协商状态**：

| 维度 | 隔离效果 | 说明 |
|------|---------|------|
| ETag | ✅ 完全隔离 | 用户 A 的 ETag 可能与用户 B 不同（如果刷新时机不同） |
| Last-Modified | ✅ 完全隔离 | 同上 |
| IgnoreHTTPCache | ✅ 完全隔离 | 用户 A 可以忽略缓存，用户 B 正常使用缓存 |
| NextCheckAt | ✅ 完全隔离 | 用户 A 可能在 T1 刷新，用户 B 可能在 T2 刷新 |
| ParsingErrorCount | ✅ 完全隔离 | 用户 A 的 feed 可能报错，用户 B 正常 |

**场景示例**：

```
T0: 用户 A 和用户 B 都订阅了 https://example.com/feed.xml

T1: 调度器触发用户 A 的 feed 刷新
    → 发送 HTTP 请求，携带 If-None-Match: "abc123"
    → 服务器返回 304 Not Modified
    → 用户 A 的 etag_header 保持 "abc123"

T2: 用户 B 手动点击"强制刷新"
    → 发送 HTTP 请求，不携带缓存头（forceRefresh=true）
    → 服务器返回 200 + 新 ETag: "xyz789"
    → 用户 B 的 etag_header 更新为 "xyz789"

此时:
  用户 A: etag_header = "abc123"
  用户 B: etag_header = "xyz789"
  两者完全独立，互不干扰
```

#### 17.2.2 条目数据隔离

条目通过 `(feed_id, hash)` 唯一约束去重，**不是** `(user_id, hash)`：

```sql
-- internal/storage/entry.go:212-217
func (s *Storage) entryExists(tx *sql.Tx, entry *model.Entry) (bool, error) {
    // Note: This query uses entries_feed_id_hash_key index
    // (filtering on user_id is not necessary).
    err := tx.QueryRow(
        `SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1`,
        entry.FeedID, entry.Hash,
    ).Scan(&result)
}
```

**这意味着**：
- 用户 A 的 feed（id=1）和用户 B 的 feed（id=2）可以有**相同 hash 的条目**
- 各自独立存储，互不干扰
- 同一条文章在两个用户下是完全独立的记录

**数据层面**：
```
┌──────────────────────────────────────────────────┐
│              entries 表                           │
├──────┬─────────┬─────────┬──────────┬────────────┤
│  id  │ user_id │ feed_id │   hash   │   title    │
├──────┼─────────┼─────────┼──────────┼────────────┤
│ 101  │   100   │    1    │ h_sha256 │ Article X  │ ← 用户 A 的条目
│ 102  │   200   │    2    │ h_sha256 │ Article X  │ ← 用户 B 的条目（相同 hash，不同 feed_id）
└──────┴─────────┴─────────┴──────────┴────────────┘
```

#### 17.2.3 条目状态隔离

每个用户的条目有独立的状态（read/unread/removed）：

```
用户 A: Article X → status = "read"     （已读）
用户 B: Article X → status = "unread"   （未读）
```

状态字段存储在 `entries` 表中，与 `user_id` 直接关联。

### 17.3 缓存隔离的代价：重复 HTTP 请求

由于每个用户有独立的 feed 记录，**同一外部 URL 会被多次请求**：

```
调度器批处理:
  → 选中 feed(id=1, user_id=100, feed_url="https://example.com/feed.xml")
  → 选中 feed(id=2, user_id=200, feed_url="https://example.com/feed.xml")

Worker 1 处理 feed#1:
  → HTTP GET https://example.com/feed.xml (If-None-Match: "abc123")
  → 304 Not Modified

Worker 2 处理 feed#2:
  → HTTP GET https://example.com/feed.xml (If-None-Match: "xyz789")
  → 200 OK (因为 ETag 不同)
```

**问题**：
1. 相同 URL 被请求两次（或更多次，取决于订阅用户数）
2. 不同用户的 ETag 可能不一致（如上例），导致一个获得 304、另一个获得 200
3. 200 响应需要完整下载和解析，但内容实际与另一个用户的 304 内容相同

**缓解机制**：`POLLING_LIMIT_PER_HOST` 限制了同一批次中同一主机的 feed 数量，间接限制了重复请求的并发度。但不同批次或不同调度周期仍会产生重复。

### 17.4 缓存隔离的收益：用户级定制

完全隔离的设计也为用户级定制提供了基础：

| 定制维度 | 代码位置 | 说明 |
|---------|---------|------|
| **IgnoreHTTPCache** | `model/feed.go:50` | 用户 A 可忽略缓存强制刷新，用户 B 正常缓存 |
| **UserAgent** | `handler/handler.go:225` | 不同用户可设置不同 UA，可能导致服务器返回不同内容 |
| **Cookie** | `handler/handler.go:226` | 不同用户的 Cookie 不同，可能影响个性化 feed 内容 |
| **Username/Password** | `handler/handler.go:224` | 认证信息不同，可能看到不同的私密 feed |
| **FetchViaProxy** | `model/feed.go:52` | 用户 A 走代理，用户 B 直连，源站可能返回不同内容 |
| **ProxyURL** | `model/feed.go:64` | 每个用户可配置独立代理 |
| **ScraperRules/RewriteRules** | `model/feed.go:38-39` | 条目处理规则不同 |
| **DisableHTTP2** | `model/feed.go:54` | 传输层差异可能影响 CDN 行为 |
| **AllowSelfSignedCertificates** | `model/feed.go:51` | TLS 验证差异可能导致不同的响应链路 |

**关键认识**：由于以上定制维度的存在，即使两个用户订阅了相同的 feed URL，他们实际收到的内容可能**不同**（例如带认证的 feed、个性化推荐 feed、基于 Cookie 的内容过滤等）。因此缓存隔离不是冗余，而是**功能正确性的必要条件**。

### 17.5 调度层面的隔离

#### 17.5.1 批次查询的 user_id 作用域

手动刷新（UI/API）时，批次查询限定在特定用户：

```go
// internal/ui/feed_refresh.go (refreshAllFeeds)
jobs, err := h.store.NewBatchBuilder().
    WithoutDisabledFeeds().
    WithUserID(userID).              // ← 限定当前用户
    WithLimitPerHost(config.Opts.PollingLimitPerHost()).
    FetchJobs()
```

后台调度时，批次查询**不限定用户**（处理所有用户的 feed）：

```go
// internal/cli/scheduler.go (feedScheduler)
jobs, err := store.NewBatchBuilder().
    WithBatchSize(batchSize).
    WithErrorLimit(errorLimit).
    WithoutDisabledFeeds().
    WithNextCheckExpired().
    WithLimitPerHost(limitPerHost).
    // 注意：没有 WithUserID()，处理所有用户
    FetchJobs()
```

#### 17.5.2 刷新操作的 user_id 校验

`RefreshFeed` 函数接收 `userID` 和 `feedID` 参数，并通过 `FeedByID` 查询确保 feed 归属该用户：

```go
// internal/reader/handler/handler.go:202
originalFeed, storeErr := store.FeedByID(userID, feedID)

// internal/storage/feed.go:200-213
func (s *Storage) FeedByID(userID, feedID int64) (*model.Feed, error) {
    feed, err := s.NewFeedQueryBuilder(userID).  // ← user_id 作为查询条件
        WithFeedID(feedID).
        GetFeed()
    // ...
}

// feedQueryBuilder 默认条件:
// conditions: []string{"f.user_id = $1"}
```

**安全保证**：用户 A 无法通过 `RefreshFeed` 访问或修改用户 B 的 feed 缓存数据，因为 SQL 查询始终包含 `user_id = $1` 条件。

#### 17.5.3 UpdateFeed 的 user_id 双重校验

```sql
-- internal/storage/feed.go:373-374
UPDATE feeds SET ...
WHERE id=$40 AND user_id=$41
```

即使应用层传入了错误的 feed 对象，`WHERE user_id=$41` 条件确保不会跨用户更新缓存字段。

### 17.6 隔离模型总结

```
                    外部 Feed URL
                   https://example.com/feed.xml
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
         ┌────────┐   ┌────────┐   ┌────────┐
         │ User A │   │ User B │   │ User C │
         │ feed#1 │   │ feed#2 │   │ feed#3 │
         ├────────┤   ├────────┤   ├────────┤
         │ ETag   │   │ ETag   │   │ ETag   │  ← 各自独立
         │ LastMo │   │ LastMo │   │ LastMo │  ← 各自独立
         │ NextCh │   │ NextCh │   │ NextCh │  ← 各自独立
         │ Errors │   │ Errors │   │ Errors │  ← 各自独立
         │ Cookie │   │ Cookie │   │ Cookie │  ← 可能不同
         │ UA     │   │ UA     │   │ UA     │  ← 可能不同
         │ Proxy  │   │ Proxy  │   │ Proxy  │  ← 可能不同
         ├────────┤   ├────────┤   ├────────┤
         │entries │   │entries │   │entries │  ← 各自独立
         │ A-101  │   │ B-201  │   │ C-301  │
         │ A-102  │   │ B-202  │   │ C-302  │
         └────────┘   └────────┘   └────────┘
              │             │             │
              ▼             ▼             ▼
         同一外部 URL 被请求 3 次（无共享缓存）
```

**隔离的维度**：

| 维度 | 隔离 | 共享 | 说明 |
|------|------|------|------|
| Feed 记录 | ✅ 每用户独立 | ❌ | 每用户一条 feed 记录 |
| ETag / Last-Modified | ✅ 每用户独立 | ❌ | 缓存协商状态独立 |
| NextCheckAt | ✅ 每用户独立 | ❌ | 调度时间独立 |
| HTTP 请求 | ❌ 重复发送 | ❌ | 同一 URL 被多次请求 |
| 响应内容 | ✅ 可能不同 | ❌ | Cookie/UA/认证差异 |
| 条目数据 | ✅ 每用户独立 | ❌ | 条目存储和状态独立 |
| Feed Icon | ❌ | ✅ 共享 | `icons` 表按 URL hash 全局去重 |
| Feed Icon 检查 | ❌ | ✅ 共享 | `icon.NewIconChecker` 跨用户共享图标 |

### 17.7 潜在改进：共享缓存层

如果需要减少重复 HTTP 请求，可以考虑在应用层添加一个**共享缓存层**（类似 HTTP 共享缓存 / RFC 9111 第 3.1 节的 "shared cache"）：

```
┌─────────┐     ┌───────────────┐     ┌──────────┐
│ User A  │     │               │     │          │
│ feed#1  │────▶│  共享缓存层   │────▶│  源站    │
│         │     │  (内存/Redis) │     │          │
├─────────┤     │               │     │          │
│ User B  │     │  key: URL     │     │          │
│ feed#2  │────▶│  val: ETag,   │     │          │
│         │     │       body,   │     │          │
├─────────┤     │       TTL     │     │          │
│ User C  │     │               │     │          │
│ feed#3  │────▶│               │     │          │
└─────────┘     └───────────────┘     └──────────┘
```

**注意事项**：
- 共享缓存只适用于**无认证、无 Cookie、无个性化**的公共 feed
- 对于带认证或个性化的 feed，缓存键必须包含 `(URL, Cookie, Username)` 等上下文
- 需要处理 `Vary` 头语义（如 `Vary: Cookie` 时，不同 Cookie 值需要不同缓存条目）
- 当前 Miniflux **没有**实现此优化

---

## 十八、相关文件清单

| 文件路径 | 作用 |
|----------|------|
| `internal/model/feed.go` | Feed 数据模型，缓存字段定义、ScheduleNextCheck 调度逻辑、错误计数、WithTranslatedErrorMessage |
| `internal/model/feed_creation_request.go` | Feed 创建请求模型，包含 ETag/LastModified 传递（订阅发现路径） |
| `internal/model/job.go` | Job 数据模型，用于刷新任务队列 |
| `internal/model/web_session.go` | Web 会话管理，包含 ForceRefreshInterval 防刷机制 |
| `internal/reader/fetcher/request_builder.go` | HTTP 请求构建，Accept-Encoding 设置（br,gzip）、三级代理配置、Transport 配置 |
| `internal/reader/fetcher/response_handler.go` | HTTP 响应处理，gzip/br 解码器、ETag/Last-Modified 解析、错误分类、Cloudflare 挑战检测 |
| `internal/reader/fetcher/response_handler_test.go` | 响应处理测试，包含 Cloudflare 挑战、IsModified 测试用例 |
| `internal/reader/fetcher/readclosers.go` | 自定义 Brotli/Gzip ReadCloser 解码器实现 |
| `internal/reader/handler/handler.go` | Feed 创建/刷新业务逻辑，ETag 写入、NextCheckAt 前置更新、429 限流处理、错误持久化出口 |
| `internal/reader/rss/rss.go` | RSS XML 结构定义，包含 TTL 字段 |
| `internal/reader/rss/adapter.go` | RSS 适配器，TTL 字段解析转换 |
| `internal/reader/processor/processor.go` | Feed 条目处理逻辑 |
| `internal/reader/icon/checker.go` | Feed Icon 检查器，跨用户共享图标 |
| `internal/reader/opml/handler.go` | OPML 导入，FeedURLExists 去重检查 |
| `internal/storage/feed.go` | Feed 数据持久化，CreateFeed/UpdateFeed/UpdateFeedError 三条路径 |
| `internal/storage/feed_query_builder.go` | Feed 查询构建器，user_id 条件注入 |
| `internal/storage/batch.go` | 批次构建，BATCH_SIZE 限制、WithUserID 用户隔离、WithErrorLimit 错误过滤、WithLimitPerHost 主机限流 |
| `internal/storage/entry.go` | 条目数据操作，entryExists (feed_id,hash) 去重、RefreshFeedEntries 条目刷新、FOR UPDATE SKIP LOCKED 行锁 |
| `internal/storage/nav_metadata.go` | 导航元数据，错误 feed 计数统计 |
| `internal/database/migrations.go` | 数据库表结构定义，UNIQUE(user_id,feed_url) 和 UNIQUE(feed_id,hash) 约束 |
| `internal/validator/feed.go` | Feed 验证，FeedURLExists/AnotherFeedURLExists 去重校验 |
| `internal/cli/refresh_feeds.go` | CLI 批量刷新调度入口，Worker Pool 推送任务 |
| `internal/cli/scheduler.go` | 后台调度器，time.Tick 固定间隔触发，传递 BATCH_SIZE/LIMIT_PER_HOST 等配置 |
| `internal/worker/pool.go` | Worker 池管理，WORKER_POOL_SIZE 并发上限、channel 任务队列 |
| `internal/worker/worker.go` | Worker 执行逻辑，调用 RefreshFeed |
| `internal/ui/feed_refresh.go` | Web UI 刷新接口（单个/全部），ForceRefreshInterval 防刷 |
| `internal/ui/category_refresh.go` | Web UI 分类刷新接口 |
| `internal/api/feed_handlers.go` | API 刷新接口 |
| `internal/proxyrotator/proxyrotator.go` | 代理轮换池实现，HTTP_CLIENT_PROXIES 处理 |
| `internal/config/options.go` | 配置选项，BATCH_SIZE/WORKER_POOL_SIZE/POLLING_PARSING_ERROR_LIMIT/HTTP_CLIENT_TIMEOUT 等配置定义 |
| `internal/config/options_parsing_test.go` | 配置解析测试，包含 PARSING_ERROR_LIMIT 测试用例 |
| `internal/http/server/middleware.go` | HTTP 服务器中间件，X-Forwarded-Proto 处理 |
| `internal/http/response/builder.go` | HTTP 响应构建，Miniflux 作为服务端的 Vary: Accept-Encoding 设置 |
| `internal/http/request/client_ip.go` | 客户端 IP 解析，X-Forwarded-For / X-Real-Ip 处理 |
| `internal/locale/translations/zh_CN.json` | 中文翻译，包含 Cloudflare 挑战提示信息 |
