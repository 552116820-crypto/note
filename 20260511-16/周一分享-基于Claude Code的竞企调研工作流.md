# 周一分享：搭建基于 Claude Code 的竞企调研工作流

> 分享目标：不是介绍“AI 很强”，而是复盘两个真实项目里，如何把 Claude Code 从聊天工具用成一套可复用、可并行、可审查的竞企/产品调研流水线。

## 0. 先用 5 分钟讲清楚结论

这次分享可以从一句话开始：

**竞企调研的难点不在“让 AI 搜资料”，而在把不确定的网页、图片、PDF、公司边界、品类边界，变成一套能反复跑、能追溯、能人审、最后能交付 Excel 的工程流程。**

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

---

## 1. Skill 与 Agent：一个是流程说明书，一个是干活的人

### 1.1 三个概念要分清

| 类型 | 放在哪里 | 作用 | 是否开新上下文 | 适合做什么 |
|---|---|---|---|---|
| Agent / subagent | `.claude/agents/*.md` | 定义一个子代理 | 是 | 长任务、并行任务、隔离上下文 |
| Slash command | `.claude/commands/*.md` | 一键注入主流程 prompt | 否 | `/run-position P01` 这种总控入口 |
| Skill | `.claude/skills/*/SKILL.md` 或全局 skill | 可复用流程说明 | 通常否 | 单家公司调研、某类网页处理 SOP |

我的理解：

- **Agent 是执行单元**：可以派出去跑一家公司、一批展商、一次 review。
- **Skill 是流程资产**：告诉 agent 按哪些 phase 做，遇到边界怎么处理，输出 schema 是什么。
- **Command 是总入口**：让主会话按固定剧本调度 agent 和 Python 脚本。

### 1.2 在两个项目里的实际差别

研发岗竞企图谱一开始更像主会话顺序跑：找展会、抓展商、验证公司、写 xlsx。后来才抽象出 `/run-position P0X` 和几个 subagent。

涂料产品调研则更适合 skill：每家公司都要做类似的事，读官网、找产品树、拆型号、判断行业列、输出 JSON。所以沉淀成了 `research-one-company/SKILL.md`，让每个 subagent 先 Read 这个文件，再按 Phase 0 到 Phase 9 执行。

一个关键教训是：**项目级 skill 不一定会被 subagent 自动继承。** 实测 subagent 可以 Read 项目文件，但不一定能直接通过 Skill 工具调用项目 skill。所以最终采用的稳妥方案是：

```text
subagent 开工前先 Read：
- CLAUDE.md
- .claude/skills/research-one-company/SKILL.md
- 关键 reference / METHOD.md
```

这比在 prompt 里复制 200 行规则更稳，也避免规则双写。

---

## 2. Harness：为什么同一套流程换到底座会变差

Harness 可以理解为 Claude Code 这个运行时本体：它负责加载 CLAUDE.md、识别 `.claude/agents`、注册 slash command、管理工具、调度 subagent、接 MCP。

这里要区分两层：

| 层 | 例子 | 属于谁 |
|---|---|---|
| 机制层 | Claude Code 怎么扫描 agents/commands/skills，怎么调工具 | harness |
| 内容层 | 我们写的 agent prompt、SKILL.md、CLAUDE.md | 项目资产 |

为什么这套流程给 Codex 或其它工具直接跑效果会差？主要不是模型单点能力，而是整个 workflow 依赖了 Claude Code harness 的几个特性：

1. **subagent + 后台并行 + 通知回收**：涂料项目里可以每家公司一个 subagent，主会话只收 JSON，不读 transcript。
2. **多模态 Read 当 OCR 用**：传美讯这种型号在图片里，直接 Read 图比传统 HTML 抓取强很多。
3. **长流程指令跟随**：SKILL.md 要求 Phase 0 到 Phase 9 严格执行，还要回传 `skill_trace`。
4. **CLAUDE.md / memory / hooks 自动注入**：很多踩坑经验不是临时 prompt，而是项目常驻上下文。
5. **MCP 工具矩阵**：chrome-devtools、browser-use、WebFetch、Read 的工具名和使用方式都写进了流程。

所以这套东西不是“换个模型继续跑”，而是**基于 Claude Code 工具矩阵和运行时能力组装出来的调研操作系统**。

---

## 3. Plan Mode：先锁边界，再动手

Plan mode 最大价值不是“让 AI 写计划”，而是逼自己在动手前回答几个问题：

1. 输出长什么样？
2. 输入是什么形态？图片、名单、关键词，还是官网 URL？
3. 什么算命中，什么算相邻，什么必须排除？
4. 是否需要 skill、MCP、subagent？
5. 哪些步骤可并行，哪些步骤必须人审？

我后来给 plan mode 加了一个习惯：**在 plan 里列出本次可见的能力扫描范围**。

示例：

```markdown
## 会用到的能力

扫描范围：
- Skills: review, simplify, schedule, ...
- MCP: chrome-devtools, browser-use, playwright

会用：
- chrome-devtools：处理 SPA 渲染和网络请求
- review：跑完后审查边界错误

考虑过但不用：
- playwright：本次没有 UI 自动化测试
```

这样做的好处是：如果某个 skill 或 MCP 没出现在扫描范围里，就知道不是 AI 忘了用，而是本次会话根本没注入。

---

## 4. Agentic Workflow：把调研拆成流水线，而不是一次性问答

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

---

## 5. 并发与 Subagent：吞吐提升，但必须有契约

并发不是简单地“多开几个 AI”。如果没有返回契约，主会话会被几十个 transcript 淹没。

涂料项目里更稳定的做法是：

```json
{
  "company_short": "chuanmeixun",
  "status": "ok",
  "findings_path": "findings/batch/company.json",
  "product_count": 168,
  "blockers": [],
  "skill_trace": "SKILL-RESEARCH-ONE-COMPANY-v11"
}
```

主会话只关心：

- 文件写到哪里；
- 是否完成；
- 多少产品；
- 有无 blocker；
- skill_trace 是否匹配。

### 5.1 并发的收益

传美讯和长兴光电的实战里，两家公司可以同时跑：

- 长兴光电：Cloudflare 站点，用 `curl_cffi` 过反爬，得到 420 SKU。
- 传美讯：凡科站点，型号藏在图片里，先 OCR 得到 80 行，review 后修正到 168 行。

并发让主会话可以一边等 subagent，一边准备 build xlsx 脚本。

### 5.2 并发的风险

1. **工具配额冲突**：多个会话同时 WebSearch/WebFetch/subagent，会撞 rate limit。
2. **共享文件冲突**：不同岗位目录可以并行，但共同索引文件如 `docs/positions-index.md` 可能冲突。
3. **规则版本漂移**：subagent 没读到新版 SKILL.md，旧规则继续跑。

所以必须加 `skill_trace`，主会话发现不匹配就重派或停止。

---

## 6. Review：AI 调研最容易错在“口径”和“边界”

Review 不是可选项。两个项目里最有价值的修正都来自 review。

### 6.1 P01 的 ABB / 日立能源案例

最开始 ABB 被判为植物基绝缘油命中，因为历史上 ABB 有 BIOTEMP 技术线。但 review 后发现：

- ABB Power Grids 业务已转给 Hitachi Energy；
- 合肥 ABB 变压器公司股权也已变更为日立能源；
- 当前更应该命中的是日立能源，而不是 ABB (CHINA) LTD.。

这暴露了两个问题：

1. AI 会把过时行业知识当当前事实；
2. 如果 `product_text` 里出现“植物基绝缘油”等关键词，会被 assign 规则误命中。

修正方式：

- ABB 改为 unmatched / adjacent；
- 日立能源改为 matched；
- evidence_sources 补真实可验 URL；
- 重跑 assign 和 build_xlsx。

### 6.2 P02 的 TAV 502 案例

原规则把 HTTP 502 也视为 live，因为“站点存在”。但用户实际打开不了，HR 交付场景里这不该算可用官网。

后来规则收紧为：**只有 HTTP 200 才算 live，其它 3xx/4xx/5xx 都按 dead 或待人工 override。**

这说明：技术上“域名存在”和业务上“可作为证据链接”不是一回事。

### 6.3 传美讯 OCR 案例

第一次传美讯只得到 80 行，review 后发现：

- ECOTANK 一张图里有 18 个型号，不能只取第一个；
- 墨水瓶、包装盒不是涂料行业产品，要剔除；
- OCR 479 张图太慢，应该先 triage 再 crop。

修正后：

- 传美讯从 80 行变成 168 行；
- 包装/容器产品全部剔除；
- OCR 时间从约 34 分钟降到约 7.5 分钟；
- 多 code 拆行规则写回 SKILL.md。

这里的核心经验是：**工具能力上线之后，必须同步补流程规则。否则 AI 能抓到信息，但会用错。**

---

## 7. JS 渲染与 AI 搜索 MCP：WebFetch 不够，退路链要提前设计

很多企业官网现在是 SPA：服务器只返回一个空壳 HTML，真正内容靠浏览器执行 JavaScript 后才出现。

所以裸 WebFetch 经常只能抓到：

```html
<div id="root"></div>
<script src="bundle.js"></script>
```

这对竞企调研很致命，因为产品列表、型号、应用行业都可能是 JS 渲染出来的。

### 7.1 推荐退路链

从轻到重：

```text
1. 普通 WebFetch / curl
2. Jina Reader：r.jina.ai/<url>，把页面转 Markdown
3. 找 SPA 的真实 API：用 chrome-devtools 看 network
4. Firecrawl：批量 map / crawl / extract
5. chrome-devtools / browser-use：真实浏览器交互
6. 仍失败：blocked，进入人工决策
```

### 7.2 对两个项目的适配

| 项目 | 更适合的工具 | 原因 |
|---|---|---|
| 研发岗竞企图谱 | Jina Reader + Tavily 可选 | 多是找官网和验证品类，不需要深爬整站 |
| 涂料产品调研 | Jina Reader + Firecrawl + chrome-devtools | 要深入产品详情页、产品树、SPA、PDF、图片 OCR |

我的建议是：

- 两个项目都可以优先把 `r.jina.ai/` 作为 WebFetch 失败后的第一退路。
- 涂料项目如果遇到大厂站点，可以评估 Firecrawl 的 map/extract，减少 subagent 逐页点菜单的 token 消耗。
- chrome-devtools 和 browser-use 保留给复杂交互、登录、弹窗、地区选择、网络请求分析。

---

## 8. 最后沉淀成一套 SOP

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
.claude/skills/<workflow>/SKILL.md
config/*.yaml
pipeline/*.py
positions/ 或 findings/
decisions/*.tsv
docs/workflow.md
```

### 第三步：先用 alpha 模式跑 3-5 个样本

不要一上来就并发 100 家。先让主会话完整跑几个样本，把边界问题暴露出来。

### 第四步：把踩坑写进 SKILL.md

例如：

- 包装/容器不算涂料产品；
- 一张图多个型号要拆多行；
- 树脂类产品不标行业列；
- HTTP 200 才算 live；
- blocked 和 unmatched 分开。

### 第五步：再上 subagent 并发

并发前必须有：

- 输入清单；
- 输出 JSON schema；
- skill_trace；
- blocker 处理策略；
- 主会话不读 transcript，只读结果文件。

### 第六步：跑 review 和 diff

每次出表后至少做三类检查：

1. 命中公司是否真的对岗位有挖人意义；
2. unmatched 是否有误杀；
3. 证据链接是否能打开，是否支持当前判断。

---

## 9. 可以用作分享收尾的一句话

**Claude Code 真正有价值的地方，不是一次回答得多聪明，而是它可以被工程化：把人的判断口径写进 CLAUDE.md，把重复流程写进 SKILL.md，把重活派给 subagent，把确定性步骤交给 Python，把争议留给 review。这样，竞企调研才从一次性搜索，变成可复用的组织能力。**
