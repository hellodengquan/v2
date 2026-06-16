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

## 八、高级特性深度分析

### 8.1 运行时动态注册第三方 Channel 扩展

**当前实现状态：不支持**

Miniflux 的推送框架采用 **硬编码静态注册** 模式，而非动态插件机制：

```go
// internal/integration/integration.go 中的模式
if userIntegrations.WebhookEnabled { ... }
if userIntegrations.SlackEnabled { ... }
if userIntegrations.TelegramBotEnabled { ... }
// ... 每个渠道都是独立的 if 判断
```

**代码证据**：
- 所有渠道在 `internal/model/integration.go` 中以硬编码字段定义（20+ 个 `*Enabled` 布尔字段）
- `internal/integration/integration.go` 中顺序导入所有渠道包（第 9-36 行）
- 分发逻辑 `PushEntries` 中每个渠道独立判断，没有 `map[string]Handler` 或 `registry` 模式
- 没有 `plugin` 包、`interface` 注册机制或 Go plugin 系统

**扩展方式**：添加新渠道需要修改三处代码：
1. `internal/model/integration.go`：添加配置字段
2. `internal/integration/<newchannel>/`：实现渠道客户端
3. `internal/integration/integration.go`：添加导入和分发逻辑

### 8.2 用户自定义模板与 i18n

**消息模板：硬编码，不支持用户自定义**

三个渠道的消息格式都是代码写死的：

```go
// Telegram: telegrambot.go:17-22
formattedText := fmt.Sprintf(
    `<b>%s</b> - <a href=%q>%s</a>`,
    feed.Title, entry.URL, entry.Title,
)

// Slack: slack.go:36-66
requestBody, _ := json.Marshal(&slackMessage{
    Attachments: []slackAttachments{{
        Title: "RSS feed update from Miniflux",
        // ... 固定字段结构
    }},
})

// Webhook: webhook.go:78-114
// 固定的 WebhookEntry / WebhookFeed 结构体字段映射
```

**i18n 国际化：不应用于推送消息**

- `internal/locale/` 包仅用于 Web UI 界面和错误消息翻译
- 推送消息中的固定文本（如 Slack 的 "RSS feed update from Miniflux"）是硬编码英文
- Telegram 按钮文本（"Go to Miniflux"、"Go to article"、"Comments"）也是硬编码英文

### 8.3 毒丸消息（Dead Letter Queue）

**当前实现状态：不存在**

Miniflux 推送框架没有 DLQ 机制：

```go
// 所有渠道推送失败后仅记录日志，没有持久化
if err := telegrambot.PushEntry(...); err != nil {
    slog.Error("Unable to send entry to Telegram", ...)
    // 没有：将消息写入 DLQ 表、没有重试队列
}
```

**缺失的能力**：
- 没有数据库表存储失败的推送消息
- 没有死信队列重试机制
- 没有消息重放功能
- 失败即丢失，无法追溯

### 8.4 优先级队列与紧急通知

**消息优先级：部分渠道支持，但无全局优先级队列**

Ntfy 和 Pushover 渠道支持单条消息级别的优先级配置，但不是在推送调度层面实现：

```go
// Ntfy: ntfy.go:48-56
ntfyMessage := &ntfyMessage{
    Topic:    c.ntfyTopic,
    Message:  entry.Title,
    Priority: c.ntfyPriority,  // Feed 级配置，范围 1-5
}

// Pushover: pushover.go:76-90
msg := &message{
    Priority: c.priority,  // Feed 级配置，范围 -2 到 2
}
```

**配置方式**（`internal/model/feed.go:61-63`）：
```go
NtfyPriority      int  // 每个 Feed 可单独设置
PushoverPriority  int  // 每个 Feed 可单独设置
```

**限制**：
- 只有 Ntfy 和 Pushover 支持优先级
- 优先级是 Feed 级配置，不是按事件紧急程度动态判断
- 没有全局优先级队列，高优先级消息不会插队
- 三个目标渠道（Telegram、Slack、Webhook）不支持优先级

### 8.5 多 Channel 同时启用时的去重

**推送层面：无去重，可能重复推送**

如果用户同时启用多个渠道，同一条 Entry 会推送到所有启用的渠道，推送之间没有去重：

```go
// integration.go:511-692
if userIntegrations.WebhookEnabled {
    webhookClient.SendNewEntriesWebhookEvent(feed, entries)  // 第一次推送
}
if userIntegrations.SlackEnabled {
    slackClient.SendSlackMsg(feed, entries)  // 第二次推送，同一条 entry
}
if userIntegrations.TelegramBotEnabled {
    for _, entry := range entries {
        telegrambot.PushEntry(feed, entry, ...)  // 第三次推送
    }
}
```

**存储层面：基于 Hash 的条目去重**

虽然推送不做去重，但条目入库时有严格的去重机制：

```go
// internal/storage/entry.go:213-224
func (s *Storage) entryExists(tx *sql.Tx, entry *model.Entry) (bool, error) {
    // 基于 feed_id + hash 的唯一索引去重
    err := tx.QueryRow(`SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1`, 
        entry.FeedID, entry.Hash).Scan(&result)
}
```

**Hash 生成**（`internal/crypto/crypto.go:18-23`）：
```go
func HashFromBytes(value []byte) string {
    h := fnv.New128a()  // 非加密哈希，用于去重
    h.Write(value)
    return hex.EncodeToString(h.Sum(nil))
}
```

### 8.6 订阅事件大量产生时的批量合并

**各渠道策略不一致**

| 渠道 | 批量策略 | 代码位置 |
|------|----------|----------|
| Webhook | 批量推送，一次 HTTP 请求包含所有新条目 | `webhook.go:73` |
| Slack | 内部循环逐条发送，每条独立 HTTP 请求 | `slack.go:35` |
| Telegram | 外层循环逐条推送，每条独立 HTTP 请求 | `integration.go:667` |

**Webhook 批量实现**：
```go
// webhook.go:73-114
func (c *Client) SendNewEntriesWebhookEvent(feed *model.Feed, entries model.Entries) error {
    webhookEntries := make([]*WebhookEntry, 0, len(entries))
    for _, entry := range entries {
        webhookEntries = append(webhookEntries, &WebhookEntry{...})
    }
    // 一次请求发送所有 entries
    return c.makeRequest(NewEntriesEventType, &WebhookNewEntriesEvent{
        Feed:    &WebhookFeed{...},
        Entries: webhookEntries,  // 批量
    })
}
```

**Slack 逐条实现**：
```go
// slack.go:34-98
func (c *Client) SendSlackMsg(feed *model.Feed, entries model.Entries) error {
    for _, entry := range entries {  // 循环逐条
        requestBody, _ := json.Marshal(...)
        // 每条 entry 独立 HTTP 请求
        httpClient.Do(request)
    }
}
```

### 8.7 Webhook 签名校验防伪造

**已实现完整的 HMAC-SHA256 签名机制**

**签名生成**（`internal/integration/webhook/webhook.go:134`）：
```go
request.Header.Set("X-Miniflux-Signature", 
    crypto.GenerateSHA256Hmac(c.webhookSecret, requestBody))
request.Header.Set("X-Miniflux-Event-Type", eventType)
```

**HMAC 实现**（`internal/crypto/crypto.go:48-52`）：
```go
func GenerateSHA256Hmac(secret string, data []byte) string {
    h := hmac.New(sha256.New, []byte(secret))
    h.Write(data)
    return hex.EncodeToString(h.Sum(nil))
}
```

**服务端校验方式**（接收方需要实现）：
```
1. 读取请求体原始内容
2. 使用相同的 webhookSecret 计算 HMAC-SHA256
3. 与 X-Miniflux-Signature 头比较（使用恒定时间比较防止时序攻击）
4. 参考实现：crypto.ConstantTimeCmp(a, b)
```

**安全特性**：
- 使用 SHA-256 哈希算法
- Hex 编码格式传输
- 可配合 `crypto/subtle.ConstantTimeCompare` 进行安全比较（`crypto.go:59-60`）
- 事件类型头 `X-Miniflux-Event-Type` 用于区分事件类型

### 8.8 Dispatch 进度反馈与重试可视化

**当前实现：无进度反馈，仅有基础指标监控**

**Prometheus 指标监控**（`internal/metric/metric.go`）：

```go
// Feed 刷新耗时指标（worker.go:42-48）
metric.BackgroundFeedRefreshDuration.WithLabelValues(status).
    Observe(time.Since(startTime).Seconds())

// 可用指标：
BackgroundFeedRefreshDuration  // Feed 刷新耗时直方图（分 success/error）
ScraperRequestDuration         // 爬虫请求耗时
ArchiveEntriesDuration         // 归档耗时
brokenFeedsGauge               // 失败 Feed 数量
entriesGauge                   // 条目数量统计（unread/read）
```

**缺失的可视化能力**：
- 没有推送任务的进度追踪
- 没有重试次数统计
- 没有各渠道推送成功率/失败率指标
- 没有 Web UI 展示推送历史
- 没有推送队列积压监控

**日志记录**（仅错误日志）：
```go
slog.Error("Unable to send entry to Telegram",
    slog.Int64("user_id", userIntegrations.UserID),
    slog.Int64("entry_id", entry.ID),
    slog.String("entry_url", entry.URL),
    slog.Any("error", err),
)
```

## 九、架构总结（补充版）

### 9.1 设计优点

1. **解耦清晰**：Feed 刷新与集成推送完全分离，通过 goroutine 异步执行
2. **容错性好**：单个渠道或单个 Feed 失败不影响整体系统
3. **配置灵活**：每个用户可独立配置多个渠道，支持 Feed 级 Webhook 覆盖
4. **背压机制**：Worker 池 + 无缓冲 channel 实现自然限流
5. **安全考虑**：Webhook 签名验证、私有网络访问控制
6. **安全哈希**：条目基于 FNV-128a 哈希去重，Webhook 使用 HMAC-SHA256 防伪造

### 9.2 潜在改进点（已验证的缺失功能）

1. **推送无重试**：集成推送失败后不会重试，可能导致消息丢失
2. **推送无队列**：大量 Feed 同时刷新时，可能产生大量推送 goroutine
3. **无速率限制**：对第三方 API 的调用速率没有限制（如 Telegram 的每分钟消息限制）
4. **批量不一致**：Webhook 批量推送 vs Telegram/Slack 单条推送，策略不统一
5. **无动态扩展**：添加新渠道需要修改核心代码，不支持插件
6. **无消息模板**：推送格式硬编码，用户无法自定义
7. **无 DLQ**：失败消息无法追溯和重放
8. **无优先级队列**：高优先级消息无法插队处理
9. **无推送去重**：多渠道同时启用时同一条消息会重复推送
10. **无进度可视化**：没有推送状态追踪和 UI 展示

### 9.3 关键代码位置（补充版）

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
| 条目去重 | `internal/storage/entry.go` | 213 (`entryExists`) |
| 哈希生成 | `internal/crypto/crypto.go` | 18 (`HashFromBytes`) |
| HMAC 签名 | `internal/crypto/crypto.go` | 48 (`GenerateSHA256Hmac`) |
| Ntfy 优先级 | `internal/integration/ntfy/ntfy.go` | 54 (`Priority: c.ntfyPriority`) |
| Pushover 优先级 | `internal/integration/pushover/pushover.go` | 88 (`Priority: c.priority`) |
| 指标监控 | `internal/metric/metric.go` | 24 (`BackgroundFeedRefreshDuration`) |

## 十、高级架构特性深度分析（续）

### 10.1 IntegrationPluginInterface 插件市场发现机制

**当前实现状态：完全不存在**

Miniflux 采用 **静态编译 + 硬编码导入** 模式，没有插件架构：

```go
// internal/integration/integration.go:9-37
// 全部 20+ 个渠道都是静态 import
import (
    "miniflux.app/v2/internal/integration/apprise"
    "miniflux.app/v2/internal/integration/archiveorg"
    "miniflux.app/v2/internal/integration/betula"
    // ... 每个渠道单独 import
    "miniflux.app/v2/internal/integration/webhook"
)
```

**缺失的插件能力**：
- 没有 `IntegrationPluginInterface` 接口定义
- 没有 `plugin` 包或 `registry` 注册中心
- 没有插件市场、发现机制或版本管理
- 没有 Go plugin 系统或 WASM 扩展支持
- 没有动态加载/卸载机制

**扩展新渠道的唯一方式**：修改三处源码并重新编译：
1. `internal/model/integration.go`：添加 `*Enabled` 布尔字段和配置字段
2. `internal/integration/<newchannel>/`：实现客户端逻辑
3. `internal/integration/integration.go`：添加 import 和 `PushEntries`/`SendEntry` 分发逻辑

### 10.2 自定义模板与 i18n 资源包冲突解决

**推送消息模板：完全硬编码，不支持自定义**

三个目标渠道的消息格式都是代码内写死的字符串拼接：

```go
// Telegram: telegrambot.go:17-22
formattedText := fmt.Sprintf(`<b>%s</b> - <a href=%q>%s</a>`, 
    feed.Title, entry.URL, entry.Title)

// Slack: slack.go:36-66
Title: "RSS feed update from Miniflux",  // 固定英文标题
```

**i18n 国际化：仅用于 Web UI，不应用于推送**

i18n 系统架构（`internal/locale/catalog.go`）：
```go
// 翻译资源通过 embed 打包
//go:embed translations/*.json
var translationFiles embed.FS

// 运行时加载到内存 map
var defaultCatalog = make(catalog, len(AvailableLanguages))
```

**i18n 资源冲突解决：不存在**
- 翻译资源在编译时通过 `embed.FS` 打包，没有运行时覆盖机制
- 没有多语言包优先级或冲突检测逻辑
- 推送消息中没有调用 `locale.Printf()` 或类似的翻译函数
- Telegram 按钮、Slack 标题等固定文本均为硬编码英文

**相关代码证据**：
- `internal/locale/translations/` 目录下 20+ 个 JSON 语言包
- 但 `internal/integration/` 目录下所有文件都没有 import `miniflux.app/v2/internal/locale`

### 10.3 DLQ 重投策略

**当前实现状态：完全不存在**

Miniflux 推送框架没有死信队列（Dead Letter Queue）概念：

```go
// 所有渠道推送失败后仅记录日志，没有持久化
if err := webhookClient.SendNewEntriesWebhookEvent(...); err != nil {
    slog.Error("Unable to send entry to Webhook",
        slog.Int64("user_id", userIntegrations.UserID),
        slog.Any("error", err),
    )
    // 没有：将消息写入 DLQ 表
    // 没有：设置延迟重试时间
    // 没有：记录失败次数和原因
}
```

**缺失的 DLQ 能力**：
- 没有数据库表存储失败的推送消息（如 `integration_dlq`）
- 没有重投策略（指数退避、最大重试次数、死信阈值）
- 没有手动重放按钮或 API
- 没有 DLQ 监控指标
- 没有消息审计日志

**相关的现有机制（可参考）**：
- Feed 抓取失败有 `ParsingErrorCount` 计数机制（`feed.go:100-149`），但不适用于推送
- Web Session 有 `Rotate()` 轮换机制（`web_session.go:68-76`），可作为重试策略参考

### 10.4 PriorityChannel 优先级反转检测

**当前实现状态：完全不存在**

Miniflux 没有优先级通道（PriorityChannel）或优先级反转检测机制：

```go
// worker/pool.go:14-20
type Pool struct {
    queue chan model.Job  // 普通无缓冲 channel，无优先级
    wg    sync.WaitGroup
}

// 任务从 channel 中 FIFO 取出，没有优先级排序
for job := range c {
    feedHandler.RefreshFeed(...)  // 按接收顺序执行
}
```

**优先级支持现状**：
- 仅 Ntfy 和 Pushover 支持消息级优先级（Feed 静态配置）
- 没有运行时动态优先级调整
- 没有优先级队列（如 heap 实现）
- 没有优先级反转检测和继承协议

**缺失的能力**：
- 没有 `PriorityChannel` 数据结构
- 没有优先级分类（紧急/高/中/低）
- 没有优先级反转检测算法（如检测低优先级任务持有锁阻塞高优先级）
- 没有优先级继承或优先级天花板协议

### 10.5 dedupKey 哈希碰撞兜底

**当前实现状态：部分实现，有兜底但不完善**

**数据库层面的兜底**（`database/migrations.go:88`）：
```sql
-- 唯一索引防止哈希碰撞导致重复插入
unique (feed_id, hash),
```

**RSS 适配器层面的 GUID 碰撞处理**（`internal/reader/rss/adapter.go:116-139`）：
```go
// Generate the entry hash.
//
// The RSS 2.0 spec requires <guid> to uniquely identify the item, but
// some feeds ship the same GUID for every entry. Keep the first
// occurrence stable (so existing stored entries still match) and
// disambiguate later collisions using the entry URL or, as a last
// resort, the item position.
switch {
case item.GUID.Data != "":
    n := seenGUIDs[item.GUID.Data]
    seenGUIDs[item.GUID.Data] = n + 1
    switch {
    case n == 0:
        entry.Hash = crypto.SHA256(item.GUID.Data)  // 第一次使用 GUID 哈希
    case entry.URL != "":
        entry.Hash = crypto.SHA256(item.GUID.Data + "|" + entry.URL)  // 碰撞后追加 URL
    default:
        entry.Hash = crypto.SHA256(item.GUID.Data + "|" + strconv.Itoa(n))  // 最后使用位置
    }
case entryURL != "":
    entry.Hash = crypto.SHA256(entryURL)
default:
    entry.Hash = crypto.SHA256(entry.Title + entry.Content)
}
```

**哈希算法分层**（`internal/crypto/crypto.go`）：
| 用途 | 算法 | 长度 | 位置 |
|------|------|------|------|
| 条目去重（存储层） | FNV-128a | 128位 | `HashFromBytes()` 第18行 |
| 条目去重（RSS 适配层） | SHA-256 | 256位 | `SHA256()` 第26行 |
| Webhook 签名 | HMAC-SHA256 | 256位 | `GenerateSHA256Hmac()` 第48行 |
| Session Secret 哈希 | SHA-256 | 256位 | `hashWebSessionSecret()` |
| 密码哈希 | bcrypt | - | `HashPassword()` 第43行 |

**仍缺失的兜底能力**：
- 没有主动的哈希碰撞检测（如存储前先检查是否存在不同内容的相同哈希）
- 没有碰撞发生时的告警机制
- 没有哈希算法升级路径（如从 FNV-128a 迁移到更强的哈希）
- FNV-128a 是非加密哈希，理论碰撞概率高于 SHA-256

### 10.6 BatchCoalescer 自适应窗口

**当前实现状态：不存在自适应批组合并，但有调度间隔算法**

**调度间隔算法**（`internal/model/feed.go:122-149`）：
```go
func (f *Feed) ScheduleNextCheck(weeklyCount int, refreshDelay time.Duration) time.Duration {
    // 默认使用全局配置的最小轮询间隔
    interval := config.Opts.SchedulerRoundRobinMinInterval()

    // 基于条目频率的自适应调度（Entry Frequency 模式）
    if config.Opts.PollingScheduler() == SchedulerEntryFrequency {
        if weeklyCount <= 0 {
            interval = config.Opts.SchedulerEntryFrequencyMaxInterval()
        } else {
            // 根据每周条目数动态计算间隔
            interval = (7 * 24 * time.Hour) / time.Duration(weeklyCount*config.Opts.SchedulerEntryFrequencyFactor())
            interval = min(interval, config.Opts.SchedulerEntryFrequencyMaxInterval())
            interval = max(interval, config.Opts.SchedulerEntryFrequencyMinInterval())
        }
    }

    // 考虑 Retry-After、Cache-Control 等头部
    interval = max(interval, refreshDelay)

    // 限制最大间隔
    interval = min(interval, config.Opts.SchedulerRoundRobinMaxInterval())

    f.NextCheckAt = time.Now().Add(interval)
    return interval
}
```

**两种调度模式**：
1. **Round-Robin 模式**（默认）：固定间隔 `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL`（默认60分钟）
2. **Entry Frequency 模式**：根据每周条目数动态调整，公式：`7天 / (每周条目数 × 因子)`，范围在最小和最大间隔之间

**缺失的 BatchCoalescer 能力**：
- 没有基于时间窗口的批组合并（如收集 5 秒内的所有新条目再推送）
- 没有自适应窗口大小调整（根据负载动态调整批大小）
- 没有滑动窗口或翻滚窗口算法
- 推送层面仍然是：有新条目立即推送，没有延迟合并

**相关配置参数**：
| 参数 | 默认值 | 说明 |
|------|--------|------|
| `POLLING_SCHEDULER` | `round_robin` | 调度算法：`round_robin` 或 `entry_frequency` |
| `SCHEDULER_ENTRY_FREQUENCY_MIN_INTERVAL` | 20分钟 | 条目频率模式最小间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_MAX_INTERVAL` | 180分钟 | 条目频率模式最大间隔 |
| `SCHEDULER_ENTRY_FREQUENCY_FACTOR` | 2 | 条目频率计算因子 |
| `SCHEDULER_ROUND_ROBIN_MIN_INTERVAL` | 60分钟 | 轮询模式最小间隔 |
| `SCHEDULER_ROUND_ROBIN_MAX_INTERVAL` | 180分钟 | 轮询模式最大间隔 |

### 10.7 HMAC 密钥轮换

**Webhook 密钥轮换：不存在，但有 Session 轮换可参考**

**Webhook 现状**：
```go
// webhook/webhook.go:27-37
type Client struct {
    webhookURL    string  // 固定，无版本
    webhookSecret string  // 固定，无版本号或轮换机制
}
```

**可参考的 Session 轮换实现**（`internal/model/web_session.go:68-76`）：
```go
// Rotate assigns a new ID and secret in place, returning the previous ID
// and the new raw secret. Rotating on authentication prevents session fixation.
func (s *WebSession) Rotate() (oldID, newSecret string) {
    oldID = s.ID
    newSecret = rand.Text()
    s.ID = rand.Text()
    s.SecretHash = hashWebSessionSecret(newSecret)
    return oldID, newSecret
}
```

**Session 轮换调用点**（`internal/ui/auth.go:20-33`）：
```go
// authenticateWebSession binds the current browser session to the given user,
// rotates its identifier and secret, and refreshes the client cookie.
func authenticateWebSession(w http.ResponseWriter, r *http.Request, ...) error {
    session := request.WebSession(r)
    session.SetUser(user)

    oldID, secret := session.Rotate()  // 认证时轮换防止会话固定
    if err := store.RotateWebSession(oldID, session); err != nil {
        return err
    }
    setSessionCookie(w, session, secret)
    return nil
}
```

**Webhook 密钥轮换缺失的能力**：
- 没有密钥版本号（如 `X-Miniflux-Key-Version` 头）
- 没有双密钥支持（同时接受新旧密钥实现无缝轮换）
- 没有密钥过期提醒或自动轮换
- 没有 `RotateWebhookSecret()` API 或 UI 操作
- 没有使用 `crypto/rand` 生成安全随机密钥的强制要求

**数据库存储层面**：`integrations` 表中 `webhook_secret` 是普通 text 字段，无版本或时间戳。

### 10.8 DispatchTracker 大量 in-flight 时 UI 优化

**当前实现状态：完全不存在**

Miniflux 没有 in-flight 请求追踪或 DispatchTracker：

```go
// handler.go:338
go integration.PushEntries(feed, newEntries, userIntegrations)
// 启动 goroutine 后立即返回，没有追踪
// 没有：记录 in-flight 计数
// 没有：等待完成或超时控制
// 没有：上下文取消传播
```

**缺失的能力**：
- 没有 `DispatchTracker` 结构追踪每个推送请求的状态
- 没有 in-flight 计数或并发限制
- 没有上下文取消（`context.Context`）传播
- 没有 UI 展示推送状态（排队中/发送中/成功/失败）
- 没有推送速率限制或熔断机制
- 没有推送历史记录和搜索

**现有的相关监控**（仅供参考）：
- Prometheus 指标 `BackgroundFeedRefreshDuration`：追踪 Feed 刷新耗时
- `brokenFeedsGauge`：失败 Feed 数量
- 但没有任何推送相关的指标

**可参考的 in-flight 处理模式**（Web Session）：
```go
// web_session_test.go:108
// Rotate must preserve the CSRF token so in-flight forms remain valid
t.Error("Rotate must preserve the CSRF token so in-flight forms remain valid")
```
Session 轮换时保留 CSRF token 以确保进行中的表单仍然有效，但这是唯一涉及 "in-flight" 概念的代码。

## 十一、架构总结（完整版）

### 11.1 设计优点

1. **解耦清晰**：Feed 刷新与集成推送完全分离，通过 goroutine 异步执行
2. **容错性好**：单个渠道或单个 Feed 失败不影响整体系统
3. **配置灵活**：每个用户可独立配置多个渠道，支持 Feed 级 Webhook 覆盖
4. **背压机制**：Worker 池 + 无缓冲 channel 实现自然限流
5. **安全考虑**：Webhook 签名验证、私有网络访问控制
6. **安全哈希**：条目基于 FNV-128a 哈希去重，Webhook 使用 HMAC-SHA256 防伪造
7. **调度智能**：支持基于条目频率的自适应调度间隔
8. **碰撞兜底**：RSS GUID 重复时有分级 fallback 策略（GUID → GUID+URL → GUID+position）
9. **数据库保障**：`(feed_id, hash)` 唯一索引从存储层面防止重复

### 11.2 潜在改进点（已验证的缺失功能，共18项）

1. **推送无重试**：集成推送失败后不会重试，可能导致消息丢失
2. **推送无队列**：大量 Feed 同时刷新时，可能产生大量推送 goroutine
3. **无速率限制**：对第三方 API 的调用速率没有限制（如 Telegram 的每分钟消息限制）
4. **批量不一致**：Webhook 批量推送 vs Telegram/Slack 单条推送，策略不统一
5. **无动态扩展**：添加新渠道需要修改核心代码，不支持插件
6. **无消息模板**：推送格式硬编码，用户无法自定义
7. **无 DLQ**：失败消息无法追溯和重放
8. **无优先级队列**：高优先级消息无法插队处理
9. **无推送去重**：多渠道同时启用时同一条消息会重复推送
10. **无进度可视化**：没有推送状态追踪和 UI 展示
11. **无插件接口**：没有 `IntegrationPluginInterface` 或插件市场
12. **无推送 i18n**：推送消息不支持国际化
13. **无 DLQ 重投策略**：没有指数退避、最大重试次数等策略
14. **无优先级反转检测**：没有 PriorityChannel 或优先级继承
15. **无主动碰撞检测**：FNV-128a 哈希碰撞时没有告警
16. **无自适应批组合并**：没有 BatchCoalescer 或滑动窗口合并
17. **无 HMAC 密钥轮换**：Webhook 密钥无法安全轮换
18. **无 DispatchTracker**：没有 in-flight 请求追踪和 UI 优化

### 11.3 关键代码位置（完整版）

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
| 条目去重 | `internal/storage/entry.go` | 213 (`entryExists`) |
| FNV 哈希 | `internal/crypto/crypto.go` | 18 (`HashFromBytes`) |
| SHA256 哈希 | `internal/crypto/crypto.go` | 26 (`SHA256`) |
| HMAC 签名 | `internal/crypto/crypto.go` | 48 (`GenerateSHA256Hmac`) |
| 恒定时间比较 | `internal/crypto/crypto.go` | 59 (`ConstantTimeCmp`) |
| Ntfy 优先级 | `internal/integration/ntfy/ntfy.go` | 54 (`Priority: c.ntfyPriority`) |
| Pushover 优先级 | `internal/integration/pushover/pushover.go` | 88 (`Priority: c.priority`) |
| 指标监控 | `internal/metric/metric.go` | 24 (`BackgroundFeedRefreshDuration`) |
| GUID 碰撞兜底 | `internal/reader/rss/adapter.go` | 116-139 |
| 自适应调度 | `internal/model/feed.go` | 122 (`ScheduleNextCheck`) |
| Session 轮换 | `internal/model/web_session.go` | 68 (`Rotate`) |
| i18n 资源 | `internal/locale/catalog.go` | 20 (`embed.FS` 翻译包) |
| 唯一索引 | `internal/database/migrations.go` | 88 (`unique (feed_id, hash)`) |

## 十二、边界场景深度分析

### 12.1 Abandoned Plugin 检测机制

**当前实现状态：不存在专门的 Plugin 机制，但有 Orphan 资源清理**

Miniflux 没有插件系统，因此不存在 abandoned plugin 检测。但存在类似的 **Orphan（孤立）资源清理机制**：

**Orphan Icons 清理**（`internal/storage/icon.go:149-166`）：
```go
// CleanupOrphanIcons removes icons that are no longer associated with any
// feed. Such rows accumulate when feeds are deleted (the cascade only removes
// the feed_icons mapping, not the dedup-by-hash icons row) or when a feed's
// icon is replaced by StoreFeedIcon.
func (s *Storage) CleanupOrphanIcons() (int64, error) {
    result, err := s.db.Exec(`
        DELETE FROM icons
        WHERE NOT EXISTS (
            SELECT 1 FROM feed_icons WHERE feed_icons.icon_id = icons.id
        )
    `)
    n, _ := result.RowsAffected()
    return n, nil
}
```

**清理任务调度**（`internal/cli/cleanup_tasks.go:16-58`）：
```go
func runCleanupTasks(store *storage.Storage) {
    // 1. 清理过期 Web Sessions
    store.CleanOldWebSessions(config.Opts.CleanupRemoveSessionsInterval())
    // 2. 归档已读条目
    store.ArchiveEntries(model.EntryStatusRead, ...)
    // 3. 归档未读条目
    store.ArchiveEntries(model.EntryStatusUnread, ...)
    // 4. 清理孤立图标
    store.CleanupOrphanIcons()
}
```

**已废弃配置检测**（`internal/config/parser.go:163-171`）：
```go
if key == "FILTER_ENTRY_MAX_AGE_DAYS" {
    slog.Warn("Configuration option FILTER_ENTRY_MAX_AGE_DAYS is deprecated; use user filter rule max-age:<duration> instead")
}
// Ignore unknown configuration keys to avoid parsing unrelated environment variables.
return nil
```

**时区废弃映射**（`internal/database/migrations.go:1184-1358`）：
```go
// This migration replaces deprecated timezones by their equivalent on Debian Trixie.
var deprecatedTimeZoneMap = map[string]string{
    "Africa/Asmera": "Africa/Asmara",
    "America/Argentina/ComodRivadavia": "America/Argentina/Catamarca",
    "America/Buenos_Aires": "America/Argentina/Buenos_Aires",
    // ... 40+ 个废弃时区映射
}
```

**仍缺失的能力**：
- 没有渠道/集成健康检查（检测不再响应的集成）
- 没有长期未使用的集成配置自动检测
- 没有集成 API 凭证过期提醒

### 12.2 LayeredResolution 热更新层级混乱

**当前实现状态：完全不存在热更新机制**

Miniflux 所有配置和资源都是**启动时一次性加载**，不支持运行时热更新：

```go
// 配置加载：启动时 parse 环境变量，无运行时重载
parser := config.NewConfigParser()
config.Opts, err = parser.ParseEnvironmentVariables()

// i18n 资源：启动时通过 embed.FS 加载到内存
//go:embed translations/*.json
var translationFiles embed.FS

// 集成渠道：编译时静态 import，无法增删
import (
    "miniflux.app/v2/internal/integration/telegrambot"
    "miniflux.app/v2/internal/integration/slack"
    "miniflux.app/v2/internal/integration/webhook"
)
```

**层级混乱风险不存在，但有多层配置覆盖**：
- **Feed 级配置**：每个 Feed 可单独设置 UserAgent、Cookie、Proxy 等（优先级最高）
- **用户级配置**：每个用户可单独设置 Theme、Language、Filter Rules
- **全局配置**：环境变量/配置文件（优先级最低）

**配置覆盖示例**：
```go
// Feed 级 Webhook 覆盖全局 Webhook
// integration.go:542-553
if feed.WebhookURL != "" {
    webhookURL = feed.WebhookURL
}
if feed.WebhookSecret != "" {
    webhookSecret = feed.WebhookSecret
}
```

**缺失的能力**：
- 没有配置热重载（SIGHUP 或 API 触发）
- 没有配置变更版本追踪
- 没有多租户配置隔离冲突检测
- 没有 i18n 翻译包运行时更新

### 12.3 ExponentialBackoff 最大重投次数

**当前实现状态：集成推送无重试，Feed 抓取有错误计数但无指数退避**

**Feed 抓取错误计数**（`internal/model/feed.go:100-110`）：
```go
// 仅线性计数，无指数退避
func (f *Feed) WithTranslatedErrorMessage(message string) {
    f.ParsingErrorCount++          // 每次失败 +1
    f.ParsingErrorMsg = message
}

func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0         // 成功直接归零
    f.ParsingErrorMsg = ""
}
```

**调度器错误过滤**（`internal/cli/scheduler.go:36-42`）：
```go
WithErrorLimit(errorLimit)  // POLLING_PARSING_ERROR_LIMIT = 3（固定阈值）
```

**限流延迟处理**（`internal/reader/handler/handler.go:245-255`）：
```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()  // 只解析 Retry-After
    calculatedNextCheckInterval := originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
}
```

**Retray-After 解析**（`internal/reader/fetcher/response_handler.go:80-96`）：
```go
func (r *ResponseHandler) ParseRetryDelay() time.Duration {
    retryAfterHeaderValue := r.httpResponse.Header.Get("Retry-After")
    // 支持秒数格式
    if seconds, err := strconv.Atoi(retryAfterHeaderValue); err == nil {
        return time.Duration(seconds) * time.Second
    }
    // 支持 HTTP-date 格式
    if t, err := time.Parse(time.RFC1123, retryAfterHeaderValue); err == nil {
        return time.Until(t).Truncate(time.Second)
    }
    return 0
}
```

**缺失的能力**：
- 没有指数退避算法（如 `delay = base * 2^attempt`）
- 没有最大重试次数限制（超过错误计数阈值直接禁用，无重投恢复）
- 没有抖动（Jitter）防止惊群效应
- 没有推送层面的任何重试机制

### 12.4 Priority Inheritance 死锁检测

**当前实现状态：完全不存在**

Miniflux 使用 Go 并发原语，但没有死锁检测或优先级继承：

```go
// Worker 池：简单 goroutine + channel
type Pool struct {
    queue chan model.Job  // 无缓冲 channel，无优先级
    wg    sync.WaitGroup
}

// Feed 处理：无锁设计（每个 Feed 独立处理）
func RefreshFeed(store *storage.Storage, userID, feedID int64, force bool) error {
    // 每个 Feed 独立操作，不需要分布式锁
    // 数据库级别的事务隔离提供一致性
}
```

**并发安全机制**：
- 使用 Go channel 进行 worker 间通信，避免显式锁
- 数据库使用 `BEGIN` 事务处理读写一致性
- 每个 Feed 的刷新是独立的，不同 Feed 之间不共享资源
- 推送使用独立 goroutine，无共享状态

**缺失的能力**：
- 没有 `sync.Mutex` 或 `sync.RWMutex` 的死锁检测
- 没有优先级继承协议（Priority Inheritance Protocol）
- 没有优先级天花板协议（Priority Ceiling Protocol）
- 没有 goroutine 泄漏检测或监控

### 12.5 BloomFilter False Positive 阈值动态调整

**当前实现状态：完全不存在 Bloom Filter**

Miniflux 未使用布隆过滤器，去重完全依赖数据库唯一索引：

```go
// 条目去重：基于 (feed_id, hash) 唯一索引
func (s *Storage) entryExists(tx *sql.Tx, entry *model.Entry) (bool, error) {
    err := tx.QueryRow(`SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1`,
        entry.FeedID, entry.Hash).Scan(&result)
}

// RSS 适配器 GUID 碰撞兜底：哈希算法选择
// - SHA-256（加密安全哈希）用于 RSS GUID 去重
// - FNV-128a（非加密快速哈希）用于存储层条目去重
```

**去重策略对比**：
| 方案 | 假阳性 | 内存 | 速度 | Miniflux 是否使用 |
|------|--------|------|------|-------------------|
| 布隆过滤器 | 可调（如 0.1%） | O(n) 极小 | 极快 | ❌ |
| 数据库唯一索引 | 0（精确） | 数据库存储 | SQL 查询 | ✅ |
| FNV-128a 哈希 | 极低（理论） | 128 bits | 快 | ✅（存储层） |
| SHA-256 哈希 | 可忽略 | 256 bits | 中等 | ✅（RSS 适配层） |

**缺失的能力**：
- 没有内存级快速去重前置过滤
- 没有假阳性阈值动态调整
- 没有计数布隆过滤器支持删除操作
- 没有分层缓存（内存 → 数据库）的去重优化

### 12.6 Token Bucket 极端流量退化

**当前实现状态：不存在 Token Bucket 限流算法，但有被动限流处理**

**被动限流（消费方驱动）**（`internal/reader/fetcher/response_handler.go:98-100`）：
```go
func (r *ResponseHandler) IsRateLimited() bool {
    return r.httpResponse != nil && r.httpResponse.StatusCode == http.StatusTooManyRequests
}
```

**调度层面的限流**（`internal/cli/scheduler.go:36-42`）：
```go
jobs, err := store.NewBatchBuilder().
    WithBatchSize(batchSize).              // BATCH_SIZE = 100
    WithErrorLimit(errorLimit).              // POLLING_PARSING_ERROR_LIMIT = 3
    WithoutDisabledFeeds().
    WithLimitPerHost(limitPerHost).          // POLLING_LIMIT_PER_HOST = 0（不限制）
    WithNextCheckExpired().
    FetchJobs()
```

**Worker 池自然限流**（`internal/worker/pool.go:33-43`）：
```go
// 固定 16 个 worker，超过则阻塞在 channel 写入
func NewPool(store *storage.Storage, nbWorkers int) *Pool {
    queue := make(chan model.Job)  // 无缓冲 channel = 自然背压
    for i := range nbWorkers {     // WORKER_POOL_SIZE = 16
        go worker.Run(queue, &wg)
    }
}
```

**推送层面的限流缺失**：
- 没有客户端侧 Token Bucket 限流
- 没有每渠道 QPS 限制（如 Telegram Bot API 限 30 条/秒）
- 没有突发流量退避策略
- 没有熔断机制（连续失败后暂停推送）

**极端流量下的退化行为**：
- 大量 Feed 同时刷新 → Worker 池满 → channel 阻塞 → 调度器阻塞（可接受）
- 大量新条目产生 → 启动大量推送 goroutine → 内存和 FD 压力增大（潜在风险）
- 第三方 API 限流 → 推送失败 → 日志记录 → 消息丢失（主要风险）

### 12.7 HMAC Dual-Active Period 审计追踪

**当前实现状态：不存在密钥轮换和双活期，无审计日志**

**Webhook HMAC 现状**（`internal/integration/webhook/webhook.go:134`）：
```go
request.Header.Set("X-Miniflux-Signature",
    crypto.GenerateSHA256Hmac(c.webhookSecret, requestBody))
// 只有一个密钥，无版本号，无双活期
```

**可参考的 Session 审计**（`internal/storage/web_session.go:130-159`）：
```go
func (s *Storage) RotateWebSession(oldID string, session *model.WebSession) error {
    err = s.db.QueryRow(`
        UPDATE web_sessions
        SET id=$2, secret_hash=$3, user_id=$4, state=$5, created_at=now()
        WHERE id=$1
        RETURNING created_at
    `, oldID, session.ID, session.SecretHash, ...).Scan(&session.CreatedAt)
}
```

**缺失的 HMAC 安全能力**：
- 没有 `X-Miniflux-Key-Version` 头标识密钥版本
- 没有双活期（Dual-Active Period）支持新旧密钥同时验证
- 没有密钥轮换操作日志
- 没有 Webhook 请求审计追踪（请求体、时间、结果）
- 没有 `crypto/rand` 强制生成密钥的校验

**数据库存储现状**：
- `integrations` 表中 `webhook_secret` 为普通 `text` 字段
- 无 `webhook_secret_version`、`webhook_secret_rotated_at`、`webhook_secret_expires_at` 字段
- 无 `webhook_audit_log` 表记录每次 Webhook 请求

### 12.8 Virtualized List 滚动跳转精确定位

**当前实现状态：实现了 DOM 列表滚动定位，但不是真正的虚拟列表**

**滚动定位函数**（`internal/ui/static/js/app.js:50-64`）：
```javascript
function scrollPageTo(element, evenIfOnScreen) {
    const windowScrollPosition = window.scrollY;
    const windowHeight = document.documentElement.clientHeight;
    const viewportPosition = windowScrollPosition + windowHeight;
    const itemBottomPosition = element.offsetTop + element.offsetHeight;

    if (evenIfOnScreen || viewportPosition - itemBottomPosition < 0 || viewportPosition - element.offsetTop > windowHeight) {
        window.scrollTo(0, element.offsetTop - 10);
    }
}
```

**列表项跳转**（`internal/ui/static/js/app.js:389-435`）：
```javascript
function goToListItem(offset) {
    const items = getVisibleEntries();  // 获取所有可见 .items .item
    const currentItem = document.querySelector(".current-item");
    const currentIndex = items.indexOf(currentItem);

    // 支持 TOP（首页）/ BOTTOM（末页）/ 数字偏移
    let newIndex;
    if (offset === TOP) {
        newIndex = 0;
    } else if (offset === BOTTOM) {
        newIndex = items.length - 1;
    } else {
        newIndex = (currentIndex + offset + items.length) % items.length;  // 环形循环
    }

    // 切换选中状态并滚动
    currentItem.classList.remove("current-item");
    newItem.classList.add("current-item");
    newItem.focus();
    scrollPageTo(newItem);
}
```

**导航入口**（`internal/ui/static/js/app.js:294-332`）：
```javascript
function goToPreviousPage(offset) {
    if (isListView()) {
        goToListItem(offset);  // 列表视图：在条目间跳转
    } else {
        goToPage("previous");   // 条目视图：翻页
    }
}
```

**真实虚拟列表缺失**：
- 没有 `virtualized-list` 组件（GitHub 使用的那种）
- 没有 DOM 回收（只渲染视口内元素）
- 没有行高预估和动态调整
- 没有滚动到指定 ID 或日期的精确定位（只能按偏移量跳转）
- 没有大数据量（万条以上）性能优化

**当前实现的局限**：
- 一次渲染所有条目，条目数多时 DOM 节点过多影响性能
- 使用 `offsetTop` 计算位置，嵌套布局下可能不准确
- 预留 10px 顶部间距，无动态计算 header 高度

## 十三、架构总结（最终版）

### 13.1 设计优点

1. **解耦清晰**：Feed 刷新与集成推送完全分离，通过 goroutine 异步执行
2. **容错性好**：单个渠道或单个 Feed 失败不影响整体系统
3. **配置灵活**：每个用户可独立配置多个渠道，支持 Feed 级 Webhook 覆盖
4. **背压机制**：Worker 池 + 无缓冲 channel 实现自然限流
5. **安全考虑**：Webhook 签名验证、私有网络访问控制
6. **安全哈希**：条目基于 FNV-128a 哈希去重，Webhook 使用 HMAC-SHA256 防伪造
7. **调度智能**：支持基于条目频率的自适应调度间隔
8. **碰撞兜底**：RSS GUID 重复时有分级 fallback 策略（GUID → GUID+URL → GUID+position）
9. **数据库保障**：`(feed_id, hash)` 唯一索引从存储层面防止重复
10. **资源清理**：Orphan Icons、过期 Session、旧条目归档定期清理
11. **废弃兼容**：配置项废弃有警告日志，时区废弃有自动迁移映射

### 13.2 潜在改进点（已验证的缺失功能，共26项）

1. **推送无重试**：集成推送失败后不会重试，可能导致消息丢失
2. **推送无队列**：大量 Feed 同时刷新时，可能产生大量推送 goroutine
3. **无速率限制**：对第三方 API 的调用速率没有限制（如 Telegram 的每分钟消息限制）
4. **批量不一致**：Webhook 批量推送 vs Telegram/Slack 单条推送，策略不统一
5. **无动态扩展**：添加新渠道需要修改核心代码，不支持插件
6. **无消息模板**：推送格式硬编码，用户无法自定义
7. **无 DLQ**：失败消息无法追溯和重放
8. **无优先级队列**：高优先级消息无法插队处理
9. **无推送去重**：多渠道同时启用时同一条消息会重复推送
10. **无进度可视化**：没有推送状态追踪和 UI 展示
11. **无插件接口**：没有 `IntegrationPluginInterface` 或插件市场
12. **无推送 i18n**：推送消息不支持国际化
13. **无 DLQ 重投策略**：没有指数退避、最大重试次数等策略
14. **无优先级反转检测**：没有 PriorityChannel 或优先级继承
15. **无主动碰撞检测**：FNV-128a 哈希碰撞时没有告警
16. **无自适应批组合并**：没有 BatchCoalescer 或滑动窗口合并
17. **无 HMAC 密钥轮换**：Webhook 密钥无法安全轮换
18. **无 DispatchTracker**：没有 in-flight 请求追踪和 UI 优化
19. **无集成健康检查**：没有检测不再响应的集成配置
20. **无配置热重载**：配置变更需重启服务
21. **无指数退避重试**：失败重试是固定阈值，无抖动和退避
22. **无死锁检测**：并发原语无监控
23. **无布隆过滤器**：内存级快速去重缺失
24. **无客户端限流**：无 Token Bucket 控制第三方 API 调用速率
25. **无密钥审计**：HMAC 操作无日志追踪
26. **无虚拟列表**：前端大数据量渲染性能未优化

### 13.3 关键代码位置（最终版，31个模块）

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
| 条目去重 | `internal/storage/entry.go` | 213 (`entryExists`) |
| FNV 哈希 | `internal/crypto/crypto.go` | 18 (`HashFromBytes`) |
| SHA256 哈希 | `internal/crypto/crypto.go` | 26 (`SHA256`) |
| HMAC 签名 | `internal/crypto/crypto.go` | 48 (`GenerateSHA256Hmac`) |
| 恒定时间比较 | `internal/crypto/crypto.go` | 59 (`ConstantTimeCmp`) |
| Ntfy 优先级 | `internal/integration/ntfy/ntfy.go` | 54 (`Priority: c.ntfyPriority`) |
| Pushover 优先级 | `internal/integration/pushover/pushover.go` | 88 (`Priority: c.priority`) |
| 指标监控 | `internal/metric/metric.go` | 24 (`BackgroundFeedRefreshDuration`) |
| GUID 碰撞兜底 | `internal/reader/rss/adapter.go` | 116-139 |
| 自适应调度 | `internal/model/feed.go` | 122 (`ScheduleNextCheck`) |
| Session 轮换 | `internal/model/web_session.go` | 68 (`Rotate`) |
| i18n 资源 | `internal/locale/catalog.go` | 20 (`embed.FS` 翻译包) |
| 唯一索引 | `internal/database/migrations.go` | 88 (`unique (feed_id, hash)`) |
| Orphan 清理 | `internal/storage/icon.go` | 149 (`CleanupOrphanIcons`) |
| 定期清理任务 | `internal/cli/cleanup_tasks.go` | 16 (`runCleanupTasks`) |
| 配置废弃警告 | `internal/config/parser.go` | 166 (`slog.Warn deprecated`) |
| 时区废弃映射 | `internal/database/migrations.go` | 1184 (`deprecatedTimeZoneMap`) |
| 限流检测 | `internal/reader/fetcher/response_handler.go` | 98 (`IsRateLimited`) |
| Retry-After 解析 | `internal/reader/fetcher/response_handler.go` | 80 (`ParseRetryDelay`) |
| 前端滚动定位 | `internal/ui/static/js/app.js` | 55 (`scrollPageTo`) |
| 列表项跳转 | `internal/ui/static/js/app.js` | 389 (`goToListItem`) |
| Session 数据库轮换 | `internal/storage/web_session.go` | 130 (`RotateWebSession`) |

## 十四、边界场景深度分析（续）

### 14.1 Staleness 阈值的运维侧配置入口

**已实现完整的配置体系，环境变量驱动**

Miniflux 通过环境变量配置多维度的 staleness（过期）阈值，运维侧可通过容器环境变量或配置文件注入：

**配置定义**（`internal/config/options.go:125-155`）：
```go
"CLEANUP_ARCHIVE_BATCH_SIZE": {
    parsedIntValue: 10000,          // 每次归档批大小
    rawValue:       "10000",
    valueType:      intType,
    validator:      func(rawValue) error { return validateGreaterOrEqualThan(rawValue, 1) },
},
"CLEANUP_ARCHIVE_READ_DAYS": {
    parsedDuration: time.Hour * 24 * 60,   // 已读条目 60 天后归档
    rawValue:       "60",
    valueType:      dayType,
},
"CLEANUP_ARCHIVE_UNREAD_DAYS": {
    parsedDuration: time.Hour * 24 * 180,  // 未读条目 180 天后归档
    rawValue:       "180",
    valueType:      dayType,
},
"CLEANUP_FREQUENCY_HOURS": {
    parsedDuration: time.Hour * 24,        // 每 24 小时执行一次清理
    rawValue:       "24",
    valueType:      hourType,
},
"CLEANUP_REMOVE_SESSIONS_DAYS": {
    parsedDuration: time.Hour * 24 * 30,   // 30 天未使用的 Session 过期
    rawValue:       "30",
    valueType:      dayType,
},
```

**配置访问方法**：
```go
// 获取方法在 options.go:652-660
func (c *configOptions) CleanupArchiveBatchSize() int { return c.options["CLEANUP_ARCHIVE_BATCH_SIZE"].parsedIntValue }
func (c *configOptions) CleanupArchiveReadInterval() time.Duration { return c.options["CLEANUP_ARCHIVE_READ_DAYS"].parsedDuration }
func (c *configOptions) CleanupArchiveUnreadInterval() time.Duration { return c.options["CLEANUP_ARCHIVE_UNREAD_DAYS"].parsedDuration }
func (c *configOptions) CleanupFrequency() time.Duration { return c.options["CLEANUP_FREQUENCY_HOURS"].parsedDuration }
func (c *configOptions) CleanupRemoveSessionsInterval() time.Duration { return c.options["CLEANUP_REMOVE_SESSIONS_DAYS"].parsedDuration }
```

**归档执行 + Tombstone 防重摄入**（`internal/storage/entry.go:362-404`）：
```go
func (s *Storage) ArchiveEntries(status string, interval time.Duration, limit int) (int64, error) {
    query := `
        WITH to_delete AS (
            SELECT id, feed_id, hash
            FROM entries
            WHERE
                status=$1 AND
                starred is false AND          -- 星标条目永不过期
                share_code='' AND             -- 有分享码的条目永不过期
                created_at < now() - $2::interval  -- 超过 staleness 阈值
            ORDER BY created_at ASC
            FOR UPDATE SKIP LOCKED           -- 跳过锁定行避免死锁
            LIMIT $3
        ), deleted AS (
            DELETE FROM entries
            USING to_delete
            WHERE entries.id = to_delete.id
            RETURNING entries.feed_id, entries.hash
        )
        INSERT INTO entry_tombstones (feed_id, hash)  -- 写入墓碑防止重新摄入
        SELECT feed_id, hash FROM deleted WHERE hash <> ''
        ON CONFLICT (feed_id, hash) DO NOTHING
    `
}
```

**运维配置入口汇总**：
| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `CLEANUP_FREQUENCY_HOURS` | 24h | 清理任务执行频率 |
| `CLEANUP_ARCHIVE_READ_DAYS` | 60天 | 已读条目 staleness 阈值 |
| `CLEANUP_ARCHIVE_UNREAD_DAYS` | 180天 | 未读条目 staleness 阈值 |
| `CLEANUP_ARCHIVE_BATCH_SIZE` | 10000 | 每次归档批大小 |
| `CLEANUP_REMOVE_SESSIONS_DAYS` | 30天 | Web Session staleness 阈值 |
| `POLLING_PARSING_ERROR_LIMIT` | 3 | Feed 抓取失败禁用阈值 |

### 14.2 ResourceBundle.SyncVersion 回滚路径

**当前实现：数据库 Schema 单向升级，无回滚路径**

**Schema 版本管理**（`internal/database/migrations.go:13`）：
```go
var schemaVersion = len(migrations)  // 版本号 = 迁移数组长度，单向递增
```

**升级逻辑**（`internal/database/database.go:13-51`）：
```go
func Migrate(db *sql.DB) error {
    var currentVersion int
    db.QueryRow(`SELECT version FROM schema_version`).Scan(&currentVersion)

    for version := currentVersion; version < schemaVersion; version++ {
        newVersion := version + 1

        tx, err := db.Begin()

        if err := migrations[version](tx); err != nil {   // 执行单个迁移
            tx.Rollback()                                  // 失败回滚事务
            return fmt.Errorf("[Migration v%d] %v", newVersion, err)
        }

        tx.Exec(`TRUNCATE schema_version`)
        tx.Exec(`INSERT INTO schema_version (version) VALUES ($1)`, newVersion)
        tx.Commit()
    }
}
```

**启动时版本检查**（`database.go:53-60`）：
```go
func IsSchemaUpToDate(db *sql.DB) error {
    var currentVersion int
    db.QueryRow(`SELECT version FROM schema_version`).Scan(&currentVersion)
    if currentVersion < schemaVersion {
        return fmt.Errorf(`the database schema is not up to date: current=v%d expected=v%d`,
            currentVersion, schemaVersion)
    }
    return nil  // 只检查版本不低于，不回滚
}
```

**回滚能力缺失清单**：
- 没有 `migrations_down` 反向迁移数组
- 没有 `rollback` CLI 命令（仅有 `-migrate` 正向迁移）
- 没有 Schema 版本快照备份机制
- 没有降级启动（旧版二进制 + 新版 Schema）的兼容层
- `schema_version` 表只有 `version` 字段，无 `applied_at`、`rollback_sql` 等元数据

**唯一的"回滚"方式**：备份数据库 → 升级失败 → 恢复备份

### 14.3 重投上限自适应策略

**当前实现：固定阈值线性计数，无自适应调整**

**Feed 抓取错误计数**（`internal/model/feed.go:100-110`）：
```go
// 固定阈值 3，无自适应
func (f *Feed) WithTranslatedErrorMessage(message string) {
    f.ParsingErrorCount++          // 线性递增，无加权
    f.ParsingErrorMsg = message
}

func (f *Feed) ResetErrorCounter() {
    f.ParsingErrorCount = 0         // 成功直接归零，无滑动窗口
    f.ParsingErrorMsg = ""
}
```

**调度器错误过滤**（`internal/cli/scheduler.go:36-42`）：
```go
WithErrorLimit(errorLimit)
// errorLimit = config.Opts.PollingParsingErrorLimit() = 3（固定）
```

**调度频率自适应（仅轮询间隔）**（`internal/model/feed.go:122-149`）：
```go
func (f *Feed) ScheduleNextCheck(weeklyCount int, refreshDelay time.Duration) time.Duration {
    // 只有 Entry Frequency 模式下根据条目频率动态调整间隔
    if config.Opts.PollingScheduler() == SchedulerEntryFrequency {
        interval = (7 * 24 * time.Hour) / time.Duration(weeklyCount*factor)
        interval = min(interval, maxInterval)
        interval = max(interval, minInterval)
    }
    // 但重试上限（错误计数阈值）不调整
}
```

**缺失的自适应能力**：
- 没有根据历史失败率动态调整 `ParsingErrorLimit`
- 没有指数退避（`delay = base * 2^attempt`）
- 没有抖动（Jitter）防止惊群
- 没有按失败原因（网络超时 / 403 / 解析错误）分类的差异化策略
- 推送层面没有任何重试，更谈不上自适应

**对比：各层面重试策略现状**：
| 层面 | 重试机制 | 上限 | 自适应 |
|------|----------|------|--------|
| Feed 轮询 | 线性计数 + 重试时间窗口 | POLLING_PARSING_ERROR_LIMIT=3（固定） | 仅轮询间隔自适应，上限不调整 |
| HTTP 抓取 | 支持 Retry-After 头 | 无 | 依赖服务端响应 |
| 集成推送 | 无 | - | - |
| 数据库事务 | 单事务内自动回滚 | 立即失败上报 | 不重试 |

### 14.4 Wait-For Graph 死锁检测周期

**当前实现：完全不存在死锁检测机制**

Miniflux 的并发设计基于"尽量避免使用锁"的哲学，因此没有实现 Wait-For Graph：

**并发设计模式**：
```go
// 1. Worker 池：channel + goroutine，无显式锁
type Pool struct {
    queue chan model.Job  // 无缓冲 channel
    wg    sync.WaitGroup
}

// 2. 数据库级并发：PostgreSQL MVCC + FOR UPDATE SKIP LOCKED
WITH to_delete AS (
    SELECT id FROM entries ...
    FOR UPDATE SKIP LOCKED  -- 跳过锁定行，不等锁 = 天然避免死锁
)

// 3. 推送并发：独立 goroutine 独立状态
go integration.PushEntries(feed, newEntries, userIntegrations)  // 每 Feed 独立，无共享状态
```

**缺失的死锁治理能力**：
- 没有 `sync.Mutex` 锁等待图（WFG）构建
- 没有周期检测（如每 10 秒扫描 goroutine stack trace）
- 没有死锁检测超时机制
- 没有 goroutine 泄漏监控（pprof 除外，需手动调用）
- 没有数据库死锁监控（PostgreSQL `pg_stat_activity` 除外，需 DBA 手动检查）

**唯一的隐式保护**：`FOR UPDATE SKIP LOCKED` 在批量操作中避免了数据库级别的行锁等待，消除了最常见的死锁来源。

### 14.5 BloomFilter 阈值上下界

**当前实现：完全不存在布隆过滤器**

Miniflux 使用多层精确去重策略，不使用概率数据结构：

**去重层级**：
```
层1: RSS 适配层 GUID 碰撞兜底 (SHA-256)
  ↓
层2: 存储层条目哈希 (FNV-128a)
  ↓
层3: 数据库唯一索引 (feed_id, hash)
  ↓
层4: 归档墓碑 entry_tombstones (归档后防止重摄入)
```

**条目去重检查**（`internal/storage/entry.go:213-231`）：
```go
func (s *Storage) entryExists(tx *sql.Tx, entry *model.Entry) (bool, error) {
    err := tx.QueryRow(`SELECT true FROM entries WHERE feed_id=$1 AND hash=$2 LIMIT 1`,
        entry.FeedID, entry.Hash).Scan(&result)
}
```

**墓碑防止重摄入**（`entry.go:386-388`）：
```sql
INSERT INTO entry_tombstones (feed_id, hash)
SELECT feed_id, hash FROM deleted WHERE hash <> ''
ON CONFLICT (feed_id, hash) DO NOTHING
```

**不存在的能力**：
- 没有内存级布隆过滤器快速过滤
- 没有假阳性（False Positive）概率参数配置
- 没有上下界动态调整（根据内存压力 / 条目数量）
- 没有计数布隆过滤器（支持删除操作）
- 没有分层布隆（ColdFilter + HotFilter）架构

**设计权衡分析**：

| 维度 | 布隆过滤器方案 | Miniflux 当前方案（精确去重） |
|------|---------------|-------------------------------|
| 假阳性 | 0.01%~1% 可调 | 0（精确） |
| 内存占用 | O(n) 极小（万级条目 < 100KB） | 数据库 B-Tree 索引 |
| 查询速度 | 内存访问 ~100ns | SQL 查询 ~1ms |
| 删除支持 | 需计数布隆（2-4x内存） | 原生支持 |
| 部署复杂度 | 需配置参数 | 零配置 |
| 假阳性后果 | 新条目被误判为重复而丢弃 | 无 |

Miniflux 选择零配置 + 零假阳性的精确方案，牺牲部分性能换取可靠性。

### 14.6 Leaky Bucket 恢复到 Token Bucket 的时机

**当前实现：不存在 Bucket 限流算法体系**

Miniflux 采用三层被动限流，没有主动的 Token Bucket 或 Leaky Bucket，因此不存在"降级→恢复"状态机：

**三层被动限流**：
```
层1: Worker 池固定大小 = 16（自然背压，不区分桶类型）
  ↓
层2: 调度器 BATCH_SIZE = 100 + LIMIT_PER_HOST = 0（可配置每主机限制）
  ↓
层3: 被动检测 HTTP 429 + 解析 Retry-After
```

**429 响应处理**（`internal/reader/fetcher/response_handler.go:98-100`）：
```go
func (r *ResponseHandler) IsRateLimited() bool {
    return r.httpResponse != nil &&
        r.httpResponse.StatusCode == http.StatusTooManyRequests
}
```

**恢复时机（隐式）**（`internal/reader/handler/handler.go:245-255`）：
```go
if responseHandler.IsRateLimited() {
    retryDelay := responseHandler.ParseRetryDelay()
    // 将下一次检查时间推迟 Retry-After 指定的时间
    originalFeed.ScheduleNextCheck(weeklyEntryCount, retryDelay)
    // 经过 retryDelay 后，系统自然恢复正常调度
    // 无需状态转换，因为没有"限流状态"的概念
}
```

**不存在的能力**：
- 没有 Token Bucket → Leaky Bucket 的平滑降级
- 没有 Leaky Bucket → Token Bucket 的自动恢复判断
- 没有 429 计数 / 成功率的状态机转换阈值
- 没有半开状态（Half-Open）探测恢复
- 没有熔断机制（连续失败后暂停一段时间）

**隐含的恢复逻辑**：
- 每次调度独立判断，不保留历史状态
- `Retry-After` 超时后，下次调度与正常 Feed 无区别
- 错误计数 `ParsingErrorCount` 只有成功时清零，与限流状态不关联

### 14.7 密钥泄露后的 KeyId 回收流程

**当前实现：API Key 可即时删除，但无版本号和泄露审计**

**API Key 生命周期管理**（`internal/storage/api_key.go`）：

**创建**（`api_key.go:73-101`）：
```go
func (s *Storage) CreateAPIKey(userID int64, description string) (*model.APIKey, error) {
    query := `INSERT INTO api_keys (user_id, token, description)
              VALUES ($1, $2, $3) RETURNING ...`
    return s.db.QueryRow(
        query,
        userID,
        crypto.GenerateRandomStringHex(32),  // 256-bit 随机 hex
        description,
    )
}
```

**删除（回收）**（`api_key.go:103-119`）：
```go
func (s *Storage) DeleteAPIKey(userID, keyID int64) error {
    result, err := s.db.Exec(`DELETE FROM api_keys WHERE id = $1 AND user_id = $2`,
        keyID, userID)
    count, _ := result.RowsAffected()
    if count == 0 {
        return ErrAPIKeyNotFound
    }
    return nil
}
```

**Web Session 紧急轮换**（`internal/storage/web_session.go:130-159`）：
```go
func (s *Storage) RotateWebSession(oldID string, session *model.WebSession) error {
    err = s.db.QueryRow(`
        UPDATE web_sessions
        SET id=$2, secret_hash=$3, user_id=$4, state=$5, created_at=now()
        WHERE id=$1
        RETURNING created_at
    `, oldID, session.ID, session.SecretHash, ...).Scan(&session.CreatedAt)
}
```

**回收能力对比**：
| 密钥类型 | 删除/回收 | KeyId 版本 | 泄露审计日志 | 双活过渡期 |
|----------|-----------|------------|--------------|------------|
| API Key | ✅ `DELETE FROM api_keys` | ❌ 无 KeyId 版本号 | ❌ 无删除审计 | ❌ 立即失效 |
| Web Session | ✅ `Rotate()` 或过期清理 | ❌ 轮换但无版本 | ❌ 无轮换审计 | ❌ 新 ID 替换旧 ID |
| Webhook Secret | ⚠️ 通过集成页面更新值 | ❌ 无版本 | ❌ 无变更日志 | ❌ 立即用新密钥签名 |
| WebAuthn Credentials | ✅ `DeleteCredential` | ❌ 无版本 | ❌ 无 | ❌ 立即失效 |

**UI 回收入口**（`internal/ui/ui.go:139`）：
```go
mux.HandleFunc("POST /keys/{keyID}/delete", handler.deleteAPIKey)
// 路由：{keyID} 是自增 ID，不是 token 值
```

**缺失的企业级能力**：
- 没有 KeyId 前缀标识（如 `mk_live_xxx` / `mk_test_xxx`）
- 没有泄露事件后的批量吊销
- 没有密钥使用审计（创建时间、最后使用时间、使用次数、IP 来源）
- 没有 HMAC 密钥版本号（无法判断签名是旧密钥还是新密钥）
- 没有主动令牌扫描（检测提交到 GitHub 的泄露密钥）

### 14.8 Virtualized List 滚动跳转动画

**当前实现：滚动跳转无动画，但存在其他动画系统**

**滚动跳转实现**（`internal/ui/static/js/app.js:50-64`）：
```javascript
function scrollPageTo(element, evenIfOnScreen) {
    const windowScrollPosition = window.scrollY;
    const windowHeight = document.documentElement.clientHeight;
    const viewportPosition = windowScrollPosition + windowHeight;
    const itemBottomPosition = element.offsetTop + element.offsetHeight;

    if (evenIfOnScreen ||
        viewportPosition - itemBottomPosition < 0 ||       // 元素底部超出视口
        viewportPosition - element.offsetTop > windowHeight) {  // 元素顶部超出视口
        window.scrollTo(0, element.offsetTop - 10);  // ⚡ 瞬时跳转，无过渡动画
    }
}
```

**动画系统现状**（分散在 UI 各模块）：

**Toast 通知动画**（`app.js:269-275` + `common.css:296-297`）：
```javascript
toastElementWrapper.addEventListener("animationend", () => {
    toastElementWrapper.remove();
});
setTimeout(() => toastElementWrapper.classList.add("toast-animate"), 100);
```
```css
.toast-animate {
    animation: toastKeyFrames 2s;  /* 2秒渐入渐出 */
}
```

**触摸滑动动画**（`touch_handler.js:37-88`）：
```javascript
onItemTouchStart(event) {
    this.touch.element.style.transitionDuration = "0s";  // 跟随手指移动，无过渡
}
onItemTouchMove(event) {
    const tx = (absDistance > 75 ? Math.sqrt(absDistance - 75) + 75 : absDistance) * Math.sign(distance);
    this.touch.element.style.transform = `translateX(${tx}px)`;  // 阻尼效果
}
onItemTouchEnd(event) {
    if (Math.abs(this.calculateDistance()) > 75) {
        toggleEntryStatus(this.touch.element);  // 达到阈值切换状态
    }
    if (this.touch.moved) {
        this.touch.element.style.transitionDuration = "0.15s";  // 松手 150ms 回弹
        this.touch.element.style.transform = "none";
    }
}
```

**页面过渡动画**（`common.css:85-92`）：
```css
/* Smoother pages transition */
@view-transition {
    navigation: auto;
}
@view-transition {
    navigation: cross-fade;
}
```

**滚动动画缺失清单**：
- 没有 `window.scrollTo({top, behavior: "smooth"})` 平滑滚动
- 没有键盘跳转的视觉过渡（当前项 → 目标项）
- 没有滚动到顶部/底部的弹性动效
- 没有骨架屏加载动画
- 没有进入视口的渐入动画（IntersectionObserver）

**阻尼函数分析**（触摸滑动）：
```
absDistance ≤ 75px:  tx = absDistance                // 线性跟随
absDistance > 75px:  tx = sqrt(absDistance - 75) + 75  // 亚线性阻尼，越拉越难
```
这种设计防止用户无限制滑动，同时提供超阈值后的"弹性"反馈。

## 十五、架构总结（终极版）

### 15.1 设计优点（15项）

1. **解耦清晰**：Feed 刷新与集成推送完全分离，通过 goroutine 异步执行
2. **容错性好**：单个渠道或单个 Feed 失败不影响整体系统
3. **配置灵活**：每个用户可独立配置多个渠道，支持 Feed 级 Webhook 覆盖
4. **背压机制**：Worker 池 + 无缓冲 channel 实现自然限流
5. **安全考虑**：Webhook 签名验证、私有网络访问控制
6. **安全哈希**：条目基于 FNV-128a 哈希去重，Webhook 使用 HMAC-SHA256 防伪造
7. **调度智能**：支持基于条目频率的自适应调度间隔（Round-Robin / Entry Frequency）
8. **碰撞兜底**：RSS GUID 重复时有三级 fallback 策略
9. **数据库保障**：`(feed_id, hash)` 唯一索引从存储层面防止重复
10. **资源清理**：Orphan Icons、过期 Session、旧条目归档定期清理
11. **废弃兼容**：配置项废弃有警告日志，时区废弃有自动迁移映射
12. **运维友好**：staleness 阈值全量环境变量配置，无需改源码
13. **墓碑机制**：条目归档后写入 `entry_tombstones` 防止重新摄入
14. **死锁避免**：`FOR UPDATE SKIP LOCKED` 跳过锁定行，天然避免数据库死锁
15. **零配置去重**：精确去重多层兜底，零假阳性风险

### 15.2 潜在改进点（已验证的缺失功能，共34项）

1. **推送无重试**：集成推送失败后不会重试，可能导致消息丢失
2. **推送无队列**：大量 Feed 同时刷新时，可能产生大量推送 goroutine
3. **无速率限制**：对第三方 API 的调用速率没有限制（如 Telegram 的每分钟消息限制）
4. **批量不一致**：Webhook 批量推送 vs Telegram/Slack 单条推送，策略不统一
5. **无动态扩展**：添加新渠道需要修改核心代码，不支持插件
6. **无消息模板**：推送格式硬编码，用户无法自定义
7. **无 DLQ**：失败消息无法追溯和重放
8. **无优先级队列**：高优先级消息无法插队处理
9. **无推送去重**：多渠道同时启用时同一条消息会重复推送
10. **无进度可视化**：没有推送状态追踪和 UI 展示
11. **无插件接口**：没有 `IntegrationPluginInterface` 或插件市场
12. **无推送 i18n**：推送消息不支持国际化
13. **无 DLQ 重投策略**：没有指数退避、最大重试次数等策略
14. **无优先级反转检测**：没有 PriorityChannel 或优先级继承
15. **无主动碰撞检测**：FNV-128a 哈希碰撞时没有告警
16. **无自适应批组合并**：没有 BatchCoalescer 或滑动窗口合并
17. **无 HMAC 密钥轮换**：Webhook 密钥无法安全轮换
18. **无 DispatchTracker**：没有 in-flight 请求追踪和 UI 优化
19. **无集成健康检查**：没有检测不再响应的集成配置
20. **无配置热重载**：配置变更需重启服务
21. **无指数退避重试**：失败重试是固定阈值，无抖动和退避
22. **无死锁检测**：并发原语无监控，仅靠 SKIP LOCKED 避免
23. **无布隆过滤器**：内存级快速去重缺失（选择精确方案替代）
24. **无客户端限流**：无 Token Bucket 控制第三方 API 调用速率
25. **无密钥审计**：HMAC 操作无日志追踪，API Key 无版本管理
26. **无虚拟列表**：前端大数据量渲染性能未优化
27. **无 Schema 回滚**：数据库迁移单向，失败需恢复备份
28. **无重试上限自适应**：`ParsingErrorLimit=3` 固定，不根据历史调整
29. **无布隆过滤器动态阈值**：概率去重方案不存在
30. **无限流状态机**：Token/Leaky Bucket 状态转换缺失
31. **无密钥版本管理**：HMAC/Webhook Secret 无 KeyId 标识和双活过渡期
32. **无滚动平滑动画**：键盘跳转 `window.scrollTo` 瞬时，无过渡
33. **无密钥泄露审计**：API Key 删除/使用无详细日志
34. **无半开状态探测**：限流恢复无渐进式探测验证

### 15.3 关键代码位置（终极版，41个模块）

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
| 错误计数 | `internal/model/feed.go` | 100 (`WithTranslatedErrorMessage`) |
| 调度间隔 | `internal/model/feed.go` | 122 (`ScheduleNextCheck`) |
| 条目去重 | `internal/storage/entry.go` | 213 (`entryExists`) |
| 条目归档+墓碑 | `internal/storage/entry.go` | 362 (`ArchiveEntries`) |
| FNV 哈希 | `internal/crypto/crypto.go` | 18 (`HashFromBytes`) |
| SHA256 哈希 | `internal/crypto/crypto.go` | 26 (`SHA256`) |
| HMAC 签名 | `internal/crypto/crypto.go` | 48 (`GenerateSHA256Hmac`) |
| 恒定时间比较 | `internal/crypto/crypto.go` | 59 (`ConstantTimeCmp`) |
| Ntfy 优先级 | `internal/integration/ntfy/ntfy.go` | 54 (`Priority: c.ntfyPriority`) |
| Pushover 优先级 | `internal/integration/pushover/pushover.go` | 88 (`Priority: c.priority`) |
| 指标监控 | `internal/metric/metric.go` | 24 (`BackgroundFeedRefreshDuration`) |
| GUID 碰撞兜底 | `internal/reader/rss/adapter.go` | 116-139 |
| Session 轮换 | `internal/model/web_session.go` | 68 (`Rotate`) |
| i18n 资源 | `internal/locale/catalog.go` | 20 (`embed.FS` 翻译包) |
| 唯一索引 | `internal/database/migrations.go` | 88 (`unique (feed_id, hash)`) |
| Orphan 清理 | `internal/storage/icon.go` | 149 (`CleanupOrphanIcons`) |
| 定期清理任务 | `internal/cli/cleanup_tasks.go` | 16 (`runCleanupTasks`) |
| 配置废弃警告 | `internal/config/parser.go` | 166 (`slog.Warn deprecated`) |
| 时区废弃映射 | `internal/database/migrations.go` | 1184 (`deprecatedTimeZoneMap`) |
| 限流检测 | `internal/reader/fetcher/response_handler.go` | 98 (`IsRateLimited`) |
| Retry-After 解析 | `internal/reader/fetcher/response_handler.go` | 80 (`ParseRetryDelay`) |
| 前端滚动定位 | `internal/ui/static/js/app.js` | 55 (`scrollPageTo`) |
| 列表项跳转 | `internal/ui/static/js/app.js` | 389 (`goToListItem`) |
| Session DB 轮换 | `internal/storage/web_session.go` | 130 (`RotateWebSession`) |
| Staleness 配置 | `internal/config/options.go` | 125-155 (`CLEANUP_*` 定义) |
| Schema 单向迁移 | `internal/database/database.go` | 13 (`Migrate` 单向升级) |
| 版本完整性检查 | `internal/database/database.go` | 53 (`IsSchemaUpToDate`) |
| API Key 创建 | `internal/storage/api_key.go` | 73 (`CreateAPIKey`) |
| API Key 删除 | `internal/storage/api_key.go` | 104 (`DeleteAPIKey`) |
| 清理配置访问器 | `internal/config/options.go` | 652-660 |
| Toast 动画 | `internal/ui/static/js/app.js` | 269 (`animationend` 监听) |
| 触摸阻尼动画 | `internal/ui/static/js/touch_handler.js` | 37-88 |
| 页面过渡动画 | `internal/ui/static/css/common.css` | 85-92 (`@view-transition`) |
