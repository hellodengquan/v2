# Reader 模式正文提取机制分析

本文档详细分析 Miniflux 中 Reader 模式（爬虫/全文提取）如何从网页中提取正文内容，包括提取规则、清洗流程和回退机制。

---

## 一、整体架构与处理流程

### 1.1 核心调用链路

```
handler.CreateFeed/RefreshFeed
  └── processor.ProcessFeedEntries
        ├── 过滤规则检查（filter.IsBlockedEntry）[前置]
        ├── URL 清理（urlcleaner.RemoveTrackingParameters）
        ├── URL 重写（rewrite.RewriteEntryURL）
        ├── scraper.ScrapeWebsite [如果启用 Crawler]
        │     ├── 内容类型校验
        │     ├── 字符集处理
        │     ├── 分支1：自定义规则 → findContentUsingCustomRules
        │     └── 分支2：通用算法 → readability.ExtractContent
        ├── rewrite.ApplyContentRewriteRules [内容重写]
        ├── 过滤规则检查（filter.IsBlockedEntry）[后置]
        └── sanitizer.SanitizeHTML [HTML消毒]
```

### 1.2 关键文件位置

| 模块 | 文件路径 | 职责 |
|------|---------|------|
| 请求构建 | `internal/reader/fetcher/request_builder.go` | HTTP 请求构造、代理注入、UA 设置 |
| 代理轮换 | `internal/proxyrotator/proxyrotator.go` | 多代理轮询调度 |
| 响应处理 | `internal/reader/fetcher/response_handler.go` | 响应读取、编码解压、错误归类 |
| 核心提取算法 | `internal/reader/readability/readability.go` | 基于评分的通用正文提取 |
| 爬虫调度 | `internal/reader/scraper/scraper.go` | 选择提取策略（自定义/通用） |
| 内容处理器 | `internal/reader/processor/processor.go` | 编排完整处理流程 |
| 内容压缩 | `internal/reader/processor/utils.go` | HTML minifier 压缩 |
| 内容重写 | `internal/reader/rewrite/content_rewrite*.go` | 内容后处理规则 |
| HTML消毒 | `internal/reader/sanitizer/sanitizer.go` | 安全过滤、标签白名单 |
| 媒体代理 | `internal/mediaproxy/rewriter.go` + `url.go` | 图片/音视频 URL 代理重写 |
| 条目过滤 | `internal/reader/filter/filter.go` | 基于规则的条目筛选 |

### 1.3 完整处理链路（含抓取 & 渲染端）

```
抓取 & 提取阶段（服务端存储前）:
  request_builder → proxyrotator(可选) → HTTP 抓取 → response_handler
    → scraper(策略选择) → readability / custom_rules
    → minifyContent 压缩 → rewrite_rules → filter(后置) → sanitizer → 存数据库

渲染阶段（读取时）:
  读数据库 → template 渲染 → proxyFilter(mediaproxy.RewriteDocumentWithRelativeProxyURL)
    → 浏览器展示
```

**注意**：mediaproxy 不在存储阶段执行，而是在**渲染/API 输出时**执行。存储的是原始绝对 URL。

---

## 二、提取规则：Readability 核心算法

### 2.1 预处理阶段

#### 步骤1：移除 `<script>` 和 `<style>`

```go
document.Find("script,style").Remove()   // readability.go:86
```

#### 步骤2：移除不可能的候选节点（`removeUnlikelyCandidates`）

通过元素的 `class` 和 `id` 属性匹配关键词列表，排除导航、广告、评论等非正文区域。

**三重移除判定策略**（`shouldRemoveCandidate`, readability.go:190-214）：

| 优先级 | 列表 | 示例关键词 | 行为 |
|--------|------|-----------|------|
| 1（强） | `strongCandidatesToRemove` | `popupbody`, `-ad`, `g-plus` | 无条件移除 |
| 2（可能误判） | `maybeCandidateToRemove` | `article`, `body`, `main`, `content`, `column` | 作为"误判白名单" |
| 3（弱） | `unlikelyCandidateToRemove` | `banner`, `sidebar`, `comment`, `footer`, `header`, `menu`, `social`, `sponsor` | 除非同时匹配 maybe，否则移除 |

**判定逻辑：**
- 若匹配 strong → **移除**
- 若同时匹配 maybe + unlikely → **保留**（假阳性保护）
- 仅匹配 unlikely → **移除**

**保护机制：**
- 永远不删除 `<html>` 和 `<body>` 标签
- 不删除 `<pre>` 和 `<code>` 内的任何元素（readability.go:227-229）

#### 步骤3：转换误用的 `<div>` 为 `<p>`（`transformMisusedDivsIntoParagraphs`）

很多网站用 `<div>` 包裹纯文本而非语义化标签。算法将以下 div 转为 `<p>`：
- 无子节点的空 div
- 所有子节点均为"行内类型"（非块级元素：非 a/blockquote/div/dl/img/ol/p/pre/table/ul）

---

### 2.2 候选节点评分（`getCandidates`）

对所有 `section,h2,h3,h4,h5,h6,p,td,pre,div` 标签进行评分：

#### 段落级评分（单个 `<p>` 等元素）

| 因子 | 规则 | 分值 |
|------|------|------|
| 基础分 | 段落本身存在 | +1 |
| 逗号数 | 逗号数量 + 1 | +(逗号数+1) |
| 文本长度 | 每100字符加1分，上限3分 | +min(长度/100, 3) |
| 过滤门槛 | 文本 < 25 字符 | 跳过不计分 |

#### 父节点继承

段落评分会累加到：
- **父节点**：加 100% 段落分
- **祖父节点**：加 50% 段落分（readability.go:294）

#### 节点初始权重（`scoreNode`）

| 标签 | 初始分 |
|------|--------|
| `div` | +5 |
| `pre`, `td`, `blockquote`, `img` | +3 |
| `address`, `ol`, `ul`, `dl`, `dd`, `dt`, `li`, `form` | -3 |
| `h1`~`h6`, `th` | -5 |
| 其他（p, section 等） | 0 |

#### 关键词权重（`getWeight`）

检查 `class` 和 `id` 属性（大小写不敏感）：

| 类型 | 关键词 | 权重 |
|------|--------|------|
| 正关键词 | `article`, `blog`, `body`, `content`, `entry`, `h-entry`, `hentry`, `main`, `page`, `pagination`, `post`, `story`, `text` | **+25** |
| 负关键词 | `author`, `banner`, `byline`, `com-`, `combx`, `comment`, `contact`, `dateline`, `foot`, `hid`, `masthead`, `media`, `meta`, `modal`, `outbrain`, `promo`, `related`, `scroll`, `share`, `shopping`, `shoutbox`, `sidebar`, `skyscraper`, `sponsor`, `tags`, `tool`, `widget`, `writtenby` | **-25** |

> **注意**：负关键词优先级更高——先遍历负列表，匹配即返回。因此 `class="article comment"` 得分为 -25。

#### 链接密度修正

最后，所有候选节点评分乘以 `(1 - linkDensity)`（readability.go:302）：
- 纯文本：评分不变（×1.0）
- 50% 文字是链接：评分减半（×0.5）
- 全是链接：评分归零（×0）

> 这是区分"正文"和"链接列表/导航"的关键机制。

---

### 2.3 选择最优候选（`getTopCandidate`）

取评分最高的节点作为正文容器。**回退机制**：若无任何候选（评分全为0或无匹配），直接使用 `<body>` 作为顶部分，评分为 0。

```go
if best == nil {
    best = &candidate{document.Find("body"), 0}  // readability.go:251
}
```

---

### 2.4 组装文章内容（`getArticle`）

不仅保留最佳候选，还检查其**兄弟节点**，避免遗漏被广告分隔的正文：

#### 兄弟节点阈值

```
siblingScoreThreshold = max(10, topCandidate.score / 5)
```

即兄弟节点评分 ≥ 最高评分的 20%（下限10分），也被纳入。

#### 段落特殊判定

不在候选列表但作为兄弟的 `<p>` 元素单独判定：

| 条件 | 结果 |
|------|------|
| 长度 ≥ 80 字符 **且** 链接密度 < 0.25 | ✅ 包含 |
| 长度 < 80 字符 **且** 链接密度 = 0 **且** 含完整句子 | ✅ 包含 |
| 其他情况 | ❌ 排除 |

**完整句子判定**（`containsSentence`）：以 `.` 结尾 或 包含 `". "`。

#### 标签包装

- `<p>` 标签：保持 `<p>` 包装
- 其他所有标签：统一用 `<div>` 包装
- 最终整体包裹在一个外层 `<div>` 中

---

## 三、HTTP 抓取层：请求构建与代理机制

### 3.1 请求构建器（RequestBuilder）

`internal/reader/fetcher/request_builder.go` 是所有出站 HTTP 请求的统一入口，采用**建造者模式**链式配置。

#### 3.1.1 代理优先级（三档）

`ExecuteRequest`（request_builder.go:143-157）按以下优先级选择代理，**只选一种**：

| 优先级 | 代理源 | 配置方式 | 适用场景 |
|--------|--------|---------|---------|
| 1（最高） | Feed 级自定义代理 | `feed.ProxyURL` 字段 | 单个 Feed 指定代理 |
| 2 | 应用级客户端代理 | `config.Opts.HTTPClientProxy()` + `UseCustomApplicationProxyURL(true)` | 全局代理 |
| 3（最低） | 代理轮换器 | `proxyrotator.ProxyRotatorInstance` | 多 IP 轮询避封禁 |

都未配置时走直连。

#### 3.1.2 代理轮换器（ProxyRotator）

`internal/proxyrotator/proxyrotator.go` —— 简单的**轮询（Round-Robin）**调度：

```go
func (pr *ProxyRotator) GetNextProxy() *url.URL {
    pr.mutex.Lock()
    proxy := pr.proxies[pr.currentIndex]
    pr.currentIndex = (pr.currentIndex + 1) % len(pr.proxies)  // 环形索引
    pr.mutex.Unlock()
    return proxy
}
```

- 全局单例：`ProxyRotatorInstance`
- 线程安全：`sync.Mutex` 保护索引自增
- 启动时从 `PROXY_ROTATOR_URLS` 环境变量（逗号分隔）初始化
- 每次 HTTP 请求调用 `GetNextProxy()` 取下一个，实现请求级负载均衡

#### 3.1.3 User-Agent 注入

`WithUserAgent(userAgent, defaultUserAgent)`（request_builder.go:75-82）：

```go
if userAgent != "" {
    r.headers.Set("User-Agent", userAgent)   // Feed 自定义 UA
} else {
    r.headers.Set("User-Agent", defaultUserAgent)  // 默认 UA
}
```

**设计意图**：
- 每个 Feed 可单独配置 UA，绕过针对特定爬虫 UA 的封禁
- 默认 UA 通常是 `Miniflux/版本号` 之类的标准标识
- 同时还支持自定义 Cookie、Basic Auth、ETag、Last-Modified 等缓存头

#### 3.1.4 安全防护：私有网络访问拦截

在 `Dialer.Control` 回调中检查（request_builder.go:177-190），**DNS 解析之后、TCP 连接之前**执行检查，避免 TOCTOU / DNS 重绑定攻击：

- 默认拦截所有私有 IP 访问（环回、内网、链路本地等）
- 显式配置的代理地址**例外**（允许连接代理本身）
- 重定向后的目标地址也会被检查（因为每个连接都经过 dialer）

#### 3.1.5 其他请求配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| 超时 | 20s | 连接+读取总超时 |
| Accept 头 | `application/xml, application/atom+xml, ..., text/html, */*` | 兼顾 Feed 和 网页 |
| Accept-Encoding | `br, gzip` | 支持 Brotli 和 Gzip 压缩 |
| Connection | `close` | 禁用 keep-alive，避免连接池泄漏 |
| 重定向 | 跟随 | 可用 `WithoutRedirects()` 禁用 |
| 压缩 | 启用 | 可用 `WithoutCompression()` 禁用 |
| HTTP/2 | 启用 | 可用 `DisableHTTP2()` 禁用 |
| TLS 校验 | 严格 | 可用 `IgnoreTLSErrors(true)` 放宽 |

### 3.2 响应处理器（ResponseHandler）

`internal/reader/fetcher/response_handler.go` 统一处理响应：

- **自动解压**：识别 `Content-Encoding: br/gzip`，用对应解压器包装 Body
- **大小限制**：`http.MaxBytesReader` 防止超大响应
- **错误分类**：将网络错误、TLS 错误、HTTP 状态码错误转为本地化错误
- **Cloudflare 检测**：通过 `cf-mitigated: challenge` 响应头识别 Cloudflare 拦截页
- **缓存协商**：读取 ETag / Last-Modified / Cache-Control 用于增量刷新

### 3.3 sameSite 跨站重定向判定

`scraper.ScrapeWebsite`（scraper.go:35-36）：

```go
sameSite := urllib.Domain(websiteURL) == urllib.Domain(responseHandler.EffectiveURL())
```

**判定方式**：
- 比较原始请求 URL 的域名与最终响应 URL 的域名
- 使用 `urllib.Domain()` 提取注册域名（如 `example.co.uk`，不是 IP 也不是子域名）
- 完全一致才算同站

**触发跨站的典型场景**：
1. 短链接跳转（t.cn → 目标站）
2. 移动端页面跳转（m.example.com → www.example.com 算同站，因为 domain 都是 example.com）
3. CDN 跳转
4. 登录/支付墙跳转

---

## 四、提取策略的双轨制（Scraper 层）

`scraper.ScrapeWebsite`（scraper.go:21-71）是正文提取的入口，根据条件选择两种策略：

### 4.1 策略选择条件

```go
if sameSite && rules != "" {
    // 使用自定义 CSS 选择器规则
    baseURL, extractedContent, err = findContentUsingCustomRules(...)
} else {
    // 使用 Readability 通用算法
    baseURL, extractedContent, err = readability.ExtractContent(...)
}
```

**触发自定义规则的两个必要条件：**
1. **同站跳转**：原始 URL 域名 == 最终响应 URL 域名（防止跨站后规则不匹配）
2. **规则存在**：用户配置的 `ScraperRules` 非空，或命中预定义规则表

### 4.2 自定义 CSS 选择器规则（`findContentUsingCustomRules`）

直接用 CSS 选择器定位元素，拼接所有匹配元素的 outerHTML：

```go
document.Find(rules).Each(func(i int, s *goquery.Selection) {
    if content, err := goquery.OuterHtml(s); err == nil {
        buf.WriteString(content)
    }
})
```

#### 4.2.1 预定义 Scraper 规则（`internal/reader/scraper/rules.go`）

针对知名网站的优化规则（部分示例）：

| 网站 | CSS 选择器 |
|------|-----------|
| `arstechnica.com` | `div.post-content` |
| `bbc.co.uk` | `div.vxp-column--single, div.story-body__inner, ul.gallery-images__list` |
| `github.com` | `article.entry-content` |
| `lemonde.fr` | `article` |
| `npr.org` | `#storytext` |
| `wikipedia.org` | `div#content` |
| `xkcd.com` | `div#comic` |

用户可通过 Feed 的 `ScraperRules` 字段覆盖或添加规则。

---

## 五、内容清洗流程

### 5.1 内容压缩（minifier）

**位置**：提取之后、重写之前执行（processor.go:139 和 processor.go:214）

使用 `github.com/tdewolff/minify/v2` 库的 HTML minifier，在 `processor/utils.go:69-73` 实现：

```go
var htmlMinifier = newHTMLMinifier()  // 全局单例

func newHTMLMinifier() *minify.M {
    m := minify.New()
    m.Add("text/html", &html.Minifier{
        KeepEndTags:         true,    // 保留结束标签（避免破坏DOM）
        KeepQuotes:          true,    // 保留属性引号
        KeepComments:        false,   // 删除注释
        KeepSpecialComments: false,   // 删除特殊注释（如 IE 条件注释）
        KeepDefaultAttrVals: false,   // 删除默认属性值
    })
    return m
}

func minifyContent(content string) string {
    ret, _ := htmlMinifier.String("text/html", content)
    return ret
}
```

**压缩策略的平衡设计**：
- **保留结束标签**：防止某些浏览器对可选结束标签的解析差异
- **保留属性引号**：避免无引号属性值导致的解析问题
- **删除注释**：HTML 注释对正文阅读无意义，减小存储体积
- **失败时返回原文**：minify 出错不影响内容可用性

**压缩收益**：减少 10-30% 的 HTML 体积（空白、注释冗余），降低数据库存储和传输开销。

### 5.2 内容重写规则（Rewrite Rules）

在提取**之后**、消毒**之前**执行（processor.go:144），用于站点特定修复。

#### 执行顺序

1. **预定义规则**（按域名，见 `content_rewrite_rules.go`）
2. **用户自定义规则**（Feed 的 `RewriteRules` 字段，若存在则替换预定义）
3. **强制规则**：`add_pdf_download_link`（始终追加）

#### 可用规则清单

| 规则 | 功能 |
|------|------|
| `add_image_title` | 将 img 的 title 属性转为 figure+figcaption |
| `add_dynamic_image` | 懒加载图片修复：从 data-src/data-original 等属性恢复 src |
| `add_dynamic_iframe` | 懒加载 iframe 修复 |
| `add_youtube_video` | 从 URL 解析 YouTube ID 并插入嵌入播放器 |
| `add_invidious_video` | Invidious 视频嵌入 |
| `fix_medium_images` | Medium 站点 noscript 图片恢复 |
| `use_noscript_figure_images` | 用 noscript 内的 img 替换 figure 中的懒加载图 |
| `nl2br` | 换行符转 `<br>` |
| `convert_text_links` | 纯文本 URL 转 `<a>` 链接 |
| `replace("pattern"|"replacement")` | 自定义正则替换内容 |
| `replace_title("pattern"|"replacement")` | 自定义正则替换标题 |
| `remove("css_selector")` | 按 CSS 选择器移除元素 |
| `remove_tables` | 剥除 table 结构保留内容 |
| `remove_clickbait` | 标题首字母大写规范化 |
| `add_pdf_download_link` | URL 以 .pdf 结尾时添加下载链接 |
| `fix_ghost_cards` | Ghost 博客的卡片 bookmark 转普通链接 |
| `remove_img_blur_params` | 移除图片 URL 的 blur= 查询参数 |
| `base64_decode("selector")` | Base64 解码指定选择器的文本 |

#### 动态图片属性优先级（`add_dynamic_image`）

按以下顺序查找真实图片 URL，找到第一个即停止：
```
data-src → data-original → data-orig → data-url → data-orig-file →
data-large-file → data-medium-file → data-original-mos →
data-2000src → data-1000src → data-800src → data-655src →
data-500src → data-380src → data-srcset → <noscript> 回退
```

#### 预定义 Rewrite 规则示例（部分）

| 域名 | 规则 |
|------|------|
| `medium.com` | `fix_medium_images` |
| `xkcd.com` | `add_image_title` |
| `youtube.com` | `add_youtube_video` |
| `theverge.com` | `add_dynamic_image, remove("div.duet--recirculation--related-list, .hidden")` |
| `bleepingcomputer.com` | `add_dynamic_image, remove(".ia_ad, .cz-related-article-wrapp, div[align]")` |

---

### 5.3 HTML 消毒（Sanitizer）

流程的**最后一道安全处理**（processor.go:165），确保输出 HTML 安全合规。

#### 核心策略：白名单机制

**允许的 HTML 标签**：`a, abbr, acronym, aside, audio, blockquote, b, br, caption, cite, code, dd, del, dfn, dl, dt, em, figcaption, figure, h1~h6, hr, i, iframe, img, ins, kbd, li, ol, p, picture, pre, q, rp, rt, rtc, ruby, s, small, samp, source, strong, sub, sup, table, td, tfoot, th, thead, time, tr, u, ul, var, video, wbr` + 完整 MathML 标签集

**不允许的标签**（直接跳过，保留子内容）：`div, span, section, article, header, footer` 等结构性标签（内容保留，标签剥离）

**完全阻止的标签**：`noscript`, `script`, `style`（连内容一起丢弃）

#### 安全检查项

| 检查项 | 规则 |
|--------|------|
| iframe 域名白名单 | 仅允许 `youtube.com`, `youtube-nocookie.com`, `player.vimeo.com`, `player.bilibili.com`, `bandcamp.com`, `open.spotify.com`, `soundcloud.com` 等 |
| 像素追踪图片 | 宽高均为 0 或 1 的 img 直接丢弃 |
| 隐藏元素 | 带 `hidden` 属性的元素丢弃 |
| 外链 scheme 校验 | 仅允许 http/https/ftp/mailto 等合法协议 |
| 社交分享资源 URL | 包含 `facebook.com/sharer`, `twitter.com/intent`, `linkedin.com/shareArticle` 等子串的链接被禁 |
| Data URI 图片 | 仅允许 image/* 类型的 data: URI |
| 相对 URL 解析 | 所有相对路径基于 baseURL 转为绝对 URL |
| 追踪参数移除 | URL 查询串中的追踪参数被清理 |

#### 自动属性注入

安全与体验增强属性（sanitizer.go:559-582）：

| 标签 | 注入属性 |
|------|---------|
| `<a>`（非锚点） | `rel="noopener noreferrer"`, `referrerpolicy="no-referrer"`, 可选 `target="_blank"` |
| `<video>`, `<audio>` | `controls` |
| `<iframe>` | `sandbox="allow-scripts allow-same-origin allow-popups allow-popups-to-escape-sandbox"`, `loading="lazy"` |
| `<iframe>`（YouTube） | 额外加 `referrerpolicy="strict-origin-when-cross-origin"` |
| `<img>` | `loading="lazy"` |

---

### 5.4 媒体代理（Media Proxy）

**位置**：不在存储阶段执行，在**渲染/API 输出时**执行（模板层 `proxyFilter` 函数）。

#### 5.4.1 调用时机

| 调用场景 | 函数 | 位置 |
|---------|------|------|
| Web 模板渲染 | `proxyFilter` → `RewriteDocumentWithRelativeProxyURL` | `internal/template/functions.go:76` |
| REST API 输出 | `RewriteDocumentWithAbsoluteProxyURL` | `internal/api/entry_handlers.go` |
| Google Reader API | `RewriteDocumentWithAbsoluteProxyURL` | `internal/googlereader/handler.go:693` |
| Fever API | `RewriteDocumentWithAbsoluteProxyURL` | `internal/fever/handler.go:319` |
| 入口 Scraper API | `RewriteDocumentWithRelativeProxyURL` | `internal/ui/entry_scraper.go:64` |

**设计原因**：代理 URL 包含路径前缀（`BasePath`）和签名，这些是**运行时配置**，不是内容的固有属性。存库时存储原始 URL，渲染时再转换，避免配置变更导致数据失效。

#### 5.4.2 代理模式

由 `MEDIA_PROXY_MODE` 环境变量控制：

| 模式 | 行为 |
|------|------|
| `none` | 不代理任何媒体资源（默认） |
| `http-only` | 仅代理 HTTP 协议的媒体（混合内容场景，HTTPS 页面中的 HTTP 图片会被浏览器拦截） |
| `all` | 代理所有 HTTP/HTTPS 媒体 |

#### 5.4.3 资源类型

由 `MEDIA_PROXY_RESOURCE_TYPES` 控制，可选项：`image`, `audio`, `video`

处理的 HTML 元素：
- **image**：`<img>`, `<picture> <source>`；`<video poster>`（如果未启用 video 类型）
- **audio**：`<audio>`, `<audio> <source>`
- **video**：`<video>`, `<video> <source>`, `<video poster>`

同时支持 `srcset` 属性的解析和重写（`sanitizer.ParseSrcSetAttribute`）。

#### 5.4.4 URL 签名机制

`internal/mediaproxy/url.go` 使用 HMAC-SHA256 签名，防止代理被滥用：

```go
func ProxifyRelativeURL(mediaURL string) string {
    mac := hmac.New(sha256.New, config.Opts.MediaProxyPrivateKey())
    mac.Write([]byte(mediaURL))
    digest := mac.Sum(nil)
    // 格式: /proxy/{hmac_signature}/{base64_url}
    return fmt.Sprintf("%s/proxy/%s/%s",
        config.Opts.BasePath(),
        base64.URLEncoding.EncodeToString(digest),
        base64.URLEncoding.EncodeToString([]byte(mediaURL)))
}
```

代理端点收到请求后，用相同密钥重新计算 HMAC 并对比，只有签名一致才会去请求真实媒体资源。这样防止了攻击者构造任意 URL 让服务器做 SSRF 请求。

#### 5.4.5 自定义外部代理

如果配置了 `MEDIA_CUSTOM_PROXY_URL`，则不使用内置代理，直接将 URL 转发到外部代理服务：

```go
func proxifyURLWithCustomProxy(mediaURL string, customProxyURL *url.URL) string {
    absoluteURL := customProxyURL.JoinPath(base64.URLEncoding.EncodeToString([]byte(mediaURL)))
    return absoluteURL.String()
}
```

---

## 六、回退机制详解

### 6.1 多层回退链

```
提取策略选择：
├─ 用户配置 ScraperRules（同站）→ 自定义 CSS 选择器
│    └─ 失败/无匹配 → 返回空内容 → 保留 Feed 原始摘要
├─ 命中预定义 ScraperRules → 同上
└─ 其他情况 → Readability 算法
     ├─ 有高分候选 → 最佳节点 + 兄弟节点
     ├─ 候选分数极低 → 退化为 <body> 全选
     └─ 解析异常 → 返回错误 → 保留 Feed 原始摘要
```

### 6.2 回退1：无候选节点 → body 兜底

`getTopCandidate`（readability.go:239-255）如果所有节点评分都为 0 或没有任何评分段落，返回 `<body>` 元素，评分为 0。此时 `getArticle` 会把 body 的所有子元素按兄弟规则判定纳入，实际等同于返回整个页面的"净化版"。

### 6.3 回退2：Scraper 失败 → 保留原始内容

`ProcessFeedEntries`（processor.go:129-141）：
- 若 `scraper.ScrapeWebsite` 返回错误 → 记录日志但**不中断**
- 若提取内容为空字符串 → 不替换原有内容
- 只有成功且非空时，才用提取内容替换 Feed 摘要

```go
if scraperErr != nil {
    slog.Warn("Unable to scrape entry", ...)  // 记录警告，继续
} else if extractedContent != "" {
    entry.Content = minifyContent(extractedContent)  // 成功才替换
    contentExtractedSuccessfully = true
}
```

### 6.4 回退3：跨站重定向 → 禁用自定义规则

`ScrapeWebsite`（scraper.go:35-36）如果请求被重定向到其他域名：
- `sameSite` 判定为 false
- 即使有自定义 ScraperRules 也被**禁用**
- 自动降级为 Readability 通用算法

### 6.5 回退4：内容类型不符 → 拒绝提取

`isAllowedContentType`（scraper.go:105-109）只处理：
- `text/html/*`
- `application/xhtml+xml`

其他（PDF、图片、JSON、纯文本）直接报错返回，避免无效解析。

### 6.6 回退5：字符集自动转换

`encoding.NewCharsetReader`（scraper.go:42-45）处理非 UTF-8 编码的页面，支持：
- HTTP `Content-Type` 头声明的编码
- HTML `<meta>` 标签声明的编码（包括在文档 1024 字节之后才声明的情况）
- ISO-8859-1, Windows-1252, KOI8-R, UTF-8 BOM 等常见编码

### 6.7 回退6：私有网络访问被拒

`RequestBuilder` 的 dialer 控制回调（request_builder.go:177-190）：
- 所有出站请求都经过私有 IP 检查
- 重定向后的目标地址也会被检查
- 保护 SSRF 攻击（服务端请求伪造）
- 可通过 `FETCHER_ALLOW_PRIVATE_NETWORKS=1` 关闭（开发环境用）

---

## 七、过滤与筛选机制

### 7.1 双重过滤时机

过滤在处理流程中运行**两次**（processor.go:75, 147）：
1. **Scrape 之前**：基于 Feed 原始摘要 + 标题/URL 过滤
2. **Scrape 之后**：基于提取的全文内容重新过滤（仅当成功提取时）

### 7.2 过滤规则类型

#### 结构化过滤规则（`BlockFilterEntryRules` / `KeepFilterEntryRules`）

每行一条，格式：`RuleType=正则表达式`

| 规则类型 | 作用域 |
|---------|--------|
| `EntryTitle` | 条目标题 |
| `EntryURL` | 条目 URL |
| `EntryCommentsURL` | 评论 URL |
| `EntryContent` | 条目内容（Scrape 后可匹配全文） |
| `EntryAuthor` | 作者 |
| `EntryTag` | 标签（任一标签匹配即可） |
| `EntryDate` | 发布日期（特殊语法：`future`, `before:YYYY-MM-DD`, `after:YYYY-MM-DD`, `between:YYYY-MM-DD,YYYY-MM-DD`, `max-age:30d`） |

#### 正则过滤规则（`BlocklistRules` / `KeeplistRules`）

简单正则字符串，同时匹配：URL、Title、Author、Tags（任一标签）。

### 7.3 判定逻辑（`IsBlockedEntry`）

```
流程：
1. 匹配 BlockFilterEntryRules → 命中则屏蔽
2. 匹配 BlocklistRules（正则）→ 命中则屏蔽
3. 若有 KeepFilterEntryRules → 必须匹配至少一条，否则屏蔽
4. 若有 KeeplistRules → 必须匹配正则，否则屏蔽
5. 以上都未触发 → 保留
```

用户级规则和 Feed 级规则通过 `ParseRules` 合并生效。

---

## 八、设计决策分析

### 8.1 ScraperRules 与 Readability 的取舍设计

**两种策略的对比：**

| 维度 | CSS 选择器（ScraperRules） | Readability 通用算法 |
|------|---------------------------|---------------------|
| 精确度 | 极高（精确定位正文容器） | 中等（基于评分推测） |
| 适用范围 | 仅匹配特定站点结构 | 通用，任何网站都能试 |
| 维护成本 | 高（网站改版即失效） | 零维护 |
| 提取速度 | 快（一次选择器查询） | 慢（遍历所有候选节点 + 评分计算） |
| 健壮性 | 差（结构一变就废） | 好（基于统计特征） |
| 内容完整性 | 完全可控 | 可能漏段落或带杂质 |

**分层策略设计思想**：
1. **优先定制**：有明确 CSS 规则时用定制，保证质量
2. **通用兜底**：没有规则时自动降级 Readability，保证可用性
3. **sameSite 闸门**：跨站跳转时强制用 Readability，因为 CSS 规则是按源站域名匹配的，跳转到第三方站点后规则完全不适用
4. **渐进增强**：预定义规则表覆盖主流站点，用户也可自行添加，形成"社区贡献 + 通用算法"的双层保障

### 8.2 强制 sameSite 的深层理由

`scraper.go:35` 要求 `sameSite && rules != ""` 两个条件同时满足才使用自定义规则：

**安全原因：**
- **CSS 选择器注入风险**：ScraperRules 是用户配置的，如果 URL 被恶意重定向到钓鱼站点，选择器可能匹配到意外内容（虽然 CSS 选择器本身不执行代码，但可能提取到错误/有害内容并展示给用户）
- **域名绑定原则**：规则是针对特定域名编写的，跨站后 DOM 结构完全不同，强制同站保证规则的语义正确性

**产品原因：**
- **短链接场景**：很多 Feed 的条目 URL 是短链接（t.cn、bit.ly），点击后才跳转到真实文章。如果直接用原始域名匹配规则，永远匹配不上
- **同站判定用注册域名**：`urllib.Domain()` 提取的是 eTLD+1（如 `example.co.uk`），所以 `www.example.com` → `m.example.com` 算同站，子域名跳转不触发降级
- **可预期行为**：用户配置的是"针对这个网站的规则"，跳转到别的网站自动换算法，符合直觉

### 8.3 Sanitizer 剥离 div/span 但保留子内容的理由

`sanitizer.go` 对 `div`, `span`, `section`, `article` 等结构性标签的处理是"剥壳留肉"——**删除标签本身，但保留其内部的文本和子元素**。

**安全角度：**
- `div` 和 `span` 本身是**中性标签**，不携带脚本或交互行为，没有直接安全风险
- 真正危险的是 `script`、`iframe`、`form`、带事件处理器的元素等，这些被完全阻止或严格限制
- 直接删除结构标签的**内容**会丢失正文，得不偿失

**阅读体验角度：**
- Reader 模式的核心是"阅读"，不是"原样呈现"
- `div` 通常用于布局控制（float、flex、grid、margin），这些样式在阅读模式下反而干扰阅读
- 剥离 `div` 让内容回归流式布局，统一渲染体验
- 保留内容（p、img、blockquote 等语义标签）保证正文完整

**为什么不直接保留 div 再清除样式？**
- 样式属性（style attribute）本身可以做很多事情：注入背景图追踪、设置透明层遮挡、调整字体大小到不可读等
- 白名单机制更简单可靠：只放行已知安全的标签和属性
- 结构性标签对阅读无增益，删之无害

**类比设计**：类似 Readability 的"内容提取 + 语义标签保留"思路，sanitizer 是第二道防线，进一步净化输出。

---

## 九、性能优化设计

### 9.1 内存优化：避免重复字符串分配

`sumMapOnSelection`（readability.go:116-136）直接递归遍历 DOM 节点计算长度和逗号数，而不调用 `goquery.Selection.Text()`（会拼接整个文本字符串），节省大文档的内存和 GC 压力。

### 9.2 Regex 缓存

`filter.cachedRegex`（filter.go:53-72）使用 `sync.Map` 缓存编译后的正则，上限 1024 条，超限清空。避免重复编译相同规则。

### 9.3 基准测试覆盖

`readability_test.go` 包含：
- `BenchmarkExtractContent`：真实页面（GitHub / Wikipedia）提取性能
- `BenchmarkGetWeight`：关键词匹配性能
- `BenchmarkTransformMisusedDivsIntoParagraphs`：DOM 变换性能

### 9.4 Minifier 全局单例

`htmlMinifier` 在包初始化时创建一次（`processor/utils.go:17`），避免每次请求都重新构建 minify 实例和配置。

### 9.5 连接关闭策略

`Connection: close` 头（request_builder.go:270）禁用 HTTP keep-alive，避免大量 Feed 抓取任务耗尽连接池或导致连接泄漏。代价是每次请求重连 TCP，对抓取延迟不敏感的后台任务可接受。

---

## 十、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 请求构建器主入口 | `internal/reader/fetcher/request_builder.go` | 143-284 |
| 代理轮换器 | `internal/proxyrotator/proxyrotator.go` | 11-56 |
| 响应处理器 | `internal/reader/fetcher/response_handler.go` | 23-275 |
| sameSite 判定 | `internal/reader/scraper/scraper.go` | 35-36 |
| Readability 主入口 | `internal/reader/readability/readability.go` | 72-102 |
| 不可能候选移除 | `internal/reader/readability/readability.go` | 190-237 |
| 候选评分算法 | `internal/reader/readability/readability.go` | 260-306 |
| 节点关键词权重 | `internal/reader/readability/readability.go` | 308-363 |
| 文章+兄弟节点组装 | `internal/reader/readability/readability.go` | 140-188 |
| 提取策略选择 | `internal/reader/scraper/scraper.go` | 21-71 |
| 完整处理流程编排 | `internal/reader/processor/processor.go` | 27-177 |
| HTML minifier 压缩 | `internal/reader/processor/utils.go` | 17-73 |
| 内容重写规则引擎 | `internal/reader/rewrite/content_rewrite.go` | 104-149 |
| 动态懒加载图片修复 | `internal/reader/rewrite/content_rewrite_functions.go` | 104-184 |
| HTML 白名单消毒 | `internal/reader/sanitizer/sanitizer.go` | 164-201 |
| 属性消毒 + URL 处理 | `internal/reader/sanitizer/sanitizer.go` | 440-585 |
| 媒体代理重写器 | `internal/mediaproxy/rewriter.go` | 19-95 |
| 媒体代理 URL 签名 | `internal/mediaproxy/url.go` | 16-61 |
| 模板层 proxyFilter | `internal/template/functions.go` | 76-85 |
| 条目过滤规则 | `internal/reader/filter/filter.go` | 101-127 |
