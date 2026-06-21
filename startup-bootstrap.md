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

### 2.2 覆盖机制实现细节：空字符串为何不覆盖

代码位置：`internal/config/parser.go:163-240` `parseLine()` + 各类型 `parseXxxValue()` 函数

#### 完整调用链（以 string 类型为例）

```
环境变量 "LOG_FILE=" (空值) 进入 parseLines()
    │
    └─ parseLine("LOG_FILE", "")            parser.go:163
         │
         ├─ 1. 查表: options["LOG_FILE"] 存在，valueType = stringType
         ├─ 2. validator (如有) 通过
         └─ 3. switch valueType → stringType 分支:
              │
              └─ field.parsedStringValue = parseStringValue("", field.parsedStringValue)
                        │                               ↑
                        │                        这是当前已有的值（来自默认值或配置文件）
                        │
                        └─ parseStringValue("" , "stderr")
                             │
                             └─ if "" == "" → return "stderr"  ← 原值不变！
```

#### 各类型对空值的判定逻辑

| 类型 | 解析函数 | 空值判定条件 | 行为 |
|------|---------|-------------|------|
| `stringType` | `parseStringValue` | `value == ""` | 返回 fallback，不覆盖 |
| `stringListType` | `parseStringListValue` | `value == ""` | 返回 fallback，不覆盖 |
| `boolType` | `parseBoolValue` | `value == ""` | 返回 fallback，不覆盖（注意 bool 设为空字符串不等价于 false） |
| `intType` | `parseIntValue` | `value == ""` **或** `strconv.Atoi` 失败 | 返回 fallback，不覆盖（"0" 是合法值，会覆盖为 0） |
| `int64Type` | `ParsedInt64Value` | 同上 | 同上 |
| `secondType/minuteType/hourType/dayType` | `parseDurationValue` | `value == ""` **或** `strconv.Atoi` 失败 | 返回 fallback，不覆盖 |
| `urlType` | `parseURLValue` | `value == ""` | 返回 fallback，不覆盖 |
| `secretFileType` | `readSecretFileValue` | 文件内容 `TrimSpace` 后 `== ""` | **报错**（不静默，区别于其他类型） |
| `bytesType` | 内联在 parseLine | `value == ""` | 跳过赋值，保持原值 |

**测试验证**（见 `parser_test.go` 和 `options_parsing_test.go`）：

```go
// parser_test.go:21-25
result := parseStringValue("", "fallback")
// result == "fallback" ✅

// options_parsing_test.go:368-381
os.Setenv("LOG_FILE", "")          // 环境变量设为空字符串
configParser.ParseEnvironmentVariables()
configOptions.LogFile() == "stderr"  // 仍为默认值 ✅
```

**特别注意 `intType` 的"伪覆盖"陷阱**：
- `parseIntValue("0", 42)` → 返回 **0**（覆盖成功，"0" 不是空）
- `parseIntValue("", 42)` → 返回 **42**（不覆盖）
- `parseIntValue("invalid", 42)` → 返回 **42**（解析失败也不覆盖，无报错）

这意味着如果想把 `BATCH_SIZE` 设为 0（虽然不合法，但在边界校验之前），必须显式写 `BATCH_SIZE=0`，而不是 `BATCH_SIZE=`。

#### 未知键的处理

`parseLine()` 开头（`parser.go:164-171`）：
```go
field, exists := cp.options.options[key]
if !exists {
    if key == "FILTER_ENTRY_MAX_AGE_DAYS" {
        slog.Warn("Configuration option FILTER_ENTRY_MAX_AGE_DAYS is deprecated...")
    }
    // 其他未知键：静默忽略，直接 return nil
    return nil
}
```

即：系统里无关的环境变量（如 `PATH`、`HOME`、`SHELL`）会被完全忽略，不会污染配置也不会报错。只有 `FILTER_ENTRY_MAX_AGE_DAYS` 这一个已废弃键会打警告日志。

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

### 3.3 Migrate() 内部逻辑与 Schema 版本对照

代码位置：`internal/database/migrations.go` + `internal/database/database.go:13-61`

#### 重要前置说明：Miniflux 的迁移组织方式

很多项目用 `migrations/001_xxx.sql`、`migrations/002_xxx.sql` 这样的独立 SQL 文件，通过文件名前缀的数字与 schema_version 对照。**Miniflux 不采用这种模式**——所有迁移 SQL 都内联在 Go 代码的 `migrations` 数组中：

```go
// migrations.go:13-16
var schemaVersion = len(migrations)  // 当前代码期望的目标版本 = 数组长度

var migrations = [...]func(tx *sql.Tx) error{
    func(tx *sql.Tx) (err error) {   // migrations[0]  ← version 0 → 执行后变为 v1
        sql := `CREATE TABLE schema_version (...); CREATE TABLE users (...); ...`
        _, err = tx.Exec(sql)
        return err
    },
    func(tx *sql.Tx) (err error) {   // migrations[1]  ← version 1 → 执行后变为 v2
        ...
    },
    // ... 依次追加，新增迁移必须放在数组末尾
}
```

因此：**不存在 "migration 文件名与版本号的匹配校验"**——因为根本没有文件。版本号就是数组下标，对照关系是硬编码的数组顺序。

#### 版本号对照关系详解

```
数据库 schema_version 表的值    migrations 数组下标      执行操作
────────────────────────────   ────────────────────   ─────────────
             0              →   不执行任何迁移         （不可能的状态，除非手动篡改）
             1              →   migrations[1..N]      （从第 2 个迁移开始跑）
            ...
             v              →   migrations[v .. N-1]  （执行 v 到 N-1 之间所有迁移）
             N              →   不执行（已是最新）
```

注意这个容易混淆的 off-by-one：
- `migrations[0]` 执行完成后，`schema_version` 被写入 **1**（不是 0）
- 数据库当前版本 `v` 表示 "已经执行完 `migrations[0]` 到 `migrations[v-1]`"
- 需要从 `migrations[v]` 开始继续执行

#### Migrate() 完整执行流程

```go
func Migrate(db *sql.DB) error {
    var currentVersion int
    // ── Step 1: 读当前版本 ──────────────────────────────────────
    db.QueryRow(`SELECT version FROM schema_version`).Scan(&currentVersion)
    // ↑ 刻意忽略 Scan 错误
    //   - 如果表存在 → currentVersion = 表中的整数值
    //   - 如果表不存在 → Scan 返回 sql.ErrNoRows，currentVersion 保持零值 0
    //   - 如果是纯空库（连 PostgreSQL 本身都没 schema）→ 也会出错 → currentVersion=0
    //
    //   刚好 migrations[0] 就是 "CREATE TABLE schema_version + 建所有初始表"，
    //   所以 currentVersion=0 时从 migrations[0] 开始跑，恰好完成首次初始化。
    //   这是整个迁移系统的核心隐式约定。

    slog.Info("Running database migrations",
        slog.Int("current_version", currentVersion),
        slog.Int("latest_version", schemaVersion),
    )

    // ── Step 2: 逐个执行缺失迁移 ────────────────────────────────
    for version := currentVersion; version < schemaVersion; version++ {
        newVersion := version + 1

        tx, err := db.Begin()   // 每个迁移独立事务
        if err != nil { return ... }

        if err := migrations[version](tx); err != nil {
            tx.Rollback()       // 出错回滚，不污染后续迁移
            return fmt.Errorf("[Migration v%d] %v", newVersion, err)
        }

        // ── Step 3: 更新 schema_version 表 ─────────────────────
        // 先清空（表里永远只有一行），再插入新版本号
        tx.Exec(`TRUNCATE schema_version`)
        tx.Exec(`INSERT INTO schema_version (version) VALUES ($1)`, newVersion)
        // ↑ 注意：schema_version 的 version 列定义是 text 类型，
        //   但插入和查询都用整数——PostgreSQL 会自动做类型转换。
        //   migrations[0] 建表时写的是 "version text not null"，
        //   这是早期设计遗留，但因两边都走 int↔text 隐式转换，工作正常。

        if err := tx.Commit(); err != nil {
            return fmt.Errorf("[Migration v%d] %v", newVersion, err)
        }
    }

    return nil
}
```

**关键特性总结**：
1. **迁移 ID = 数组下标**：从 0 开始，连续递增，没有分支、没有时间戳命名
2. **事务隔离**：每个版本一个独立事务；失败即 Rollback，不会留下半执行状态
3. **幂等执行**：如果版本已是最新，循环体一次都不会跑，直接返回
4. **首次安装的隐式约定**：`schema_version` 表不存在 → `Scan` 出错被忽略 → `currentVersion=0` → 从 `migrations[0]` 开始（建表），恰好闭环
5. **没有文件名校验/排序逻辑**：因为不使用外部 SQL 文件，`migrations` 数组的声明顺序就是唯一的执行顺序来源；如果有人在数组中间插入新迁移而非追加末尾，就会导致版本号错位，已升级到 v57 的数据库会重新执行被插入位置之后的迁移（从而出现 "relation already exists" 错误）。这就是代码注释 "Order is important. Add new migrations at the end of the list." 的原因。

#### schema_version 表的类型细节（易被忽略）

`migrations[0]` 建表 DDL：
```sql
CREATE TABLE schema_version (
    version text not null   -- 注意：是 TEXT，不是 INTEGER
);
```

但读写时都用整数：
```go
// 读：Scan 到 int 变量
var currentVersion int
db.QueryRow(`SELECT version FROM schema_version`).Scan(&currentVersion)

// 写：用 $1 占位符传 int
tx.Exec(`INSERT INTO schema_version (version) VALUES ($1)`, newVersion)
```

依赖 PostgreSQL 的隐式类型转换（int → text 写入，text → int 读出）。虽然能工作，但如果表里手动写入了非数字字符串（如 `abc`），`Scan` 会失败，`currentVersion=0`，导致从 `migrations[0]` 重新开始执行（会出现一堆 "relation already exists" 错误）。生产环境不要手动改 `schema_version` 表。

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
