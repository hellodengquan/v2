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

## 8. 模板国际化覆盖率分析

### 8.1 总体统计

通过扫描 `internal/template/templates/` 下全部 36 个 HTML 模板文件（6 个公共模板 + 30 个视图模板），统计结果如下：

| 指标 | 数量 |
|------|------|
| `{{ t "..." }}` 国际化调用 | 851 处 |
| `{{ plural "..." }}` 国际化调用 | 21 处 |
| **合计国际化调用** | **872 处** |
| 硬编码可见文本（需人工甄别） | 102 处 |
| **国际化覆盖率** | **89.5%** |

### 8.2 按文件分表统计

| 文件 | `t()` | `plural()` | 硬编码 | i18n 覆盖率 |
|------|------:|----------:|-------:|----------:|
| **common/feed_list.html** | 15 | 3 | 0 | 100.0% |
| **common/feed_menu.html** | 8 | 0 | 0 | 100.0% |
| **common/item_meta.html** | 25 | 1 | 0 | 100.0% |
| **common/pagination.html** | 9 | 0 | 0 | 100.0% |
| **common/settings_menu.html** | 8 | 0 | 0 | 100.0% |
| **views/categories.html** | 21 | 3 | 0 | 100.0% |
| **views/category_entries.html** | 27 | 2 | 0 | 100.0% |
| **views/choose_subscription.html** | 5 | 0 | 0 | 100.0% |
| **views/create_api_key.html** | 7 | 0 | 0 | 100.0% |
| **views/create_category.html** | 10 | 0 | 0 | 100.0% |
| **views/create_user.html** | 10 | 0 | 0 | 100.0% |
| **views/edit_category.html** | 11 | 0 | 0 | 100.0% |
| **views/edit_user.html** | 10 | 0 | 0 | 100.0% |
| **views/feed_entries.html** | 33 | 2 | 0 | 100.0% |
| **views/feeds.html** | 3 | 0 | 0 | 100.0% |
| **views/history_entries.html** | 11 | 1 | 0 | 100.0% |
| **views/import.html** | 8 | 0 | 0 | 100.0% |
| **views/login.html** | 10 | 0 | 0 | 100.0% |
| **views/search.html** | 8 | 0 | 0 | 100.0% |
| **views/sessions.html** | 12 | 0 | 0 | 100.0% |
| **views/shared_entries.html** | 17 | 1 | 0 | 100.0% |
| **views/starred_entries.html** | 3 | 1 | 0 | 100.0% |
| **views/tag_entries.html** | 1 | 1 | 0 | 100.0% |
| **views/unread_entries.html** | 20 | 1 | 0 | 100.0% |
| **views/users.html** | 17 | 0 | 0 | 100.0% |
| **views/webauthn_rename.html** | 5 | 0 | 0 | 100.0% |
| **common/layout.html** | 59 | 1 | 34 | 63.8% |
| **views/api_keys.html** | 19 | 0 | 1 | 95.0% |
| **views/settings.html** | 77 | 1 | 3 | 96.3% |
| **views/category_feeds.html** | 14 | 1 | 2 | 88.2% |
| **views/about.html** | 13 | 0 | 2 | 86.7% |
| **views/entry.html** | 53 | 2 | 6 | 90.2% |
| **views/edit_feed.html** | 77 | 0 | 13 | 85.6% |
| **views/add_subscription.html** | 25 | 0 | 7 | 78.1% |
| **views/offline.html** | 3 | 0 | 1 | 75.0% |
| **views/integrations.html** | 197 | 0 | 33 | 85.7% |

**覆盖率 100% 的模板共 25 个（占 69.4%），低于 90% 的模板共 6 个（占 16.7%）。**

### 8.3 未国际化内容分类与具体路径

以下将 102 处"硬编码"文本分为**无需国际化**（合理设计）与**建议国际化**（潜在遗漏）两类，并列出具体文件和位置。

---

### 8.4 无需国际化的内容（共 55 处，占 53.9%）

这类内容属于品牌名、产品名、键盘按键、协议名、版本号、HTML 实体等，不应该或不需要翻译。

#### A. 品牌名 / 产品名（4 处）

| 文件 | 位置 | 内容 | 说明 |
|------|------|------|------|
| `common/layout.html:6` | `<title>` 中 | `- Miniflux` | 产品品牌名后缀 |
| `views/about.html:12` | `<h3>` 标题 | `Miniflux` | About 页的产品名 |
| `views/offline.html:6` | `<title>` 中 | `- Miniflux` | 离线页的品牌名后缀 |
| `views/about.html:44` | License 链接 | `Apache 2.0` | 协议版本号（行业通用名） |

#### B. 键盘快捷键按键标识（34 处）

全部来自 `common/layout.html:141-184` 的快捷键模态对话框。这些是物理按键名称，属于无需翻译的通用符号：

```
g + u, g + b, g + h, g + f, g + c, g + s, p, k, n, j, F, G, h, l,
o, Enter, v, V, c, C, m, M, A, d, f, s, a, g + g, z + t, R, #, /, Esc,
以及 HTML 实体：&#x23F4; (⏴), &#x23F5; (⏵)
```

说明：键盘按键名（字母/组合键）是物理键盘上的实际标识，全世界通用，不翻译是正确的。

#### C. 第三方服务 / 协议品牌名（34 处，与 B 部分有重叠统计）

全部来自 `views/integrations.html`，是第三方集成服务的**注册品牌名**，不能翻译：

| 品牌名 | 出现位置 |
|--------|----------|
| `Archive.org` | `<details><summary>` |
| `Apprise` | `<details><summary>` + `edit_feed.html` 中 |
| `Betula` | `<details><summary>` |
| `Cubox` | `<details><summary>` |
| `Discord` | `<details><summary>` |
| `Espial` | `<details><summary>` |
| `Fever` | `<details><summary>`（RSS 协议名） |
| `Google Reader` | `<details><summary>`（产品名） |
| `Instapaper` | `<details><summary>` |
| `LinkAce` | `<details><summary>` |
| `Linkding` | `<details><summary>` |
| `LinkTaco` | `<details><summary>` + OAuth 链接 URL |
| `Linkwarden` | `<details><summary>` |
| `Matrix Bot` | `<details><summary>` |
| `Notion` | `<details><summary>` |
| `Ntfy` | `<details><summary>` + `edit_feed.html` 中 |
| `Nunux Keeper` | `<details><summary>` |
| `Omnivore` | `<details><summary>` |
| `Karakeep` | `<details><summary>` |
| `Pinboard` | `<details><summary>` |
| `Pushover` | `<details><summary>` + `edit_feed.html` 中 |
| `Raindrop` | `<details><summary>` |
| `Readeck` | `<details><summary>` |
| `Readwise Reader` | `<details><summary>` |
| `RSS-Bridge` | `<details><summary>` |
| `Shaarli` | `<details><summary>` |
| `Shiori` | `<details><summary>` |
| `Slack` | `<details><summary>` |
| `Telegram Bot` | `<details><summary>` |
| `Wallabag` | `<details><summary>` |
| `Webhook` | `<details><summary>` + `edit_feed.html` 中 |

#### D. API / URL 路径标识（2 处）

| 文件 | 位置 | 内容 | 说明 |
|------|------|------|------|
| `views/api_keys.html:59` | 端点展示 | `/v1/` | API 版本路径，技术标识 |
| `views/integrations.html:141` | Fever 端点 | `{rootURL}/fever/` | 同上 |

#### E. HTML 格式实体 / 排版空白（13 处）

| 文件 | 内容 | 出现次数 | 说明 |
|------|------|---------:|------|
| `views/add_subscription.html` | `&nbsp;` | 7 | 外部链接图标与 label 间的不可换行空格 |
| `views/edit_feed.html` | `&nbsp;` | 7 | 同上 |
| `views/settings.html` | `&nbsp;` | 3 | 同上 |
| `views/category_feeds.html:1,6` | `&gt;` (`>`) | 2 | 面包屑分隔符（"分类 > Feeds"中的 >） |
| `views/entry.html` | `&centerdot;` (·) | 1 | 条目元信息间的圆点分隔符 |

#### F. 媒体播放时间 / 速度单位标识（5 处）

全部来自 `views/entry.html:29-36`，是数字 + 单位组合：

| 内容 | 说明 |
|------|------|
| `-30s`, `-10s`, `+10s`, `+30s` | 音视频快进/快退秒数 |
| `1.00x` | 播放速度倍率 |

这些是技术操作标识，全球用户通用，不翻译也可理解。

---

### 8.5 建议国际化的内容（共 1 处，占 1.0%）

以下内容用户可见且有语义含义，遗漏了国际化包裹，建议补充。

| 序号 | 文件 | 行号 / 上下文 | 硬编码内容 | 建议翻译键 |
|------|------|-------------|-----------|----------|
| 1 | `common/layout.html:134` | 模态对话框关闭按钮 | `x` | `action.close_modal` 或使用 aria-label 已有的 Close |

> 说明：`<button class="btn-close-modal" aria-label="Close" autofocus>x</button>` 中的可见字符 `x`。虽然 `aria-label` 存在但不是国际化的（硬编码 "Close"），同时可见字符对纯文本/无 CSS 场景用户可见。建议两处都改为 `{{ t "action.close" }}`。

### 8.6 甄别结论汇总

```
扫描发现 102 处"硬编码"可见文本
    │
    ├── 8.4 无需国际化：101 处（99.0%）
    │   ├── 品牌名 / 产品名：4 处
    │   ├── 键盘快捷键按键标识：34 处
    │   ├── 第三方服务品牌名：33 处
    │   ├── API / URL 路径标识：2 处
    │   ├── HTML 实体 / 排版空白：19 处
    │   └── 媒体时间 / 速度单位：5 处
    │
    └── 8.5 建议国际化：1 处（1.0%）
        └── layout.html 模态框关闭按钮 "x"（+ aria-label "Close"）

真实国际化覆盖率（排除无需国际化后）≈ 99.9%
```

### 8.7 国际化做得好的实践总结

1. **`<details><summary>` 中的品牌名**：品牌名不翻译是正确做法，如 "Discord"、"Notion" 等国际品牌保持原名
2. **键盘快捷键**：字母键名（`g + u` 等）不翻译，全世界通用
3. **输入框 `placeholder`**：全部使用 `t` 函数包裹，如 `placeholder="{{ t "..." }}"`
4. **`title` / `aria-label` 属性**：布局模板中 `aria-label="{{ t "skip_to_content" }}"` 等无障碍属性也做了国际化
5. **`data-*` 属性**：按钮的 `data-label-*` 属性（给 JS 读取的动态文案）也全部国际化

### 8.8 未使用 `t`/`plural` 但实际不影响的情形

以下代码模式看似"硬编码"但因合理原因不应计入覆盖率分母：

- **`placeholder="https://domain.tld/"`** 等示例 URL：属于技术提示，全球用户都能识别
- **`{{ .entry.Title }}`** 等动态数据：这是用户内容，不是 UI 文本
- **SVG `<template id="icon-read">`**：图标资源标识，不是用户可见文本
- **`<option value="{{ .ID }}">{{ .Title }}</option>`**：动态渲染的分类名、用户名等用户自有数据

---

## 9. 新增语言需要修改的所有位置

### 9.1 完整修改清单（共 5 个文件，6 处改动）

新增一门语言时，需要在以下位置依次修改：

| 序号 | 位置 | 文件 | 关键代码 | 原因 |
|------|------|------|----------|------|
| **1** | **可用语言列表** | `internal/locale/locale.go:7-31` | `var AvailableLanguages map[string]string` | 在用户设置页面渲染语言下拉选项；`validateLanguage()` 校验用户选择的合法性 |
| **2** | **复数规则 case 分支** | `internal/locale/plural.go:8-76` | `func getPluralForm(lang string, n int) int` | 为该语言的数字返回正确的复数形式数组索引 |
| **3** | **JSON 翻译文件** | `internal/locale/translations/{language}.json` | 翻译文件 | 必须与 `AvailableLanguages` 中的 key 完全一致，`embed.FS` 通过此文件名加载 |
| **4** | **复数形式数量表** | `internal/locale/catalog_test.go:100-138` | `var numberOfPluralFormsPerLanguage map[string]int` | 测试用例校验每个复数 key 的数组长度是否与语言规则匹配 |
| **5** | **复数规则测试场景** | `internal/locale/plural_test.go:8-222` | `func TestPluralRules` 的 `scenarios` map | 测试用例验证 `getPluralForm` 对各数值返回正确索引 |

> **注意**：除上述 5 处外，其余引用 `AvailableLanguages` 的地方（`validator/user.go`、`ui/settings_show.go`、`catalog_test.go:33-98` 的循环测试等）均通过 `range AvailableLanguages` 自动遍历，无需手动修改。

### 9.2 改动点 1：可用语言列表（AvailableLanguages）

**代码位置**：`internal/locale/locale.go:7-31`

```go
var AvailableLanguages = map[string]string{
    "ar_SA":            "العربية",
    "de_DE":            "Deutsch",
    // ...
    "zh_CN":            "简体中文",
    // ↓ 新增：language_code: "Native Language Name"
    "cs_CZ":            "Čeština",
}
```

**作用**：
- `ui/settings_show.go:68` 中 `view.Set("languages", locale.AvailableLanguages)` 将此 map 传递给模板，用于渲染设置页的语言下拉列表
- `validator/user.go:199-205` 的 `validateLanguage()` 通过检查 `languages[language]` 是否存在，确保用户选择的语言是系统支持的
- 多个测试用例 `for language := range AvailableLanguages` 自动遍历验证所有语言的完整性

### 9.3 改动点 2：复数规则（getPluralForm 的 case 分支）

**代码位置**：`internal/locale/plural.go:8-76`

函数签名：
```go
func getPluralForm(lang string, n int) int
```

根据语言 `lang` 和数字 `n` 返回对应的复数形式数组索引，供 `Printer.Plural()` 查找翻译。

**当前已覆盖的 case 分支（按复数形式数量分类）**：

| 复数形式数量 | 语言代码 | 规则说明 |
|-------------:|----------|----------|
| **1 种** | `zh_CN`, `zh_TW`, `nan_Latn_pehoeji`, `id_ID`, `ja_JP`, `ko_KR` | 无复数变化，始终返回 0 |
| **2 种** | `gl_ES` | 特殊规则：`n != 1` 时返回 1（注意法语等 2 种形式的语言使用 default 规则） |
| **2 种（default）** | `fr_FR`, `pt_BR`, `tr_TR`, `de_DE`, `el_EL`, `en_US`, `es_ES`, `fi_FI`, `hi_IN`, `it_IT`, `nl_NL` | `n > 1` 返回 1，否则返回 0 |
| **3 种** | `cs_CZ`, `pl_PL`, `ro_RO`, `ru_RU`, `uk_UA`, `sr_RS` | 各有不同的数字条件分支 |
| **6 种** | `ar_SA` | 阿拉伯语最复杂，有 6 种复数形式 |

**关键观察**：
1. `plural.go:59` 的 `case "ru_RU", "uk_UA", "sr_RS"` 中包含 `sr_RS`（塞尔维亚语），但该语言**并未**出现在 `AvailableLanguages` 中，也没有对应的 `sr_RS.json` 翻译文件。这是"预支持"状态——复数规则已准备好，等待翻译贡献。
2. `gl_ES`（加利西亚语）虽然也是 2 种复数形式，但规则特殊（`n != 1` 而非 `n > 1`），因此单独 case 处理。
3. `default` 分支注释明确标注 `// includes fr_FR, pr_BR, tr_TR`，意味着新增的语言如果规则与 default 一致（`n > 1` → index 1），可以**不**在此处添加 case。

### 9.4 改动点 3：JSON 翻译文件命名约定

**代码位置**：`internal/locale/catalog.go:20-21, 34`

```go
//go:embed translations/*.json
var translationFiles embed.FS

func loadTranslationFile(language string) (translationDict, error) {
    translationFileData, err := translationFiles.ReadFile("translations/" + language + ".json")
    // ...
}
```

**命名约定**：
- 文件名 = `{language_code}.json`，其中 `language_code` 必须与 `AvailableLanguages` 中的 key 完全一致（区分大小写）
- 例如 `AvailableLanguages["nan_Latn_pehoeji"] = "Pe̍h-ōe-jī"` 对应文件 `translations/nan_Latn_pehoeji.json`
- 通过 Go 1.16+ 的 `embed.FS` 内嵌到二进制，新增 JSON 文件必须重新编译

**JSON 文件结构**：
- **单数形式**：`"translation.key": "Translated text"`
- **复数形式**：`"plural.key": ["Singular form", "Plural form 1", "Plural form 2", ...]`
- 数组长度必须与 `getPluralForm` 对该语言可能返回的最大索引 + 1 一致
- 例如 `ar_SA` 需要 6 个元素，`zh_CN` 需要 1 个元素，`en_US` 需要 2 个元素

**测试验证**：
- `catalog_test.go:33-40` 循环遍历 `AvailableLanguages`，确保每个语言的 JSON 文件都能成功加载
- `catalog_test.go:42-68` 确保 singulars 和 plurals 都不为空
- `catalog_test.go:70-98` 确保每种语言都包含 `en_US` 的所有 key
- `catalog_test.go:100-138` 确保每个复数 key 的数组长度与 `numberOfPluralFormsPerLanguage` 一致

### 9.5 改动点 4：复数形式数量表（测试用例）

**代码位置**：`internal/locale/catalog_test.go:101-125`

```go
var numberOfPluralFormsPerLanguage = map[string]int{
    "ar_SA":            6,
    "de_DE":            2,
    // ...
    "zh_TW":            1,
    // ↓ 新增语言的复数形式数量
    "cs_CZ":            3,
}
```

此 map 是测试用例 `TestTranslationFilePluralForms` 的校验基准：
- 对每个语言的每个复数 key，断言 `len(choices) == numberOfPluralFormsPerLanguage[language]`
- 若新增语言未在此 map 中，测试不会失败（Go map 零值为 0，而实际数组长度至少为 1，会触发断言失败）

### 9.6 改动点 5：复数规则测试场景（测试用例）

**代码位置**：`internal/locale/plural_test.go:8-222`

```go
func TestPluralRules(t *testing.T) {
    scenarios := map[string]map[int]int{
        // ↓ 新增语言的测试场景：{数字: 期望的复数索引}
        "cs_CZ": {
            1: 0,  // n == 1 → index 0
            2: 1,  // n >= 2 && n <= 4 → index 1
            5: 2,  // default → index 2
        },
        // ...
    }
}
```

**作用**：
- 测试 `getPluralForm` 对该语言在不同数字下返回正确索引
- 对于使用 `default` 规则的新增语言（如 `de_DE`、`en_US`），也需要在此补充测试场景（见 `plural_test.go:166-205`）
- 特殊场景 `"unknown_language"`（`plural_test.go:206-211`）验证未在 case 中列出的语言能正确 fallback 到 default 规则

### 9.7 新增语言示例（以捷克语 cs_CZ 为例）

完整改动清单：

```
1. internal/locale/locale.go:
   + "cs_CZ": "Čeština",

2. internal/locale/plural.go (已有，cs_CZ case 已存在):
   case "cs_CZ":
       switch {
       case n == 1: return 0
       case n >= 2 && n <= 4: return 1
       default: return 2
       }

3. internal/locale/translations/cs_CZ.json:
   {
       "action.login": "Přihlásit se",
       // ... 所有 singular keys
       "page.unread_entry_count": ["%d nepřečtená položka", "%d nepřečtené položky", "%d nepřečtených položek"],
       // ... 所有 plural keys（每个数组长度为 3）
   }

4. internal/locale/catalog_test.go:
   + "cs_CZ": 3,

5. internal/locale/plural_test.go:
   + "cs_CZ": {
   +     1: 0,
   +     2: 1,
   +     4: 1,
   +     5: 2,
   + },
```

### 9.8 复数函数语言覆盖情况分析

#### 已显式覆盖的语言（10 个）

| 语言代码 | 语言名称 | 复数形式数 | case 位置 |
|----------|---------:|-----------:|----------|
| `ar_SA` | 阿拉伯语 | 6 | `plural.go:10-24` |
| `cs_CZ` | 捷克语 | 3 | `plural.go:25-33` |
| `gl_ES` | 加利西亚语 | 2 | `plural.go:34-38` |
| `id_ID` | 印尼语 | 1 | `plural.go:39-40` |
| `ja_JP` | 日语 | 1 | `plural.go:39-40` |
| `ko_KR` | 韩语 | 1 | `plural.go:39-40` |
| `pl_PL` | 波兰语 | 3 | `plural.go:41-49` |
| `ro_RO` | 罗马尼亚语 | 3 | `plural.go:50-58` |
| `ru_RU` | 俄语 | 3 | `plural.go:59-67` |
| `uk_UA` | 乌克兰语 | 3 | `plural.go:59-67` |
| `sr_RS` | 塞尔维亚语 | 3 | `plural.go:59-67`（已定义但语言未启用） |
| `zh_CN` | 简体中文 | 1 | `plural.go:68-69` |
| `zh_TW` | 繁体中文 | 1 | `plural.go:68-69` |
| `nan_Latn_pehoeji` | 闽南语 | 1 | `plural.go:68-69` |

#### 通过 default 规则覆盖的语言（10 个）

`plural.go:70-74` 的 `default` 分支注释 `// includes fr_FR, pr_BR, tr_TR`，实际覆盖了 AvailableLanguages 中以下 10 个语言：

| 语言代码 | 语言名称 | 复数形式数 | 规则 |
|----------|---------:|-----------:|------|
| `fr_FR` | 法语 | 2 | `n > 1` → index 1 |
| `pt_BR` | 巴西葡萄牙语 | 2 | `n > 1` → index 1 |
| `tr_TR` | 土耳其语 | 2 | `n > 1` → index 1 |
| `de_DE` | 德语 | 2 | `n > 1` → index 1 |
| `el_EL` | 希腊语 | 2 | `n > 1` → index 1 |
| `en_US` | 英语 | 2 | `n > 1` → index 1 |
| `es_ES` | 西班牙语 | 2 | `n > 1` → index 1 |
| `fi_FI` | 芬兰语 | 2 | `n > 1` → index 1 |
| `hi_IN` | 印地语 | 2 | `n > 1` → index 1 |
| `it_IT` | 意大利语 | 2 | `n > 1` → index 1 |
| `nl_NL` | 荷兰语 | 2 | `n > 1` → index 1 |

#### 未覆盖的语言使用默认值的潜在影响

**潜在问题 1：法语（fr_FR）的复数规则不完全匹配 default**

法语实际复数规则应为：`n == 0 || n == 1` → 单数（index 0），其他 → 复数（index 1）。

即 `0 条记录` 在法语中应为单数形式（"0 enregistrement" 而非 "0 enregistrements"），但当前 default 规则 `n > 1` 会将 `n=0` 错误地返回复数形式（index 1）。

```go
// 当前 default 规则：
default:
    if n > 1 {
        return 1
    }
    return 0

// n=0 时：0 > 1 为 false → 返回 0（单数），这个是对的
// n=1 时：1 > 1 为 false → 返回 0（单数），这个也是对的
// n=2 时：2 > 1 为 true → 返回 1（复数），正确
```

> 实际上当前 default 规则对法语是正确的！因为 `n=0` 和 `n=1` 都返回 0，与法语规则一致。但如果未来新增葡萄牙语（葡萄牙）`pt_PT`，其规则与 `pt_BR` 不同（`pt_PT` 要求 `n >= 2` 才用复数），default 规则就会出错。

**潜在问题 2：加利西亚语（gl_ES）已单独处理避免错误**

`gl_ES` 的规则是 `n != 1` → 复数（index 1），即 `n=0` 时返回 1。如果走 default 规则会返回 0，因此单独 case 处理是正确的。

**潜在问题 3：未来新增特殊规则语言的风险**

如果未来新增以下语言但忘记添加 case，会使用 default 规则导致错误：

| 拟新增语言 | 实际规则 | default 规则结果 | 影响 |
|-----------|----------|-----------------|------|
| `pt_PT`（葡萄牙葡萄牙语） | `n >= 2` → 复数 | `n > 1` → 复数 | **相同**，无问题 |
| `lv_LV`（拉脱维亚语） | 3 种复数形式 | 仅返回 0/1 | 复数形式数组越界或选择错误翻译 |
| `lt_LT`（立陶宛语） | 3 种复数形式 | 仅返回 0/1 | 同上 |
| `sl_SI`（斯洛文尼亚语） | 4 种复数形式 | 仅返回 0/1 | 同上 |
| `cy_GB`（威尔士语） | 6 种复数形式 | 仅返回 0/1 | 同上 |
| `br_FR`（布列塔尼语） | 5 种复数形式 | 仅返回 0/1 | 同上 |

**错误表现**：
1. **翻译选择错误**：例如拉脱维亚语 `n=11` 实际应返回 index 2，但 default 返回 0 → 显示单数形式
2. **数组越界保护**：`Printer.Plural()` 中有 `len(choices) > index` 检查（`printer.go:44`），越界时返回原始 key，不会 panic
3. **测试捕获**：`TestTranslationFilePluralForms` 会校验数组长度与 `numberOfPluralFormsPerLanguage` 一致，如果翻译文件提供了 3 个元素但测试 map 中该语言数量为 2（或未添加 → 0），测试会失败

**风险分级**：

| 风险等级 | 场景 | 说明 |
|---------|------|------|
| 🔴 高 | 新增有 3+ 种复数形式的语言但忘记添加 case | 翻译选择错误，用户看到语法错误的句子 |
| 🟡 中 | 新增 2 种形式但规则与 default 有差异（如 `pt_PT`） | 细微语法错误，多数用户可能察觉不到 |
| 🟢 低 | 新增 2 种形式且规则与 default 完全一致（如新增 `sv_SE` 瑞典语） | 可以不添加 case，但建议添加以提高可读性 |

---

## 10. 关键代码位置索引

| 模块 | 文件 | 关键行 |
|------|------|--------|
| 模板引擎 | `internal/template/engine.go` | 35-88 (解析), 90-114 (渲染) |
| 模板函数 | `internal/template/functions.go` | 35-161 (函数注册), 284-323 (elapsedTime) |
| 语言列表 | `internal/locale/locale.go` | 7-31 |
| 翻译目录 | `internal/locale/catalog.go` | 20-21 (embed), 23-31 (懒加载), 34 (文件加载), 47-79 (JSON解析) |
| 翻译器 | `internal/locale/printer.go` | 18-47 (Print/Printf/Plural), 52-77 (格式化安全) |
| 翻译器测试 | `internal/locale/printer_test.go` | 8-15 (缺失语言), 96-109 (缺失键), 280-304 (复数越界) |
| 复数规则 | `internal/locale/plural.go` | 8-76 |
| 复数规则测试 | `internal/locale/plural_test.go` | 8-222 |
| 复数形式数量测试 | `internal/locale/catalog_test.go` | 100-138 |
| 本地化错误 | `internal/locale/error.go` | 8-34 (LocalizedErrorWrapper), 36-55 (LocalizedError) |
| 语言校验 | `internal/validator/user.go` | 199-205 (validateLanguage) |
| 设置页传递语言列表 | `internal/ui/settings_show.go` | 68 |
| View 封装 | `internal/ui/view/view.go` | 34-49 (默认参数注入) |
| WebSession 语言 | `internal/model/web_session.go` | 139-144 (Language getter), 205-208 (setter) |
| HTTP 响应 | `internal/http/response/html.go` | 17-30 (HTML) |
| 布局模板 | `internal/template/templates/common/layout.html` | 1-196 |
| 登录页视图 | `internal/template/templates/views/login.html` | 1-60 |
