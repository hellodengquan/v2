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

## 7. 翻译键缺失与语言文件加载失败的错误处理

### 7.1 总体策略：静默降级，不报错不崩溃

整个 locale 系统的错误处理哲学是**静默降级（graceful degradation）**：无论翻译键缺失、语言不存在还是文件加载失败，都不会抛出 panic 或返回 HTTP 500，而是始终返回一个合理的字符串值，保证页面可渲染。

### 7.2 翻译键缺失时的行为

#### `Print(key)` — 两种降级路径

代码位置：`internal/locale/printer.go:18-25`

```go
func (p *Printer) Print(key string) string {
    if dict, err := getTranslationDict(p.language); err == nil {
        if str, ok := dict.singulars[key]; ok {
            return str
        }
    }
    return key
}
```

降级路径分析：

| 场景 | 条件分支 | 返回值 | 说明 |
|------|----------|--------|------|
| 语言文件加载成功，键存在 | `err == nil && ok == true` | 翻译后的字符串 | 正常路径 |
| 语言文件加载成功，键不存在 | `err == nil && ok == false` | 原始 key | 如 `Print("missing.key")` → `"missing.key"` |
| 语言文件加载失败 | `err != nil` | 原始 key | 进入外层 `return key` |

**结论：键缺失时静默返回原始 key 字符串，不使用 `en_US` 作为备选，不报错。**

测试验证（`internal/locale/printer_test.go:96-109`）：

```go
func TestPrintWithMissingKey(t *testing.T) {
    defaultCatalog = catalog{
        "en_US": translationDict{
            singulars: map[string]string{
                "existing.key": "value",
            },
        },
    }
    translation := NewPrinter("en_US").Print("missing.key")
    // 期望值是 "missing.key"，不是 en_US 的任何翻译
    if translation != "missing.key" {
        t.Errorf(`Wrong translation, got %q`, translation)
    }
}
```

#### `Printf(key, args...)` — 继承 Print 的降级行为

代码位置：`internal/locale/printer.go:28-30`

```go
func (p *Printer) Printf(key string, args ...any) string {
    return formatTranslation(p.Print(key), args...)
}
```

`Printf` 内部调用 `Print(key)`，因此键缺失时 `Print` 返回原始 key，然后 `formatTranslation` 对该 key 做格式化处理：

| 场景 | 返回值 | 说明 |
|------|--------|------|
| key 缺失，key 不含 `%` 占位符 | 原始 key | `hasFormattingDirective` 返回 false，忽略 args |
| key 缺失，key 本身含 `%d` 等 | `fmt.Sprintf(key, args...)` | key 充当格式化模板（意外行为但不会崩溃） |

测试验证（`internal/locale/printer_test.go:67-85`）：

```go
func TestPrintfWithMissingKeyAndPlaceholder(t *testing.T) {
    // key "Status: %s" 不在 fr_FR 的翻译字典中
    translation := NewPrinter("fr_FR").Printf("Status: %s", "ok")
    // key 本身含 %s，因此 args 被用于格式化，返回 "Status: ok"
    if translation != "Status: ok" {
        t.Errorf(`Wrong translation, got %q`, translation)
    }
}
```

#### `Plural(key, n, args...)` — 三种降级路径

代码位置：`internal/locale/printer.go:33-47`

```go
func (p *Printer) Plural(key string, n int, args ...any) string {
    dict, err := getTranslationDict(p.language)
    if err != nil {
        return key
    }

    if choices, found := dict.plurals[key]; found {
        index := getPluralForm(p.language, n)
        if len(choices) > index {
            return formatTranslation(choices[index], args...)
        }
    }

    return key
}
```

降级路径分析：

| 场景 | 条件分支 | 返回值 | 说明 |
|------|----------|--------|------|
| 语言文件加载失败 | `err != nil` | 原始 key | 第一层降级 |
| 键不在 plurals 中 | `found == false` | 原始 key | 第二层降级 |
| 复数形式数组长度不够 | `len(choices) <= index` | 原始 key | 第三层降级（越界保护） |
| 正常 | `len(choices) > index` | `formatTranslation(choices[index], args...)` | 正常路径 |

测试验证（`internal/locale/printer_test.go:280-304`）：

```go
func TestPluralWithIndexOutOfBounds(t *testing.T) {
    // 捷克语有 3 种复数形式，但翻译数组只有 1 个元素
    defaultCatalog["cs_CZ"] = translationDict{
        plurals: map[string][]string{
            "limited.key": {"only one form"},
        },
    }
    printer := NewPrinter("cs_CZ")
    // n=5 时 getPluralForm("cs_CZ", 5) 返回 index 2，但 choices 只有 1 个元素
    translation := printer.Plural("limited.key", 5)
    // 越界保护生效，返回原始 key
    if translation != "limited.key" {
        t.Errorf(`Wrong translation, got %q`, translation)
    }
}
```

### 7.3 语言文件加载失败时的行为

#### 加载流程与错误传播

代码位置：`internal/locale/catalog.go:23-31`

```go
func getTranslationDict(language string) (translationDict, error) {
    if _, ok := defaultCatalog[language]; !ok {
        var err error
        if defaultCatalog[language], err = loadTranslationFile(language); err != nil {
            return translationDict{}, err
        }
    }
    return defaultCatalog[language], nil
}
```

可能失败的环节：

| 环节 | 失败原因 | 代码位置 |
|------|----------|----------|
| `translationFiles.ReadFile()` | 语言代码对应的 JSON 文件不存在（`embed.FS` 中无此文件） | `catalog.go:34` |
| `parseTranslationMessages()` | JSON 语法错误或值类型非法（非 string/[]any） | `catalog.go:39` |

#### 加载失败不会导致 HTTP 500

关键观察：`getTranslationDict` 返回 `(translationDict{}, err)`，但调用方 `Print`、`Printf`、`Plural` 全部**吞掉错误**：

```go
// Print: err != nil 时走 return key
if dict, err := getTranslationDict(p.language); err == nil { ... }
return key

// Plural: err != nil 时直接 return key
dict, err := getTranslationDict(p.language)
if err != nil {
    return key
}
```

**因此，即使语言文件加载失败：**
1. `Print`/`Printf`/`Plural` 静默返回原始 key
2. 模板渲染不会中断
3. HTTP 请求正常返回 200，页面上显示的是翻译 key（如 `"page.login.title"`）而非翻译后的文本
4. **不会产生 HTTP 500 状态码**

测试验证（`internal/locale/printer_test.go:8-15`）：

```go
func TestPrintfWithMissingLanguage(t *testing.T) {
    defaultCatalog = catalog{}
    translation := NewPrinter("invalid").Printf("missing.key")
    // "invalid" 语言不在 catalog 中，embed.FS 中也没有 invalid.json
    // loadTranslationFile 会失败，但 Printf 静默降级
    if translation != "missing.key" {
        t.Errorf(`Wrong translation, got %q`, translation)
    }
}
```

### 7.4 不存在 en_US 备选降级

**整个 locale 系统没有 `en_US` 备选降级机制。** 当某种语言的翻译键缺失时，系统不会回退到 `en_US` 对应的翻译。这一点在测试中有明确验证：

```go
// fr_FR 的 plurals 中没有 "number_of_users" 键
// 但也不会去 en_US 查找，直接返回原始 key
func TestPluralWithMissingTranslation(t *testing.T) {
    defaultCatalog = catalog{
        "en_US": translationDict{
            plurals: map[string][]string{
                "number_of_users": {"%d user (%s)", "%d users (%s)"},
            },
        },
        "fr_FR": translationDict{},  // 空字典，没有任何键
    }
    translation := NewPrinter("fr_FR").Plural("number_of_users", 2)
    expected := "number_of_users"  // 返回 key，不是 en_US 的 "%d users"
}
```

同样，`LocalizedError.Translate` 在语言无效时也不回退到 `en_US`：

```go
// internal/locale/error.go:26-33
func (l *LocalizedErrorWrapper) Translate(language string) string {
    if l.translationKey == "" {
        if l.originalErr == nil {
            return ""
        }
        return l.originalErr.Error()
    }
    return NewPrinter(language).Printf(l.translationKey, l.translationArgs...)
}
```

当 `language` 无效时，`Printf` → `Print` → key 缺失 → 返回原始 key。

### 7.5 唯一会报错的场景：Render 中 language 字段缺失

代码位置：`internal/template/engine.go:97`

```go
printer := locale.NewPrinter(data["language"].(string))
```

如果 `data["language"]` 为 `nil`（即调用方未传入 `language` 参数），此处的类型断言 `.(string)` 会触发 **panic**。这是整个流程中唯一可能因语言相关原因导致崩溃的地方——但它本质上是调用方的编程错误（忘记在 view 参数中设置 `language`），而非翻译系统本身的容错问题。

正常流程中 `view.New()` 始终会从 `WebSession` 取出 `language`（默认 `en_US`），所以此 panic 在生产环境不会触发。

### 7.6 错误处理策略总结

```
翻译键缺失 / 语言文件加载失败
    │
    ├── Print ───────► 返回原始 key 字符串
    ├── Printf ──────► 返回原始 key（不含 % 则忽略 args；含 % 则将 key 当模板格式化）
    └── Plural ──────► 返回原始 key 字符串

复数形式数组越界
    │
    └── Plural ──────► 返回原始 key 字符串（len(choices) <= index 保护）

language 参数缺失（data["language"] == nil）
    │
    └── Engine.Render ► panic（类型断言失败，编程错误，生产环境不会触发）

是否有 en_US 备选降级？
    │
    └── 否。所有降级路径均直接返回原始 key，不回退到其他语言。

加载失败是否导致 HTTP 500？
    │
    └── 否。所有错误被静默吞掉，页面正常渲染，翻译 key 作为文本显示。
```

---

## 8. 关键代码位置索引

| 模块 | 文件 | 关键行 |
|------|------|--------|
| 模板引擎 | `internal/template/engine.go` | 35-88 (解析), 90-114 (渲染) |
| 模板函数 | `internal/template/functions.go` | 35-161 (函数注册), 284-323 (elapsedTime) |
| 语言列表 | `internal/locale/locale.go` | 7-31 |
| 翻译目录 | `internal/locale/catalog.go` | 23-31 (懒加载), 47-79 (JSON解析) |
| 翻译器 | `internal/locale/printer.go` | 18-47 (Print/Printf/Plural), 52-77 (格式化安全) |
| 翻译器测试 | `internal/locale/printer_test.go` | 8-15 (缺失语言), 96-109 (缺失键), 280-304 (复数越界) |
| 复数规则 | `internal/locale/plural.go` | 8-76 |
| 本地化错误 | `internal/locale/error.go` | 8-34 (LocalizedErrorWrapper), 36-55 (LocalizedError) |
| View 封装 | `internal/ui/view/view.go` | 34-49 (默认参数注入) |
| WebSession 语言 | `internal/model/web_session.go` | 139-144 (Language getter), 205-208 (setter) |
| HTTP 响应 | `internal/http/response/html.go` | 17-30 (HTML) |
| 布局模板 | `internal/template/templates/common/layout.html` | 1-196 |
| 登录页视图 | `internal/template/templates/views/login.html` | 1-60 |
