# Miniflux 启动引导路径全解析

本文档沿着代码从 `main()` 到首个 HTTP 请求可被服务之间的完整链路，详细拆解配置加载、数据库连接、Schema 迁移、以及后台服务启动的顺序与依赖关系。

---

## 一、全局启动流程总览

### 入口调用链

```
main.go:11  main()
  └─ cli.Parse()                                internal/cli/cli.go:40
       │
       ├─ 1. 命令行参数解析（flag.Parse）
       ├─ 2. 配置加载（三阶段：默认值 → 配置文件 → 环境变量）
       ├─ 3. 配置合法性校验
       ├─ 4. 日志系统初始化
       ├─ 5. 静态资源 Bundle 生成（二进制/CSS/JS）
       ├─ 6. 数据库连接池建立 + Ping
       ├─ 7. [可选分支] 仅迁移 / 单次命令模式（提前 return）
       ├─ 8. 数据库迁移执行（RUN_MIGRATIONS=true）
       ├─ 9. Schema 版本一致性检查
       ├─ 10. 管理员自动创建（CREATE_ADMIN=true）
       ├─ 11. 代理旋转器初始化
       ├─ 12. [可选分支] 仅刷新 feeds / 清理任务（提前 return）
       └─ 13. startDaemon(store)                internal/cli/daemon.go:23
             ├─ Worker 池创建
             ├─ 调度器启动（feed 刷新 + 清理定时任务）
             ├─ HTTP 服务器启动（所有监听地址并行）
             ├─ Metrics 采集协程
             └─ Systemd 就绪通知（READY=1 + Watchdog）
```

**关键结论**：整个启动过程是**严格串行**的，前一步失败会直接 `os.Exit(1)`，不会进入下一步。只有当所有步骤通过后，HTTP 端口才真正开始监听。

---

## 二、配置加载：优先级与覆盖规则

### 2.1 配置初始化的三层结构

代码位置：`internal/config/parser.go:25-37` 和 `internal/config/options.go:63-609`

```go
// 第 1 层：硬编码默认值
cfg := config.NewConfigParser()           // cli.go:80
   // → 内部调用 NewConfigOptions()，填充 70+ 项默认值
   //   例：DATABASE_URL="user=postgres password=postgres dbname=miniflux2 sslmode=disable"
   //       LISTEN_ADDR="127.0.0.1:8080"
   //       RUN_MIGRATIONS=false

// 第 2 层：配置文件（仅当 -c / --config-file 指定时）
if flagConfigFile != "" {
    config.Opts, err = cfg.ParseFile(flagConfigFile)   // cli.go:83
}

// 第 3 层：环境变量（无条件执行，总会覆盖）
config.Opts, err = cfg.ParseEnvironmentVariables()     // cli.go:89
```

**覆盖规则**：后加载者覆盖前者。最终优先级 = **环境变量 > 配置文件 > 默认值**。

### 2.2 覆盖机制实现细节

代码位置：`internal/config/parser.go:163-240` `parseLine()`

所有解析函数都使用 "新值非空则覆盖，否则保留旧值" 模式，例如：

```go
func parseStringValue(value string, fallback string) string {
    if value == "" {
        return fallback  // 空值不覆盖，保留上一层（默认/配置文件）的值
    }
    return value
}
```

这意味着：
- 在配置文件里写 `DATABASE_URL=`（空字符串）→ **不会** 覆盖默认值
- 在环境变量里 `export DATABASE_URL=` → 同样不会覆盖，行为一致
- 未知键被**静默忽略**（parser.go:164-171），不会因无关环境变量报错

### 2.3 `_FILE` 后缀的密钥注入机制

对于 `DATABASE_URL_FILE`、`ADMIN_PASSWORD_FILE` 等 `secretFileType` 选项，解析时不走普通赋值，而是：

```go
case secretFileType:
    secretValue, err := readSecretFileValue(value)   // 读文件，trim 空格
    if field.targetKey != "" {
        targetField.parsedStringValue = secretValue   // 写入目标键
        targetField.rawValue = secretValue
    }
```

即：设置 `DATABASE_URL_FILE=/run/secrets/db_url` 等价于直接设置 `DATABASE_URL` 为该文件内容。  
**同时设置两者时**：由于 `_FILE` 变量和目标变量是分别独立 parseLine 的，取决于环境变量列表里谁先出现（`os.Environ()` 顺序不确定），因此建议**只设一种**。

### 2.4 后处理阶段（postParsing）

代码位置：`internal/config/parser.go:96-141`

所有配置行解析完后统一执行：
1. 解析 `BASE_URL` → 拆分出 `rootURL`（协议+主机）和 `basePath`（路径部分）
2. 解析 `YOUTUBE_EMBED_URL_OVERRIDE` → 提取域名
3. 若未设置 `MEDIA_PROXY_PRIVATE_KEY` → **随机生成 16 字节**（注意：多实例部署会导致不一致，务必显式设置）
4. 若设置了 `PORT` → 覆盖 `LISTEN_ADDR` 为 `:$PORT`

---

## 三、数据库连接、迁移与 Schema 检查

### 3.1 连接池建立

代码位置：`internal/cli/cli.go:157-166`

```go
db, err := database.NewConnectionPool(
    config.Opts.DatabaseURL(),
    config.Opts.DatabaseMinConns(),     // 默认 1
    config.Opts.DatabaseMaxConns(),     // 默认 20
    config.Opts.DatabaseConnectionLifetime(),  // 默认 5 分钟
)
```

`sql.Open("postgres", dsn)` **并不真正建立 TCP 连接**，只是初始化驱动结构体。真正的连通性测试在下一行：

```go
store := storage.NewStorage(db)
if err := store.Ping(); err != nil { ... }   // cli.go:170-172
    // → 使用 5 秒超时 PingContext 探测
```

**这一步如果 PostgreSQL 不可达（未启动 / 网络不通 / 密码错），进程会直接退出，不会继续后面的迁移逻辑。**

### 3.2 迁移的两种触发方式

代码位置：`internal/cli/cli.go:174-179` 和 `216-224`

| 方式 | 触发条件 | 行为 | 后续 |
|------|---------|------|------|
| **显式迁移命令** | `-migrate` 标志 | 执行 `database.Migrate(db)` | 执行完立即 `return`，不启动 daemon |
| **自动迁移** | `RUN_MIGRATIONS=true` 环境变量 | 同上 `database.Migrate(db)` | 继续往下走，启动 daemon |

两种方式调用的是同一个函数。

### 3.3 Migrate() 内部逻辑

代码位置：`internal/database/migrations.go:13-51`

```go
func Migrate(db *sql.DB) error {
    var currentVersion int
    db.QueryRow(`SELECT version FROM schema_version`).Scan(&currentVersion)
    // 注意：这里没有检查 Scan 错误！
    // 首次安装（无 schema_version 表）时 Scan 返回 error，currentVersion=0
    // 刚好 migrations[0] 就是 CREATE TABLE schema_version + 所有初始表

    for version := currentVersion; version < schemaVersion; version++ {
        tx, _ := db.Begin()
        migrations[version](tx)    // 执行第 version+1 号迁移
        tx.Exec(`TRUNCATE schema_version`)
        tx.Exec(`INSERT INTO schema_version (version) VALUES ($1)`, version+1)
        tx.Commit()
    }
}
```

**关键特性**：
- 迁移 ID = `migrations` 数组下标，**从 0 开始**，`schemaVersion = len(migrations)`
- 每个版本在**独立事务**中执行；任何一步失败会 Rollback 并返回错误，**不会继续后续迁移**
- 首次安装（schema_version 不存在）：`Scan` 返回 `sql.ErrNoRows`，但代码忽略错误，`currentVersion=0`，恰好从第 0 个迁移（建表）开始，**这是设计上的隐式约定**

### 3.4 Schema 一致性守门检查

代码位置：`internal/cli/cli.go:222-224`

```go
if err := database.IsSchemaUpToDate(db); err != nil {
    printErrorAndExit(err)
}
```

**这个检查无论是否执行过迁移都会运行**，是启动 daemon 前的最后一道关卡：

```go
func IsSchemaUpToDate(db *sql.DB) error {
    var currentVersion int
    db.QueryRow(`SELECT version FROM schema_version`).Scan(&currentVersion)
    if currentVersion < schemaVersion {
        return fmt.Errorf("the database schema is not up to date: current=v%d expected=v%d", ...)
    }
    return nil
}
```

**可能导致启动失败的三种典型场景**：

| 场景 | RUN_MIGRATIONS=false | RUN_MIGRATIONS=true |
|------|---------------------|---------------------|
| **全新空库** | schema_version 不存在 → currentVersion=0 < N → 失败 ❌ | 迁移 0..N 全部执行 → 通过 ✅ |
| **旧版库已存在** | currentVersion < N → 失败 ❌（报错提示先执行迁移） | 执行缺失的迁移 → 通过 ✅ |
| **手动升级了二进制但没迁移** | 同上失败 ❌ | 同上自动升级 ✅ |

> 💡 **核心结论**：生产环境如果担心迁移破坏数据，用 `RUN_MIGRATIONS=false` + 先跑一次 `-migrate` 子命令做人工确认；容器化/单实例部署可以直接开 `RUN_MIGRATIONS=true` 简化运维。

---

## 四、Daemon 启动：从初始化到首请求可用

代码位置：`internal/cli/daemon.go:23-102` `startDaemon()`

### 4.1 启动顺序

```
startDaemon(store)
  │
  ├─ 1. worker.NewPool(size)                      // 内存对象，不涉及 IO
  │
  ├─ 2. if HasSchedulerService && !MaintenanceMode:
  │      runScheduler()
  │        ├─ go feedScheduler()    // 每 POLLING_FREQUENCY 触发一次（默认 60min）
  │        └─ go cleanupScheduler() // 每 CLEANUP_FREQUENCY_HOURS 触发一次（默认 24h）
  │
  ├─ 3. if HasHTTPService:
  │      server.StartWebServer()
  │        ├─ setupAutocert()                   // 如果配置了 CERT_DOMAIN
  │        │    └─ 启动 :80 ACME challenge server（goroutine）
  │        ├─ 对每个 LISTEN_ADDR:
  │        │    └─ go Serve() / ServeTLS()      // 每个地址独立 goroutine
  │        └─ 返回所有 *http.Server 句柄
  │
  ├─ 4. if HasMetricsCollector:
  │      go collector.GatherStorageMetrics()    // 每 METRICS_REFRESH_INTERVAL 更新一次
  │
  └─ 5. Systemd 通知
       ├─ SdNotify(READY=1)          ← 此时服务才算"启动完成"
       └─ if HasWatchdog:
            go 每 (interval/3) Ping DB 并发 NOTIFY_WATCHDOG=1
```

### 4.2 HTTP 服务器真正开始监听的时刻

`internal/http/server/server.go` 中所有 `startXXXServer()` 函数都是**异步 goroutine**：

```go
func startHTTPServer(server *http.Server) {
    go func() {
        server.ListenAndServe()  // ← 这里才真正 bind 端口，开始 accept
    }()
}
```

因此：
- `StartWebServer()` 返回 ≠ 端口已经 bind 完成（存在几十毫秒的 goroutine 调度窗口）
- 但返回后 `startDaemon` 立刻发 systemd `READY=1`，所以**如果 systemd 之后立即探活，理论上可能遇到瞬时 ECONNREFUSED**
- 健康检查端点 `/healthz` 由 `internal/http/server/healthcheck.go` 提供，它不依赖数据库，只返回 200 OK

### 4.3 多监听地址并行启动

`LISTEN_ADDR` 支持逗号分隔多值（类型 `stringListType`），默认 `["127.0.0.1:8080"]`。  
每个地址会**各自判断模式**（HTTP/TLS/UnixSocket/Systemd），互不影响。

TLS 模式的优先级链（`determineListenTargets()`）：
```
hasAutocert(CERT_DOMAIN) → modeAutocertTLS
  else hasCertFiles(CERT_FILE+KEY_FILE) → modeTLS
    else → modeHTTP
```

如果启用了任一 TLS 模式，`config.Opts.SetHTTPSValue(true)` 会被调用，影响后续 Cookie Secure flag、HSTS 等行为。

### 4.4 维护模式（MAINTENANCE_MODE）的影响

- `HasSchedulerService() && !HasMaintenanceMode()` → **调度器完全不启动**（不刷新 feed、不清理）
- HTTP 服务器仍会启动，访问 UI 时会显示维护消息但仍可静态资源可访问
- 管理员自动创建 `CREATE_ADMIN` 不受维护模式影响，仍在 daemon 启动前执行

---

## 五、管理员自动创建的时机

代码位置：`internal/cli/cli.go:226-228`

```go
if config.Opts.CreateAdmin() {
    createAdminUserFromEnvironmentVariables(store)
}
```

**执行时机**：
1. 数据库连接 ✅
2. Schema 迁移 + 版本检查 ✅（确保 `users` 表存在）
3. 但在调度器 / HTTP 服务器启动 **之前**

这意味着：
- 首次部署时 `CREATE_ADMIN=true` + `RUN_MIGRATIONS=true` 组合可以**一条龙**完成建库+建表+建管理员
- `createAdminUserFromEnvironmentVariables` 内部逻辑（`create_admin.go`）：若 `ADMIN_USERNAME` 对应用户已存在，则**静默跳过**，不会重置密码（这是幂等设计，但升级部署不会自动同步新密码）

---

## 六、启动失败排查顺序（基于代码路径）

按照代码执行顺序，出现问题时按下列顺序排查：

| 阶段 | 典型报错 | 代码位置 |
|------|---------|---------|
| 配置解析 | `invalid boolean value for key XXX` | `parser.go:191` |
| 配置校验 | `DATABASE_MIN_CONNS must be less than or equal to DATABASE_MAX_CONNS` | `parser.go:54-93` |
| 日志初始化 | `unable to open log file` | `cli.go:125-128` |
| 静态资源 | `unable to generate javascript bundle` | `cli.go:149-155` |
| DB 连接 | `unable to connect to database` | `cli.go:163-165`（实际上是 Ping 失败） |
| 迁移执行 | `[Migration vXX] ERROR: relation ... already exists` | `migrations.go:27-47` |
| Schema 检查 | `the database schema is not up to date: current=v42 expected=v57` | `migrations.go:58` |
| 管理员创建 | `ADMIN_USERNAME is not set` | `create_admin.go` |
| 端口监听 | `HTTP server failed to start on :8080: bind: address already in use` | `server.go:264-266` |

---

## 七、完整时序图（关键节点）

```
时间轴 ──────────────────────────────────────────────────────────────────►
        │
        ├─ [CLI] flag.Parse()
        │
        ├─ [CONFIG] NewConfigOptions()  ← 所有默认值写入内存
        ├─ [CONFIG] ParseFile()         ← 仅 -c 时执行（可能覆盖默认）
        ├─ [CONFIG] ParseEnvironmentVariables()  ← 总会执行（覆盖前两层）
        ├─ [CONFIG] postParsing()       ← BASE_URL 拆分、随机密钥生成
        ├─ [CONFIG] Validate()          ← 互斥检查、边界检查
        │
        ├─ [LOGGER] InitializeDefaultLogger()
        │
        ├─ [STATIC] GenerateBinaryBundles()
        ├─ [STATIC] GenerateStylesheetsBundles()
        ├─ [STATIC] GenerateJavascriptBundles()
        │
        ├─ [DB] sql.Open("postgres", dsn)   ← 不建连
        ├─ [DB] store.Ping(5s timeout)      ← 首次真实建连 & 鉴权
        │
        ├─ 若 -migrate 标志:
        │    └─ Migrate() → return (进程结束，不启动服务)
        │
        ├─ 若 RUN_MIGRATIONS=true:
        │    └─ Migrate()
        │
        ├─ [DB] IsSchemaUpToDate()  ← 不通过就 Exit(1)
        │
        ├─ [ADMIN] CREATE_ADMIN=true → createAdminUserFromEnv()
        │
        ├─ [PROXY] HTTP_CLIENT_PROXIES → ProxyRotatorInstance 初始化
        │
        ├─ [WORKER] NewPool(size)
        ├─ [SCHED] go feedScheduler()    ← 首个 tick 在 POLLING_FREQUENCY 之后
        ├─ [SCHED] go cleanupScheduler()
        │
        ├─ [HTTP] go ListenAndServe() / ServeTLS()  ← 端口 bind 完成
        ├─ [METRIC] go GatherStorageMetrics()
        │
        ├─ [SYSTEMD] NOTIFY_SOCKET → READY=1  ← 【对外宣告就绪】
        ├─ [SYSTEMD] go Watchdog loop (Ping DB)
        │
        └─ ─ ─ ─ 阻塞等待 SIGINT/SIGTERM ─ ─ ─ 服务正常运行中 ─ ─ ─ ─
```

---

## 八、核心设计要点总结

1. **配置三层覆盖 + 空值不覆盖**：环境变量总是最后加载，确保可在不改动配置文件的情况下覆盖任何选项；同时 Docker Secret / Kubernetes Secret 通过 `_FILE` 后缀无缝集成。

2. **数据库迁移与 Schema 检查是分离的两步**：Migrate() 是可选的（可以用 `-migrate` 人工跑），但 IsSchemaUpToDate() 是**强制闸门**，版本对不上就绝不开机，避免代码与表结构不一致导致的运行时灾难。

3. **首次安装的隐式约定**：`schema_version` 表不存在时 Scan 出错 `currentVersion=0` 恰好从 migrations[0]（建表迁移）开始，省去了额外的 bootstrap 分支判断。

4. **所有网络监听都在 goroutine 中启动**：`startDaemon()` 返回后立刻发 systemd READY，存在 goroutine 调度与 bind 之间的**极小时间窗口**（极端场景下可能需要健康检查兜底）。

5. **严格串行的失败模型**：任一步骤失败就 `os.Exit(1)`，不会出现"半初始化"状态，简化了故障排查。

6. **维护模式只停调度器，不停 HTTP 服务**：这样运维可以继续访问 UI 排查问题，但不会产生新的 feed 抓取压力。
