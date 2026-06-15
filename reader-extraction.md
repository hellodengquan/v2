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
| 核心提取算法 | `internal/reader/readability/readability.go` | 基于评分的通用正文提取 |
| 爬虫调度 | `internal/reader/scraper/scraper.go` | 选择提取策略（自定义/通用） |
| 内容处理器 | `internal/reader/processor/processor.go` | 编排完整处理流程 |
| 内容重写 | `internal/reader/rewrite/content_rewrite*.go` | 内容后处理规则 |
| HTML消毒 | `internal/reader/sanitizer/sanitizer.go` | 安全过滤、标签白名单 |
| 条目过滤 | `internal/reader/filter/filter.go` | 基于规则的条目筛选 |

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

## 三、提取策略的双轨制（Scraper 层）

`scraper.ScrapeWebsite`（scraper.go:21-71）是正文提取的入口，根据条件选择两种策略：

### 3.1 策略选择条件

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

### 3.2 自定义 CSS 选择器规则（`findContentUsingCustomRules`）

直接用 CSS 选择器定位元素，拼接所有匹配元素的 outerHTML：

```go
document.Find(rules).Each(func(i int, s *goquery.Selection) {
    if content, err := goquery.OuterHtml(s); err == nil {
        buf.WriteString(content)
    }
})
```

#### 预定义 Scraper 规则（`internal/reader/scraper/rules.go`）

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

## 四、内容清洗流程

### 4.1 内容重写规则（Rewrite Rules）

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

### 4.2 HTML 消毒（Sanitizer）

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

## 五、回退机制详解

### 5.1 多层回退链

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

### 5.2 回退1：无候选节点 → body 兜底

`getTopCandidate`（readability.go:239-255）如果所有节点评分都为 0 或没有任何评分段落，返回 `<body>` 元素，评分为 0。此时 `getArticle` 会把 body 的所有子元素按兄弟规则判定纳入，实际等同于返回整个页面的"净化版"。

### 5.3 回退2：Scraper 失败 → 保留原始内容

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

### 5.4 回退3：跨站重定向 → 禁用自定义规则

`ScrapeWebsite`（scraper.go:35-36）如果请求被重定向到其他域名：
- `sameSite` 判定为 false
- 即使有自定义 ScraperRules 也被**禁用**
- 自动降级为 Readability 通用算法

### 5.5 回退4：内容类型不符 → 拒绝提取

`isAllowedContentType`（scraper.go:105-109）只处理：
- `text/html/*`
- `application/xhtml+xml`

其他（PDF、图片、JSON、纯文本）直接报错返回，避免无效解析。

### 5.6 回退5：字符集自动转换

`encoding.NewCharsetReader`（scraper.go:42-45）处理非 UTF-8 编码的页面，支持：
- HTTP `Content-Type` 头声明的编码
- HTML `<meta>` 标签声明的编码（包括在文档 1024 字节之后才声明的情况）
- ISO-8859-1, Windows-1252, KOI8-R, UTF-8 BOM 等常见编码

---

## 六、过滤与筛选机制

### 6.1 双重过滤时机

过滤在处理流程中运行**两次**（processor.go:75, 147）：
1. **Scrape 之前**：基于 Feed 原始摘要 + 标题/URL 过滤
2. **Scrape 之后**：基于提取的全文内容重新过滤（仅当成功提取时）

### 6.2 过滤规则类型

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

### 6.3 判定逻辑（`IsBlockedEntry`）

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

## 七、性能优化设计

### 7.1 内存优化：避免重复字符串分配

`sumMapOnSelection`（readability.go:116-136）直接递归遍历 DOM 节点计算长度和逗号数，而不调用 `goquery.Selection.Text()`（会拼接整个文本字符串），节省大文档的内存和 GC 压力。

### 7.2 Regex 缓存

`filter.cachedRegex`（filter.go:53-72）使用 `sync.Map` 缓存编译后的正则，上限 1024 条，超限清空。避免重复编译相同规则。

### 7.3 基准测试覆盖

`readability_test.go` 包含：
- `BenchmarkExtractContent`：真实页面（GitHub / Wikipedia）提取性能
- `BenchmarkGetWeight`：关键词匹配性能
- `BenchmarkTransformMisusedDivsIntoParagraphs`：DOM 变换性能

---

## 八、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| Readability 主入口 | `internal/reader/readability/readability.go` | 72-102 |
| 不可能候选移除 | `internal/reader/readability/readability.go` | 190-237 |
| 候选评分算法 | `internal/reader/readability/readability.go` | 260-306 |
| 节点关键词权重 | `internal/reader/readability/readability.go` | 308-363 |
| 文章+兄弟节点组装 | `internal/reader/readability/readability.go` | 140-188 |
| 提取策略选择 | `internal/reader/scraper/scraper.go` | 21-71 |
| 完整处理流程编排 | `internal/reader/processor/processor.go` | 27-177 |
| 内容重写规则引擎 | `internal/reader/rewrite/content_rewrite.go` | 104-149 |
| 动态懒加载图片修复 | `internal/reader/rewrite/content_rewrite_functions.go` | 104-184 |
| HTML 白名单消毒 | `internal/reader/sanitizer/sanitizer.go` | 164-201 |
| 属性消毒 + URL 处理 | `internal/reader/sanitizer/sanitizer.go` | 440-585 |
| 条目过滤规则 | `internal/reader/filter/filter.go` | 101-127 |
