
---

### Turn 12  *(2026-05-19 03:41)*

**User:**

操作型skill，参考型skill，网上有哪些好的

**Assistant:**

先用我能直接看到的样本作底，再去网上搜补充。本地 `claude-plugins-official` 这个 marketplace 已经缓存了一批官方 plugin/skill，先扫一遍。

这个 marketplace 量不少。我挑几个最有代表性的快速扫一下结构，看哪些有 SKILL.md（不是所有 plugin 都内嵌 skill），再分操作型 / 参考型给你。

这一扫已经有结论 — **Anthropic 官方很多 plugin 根本不用 skill**（code-simplifier、pr-review-toolkit、feature-dev、code-modernization 都是 commands + agents 组合）。这印证了上一轮的判断。

带 SKILL.md 的有 7 个 plugin。让我并行读 4 个最有代表性的 frontmatter，确认分类。

最后看一下 plugin-dev 那三个 skill 和另外两个 mcp 的 sibling，把样本看全。

你这个分类（操作型 / 参考型）很准确 — Anthropic 自己写的官方 skill 正好就是按这两类分的。本地 marketplace 里 11 个有 SKILL.md 的 plugin，按这个分类列下来非常清晰。

## 参考型 skill（5 个 / 提供知识，无步骤）

| skill | 行数 | 内容 | 看点 |
|---|---|---|---|
| **frontend-design** | 41 | UI/UX 审美原则、字体/色彩/动效指南 | **极简范本**。1.5KB 解决"该用什么品味"，显式 NEVER 反模式清单 |
| **writing-hookify-rules** | 374 | hookify rule 语法 + pattern 参考 | 触发词列得很全（"create a hookify rule"/"add a hookify rule" 等） |
| **plugin-structure** | 476 | 插件目录布局、plugin.json、组件组织规范 | 架构参考类的标准形态 |
| **plugin-settings** | 544 | `.claude/plugin-name.local.md` 配置文件 pattern | 把一个具体协议讲清楚 |
| **mcp-integration** | 554 | MCP server 类型 (SSE/stdio/HTTP/WS) 配置参考 | 协议配置手册 |

特征：节标题是 **Overview / Format / Patterns** 这种**陈述性**的；没有 Phase / Step / Workflow。

## 操作型 skill（6 个 / 执行流程）

| skill | 行数 | 内容 | 看点 |
|---|---|---|---|
| **session-report** | **42** | 从 transcripts 生成 HTML 报告 | **极简操作型范本**。`## Steps` 直接列步骤 |
| **claude-md-improver** | 179 | 审计 CLAUDE.md 质量 + 改 | Workflow / Quality Report / Templates 三节经典结构 |
| **build-mcpb** | 199 | 打包 MCP server 成 .mcpb | |
| **build-mcp-server** | 221 | 引导用户建 MCP server | **Phase 1/2/3**：询问用例 → 推荐部署模型 → 选 tool 模式 |
| **build-mcp-app** | 392 | 给 MCP server 加 UI widget | description 明确写"AFTER build-mcp-server"，**操作型 skill 互相串联**的例子 |
| **skill-creator** | 485 | "造 skill 的 skill" | 含 grader/analyzer/comparator agents + references/schemas.md，**复杂操作型 skill 怎么组织内部资源**的范本 |

特征：节标题是 **Steps / Workflow / Phase N**；description 里有明确触发动词（"create / build / audit / generate"）。

## 最值得你直接读的几个（按学习价值排）

1. **`frontend-design`** — 1.5KB 学怎么写**极简参考型 skill**
2. **`session-report`** — 1.5KB 学怎么写**极简操作型 skill**
3. **`skill-creator`** — 485 行 + agents/ + references/，学**复杂 skill 怎么组织内部资源** + 它自己讲怎么造 skill 的元设计
4. **`claude-md-improver`** — 学**有产物的操作型 skill**怎么把 Workflow + 输出报告模板放在一起
5. **`build-mcp-server`** — 学**操作型 skill 之间通过"AFTER X"声明依赖串联**

## 一个比"哪些好"更重要的发现

回看 marketplace 里**没用 skill 的** plugin：

- `code-simplifier`、`code-review`、`pr-review-toolkit`、`feature-dev`、`code-modernization`、`commit-commands`、`agent-sdk-dev`、`hookify` 主流程 → **全是 commands + agents 组合，没用 skill**

Anthropic 自己的选择印证了上一轮我们的判断：**当流程明确、入口固定、需要工具集隔离 / 子上下文时，commands + agents 才是首选 — 而不是 skill**。 skill 适合"模型自主识别意图、按需召唤指令"这种**不确定何时被需要**的能力。

具体例子：`pr-review-toolkit` 里有 7 个 agent (code-reviewer / pr-test-analyzer / silent-failure-hunter / type-design-analyzer 等) + 1 个 `/review-pr` command。**完全没 skill**。流程明确就 commands+agents，干净利落。

## 网上从哪里拿

本地这些都是从 Anthropic 官方 marketplace 同步下来的，源仓库就在 GitHub：

- **`github.com/anthropics/claude-plugins-official`** — 上面所有样本的源
- 你也可以 `cat` 本地缓存：`~/.claude/plugins/marketplaces/claude-plugins-official/plugins/<name>/`

至于第三方社区收集的 awesome-claude-skills 之类的列表，我没有 100% 确定的 URL，不胡乱给 — 但 GitHub 搜 `claude-code skills` topic 应该能找到一些社区 marketplace。本地这 11 个官方样本，作为学习材料**已经覆盖了 skill 设计的几乎所有 idiom**，不一定需要再去找。

## 给你的下一步建议

如果想学最快：**`cat frontend-design/SKILL.md` 和 `cat session-report/SKILL.md`**（一个参考型一个操作型，加起来 80 行），看完基本就明白每类怎么写。然后再翻 `skill-creator/SKILL.md` 看 Anthropic 自己怎么教别人造 skill — 那一份是元教材。

---

### Turn 13  *(2026-05-19 03:51)*

**User:**

帮我看看，skill-creator/SKILL.md 看 Anthropic 自己怎么教别人造 skill

**Assistant:**

我读 SKILL.md 全文，同时列一下它的 references 和 agents 子目录有什么。

读完了。这份 SKILL.md（485 行 + agents/ + references/ + scripts/ + assets/ + eval-viewer/）信息密度极大。我**不复述**，挑出最有价值、最反直觉、跟我们前面讨论**有出入**的几个点。

## 最值得拎出来的 7 个洞察

### 1. description 要"pushy"对抗模型 undertrigger（line 67）

官方原话：

> "currently Claude has a tendency to 'undertrigger' skills -- to not use them when they'd be useful. To combat this, please make the skill descriptions a little bit 'pushy'."

举的例子：原本 "How to build a simple fast dashboard..." 改成 "How to build a simple fast dashboard... **Make sure to use this skill whenever the user mentions dashboards, data visualization, internal metrics, or wants to display any kind of company data, even if they don't explicitly ask for a 'dashboard.'**"

**修正我之前的说法**：我说"description ≤ 100 字"过于死板。官方实际偏好**触发面写够、必要时主动指令模型用 ("Make sure to use this skill whenever ...")**，宁可长一点也别让模型漏触发。

### 2. 用"explain why"代替全大写 MUST / ALWAYS / NEVER（line 139, 302）

> "If you find yourself writing ALWAYS or NEVER in all caps, or using super rigid structures, that's a yellow flag — if possible, reframe and explain the reasoning so that the model understands why the thing you're asking for is important."

模型有 theory of mind，告诉它"为什么"比给它一堆 MUST 更有效。**例外**：反模式清单可以用 NEVER（`frontend-design` 里的"NEVER use Inter / Roboto"是合法的）— 全大写不是不能用，是不能**满篇都是**。

### 3. skill 有三类 bundled resources，不止 references（line 75-84）

我前面几轮只讲了 `references/`，**这是漏的**。官方分类完整版：

```
skill-name/
├── SKILL.md (必需)
├── scripts/    ← 确定性 / 重复性任务的可执行代码
├── references/ ← 按需载入的文档
└── assets/     ← 输出用的模板 / 图标 / 字体
```

**判断什么时候 skill 该带 `scripts/`**（line 304）：

> "If all 3 test cases resulted in the subagent writing a `create_docx.py` or a `build_chart.py`, that's a strong signal the skill should bundle that script."

**模型反复重写同一个 helper → 打包成 scripts/**。这是个量化标准，不是凭感觉。

### 4. SKILL.md 主体上限是 500 行（line 96）

> "Keep SKILL.md under 500 lines; if you're approaching this limit, add an additional layer of hierarchy along with clear pointers about where the model using the skill should go next to follow up."

> "For large reference files (>300 lines), include a table of contents"

具体数字：主体 < 500 行，单个 reference > 300 行就要加 TOC。**`research-one-company` 那个 800 行严重越界**，符合我们的反例判断。

### 5. 模型只在自己搞不定时才召唤 skill（line 396-400）

> "Claude only consults skills for tasks it can't easily handle on its own — simple, one-step queries like 'read this PDF' may not trigger a skill even if the description matches perfectly."

这点很重要 — 它意味着：

- 给极简任务（"读这个 PDF"、"写个 fib"）做 skill 没用，模型不会召唤
- 测试 skill 的 eval prompt 要**够复杂、够具体**才能测出触发效果
- skill 真正的价值场景是**多步、专业化、模型独自做容易出错**的任务

### 6. 跨变体的组织模式：主 SKILL.md 做选择，references 分文件（line 100-109）

```
cloud-deploy/
├── SKILL.md (workflow + 选择哪个云)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

主 SKILL.md 不写"AWS 怎么部署 + GCP 怎么部署 + Azure 怎么部署"挤一堆，而是写"先识别用户用哪个云，然后去读对应 reference"。**模型只读到相关的那一份**。

这跟 `agentic-harness-patterns-zh` 把 9 个 pattern 分成 9 个 reference 文件是一致的设计。

### 7. description 优化是一个独立的工程问题（line 333-404）

我前面完全没提这个。Anthropic 自己做了一整套**触发率优化流程**：

1. 生成 20 条 eval query（10 should-trigger + 10 should-not-trigger）
2. **重点写 near-miss 测例**（关键词相似但应该不触发的反例）
3. 跑 `scripts/run_loop.py` 自动迭代 5 轮
4. 60% train / 40% test 划分，**按 test 分数选最佳**，避免过拟合到 train

具体来说 — **写一个 skill 之后，你应该真的跑 eval 测它的 description 触发率**，而不是凭感觉觉得"写得很 pushy 应该够了"。

## 整体方法论：6 步循环（line 10-21）

精简版：

```
1. 想清楚做什么 + 大概怎么做
2. 写 SKILL.md 草稿
3. 写 2-3 个真实 user prompt 作测试
4. 双盘并行跑: with-skill + baseline (without-skill)
5. 评估 (qualitative 看输出 + quantitative 看 assertions)
6. 根据反馈改 skill, 循环到满意
   ↓
   最后跑 description 优化
   ↓
   package_skill 打包发布
```

**关键设计**：with-skill vs without-skill 双盘对照。**没有对照组，你不知道 skill 是不是真的有用**。

## 跟我们前面讨论的对照

| 我们之前的说法 | Anthropic 官方做法 | 评 |
|---|---|---|
| description ≤ 100 字 | 触发面够 + pushy 即可 | **我之前过于死板**，应改为"够 pushy 触发面够，宁可长不要漏" |
| 主体薄、几千字 | < 500 行 | 一致 |
| 召唤式不是 SOP 式 | imperative form + explain why | 一致，但我没强调 "explain why > MUST" |
| references 拆分 | references / scripts / assets 三类 | **我前面只讲了 1/3**，scripts 和 assets 也是 skill 的合法组成 |
| 显式划边界 (NEVER / 不适用于) | 慎用 NEVER，优先 explain why | 部分修正：NEVER 可以用但别滥用 |
| **没讲过：评测对照** | with-skill vs without-skill 双盘 | **关键缺失** |
| **没讲过：description 量化优化** | 60/40 切分 + 5 轮迭代 + near-miss 测例 | **关键缺失** |

## 一句话总结

**我们前面讨论停留在"skill 形态判断"层面（什么时候该做 skill、放哪里、写多少字），到了实际开发，Anthropic 自己的做法多了一整套"测试 + 迭代 + 量化优化"环节**。如果你真要给一个 skill"上质量保证"，光看形态对不对是不够的 — 还得跑双盘对照测出 with-skill 比 baseline 强多少、跑 description 优化测出触发率高不高。

## 给你的下一步

如果想真把它学到手，建议按这个顺序读：

1. **再读一遍 SKILL.md line 297-306**（怎么改 skill 的心态）— 这一段是这份文档最浓缩的部分
2. **看 `agents/grader.md`**（怎么把 assertion 评估自动化）
3. **看 `references/schemas.md`**（eval / grading 的 JSON 结构）
4. 然后挑一个你自己想做的 skill（比如把那份"局限性考量.md"做成用户级 skill），按这个流程实际跑一遍

要不要我读一下 `agents/grader.md` 和 `references/schemas.md`，把"评测怎么跑"那一层也讲清？

---

### Turn 14  *(2026-05-19 03:56)*

**User:**

本地 marketplace 里 11 个有SKILL.md 的 plugin。这个plugin是什么？为什么自带skill

**Assistant:**

好问题。先快速核实结构再答 — 看一下 plugin 包装里到底有什么。

事实搞清了。用户问的"为什么自带 skill"其实问题前提稍有偏差 — plugin 不是"自带 skill"，是 plugin 这个抽象**比 skill 高一层**，skill 是 plugin 内部的**可选组件之一**。下面把层级讲清。

## 层级关系

```
Marketplace (集中分发)
└── Plugin (分发单元, 像 npm package)
    ├── .claude-plugin/plugin.json   ← 必需。标识 + 元数据
    ├── skills/                       ← 0 个或多个 SKILL.md
    ├── commands/                     ← 0 个或多个 slash command
    ├── agents/                       ← 0 个或多个 subagent 定义
    ├── hooks/                        ← 0 个或多个 hook 配置 (可选)
    ├── .mcp.json                     ← MCP server 配置 (可选)
    └── README.md / LICENSE
```

**Plugin** = 一个有 `.claude-plugin/plugin.json` 的目录。它是 Claude Code 的**分发单元**，类比 npm package、VSCode extension、Chrome extension。

**Skill** = plugin 内部 `skills/` 子目录下的内容，是 plugin 可包含的**一种**组件。

**Marketplace** = 一个有 `.claude-plugin/marketplace.json` 的 git 仓库，列出多个 plugin 的元数据 + 安装源（git URL + ref + sha）。本地这个 `claude-plugins-official` 就是 Anthropic 维护的官方 marketplace。

## 关键事实：一个 plugin 里**可以**没有 skill

看刚才看到的真实数据，三种形态都存在：

| 形态 | 例子 | 内部结构 |
|---|---|---|
| **只有 skill** | `frontend-design` | `skills/frontend-design/SKILL.md` 一份 41 行的文档，没别的 |
| **只有 agents + commands** | `pr-review-toolkit` | 7 个 agent + 1 个 `/review-pr` command，**完全没 skill** |
| **混合** | `plugin-dev`, `skill-creator` | skill + agents + scripts + assets 都有 |

所以"plugin 自带 skill"这个表述要修正为：**plugin 是包装，作者根据这个功能的最佳形态决定包里放什么 — skill 只是选项之一**。

## 为什么有的 plugin **选**放 skill

回到上一轮分类，作者通常在以下场景把功能做成 skill：

1. **能力是"召唤式知识"** — 比如 `frontend-design` 的审美指南，模型在合适场景应该主动想起来用
2. **能力是"多步操作流程"且需要灵活触发** — 比如 `claude-md-improver` 在用户说"检查我的 CLAUDE.md"时被召唤
3. **能力是"多个子组件共享的参考"** — 比如 `plugin-dev` 里 `plugin-structure` skill 被它的 commands 和 agents 共同引用

## 为什么有的 plugin 不放 skill

`pr-review-toolkit` 是最干净的例子。它的能力是"用 7 个专门 reviewer 并行扫一个 PR"：

- 触发模式明确（用户输入 `/review-pr`，或写脚本调） → **command + agent 比 skill 直接**
- 7 个 reviewer 需要**独立子上下文 + 工具集隔离** → **agent 必需**
- 没有"模型自主识别要不要召唤"的需求 → **skill 反而是冗余抽象**

所以这种功能的最佳分发形态就是"1 command + 7 agents 打包成 plugin"。**plugin 这个壳是为了分发,里面装什么由功能形态决定。**

## 一个 plugin 内组件互相调用的关系

举个真实例子 `skill-creator`（我们刚看过）：

```
skill-creator/
├── SKILL.md          ← 主入口（操作型 skill：教用户造 skill）
├── agents/
│   ├── grader.md     ← SKILL.md 里调用："spawn a grader subagent that reads agents/grader.md"
│   ├── analyzer.md   ← 同上
│   └── comparator.md ← 同上
├── references/
│   └── schemas.md    ← SKILL.md 里链接："see references/schemas.md for the schema"
├── scripts/
│   ├── aggregate_benchmark.py  ← SKILL.md 指令调用
│   └── run_loop.py             ← SKILL.md 指令调用
├── assets/
│   └── eval_review.html        ← SKILL.md 指令使用作模板
└── eval-viewer/
    └── generate_review.py      ← SKILL.md 指令调用
```

主 SKILL.md 像"导演" — 它在适当时机指示模型：spawn 这个 agent / read 那个 reference / run 这个 script / 用这个 asset。**plugin 把它们绑成一个包**，安装一次全到位。

## 重新回答你的问题

> "本地 marketplace 里 11 个有 SKILL.md 的 plugin。这个 plugin 是什么？为什么自带 skill"

- **plugin 是什么**：Claude Code 的分发单元，一个文件夹 + plugin.json
- **为什么"自带"skill**：作者觉得这个 plugin 提供的能力**最适合做成 skill 形态**（召唤式 + 模型自主触发），所以把 skill 文件放进 plugin 这个分发壳里。skill 不是 plugin 必带的 — marketplace 里 30+ 个 plugin 大多数都没 skill
- **plugin 和 skill 是不同层级**：plugin 是包装，skill 是包装里的一种可选物品。其他可选物品还有 commands、agents、hooks、MCP servers

类比一句：**plugin 之于 skill，约等于 VSCode extension 之于 keybinding** — extension 可以选择带 keybinding 也可以不带；带不带取决于它要分发什么功能。

---

### Turn 15  *(2026-05-19 06:03)*

**User:**

怎么自己做一个 plugin

**Assistant:**

正好 — Anthropic 自己写了一个 `plugin-dev` plugin 专门教这个。先看它的完整内容，再给你提炼。

并行读关键 4 个文件 — example-plugin 的真实最小样本 + Anthropic 推荐的 minimal-plugin + 创建命令。

好 — Anthropic 自己在 `plugin-dev` 这个 plugin 里就有完整教程，加上 `example-plugin` 这个最小样本，照着学最直接。给你提炼出来。

## 30 秒上手：最小可行 plugin（来自官方 minimal-plugin 样本）

```
hello-world/
├── .claude-plugin/
│   └── plugin.json            ← 只需要 {"name": "hello-world"}
└── commands/
    └── hello.md               ← 你的第一个 slash command
```

两个文件就够。`/hello` 立即可用。

`.claude-plugin/plugin.json`：

```json
{
  "name": "hello-world"
}
```

`commands/hello.md`：

```markdown
---
name: hello
description: Prints a friendly greeting
---

# Hello Command

Output the following greeting to the user, with current timestamp:

> Hello! This is from the hello-world plugin.
```

这就是一个可装、可跑的 plugin。**name 是唯一必需字段**，其它都可选。

## 完整目录结构（官方规范）

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json              # 必需：manifest
├── commands/                    # slash command 们（每个 .md 文件 = 一个命令）
├── agents/                      # subagent 定义们
├── skills/                      # skill 们（每个 skill 是一个子目录）
│   └── skill-name/
│       └── SKILL.md             
├── hooks/
│   └── hooks.json               # 事件钩子配置
├── .mcp.json                    # MCP server 配置
└── scripts/                     # 辅助脚本
```

**关键规则（违反会装不上）**：

1. **manifest 必须在 `.claude-plugin/` 下** — 不是放在根
2. **component 目录必须在 plugin 根** — 不是放在 `.claude-plugin/` 里
3. **只为你实际用到的 component 建目录** — 不要建空目录
4. **全部用 kebab-case** 命名（`plugin-name` ✓ / `pluginName` ✗ / `plugin_name` ✗）

## plugin.json 字段表

| 字段 | 必需? | 说明 |
|---|---|---|
| `name` | ✅ | kebab-case，全局唯一 |
| `version` | 推荐 | semver (`1.0.0`) |
| `description` | 推荐 | 一句话说明 |
| `author` | 推荐 | `{name, email, url}` |
| `homepage` | 可选 | 文档 URL |
| `repository` | 可选 | 仓库 URL |
| `license` | 可选 | `MIT` 等 |
| `keywords` | 可选 | 数组，便于发现 |
| `commands` / `agents` / `hooks` / `mcpServers` | 可选 | 自定义路径，**补充**默认目录而不是替换 |

## 选 plugin 里放什么 component（这是我们前几轮讨论的精华）

按你要分发的能力**形态**来选：

| 你的能力是... | 用 | 理由 |
|---|---|---|
| 模型在合适场景**自主识别**召唤的知识 / 流程 | **skill** | 召唤式触发 |
| 用户主动输入 `/foo` 触发的快捷动作 | **command** | 用户主权 |
| 需要**独立子上下文 + 工具集裁剪**的工作者 | **agent** | 子上下文隔离 |
| 生命周期事件（提交前 / 工具调用后等）的副作用 | **hook** | 自动注入 |
| 外部服务 / 第三方工具集成 | **MCP server** (`.mcp.json`) | 工具协议层 |

可以**共存** — `skill-creator` 同时有 skill + agents + scripts + assets；`pr-review-toolkit` 只有 agents + command。看你的功能需要哪几种形态。

## 本地装一个 plugin 来测试

最简单的办法是用本地路径安装，不需要发布到 marketplace：

```bash
# 在你想要的位置建好 plugin 目录
mkdir -p ~/my-plugins/hello-world/.claude-plugin
mkdir -p ~/my-plugins/hello-world/commands
# (写好 plugin.json 和 commands/hello.md)

# 在 Claude Code 里安装
> /plugin install ~/my-plugins/hello-world
```

或者用 `/plugin` 命令进 plugin 管理面板（具体 UI 看 Claude Code 当前版本）。

## 从哪开始最快

### 路径 A：复制 `example-plugin` 作脚手架（推荐起步）

```bash
cp -r ~/.claude/plugins/marketplaces/claude-plugins-official/plugins/example-plugin \
      ~/my-plugins/my-first-plugin
# 然后改名 + 改内容
```

`example-plugin` 是 Anthropic 自己写的"演示所有 component 类型"的样本，包含 commands、skills、`.mcp.json`、`plugin.json`。删掉你不要的，改成你的内容。

### 路径 B：装 `plugin-dev` 用它的 `/create-plugin` 命令引导

```
> /plugin install plugin-dev
> /create-plugin
```

`plugin-dev` 是 Anthropic 自己的"造 plugin 的 plugin"，里面有：

- 5 个 skill：`plugin-structure`、`plugin-settings`、`mcp-integration`、`skill-development`、`agent-development`、`hook-development`、`command-development`
- 3 个 agent：`plugin-validator`（验证 plugin 合规）、`skill-reviewer`、`agent-creator`
- 1 个 command：`/create-plugin`

这是**最权威的引导**，因为它由 Anthropic 自己写。

### 路径 C：纯手工，参考样本

照着 minimal-plugin 的两个文件起步，逐步加 component。深入参考：

- `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/skills/plugin-structure/SKILL.md` —— 完整规范（476 行）
- `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/skills/plugin-structure/examples/minimal-plugin.md` —— 最简样本
- `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/skills/plugin-structure/examples/standard-plugin.md` —— 中等
- `~/.claude/plugins/marketplaces/claude-plugins-official/plugins/plugin-dev/skills/plugin-structure/examples/advanced-plugin.md` —— 复杂

## 一个建议的实操路径

如果想真上手做一个对你有用的 plugin，我建议这样做：

1. **先选好你要做的能力**（比如 "我想把那份'局限性考量.md'变成可分发的 skill"）
2. **决定 component 形态**（参考型 skill → 只有一个 skill 在 plugin 里）
3. **复制 `example-plugin` 作脚手架**，删掉你不要的部分
4. **填进你的内容**，本地装上测试
5. **跑测试** — `frontend-design` 那种参考型 skill，最简单的测试是问几个真实场景看能不能召唤
6. **公开分发**（可选） — 推到 GitHub，加到一个 marketplace.json

要不要我直接帮你把那份"局限性考量.md" 包装成一个本地 plugin？最少 3 个文件就能跑起来 — 一个 `plugin.json`、一个 `SKILL.md`、加上你那份内容。

---

### Turn 16  *(2026-05-19 06:13)*

**User:**

用plugin和skill有什么不同，我公开分发到网上用哪种

**Assistant:**

这个问题里有个小前提偏差需要先理清：**plugin 和 skill 不是二选一**，它们是**不同层级**的东西。然后再讲分发。

## 关系澄清

```
Plugin（分发包装）
  └── skills/<name>/SKILL.md  ← skill 是 plugin 里可装的一种内容
```

两种部署形态都是合法的：

| 形态 | 长什么样 | 用途 |
|---|---|---|
| **裸 skill** | 一个 `SKILL.md` 文件放 `~/.claude/skills/<name>/` | 自用 / 本机开发 |
| **plugin 装着 skill** | 一个完整 plugin 目录，里面有 `.claude-plugin/plugin.json` + `skills/<name>/SKILL.md` | 分发给别人用 |

所以问"用 plugin 还是用 skill"准确版本是：**"我的 skill 该不该打包成 plugin 来分发？"**

## 公开分发：用 plugin，没有第二个选项

**Claude Code 没有"skill marketplace"，只有"plugin marketplace"**。这是事实层面的:

| 想做的事 | 裸 SKILL.md | plugin |
|---|---|---|
| 自己机器上用 | ✅ 丢进 `~/.claude/skills/` 就行 | ✅ |
| 给朋友看 | ⚠️ 发文件让对方手动 cp | ✅ `/plugin install <git-url>` |
| 进 marketplace 让人搜得到 | ❌ 不支持 | ✅ |
| 版本管理（v1.0.0 → v1.0.1） | ❌ 用户得手动 git pull 重新 cp | ✅ marketplace.json 里写 git ref / sha |
| 一键安装 / 卸载 | ❌ | ✅ `/plugin install` / `/plugin uninstall` |
| 用户能查更新 | ❌ | ✅ |
| 以后想加 command / agent | ❌ 用户得自己再 cp 一份 | ✅ 同一 plugin 包多加文件就行 |

**裸 SKILL.md 的所有问题在于：它没有分发协议**。用户要装就只能"git clone → cp 进 ~/.claude/skills/"，没有版本、没有索引、没有更新机制。

## 把 skill 包成 plugin 的额外成本

几乎为零 —— **就多两个文件**：

```
my-cool-skill-plugin/
├── .claude-plugin/
│   └── plugin.json          ← 新增 5 行 JSON
├── skills/
│   └── my-cool-skill/
│       └── SKILL.md         ← 原来的 SKILL.md 文件
├── README.md                ← 新增（GitHub 仓库的门面）
└── LICENSE                  ← 新增（公开发布建议加）
```

`plugin.json` 最简：

```json
{
  "name": "my-cool-skill-plugin",
  "version": "1.0.0",
  "description": "一句话说明",
  "author": { "name": "你的名字" }
}
```

对比这点成本和分发能力的差距，**只要打算让别人用，就值得包成 plugin**。

## 我的建议

| 场景 | 推荐 |
|---|---|
| 还在草稿、就你自己用 | 直接 `~/.claude/skills/<name>/SKILL.md`，连 plugin 都不用 |
| 想分享给同事 1-2 人 | 还可以直接发 SKILL.md 文件让他 cp；或者已经直接做 plugin 也行 |
| **公开放 GitHub，让任何人能用** | **必须是 plugin** —— 这是唯一可用的分发协议 |
| 进官方 marketplace（`anthropics/claude-plugins-official`）或第三方 marketplace | **必须是 plugin**，且 plugin 仓库要符合 marketplace.json 的 `source` 字段（git URL + path + ref + sha） |

## 容易混淆的两个事实

1. **"分享 skill"和"分享 plugin"在公开分发场景下是一回事** —— 因为 skill 没有自己的分发渠道。你说"我分享了一个 skill"，实际上你分享的是"一个只装了 skill 的 plugin"。

2. **plugin 不一定要装 skill** —— `pr-review-toolkit` 只装 agents + commands，根本没 skill。所以反过来也成立：plugin 是分发壳，**装啥都行**，未来你想从"只分享 skill"扩展到"还想分享一个配套 slash command"，原 plugin 直接加文件就行，不用重构。

## 一句话总结

- **plugin vs skill** 不是平级二选一 —— plugin 是分发壳，skill 是塞进去的内容
- **公开分发** 没得选 —— 必须打包成 plugin
- **代价极小** —— 多 1 个 `plugin.json`（5 行）+ 1 个 README，换来一键安装、版本管理、marketplace 可发现

---

### Turn 17  *(2026-05-19 06:16)*

**User:**

manifest是什么

**Assistant:**

软件工程通用术语 — **manifest = 包的"身份证 + 装箱单"**。它不是程序逻辑代码，而是描述这个包**是什么、属于谁、有什么内容**的元数据文件。

## 跨生态类比

你在别的地方肯定见过它的同类：

| 生态 | manifest 文件 |
|---|---|
| npm / Node.js | `package.json` |
| Python | `pyproject.toml` / `setup.py` |
| Rust | `Cargo.toml` |
| Go | `go.mod` |
| VSCode extension | `package.json` |
| Chrome extension | `manifest.json` |
| Android | `AndroidManifest.xml` |
| Docker / OCI | `Dockerfile` + image manifest |
| **Claude Code plugin** | **`.claude-plugin/plugin.json`** |
| **Claude Code marketplace** | **`.claude-plugin/marketplace.json`** |

它们功能高度一致 —— 都是回答四个问题：

1. **我是谁** —— name、version
2. **我有什么** —— entry point、组件列表、依赖
3. **我属于谁** —— author、license、repository
4. **怎么发现我** —— description、keywords、homepage

## Claude Code 里的两种 manifest

**`plugin.json`**（plugin 自己的身份证）：

```json
{
  "name": "my-plugin",                ← 我叫啥
  "version": "1.0.0",                  ← 哪个版本
  "description": "做 X 的工具",         ← 我是干嘛的
  "author": { "name": "..." },         ← 谁做的
  "commands": "./custom-commands"      ← (可选) 我的组件在非默认位置
}
```

它是 Claude Code 加载 plugin 时**第一个读**的文件 —— 没有它，Claude Code 不会认这个目录是个 plugin。

**`marketplace.json`**（marketplace 的目录清单）：

```json
{
  "name": "claude-plugins-official",
  "description": "...",
  "owner": { "name": "Anthropic" },
  "plugins": [                          ← 这个 marketplace 收录了哪些 plugin
    {
      "name": "frontend-design",
      "description": "...",
      "source": {                       ← 怎么把它装下来
        "source": "git-subdir",
        "url": "https://github.com/...",
        "ref": "v1.0.0",
        "sha": "abc..."
      }
    },
    ...
  ]
}
```

marketplace.json 是"商店货架清单"，列出每个 plugin 的去哪儿拉、什么版本、什么 git commit。

## 为什么需要 manifest（而不是直接扫文件）

如果没有 manifest，系统就只能扫整个目录树才知道这是什么。manifest 的存在让运行时可以：

1. **快速识别** —— 一眼看到 `name`、`version`，不用扫文件
2. **校验合法性** —— 缺 `name` 就拒绝加载，避免一半加载到一半失败
3. **声明依赖 / 配置** —— 告诉 Claude Code "我的 commands 在非默认路径 `./custom-commands`"，否则它只查默认目录
4. **支持索引** —— marketplace.json 列 100 个 plugin 时，不需要把每个 plugin 完整下载下来才知道它叫啥
5. **支持版本管理 / 校验** —— `version` 字段 + git `sha` 让用户能锁定版本、检测更新

简单说：**manifest 是包系统的"封面"**，让消费方在不打开整个包的情况下知道该不该用、怎么用、用哪个版本。

## 类比一句

如果一个 plugin 是一本书：

- 书的内容（commands / agents / skills / hooks 文件）= 正文
- **`.claude-plugin/plugin.json`** = 这本书的**版权页 + 目录页** —— 书名、作者、版本、ISBN、章节布局
- **`.claude-plugin/marketplace.json`** = **图书馆的索引卡** —— 馆藏哪些书、每本书放在哪个书架（git URL）、哪个版次（ref / sha）

没有版权页，图书管理员（Claude Code）不知道这书叫啥；没有索引卡，读者不知道图书馆有哪些书。

---

### Turn 18  *(2026-05-19 06:18)*

**User:**

https://github.com/keli-wen/agentic-harness-patterns-skill，这算哪种，这不就是把skills上传到github给人用的吧，也没有用plugin

**Assistant:**

让我直接打开看 — 你说的可能是对的，我需要核实是否要修正前面的说法。

仓库结构看清了 — 你完全说对了，**没有 `.claude-plugin/` 目录、没有 `plugin.json`**。它就是裸 skill 放 GitHub。我前几轮"公开分发必须用 plugin"的说法有偏差，需要修正。先确认一下作者的安装引导方式。

我前几轮"公开分发必须用 plugin"的说法**错了** —— 这个仓库正是反例。直接给你完整修正版。

## 这个仓库实际是什么

**裸 skill 仓库 + 第三方跨 agent 分发协议**。结构：

```
agentic-harness-patterns-skill/
├── README.md / LICENSE
├── skills/
│   ├── agentic-harness-patterns/      ← 英文版 skill
│   │   ├── SKILL.md
│   │   ├── metadata.json
│   │   └── references/
│   └── agentic-harness-patterns-zh/   ← 中文版 skill
│       ├── SKILL.md
│       ├── metadata.json
│       └── references/
└── docs/                              ← 设计过程文档
```

**完全没有 `.claude-plugin/plugin.json`**。它走的分发协议是 README 里写的：

```bash
npx skills add github:keli-wen/agentic-harness-patterns-skill
```

这是 **`github.com/vercel-labs/skills`** 维护的一个**跨 agent 标准** —— 不是 Anthropic 的，是 Vercel Labs 做的"open agent skills ecosystem"，**支持 Claude Code、Codex、Cursor 等 40+ 个 agent**。它读 GitHub 仓库的 `skills/<name>/SKILL.md` 结构，安装到当前 agent 的 skill 目录（Claude Code 就是 `~/.claude/skills/`）。

`metadata.json` 不是 Anthropic 标准的一部分，是 Vercel skills CLI 协议要求的字段。

## 完整的分发地图（修正版）

公开分发 skill 实际上有**三种**协议，不是一种：

| 协议 | 包装形态 | 安装方式 | 哪个生态 | 覆盖范围 |
|---|---|---|---|---|
| **Anthropic plugin** | `.claude-plugin/plugin.json` + `skills/...` | `/plugin install ...` | Anthropic 官方 | 只 Claude Code |
| **Vercel skills CLI** | 裸 `skills/<name>/SKILL.md` + `metadata.json` | `npx skills add github:owner/repo` | Vercel Labs / 社区 | Claude Code + Codex + Cursor 等 40+ agent |
| **完全裸仓库** | 直接 `SKILL.md` 文件 | 用户手动 `git clone` + `cp` 到 `~/.claude/skills/` | 无协议 | 任何支持 SKILL.md 的 agent |

## 这个作者为什么选 Vercel 协议而不是 plugin

从 README 字里行间能看出来：

1. **跨 agent 复用** —— "works with Claude Code, Codex, and 40+ other agents"。`plugin.json` 是 Claude Code 专有格式，包成 plugin 等于把这个 skill 锁死到 Claude Code 一个 agent 里。而 `SKILL.md` 本身是 portable 的 —— 其他 agent（Codex、Cursor）也能读。
2. **内容是纯知识 / 设计模式** —— 没有 command、agent、hook 之类 Claude Code 特有的组件需要分发，plugin 这一层抽象就是冗余包装。
3. **生态站队** —— 主动加入 Vercel Labs 的跨 agent 联盟，未来扩展性更大。

## 修正：什么时候用什么

| 你想分发的内容 | 推荐协议 |
|---|---|
| 纯 skill（一份知识 / 设计模式 / 风格指南）+ 想跨 agent 用 | **Vercel skills CLI**（裸 skill 仓库 + `npx skills add`） |
| skill + 配套 commands / agents / hooks / MCP（Claude Code 特有组件）打包 | **Anthropic plugin**（`.claude-plugin/plugin.json`） |
| 只想给自己 / 朋友用 | 直接 `~/.claude/skills/` 放裸 SKILL.md，不分发 |
| 进 **Anthropic 官方 marketplace** 被官方收录 | **必须** Anthropic plugin |
| 进 Vercel skills 生态被它索引 | **必须** Vercel 协议 |

## 我前几轮哪里错了

错在把 **Anthropic 自己的 plugin 协议**当成了**唯一的公开分发协议**。实际上：

- Claude Code 的 skill 子系统支持读 `~/.claude/skills/` 里的任何 SKILL.md，**不管它怎么进来的**(plugin 装的 / npx 装的 / 手动 cp 的)
- "怎么进来的" 是分发协议层决定的 —— Anthropic plugin、Vercel skills CLI、甚至手动安装都是合法路径
- "公开分发必须 plugin" 应改成 **"想用 `/plugin install` 一键安装、想进 Anthropic 官方 marketplace 必须是 plugin"**；想跨 agent 分发反而**不应该**用 plugin

## 对你的实际建议（修正版）

如果你打算把"局限性考量"那份内容分发到网上：

- **若仅给 Claude Code 用户 + 想被 Anthropic 官方 marketplace 收录** → 用 plugin
- **若想跨 agent（让 Cursor / Codex 用户也能用）+ 内容是纯 skill 知识** → 用 Vercel skills CLI 协议（这是你给的样本 `keli-wen` 选的路） —— 仓库做成 `skills/<name>/SKILL.md` 结构 + 加 `metadata.json` + README 写 `npx skills add ...`
- **若就给同事 1-2 人** → 完全裸仓库就行，让对方手动 cp

`keli-wen` 那种形态在"想最大化跨 agent 覆盖 + 内容是纯 skill"的场景下是更好的选择。我前面没提到这个生态，是真实漏了一块。

要不要我帮你看一下 Vercel skills CLI 的 `metadata.json` 字段规范是什么，确认下你做的话需要写什么？

---
