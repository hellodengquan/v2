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
