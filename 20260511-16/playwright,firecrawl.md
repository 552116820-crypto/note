# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `d7df11ae-9e53-4e4a-9aa7-67d76bb4518c`
- **范围**: Turn 1-3
- **导出时间**: 2026-05-21 08:39:09

---

### Turn 1  *(2026-05-21 00:21)*

**User:**

playwright 和 firecrawl 这两个mcp是不是差不多

**Assistant:**

不太一样。表面上都能"访问网页"，但定位差别挺大：

| | **Playwright MCP** | **Firecrawl MCP** |
|---|---|---|
| 本质 | 本地浏览器自动化 | 云端抓取/爬取服务（API） |
| 核心动作 | click / type / fill / snapshot / 截图 | scrape / crawl / map / search / extract |
| 强项 | **交互**：登录、点按钮、填表单、测自己的网页 | **取内容**：批量页面 → 干净 markdown / 结构化数据 |
| JS 渲染 | 真浏览器本地跑 | 服务端渲染，自带反爬/代理池 |
| 规模 | 单页、一步步操作 | 整站爬取、批量、给 LLM/RAG 喂数据 |

## 怎么选

- **要"操作"一个页面**（登录态、点击流程、测试你自己开发的前端）→ Playwright
- **要"拿到"网页内容**（把一堆 URL 变成 markdown、爬整站、抽结构化字段、网页搜索）→ Firecrawl

## 为什么你会觉得"差不多"

Firecrawl 新版也加了 `firecrawl_browser_*` 和 `firecrawl_interact` 这类浏览器会话工具，所以**有重叠区**——简单的"打开页面点一下再抓内容"两者都能做。但：

- Playwright 不擅长把内容清洗成 LLM 友好的 markdown，也没有整站爬取/搜索。
- Firecrawl 的交互能力是补充功能，深度复杂的 UI 自动化和调试体验不如 Playwright。

一句话：**Playwright 是"机械手"，Firecrawl 是"内容管道"。** 你这边还装了 `chrome-devtools`（偏调试/性能）、`jina` / `tavily`（偏搜索+读取）——如果只是要网页内容，jina/tavily/firecrawl 三选一即可，不必和 Playwright 二选一。

---

### Turn 2  *(2026-05-21 00:28)*

**User:**

爬取数据需要ocr的场景，Firecrawl能做吗

**Assistant:**

简短答案：**Firecrawl 不是 OCR 工具**，它的核心是把网页 DOM/HTML 转成 markdown——图片里"烧进去"的文字它读不到。

## 分场景看

| 内容形态 | Firecrawl 能不能拿到 |
|---|---|
| 网页里是真实文本（DOM 节点） | ✅ 没问题，这是它本职 |
| 文字烧进图片（截图、信息图、JPG 里的字） | ❌ scrape 读的是 DOM，图里的字拿不到 |
| 文本层 PDF（可复制选中的） | ✅ `firecrawl_parse` / scrape 的 PDF 解析能拿 |
| 扫描件 / 图片型 PDF | ⚠️ 灰色地带，不保证有 OCR，取决于它后端当前实现 |

几个容易踩的点：

- **`firecrawl_extract` 不是 vision OCR**。它是在"已经抓到的文本"上跑 LLM 做结构化抽取——上游没文本，它也变不出来。
- **`screenshot` 格式 ≠ OCR**。Firecrawl 能截图，但截图只是图片，不会自动转文字。

## 需要 OCR 的推荐架构

把"抓"和"认字"拆开：

```
Firecrawl 抓页面/列 URL  →  发现图片型内容  →  专门的 OCR/多模态模型识别
```

你工具箱里更对口 OCR 场景的：

- **`mcp__jina__extract_pdf`**：专门抽 PDF 里的图/表/公式，对扫描件比 Firecrawl 友好。
- **`mcp__jina__read_url`**：Jina Reader 对页内图片有 captioning/识别能力，比纯 DOM 抓取多一层。
- 量大或要精度，直接上多模态模型（Claude / 专用 OCR 服务）对图片批量识别。

## 一个不确定点

"扫描件 PDF 到底 OCR 不 OCR"是 Firecrawl 后端实现细节，我的知识可能过时。**要我去查一下 Firecrawl 最新文档确认这块吗？** 几秒钟的事。

---

### Turn 3  *(2026-05-21 00:30)*

**User:**

要

**Assistant:**

查完了，**我上一条关于"扫描件 PDF 是灰色地带"的说法已经过时，得更正**。

## 更正：扫描件 PDF 现在能 OCR ✅

Firecrawl 2025 年上线了 **Fire-PDF（PDF Parser v2）**——Rust 写的解析引擎，扫描件、图片型、混合 PDF 都能转成 markdown。扫描区域会送进视觉模型（GLM-OCR）识别。

三档模式，用 `parsers` 参数控制：

```js
parsers: [{ type: "pdf", mode: "ocr", maxPages: 20 }]
```

| mode | 行为 |
|---|---|
| `auto`（默认） | 先试文本层，抓不到自动回退 OCR |
| `fast` | 只读文本层，跳过扫描页，最快 |
| `ocr` | 强制逐页 OCR，扫描件/auto 误判时用 |

它还有个开源的 `pdf-inspector` 先做分类（看字体编码、图片占比），只把扫描页路由进 OCR，所以不会无脑全 OCR 拖慢。

## 但有边界：OCR 只覆盖"文档"，不覆盖"网页图片"

文档明确写的支持格式只有：**PDF、Word（docx/doc/odt/rtf）、Excel（xlsx/xls）**。

所以我上一条这部分**仍然成立**：

- ❌ 独立的 PNG / JPG 图片文件 —— 文档没说支持
- ❌ HTML 网页里嵌的图片（截图、信息图、JPG 里烧的字）—— 不会被 OCR
- ✅ 只要内容是 PDF（哪怕纯扫描件），就能 OCR

## 端点

- ✅ `scrape`（URL 指向 PDF）、`parse`（上传文件）支持 `parsers`
- ⚠️ `crawl` 文档没提 —— 整站爬 PDF 时这块要自己验证

## 给你的结论

- **要爬的是 PDF（含扫描件）** → Firecrawl 现在够用，`mode` 设 `auto` 或 `ocr` 即可，不用再外挂 OCR。
- **要 OCR 网页里的图片 / 独立图片** → Firecrawl 还是不行，得走多模态模型（Claude vision、专用 OCR），或试 `jina read_url` 的图片识别。

用 MCP 时对应 `firecrawl_scrape` / `firecrawl_parse`——记得确认这俩工具的 schema 有没有透出 `parsers` 参数，没透出就得直接调 Firecrawl API。

Sources:
- [Document Parsing | Firecrawl Docs](https://docs.firecrawl.dev/features/document-parsing)
- [Introducing Fire-PDF: Firecrawl's New PDF Parsing Engine](https://www.firecrawl.dev/blog/fire-pdf-launch)
- [Introducing PDF Parser v2: Faster Extraction with Auto Mode](https://www.firecrawl.dev/blog/pdf-parser-v2)

---
