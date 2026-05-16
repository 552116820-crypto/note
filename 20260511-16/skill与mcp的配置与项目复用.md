
---

### Turn 15  *(2026-05-16 02:52)*

**User:**

所以browser-use这个mcp真的有用吗？

**Assistant:**

诚实说：**在你这个项目里几乎用不上**。我上一轮把它跟 chrome-devtools 并列列出来主要是为了"理论上覆盖全场景"，但回到你的真实痛点——它的独特价值很弱。

---

## 三条具体理由

### 1. 你真正需要它的场景几乎不存在

我列的"复杂多步交互（点筛选 → 翻页 → 抓 list）"听起来很合理，但你这项目的实际展会站基本是：
- 公开访问（无登录）→ 用不上"自然语言驱动"
- SPA 渲染 → Firecrawl `waitFor:8000` 已能解决
- 多页 → `firecrawl_crawl` 异步整站

真到了 Firecrawl 搞不定的极少数 case（如要登录），chrome-devtools 同样能搞定（fill 表单 + click），还更可控。

### 2. 它内部要再调一个 LLM，黑盒 + 成本不可控

`retry_with_browser_use_agent` 这个工具的工作方式是：把你的任务委托给一个独立的 LLM agent（通常另起一个会话），让它自己"看页面 → 决策 → 操作"。
- 它看到啥、决策依据啥、抓了哪段，你这边看不到——跟你工作流里"`evidence_quote` 来源可追溯"的硬规则冲突
- 背后的 LLM 调用成本不算 Claude 这边的 token，需要单独配 LLM key（OpenAI 之类），是另一笔账

### 3. 跟 chrome-devtools 高度重叠

`browser_navigate` / `browser_click` / `browser_type` / `browser_extract_content`——这些 chrome-devtools 都有等价工具，且 chrome-devtools 还多了 network panel、console、lighthouse 这些 browser-use 没有的"调试视角"。

留两个等价工具会让"该用哪个"的判断变复杂——尤其 agent 文件的工具列表越短越不容易迷糊。

---

## 建议：砍掉 docs 里的 browser-use 推荐，只留 chrome-devtools

把 `docs/mcp-tools.md` 那一节里：
- 删掉 browser-use 相关行（"复杂多步交互"那行 + 决策段第二条）
- 把 chrome-devtools 列得更明确（精确控制 + 网络/console 调试）

这样未来你或 agent 看 docs 不会被两个等价选项绊住。

要我改吗？大概只删 5-6 行。

---

### Turn 16  *(2026-05-16 02:56)*

**User:**

browser-use这个mcp内部还要调一个llm，给来源追溯有加了一层黑盒，在爬取数据这方面就根本没有价值

**Assistant:**

确认你的判断。直接改 `docs/mcp-tools.md` 砍掉 browser-use 那部分，只留 chrome-devtools。

改完。差异：
- 删了表里"或 browser-use"和 browser-use 那一行
- "复杂多步交互"那行改成**"先用 chrome-devtools 摸索，稳定了写 yaml 走 playwright"**——指向已有的 playwright 路径，避免诱惑用 browser-use
- 加一行 blockquote 简短说明 browser-use 为啥不推荐（黑盒 + 追溯冲突 + 能力重叠），让未来你或别人翻这文档时不用重新评估一遍
- 删了"chrome-devtools vs browser-use 决策"那段（没了对比对象）

顺便把这个判断存进 memory，避免我未来又复读"browser-use 也能干这个" 这种水分推荐。

发现 MEMORY 索引里 [MCP setup] 那行还写着 omnisearch（已被我们 2 轮前砍掉换成 tavily 了），顺便修正 + 加 browser-use 索引。

memory 内容也过时了（还写着 omnisearch）。一并修正。

3 个改动都已落地：

1. **`docs/mcp-tools.md`** — 砍 browser-use，只留 chrome-devtools；blockquote 说明为啥不要 browser-use
2. **MEMORY** — 修正 [MCP setup] 索引（omnisearch → tavily），加 [browser-use 不进爬取] 条目
3. **`project_mcp_setup.md`** — 内容更新，加一条"不要再建议聚合 MCP" 的硬规则

这下未来的 Claude 会话翻 memory 时，不会再无中生有推荐 browser-use 或 omnisearch 这种"理论好但实战有坑"的工具。

---

### Turn 17  *(2026-05-16 03:08)*

**User:**

browser-use内部还要调一个llm，给来源追溯有加了一层黑盒，在爬取数据，需要强依据验证这方面就根本没有价值，评价一下我说的

**Assistant:**

评价：**判断核心准确、且聚焦**——比我上一轮列的三条理由更精炼，直接戳到要害。具体说：

**1. "黑盒 vs 强依据"是真正的矛盾**

你抓住的是这个工具的**类型问题**，不是性能问题。性能（慢、贵、卡 quota）是可以改善的——升 plan 就行；但 audit 黑盒是设计决定的。`retry_with_browser_use_agent` 把决策权让出去，回来只有结论没有路径，对 evidence_quote / source_url / is_inferred 三层硬规则就是结构性冲突。

工程上没有任何"在 browser-use 后面补一层校验"的方案能修这个——补校验等于干两遍活，那直接用 chrome-devtools 就行了。

**2. 论断有 1 处可以更精确**

"内部要调一个 llm" 严格说指 **agent 模式**（`retry_with_browser_use_agent`）。底层的 `browser_navigate` / `browser_click` 那些工具理论上 Claude 可以一步步直接调，不强制走 agent loop。

但——一旦你 Claude 自己一步步调，就跟 chrome-devtools 完全等价了，用 browser-use 反而多一层抽象。所以**用 browser-use 而不用 chrome-devtools 的唯一卖点就是 agent loop**，而 agent loop 就是 LLM 黑盒。结论上殊途同归。

**3. 还有一条你没说出来但隐含的硬道理**

这种"看 audit 属性而不只是看技术能力"的判断方式，可以作为本项目评估所有未来 MCP / 工具的第一道筛子：

> **任何在抓数据/找证据链路里"自己有自主决策权"的工具，都要先怀疑它能不能塞进 evidence_quote 追溯链；过不了这关，技术再强也不进 .mcp.json。**

按这个标准，未来如果有人推荐"AI 全自动竞品分析 MCP"、"自适应公司图谱构建 MCP"之类听起来酷但内置 LLM 决策的玩意，应该一律先质疑追溯——而不是先看 demo。

要不要把这条原则落进 `docs/mcp-tools.md` 顶部，作为"工具准入第一原则"？

---

### Turn 18  *(2026-05-16 03:11)*

**User:**

要

**Assistant:**

加到 `docs/mcp-tools.md` 顶部，在"策略"和"决策树"之间。新章节叫**"## 工具准入第一原则"**，~15 行：

- blockquote 高亮原则本身
- 给透明手术刀 vs 黑盒的判断方法
- 正例（Tavily/Jina/Firecrawl）+ 反例（browser-use）
- 收尾一句："**先看这条，再看技术能力**——demo 多酷都不能跳过"

顺便把这条原则升进 memory（让未来对话评估新 MCP 时直接想起来，不用每次翻 docs）。

Edit 失败——文件被 Claude Code 加了自动 metadata（node_type / originSessionId），跟我刚写的不完全一致。Re-Read 后再 Edit。

直接 Write 重写整个文件（保留 Claude Code 自动加的 metadata 结构）。

落地完毕：

- `docs/mcp-tools.md` 顶部 → 准入原则成显眼章节
- `MEMORY.md` 索引那行升级为 "工具准入第一原则" 标题
- `feedback_browser_use_not_for_scraping.md` 整体重构：先讲元原则，browser-use 降为反例 + 应用要点

未来评估新 MCP / 工具时，无论我从 docs 读还是从 memory 读，都会先碰到"audit 追溯优先于技术能力"这条硬规则——browser-use 那种"听起来很全面"的诱惑不会再被我无意识列出来。

No response requested.

---

### Turn 19  *(2026-05-16 03:16)*

**User:**

有些skill会内置mcp工具吗？

**Assistant:**

可以，但要分清楚是哪种"内置"。三种情况：

---

## 1. SKILL.md 本身不能定义 MCP server

Skill 的本质是**指令包**：一个 markdown 文件描述"什么场景触发我、我怎么干活"。它不能"自带"一个 server。

但 SKILL.md 可以：
- 在 `description` 里写**触发条件**（"trigger when 用户调用 X MCP 失败"）
- 在指令里**假设某 MCP 已配好**并直接调（如 `mcp__firecrawl__firecrawl_scrape ...`）
- 写"requires X MCP"提示用户先装

→ 这种 skill **依赖** MCP，不**自带** MCP。装 skill 跟装 MCP 是两回事。

---

## 2. Plugin（真正的"打包内置"形式）

Claude Code 的 plugin 机制是把 **skill + agent + slash command + MCP server 配置** 打包成一个发布单元，安装时一次性注册到 `~/.claude.json`。

这才是真正的"skill 内置 MCP"——你装一个 plugin，它会：
- 注册 SKILL.md
- 同时往 `~/.claude.json` 写 MCP server 配置
- 可能要求你手动补 API key

例子（官方/常见）：
- `firecrawl` plugin（如果存在）→ 带 firecrawl-mcp 配置 + 一个 "scrape this page" skill
- `tavily` plugin → 带 tavily-mcp + "research this topic" skill
- 部分 ClawHub 上"Web Research"、"Sales Intelligence"类组合包

---

## 3. ClawHub / SkillHub / 第三方市场

这些市场上**绝大多数**单 skill 不带 MCP（纯指令包）。少数标注"plugin"或"requires server"的会同时引导你装 MCP。

判断方法：
- 进 skill 详情页看 `requires:` / `dependencies:` 字段
- 看 SKILL.md 里有没有 `mcp__*` 字样的工具调用示例

---

## 对你项目的实际意义

你目前的 skills（grill-me、run-position、review、security-review、claude-api 等）都是**纯指令包**，不内置 MCP。你的 MCP（tavily/firecrawl/jina）是手工在 `.mcp.json` 配的，跟 skill 解耦。

这个组合方式实际上是**最灵活的**：
- skill 只描述"怎么思考"
- MCP 只提供"工具能力"
- 你在中间用 docs/mcp-tools.md 决定"哪个 skill 用哪个工具"

如果未来想图省事，可以发一个 plugin 把 `run-position` skill + `tavily/firecrawl/jina` MCP 配置一起打包，团队/未来的你装一次到位。但短期没必要——你的 .mcp.json + .env 已经定型，再套层 plugin 反而引入新的"装到哪、版本对不对"的问题。

---

### Turn 20  *(2026-05-16 03:20)*

**User:**

run-position skill是什么，我没在这个项目装skill，也没用skill

**Assistant:**

我说错了——道歉。`run-position` 是 **slash command**（你项目里的 `.claude/commands/run-position.md`），不是 skill。我上一轮说成 "skill" 是混淆了术语。

---

## 准确分类

你项目里 `.claude/` 目录下只有两类东西：

| 类型 | 你项目有的 | 干啥 |
|---|---|---|
| `commands/*.md` | `run-position.md` | 用 `/run-position P0X` 触发的 slash command |
| `agents/*.md` | `exhibition-finder.md`、`exhibition-verifier.md`、`cross-batch-merger.md` | subagent，由主对话 `spawn Agent(subagent_type=...)` 调 |

**没有 skills/ 目录，没装任何 skill**。

---

## 那 system reminder 里列的那一堆是什么

Claude Code 的 `Skill` 工具是个**统一调用接口**，把这几类都列进 "available skills"：
- 项目级 slash command（你的 `run-position`）
- 全局/内置 skill（`grill-me`、`init`、`review`、`security-review`、`claude-api`、`update-config` 等——Anthropic 默认带或 plugin 安装的，都在 `~/.claude/skills/` 下，跟项目无关）
- Plugin 安装的 skill（`frontend-design:frontend-design` 那种带 namespace 的）

所以 system reminder 那个 "available skills" 列表混合了三种来源，我上一轮没区分清楚就拿 `run-position` 当例子，等于把你项目的 slash command 错称成 skill。

---

## 回到你最初的问题"skill 会内置 mcp 工具吗"

修正回答：
- 你项目里 `run-position` slash command 不内置 MCP——它通过 `.mcp.json`（项目级）或 `~/.claude.json`（全局）配的 MCP，跟 command 文件解耦
- 你用过的全局 skill（grill-me / review / security-review 等）都是纯指令包，也不带 MCP
- 真正能"打包内置 MCP"的是 **plugin** 这种更高层的发布单元，但你没装过 plugin

所以**目前你这套配置完全没碰过 plugin 那种"自动装 MCP"机制**，你装 MCP 一直是手工编辑 `.mcp.json` + `.env`——清晰可控。

---

### Turn 21  *(2026-05-16 03:34)*

**User:**

如果这个项目要给到其他团队，这些配置要怎么一次装到位？用plugin可以吗？其他团队连claude code cli都没装的那种，可能需要小白的教程

**Assistant:**

简答：**plugin 不能一次装到位，最多帮你省 10% 的事**。真正的痛点是 OS 级和 API 注册环节，plugin 都解决不了。

---

## 拆开 3 层看分发难度

| 层 | 内容 | plugin 能解决吗 |
|---|---|---|
| **L1 OS 级** | 装 Node.js、装 Claude Code CLI、登录 Anthropic | ❌ plugin 是 Claude Code 内的概念，前提是 Claude Code 已装好 |
| **L2 第三方账号** | 注册 Tavily / Firecrawl / Jina，拿 3 个 API key | ❌ plugin 不能帮人注册第三方账号 |
| **L3 项目本体** | clone repo、装 Python deps（playwright）、填 .env、配 ~/.bashrc 函数、`.mcp.json` 注册 | ⚠️ plugin 能把 `.mcp.json` 那步省掉，其它还是要手动 |

L3 里 plugin 唯一帮的就是"自动注册 MCP server 配置到 `~/.claude.json`"——但你还是要让用户自己填 API key。所以**总流程从 8 步降到 7 步**，没本质改善。

---

## 真正能"一次装到位"的方案

写一份 **INSTALL.md 教程 + 一个 bootstrap.sh 引导脚本**。

### INSTALL.md 大纲（约 50 行）

```
## 前置准备
1. 注册 3 家 API（含截图 + 链接）
   - Tavily：https://app.tavily.com → 拿 tvly-xxx
   - Firecrawl：https://www.firecrawl.dev → 拿 fc-xxx
   - Jina：https://jina.ai/?sui=apikey → 拿 jina_xxx

## 系统要求
- Linux / macOS / WSL2（不支持 Windows native）
- Python 3.10+
- Node.js 18+

## 安装步骤
1. 装 Node.js
   - Mac:    brew install node
   - Ubuntu: curl ... | bash + nvm install 22
2. 装 Claude Code CLI
   - npm install -g @anthropic-ai/claude-code
3. 登录 Anthropic
   - claude login
4. clone 项目 + 跑 bootstrap
   - git clone <repo>
   - cd <repo>
   - ./scripts/bootstrap.sh    # 引导你填 3 个 key + 装 Python deps + 配 ~/.bashrc 函数
5. 验证
   - cd <repo>
   - claude
   - 输 /mcp，看 3 个 server 全 ✓
```

### bootstrap.sh 干啥（约 80 行）

- 检测 OS，如不支持就提示退出
- 检测 Node / Python / Claude Code 是否已装，缺啥提示去装啥（不强行帮装，避免 sudo）
- 引导用户输入 3 个 API key（命令行交互），写入 `.env`
- 装 Python deps：`pip install -r requirements.txt && playwright install chromium`
- 检查 `~/.bashrc` 是否已有 `claude()` 函数，没有就追加
- 跑一次 smoke test（curl 三个 API endpoint，验 key 有效）

---

## 我的建议

**先写 INSTALL.md，bootstrap.sh 视情况**。理由：

- 如果接收方是技术团队（会 git / npm / pip），INSTALL.md 一份够了，bootstrap.sh 反而是多余抽象
- 如果接收方是 HR 全小白……他们可能根本不该自己跑 Claude Code，应该是你跑完出 xlsx 给他们用
- bootstrap.sh 是个甜区方案，但要维护跨平台兼容（Mac / Ubuntu / WSL 三套 brew/apt/nvm 命令）

要我现在写 INSTALL.md 吗？写之前一个问题：

那 INSTALL.md / bootstrap.sh 都用不上——**完全小白根本不该自己装这套**。从"分发工具"切换成"分发成果"才是正解。

---

## 重新拆这个交付链

| 角色 | 干啥 | 需要什么 |
|---|---|---|
| **你 / 项目运营者** | 跑 `/run-position P0X`，把 result.xlsx 交付出去 | 现有的 `.mcp.json` / `.env` / `~/.bashrc` 那套（已成型） |
| **HR / 业务同事**（小白） | 收 xlsx → 看公司清单 → 联系人选 → 主动挖人 | xlsx 怎么读、字段意思、怎么验证证据链 |

小白要的是**怎么用 xlsx 干活**，不是"怎么把这套跑起来"。后者你做就行。

---

## 真正要写的两份文档

### 1. `USER_GUIDE.md`（给 HR 看，最关键）

主要内容（约 40 行 + 截图）：

```
## 你拿到的是什么
- 一份 result.xlsx，含 3 个 sheet：总览 / P0X 命中 / unmatched
- 每行一家公司，每行有官网链接、命中品类、来源展会、证据原文

## 怎么用
1. 打开"P0X 命中" sheet
2. A 列公司名是 hyperlink，点进去就是该公司官网
3. F 列 "match_reason" 告诉你为什么这家公司被命中（如"主营浮选药剂"）
4. G 列 "exhibit_sources" 列出来源展会 + 展商目录 URL + evidence_quote
   → evidence_quote 是从展商目录抓的原文，你可以打开 URL 用 Ctrl+F 验

## 主动挖人步骤
1. 选择目标公司（参考 main_categories 是否对口）
2. 点公司名打开官网，找"关于我们" / "联系我们" 看公司规模和地点
3. 用脉脉 / LinkedIn 搜 "<公司名> 研发"，找在职员工
4. 沟通时带上 evidence_quote，证明对该公司了解（避免被识破是冷推）

## 字段说明（表格）
- main_categories: ...
- url_check_status: live=活官网，dead=域名挂了
- url_confidence: 0-1，<0.5 表示官网证据不强（如阿里店铺当作官网）
- is_inferred=true: 推断池，没在官方展商目录直接抓到，谨慎用
- ...

## 常见问题
- 为什么公司 X 没出现？→ 可能 categories.yaml 没覆盖该品类，找运营反馈
- 公司链接打不开？→ 看 url_check_reason 字段，可能反爬或暂时挂了
```

### 2. `RUNBOOK.md`（给你自己 / 接手的运营者，可选）

短一些（约 30 行）：

```
## 跑流程
- cd 项目目录
- claude
- /run-position P0X
- 等闸门 1（用户审 exhibitions.csv）→ 审完回 ok
- 等闸门 2（merge_suspects.tsv 人审）→ 审完回 ok
- 拿 positions/P0X_*/result.xlsx 出来交付

## 交付方式
- xlsx 上传到 <共享盘 / 邮件 / 内部 IM>
- 附 USER_GUIDE.md 链接给收件人

## 出问题
- /mcp 看三个 server 是否 ✓ connected
- 命中率 < 10% → 看 categories.yaml main_categories 是否过窄
- 详细排查见 docs/mcp-tools.md 末尾
```

---

要我先写哪份？我推荐 **USER_GUIDE.md** 优先（HR 侧能直接用，价值最大）；RUNBOOK.md 是给"未来接手你的人"看的，可以等真要交接时再写。

---

### Turn 22  *(2026-05-16 03:41)*

**User:**

怎么把这个项目要给到其他团队使用，这些配置要怎么一次装到位？用plugin可以吗？其他团队连claude code cli都还没安装

**Assistant:**

直接说结论：**plugin 解决不了你最大的痛点**。它能打包 Claude Code 内的 agents/commands/MCP 配置，但不能装 Claude Code CLI 本身、Node、Python、playwright 这些运行时。你团队是从 0 开始的话，plugin 反而是最末端的事。

---

## 分层看要装什么（plugin 只盖最底下一层）

| 层 | 装的什么 | plugin 能帮吗 |
|---|---|---|
| 1. OS 工具 | Node 18+、Python 3.11+、git | ❌ |
| 2. Claude Code CLI | `npm i -g @anthropic-ai/claude-code` + `claude login` 或 ANTHROPIC_API_KEY | ❌ |
| 3. 项目代码 | `git clone` 你的 repo | ❌ |
| 4. Python 依赖 | `pip install -r requirements.txt` + `playwright install chromium` | ❌ |
| 5. Claude Code 项目配置 | `.mcp.json` / `.claude/agents/*` / `.claude/commands/*` | ✅ 但 git clone 已经带了 |
| 6. ~/.bashrc claude wrapper | 自动 source `.env` 的 shell 函数 | ❌ |
| 7. MCP API keys | Tavily/Firecrawl/Jina（每人自己注册，不能共享） | ❌ |

**第 5 层是 plugin 唯一能帮的部分，但你已经把 `.claude/` 和 `.mcp.json` commit 进 git 了——团队 clone 后自动拿到，不需要 plugin。**

---

## 推荐方案：`scripts/setup.sh` + `docs/SETUP.md`

跟 plugin 比，这是**最少惊喜、最不依赖 marketplace** 的方案：

**`scripts/setup.sh`**（一键脚本，做 1-6 层；7 层引导用户自己填）：
- 检查 Node / Python / git / Claude Code CLI 是否在 PATH，缺则打印安装命令并退出
- `python -m venv .venv && .venv/bin/pip install -r requirements.txt`
- `playwright install chromium`
- 幂等地往 `~/.bashrc` 追加 claude wrapper 函数（grep 一下没有再追加）
- `cp .env.example .env`（如果 .env 不存在），打印"现在去 https://app.tavily.com 等注册 key 填进 .env"
- 不要碰 Anthropic API key——让用户自己 `claude login` 或 export ANTHROPIC_API_KEY

**`docs/SETUP.md`**（4 步，新人 30 分钟内跑通）：
1. 装 Node 18+ / Python 3.11+ / git（贴每平台命令）
2. `npm i -g @anthropic-ai/claude-code` + `claude login`（用 Anthropic 账号 / API key）
3. `git clone <repo> && cd ... && ./scripts/setup.sh`
4. 注册 Tavily/Firecrawl/Jina 三个 key（贴注册链接）→ 填 `.env`
5. `claude` 启动 → `/run-position P01` 跑通验证

---

## plugin 在你场景的真实价值（小，但不是 0）

**有意义的场景**：你想让别的项目（非本 repo）的同事也能用 `/run-position` 这个 slash command。这时把 `run-position` + 三个 agent + `.mcp.json` 抽成 plugin 发布，他们 `claude plugin install <你的 plugin>` 一下，不用 clone 你的 repo 也能用。

**但这跟"团队部署本项目"完全是两件事**——本项目 = 业务 pipeline + Python + 7 个岗位的产出，根本不该跨 repo 复用，团队成员就该 clone 这个 repo 用。

---

## 下一步你要决定的几件事

要执行这个方案前，有 3 个细节得对齐——我列出来你直接回，写 setup.sh 时按你的回答来：

先回答你最关键的问题——**Claude Desktop 不能当 Claude Code 用**。

---

## 为什么 Claude Desktop 不行

它们是 Anthropic 两条**不同的产品线**：

| 产品 | 跑 `/run-position` 这种工作流 | 支持本地文件操作 / Python pipeline | 支持 `.claude/agents/`、`.claude/commands/` |
|---|---|---|---|
| **Claude Code (CLI)** | ✅ | ✅ | ✅ |
| **Claude Code (VS Code 扩展)** | ✅（功能等价 CLI） | ✅ | ✅ |
| **Claude Desktop（桌面 app）** | ❌ | ❌（沙盒） | ❌（只读 MCP） |
| **Claude.ai/code (Web)** | ❌ | ❌ | 部分 |

Claude Desktop 能装 MCP servers，但不能加载你项目里的 `.claude/agents/exhibition-finder.md` 也不能跑 `pipeline/scrape_exhibition.py`——`/run-position` 整套基础设施就断了。Web 版同理。

---

## 团队真正可选的两条路

**A. Claude Code CLI**（你现在的方式）—— 最完整，需要熟 terminal
**B. Claude Code VS Code 扩展** —— 同样的能力，UI 更友好；很多人本来就在用 VS Code，**对非工程背景的成员推荐这条**

两条都需要：
- Claude Pro/Max 账号 + `claude login`（你已确认这个方案）
- 本地 Node 18+、Python 3.11+、git
- git clone 本项目
- 跑 setup.sh 装 Python deps

差别只在最后一步：A 是 terminal `claude`，B 是 VS Code 里 Cmd+L 打开 Claude 面板。

---

## 还需要你确认 OS（之前那题没答）

OS 决定 setup.sh 怎么写，请选一下：

好——Windows + WSL 混合 + CLI/VS Code 并列。我先看一下项目现状（有没有 requirements.txt、README 部署说明），再设计 SETUP.md 和 setup.sh。

没有 requirements.txt / README——要新建。再扫下 tests + pip freeze 确认完整依赖。

依赖盘点齐了——10 个第三方包，全在你的 env 里在跑。我现在并行建 3 个文件：`requirements.txt` + `scripts/setup.sh` + `docs/SETUP.md`。

三件套齐：

| 文件 | 大小 | 作用 |
|---|---|---|
| `requirements.txt` | 800B | 10 个 Python 包，pin major + 弹性 minor |
| `scripts/setup.sh` | 4KB | 5 步幂等装机：检 prereq → .venv → playwright → bashrc wrapper → .env |
| `docs/SETUP.md` | 5.7KB | 完整文档：6 章节 + 故障排查表 |

---

## 团队拿到项目后的真实路径

```bash
# 0. 装 prereq（按 SETUP.md §2，5-15 分钟）
# Windows 用户先 wsl --install -d Ubuntu

# 1. clone + 装
git clone <repo> && cd category-based-rd-talent-source-mapping
./scripts/setup.sh                    # 5-10 分钟

# 2. 装 Claude Code（CLI 或 VS Code 二选一）+ claude login

# 3. 注册 3 个 key 填 .env（5-10 分钟）

# 4. source ~/.bashrc + claude + /mcp + /run-position P01
```

**总耗时约 30-45 分钟**，跟 SETUP.md 估的一致。

---

## 建议（要不要做你定）

1. **CLAUDE.md 顶部加"团队部署看 docs/SETUP.md"一行**——CLAUDE.md 是项目宪章，新人 clone 后会先看，应该指引到部署文档。要我加吗？
2. **commit 这次新增的 7 个文件**：
   - `.mcp.json` / `.env.example` / `scripts/start-claude.sh`（前面会话加的）
   - `requirements.txt` / `scripts/setup.sh` / `docs/SETUP.md` / `docs/mcp-tools.md`（这次加的）
   - 还有 4 处 agent/command 的修改：`.claude/agents/exhibition-finder.md` / `exhibition-verifier.md` / `.claude/commands/run-position.md`、`.gitignore`
3. **可选删除 `scripts/start-claude.sh`**——前面建的显式 wrapper，现在被 `setup.sh` + `~/.bashrc` 函数取代，应急诊断价值低。要不要砍？

需要哪几个我帮你做？

---

### Turn 23  *(2026-05-16 06:16)*

**User:**

mcp，claude配置，skill这些都通过git传输吗？source ~/.bashrc + claude + /mcp + /run-position P01 这是什么？plugin吗？删除 scripts/start-claude.sh

**Assistant:**

三个问题分别答。先把 start-claude.sh 删了。

✅ 删完，`scripts/` 现在只剩 `setup.sh`。

---

## Q1：MCP / Claude 配置 / skill 怎么传递

要分**项目内 vs 用户机器全局**两类——git 只能传项目内。

### 🟢 项目内（git tracked，团队 clone 自动拿）

| 文件 / 目录 | 内容 |
|---|---|
| `.mcp.json` | MCP server 配置（用 `${VAR}` 占位，无密钥，可放心 commit） |
| `.claude/agents/*.md` | 项目专属 subagent（exhibition-finder / verifier / cross-batch-merger） |
| `.claude/commands/*.md` | 项目专属 slash command（`/run-position`） |
| `config/*.yaml`、`pipeline/*.py`、`docs/*.md`、`CLAUDE.md` | 业务代码 + 文档 |
| `requirements.txt`、`scripts/setup.sh` | 部署脚本 |
| `.env.example` | API key 占位模板（**不含真 key**） |

### 🟡 项目内但 gitignored（每人自己填，clone 后自己生成）

| 文件 | 谁生成 |
|---|---|
| `.env` | 团队成员注册 Tavily/Firecrawl/Jina 后自己填 |
| `.claude/settings.local.json` | 个人权限偏好，Claude Code 第一次运行时自动建 |

### 🔴 用户机器全局（git 完全管不到，要用 `setup.sh` / `claude login` 各自重建）

| 文件 / 状态 | 怎么来 |
|---|---|
| `~/.bashrc` 的 `claude()` wrapper 函数 | `setup.sh` 自动追加（幂等） |
| Anthropic 账号登录态（API key 或 session） | `claude login` 走一次 |
| `~/.claude/skills/*` 全局 skill（grill-me、review 等） | Claude Code 自带 / 用户自己装的 plugin。**不跟项目走**，每人可以装不同的 |
| Claude Code CLI 本身 | `npm install -g @anthropic-ai/claude-code` |

---

## Q2：`source ~/.bashrc + claude + /mcp + /run-position P01` 是啥

不是 plugin，**也不是一行复合命令**——是我图省事用 `+` 把 4 个不同环境的步骤串了，写法很差，让你产生混淆。歉。

正确读法：

```bash
# === 在 bash 终端里跑 ===
source ~/.bashrc                # ① 让 setup.sh 加的 wrapper 函数生效（仅当前终端，或开新终端自动加载）
cd /path/to/project             # ② 进项目目录（wrapper 需要从这里往上找 .env）
claude                          # ③ 启动 Claude Code CLI，进入 Claude 会话
```

进 Claude 会话后：

```
# === 在 Claude 会话内输入（不是 bash 命令）===
/mcp                            # ④ 看 MCP server 状态，确认 tavily/firecrawl/jina 三个 ✓
/run-position P01               # ⑤ 触发本项目 slash command 开始工作流
```

`/mcp` 和 `/run-position` 是 Claude 会话内的 slash command（前面斜杠就是这个意思），跟 bash 终端命令完全不同环境。

---

## 还要做的

- **CLAUDE.md 顶部加部署指引**（建议）—— 团队成员 clone 后先看 CLAUDE.md，应该一眼看到 "团队部署看 docs/SETUP.md"。要我加这一行吗？
- **commit 这次的改动**（建议）—— 一共 9 个文件改动 / 新增，要不要我列一下 git diff 然后帮你 commit？

---
