# 周一分享：搭建基于 Claude Code 的竞企调研工作流

> 分享目标：不是介绍"AI 很强"，而是复盘两个真实项目里，如何把 Claude Code 从聊天工具用成一套可复用、可并行、可审查的竞企/产品调研流水线。

<!-- PART I · 锚定结论 -->

## 0. 先用 5 分钟讲清楚结论

**要点**：竞企调研的难点不在搜资料，而在把不确定的输入变成可复用、可追溯、可交付的工程流程。

这次分享可以从一句话开始：

**竞企调研的难点不在"让 AI 搜资料"，而在把不确定的网页、图片、PDF、公司边界、品类边界，变成一套能反复跑、能追溯、能人审、最后能交付 Excel 的工程流程。**

我这周主要验证了两类项目：

| 项目 | 目标 | 输入 | 输出 | Claude Code 的角色 |
|---|---|---|---|---|
| 研发岗竞企图谱 | 为 7 个研发岗找可挖人的竞争企业 | 展会图片、展商名单、岗位品类 | 公司 × 岗位的 xlsx | 找展会、识别公司、验证品类、处理边界 |
| 涂料行业产品调研 | 把同行产品按三级结构和行业列汇总 | 同行公司官网、产品页、图片/PDF | 公司 × 产品 × 行业的 xlsx | 拆产品树、识别型号、找证据 URL、并行调研 |

两个项目的共同方法论是：

1. **先定交付物**：样板 xlsx、列名、超链接、合并单元格、证据格式先固定。
2. **再定边界**：什么算竞争企业，什么只是相邻品类，什么必须 unmatched。
3. **把工作分层**：Python 做确定性工作，Claude 做语义判断，人做人审 override。
4. **用 skill / agent / command 固化流程**：不是每次从头 prompt，而是沉淀成项目资产。
5. **用 review 纠偏**：AI 第一次跑出来的结果一定有口径问题，必须留审查与回滚通道。

接下来 9 节按这个结构展开：先讲 3 个核心抽象（§1-3），再讲 2 个工程化抓手（§4-5），最后看成果、教训和 SOP（§6-9）。

<!-- PART II · 三个核心抽象 -->

## 1. Skill 与 Agent：先选对形态，再写内容

**要点**：干活外包用 agent，改变姿态用 skill，项目级总入口用 command；这个项目 0 个 skill 是合理的。

### 1.1 三个概念要分清

| 类型 | 放在哪里 | 作用 | 是否开新上下文 | 适合做什么 |
|---|---|---|---|---|
| Agent / subagent | `.claude/agents/*.md` | 定义一个子代理 | 是 | 长任务、并行任务、隔离上下文 |
| Slash command | `.claude/commands/*.md` | 一键注入主流程 prompt | 否 | `/run-position P01` 这种总控入口 |
| Skill | `.claude/skills/*/SKILL.md` 或全局 skill | 可复用流程说明 | 通常否 | 改变对话姿态（grill-me）、跨项目通用能力 |

我的理解：

- **Agent 是执行单元**：可以派出去跑一家公司、一批展商、一次 review。
- **Skill 是流程资产 / 姿态切换**：改变我处理对话的方式（例如 `/grill-me` 让我变成审问者）。
- **Command 是总入口**：让主会话按固定剧本调度 agent 和 Python 脚本。

### 1.2 涂料项目其实更适合写成 agent

涂料项目历史上写成了 `research-one-company/SKILL.md`，但从"该用哪种形态"的角度复盘，更应该是 agent。真正的判断标准是这张表：

| 任务特征 | 倾向 |
|---|---|
| 大量 web fetch / OCR / 解析 HTML | → agent（隔离 token） |
| 文档篇幅长（>200 行 Phase 指令） | → agent（不进主会话） |
| 输出是文件 / 结构化 JSON | → agent（只回 summary 就够） |
| 可能批量并行（10+ 家） | → agent（必须） |
| 单次调用即结束，不需要继续对话 | → agent |

涂料项目的"单家调研"**5 条全中**——648 行 Phase 文档 + 每家公司十几次 fetch_html + OCR + 详情页解析——本该从一开始就写成 agent。skill 形态最早跑通后没有动力重构，但作者自己在 SKILL.md 里也承认"批量调度（10+ 家并行）场景下不要直接调，那是主会话用 Agent 工具的活"。

**记忆方法**：

> **干活外包 → agent；改变我的姿态 → skill**。

- `research-one-company` 是典型"干活"：抓站、解析、写 JSON，跑完没我什么事 → 该用 agent。
- `grill-me` 是典型"改姿态"：让我反复追问，整段对话都受它影响，必须在主会话里 → 该用 skill。

### 1.3 项目级 skill 不会被 subagent 自动继承

实测 subagent 可以 Read 项目里的 SKILL.md，但不一定能直接通过 Skill 工具调用项目 skill。最稳的做法是：

```text
subagent 开工前先 Read：
- CLAUDE.md
- .claude/skills/research-one-company/SKILL.md（如果还保留 skill 形态）
- 关键 reference / METHOD.md
```

这比在 prompt 里复制 200 行规则更稳，也避免规则双写。

### 1.4 为什么这两个项目 0 个 skill 是合理设计

skill 的设计假设是 **"改变我的对话姿态"** 或 **"短小通用能力"**。这两个项目的特征恰好都不是：

| 项目特征 | 谁吃下了 |
|---|---|
| 工作流**高度专业化**（展会反搜 → 验证 → 评分 → xlsx） | `.claude/commands/run-position.md` 整份剧本 |
| 重活全是**长任务 + 重 IO** | `.claude/agents/` 下的 3 个 subagent（finder / verifier / merger） |
| 用户交互是**多轮闸门** | command 剧本里的"等用户回 'ok'"约束 |

这三件事 **commands + agents 已经吃满，skill 能填的缝很少**。再回头看 skill 的甜区：

> skill 的甜区是**"会反复用 + 描述足够具体让 LLM 主动触发"**。这个项目大部分场景是**一次性专业流程**，写进文档让我读就行。

所以社区 skill 帮不上不是 AI 选错工具，是这个 repo 根本不需要。

### 1.5 那 skill 的价值到底在哪

不在通用性，在**可移植性**：

- command / agent 锁死在单 repo
- skill 可以全局可用 / 跨项目复用 / 打包发布

对一个深耕单领域 + 单 repo 跑到底的项目，skill 价值低。如果以后跨多项目（研发岗 + 涂料 + 客户尽调）有重复动作（"验证一家公司是不是真有官网"、"中文公司名归一"），再把这些抽出来沉淀成私人 skill 库。

---

## 2. Plan Mode：先锁边界，再动手

**要点**：plan mode 不是默认开，已有详细 plan 时直接喂；它真正的价值是只读护栏 + 强制核对外部假设。

### 2.1 不是每次都要进 plan mode

Plan mode 的核心价值有两个：

1. **生成计划** — 让 AI 帮你写步骤。
2. **只读安全护栏** — 进入 plan mode 后我只能读不能写，强制走"先确认再动手"。

**判断要不要进 plan mode**：

| 场景 | 是否需要 plan mode |
|---|---|
| 完全没想清楚要干啥 | ✅ 让 AI 边读边帮你拟计划 |
| 已经有详细 md 计划，但里面有外部假设需先核对（文件路径是否存在、字段是否还叫这个名、依赖版本对不对）| ✅ 进 plan mode 让我先验证再动手 |
| 已经有详细计划，所有路径和依赖都拍板了 | ❌ 直接喂 md 文件 + TaskCreate 拆任务执行，跳过 plan mode 更顺 |

实战经验：当我把 `新plan，纯claude code.md` 那份 16K 字的 plan 喂过去后，AI 直接判断"不需要 plan mode 了，可以开干"——因为 6 个关键决策已经拍板、5 个 Phase + 闸门式人工 review、断点续传机制、目录结构、out-of-scope 全列清了。这种成熟度的 plan，进 plan mode 反而是浪费。

### 2.2 Plan mode 真正逼你回答的问题

不管进不进 plan mode，开工前都要先回答几个问题：

1. 输出长什么样？（样板 xlsx / JSON schema / 报告模板）
2. 输入是什么形态？图片、名单、关键词，还是官网 URL？
3. 什么算命中，什么算相邻，什么必须排除？
4. 是否需要 skill、MCP、subagent？
5. 哪些步骤可并行，哪些步骤必须人审？

这几个问题你自己心里有答案，AI 才能照着跑；否则 AI 会按它的"通用预设"填坑，跑出来不是你要的。

### 2.3 在 plan 里列扫描范围

我给 plan mode 加的一个习惯：**在 plan 里列出本次可见的能力扫描范围**。

示例：

```markdown
## 会用到的能力

扫描范围（本次会话可见）：
- Skills (N): name1, name2, name3, ...
- MCP namespaces (M): tavily, firecrawl, jina, chrome-devtools, ...
- Agent: exhibition-finder, exhibition-verifier, cross-batch-merger

会用：
- MCP firecrawl：抓 SPA 展会展商名单
- Skill review：跑完后审查边界错误
- Agent exhibition-verifier：批量验证公司品类

考虑过但不用：
- browser-use：内嵌 LLM 决策、与 evidence_quote 追溯链冲突，被 chrome-devtools 覆盖
```

这样做的好处是：如果某个 skill 或 MCP 没出现在扫描范围里，就知道不是 AI 忘了用，而是本次会话根本没注入。

---

## 3. Agentic Workflow：把调研拆成流水线，而不是一次性问答

**要点**：5 层模型 + 8 步流水线把 LLM、Python、人审分工清楚；研发岗项目就靠 3 个 agent + 1 个 command 跑完所有岗位。

### 3.1 5 层模型 + 8 步流水线

我现在更倾向于把竞企调研拆成 5 层：

```text
L4 交付层：xlsx / 报告 / README
L3 人审层：decisions/*.tsv、manual review、override
L2 语义层：Claude 判断品类、产品、行业、证据
L1 确定层：Python 抓取、归一化、URL 活检、去重、写表
L0 原料层：图片、PDF、网页缓存、临时文件
```

对应到流水线就是：

| 步骤 | 做什么 | 产物 |
|---|---|---|
| 1 | 种子采集 | 展会、图片、公司名单、关键词 |
| 2 | 公司名归一和去重 | candidates.jsonl |
| 3 | URL 活检 | live / dead / parking / blocked |
| 4 | 语义验证 | verified.jsonl / findings.json |
| 5 | 规则匹配 | matched / unmatched |
| 6 | 人审 override | decisions.tsv |
| 7 | 出 xlsx | result.xlsx |
| 8 | review / diff | 修正报告、版本差异 |

这里有两条原则很重要：

- **Python 不调 LLM SDK**：Python 只做确定性工作，语义判断交给 Claude 主会话或 subagent。
- **blocked 不等于 unmatched**：打不开、反爬、PDF 解析失败，是 blocked；业务不匹配才是 unmatched。

### 3.2 研发岗项目的三个 Agent

研发岗项目实际落地的就是 `.claude/agents/` 下的三个 subagent，每个都对应流水线的一段：

| Agent | 触发步骤 | 输入 | 输出 | 干啥 |
|---|---|---|---|---|
| **exhibition-finder** | Step 1 | `position_id` | `exhibitions.csv`（15 列，含 density/priority/exhibitor_list_access） | 给岗位反搜展会，用 MCP search 跑 query 模板，估算密度优先级 |
| **exhibition-verifier** | Step 5 | 公司清单 + 岗位 ID + 展会 tag | `verify_batch<N>.jsonl` | 品牌简称搜索 → 找官网 → url_check → 评估品类命中 → 写 JSONL |
| **cross-batch-merger** | Step 6 | 多份 verify_batch.jsonl + 已有 verified.jsonl | `merge_suspects.tsv`（5 列 + 用户审第 6 列） | rapidfuzz 跨批 fuzzy dedup，每对给 MERGE/LINK/SEPARATE 建议 |

每个 agent 都有：

- **frontmatter 里写 description + tools + model + trace_token**：description 决定我会不会主动选它；trace_token 写进每次输出，主会话校验版本不漂移。
- **Phase 0 必读清单**：包括 `CLAUDE.md`、`docs/conventions.md`、`config/categories.yaml`，避免主会话每次重复在 prompt 里复制规则。
- **Final message 5 行总结**：主会话只读 summary 不读 transcript，token 占用极小。

### 3.3 三个 agent 协作的关键设计

**1. orchestrator 是 markdown，不是程序**

`.claude/commands/run-position.md` 整份文档就是写给我看的 8 步剧本。slash command 没有解释器、没有 runtime——我读完按剧本走。

**2. 闸门 = prompt 约束**

剧本里写"等用户回 'ok' 才能继续"，是纯 prompt 约束，不是程序里的 `input()`。这让每步都可中断、可 `--redo stepN`、能跨会话续跑。

**3. 重活塞进 subagent 隔离上下文**

一个 verifier 要消化 30 家公司的 web 搜索 + 官网验证结果。如果在主会话跑，几个 batch 下来 token 就爆了。子 agent 跑完只返回 5 行 summary，主会话 token 几乎不变。

**4. 决策落 TSV 让用户人审**

merge_suspects.tsv 第 6 列留空给用户填决策（MERGE/LINK/SEPARATE/SKIP）。这种判断 LLM 不靠谱，留人审，决策落盘可复现。

**5. Python 做确定性逻辑**

凡是"算分 / 写 xlsx / 合并 jsonl"这种**不需要判断**的事，全交 Python：

```bash
python -m pipeline.merge_batches --position P0X
python -m pipeline.apply_decisions --position P0X
python -m pipeline.assign --position P0X
python -m pipeline.build_xlsx --position P0X
```

评分公式 / xlsx 格式不能漂移，必须可测，所以用 Python 而不是让 LLM 算。

---

<!-- PART III · 工程化与纠偏 -->

## 4. AI 搜索 MCP：装上 Tavily / Jina / Firecrawl 之后

**要点**：裸 WebFetch 抓不动 SPA；Tavily 找官网 + Jina Reader 拿干净 Markdown + Firecrawl 整站爬，三件套覆盖 95% 场景。

### 4.1 为什么 WebFetch 不够

很多企业官网现在是 SPA：服务器只返回一个空壳 HTML，真正内容靠浏览器执行 JavaScript 后才出现。

Claude Code 内置 `WebFetch` 本质上是一次性 HTTP GET + HTML 解析（类似 `curl` + 文本提取），**不执行 JavaScript**。所以裸 WebFetch 经常只能抓到：

```html
<div id="root"></div>
<script src="bundle.js"></script>
```

这对竞企调研很致命，因为产品列表、型号、应用行业都可能是 JS 渲染出来的。

### 4.2 三个 MCP 的定位

我后来给项目装了三个 MCP，定位互补：

| 工具 | 本质 | 处理 SPA | 免费额度 | 项目里干啥 |
|---|---|---|---|---|
| **Tavily** | AI 优化的搜索 API，替代 WebSearch | ❌（本身是搜索） | 1000 次/月 | "找官网" / 找展商目录源头 |
| **Jina Reader** | URL → LLM-friendly Markdown | ✅（后端渲染） | 1M tokens（带 key 2000 RPM） | SPA 站抓正文、绕反爬、产品页拿干净 Markdown |
| **Firecrawl** | 托管爬虫（Playwright 集群） | ✅（后端 Chromium） | 500 credits/月 | 整站 map/crawl/extract、批量抓产品页、SPA waitFor |

接入方式：

- **Tavily** / **Firecrawl** 走本地 stdio（`npx`）
- **Jina** 用官方 Remote HTTP（`https://mcp.jina.ai/v1`），无需安装

三家都 GitHub/Google 一键登录拿 key，无需绑卡。`.mcp.json` 用 `${VAR}` 引环境变量，可放心 commit；`.env` 走 `.gitignore`。

### 4.3 装 MCP 前 vs 装 MCP 后（实测对比）

**研发岗项目 P01 redo 对比**：

| 维度 | v1（无 MCP） | v2（Tavily + Jina + Firecrawl） |
|---|---|---|
| 候选展会 | 17 | **23**（+PW09 沈阳变压器展含天然酯油 / +6 国内外展） |
| 展商抓取 | EP Shanghai 5 家样本 / 旧版 EPOWER 5 家 | **EPOWER 230 家 N3-N4 馆完整名单**（Firecrawl GBK 编码）+ xfmr.show 30 家 + CEEIA 议程 25 家 |
| verified 池 | 69 | **235**（×3.4） |
| 命中 | 7 | **8**（+Nynas / +Raj Petro，−壳牌中国误命中） |
| 关键改善 | — | Firecrawl 拿到 EPOWER 完整名单；Jina 绕过 jina.ai 422 + TLS 错误；Tavily 找到 PW09 沈阳展（v1 完全没发现） |

**研发岗项目 P02 redo 对比**：

| 维度 | v1 | v2 |
|---|---|---|
| 展会来源 | HT02 巨浪 1 个 | HT01 北京专展 OCR（v1 漏抓） + HT02 + HT08/HT09 sample |
| HT01 抓取 | 标"TLS 错 + SPA 框架，不可抓"，跳过 | **5 张 jpg vision OCR 拿到 162 家**（高分辨率 2022×3027） |
| verified | 298 | 284 |
| 命中 | 11 | **15**（+ALD HTS / +湖北天舒 / +天津/江苏丰东热处理设备） |
| 命中率 | 3.7% | **5.3%** |

P02 HT01 是 MCP 改善最大的案例：v1 finder 看到第一次 TLS 错就放弃 + JS 动画框架（ScrollReveal/AOS）误判为 SPA → 标"不可抓"。v2 加了"页面框架 vs 名单载体"二维探查后，发现实际是 **SSR HTML 外壳 + 图片扫描件载体**（CDN 图床 file2.123hl.cn），用 `curl -k` + Read 多模态 vision OCR 一举拿到 162 家展商。

涂料项目的 SPA 大厂站点（陶氏、巴斯夫、PPG）则是 Jina Reader 替代裸 WebFetch 的甜区——产品详情页（Hybris / AEM / Sitecore SPA 前端）以前 fetch 回来一半是空的，换 `r.jina.ai/<url>` 前缀立刻能拿正文 + 表格，token 还省一大半。

### 4.4 推荐退路链

从轻到重：

```text
1. 普通 WebFetch / curl                       ← 90% 静态站
2. Jina Reader：r.jina.ai/<url> → Markdown    ← SPA + 反爬 + 干净输出
3. Tavily search                              ← 找官网 / 找展商目录入口
4. 找 SPA 的真实 API：chrome-devtools 看 network
5. Firecrawl：批量 map / crawl / extract      ← 整站 / 批量
6. chrome-devtools / browser-use：真实浏览器交互
7. 仍失败：blocked，进入人工决策
```

### 4.5 工具准入第一原则（踩过 browser-use 的坑后定的）

**任何在抓数据 / 找证据链路里"自己有自主决策权"的工具，都要先怀疑它能不能塞进 evidence_quote 追溯链；过不了这关，技术再强也不进 `.mcp.json`**。

browser-use 是反例：`retry_with_browser_use_agent` 把决策权让出去，回来只有结论没有路径，对 `evidence_quote / source_url / is_inferred` 三层硬规则就是结构性冲突。性能问题可以升 plan 修，audit 黑盒是设计决定的——补一层校验等于干两遍活，那直接用 chrome-devtools 就行。

Tavily / Jina / Firecrawl 都是"透明手术刀"：你让它干啥它干啥，过程数据全留下，可以反查。

### 4.6 对两个项目的适配

| 项目 | 更适合的工具 | 原因 |
|---|---|---|
| 研发岗竞企图谱 | Jina Reader + Tavily（主力）+ Firecrawl（SPA 站偶尔用） | 多是找官网和验证品类，不需要深爬整站 |
| 涂料产品调研 | Jina Reader + Firecrawl（map/extract）+ chrome-devtools | 要深入产品详情页、产品树、SPA、PDF、图片 OCR |

两个项目都可以优先把 `r.jina.ai/` 作为 WebFetch 失败后的第一退路；涂料项目遇到大厂站点优先评估 Firecrawl 的 map/extract，减少 subagent 逐页点菜单的 token 消耗；chrome-devtools 保留给复杂交互、登录、弹窗、地区选择、网络请求分析。

---

## 5. Review：AI 调研最容易错在"口径"和"边界"

**要点**：第一次跑出来的结果一定有口径问题；review 是必经环节，要看历史知识、URL 硬规则、设备厂误命中、OCR 拆 SKU 这四类。

Review 不是可选项。两个项目里最有价值的修正都来自 review。

### 5.1 P01 的 ABB / 日立能源案例（过时行业知识）

最开始 ABB 被判为植物基绝缘油命中，因为历史上 ABB 有 BIOTEMP 技术线。但 review 后发现：

- ABB Power Grids 业务已转给 Hitachi Energy；
- 合肥 ABB 变压器公司股权也已变更为日立能源；
- 当前更应该命中的是日立能源，而不是 ABB (CHINA) LTD.。

这暴露了两个问题：

1. AI 会把过时行业知识当当前事实；
2. 如果 `product_text` 里出现"植物基绝缘油"等关键词，会被 assign 规则误命中。

修正方式：ABB 改 unmatched / 日立能源改 matched / evidence_sources 补真实可验 URL / 重跑 assign 和 build_xlsx。

### 5.2 P02 的 TAV 502 案例（URL 硬规则）

原规则把 HTTP 502 也视为 live，因为"站点存在"。但用户实际打开不了，HR 交付场景里这不该算可用官网。

后来规则收紧为：**只有 HTTP 200 才算 live，其它 3xx/4xx/5xx 都按 dead 或待人工 override。**

这说明：技术上"域名存在"和业务上"可作为证据链接"不是一回事。

### 5.3 P07 的设备厂 / 服务商误命中（字面子串）

review P07 时发现 22 家 P07 命中里至少 4 家是误命中：

- **深圳明锐理想**：主营 AOI/SPI/AXI 光学**检测设备**，product_text 写"锡膏**检查机** SPI"，被 `_hit("锡膏", ...)` 字面子串命中——其实是检测锡膏的设备厂。
- **深圳市凯泰新能源**：国际品牌**分销贸易商**，不是研发企业。
- **云锡新材料 / 田村化研东莞**：没有独立官网（链向集团母站），按 CLAUDE.md 决策 4「无可用官网 → unmatched」应该降级。

修正路径：

1. 写 `decisions/p07_force_unmatched.tsv`，把这些手动 force_unmatched + 标 reason_tag（equipment_vendor / service_provider / no_independent_url / adjacent_category）。
2. 修复 `pipeline/assign.py::_hit_positive` 的 2 个 false-negative bug：多次出现取最早 negative + 单字 neg_word（"无"/"非"/"不"）误触负向。
3. 顺手对 7 个岗位全部 audit 一遍，最终：P01 -1 / P02 0 / P03 0 / P04 0 / P05 +2 / P06 -2 / P07 -1。

核心经验是：**工具能力上线之后，必须同步补流程规则。否则 AI 能抓到信息，但会用错。**

### 5.4 传美讯 OCR 案例（涂料项目）

第一次传美讯只得到 80 行，review 后发现：

- ECOTANK 一张图里有 18 个型号，不能只取第一个；
- 墨水瓶、包装盒不是涂料行业产品，要剔除；
- OCR 479 张图太慢，应该先 triage 再 crop。

修正后传美讯从 80 行变成 168 行；OCR 时间从约 34 分钟降到约 7.5 分钟；多 code 拆行规则写回 SKILL.md。

---

<!-- PART IV · 成果与教训 -->

## 6. 最终调研结果

**要点**：研发岗 7 个跑通，215 家命中合并成 248 公司 × 213 明细的 summary.xlsx；涂料 104 家 9086 SKU × 53 列。

### 6.1 研发岗竞企图谱（7 岗位）

**`positions/P0X/result.xlsx` × 7 + 合并 `summary.xlsx`**

每个岗位单独的 result.xlsx 都有「汇总 + 明细」双 sheet，明细 20 列（对齐手工 v1 格式）：包含官网（hyperlink）、主品类、命中岗位、来源展会、参展依据 / evidence_quote、vision_confidence、url_confidence、公司类型、相关品类、产品业务判断、命中理由、命中评分、命中关键词、官网复核结论、处理建议、官网复核依据等。

经 P07 review + bug fix + 全岗位 audit 后的最终命中数：

| 岗位 | 命中 |
|---|---|
| P01 植物基绝缘油 | 8 |
| P02 热处理工艺 | 15 |
| P03 三防漆 | 20 |
| P04 特种润滑脂 | 28 |
| P05 矿产浮选药剂 | 54 |
| P06 灌封胶 / 导热胶 | 69 |
| P07 助焊剂 | 21 |
| **合计** | **215** |

`summary.xlsx` 把 7 个 `result.xlsx` + 旧 `results_photo.xlsx` 合并：

- **汇总 sheet**：7 行岗位 × 横向公司，每个 cell 都有真实 hyperlink + 蓝下划线字体，冻结 A2 → **248 公司行**。
- **明细 sheet**：213 行 × 20 列，官网列 hyperlink 完整。跨岗位的公司"命中岗位"列已合并（如"上海拜高高分子"同时命中三防漆 + 润滑脂 + 灌封胶）。

去重设计：策略 A（URL 优先 + 名字兜底），处理同公司不同写法（3M 中国 / WEVO-CHEMIE / 壳牌中国 / Dow Inc. ↔ 陶氏化学 / 汉高 ↔ Henkel）。脚本固化在 `pipeline/build_summary.py`，重跑用 `python3 -m pipeline.build_summary`，alias 表（`COMPANY_ALIASES`）以后再遇到新写法直接加一条。

### 6.2 涂料行业产品调研

**`涂料同行调研_绿色筛选.xlsx`**（位于 `E:\coatings_industry_competitor_research\save_files_green\`）

- **同行产品 sheet**：**9086 行** SKU × **53 列**
    - 基础列：序号、公司名、大类、细类、产品、产品网址（hyperlink）
    - **47 个行业列**（√ 标记）：包装印刷 / 电子电器 / 建筑 / 汽车 / 3C 电子 / 风电防腐 / 纺织印染 / 造纸 / 锂电池 / 家具 / 木器 / 钢结构 / 地坪 / 船舶防腐 等
- **调研概况 sheet**：**104 家**公司 × 6 列（序号 / 公司名 / 网址 / 状态 / 备注）

典型公司产品规模（备注列保留了关键 audit 信息）：

| 公司 | 产出 SKU | 关键路径 |
|---|---|---|
| 洋紫荆油墨（中山） | 748 行 | PDF 手册重做（旧 96 → 新 748），4 大类（包装 558 + 装饰 99 + 电子 85 + 环保 6）|
| 传美讯 | 168 行 | 凡科站型号藏图片里，OCR 拆 SKU（80 → 168）|
| 长兴光电 | 420 SKU | Cloudflare 站，`curl_cffi` 过反爬 |
| 珠海东洋色材 | 110 行 | 4 大类，portfolio-item 静态详情，SKU 在图里 → Read OCR |
| 广东欧利雅 | 85 行 | JZ 模板 h-pr--0_578_X.html 真详情（h-pd 是空壳） |
| 巴德士 | BLOCKED | 站点 curl/WebFetch 全部超时（沙箱出口屏蔽），待网络恢复后单跑 |

### 6.3 交付要点

不管哪个项目，最终交付给 HR 的 xlsx 都遵守以下硬规则：

| 规则 | 具体 |
|---|---|
| **官网必活** | `url_check_status == live`（仅 HTTP 200）。dead/parking → unmatched 或 url_overrides.tsv 手动 patch |
| **超链接到官网** | `=HYPERLINK("url","公司全称")` 公式法。cell 显示公司全称，点击跳官网 |
| **证据可追溯** | 每条命中带 `evidence_quote`（展商目录原文）+ `source_url`。HR 可打开 URL 用 Ctrl+F 复核 |
| **品类严格边界** | 邻近业务（如絮凝剂/消泡剂不命中 P05、检测设备厂不命中 P07）走 `force_unmatched.tsv` 显式声明 |
| **blocked ≠ unmatched** | 打不开是 blocked（technical），业务不匹配才是 unmatched（business） |

---

## 7. 局限性与下次改进

**要点**：核心断层在"verifier 写人类语义 ↔ pipeline 做字面匹配"；下次让 verifier 出枚举决策、不让 pipeline 猜 prose，80% 的局限自动消失。

跑完两个项目，回头看有 7 条"开局就该想到"的事情。本质上都指向同一个根因：**verifier 写的"人类语义"和 pipeline 做的"字面匹配"之间存在断层**——从 day 1 把这两件事隔离开，后面 80% 的局限都不会出现。

### 7.1 结构化字段 vs 人类 prose 的边界，开局就要画好

最大的坑：`product_text` 同时给 HR 看（含"无 X 业务" / "非 X 厂" / "应用于 Y" 等叙述性句式）也给 pipeline 看（字面子串匹配）。两个目的对 prose 的要求是冲突的。

**下次**：verifier 输出的 JSON 里 prose 字段（给人看的）不参与评分。打分只看 `main_categories[]`、`related_categories[]`、`negative_categories[]`、`company_type` 这些枚举字段。prose 是 audit trail，不是 truth source。

### 7.2 中文做子串匹配前，先列"歧义清单"

`锡膏` ⊂ `锡膏检查机`、`助焊剂` ⊂ `助焊剂清洗剂`、`硅脂` ⊂ `导热硅脂`、`焊膏` ⊂ `烧结银焊膏`、`无` ⊂ `无铅` / `无尘` / `无关`、`非` ⊂ `非晶`。这些不是边角，是核心评分准确性的来源。

**下次**：`categories.yaml` 定稿前过一遍"这些词作为子串出现在 reject 类目里多常见"。每条 main_category 列 3-5 个"看上去像但其实不是"的对照词，写进 `negative_keywords`。assign 跑通前先用 50 条人工标注样本验证。

### 7.3 company_type 这种硬标签，放在评分循环之前

`company_type ∈ {设备厂, 贸易商, 服务商, 矿山客户}` 是个 strong prior，但 `assign.py` 只在 score=0 时才拿去归类。结果"锡膏检查机"这种字面命中分数 1.0 的设备厂被归到 matched。

**下次**：assign 第一行就先看硬标签。决策顺序应该是 **force_unmatched > company_type 黑名单 > URL 硬规则 > 评分**，不是反过来。

### 7.4 否定语义不要交给 regex

`无 X 业务` / `代理 X` / `非 X 厂` / `主营并非 X` / `X 应用于 Y` / `不生产 X` / `X 不生产 Y`——我们花两轮把这些 idiom 一个个塞进 `neg_pattern`，但 `X 不生产 Y` 当 X、Y 都在窗口内的语义级 case，regex 怎么调都解决不了（WEVO 例）。

**下次**：与其在 regex 里堆 alias，不如让 verifier 直接写 `excluded_for_position: [P07]` 这种显式字段。"我看了官网，结论是不命中 P07"比"我写一段 prose 让 pipeline 猜结论"靠谱得多。

### 7.5 Day-1 就建 audit-diff 工具

这次是为了对比 bug-fix 前后才临时写的 `/tmp/audit_diff.py`。如果一开始就有这工具，每次跑完 assign 就 print "+N -M 岗位变化 K"，bug 和 regression 当天就能发现，不会等到月后人审才暴露。

**下次**：`pipeline/assign.py` 自带 `--diff <previous>` 模式，输出新增 / 失配 / 岗位变化清单。每次重跑前自动 backup 旧 `assignments.jsonl`。

### 7.6 守住"deterministic pipeline → xlsx"的链路

最坑的是发现：原 P07 result.xlsx 的 22 家公司不能从当前 verified.jsonl 跑 assign 复现。中间有手动 patch 但没记入 `decisions/`。结果每次重跑都跟"原版"对不上，要逆推哪里 diverge 了。

**下次**：xlsx 必须是 `assign → build_xlsx` 的纯函数输出。所有人审决策都进 `decisions/*.tsv`，不允许直接编辑 xlsx 或 `assignments.jsonl`。build 之前自动 `git status` 检查 verified / assignments 干净；不干净就拒绝 build。

### 7.7 命中阈值 0.6 是个 magic number，要有 sanity check

`SCORE_THRESHOLD = 0.6` + 各项权重（1.0 / 0.6 / 0.3 / 0.2 / -0.5）是拍脑袋的。本次出现的"分数刚好 0.3"失配（卡夫特、翰华锡业一度）就是阈值附近的灰带。

**下次**：预先准备约 30 条人工 ground truth（10 强匹配 + 10 弱匹配 + 10 边缘 unmatched），跑出来看 precision / recall。阈值 + 权重根据这组样本调，不是凭感觉。

---

<!-- PART V · 沉淀与收尾 -->

## 8. 最后沉淀成一套 SOP

**要点**：六步法——定契约 → 建骨架 → 跑样本 → 写规则 → 上并发 → 跑 review。

如果以后从零搭一个新的竞企调研项目，我会按这个顺序做：

### 第一步：先定业务契约

- 样板 xlsx；
- 列名、列宽、超链接格式；
- 命中 / 相邻 / 排除规则；
- evidence_url 和 evidence_quote 是否必填；
- unmatched reason_tag 枚举。

### 第二步：建立项目骨架

```text
CLAUDE.md
.claude/agents/
.claude/commands/
.claude/skills/<workflow>/SKILL.md   ← 仅当真有"姿态切换"需求
.mcp.json + .env.example             ← MCP 配置
config/*.yaml
pipeline/*.py
positions/ 或 findings/
decisions/*.tsv
docs/workflow.md + docs/conventions.md
```

### 第三步：先用 alpha 模式跑 3-5 个样本

不要一上来就并发 100 家。先让主会话完整跑几个样本，把边界问题暴露出来。

### 第四步：把踩坑写进 SKILL.md / agent prompt

例如：

- 包装/容器不算涂料产品；
- 一张图多个型号要拆多行；
- 树脂类产品不标行业列；
- HTTP 200 才算 live；
- blocked 和 unmatched 分开；
- 页面框架 vs 名单载体（v1 HT01 漏抓 162 家的教训）。

### 第五步：再上 subagent 并发

并发前必须有：输入清单 / 输出 JSON schema / skill_trace / blocker 处理策略 / 主会话不读 transcript 只读结果文件。

### 第六步：跑 review 和 diff

每次出表后至少做三类检查：

1. 命中公司是否真的对岗位有挖人意义；
2. unmatched 是否有误杀；
3. 证据链接是否能打开，是否支持当前判断。

发现规则 bug（比如 P07 review 中暴露的 `_hit_positive` 两个 false-negative）→ 修 pipeline + 加单测 + 全岗位 audit 一遍差异。

---

## 9. 可以用作分享收尾的一句话

**Claude Code 真正有价值的地方，不是一次回答得多聪明，而是它可以被工程化：把人的判断口径写进 CLAUDE.md，把重复流程写进 agent / SKILL.md，把重活派给 subagent，把确定性步骤交给 Python，把搜索 / 抓取交给透明可追溯的 MCP（Tavily / Jina / Firecrawl），把争议留给 review。这样，竞企调研才从一次性搜索，变成可复用的组织能力。**
