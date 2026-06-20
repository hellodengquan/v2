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

Provider 是否被调用取决于三层判断，但**并非每个 provider 都完整经过三层**——大部分 provider 只经第一层（用户级开关）就直接决定是否发送。

```
第一层：用户级集成开关 (userIntegrations.XXXEnabled)     ← 全部 30+ provider 必过
        ↓
第二层：Feed 级集成开关 (feed.XXXEnabled)               ← 仅 Ntfy、Pushover 需要
        ↓
第三层：Feed 级配置覆盖 (feed.WebhookURL 等)             ← 仅 Webhook、Ntfy、Apprise、Pushover 有
```

#### 第一层：用户级集成开关（全部 provider 必经）

所有 provider 都必须先检查 `model.Integration` 中的启用标志，例如：
- `userIntegrations.WebhookEnabled`
- `userIntegrations.TelegramBotEnabled`
- `userIntegrations.DiscordEnabled`

数据来源：`internal/model/integration.go`，对应数据库表 `integrations`。

这一层的判断模式完全统一——每一个 provider 都以 `if userIntegrations.XXXEnabled { ... }` 包裹，没有任何 provider 在用户级开关关闭的情况下仍有条件地发送。

#### 第二层：Feed 级开关（仅 Ntfy 和 Pushover）

以下两个 provider 在用户级开关通过后，还必须满足 Feed 级开关才能发送。这是一个 **AND 语义**的双层门控，而非 OR 语义：

| Provider | 用户级开关 | Feed 级开关 | 判断代码 | 语义 |
|---------|-----------|-----------|---------|------|
| Ntfy | `userIntegrations.NtfyEnabled` | `feed.NtfyEnabled` | `integration.go:563` `if userIntegrations.NtfyEnabled && feed.NtfyEnabled` | 两者必须同时为 true |
| Pushover | `userIntegrations.PushoverEnabled` | `feed.PushoverEnabled` | `integration.go:645` `if userIntegrations.PushoverEnabled && feed.PushoverEnabled` | 两者必须同时为 true |

**关键细节**：这两个 Feed 级开关仅在 `PushEntries`（Feed 刷新推送）中生效。在 `SendEntry`（手动保存）中，**不存在任何 Feed 级开关判断**——所有 21 个保存类 provider 仅凭用户级开关即决定发送。这意味着即使某个 Feed 在 Feed 设置中关闭了 Ntfy/Pushover 推送，用户手动保存该 Feed 下的文章时，也不会触发 Ntfy/Pushover 的 Feed 级开关过滤（因为 `SendEntry` 不涉及 Ntfy/Pushover）。

其余所有 provider（Matrix Bot、Webhook、Discord、Slack、Apprise、Telegram Bot、Readeck Push）在 `PushEntries` 中也**无 Feed 级开关**，仅凭用户级开关即决定。

#### 第三层：Feed 级配置覆盖（4 个 provider，5 个覆盖点）

当第一层（和第二层，如有）通过后，部分 provider 会进一步检查 Feed 上是否设置了覆盖配置。覆盖逻辑的核心模式是：**Feed 级字段非空时优先使用 Feed 级值，否则回退到用户级值**。

##### Webhook URL 覆盖

覆盖出现在 `SendEntry` 和 `PushEntries` 两个函数中，代码路径略有不同：

**SendEntry 路径** (`integration.go:422-428`)：
```go
var webhookURL string
if entry.Feed != nil && entry.Feed.WebhookURL != "" {
    webhookURL = entry.Feed.WebhookURL    // Feed 级覆盖
} else {
    webhookURL = userIntegrations.WebhookURL  // 用户级回退
}
```
注意 `entry.Feed != nil` 的空指针保护——手动保存场景中 entry 的 Feed 字段可能未加载，此时直接回退用户级。

**PushEntries 路径** (`integration.go:537-542`)：
```go
var webhookURL string
if feed.WebhookURL != "" {
    webhookURL = feed.WebhookURL          // Feed 级覆盖
} else {
    webhookURL = userIntegrations.WebhookURL  // 用户级回退
}
```
这里不需要空指针保护，因为 `PushEntries` 直接接收 `*model.Feed` 参数。

**WebhookSecret 不可覆盖**：无论 URL 取哪一级，`webhookSecret` 始终使用 `userIntegrations.WebhookSecret`（`integration.go:437` 和 `:551`），Feed 级没有 secret 字段。

##### Ntfy Topic 覆盖

`integration.go:564-567`：
```go
ntfyTopic := feed.NtfyTopic
if ntfyTopic == "" {
    ntfyTopic = userIntegrations.NtfyTopic   // Feed 级为空时回退用户级
}
```
优先级：`feed.NtfyTopic` > `userIntegrations.NtfyTopic`。注意空字符串 `""` 是回退条件，而非零值判断。

##### Ntfy Priority 覆盖

`integration.go:583`：
```go
client := ntfy.NewClient(
    ...
    feed.NtfyPriority,   // 直接使用 Feed 级值，无回退逻辑
    ...
)
```
`feed.NtfyPriority` 是 `int` 类型（`model/feed.go:63`），零值 `0` 表示使用 Ntfy 服务端默认优先级。此处**没有回退到用户级**——用户级没有 NtfyPriority 字段。

##### Apprise Service URLs 覆盖

`integration.go:598-601`：
```go
appriseServiceURLs := userIntegrations.AppriseServicesURL   // 先取用户级
if feed.AppriseServiceURLs != "" {
    appriseServiceURLs = feed.AppriseServiceURLs              // Feed 级非空则覆盖
}
```
优先级：`feed.AppriseServiceURLs` > `userIntegrations.AppriseServicesURL`。

##### Pushover Priority 覆盖

`integration.go:655`：
```go
client := pushover.NewClient(
    userIntegrations.PushoverUser,
    userIntegrations.PushoverToken,
    feed.PushoverPriority,     // 直接使用 Feed 级值，无回退逻辑
    ...
)
```
`feed.PushoverPriority` 是 `int` 类型（`model/feed.go:63`），与 Ntfy Priority 类似，**没有回退到用户级**——用户级没有 PushoverPriority 字段。

#### 三层判断的完整路径汇总

下表列出所有 30+ provider 在两个函数中经过的判断层级，`-` 表示该层不存在：

| Provider | 所在函数 | 第一层：用户级开关 | 第二层：Feed 级开关 | 第三层：Feed 级配置覆盖 |
|---------|---------|-----------------|-----------------|---------------------|
| Betula | SendEntry | `BetulaEnabled` | - | - |
| Pinboard | SendEntry | `PinboardEnabled` | - | - |
| Instapaper | SendEntry | `InstapaperEnabled` | - | - |
| Wallabag | SendEntry | `WallabagEnabled` | - | - |
| Notion | SendEntry | `NotionEnabled` | - | - |
| NunuxKeeper | SendEntry | `NunuxKeeperEnabled` | - | - |
| Espial | SendEntry | `EspialEnabled` | - | - |
| LinkAce | SendEntry | `LinkAceEnabled` | - | - |
| Linkding | SendEntry | `LinkdingEnabled` | - | - |
| LinkTaco | SendEntry | `LinktacoEnabled` | - | - |
| Linkwarden | SendEntry | `LinkwardenEnabled` | - | - |
| Readeck (保存) | SendEntry | `ReadeckEnabled` | - | - |
| Readwise | SendEntry | `ReadwiseEnabled` | - | - |
| Cubox | SendEntry | `CuboxEnabled` | - | - |
| Shiori | SendEntry | `ShioriEnabled` | - | - |
| Shaarli | SendEntry | `ShaarliEnabled` | - | - |
| Archive.org | SendEntry | `ArchiveorgEnabled` | - | - |
| **Webhook (保存)** | SendEntry | `WebhookEnabled` | - | `feed.WebhookURL` > `user.WebhookURL` |
| Omnivore | SendEntry | `OmnivoreEnabled` | - | - |
| Karakeep | SendEntry | `KarakeepEnabled` | - | - |
| Raindrop | SendEntry | `RaindropEnabled` | - | - |
| Matrix Bot | PushEntries | `MatrixBotEnabled` | - | - |
| **Webhook (推送)** | PushEntries | `WebhookEnabled` | - | `feed.WebhookURL` > `user.WebhookURL` |
| **Ntfy** | PushEntries | `NtfyEnabled` | `feed.NtfyEnabled` (AND) | `feed.NtfyTopic` > `user.NtfyTopic`; `feed.NtfyPriority` (无回退) |
| **Apprise** | PushEntries | `AppriseEnabled` | - | `feed.AppriseServiceURLs` > `user.AppriseServicesURL` |
| Discord | PushEntries | `DiscordEnabled` | - | - |
| Slack | PushEntries | `SlackEnabled` | - | - |
| **Pushover** | PushEntries | `PushoverEnabled` | `feed.PushoverEnabled` (AND) | `feed.PushoverPriority` (无回退) |
| Telegram Bot | PushEntries | `TelegramBotEnabled` | - | - |
| Readeck Push | PushEntries | `ReadeckPushEnabled` | - | - |

**总结**：在 30+ 个 provider 中，仅有 **4 个 provider** 存在 Feed 级配置覆盖（Webhook、Ntfy、Apprise、Pushover），仅有 **2 个 provider** 存在 Feed 级开关（Ntfy、Pushover），其余 25+ 个 provider 仅凭用户级开关一锤定音。

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

### 3.2 goroutine 生命周期与进程关闭时的泄漏风险

通知发送 goroutine 的生命周期管理存在一个关键盲区：**`go integration.PushEntries(...)` 启动的 goroutine 不被 worker 池的 `WaitGroup` 追踪**。

#### 代码追踪：worker 池的关闭流程

1. **信号捕获** (`daemon.go:77`)：`<-stop` 阻塞等待 SIGINT/SIGTERM
2. **HTTP 服务器关闭** (`daemon.go:83-95`)：5 秒超时 context
3. **Worker 池关闭** (`daemon.go:98`)：`pool.Shutdown()`

#### `pool.Shutdown()` 的实现 (`worker/pool.go:27-30`)：
```go
func (p *Pool) Shutdown() {
    close(p.queue)     // 关闭 Job 队列通道
    p.wg.Wait()        // 等待所有 worker goroutine 退出
}
```

#### Worker 池对外暴露的接口完整清单（以及为什么无法等待 in-flight 通知）

`Pool` 结构体定义 (`worker/pool.go:14-17`)：
```go
type Pool struct {
    queue chan model.Job   // 私有，小写开头
    wg    sync.WaitGroup   // 私有，小写开头
}
```

`Pool` 对外暴露的方法只有以下 3 个：

| 方法 | 签名 | 用途 | 能等待通知 goroutine? |
|-----|------|-----|---------------------|
| `Push` | `(p *Pool) Push(jobs model.JobList)` | 将 Feed 刷新 job 推入队列 | 否 |
| `Shutdown` | `(p *Pool) Shutdown()` | close(queue) + wg.Wait() | 否（只等 worker，不等通知） |
| `NewPool`（构造函数） | `NewPool(store, nbWorkers) *Pool` | 创建 pool 并启动 N 个 worker | 否 |

**关键发现**：

1. **没有 `Drain()` 方法**：与某些 Worker 池实现不同，miniflux 的 Pool 不提供 `Drain()` 方法来等待所有已入队 job 完成后再关闭。`Shutdown()` 直接 `close(p.queue)`，这意味着**关闭瞬间 queue 中尚未被 worker 取走的 job 会被丢弃**——但实际上 queue 是无缓冲 channel（`make(chan model.Job)`），`Push()` 是阻塞发送，所以 queue 中不会有积压 job，每一个 job 在 Push 时就已经被某个 worker 取走了。

2. **`wg` WaitGroup 私有不可导出**：`Pool.wg` 是私有字段（`sync.WaitGroup` 本身也是结构体，不是接口），外部调用方无法获取这个 WaitGroup，也无法通过类型断言访问它。

3. **没有第二个 WaitGroup 追踪通知 goroutine**：Pool 中只维护了一个追踪 worker goroutine 的 `wg`，没有第二个 `notifyWg sync.WaitGroup`。理论上要等待通知完成，需要：
   - `PushEntries`/`SendEntry` 启动 goroutine 前调用 `notifyWg.Add(1)`
   - goroutine 结束时 `defer notifyWg.Done()`
   - `Shutdown()` 中 `wg.Wait()` 之后再 `notifyWg.Wait()`
   - 但实际代码中完全没有这套机制。

4. **没有通过 context 传递取消信号的路径**：`RefreshFeed` 函数签名不接受 `context.Context`，`PushEntries` 函数签名也不接受 context。即使外部创建了通知专用 WaitGroup，也没有办法通过 context 告诉通知 goroutine"进程正在关闭，超时后请放弃"。

5. **调用方（`daemon.go`）没有额外的等待机制**：`daemon.go:98` 直接调用 `pool.Shutdown()`，之后：
   ```go
   slog.Info("All background jobs completed, shutting down...")
   pool.Shutdown()
   slog.Info("Process gracefully stopped")
   ```
   调用方认为 `pool.Shutdown()` 返回即代表"所有后台工作已完成"，但实际上通知 goroutine 仍在运行。daemon 没有 `time.Sleep(10s)`、`time.AfterFunc`、额外的 WaitGroup 或任何其他机制来给通知 goroutine 缓冲时间。

**结论**：从 API 设计层面，worker pool 对外**完全没有暴露**任何 Drain、WaitGroup、Context、回调钩子或其他机制供调用方等待 in-flight 通知发送完成。调用方（`startDaemon`）也没有自行实现额外的等待策略。in-flight 通知 goroutine 在进程关闭时完全处于"无人看管"状态。

#### `worker.Run` 的退出条件 (`worker/worker.go:24-49`)：
```go
func (w *worker) Run(c <-chan model.Job, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range c {   // 当 close(p.queue) 后，range 结束，goroutine 退出
        ...
        localizedError := feedHandler.RefreshFeed(...)  // 同步调用，阻塞直到返回
        ...
    }
}
```

当 `close(p.queue)` 后，所有 worker 的 `for job := range c` 循环结束，`defer wg.Done()` 执行，`p.wg.Wait()` 返回。

#### 关键问题：PushEntries goroutine 不在 WaitGroup 中

在 `handler/handler.go:337-338`：
```go
} else if userIntegrations != nil && len(newEntries) > 0 {
    go integration.PushEntries(originalFeed, newEntries, userIntegrations)
}
```

这个 `go` 启动的 goroutine 是在 `feedHandler.RefreshFeed` 内部启动的，属于 worker goroutine 的子 goroutine，但**它没有被任何 `sync.WaitGroup` 追踪**。

#### Graceful Shutdown 时的实际行为：既非显式 kill，也非等跑完

这是一个容易误解的点。进程 graceful shutdown 阶段对通知 goroutine 的实际处理**既不是主动 kill，也不是等它跑完**，而是以下时序：

**阶段 1 —— Worker 自身被等待**
- `close(p.queue)` 发出信号，worker goroutine 不再接收新 job
- 正在执行的 `feedHandler.RefreshFeed(...)` 是**同步调用**，worker 会阻塞等待它返回
- 这意味着：RefreshFeed 内部所有逻辑（抓取、解析、写数据库、启动 PushEntries goroutine）全部执行完毕后，worker 才会退出

**阶段 2 —— 通知 goroutine 被"遗弃"**
- 当 `RefreshFeed` 返回后，其中 `go integration.PushEntries(...)` 启动的子 goroutine **仍在运行**（它正在遍历 provider 并发起 HTTP 请求）
- worker goroutine 此时已经继续 for-range 或直接退出，不会等待子 goroutine
- 子 goroutine 的父 goroutine（worker）退出后，子 goroutine **不会被 Go runtime 自动 kill**——Go 中没有父子 goroutine 的层级关系，一旦启动就完全独立

**阶段 3 —— 进程退出，所有 goroutine 被强制终止**
- 所有 worker 退出后，`p.wg.Wait()` 返回
- `daemon.go:101` 输出 `"Process gracefully stopped"`，`startDaemon` 函数返回
- 当 `main` 函数返回时，Go runtime 会终止整个进程，**所有仍在运行的 goroutine（包括通知 goroutine）都会被操作系统强制终止**
- 注意：这不是 Go runtime 的优雅 goroutine 取消，而是**整个进程被终止**——正在执行的 `httpClient.Do(...)` 会因为底层 socket 被关闭而失败，任何已发送但未收到响应的请求不会被补发

**关闭时序精确分析**（单 worker 场景）：

```
时刻 T0: worker goroutine W 正在执行 RefreshFeed
时刻 T1: RefreshFeed 内部启动 go PushEntries(...)  ← 子 goroutine G 开始执行
时刻 T2: G 正在遍历 provider，已向 Discord 发起 POST，等待响应
时刻 T3: RefreshFeed 完成剩余逻辑（ResetErrorCounter、UpdateFeed）并返回
时刻 T4: W 检测到 p.queue 已关闭，执行 defer wg.Done()，W 退出
时刻 T5: G 仍在执行中（例如正在等待 Pushover API 返回）
时刻 T6: p.wg.Wait() 返回（最后一个 W 已退出）
时刻 T7: daemon 输出 "Process gracefully stopped"，startDaemon 返回
时刻 T8: main 返回，操作系统终止进程
时刻 T9: G 在执行 httpClient.Do 时因 socket 关闭返回 error，或在任意代码处被直接终止
        （最终行为取决于操作系统调度时机）
```

#### 关键代码证据：无取消机制

1. **PushEntries 函数签名不接受 context**：`func PushEntries(feed *model.Feed, entries model.Entries, userIntegrations *model.Integration)` (`integration.go:511`)，没有 `context.Context` 参数
2. **RefreshFeed 函数签名不接受 context**：`func RefreshFeed(store *storage.Storage, userID, feedID int64, forceRefresh bool) *locale.LocalizedError` (`handler.go:195`)，没有 context
3. **HTTP 客户端无 context 绑定**：`webhook/makeRequest` 使用 `http.NewRequest(http.MethodPost, ...)` 而非 `http.NewRequestWithContext(ctx, ...)`，无法通过 context 取消进行中的 HTTP 请求
4. **通知 goroutine 无 WaitGroup 追踪**：`go integration.PushEntries(...)` 没有对应的 `wg.Add(1)` / `wg.Done()`

#### SendEntry goroutine 的生命周期

手动保存场景中的 `go integration.SendEntry(entry, userIntegrations)` 同样没有 WaitGroup 追踪。但 SendEntry 通常由 HTTP handler 启动，进程关闭时 HTTP server 有 5 秒优雅关闭窗口 (`daemon.go:80`)。然而：

1. `server.Shutdown(ctx)` 只等待**正在处理的 HTTP 请求**完成，不等待 `go` 启动的子 goroutine
2. SendEntry goroutine 内的 HTTP 请求同样没有 context 绑定，无法被取消
3. 如果 SendEntry goroutine 在 5 秒内未完成，进程退出时同样会被强制中断

#### 对比：有无 context 传递的差异

| 场景 | 有 context? | 可被取消? | 被等待? |
|-----|-----------|---------|--------|
| Worker 池中 Feed 刷新 | 无 | 否 | worker 本身被等待，但子 goroutine 不被等待 |
| Feed 刷新内的 PushEntries | 无 | 否 | 否 |
| HTTP handler 内的 SendEntry | 无 | 否 | 否 |
| Metrics 收集 (`metric.Collector`) | 有 (`metricsCtx`) | 是（`cancelMetrics()` 可取消） | 否，但通过 context 优雅退出 |
| HTTP Server 请求处理 | 有 (`ctx`) | 是（`server.Shutdown` 触发取消） | 是（5 秒窗口内等待） |

**结论**：所有通知发送 goroutine 在进程关闭时都是**孤儿状态**——它们既没有 context 可以取消，也没有 WaitGroup 可以等待。RefreshFeed 本身作为同步调用会被 worker pool 等待完成，但它启动的通知子 goroutine 不会被等待。进程的"优雅停止"仅覆盖了 HTTP server 和 worker goroutine，不覆盖通知 goroutine。通知 goroutine 会在 main 返回后、进程被操作系统终止时随进程一起被强制终止。

#### 进程终止对正在发送中 HTTP 请求的精确副作用

当进程被终止时，通知 goroutine 中正在执行的 `httpClient.Do(request)` 会经历什么？这取决于 HTTP 请求的进度阶段：

##### 情况 1：请求尚未发出（DNS 解析 / TCP 连接阶段）

如果 goroutine 正在 `Transport.DialContext` 中执行 DNS 解析或 TCP 三次握手：
- 操作系统终止进程时，所有打开的文件描述符（包括未完成的 socket）被自动关闭
- DNS 解析中断：`net.LookupIP` 返回错误（context canceled 或文件描述符无效）
- TCP 连接中断：`dialer.DialContext` 返回 `connect: can't assign requested address` 或 `use of closed network connection`
- **后果**：请求从未到达服务端，目标 provider 完全不知道有此请求。下次 Feed 刷新时不会重试此通知。

##### 情况 2：TLS 握手阶段

如果 TCP 连接已建立，正在进行 TLS 握手：
- 进程终止关闭底层 TCP socket
- TLS 握手中断，对端收到 TCP RST 或 FIN
- **后果**：服务端可能看到连接被重置，但 HTTP 请求尚未发出，服务端无法识别这是通知请求

##### 情况 3：请求体正在发送（最危险的情况）

如果 HTTP 请求行和 header 已发送，请求体（JSON payload）正在传输中：
- 进程终止导致 TCP socket 关闭，发送缓冲区中未发送完的数据丢失
- 服务端可能收到**不完整的 JSON 请求体**
- **后果**：
  - 如果服务端在收到完整请求前检测到连接关闭，会返回 4xx/5xx 错误——但 miniflux 进程已死，无法接收此响应
  - 如果服务端在请求体不完整的情况下仍然处理（取决于服务端实现），可能产生**部分写入**问题——例如 Pushover API 收到不完整的 JSON 后可能返回错误，也可能按空字段处理
  - **最关键的不可逆后果**：Webhook 请求体已经到达服务端但 miniflux 已无法接收响应——服务端可能已成功处理，但 miniflux 视为失败；若 miniflux 有重试机制，会导致**重复通知**。当前没有重试机制，所以不存在此问题

##### 情况 4：请求已发完，等待响应

如果完整请求体已发送，goroutine 正在等待服务端响应：
- 进程终止关闭 socket，服务端发送的响应数据无法被接收
- 服务端角度：请求已完整到达，服务端正常处理并返回响应，但客户端在响应到达前断开
- **后果**：这是**唯一可能导致通知实际已成功但 miniflux 认为失败**的场景。由于没有重试机制，不会产生重复通知；但 miniflux 的日志中不会记录此次成功，造成通知状态的**静默不一致**——服务端认为已处理，miniflux 无记录

##### 情况 5：响应已到达，正在读取响应体

如果 `httpClient.Do` 已收到响应头，正在 `io.CopyN` 或 `json.NewDecoder` 读取响应体：
- 进程终止导致响应体读取中断
- **后果**：与情况 4 类似——通知实际已成功，但 goroutine 来不及记录日志

##### 情况 6：`httpClient.Do` 已返回，正在执行后续逻辑

如果 HTTP 请求已完成（`httpClient.Do` 返回），goroutine 正在执行后续代码（如 `response.Body.Close()`、错误判断、日志记录）：
- 进程终止导致后续逻辑被中断
- **后果**：如果 `defer response.Body.Close()` 未执行，响应体未关闭，但进程终止会回收所有 fd，所以不会泄漏

##### 副作用总结

| 请求阶段 | 服务端是否收到请求 | 通知是否生效 | 是否可能重复 | 日志中是否有记录 |
|---------|----------------|-----------|-----------|-------------|
| DNS/TCP 连接 | 否 | 否 | 否 | 否 |
| TLS 握手 | 否 | 否 | 否 | 否 |
| 请求体发送中 | 可能部分 | 取决于服务端 | 否 | 否 |
| 等待响应 | 是 | 是（静默成功） | 否 | 否（无法记录） |
| 读取响应体 | 是 | 是（静默成功） | 否 | 否（无法记录） |
| 后续逻辑执行 | 是 | 是 | 否 | 可能部分记录 |

**核心结论**：进程终止对正在发送中通知的最严重副作用不是"通知丢失"（这在当前 Fire-and-Forget 设计下本身就是预期行为），而是**静默成功**——通知实际上已被服务端处理，但 miniflux 侧无任何记录。对于有状态通知（如稍后读服务创建书签），这可能导致用户重复操作；对于纯通知推送（如 Telegram/Discord），后果较轻（用户可能重复收到通知，但当前无重试机制所以不会发生）。

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

## 四、Webhook Provider 的请求签名与 HTTP 行为

Webhook 是所有 provider 中安全机制最完善的一个，也是唯一具备请求完整性验证能力的 provider。以下深入分析其签名生成和 HTTP 客户端行为。

### 4.1 HMAC-SHA256 签名头与 JSON payload 拼装顺序

签名在 `webhook/webhook.go:134` 中生成并设置到请求头。本节从代码层面精确说明 header 键名、payload 结构与序列化顺序，因为这三点共同决定了签名的可验证性。

#### 签名头键名：`X-Miniflux-Signature`

代码中硬编码的 header 键名：
```go
request.Header.Set("X-Miniflux-Signature", crypto.GenerateSHA256Hmac(c.webhookSecret, requestBody))
```

**键名字面量**：`X-Miniflux-Signature`
- 使用 `request.Header.Set()` 而非 `Add()`，即如果已有同名 header 会被覆盖，不会出现重复头
- Go 的 `net/http` 在发送请求时会自动对 header 键名执行 canonicalization（首字母大写、连字符后首字母大写），因此实际发出的 header 为 `X-Miniflux-Signature`
- HTTP/2 场景下，Go 标准库会自动转为全小写：`x-miniflux-signature`
- 配套的事件类型头：`X-Miniflux-Event-Type`，值为常量 `NewEntriesEventType = "new_entries"` 或 `SaveEntryEventType = "save_entry"`

#### 签名计算函数 `crypto.GenerateSHA256Hmac`

实现 (`crypto/crypto.go:48-52`)：
```go
func GenerateSHA256Hmac(secret string, data []byte) string {
    h := hmac.New(sha256.New, []byte(secret))  // 以 secret 为密钥创建 HMAC-SHA256
    h.Write(data)                                // 写入完整请求体
    return hex.EncodeToString(h.Sum(nil))        // 输出小写十六进制编码
}
```

**签名流程精确说明**：
1. `hmac.New(sha256.New, []byte(secret))`：将 `webhookSecret` **字符串直接转为 `[]byte`**（UTF-8 编码，无额外处理），即使 secret 为空也不报错
2. `h.Write(data)`：写入 `json.Marshal(payload)` 的**完整原始字节**，包含 JSON 本身的空格、转义、字段顺序
3. `h.Sum(nil)`：输出 32 字节摘要
4. `hex.EncodeToString(...)`：使用 Go 标准库小写十六进制编码，输出恰好 64 个字符（a-f, 0-9）

接收方验证时必须使用完全一致的编码（小写 hex），否则会因大小写不一致导致验证失败。

#### JSON payload 结构与字段序列化顺序

签名依赖 `json.Marshal(payload)` 的**精确字节输出**。Go 的 `encoding/json` 对于结构体按**字段声明顺序**序列化，因此结构体定义中的字段顺序直接决定签名计算结果。

##### new_entries 事件 payload

对应结构体 `WebhookNewEntriesEvent` (`webhook/webhook.go:189-193`)：
```go
type WebhookNewEntriesEvent struct {
    EventType string          `json:"event_type"`   // 第 1 字段
    Feed      *WebhookFeed    `json:"feed"`         // 第 2 字段
    Entries   []*WebhookEntry `json:"entries"`      // 第 3 字段
}
```
JSON 输出顺序：`{"event_type":..., "feed":{...}, "entries":[...]}`

`WebhookFeed` 字段顺序 (`webhook/webhook.go:151-160`)：
```
id → user_id → category_id → category → feed_url → site_url → title → checked_at
```

`WebhookEntry` 字段顺序 (`webhook/webhook.go:167-187`)：
```
id → user_id → feed_id → status → hash → title → url → comments_url
→ published_at → created_at → changed_at → content → author → share_code
→ starred → reading_time → enclosures → tags → feed
```

注意 `Date` 字段在 JSON 中序列化为 `published_at`（tag 重命名），`Feed` 字段带 `omitempty`。

##### save_entry 事件 payload

对应结构体 `WebhookSaveEntryEvent` (`webhook/webhook.go:195-198`)：
```go
type WebhookSaveEntryEvent struct {
    EventType string        `json:"event_type"`  // 第 1 字段
    Entry     *WebhookEntry `json:"entry"`       // 第 2 字段
}
```
JSON 输出顺序：`{"event_type":..., "entry":{...}}`

**payload 组装代码顺序** (`webhook/webhook.go:38-70` SendSaveEntryWebhookEvent)：
1. 构建 `WebhookEntry`（先 entry 自身字段，再 Feed）
2. 构建 `WebhookSaveEntryEvent` 包装 event_type 和 entry
3. `c.makeRequest(SaveEntryEventType, payload)` → `json.Marshal(payload)` → HMAC 签名

**payload 组装代码顺序** (`webhook/webhook.go:73-114` SendNewEntriesWebhookEvent)：
1. 遍历 entries，逐个构建 `WebhookEntry`（无 Feed 字段，因为 Feed 在顶层）
2. 构建 `WebhookFeed`
3. 构建 `WebhookNewEntriesEvent` 包装 event_type、feed、entries
4. `c.makeRequest(NewEntriesEventType, payload)` → `json.Marshal(payload)` → HMAC 签名

#### 签名稳定性保证

| 因素 | 是否影响签名 | 说明 |
|-----|------------|------|
| Go 版本升级 | 否 | `encoding/json` 对结构体字段顺序的序列化是稳定规范 |
| JSON key 名称 | 是 | struct tag 修改会导致 JSON key 变化，签名失效 |
| 字段声明顺序 | 是 | 交换结构体中两个字段的顺序会改变 JSON 字节流 |
| entry.Starred 布尔值 | 是 | true/false 直接写入 JSON，影响签名 |
| entry.ReadingTime 零值 | 是 | int 零值 0 仍然序列化为 `"reading_time":0`，不省略 |
| entry.Feed omitempty | 是 | save_entry 事件中 Feed 非 nil 则序列化，nil 则省略，两者签名不同 |
| tags 空切片 vs nil | 是 | Go `[]string(nil)` 序列化为 `null`，空切片 `[]string{}` 序列化为 `[]` |
| 时间格式 | 是 | `time.Time` 默认序列化为 RFC 3339，时区偏移会影响字节 |

#### 接收方验证方式

接收方应以相同 secret 对请求体计算 HMAC-SHA256，然后与 `X-Miniflux-Signature` 头进行常量时间比较（防时序攻击），伪代码：

```
expected = hex(hmac-sha256(webhookSecret, rawRequestBodyBytes))
valid    = constantTimeCompare(lowercase(expected), lowercase(X-Miniflux-Signature))
```

关键注意：必须对**原始请求体字节**（未解析、未 re-serialize）计算签名，因为 JSON 重新序列化可能改变字段顺序、空格等细节。

#### secret 为空时的行为

当 `webhookSecret` 为空字符串 `""` 时，`[]byte("")` 长度为 0，HMAC 仍然会计算出一个值。这意味着签名头仍会被设置，只是使用空密钥。接收方若不验证签名，则无法区分合法与非法请求。

#### 签名的安全属性

- **请求体完整性**：任何对请求体的篡改都会导致签名不匹配
- **来源认证**：只有持有 `webhookSecret` 的一方才能生成有效签名
- **硬编码算法**：当前硬编码 SHA-256，无算法协商，避免算法混淆攻击
- **secret 长度不限**：Go 的 `hmac.New` 内部会处理密钥过长（hash 后使用）或过短（补零）的情况

### 4.2 请求头设置

Webhook 请求设置以下自定义头 (`webhook/webhook.go:132-135`)：

| 头名称 | 值 | 用途 |
|-------|-----|------|
| `Content-Type` | `application/json` | 标识 JSON 请求体 |
| `User-Agent` | `Miniflux/{version}` | 标识客户端版本 |
| `X-Miniflux-Signature` | HMAC-SHA256 十六进制 | 请求体验证 |
| `X-Miniflux-Event-Type` | `new_entries` 或 `save_entry` | 区分事件类型 |

### 4.3 HTTP 客户端行为

Webhook 使用 `client.NewClientWithOptions` 创建 HTTP 客户端 (`webhook/webhook.go:137`)：

```go
httpClient := client.NewClientWithOptions(client.Options{
    Timeout:              defaultClientTimeout,        // 10 秒
    BlockPrivateNetworks: !config.Opts.IntegrationAllowPrivateNetworks(),
})
```

#### 超时设置

`defaultClientTimeout = 10 * time.Second` (`webhook/webhook.go:22`)。Go 的 `http.Client.Timeout` 是端到端超时，包含从发起连接到读取响应体的全部时间，包括重定向。超过 10 秒则请求失败并返回 `context.DeadlineExceeded` 错误。

#### 私有网络阻断

`INTEGRATION_ALLOW_PRIVATE_NETWORKS` 配置项默认为 `false` (`config/options.go:301-304`)，这意味着**默认阻断对私有网络的访问**。

当 `BlockPrivateNetworks = true` 时 (`http/client/client.go:31-69`)，客户端使用自定义 `http.Transport`，其 `DialContext` 在连接建立前解析目标主机 IP 并检查是否为非公开地址：

```go
DialContext: func(ctx context.Context, network, addr string) (net.Conn, error) {
    host, port, err := net.SplitHostPort(addr)
    ips, err := net.LookupIP(host)           // DNS 解析
    var safeIP net.IP
    for _, ip := range ips {
        if !urllib.IsNonPublicIP(ip) {       // 检查是否为公开 IP
            safeIP = ip
            break
        }
    }
    if safeIP == nil {
        return nil, fmt.Errorf("connection to private network is blocked")  // 阻断
    }
    safeAddr := net.JoinHostPort(safeIP.String(), port)
    return dialer.DialContext(ctx, network, safeAddr)  // 使用安全的 IP 连接
}
```

`urllib.IsNonPublicIP` (`urllib/url.go:166-181`) 检查的 IP 类型：

| IP 类型 | 阻断? | 示例 |
|---------|------|------|
| RFC 1918 私有地址 | 是 | `192.168.x.x`, `10.x.x.x`, `172.16-31.x.x` |
| RFC 6598 共享地址空间 (CGN) | 是 | `100.64.0.0/10` |
| 回环地址 | 是 | `127.0.0.1`, `::1` |
| 链路本地地址 | 是 | `169.254.x.x`, `fe80::` |
| 组播地址 | 是 | `224.x.x.x`, `ff02::` |
| 未指定地址 | 是 | `0.0.0.0`, `::` |
| 公开 IP | 否 | `93.184.216.34`, `2001:4860:4860::8888` |

**重要**：此检查在 DNS 解析后、TCP 连接前执行，避免了 DNS 重绑定（DNS-rebinding）和 TOCTOU 竞态攻击。自定义 Transport 确保每次连接都经过此检查。

#### HTTP 重定向跟随策略

`client.NewClientWithOptions` 返回的 `http.Client` **没有设置 `CheckRedirect` 字段**。这意味着使用 Go 标准库的默认重定向策略：

1. **自动跟随**：客户端自动跟随 HTTP 301/302/303/307/308 重定向
2. **最大 10 次**：Go 的 `http.Client` 默认允许最多 10 次重定向（超出返回错误）
3. **私有网络检查的覆盖范围**：自定义 `Transport.DialContext` 仅在 **TCP 连接建立时**检查 IP，不在重定向时重新检查。这意味着：
   - 初始请求被检查（例如禁止 `http://127.0.0.1/webhook`）
   - 重定向目标**也会被检查**，因为每次重定向都需要建立新的 TCP 连接，经过同一个 `DialContext`

4. **签名 header 在重定向中的转发行为**（以下基于 Go 标准库 `net/http/client.go` 源码精确追踪）：

   Go 的 `http.Client.do` 方法在处理重定向时，通过 `makeHeadersCopier` 创建一个 header 复制函数。该函数的行为如下：

   **关键代码路径** (`net/http/client.go:614-693`)：

   ```go
   // do 方法中初始化
   copyHeaders = c.makeHeadersCopier(req)

   // 重定向循环中
   if len(reqs) > 0 {
       // ...
       req = &Request{
           Method:   redirectMethod,
           Response: resp,
           URL:      u,
           Header:   make(Header),    // ← 创建空 Header map，而非复用原始 Header
           Host:     host,
           Cancel:   ireq.Cancel,
           ctx:      ireq.ctx,
       }
       // ...
       if !stripSensitiveHeaders && reqs[0].URL.Host != req.URL.Host {
           if !shouldCopyHeaderOnRedirect(reqs[0].URL, req.URL) {
               stripSensitiveHeaders = true
           }
       }
       copyHeaders(req, stripSensitiveHeaders, !includeBody)
   }
   ```

   **`makeHeadersCopier` 的核心逻辑** (`net/http/client.go:757-829`)：

   ```go
   func (c *Client) makeHeadersCopier(ireq *Request) func(...) {
       var ireqhdr = cloneOrMakeHeader(ireq.Header)  // ← 克隆初始请求的完整 Header

       return func(req *Request, stripSensitiveHeaders, stripBodyHeaders bool) {
           for k, vv := range ireqhdr {
               sensitive := false
               body := false
               switch CanonicalHeaderKey(k) {
               case "Authorization", "Www-Authenticate", "Cookie", "Cookie2",
                    "Proxy-Authorization", "Proxy-Authenticate":
                   sensitive = true
               case "Content-Encoding", "Content-Language", "Content-Location",
                    "Content-Type":
                   body = true
               }
               if !(sensitive && stripSensitiveHeaders) && !(body && stripBodyHeaders) {
                   req.Header[k] = vv  // ← 复制到新请求
               }
           }
       }
   }
   ```

   **`X-Miniflux-Signature` 和 `X-Miniflux-Event-Type` 的转发判断**：

   这两个自定义 header 不在 Go 标准库的 `sensitive` 列表（`Authorization`/`Cookie`/`Proxy-Authorization` 等）中，也不在 `body` 列表（`Content-Type`/`Content-Encoding` 等）中。因此：

   - **`stripSensitiveHeaders = false` 时**（同域重定向）：两个 header **无条件复制**到重定向请求
   - **`stripSensitiveHeaders = true` 时**（跨域重定向）：因为这两个 header 既不是 `sensitive` 也不是 `body`，所以**仍然复制**到重定向请求

   **`stripSensitiveHeaders` 的触发条件**：

   ```go
   // net/http/client.go:688-691
   if !stripSensitiveHeaders && reqs[0].URL.Host != req.URL.Host {
       if !shouldCopyHeaderOnRedirect(reqs[0].URL, req.URL) {
           stripSensitiveHeaders = true
       }
   }
   ```

   `shouldCopyHeaderOnRedirect` (`net/http/client.go:1005-1020`) 判断目标 host 是否是源 host 的子域：
   ```go
   func shouldCopyHeaderOnRedirect(initial, dest *url.URL) bool {
       ihost := idnaASCIIFromURL(initial)
       dhost := idnaASCIIFromURL(dest)
       return isDomainOrSubdomain(dhost, ihost)  // 目标是源的同域或子域 → true → 不剥离
   }
   ```

   综合判断：`stripSensitiveHeaders = true` 当且仅当目标 host **不是**源 host 的同域或子域。但即使 `stripSensitiveHeaders = true`，`X-Miniflux-Signature` 和 `X-Miniflux-Event-Type` 仍会被复制，因为它们**不在敏感 header 列表中**。

   **重定向中请求体的行为**：

   | 重定向状态码 | 请求方法 | 请求体 | Content-Type | 签名 header | 签名是否仍匹配请求体 |
   |-----------|---------|-------|-------------|------------|-----------------|
   | 301/302/303 | GET（POST→GET） | **丢弃** | **丢弃**（body=true, stripBodyHeaders=true） | **保留** | ❌ 签名基于原始请求体，但请求体已丢弃 |
   | 307/308 | POST（保持） | **保留**（如果 `GetBody` 可用） | 保留 | **保留** | ✅ 签名与请求体匹配 |

   **关键细节**：301/302/303 重定向时 POST→GET，`stripBodyHeaders=true` 导致 `Content-Type` 被剥离，但 `X-Miniflux-Signature` 和 `X-Miniflux-Event-Type` **不会被剥离**。重定向目标收到一个无请求体的 GET 请求，但仍然携带 `X-Miniflux-Signature` 和 `X-Miniflux-Event-Type` header——签名与空请求体不匹配，若目标验证签名则会失败。

   **安全影响**：签名 header 在跨域重定向中仍被转发，意味着如果 webhook URL 配置为 `https://example.com/webhook` 而该地址返回 302 重定向到 `https://attacker.com/capture`，攻击者可以捕获 `X-Miniflux-Signature` 和 `X-Miniflux-Event-Type` header。虽然攻击者无法伪造新签名（需要 secret），但他们可以：
   - 了解到 webhook 的签名格式和事件类型
   - 对已捕获的请求体+签名进行重放攻击（在签名时效内）

   **Webhook 使用 `http.NewRequest` 而非 `http.NewRequestWithContext`**，因此 `GetBody` 未被设置。307/308 重定向时，由于 `ireq.GetBody == nil && ireq.outgoingLength() != 0`，Go 的 `redirectBehavior` 会设置 `shouldRedirect = false`，直接返回 307/308 响应而不跟随重定向。这意味着 **307/308 重定向实际上不会被跟随**，Webhook 只会跟随 301/302/303（POST→GET，丢弃请求体）。

#### HTTP 客户端行为完整总结

| 行为 | 实现 |
|-----|------|
| 超时 | 10 秒端到端（含连接、TLS 握手、请求发送、响应读取、重定向） |
| 私有网络阻断 | 默认启用，DNS 解析后在 DialContext 层检查，覆盖重定向目标 |
| 重定向跟随 | 默认策略：自动跟随，最多 10 次 |
| TLS | 使用 Go 默认 `http.Transport`，验证证书链，不允许自签证书（除非全局配置） |
| HTTP/2 | 使用 Go 默认行为，自动协商 |
| 连接复用 | 无自定义 Transport 时使用默认连接池；启用私有网络阻断时使用自定义 Transport，默认启用 Keep-Alive |
| 请求方法 | 始终 POST |
| 响应判断 | `StatusCode >= 400` 视为失败 (`webhook/webhook.go:144`) |

---

## 五、高频 Provider（Discord / Telegram / Pushover）的配置缺失回退路径

这三个是 PushEntries 中使用频率最高的 provider，但其配置校验策略和缺省行为差异显著。以下从 `integration.go` 调用点到各 provider 内部实现逐层分析。

### 5.1 配置字段总览

| 配置项 | 所在层级 | 类型 | Discord | Telegram | Pushover |
|-------|---------|------|---------|----------|----------|
| 启用开关 | 用户级 | bool | `DiscordEnabled` | `TelegramBotEnabled` | `PushoverEnabled` |
| 启用开关 | Feed 级 | bool | 无 | 无 | `feed.PushoverEnabled`（AND） |
| 核心凭证 1 | 用户级 | string | `DiscordWebhookLink`（URL） | `TelegramBotToken` | `PushoverUser` |
| 核心凭证 2 | 用户级 | string | 无 | `TelegramBotChatID` | `PushoverToken` |
| 可选参数 | 用户级 | 见下 | 无 | `TelegramBotTopicID` (*int64)、DisableWebPagePreview、DisableNotification、DisableButtons | `PushoverDevice`、`PushoverPrefix` |
| 可选参数 | Feed 级 | 见下 | 无 | 无 | `PushoverPriority` (int) |

### 5.2 Discord：零校验 + 空 URL 直接请求失败

#### 调用点 (`integration.go:613-627`)

```go
if userIntegrations.DiscordEnabled {
    client := discord.NewClient(userIntegrations.DiscordWebhookLink)
    if err := client.SendDiscordMsg(feed, entries); err != nil {
        slog.Warn("Unable to send new entries to Discord", slog.Any("error", err))
    }
}
```

**调用前置检查**：仅检查 `userIntegrations.DiscordEnabled == true`，**不检查** `DiscordWebhookLink` 是否为空、是否为合法 URL。

#### `discord.NewClient` (`discord/discord.go:30-32`)

```go
func NewClient(webhookURL string) *Client {
    return &Client{webhookURL: webhookURL}
}
```
仅保存 URL，**零校验**。

#### `discord.SendDiscordMsg` (`discord/discord.go:34-95`)

逐条 entry 发送，失败即停止：
```go
func (c *Client) SendDiscordMsg(feed *model.Feed, entries model.Entries) error {
    for _, entry := range entries {
        requestBody, err := json.Marshal(&discordMessage{...})
        // ...
        request, err := http.NewRequest(http.MethodPost, c.webhookURL, bytes.NewReader(requestBody))
        // ...
        httpClient := client.NewClientWithOptions(client.Options{
            Timeout: defaultClientTimeout,
            BlockPrivateNetworks: !config.Opts.IntegrationAllowPrivateNetworks(),
        })
        response, err := httpClient.Do(request)
        // ...
        if response.StatusCode >= 400 {
            return fmt.Errorf("discord: unable to send a notification: url=%s status=%d", c.webhookURL, response.StatusCode)
        }
    }
    return nil
}
```

**缺失配置的实际回退/错误路径**：

| 缺失配置 | 行为 | 错误时机 |
|---------|------|---------|
| `DiscordWebhookLink == ""` | `http.NewRequest` 接受空 URL（不报错），但 `httpClient.Do("")` 尝试连接空 host 失败 | HTTP 连接阶段：`Post "": http: no Host in request URL` 或 DNS 解析错误 |
| `DiscordWebhookLink` 非 URL（如 "abc"） | `http.NewRequest` 不校验 URL 格式，`httpClient.Do` 在解析时失败 | URL 解析或连接阶段 |
| `DiscordWebhookLink` 指向私有 IP | 若 `INTEGRATION_ALLOW_PRIVATE_NETWORKS=false`（默认），被 `DialContext` 阻断 | TCP 连接前：`connection to private network is blocked` |
| `DiscordWebhookLink` 合法但无效（如被删除） | Discord API 返回 401/404 | HTTP 响应阶段：`status=401` / `status=404` |

#### Discord `embed_color` 的完整 fall-through 路径

Discord 通知使用 Rich Embed 格式，`color` 字段控制 embed 左侧竖条的颜色。完整路径逐层分析：

**第 1 层：数据库层**——`integrations` 表中**没有** `discord_color`/`discord_embed_color` 列，`feeds` 表中也没有 Discord 相关颜色列。`model.Integration` 结构体中无 `DiscordColor`/`DiscordEmbedColor` 字段，`model.Feed` 结构体中也无。

**第 2 层：表单/配置层**——`ui/form/integration.go` 中没有 Discord 颜色表单字段。用户在 Web UI 的集成设置页面没有任何地方可以配置 embed 颜色。

**第 3 层：常量硬编码**——`discord/discord.go:24`：
```go
const discordMsgColor = 5793266
```
十进制 `5793266` = 十六进制 `0x5865F2` = RGB (88, 101, 242)，即 **Discord 品牌蓝色**（与 Slack 的 `#5865F2` 完全相同，见 `slack/slack.go:24`）。

**第 4 层：消息构建**——`discord/discord.go:36-63`：
```go
json.Marshal(&discordMessage{
    Embeds: []discordEmbed{
        {
            Title: "RSS feed update from Miniflux",
            Color: discordMsgColor,    // ← 直接赋值常量，无判断、无覆盖
            Fields: []discordFields{...},
        },
    },
})
```
直接硬编码赋值，没有任何 `if config != nil` 或 `if userColor != 0` 的判断逻辑。

**第 5 层：结构体 JSON 序列化**——`discord/discord.go:103-107`：
```go
type discordEmbed struct {
    Title  string          `json:"title"`
    Color  int             `json:"color"`   // ← 注意：无 omitempty
    Fields []discordFields `json:"fields"`
}
```

**关键点：`Color` 字段的 JSON tag `"color"` **无 `omitempty`**，因此：

| `Color` 赋值情况 | JSON 输出 | Discord 实际显示颜色 |
|----------------|---------|------------------|
| `Color: 5793266`（当前代码） | `"color":5793266` | Discord 品牌蓝 `#5865F2` ✅ |
| `Color: 0`（假设未来漏赋值） | `"color":0` | **黑色 `#000000`** ❌ |
| **如加了 `omitempty`** 且 `Color: 0` | `"color"` 字段完全省略 | Discord 默认颜色（通常是深灰 `#202225` 或浅色模式中的中性灰） |

**结论**：Discord embed_color 在完整路径中**每一层都没有用户可配置入口**，完全由代码常量 `discordMsgColor = 5793266` (`#5865F2`) 决定。不同于 Pushover 的 sound 字段（结构体中都不存在），Discord 的 color 字段在结构体中存在且有值，但同样**不可自定义**。

如果将来有人重构时误将 `Color: discordMsgColor` 删除（Go int 零值 0），由于无 `omitempty`，会导致 `"color":0` 发送给 Discord，embed 变为纯黑色而非默认色，这是一个容易忽略的退化点。

**关键特性**：
- 逐条发送（for-range 每条 entry 一次请求），某条失败后**后续 entry 不再发送**（return error）
- `response.Body.Close()` 没有使用 `defer`（`discord/discord.go:87`）——正常返回不影响，但如果 return error 前未 close，不会泄漏（因为 httpClient.Do 返回 err 时 response 为 nil）
- 没有重试机制，一次失败即放弃

### 5.3 Telegram Bot：无前置校验 + 空 Token/ChatID 让 Telegram API 报错

#### 调用点 (`integration.go:666-692`)

```go
if userIntegrations.TelegramBotEnabled {
    for _, entry := range entries {
        if err := telegrambot.PushEntry(
            feed,
            entry,
            userIntegrations.TelegramBotToken,
            userIntegrations.TelegramBotChatID,
            userIntegrations.TelegramBotTopicID,
            userIntegrations.TelegramBotDisableWebPagePreview,
            userIntegrations.TelegramBotDisableNotification,
            userIntegrations.TelegramBotDisableButtons,
        ); err != nil {
            slog.Error("Unable to send entry to Telegram", ...)
        }
    }
}
```

**调用前置检查**：仅检查 `TelegramBotEnabled == true`，不校验 Token、ChatID 是否为空。

#### `telegrambot.PushEntry` (`telegrambot/telegrambot.go:16-65`)

```go
func PushEntry(feed *model.Feed, entry *model.Entry, botToken, chatID string, topicID *int64,
    disableWebPagePreview, disableNotification bool, disableButtons bool) error {

    formattedText := fmt.Sprintf(`<b>%s</b> - <a href=%q>%s</a>`, feed.Title, entry.URL, entry.Title)
    message := &MessageRequest{
        ChatID:                chatID,
        Text:                  formattedText,
        ParseMode:             HTMLFormatting,
        DisableWebPagePreview: disableWebPagePreview,
        DisableNotification:   disableNotification,
    }
    if topicID != nil {
        message.MessageThreadID = *topicID
    }
    if !disableButtons {
        // 构建 InlineKeyboard 按钮...
        // 若 config.Opts.BaseURL() + entryPath 拼接失败，仅 slog.Error，不中断发送
    }
    client := NewClient(botToken, chatID)
    _, err := client.SendMessage(message)
    return err
}
```

**可选参数回退逻辑**（`PushEntry` 内部）：

| 参数 | 回退行为 | 代码位置 |
|-----|---------|---------|
| `topicID == nil` | 不设置 `MessageThreadID` 字段（JSON `omitempty`），Telegram 发送到默认话题 | `telegrambot.go:32-34` |
| `disableWebPagePreview == false`（默认） | 发送 `disable_web_page_preview:false`，Telegram 展示网页预览 | 直接使用 bool 零值 |
| `disableNotification == false`（默认） | 发送 `disable_notification:false`，正常通知 | 直接使用 bool 零值 |
| `disableButtons == false`（默认） | 构建内联按钮；若 `BaseURL` 拼接失败，仅日志记录，按钮减一 | `telegrambot.go:36-60` |
| `entry.CommentsURL == ""` | 不添加"Comments"按钮 | `telegrambot.go:53-56` |

#### `telegrambot.Client.SendMessage` (`telegrambot/client.go:74-111`)

```go
func (c *Client) SendMessage(message *MessageRequest) (*Message, error) {
    endpointURL, err := url.JoinPath(telegramAPIEndpoint, "/bot"+c.botToken, "/sendMessage")
    // ...
    requestBody, err := json.Marshal(message)
    // ...
    request, err := http.NewRequest(http.MethodPost, endpointURL, bytes.NewReader(requestBody))
    // ...
    httpClient := client.NewClientWithOptions(client.Options{Timeout: defaultClientTimeout})
    // 注意：Telegram 客户端未设置 BlockPrivateNetworks！
    response, err := httpClient.Do(request)
    // ...
    if !messageResponse.Ok {
        return nil, fmt.Errorf("telegram: unable to send message: %s (error code is %d)",
            messageResponse.Description, messageResponse.ErrorCode)
    }
}
```

**缺失配置的实际回退/错误路径**：

| 缺失配置 | 行为 | 错误时机 |
|---------|------|---------|
| `TelegramBotToken == ""` | `url.JoinPath("https://api.telegram.org", "/bot", "/sendMessage")` → `"https://api.telegram.org/bot/sendMessage"`，Telegram API 返回 401 Unauthorized | Telegram API 响应：`error code is 401` |
| `TelegramBotChatID == ""` | `chat_id` JSON 字段为空字符串，Telegram API 返回 400 Bad Request | Telegram API 响应：`error code is 400, "Bad Request: chat not found"` |
| `TelegramBotTopicID == nil` | 正常回退：不发送话题 ID，发送到群组/频道的默认话题 | 无错误 |
| `TelegramBotTopicID` 设为非法值（如不存在的话题） | Telegram API 返回 400 | Telegram API 响应 |

#### Telegram `parse_mode`：为什么不存在 "MarkdownV2 转义失败回退到 plain text" 的逻辑

用户可能会担心：如果使用 `MarkdownV2` 作为 `parse_mode` 但文本内容含有未转义的特殊字符（`_ * [ ] ( ) ~ `` > # + - = | { } . !`），Telegram API 会返回错误。但这个问题**在 miniflux 的实现中根本不会发生**，原因如下：

**第 1 层：parse_mode 是硬编码常量**——`telegrambot/telegrambot.go:27`：
```go
message := &MessageRequest{
    ParseMode: HTMLFormatting,   // ← 不是 MarkdownV2，是 HTML
    ...
}
```

常量定义 (`telegrambot/client.go:18-25`)：
```go
const (
    MarkdownFormatting   = "Markdown"     // 老版本 Markdown，已被 Telegram 标记为过时
    MarkdownV2Formatting = "MarkdownV2"   // 新版 Markdown，需要转义 18 个特殊字符
    HTMLFormatting       = "HTML"          // ← 当前实际使用
)
```

代码中虽然定义了 `MarkdownFormatting` 和 `MarkdownV2Formatting` 两个常量，但这两个常量**在整个代码库中从未被赋值使用**，仅 `HTMLFormatting` 被实际赋值给 `ParseMode` 字段。

**第 2 层：文本构造使用 HTML 标签**——`telegrambot/telegrambot.go:17-22`：
```go
formattedText := fmt.Sprintf(
    `<b>%s</b> - <a href=%q>%s</a>`,
    feed.Title,    // ← 直接插入，未做 HTML 转义
    entry.URL,     // ← 用 %q 加了双引号，也未做 HTML 属性转义
    entry.Title,   // ← 直接插入，未做 HTML 转义
)
```

文本内容（`feed.Title`、`entry.Title`）直接插入 HTML 标签之间，**没有任何 HTML 实体转义**（`&` → `&amp;`、`<` → `&lt;`、`>` → `&gt;`）。这意味着：

- 如果 `entry.Title` 包含 `<script>` 之类的字符串，JSON 中会发送 `<b>Feed Title</b> - <a href="...">Article with <script> tag</a>`，Telegram 的 HTML 解析器会按**安全白名单**解析——只允许 Telegram 支持的标签（`<b>`, `<i>`, `<a>`, `<code>`, `<pre>` 等），其他标签被移除
- 不存在 MarkdownV2 的 18 个特殊字符转义问题，因为压根没用 Markdown
- 但存在**HTML 注入风险**：恶意 RSS feed 可以通过文章标题包含 Telegram 允许的 HTML 标签，实现"标题加粗"、"插入链接"等效果——不过这在 Telegram 的安全白名单范围内，且仅影响显示效果，不影响账号安全

**第 3 层：`ParseMode` 的 JSON omitempty 行为**——`telegrambot/client.go:162-170`：
```go
type MessageRequest struct {
    ...
    ParseMode string `json:"parse_mode,omitempty"`
    ...
}
```

如果未来有人重构时**误将 `ParseMode: HTMLFormatting` 一行删除**，Go string 零值 `""` 会触发 `omitempty`，JSON 中 `"parse_mode"` 字段被完全省略。此时 Telegram API 的回退行为是：

| `ParseMode` 赋值 | JSON 输出 | Telegram 解析方式 |
|----------------|---------|----------------|
| `HTMLFormatting`（当前） | `"parse_mode":"HTML"` | **HTML 模式**：`<b>` `<a>` 等标签生效 ✅ |
| `MarkdownFormatting`（定义但未使用） | `"parse_mode":"Markdown"` | 旧版 Markdown 模式（Telegram 已弃用） |
| `MarkdownV2Formatting`（定义但未使用） | `"parse_mode":"MarkdownV2"` | 新版 Markdown，需转义 18 个特殊字符 |
| 删除赋值，保留字段（零值 `""`） | `"parse_mode"` 字段被省略（omitempty） | **纯文本模式**：所有标签原样显示 ❌ （`<b>` 不生效，字面显示） |

**结论**：

1. miniflux **根本不使用 MarkdownV2**，所以不存在 "MarkdownV2 转义失败回退到 plain text" 的逻辑分支——整个代码库没有任何地方执行 Markdown 特殊字符转义。
2. 实际使用 HTML 模式，但文本内容**未做 HTML 实体转义**，理论上存在注入 Telegram 白名单标签的风险（显示层，非安全漏洞）。
3. 如果未来重构时误删除 `ParseMode: HTMLFormatting`，`omitempty` 会导致字段被省略，Telegram 回退到纯文本模式，所有格式标签会被原样显示（`"<b>Feed Title</b>"` 而非加粗文字），这是一个容易忽略的退化点。

**关键特性**：
- 逐条发送，某条失败后 `slog.Error` 但**继续发送下一条**（不 return）——与 Discord 不同
- Telegram 客户端**没有设置** `BlockPrivateNetworks`（`telegrambot/client.go:94`），与 Webhook/Discord/Pushover 不同，这意味着若配置了恶意的 botToken 指向私有地址（实际不大可能，因为 botToken 不是 URL），不会被 SSRF 防护拦截——但此处风险可忽略，因为 Telegram API Endpoint 是硬编码的 `https://api.telegram.org`
- 没有重试机制，一次失败即放弃该条 entry

### 5.4 Pushover：前置校验 + 多项默认回退

#### 调用点 (`integration.go:645-663`)

```go
if userIntegrations.PushoverEnabled && feed.PushoverEnabled {
    client := pushover.NewClient(
        userIntegrations.PushoverUser,
        userIntegrations.PushoverToken,
        feed.PushoverPriority,
        userIntegrations.PushoverDevice,
        userIntegrations.PushoverPrefix,
    )
    if err := client.SendMessages(feed, entries); err != nil {
        slog.Warn("Unable to send new entries to Pushover", slog.Any("error", err))
    }
}
```

**调用前置检查**：`PushoverEnabled`（用户级）AND `feed.PushoverEnabled`（Feed 级），但不检查 `PushoverUser`/`PushoverToken` 是否为空。

#### `pushover.NewClient`：集中式回退与边界校验 (`pushover/pushover.go:56-74`)

```go
func NewClient(user, token string, priority int, device, urlPrefix string) *Client {
    if urlPrefix == "" {
        urlPrefix = defaultPushoverURL    // "https://api.pushover.net"
    }
    if priority < -2 {
        priority = -2                      // 最低优先级钳制
    }
    if priority > 2 {
        priority = 2                       // 最高优先级钳制
    }
    return &Client{
        user:     user,
        token:    token,
        device:   device,
        prefix:   urlPrefix,
        priority: priority,
    }
}
```

**`NewClient` 中的回退逻辑**：

| 参数 | 回退条件 | 回退值 | 说明 |
|-----|---------|-------|------|
| `urlPrefix` | `""` | `"https://api.pushover.net"` | Pushover 官方 API Endpoint |
| `priority` | `< -2` | `-2` | Pushover 最低优先级（静默通知） |
| `priority` | `> 2` | `2` | Pushover 最高优先级（紧急，需要确认） |
| `device` | 任意值（含 `""`） | 原样保存 | 空字符串表示发送到用户的所有设备 |

#### `pushover.SendMessages`：空凭证前置检查与字段 fall-through 路径 (`pushover/pushover.go:76-104`)

```go
func (c *Client) SendMessages(feed *model.Feed, entries model.Entries) error {
    if c.token == "" || c.user == "" {
        return errors.New("pushover token and user are required")
    }
    for _, entry := range entries {
        msg := &message{
            User:     c.user,
            Token:    c.token,
            Device:   c.device,    // 空字符串时 JSON 中带 omitempty，不序列化
            Message:  entry.Title,
            Title:    feed.Title,
            Priority: c.priority,
            URL:      entry.URL,
        }
        if err := c.makeRequest(msg); err != nil {
            return fmt.Errorf("pushover: unable to send message: %w", err)
        }
    }
    return nil
}
```

这是三个 provider 中**唯一在发送前做凭证非空校验**的。

#### Pushover `message` 结构体的完整字段 fall-through 分析

`message` 结构体定义 (`pushover/pushover.go:36-47`)：

```go
type message struct {
    Token string `json:"token"`
    User  string `json:"user"`

    Title    string `json:"title"`
    Message  string `json:"message"`
    Priority int    `json:"priority"`

    URL      string `json:"url"`
    URLTitle string `json:"url_title"`
    Device   string `json:"device,omitempty"`
}
```

**Pushover API 支持但 miniflux 未实现的字段**：

Pushover Messages API（`https://pushover.net/api#messages`）支持以下字段，但 miniflux 的 `message` 结构体中**不存在**这些字段：

| Pushover API 字段 | 类型 | miniflux 是否支持 | 缺失时的 Pushover 默认行为 |
|------------------|------|----------------|------------------------|
| `sound` | string | **不支持** | Pushover 使用用户设备上配置的默认通知声音 |
| `timestamp` | int (Unix) | **不支持** | Pushover 使用消息到达服务器的时间 |
| `callback` | string (URL) | **不支持** | 不设置回调（仅 priority=2 紧急通知可用） |
| `retry` | int (秒) | **不支持** | 不重试（仅 priority=2 时必须，缺失则 API 返回错误） |
| `expire` | int (秒) | **不支持** | 不设置过期（仅 priority=2 时必须，缺失则 API 返回错误） |
| `html` | int (0/1) | **不支持** | 消息按纯文本处理 |
| `monospace` | int (0/1) | **不支持** | 消息按比例字体显示 |
| `attachment` | base64 | **不支持** | 不附带图片 |

##### `sound` 字段的完整 fall-through 路径

`sound` 是用户最容易期望但缺失的字段。完整 fall-through 路径如下：

1. **数据库层**：`integrations` 表中没有 `pushover_sound` 列（`migrations.go:1048-1052` 只添加了 `pushover_enabled`/`pushover_user`/`pushover_token`/`pushover_device`/`pushover_prefix`），`feeds` 表也没有

2. **模型层**：`model.Integration` 结构体中没有 `PushoverSound` 字段（`integration.go:124-128`），`model.Feed` 结构体中也没有

3. **表单层**：`ui/form/integration.go:377-381` 中没有 `pushover_sound` 表单字段

4. **调用层**：`integration.go:645-663` 中 `pushover.NewClient(...)` 不传递 sound 参数

5. **客户端层**：`pushover.NewClient(...)` 签名为 `(user, token string, priority int, device, urlPrefix string)`，无 sound 参数

6. **消息构建层**：`pushover.SendMessages` 中 `msg := &message{...}` 不设置 `Sound` 字段

7. **序列化层**：`message` 结构体中无 `Sound` 字段，`json.Marshal` 不会输出 `"sound"` key

8. **Pushover API 层**：收到 JSON 中无 `"sound"` key，**使用用户在 Pushover 客户端/应用设置中配置的默认通知音**

**结论**：`sound` 字段从数据库到 API 的完整路径中**每一层都没有对应字段**，不是"有字段但为空"的情况，而是**完全不存在**。Pushover API 的回退行为是使用用户设备级默认设置，这意味着同一 miniflux 实例的所有通知都使用相同默认声音，无法按 Feed 或用户区分。

##### `URLTitle` 字段的 fall-through 路径

`URLTitle` 在 `message` 结构体中存在，但 `SendMessages` 中未赋值：

```go
msg := &message{
    // ...
    URL:      entry.URL,
    // URLTitle 未赋值 → 零值 ""
}
```

由于 `URLTitle` 的 JSON tag 是 `"url_title"`（**无 `omitempty`**），空字符串 `""` 仍会被序列化为 `"url_title":""` 发送给 Pushover API。Pushover 收到空 `url_title` 时，显示 URL 本身作为链接文本（等同于 `url_title` 未设置的效果）。

##### priority=2 时缺失 `retry`/`expire`/`callback` 的实际后果

当 `feed.PushoverPriority` 被钳制为 `2`（紧急优先级）时，Pushover API **要求**同时提供 `retry` 和 `expire` 字段。但 miniflux 的 `message` 结构体中不存在这两个字段，因此：

- **Pushover API 返回错误**：`{"errors":["Emergency priority messages require the 'retry' and 'expire' parameters."]}` 
- `makeRequest` 会捕获此错误（`resp.StatusCode >= 400`），解析 `errResp.Errors` 并返回
- `SendMessages` 中 `return fmt.Errorf("pushover: unable to send message: %w", err)`，后续 entry 不再发送
- **实际影响**：任何 Feed 级别设置了 `pushover_priority=2` 的通知都会**必然失败**，且该 Feed 的所有 entry 都不会被推送（第一条失败即停止）

这是一个**功能性缺陷**：`NewClient` 允许 priority=2 但不发送必需的 `retry`/`expire` 字段，导致紧急优先级实际上不可用。

#### `pushover.makeRequest`：错误响应解析 (`pushover/pushover.go:106-141`)

```go
func (c *Client) makeRequest(payload *message) error {
    url := c.prefix + "/1/messages.json"
    // ...
    httpClient := client.NewClientWithOptions(client.Options{
        Timeout:              defaultClientTimeout,
        BlockPrivateNetworks: !config.Opts.IntegrationAllowPrivateNetworks(),
    })
    resp, err := httpClient.Do(req)
    // ...
    if resp.StatusCode >= http.StatusBadRequest {
        errorMessage := resp.Status
        var errResp errorResponse
        if err := json.NewDecoder(resp.Body).Decode(&errResp); err == nil {
            if len(errResp.Errors) > 0 {
                errorMessage = strings.Join(errResp.Errors, ",")
            }
        }
        return fmt.Errorf("pushover: API error: status=%d %s", resp.StatusCode, errorMessage)
    }
}
```

**缺失配置的实际回退/错误路径**：

| 缺失配置 | 行为 | 错误时机 |
|---------|------|---------|
| `PushoverUser == ""` 或 `PushoverToken == ""` | `SendMessages` 前置检查直接返回错误 | 发送前：`"pushover token and user are required"` |
| `PushoverDevice == ""` | JSON omitempty 不序列化，Pushover 发送到用户全部设备 | 无错误，正常回退 |
| `PushoverPrefix == ""` | `NewClient` 回退到 `"https://api.pushover.net"` | 无错误，正常回退 |
| `feed.PushoverPriority < -2` | `NewClient` 钳制为 `-2` | 无错误，正常回退 |
| `feed.PushoverPriority == 2` | **Pushover API 必返回错误**（缺少 `retry`/`expire`） | Pushover API 响应：`"Emergency priority messages require the 'retry' and 'expire' parameters."` |
| `feed.PushoverPriority > 2` | `NewClient` 钳制为 `2`，同上 | 同上 |
| `feed.PushoverPriority == 0`（默认） | 直接使用 `0`，表示 Pushover 正常优先级 | 无错误 |
| `sound` 字段 | 从数据库到 API 完全不存在，Pushover 使用用户设备默认声音 | 无错误，但不可自定义 |
| `URLTitle` 字段 | 结构体有字段但未赋值，序列化为 `"url_title":""`，Pushover 显示 URL 本身作为链接文本 | 无错误 |

**关键特性**：
- 逐条发送，某条失败后**后续 entry 不再发送**（return error）——与 Discord 一致，与 Telegram 不同
- 是三个 provider 中唯一具备**前置空凭证校验**和**参数边界钳制**的
- `BlockPrivateNetworks` 已设置，防 SSRF
- 解析 Pushover API 返回的 JSON 错误消息体，错误信息比 Discord 更详细
- 没有重试机制

### 5.5 三个 Provider 回退路径对比总结

| 维度 | Discord | Telegram Bot | Pushover |
|-----|---------|-------------|----------|
| 启用开关层级 | 仅用户级 | 仅用户级 | 用户级 AND Feed 级 |
| 发送前凭证校验 | 无 | 无 | **有**（`token`/`user` 非空检查） |
| URL/Template 回退 | 无 | API Endpoint 硬编码 | Prefix 空时回退官方 API |
| 参数边界钳制 | 无 | 无 | **有**（priority 钳制 [-2, 2]） |
| 可选参数回退 | 无 | topicID nil → 默认话题；空 CommentsURL → 无评论按钮 | device 空 → 全部设备 |
| 失败后行为 | 立即停止，后续 entry 不发 | 记录日志，继续下一条 | 立即停止，后续 entry 不发 |
| 逐条 vs 批量 | 逐条 | 逐条 | 逐条 |
| 错误日志级别 | `slog.Warn` | `slog.Error` | `slog.Warn` |
| BlockPrivateNetworks | 已设置 | **未设置**（风险极低，硬编码 API URL） | 已设置 |
| 错误详情 | 仅 HTTP 状态码 | Telegram API 描述 + 错误码 | Pushover API 错误数组拼接 |



## 六、完整流程时序图

### 6.1 手动保存流程 (SendEntry)

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

### 6.2 Feed 刷新推送流程 (PushEntries)

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

## 七、关键文件索引

| 功能 | 文件路径 |
|-----|---------|
| 集成入口（SendEntry/PushEntries） | `internal/integration/integration.go` |
| 集成配置模型 | `internal/model/integration.go` |
| Feed 模型（含 Feed 级集成开关） | `internal/model/feed.go` |
| Feed 刷新核心处理 | `internal/reader/handler/handler.go` |
| 后台调度器 | `internal/cli/scheduler.go` |
| Worker 池 | `internal/worker/pool.go` |
| Worker 执行逻辑 | `internal/worker/worker.go` |
| 守护进程启动与关闭 | `internal/cli/daemon.go` |
| Web UI 保存入口 | `internal/ui/entry_save.go` |
| REST API 保存入口 | `internal/api/entry_handlers.go:232` |
| Fever API 保存入口 | `internal/fever/handler.go:401` |
| Google Reader API 保存入口 | `internal/googlereader/handler.go:187` |
| Webhook Provider 实现 | `internal/integration/webhook/webhook.go` |
| Discord Provider 实现 | `internal/integration/discord/discord.go` |
| Telegram Bot Provider 实现 | `internal/integration/telegrambot/telegrambot.go`、`internal/integration/telegrambot/client.go` |
| Pushover Provider 实现 | `internal/integration/pushover/pushover.go` |
| HMAC-SHA256 签名生成 | `internal/crypto/crypto.go` |
| HTTP 客户端（含私有网络阻断） | `internal/http/client/client.go` |
| IP 公开/私有判断 | `internal/urllib/url.go` |
| 配置选项（含 INTEGRATION_ALLOW_PRIVATE_NETWORKS） | `internal/config/options.go` |
| 数据库迁移 | `internal/database/migrations.go` |

---

## 八、总结

1. **事件触发分散**：两类事件（保存/推送）分别由 4 个和 4 个入口触发，共计 8 处触发点，统一管理困难。

2. **Provider 分流逻辑内聚但层级不一**：所有 30+ provider 的选择判断集中在 `internal/integration/integration.go` 一个文件中，但层级差异显著——仅 4 个 provider 有 Feed 级配置覆盖（Webhook、Ntfy、Apprise、Pushover），仅 2 个 provider 有 Feed 级开关（Ntfy、Pushover），其余 25+ 个 provider 仅凭用户级开关一锤定音。Feed 级开关只在 `PushEntries` 中生效，`SendEntry` 不受任何 Feed 级开关约束。

3. **无持久化重试**：所有通知采用 Fire-and-Forget 异步模式，失败仅记录日志，无队列、无重试、无死信处理。任何通知发送失败（网络错误、第三方服务不可用等）都会导致该条通知永久丢失。

4. **Worker 池从 API 设计上就没有提供等待 in-flight 通知的能力**：`Pool` 结构体的 `queue` 和 `wg` 均为私有字段，对外只暴露 `Push`、`Shutdown`、`NewPool` 三个方法。没有 `Drain()` 方法、没有第二个 WaitGroup 追踪通知 goroutine、没有 context 传递路径。调用方 `startDaemon` 在 `pool.Shutdown()` 返回后立即输出 "Process gracefully stopped" 并退出，完全没有给 in-flight 通知 goroutine 预留缓冲时间。Graceful Shutdown 的"优雅"只覆盖了 worker 本身，不覆盖通知。

5. **Graceful Shutdown 时通知 goroutine 既不被 kill 也不被等待**：通知 goroutine 在进程终止时经历"孤儿→被操作系统强制终止"的路径。对正在发送中的 HTTP 请求，副作用取决于请求进度阶段——最严重的不是"通知丢失"（在 Fire-and-Forget 设计下属于预期行为），而是**静默成功**：请求体已完整到达服务端、通知已生效，但 miniflux 侧无任何记录。对于有状态通知（书签创建），可能导致用户重复操作。

6. **Webhook 签名在重定向中始终被转发，即使跨域**：基于 Go 标准库 `net/http/client.go` 的 `makeHeadersCopier` 源码追踪，`X-Miniflux-Signature` 和 `X-Miniflux-Event-Type` 不在 `sensitive` 列表（`Authorization`/`Cookie` 等）中，也不在 `body` 列表（`Content-Type` 等）中，因此在任何重定向场景下都会被复制到新请求。301/302/303 重定向时 POST→GET 丢弃请求体，但签名 header 仍被转发，导致签名与空请求体不匹配；307/308 重定向时因 `GetBody` 未设置（`http.NewRequest` 不自动设置），实际上**不会跟随重定向**，直接返回 307/308 响应。跨域重定向存在签名 header 泄露风险。

7. **Webhook 签名依赖 JSON 结构体字段顺序**：HMAC-SHA256 签名头键名为 `X-Miniflux-Signature`（HTTP/2 下为 `x-miniflux-signature`），使用 Go 标准库小写十六进制编码。签名计算基于 `json.Marshal(payload)` 的完整原始字节输出，Go 的结构体字段声明顺序直接决定签名值，任何字段顺序调整、struct tag 改名、`omitempty` 触发都会导致签名变化。接收方必须对原始请求体字节计算签名。

8. **Discord embed_color 完全不可自定义，且零值会退化为黑色**：从数据库→模型→表单→客户端→消息构建全链路均无用户可配置入口，硬编码为十进制 `5793266` = `#5865F2`（Discord 品牌蓝）。`discordEmbed.Color` 字段无 `omitempty`，如果重构时漏赋值（Go int 零值 0），JSON 会输出 `"color":0`，Discord 显示纯黑色而非服务端默认色，这是一个容易忽略的退化点。

9. **Telegram 根本不使用 MarkdownV2，所以不存在转义失败回退的问题**：代码中定义了 `MarkdownFormatting`/`MarkdownV2Formatting`/`HTMLFormatting` 三个常量，但仅 `HTMLFormatting` 被实际赋值使用，整个代码库没有任何 Markdown 特殊字符转义逻辑。文本内容（feed.Title、entry.Title）直接插入 HTML 标签之间且**未做 HTML 实体转义**，存在 Telegram 白名单标签注入风险（仅显示层）。若未来重构时误删 `ParseMode: HTMLFormatting`，`omitempty` 会触发字段省略，Telegram 回退到纯文本模式，格式标签原样显示。

10. **Pushover 字段 fall-through 从数据库到 API 逐层缺失**：`sound`、`timestamp`、`callback`、`retry`、`expire`、`html`、`monospace`、`attachment` 八个 Pushover API 字段从数据库列→模型字段→表单→调用→客户端→消息构建→序列化每一层都不存在，不是"有字段但为空"，而是**完全不存在**。`sound` 缺失时 Pushover 使用用户设备默认声音；`URLTitle` 存在于结构体但未赋值，空字符串仍被序列化发送；最严重的功能性缺陷是 **priority=2（紧急）时缺失必需的 `retry`/`expire` 字段，导致 Pushover API 必然返回错误**，紧急优先级实际上完全不可用。

11. **三个高频 Provider 回退策略差异显著**：
    - **Discord**：零校验，空/非法 URL 直接在 HTTP 请求阶段失败，失败后停止后续 entry 发送
    - **Telegram**：无前置校验，空 Token/ChatID 由 Telegram API 返回错误，失败后继续发送下一条 entry；客户端未设置 `BlockPrivateNetworks`（风险极低）
    - **Pushover**：唯一具备前置空凭证校验和参数边界钳制（priority [-2, 2]），空 Prefix 回退官方 API，空 Device 回退全部设备，失败后停止后续 entry 发送；但 priority=2 是功能性死路

12. **可观测性薄弱**：缺乏通知发送成功率、延迟等指标，也无告警机制。
