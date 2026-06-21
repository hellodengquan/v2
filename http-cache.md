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

## 九、相关文件清单

| 文件路径 | 作用 |
|----------|------|
| `internal/model/feed.go` | Feed 数据模型，包含缓存字段 |
| `internal/reader/fetcher/request_builder.go` | HTTP 请求构建，设置缓存协商头 |
| `internal/reader/fetcher/response_handler.go` | HTTP 响应处理，解析缓存头、判断修改 |
| `internal/reader/handler/handler.go` | Feed 刷新业务逻辑 |
| `internal/storage/feed.go` | Feed 数据持久化 |
| `internal/database/migrations.go` | 数据库表结构定义 |
| `internal/cli/refresh_feeds.go` | 批量刷新调度 |
