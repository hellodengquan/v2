# Miniflux 通知触发流程分析

## 概述

Miniflux 的通知/集成系统涉及两类核心事件（手动保存、Feed 刷新推送），通过统一的 `integration` 包进行 provider 分流选择，采用 Fire-and-Forget 异步模式发送，失败仅记录日志，不具备持久化重试机制。

---

## 一、事件来源与触发条件

通知系统有两大类事件来源，触发分散在多个模块中，不易统一管理。

### 1.1 手动保存事件（SendEntry）

当用户主动"保存/收藏"某条文章时触发，调用 `integration.SendEntry(entry, userIntegrations)`。

| 触发入口 | 所在文件 | 触发条件 |
|---------|---------|---------|
| Web UI 保存按钮 | `internal/ui/entry_save.go:14` | 用户在 Web 界面点击保存 |
| REST API saveEntryHandler | `internal/api/entry_handlers.go:232` | 调用 API `/v1/entries/{entryID}/save`，且用户至少启用了一个集成 |
| Fever API handleWriteItems | `internal/fever/handler.go:401` | Fever 兼容 API 标记 `mark=item&as=saved` 时 |
| Google Reader API editTagHandler | `internal/googlereader/handler.go:187` | Google Reader 兼容 API 加星标（StarredStream）时 |

**调用模式**：所有入口均使用 `go integration.SendEntry(...)` 以 goroutine 异步执行，不阻塞 HTTP 请求响应。

### 1.2 Feed 刷新推送事件（PushEntries）

当后台定时刷新 Feed 发现新条目时自动触发，调用 `integration.PushEntries(feed, newEntries, userIntegrations)`。

| 触发入口 | 所在文件 | 触发条件 |
|---------|---------|---------|
| 后台调度器 feedScheduler | `internal/cli/scheduler.go:33` | 守护进程模式下，按 `PollingFrequency` 周期定时触发 |
| Worker 池 worker.Run | `internal/worker/worker.go:24` | 从 Job 队列取出任务执行 `RefreshFeed` |
| Feed 处理 RefreshFeed | `internal/reader/handler/handler.go:195` | 刷新后 `len(newEntries) > 0` 且 `userIntegrations != nil` 时（第 337-338 行） |
| CLI 手动刷新 refreshFeeds | `internal/cli/refresh_feeds.go:17` | 命令行执行 `miniflux -refresh-feeds` |

**调用模式**：`internal/reader/handler/handler.go:338` 使用 `go integration.PushEntries(...)` 异步执行。

---

## 二、Provider 分流与选择逻辑

Provider 分流由 `internal/integration/integration.go` 中两个核心函数完成，分别对应两类事件。

### 2.1 分流判断层级

Provider 是否被调用取决于三层判断：

```
用户级集成开关 (userIntegrations.XXXEnabled)
        ↓
Feed 级集成开关 (feed.XXXEnabled)  [部分 provider]
        ↓
Feed 级配置覆盖 (feed.WebhookURL 等)  [部分 provider]
```

#### 第一层：用户级集成开关

所有 provider 都必须先检查 `model.Integration` 中的启用标志，例如：
- `userIntegrations.WebhookEnabled`
- `userIntegrations.TelegramBotEnabled`
- `userIntegrations.DiscordEnabled`

数据来源：`internal/model/integration.go`，对应数据库表 `integrations`。

#### 第二层：Feed 级开关（仅部分 provider）

以下 provider 需要 Feed 级别也启用才会发送：

| Provider | Feed 级开关字段 | 所在文件 |
|---------|----------------|---------|
| Ntfy | `feed.NtfyEnabled` | `internal/model/feed.go:56` |
| Pushover | `feed.PushoverEnabled` | `internal/model/feed.go:55` |

对应判断：`internal/integration/integration.go:563` (Ntfy)、`:645` (Pushover)

#### 第三层：Feed 级配置覆盖（仅部分 provider）

以下配置可在 Feed 级别覆盖用户级全局配置：

| Provider | Feed 级覆盖字段 | 覆盖优先级 |
|---------|----------------|-----------|
| Webhook | `feed.WebhookURL` | feed 级 > user 级 (`integration.go:537-542`) |
| Ntfy | `feed.NtfyTopic` | feed 级 > user 级 (`integration.go:564-567`) |
| Ntfy | `feed.NtfyPriority` | 仅 feed 级 (`integration.go:583`) |
| Apprise | `feed.AppriseServiceURLs` | feed 级 > user 级 (`integration.go:598-601`) |
| Pushover | `feed.PushoverPriority` | 仅 feed 级 (`integration.go:655`) |

### 2.2 SendEntry Provider 列表（保存/稍后读类）

`internal/integration/integration.go:41-508` `SendEntry` 函数处理的 provider：

| Provider | 启用标志 | 说明 |
|---------|---------|------|
| Betula | `BetulaEnabled` | 书签服务 |
| Pinboard | `PinboardEnabled` | 书签服务 |
| Instapaper | `InstapaperEnabled` | 稍后读 |
| Wallabag | `WallabagEnabled` | 稍后读 |
| Notion | `NotionEnabled` | 知识库 |
| NunuxKeeper | `NunuxKeeperEnabled` | 文档管理 |
| Espial | `EspialEnabled` | 书签服务 |
| LinkAce | `LinkAceEnabled` | 书签服务 |
| Linkding | `LinkdingEnabled` | 书签服务 |
| LinkTaco | `LinktacoEnabled` | 书签服务 |
| Linkwarden | `LinkwardenEnabled` | 书签服务 |
| Readeck | `ReadeckEnabled` | 稍后读 |
| Readwise | `ReadwiseEnabled` | 阅读高亮 |
| Cubox | `CuboxEnabled` | 稍后读 |
| Shiori | `ShioriEnabled` | 书签服务 |
| Shaarli | `ShaarliEnabled` | 书签服务 |
| Archive.org | `ArchiveorgEnabled` | 网页存档 |
| Webhook | `WebhookEnabled` | 发送 `save_entry` 事件 |
| Omnivore | `OmnivoreEnabled` | 稍后读 |
| Karakeep | `KarakeepEnabled` | 书签服务 |
| Raindrop | `RaindropEnabled` | 书签服务 |

### 2.3 PushEntries Provider 列表（通知/推送类）

`internal/integration/integration.go:511-719` `PushEntries` 函数处理的 provider：

| Provider | 启用标志 | 需 Feed 级开关 | 发送粒度 |
|---------|---------|-------------|---------|
| Matrix Bot | `MatrixBotEnabled` | 否 | 批量发送 |
| Webhook | `WebhookEnabled` | 否 | 批量发送 `new_entries` 事件 |
| Ntfy | `NtfyEnabled` | 是 (`feed.NtfyEnabled`) | 批量发送 |
| Apprise | `AppriseEnabled` | 否 | 批量发送 |
| Discord | `DiscordEnabled` | 否 | 批量发送 |
| Slack | `SlackEnabled` | 否 | 批量发送 |
| Pushover | `PushoverEnabled` | 是 (`feed.PushoverEnabled`) | 批量发送 |
| Telegram Bot | `TelegramBotEnabled` | 否 | 逐条发送 |
| Readeck Push | `ReadeckPushEnabled` | 否 | 逐条发送 |

注意：Readeck 同时出现在两类事件中，`ReadeckEnabled` 对应手动保存，`ReadeckPushEnabled` 对应自动推送。

---

## 三、失败处理与重试机制

### 3.1 当前实现：无持久化重试（仅日志记录）

经过对代码和数据库 schema 的全面分析，**当前 Miniflux 通知系统不具备持久化的失败重试机制**。

#### 证据：

1. **无通知队列表**：数据库迁移 `internal/database/migrations.go` 中不存在 `notification_queue`、`retry_queue`、`failed_notifications` 等表。

2. **仅日志记录**：所有 provider 调用失败后，仅通过结构化日志记录：
   - `SendEntry` 中使用 `slog.Error` (例如 `integration.go:57-63`)
   - `PushEntries` 中混合使用 `slog.Error` 和 `slog.Warn` (例如 `integration.go:528-534` 使用 Error，`integration.go:553-559` 使用 Warn)

3. **Fire-and-Forget 模式**：所有通知均通过 goroutine 异步触发，调用方不等待返回结果，无法感知失败。

4. **Provider 内部无重试**：以 Webhook 为例（`internal/integration/webhook/webhook.go:117-149`），`makeRequest` 仅发起一次 HTTP 请求，无任何重试逻辑。

5. **Feed 错误计数器仅针对 Feed 抓取**：`model.Feed.ParsingErrorCount`（`internal/model/feed.go:36`）用于追踪 Feed 解析/抓取失败，与通知无关。

### 3.2 执行模式对比

| 方面 | SendEntry | PushEntries |
|-----|-----------|-------------|
| 触发方式 | 同步 HTTP 请求中启动 goroutine | 后台 worker 中启动 goroutine |
| 失败日志级别 | `slog.Error` | 部分 Error，部分 Warn |
| 调用方感知 | 完全不感知 | 完全不感知 |
| 上下文丢失 | goroutine 中无请求 context | goroutine 中无任务 context |
| 重试可能 | 无 | 无 |

### 3.3 日志字段约定

失败日志包含以下标准字段（以 `integration.go:57-62` 为例）：

```go
slog.Error("Unable to send entry to Betula",
    slog.Int64("user_id", userIntegrations.UserID),
    slog.Int64("entry_id", entry.ID),
    slog.String("entry_url", entry.URL),
    slog.Any("error", err),
)
```

### 3.4 现有可观测性

- **指标**：`metric.BackgroundFeedRefreshDuration` 仅追踪 Feed 刷新耗时，不包含通知发送指标
- **健康检查**：`internal/cli/health_check.go` 不检查通知系统状态
- **告警**：无内置告警，需依赖外部日志系统监控 `slog.Error/Warn`

---

## 四、完整流程时序图

### 4.1 手动保存流程 (SendEntry)

```
用户操作 (Web UI / API / Fever / GoogleReader)
        │
        ▼
  HTTP Handler 获取 entry + userIntegrations
        │
        ├─ 校验 entry 是否存在
        ├─ 校验集成是否启用 (API: HasSaveEntry)
        │
        ▼
  go integration.SendEntry(entry, userIntegrations)  ──► 新 goroutine
        │                                                    │
        ▼                                                    ▼
  HTTP 立即响应                                    遍历检查各 provider Enabled
  (201/202)                                                │
                                                           ▼
                                                 provider 逐一发送
                                                 (失败仅 slog.Error)
```

### 4.2 Feed 刷新推送流程 (PushEntries)

```
feedScheduler (time.Tick)
        │
        ▼
  BatchBuilder.FetchJobs() ──► JobList
        │
        ▼
  pool.Push(jobs) ──► worker queue channel
                            │
                            ▼
                    worker.Run 消费 Job
                            │
                            ▼
                    handler.RefreshFeed(store, userID, feedID, false)
                            │
                            ├─ 抓取并解析 Feed
                            ├─ processor.ProcessFeedEntries
                            ├─ store.RefreshFeedEntries → newEntries
                            │
                            ▼ (len(newEntries) > 0)
                    go integration.PushEntries(feed, newEntries, userIntegrations)
                            │                                    │
                            ▼                                    ▼
                    继续执行 Feed 更新                       遍历检查各 provider Enabled
                    ResetErrorCounter / UpdateFeed                 │
                                                                   ▼
                                                         provider 逐一发送
                                                         (失败仅 slog.Error/Warn)
```

---

## 五、关键文件索引

| 功能 | 文件路径 |
|-----|---------|
| 集成入口（SendEntry/PushEntries） | `internal/integration/integration.go` |
| 集成配置模型 | `internal/model/integration.go` |
| Feed 模型（含 Feed 级集成开关） | `internal/model/feed.go` |
| Feed 刷新核心处理 | `internal/reader/handler/handler.go` |
| 后台调度器 | `internal/cli/scheduler.go` |
| Worker 池 | `internal/worker/pool.go` |
| Worker 执行逻辑 | `internal/worker/worker.go` |
| Web UI 保存入口 | `internal/ui/entry_save.go` |
| REST API 保存入口 | `internal/api/entry_handlers.go:232` |
| Fever API 保存入口 | `internal/fever/handler.go:401` |
| Google Reader API 保存入口 | `internal/googlereader/handler.go:187` |
| Webhook Provider 实现 | `internal/integration/webhook/webhook.go` |
| 数据库迁移 | `internal/database/migrations.go` |

---

## 六、总结

1. **事件触发分散**：两类事件（保存/推送）分别由 4 个和 4 个入口触发，共计 8 处触发点，统一管理困难。

2. **Provider 分流逻辑内聚**：所有 30+ provider 的选择判断均集中在 `internal/integration/integration.go` 一个文件中，通过用户级 + Feed 级的二级开关 + 配置覆盖实现。

3. **无持久化重试**：所有通知采用 Fire-and-Forget 异步模式，失败仅记录日志，无队列、无重试、无死信处理。任何通知发送失败（网络错误、第三方服务不可用等）都会导致该条通知永久丢失。

4. **可观测性薄弱**：缺乏通知发送成功率、延迟等指标，也无告警机制。
