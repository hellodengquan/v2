# 模板渲染与本地化协作流程分析

## 1. 整体架构概览

Miniflux 的模板渲染与本地化系统由以下核心模块协作完成：

```
HTTP Request
    │
    ▼
Request Context (WebSession) ──► 提取 language
    │
    ▼
UI Handler (e.g. showLoginPage)
    │
    ├──► view.New() ──► 构建默认模板参数（含 language）
    │
    ├──► view.Set()  ──► 添加业务数据
    │
    ▼
template.Engine.Render()
    │
    ├──► 根据 language 创建 locale.Printer
    │
    ├──► 运行时注入 t / plural / elapsed 函数
    │
    ▼
Go html/template 执行
    │
    ├──► 布局模板 (layout.html) 定义 base 骨架
    ├──► 公共模板片段 (feed_menu, pagination 等)
    └──► 视图模板 (login.html, feeds.html 等) 填充 title/page_header/content
    │
    ▼
HTML 字节流 ──► response.HTML() ──► HTTP Response
```

---

## 2. 模板组合机制

### 2.1 模板文件组织

模板文件通过 Go 1.16+ 的 `embed.FS` 内嵌到二进制中，分为两类：

| 目录 | 作用 | 示例 |
|------|------|------|
| `templates/common/` | 公共可复用片段 | `layout.html`, `pagination.html`, `feed_menu.html` |
| `templates/views/` | 页面级视图 | `login.html`, `feeds.html`, `entry.html` |

代码位置：`internal/template/engine.go:15-19`

```go
//go:embed templates/common/*.html
var commonTemplateFiles embed.FS

//go:embed templates/views/*.html
var viewTemplateFiles embed.FS
```

### 2.2 依赖声明与解析顺序

`ParseTemplates()` 显式声明每个视图模板依赖哪些公共模板：

代码位置：`internal/template/engine.go:37-68`

```go
templates := map[string][]string{
    "login.html":               {"layout.html"},
    "feeds.html":               {"feed_list.html", "feed_menu.html", "item_meta.html", "layout.html", "pagination.html"},
    "add_subscription.html":    {"feed_menu.html", "layout.html", "settings_menu.html"},
    // ...
}
```

**解析顺序至关重要**：先加载依赖，再加载视图本身，这样视图中定义的 `title`、`content` 等 block 才能正确覆盖/填充布局中的占位符。

代码位置：`internal/template/engine.go:70-76`

```go
for name, dependencies := range templates {
    tpl := template.New("").Funcs(funcMap)
    for _, dependency := range dependencies {
        template.Must(tpl.ParseFS(commonTemplateFiles, "templates/common/"+dependency))
    }
    e.templates[name] = template.Must(tpl.ParseFS(viewTemplateFiles, "templates/views/"+name))
}
```

解析完成后还会做完整性校验，确保 `templates/views/` 下的所有文件都已声明。

### 2.3 模板组合原理：Block 命名约定

采用 Go `html/template` 的 `{{ define "name" }}` + `{{ template "name" . }}` 机制。

**布局模板 (`layout.html`)** 定义主骨架 `base`，内部预留三个插槽：

代码位置：`internal/template/templates/common/layout.html:1-196`

```html
{{ define "base" }}
<!DOCTYPE html>
<html lang="{{ replace .language "_" "-"}}">
<head>
    <title>{{template "title" .}} - Miniflux</title>
    ...
</head>
<body>
    {{template "page_header" .}}
    <main id="main">
        {{template "content" .}}
    </main>
</body>
</html>
{{ end }}
```

**视图模板 (`login.html`)** 负责填充这三个插槽：

代码位置：`internal/template/templates/views/login.html:1-60`

```html
{{ define "title"}}{{ t "page.login.title" }}{{ end }}
{{ define "page_header"}}{{ end }}
{{ define "content"}}
<section class="login-form">...</section>
{{ end }}
```

最终渲染时调用 `ExecuteTemplate(&b, "base", data)`，从 `base` 入口递归展开所有嵌套模板。

---

## 3. 语言选择流程

### 3.1 语言来源：WebSession

语言信息存储在用户会话 `WebSession` 中，从 HTTP 请求上下文提取。

代码位置：`internal/model/web_session.go:139-144`

```go
func (s *WebSession) Language() string {
    if s.state.Language != "" {
        return s.state.Language
    }
    return defaultSessionLanguage // "en_US"
}
```

默认值为 `en_US`，定义于 `internal/model/web_session.go:20`。

### 3.2 View 层注入 language 参数

`view.New()` 构建模板参数时，从 session 中取出 language 并放入数据 map。

代码位置：`internal/ui/view/view.go:34-49`

```go
func New(tpl *template.Engine, r *http.Request) *view {
    webSession := request.WebSession(r)
    return &view{tpl, r, map[string]any{
        "language": webSession.Language(),
        "theme":    webSession.Theme(),
        "csrf":     webSession.CSRF(),
        // ... 其他默认参数
    }}
}
```

### 3.3 可用语言列表

代码位置：`internal/locale/locale.go:7-31`

```go
var AvailableLanguages = map[string]string{
    "en_US": "English",
    "zh_CN": "简体中文",
    "zh_TW": "繁體中文",
    "fr_FR": "Français",
    // ... 共 23 种语言
}
```

---

## 4. 本地化变量替换机制

### 4.1 翻译资源加载

翻译文件以 JSON 格式存储在 `internal/locale/translations/` 下，同样通过 `embed.FS` 内嵌。

代码位置：`internal/locale/catalog.go:20-21`

```go
//go:embed translations/*.json
var translationFiles embed.FS
```

每个翻译文件包含两类条目：
- **单数形式**：`"key": "translated string"`
- **复数形式**：`"key": ["singular", "plural", ...]`

解析时通过自定义 `UnmarshalJSON` 分离存储：

代码位置：`internal/locale/catalog.go:47-79`

```go
func (t *translationDict) UnmarshalJSON(data []byte) error {
    for key, value := range tmpMap {
        switch vtype := value.(type) {
        case string:
            m.singulars[key] = vtype
        case []any:
            for _, translation := range vtype {
                m.plurals[key] = append(m.plurals[key], translation.(string))
            }
        }
    }
    return nil
}
```

翻译字典采用懒加载 + 全局缓存：首次访问某语言时加载并放入 `defaultCatalog`。

### 4.2 Printer：语言感知的翻译器

`locale.Printer` 是核心翻译组件，绑定到特定语言。

代码位置：`internal/locale/printer.go:8-16`

```go
type Printer struct {
    language string
}

func NewPrinter(language string) *Printer {
    return &Printer{language}
}
```

提供三个主要方法：

| 方法 | 用途 | 示例 |
|------|------|------|
| `Print(key)` | 纯字符串翻译 | `Print("action.login")` → `"Login"` |
| `Printf(key, args...)` | 带格式化变量 | `Printf("alert.too_many_feeds_refresh", 5)` |
| `Plural(key, n, args...)` | 复数形式翻译 | `Plural("page.unread_entry_count", 3, 3)` |

**`Print` 查找逻辑**（`internal/locale/printer.go:18-25`）：
1. 获取该语言的 `translationDict`
2. 在 `singulars` map 中查找 key
3. 找不到则原样返回 key 作为 fallback

### 4.3 复数形式：按语言规则选择索引

不同语言有不同的复数规则。`getPluralForm` 根据语言和数量返回数组索引。

代码位置：`internal/locale/plural.go:8-76`

```go
func getPluralForm(lang string, n int) int {
    switch lang {
    case "ar_SA": // 阿拉伯语有 6 种复数形式
        switch {
        case n == 0: return 0
        case n == 1: return 1
        case n == 2: return 2
        case n%100 >= 3 && n%100 <= 10: return 3
        case n%100 >= 11: return 4
        default: return 5
        }
    case "zh_CN", "zh_TW": // 中文无复数变化
        return 0
    default: // 英语等：0=单数, 1=复数
        if n > 1 { return 1 }
        return 0
    }
}
```

### 4.4 格式化变量安全处理

`formatTranslation` 做了一层保护：当翻译字符串不含格式化指令时，即使传入了多余参数也不会产生 `%!(EXTRA ...)` 错误。

代码位置：`internal/locale/printer.go:52-77`

```go
func formatTranslation(format string, args ...any) string {
    if !hasFormattingDirective(format) {
        return fmt.Sprintf(format, []any{}...)
    }
    return fmt.Sprintf(format, args...)
}
```

`hasFormattingDirective` 逐字符扫描，正确处理 `%%` 转义。

---

## 5. 模板函数与本地化的运行时绑定

这是最精巧的设计：**本地化相关函数采用"编译时占位 + 运行时覆盖"策略**。

### 5.1 编译时：注册占位函数

`funcMap.Map()` 在模板解析阶段注册三个空实现，仅为满足语法检查：

代码位置：`internal/template/functions.go:150-159`

```go
"elapsed": func(timezone string, t time.Time) string {
    return ""
},
"t": func(key any, args ...any) string {
    return ""
},
"plural": func(key string, n int, args ...any) string {
    return ""
},
```

### 5.2 运行时：按请求语言绑定真实实现

每次 `Render()` 调用时，根据当前请求的 `language` 创建 `Printer`，再用 `Funcs()` 覆盖上述三个函数。

代码位置：`internal/template/engine.go:90-107`

```go
func (e *Engine) Render(name string, data map[string]any) []byte {
    printer := locale.NewPrinter(data["language"].(string))

    tpl.Funcs(template.FuncMap{
        "elapsed": func(timezone string, t time.Time) string {
            return elapsedTime(printer, timezone, t)
        },
        "t":      printer.Printf,
        "plural": printer.Plural,
    })
    // ... 执行模板
}
```

**关键设计原因**：`Printer` 是请求级别的（每个用户语言不同），而模板是全局编译一次的，因此必须在每次渲染时动态注入绑定了当前语言的函数实现。

### 5.3 `elapsed`：本地化友好的相对时间

`elapsedTime` 是典型的本地化组合应用——根据时间差选择不同的翻译 key，且大量使用 `Plural`。

代码位置：`internal/template/functions.go:284-323`

```go
func elapsedTime(printer *locale.Printer, tz string, t time.Time) string {
    diff := now.Sub(t)
    s := diff.Seconds()
    switch {
    case s < 60:
        return printer.Print("time_elapsed.now")
    case s < 3600:
        minutes := int(diff.Minutes())
        return printer.Plural("time_elapsed.minutes", minutes, minutes)
    // ... 小时、天、周、月、年
    }
}
```

---

## 6. 完整调用链路示例（登录页）

以 `showLoginPage` 为例，追踪从请求到响应的完整路径：

### Step 1: Handler 接收请求

代码位置：`internal/ui/login_show.go:14-29`

```go
func (h *handler) showLoginPage(w http.ResponseWriter, r *http.Request) {
    view := view.New(h.tpl, r)
    view.Set("redirectURL", redirectURL)
    response.HTML(w, r, view.Render("login"))
}
```

### Step 2: View 构建参数

`view.New()` 从 session 取出 `language`（如 `"zh_CN"`）放入 `map[string]any`。

### Step 3: Engine 渲染

`view.Render("login")` → `h.tpl.Render("login.html", data)`：

1. 从 `data["language"]` 取出 `"zh_CN"`
2. 创建 `locale.Printer{language: "zh_CN"}`
3. 用该 Printer 的方法覆盖模板中的 `t`、`plural`、`elapsed`
4. 执行 `tpl.ExecuteTemplate(&b, "base", data)`

### Step 4: 模板展开

- `layout.html` 中的 `{{ replace .language "_" "-"}}` → `"zh-CN"`，写入 `<html lang="zh-CN">`
- `login.html` 的 `{{ define "title"}}{{ t "page.login.title" }}{{ end }}` 被解析：
  - 调用运行时注入的 `t` 函数（即 `printer.Printf`）
  - `printer.Printf("page.login.title")` 查找 `zh_CN.json` 中该 key
  - 返回 `"登录"`（假设翻译如此）
- 最终 `title` block 渲染结果被嵌入 `layout.html` 的 `<title>` 位置

### Step 5: 发送响应

`response.HTML()` 将渲染好的 `[]byte` 写入 HTTP 响应，设置 `Content-Type: text/html; charset=utf-8` 并应用压缩（gzip/brotli/deflate）。

---

## 7. 关键代码位置索引

| 模块 | 文件 | 关键行 |
|------|------|--------|
| 模板引擎 | `internal/template/engine.go` | 35-88 (解析), 90-114 (渲染) |
| 模板函数 | `internal/template/functions.go` | 35-161 (函数注册), 284-323 (elapsedTime) |
| 语言列表 | `internal/locale/locale.go` | 7-31 |
| 翻译目录 | `internal/locale/catalog.go` | 23-31 (懒加载), 47-79 (JSON解析) |
| 翻译器 | `internal/locale/printer.go` | 18-47 (Print/Printf/Plural), 52-77 (格式化安全) |
| 复数规则 | `internal/locale/plural.go` | 8-76 |
| View 封装 | `internal/ui/view/view.go` | 34-49 (默认参数注入) |
| WebSession 语言 | `internal/model/web_session.go` | 139-144 (Language getter), 205-208 (setter) |
| HTTP 响应 | `internal/http/response/html.go` | 17-30 (HTML) |
| 布局模板 | `internal/template/templates/common/layout.html` | 1-196 |
| 登录页视图 | `internal/template/templates/views/login.html` | 1-60 |
