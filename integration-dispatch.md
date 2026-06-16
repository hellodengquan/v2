# Miniflux 对外推送框架分析

本文档分析 Miniflux 向 Telegram、Slack、Webhook 等渠道分发订阅事件的完整链路，包括渠道注册、消息组装、失败重试与并发节流的协作机制。

## 一、整体架构概览

Miniflux 的推送框架采用 **分层设计**：

```
守护进程 (daemon.go)
    ↓
调度器 (scheduler.go) → 定期生成 Feed 刷新任务
    ↓
Worker 池 (worker/pool.go) → 并发执行 Feed 刷新
    ↓
Feed 处理器 (reader/handler/handler.go) → 抓取、解析、存储新条目
    ↓
集成推送入口 (integration/integration.go) → 分发到各渠道
    ↓
渠道实现 (telegrambot/, slack/, webhook/) → 具体消息发送
```

## 二、渠道注册机制

### 2.1 配置模型

渠道注册基于 `model.Integration` 结构体（`internal/model/integration.go`），每个渠道通过布尔标志位启用：

```go
type Integration struct {
    UserID             int64
    TelegramBotEnabled bool      // Telegram 开关
    TelegramBotToken   string    // Bot Token
    TelegramBotChatID  string    // 目标 Chat ID
    SlackEnabled       bool      // Slack 开关
    SlackWebhookLink   string    // Slack Webhook URL
    WebhookEnabled     bool      // Webhook 开关
    WebhookURL         string    // Webhook 目标 URL
    WebhookSecret      string    // Webhook 签名密钥
    // ... 其他 20+ 个渠道配置
}
```

### 2.2 渠道分发逻辑

核心分发函数 `PushEntries` 位于 `internal/integration/integration.go:511`，采用 **顺序检查 + 并发推送** 模式：

```go
func PushEntries(feed *model.Feed, entries model.Entries, userIntegrations *model.Integration) {
    // 批量推送渠道（一次调用推送多条）
    if userIntegrations.WebhookEnabled {
        webhookClient.SendNewEntriesWebhookEvent(feed, entries)
    }
    if userIntegrations.SlackEnabled {
        slackClient.SendSlackMsg(feed, entries)
    }
    
    // 单条推送渠道（循环逐条推送）
    if userIntegrations.TelegramBotEnabled {
        for _, entry := range entries {
            telegrambot.PushEntry(feed, entry, ...)
        }
    }
}
```

**关键点**：
- 渠道判断在调用层完成，不是通过接口注册
- 批量推送渠道与单条推送渠道采用不同处理策略
- 各渠道之间完全独立，一个渠道失败不影响其他渠道

## 三、消息组装策略

三个目标渠道的消息组装方式各有特点：

### 3.1 Telegram 消息组装 (`internal/integration/telegrambot/telegrambot.go:16`)

```go
func PushEntry(feed *model.Feed, entry *model.Entry, ...) error {
    // HTML 格式消息
    formattedText := fmt.Sprintf(
        `<b>%s</b> - <a href=%q>%s</a>`,
        feed.Title, entry.URL, entry.Title,
    )
    
    message := &MessageRequest{
        ChatID:      chatID,
        Text:        formattedText,
        ParseMode:   HTMLFormatting,
        ReplyMarkup: &InlineKeyboard{...}, // 可选按钮
    }
    
    return client.SendMessage(message)
}
```

**特点**：
- 单条推送，每条 Entry 发送一条消息
- 支持 HTML 格式和内联按钮（跳转到 Miniflux、原文、评论）
- 支持 Topic ID、禁用网页预览、禁用通知等配置

### 3.2 Slack 消息组装 (`internal/integration/slack/slack.go:34`)

```go
func (c *Client) SendSlackMsg(feed *model.Feed, entries model.Entries) error {
    for _, entry := range entries {
        requestBody, _ := json.Marshal(&slackMessage{
            Attachments: []slackAttachments{{
                Title: "RSS feed update from Miniflux",
                Color: slackMsgColor,
                Fields: []slackFields{
                    {Title: "Updated feed", Value: feed.Title},
                    {Title: "Article title", Value: entry.Title},
                    {Title: "Article link", Value: entry.URL},
                    {Title: "Author", Value: entry.Author, Short: true},
                    {Title: "Source website", Value: urllib.RootURL(feed.SiteURL), Short: true},
                },
            }},
        })
        // 发送 HTTP 请求...
    }
    return nil
}
```

**特点**：
- 循环逐条发送，但在 Slack 客户端内部循环
- 使用 Slack Attachment 格式，包含结构化字段
- 每个条目发送一条独立的 Slack 消息

### 3.3 Webhook 消息组装 (`internal/integration/webhook/webhook.go:73`)

```go
func (c *Client) SendNewEntriesWebhookEvent(feed *model.Feed, entries model.Entries) error {
    webhookEntries := make([]*WebhookEntry, 0, len(entries))
    for _, entry := range entries {
        webhookEntries = append(webhookEntries, &WebhookEntry{
            ID: entry.ID, Title: entry.Title, URL: entry.URL,
            Content: entry.Content, Author: entry.Author,
            // ... 完整的条目元数据
        })
    }
    
    return c.makeRequest(NewEntriesEventType, &WebhookNewEntriesEvent{
        EventType: NewEntriesEventType,
        Feed:      &WebhookFeed{...},  // Feed 完整信息
        Entries:   webhookEntries,     // 所有新条目
    })
}
```

**特点**：
- 批量推送，一次 HTTP 请求包含所有新条目
- 包含完整的 Feed 和 Entry 元数据（ID、状态、标签、附件等）
- 支持两种事件类型：`new_entries`（刷新时）和 `save_entry`（用户保存时）
- 签名验证：`X-Miniflux-Signature` 头使用 HMAC-SHA256 签名请求体

## 四、失败重试机制

Miniflux 的重试机制分为 **Feed 抓取重试** 和 **集成推送容错** 两层：

### 4.1 Feed 抓取层面的重试

#### 错误计数机制 (`internal/model/feed.go:100-110`)

```go
// 记录错误并递增计数器
func (f *Feed) WithTranslatedErrorMessage(message string) {
    f.ParsingErrorCount++
    f.ParsingErrorMsg = message
}

// 成功时重置错误计数器
func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0
    f.ParsingErrorMsg = ""
}
```

#### 调度器错误过滤 (`internal/cli/scheduler.go:36-42`)

```go
jobs, err := store.NewBatchBuilder().
    WithBatchSize(batchSize).
    WithErrorLimit(errorLimit).  // POLLING_PARSING_ERROR_LIMIT（默认3）
    WithoutDisabledFeeds().
    WithNextCheckExpired().
    FetchJobs()
```

**重试策略**：
- 当 `ParsingErrorCount` 超过 `POLLING_PARSING_ERROR_LIMIT`（默认 3），Feed 被移出调度队列
- 成功刷新后调用 `ResetErrorCounter()` 重置计数

#### 限流重试 (`internal/reader/handler/handler.go:245-255`)

```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()  // 解析 Retry-After 头
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
}
```

- 支持 `Retry-After` 响应头（秒数或 HTTP 日期格式）
- 自动根据限流延迟调整下一次检查时间

### 4.2 集成推送层面的容错

**注意：集成推送本身没有内置重试机制**，仅记录错误日志：

```go
// integration.go:684-689
if err := telegrambot.PushEntry(...); err != nil {
    slog.Error("Unable to send entry to Telegram", ...)
}
```

**容错特点**：
- 推送失败仅记录日志，不影响主流程
- 无持久化队列，失败的推送不会重试
- 各渠道独立，一个渠道失败不影响其他渠道

## 五、并发节流机制

### 5.1 Worker 池并发控制 (`internal/worker/`)

#### 池化结构 (`internal/worker/pool.go:14-44`)

```go
type Pool struct {
    queue chan model.Job  // 任务队列（无缓冲 channel）
    wg    sync.WaitGroup  // 等待所有 worker 完成
}

func NewPool(store *storage.Storage, nbWorkers int) *Pool {
    workerPool := &Pool{
        queue: make(chan model.Job),  // 无缓冲 channel，自然限流
    }
    
    for i := range nbWorkers {  // WORKER_POOL_SIZE（默认16）
        workerPool.wg.Add(1)
        worker := &worker{id: i, store: store}
        go worker.Run(workerPool.queue, &workerPool.wg)
    }
    
    return workerPool
}
```

#### Worker 执行循环 (`internal/worker/worker.go:24-49`)

```go
func (w *worker) Run(c <-chan model.Job, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range c {
        startTime := time.Now()
        localizedError := feedHandler.RefreshFeed(w.store, job.UserID, job.FeedID, false)
        // 指标采集...
    }
}
```

**并发控制要点**：
- 固定数量 worker（默认 16，通过 `WORKER_POOL_SIZE` 配置）
- 无缓冲 channel 作为队列，实现自然的背压
- worker 数量限制了同时刷新的 Feed 数量

### 5.2 调度层面的节流

#### 批处理大小 (`internal/cli/scheduler.go:37`)

```go
WithBatchSize(batchSize)  // BATCH_SIZE（默认100）
```

每轮调度最多获取 100 个 Feed 进行刷新。

#### 轮询频率 (`internal/cli/scheduler.go:34`)

```go
for range time.Tick(frequency) {  // POLLING_FREQUENCY（默认60分钟）
    // 获取下一批任务...
}
```

#### 每主机并发限制 (`internal/cli/scheduler.go:41`)

```go
WithLimitPerHost(limitPerHost)  // POLLING_LIMIT_PER_HOST（默认0，不限制）
```

限制同一主机名的并发刷新数量，避免对目标站点造成压力。

### 5.3 推送层面的异步隔离

```go
// internal/reader/handler/handler.go:338
go integration.PushEntries(originalFeed, newEntries, userIntegrations)
```

**关键点**：
- 使用 `go` 关键字启动独立 goroutine 执行推送
- 推送操作不阻塞 Feed 刷新主流程
- 推送 goroutine 与 worker 池完全隔离
- 多个 Feed 同时刷新时，可能产生大量推送 goroutine

### 5.4 HTTP 客户端超时控制

所有渠道 HTTP 请求都设置了超时：
- Telegram: `defaultClientTimeout = 10 * time.Second` (`telegrambot/client.go:19`)
- Slack: `defaultClientTimeout = 10 * time.Second` (`slack/slack.go:23`)
- Webhook: `defaultClientTimeout = 10 * time.Second` (`webhook/webhook.go:22`)

## 六、完整调用链路

### 6.1 启动流程

```
startDaemon(store) [cli/daemon.go:23]
    ↓
worker.NewPool(store, WORKER_POOL_SIZE=16) [worker/pool.go:33]
    ↓ 启动 16 个 worker goroutine
runScheduler(store, pool) [cli/scheduler.go:15]
    ↓
go feedScheduler(...) [cli/scheduler.go:18]
```

### 6.2 调度与执行流程

```
每 POLLING_FREQUENCY（60分钟）触发一次
    ↓
store.NewBatchBuilder().FetchJobs() [cli/scheduler.go:36]
    ↓ 获取最多 BATCH_SIZE（100）个到期 Feed
pool.Push(jobs) [worker/pool.go:20]
    ↓ 写入 channel，分发给空闲 worker
worker.Run() → feedHandler.RefreshFeed() [worker/worker.go:40]
```

### 6.3 Feed 刷新与推送流程

```
feedHandler.RefreshFeed(store, userID, feedID, false) [reader/handler/handler.go:195]
    ↓
1. 从数据库获取 Feed 配置
2. 构建 HTTP 请求（User-Agent、Cookie、代理等）
3. 执行请求获取 Feed 内容
4. 解析 Feed（RSS/Atom/JSON）
5. 处理条目（过滤、重写规则等）
6. store.RefreshFeedEntries() → 返回新条目列表
    ↓
userIntegrations, _ := store.Integration(userID) [handler.go:330]
    ↓
go integration.PushEntries(feed, newEntries, userIntegrations) [handler.go:338]
    ↓ 异步 goroutine 中执行
integration.PushEntries() [integration/integration.go:511]
    ├─→ Webhook: SendNewEntriesWebhookEvent(feed, entries)
    ├─→ Slack: SendSlackMsg(feed, entries)
    └─→ Telegram: 循环 PushEntry(feed, entry) for each entry
```

### 6.4 关键配置参数汇总

| 参数 | 默认值 | 位置 | 作用 |
|------|--------|------|------|
| `WORKER_POOL_SIZE` | 16 | config/options.go:588 | Worker 数量，控制并发刷新数 |
| `BATCH_SIZE` | 100 | config/options.go:107 | 每轮调度的 Feed 数量 |
| `POLLING_FREQUENCY` | 60分钟 | config/options.go:482 | 调度器轮询间隔 |
| `POLLING_PARSING_ERROR_LIMIT` | 3 | config/options.go:498 | 错误次数上限，超过后暂停调度 |
| `POLLING_LIMIT_PER_HOST` | 0 | config/options.go:490 | 每主机并发限制（0=不限制） |
| `HTTP_CLIENT_TIMEOUT` | 20秒 | config/options.go:275 | Feed 抓取超时 |
| `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL` | 60分钟 | config/options.go:556 | Feed 最小刷新间隔 |

## 七、架构总结

### 7.1 设计优点

1. **解耦清晰**：Feed 刷新与集成推送完全分离，通过 goroutine 异步执行
2. **容错性好**：单个渠道或单个 Feed 失败不影响整体系统
3. **配置灵活**：每个用户可独立配置多个渠道，支持 Feed 级 Webhook 覆盖
4. **背压机制**：Worker 池 + 无缓冲 channel 实现自然限流
5. **安全考虑**：Webhook 签名验证、私有网络访问控制

### 7.2 潜在改进点

1. **推送无重试**：集成推送失败后不会重试，可能导致消息丢失
2. **推送无队列**：大量 Feed 同时刷新时，可能产生大量推送 goroutine
3. **无速率限制**：对第三方 API 的调用速率没有限制（如 Telegram 的每分钟消息限制）
4. **批量不一致**：Webhook 批量推送 vs Telegram 单条推送，策略不统一

### 7.3 关键代码位置

| 模块 | 文件路径 | 核心行号 |
|------|----------|----------|
| 推送入口 | `internal/integration/integration.go` | 511 (`PushEntries`) |
| 触发点 | `internal/reader/handler/handler.go` | 338 (`go integration.PushEntries`) |
| Worker 池 | `internal/worker/pool.go` | 33 (`NewPool`) |
| 调度器 | `internal/cli/scheduler.go` | 33 (`feedScheduler`) |
| Telegram 实现 | `internal/integration/telegrambot/telegrambot.go` | 16 (`PushEntry`) |
| Slack 实现 | `internal/integration/slack/slack.go` | 34 (`SendSlackMsg`) |
| Webhook 实现 | `internal/integration/webhook/webhook.go` | 73 (`SendNewEntriesWebhookEvent`) |
| 配置模型 | `internal/model/integration.go` | 7 (`Integration` 结构体) |
| 错误重试 | `internal/model/feed.go` | 100-149 |
