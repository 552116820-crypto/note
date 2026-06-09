# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `d1fd68ee-7072-4178-b72e-d532eeecf65f`
- **范围**: Turn 1-2
- **导出时间**: 2026-06-09 14:36:08

---

### Turn 1  *(2026-06-09 06:12)*

**User:**

claude agent SDK 只支持claude原生模型吗？deepseek v4 pro API能完美兼容吗？用claude agent sdk作为agent框架有什么好处，相较于其他主流框架和自建框架

**Assistant:**

The `claude-api` skill I just loaded covers the Claude **API** (Messages API, Managed Agents) but not the standalone **Claude Agent SDK** in depth, and your question touches fast-moving specifics (DeepSeek's current models, native Anthropic-compatible endpoints). Let me verify the current state rather than answer from memory.

已经核实清楚了。先纠正一个前提：你问的是 **Claude Agent SDK**（即原 Claude Code SDK 改名而来、跑在你自己进程里的 agent 框架），它和我刚加载的 claude-api skill 覆盖的 Managed Agents（Anthropic 托管、只跑 Claude）是两回事。下面按你三个问题分别答。

## 一句话结论

**不是只支持 Claude 原生模型，但它是"Claude 优先"的框架。** 原生支持的三条后端路线跑的**都是 Claude 模型**；接 DeepSeek 这类第三方模型要靠"Anthropic Messages API 兼容"的端点转译。DeepSeek V4 Pro **能跑、很便宜，但谈不上"完美兼容"**——是协议层兼容 + 特性降级 + 调优不对口。

## 1. 模型支持的真实层次

底层机制：Agent SDK 和 Claude Code 共用同一套 provider plumbing，所有请求都按 **Anthropic Messages API 协议**发出。**它认的是"协议",不是"模型"。** 于是分两层：

- **原生三条路(都是 Claude 模型,只是托管在不同云)：**
  - 默认 → Anthropic API
  - `CLAUDE_CODE_USE_BEDROCK=1` → AWS Bedrock 上的 Claude
  - `CLAUDE_CODE_USE_VERTEX=1` → Google Vertex 上的 Claude
  - ⚠️ Bedrock / Vertex **不能**让你跑 DeepSeek——它们只是 Claude 的另一个托管入口。

- **第三方模型(DeepSeek / Kimi / Qwen / GPT / 本地模型)：** 把 `ANTHROPIC_BASE_URL` 指向一个能"听懂"Anthropic Messages API 的端点。两种来源：
  - **网关/代理转译**：LiteLLM、OpenRouter、DeepInfra、vLLM、claude-code-router…
  - **模型厂商自带的 Anthropic 兼容端点**：DeepSeek 就属于这一类,不需要第三方代理。

## 2. DeepSeek V4 Pro 能"完美兼容"吗？

先确认事实:DeepSeek V4 Pro 是真模型(2026-04-24 发布,与 V4-Flash 同时,约 1.6T 参数,MIT 协议),而且 DeepSeek **官方就提供 Anthropic 兼容端点**:

- base_url:`https://api.deepseek.com/anthropic`
- 模型名:`deepseek-v4-pro` / `deepseek-v4-flash`(`deepseek-chat`/`deepseek-reasoner` 2026-07-24 弃用)

配置(SDK 和 CLI 吃同一套环境变量):

```bash
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_API_KEY=sk-<你的 DeepSeek key>      # DeepSeek 文档确认 x-api-key 完整支持
ANTHROPIC_MODEL=deepseek-v4-pro
ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4-pro
ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4-flash
ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

> 注意:Claude Code/SDK 的 `--model` 会校验 Anthropic 白名单,**直接传 `deepseek-v4-pro` 会被拒**。要靠上面这些 env 覆盖来指定真实模型(传 Claude 别名时 DeepSeek 端会自动映射,未知模型名默认落到 `deepseek-v4-flash`)。

**为什么"完美"是夸大——几个具体降级点:**

| 降级点 | 说明 |
|---|---|
| **beta/version 头被忽略** | DeepSeek 文档明说 `anthropic-beta`、`anthropic-version` 直接忽略 → 所有挂在 beta 头上的能力(fine-grained tool streaming、1M context、各种 beta 工具)**静默失效** |
| **Prompt caching 不对口** | harness 重度依赖 Anthropic 的 `cache_control` 断点缓存压成本/延迟;DeepSeek 有自己的自动上下文缓存,但断点语义和计费不保证 1:1。实务上常要禁用 prompt caching 避坑,于是丢掉这块优化 |
| **thinking 块映射不一致** | DeepSeek 思考模式经兼容层暴露的形式和 Claude 的 thinking block 不完全一样,透传程度看端点实现 |
| **harness 按 Claude 调的** | 系统提示、工具定义、上下文压缩、多步工具循环的可靠性,都是和 Claude 协同调优的;换 DeepSeek 后工具调用纪律、长程 agentic 稳定性会不同(不一定差,但**必须自己实测**) |

准确说法:**协议层能用、价格香(社区实测便宜 1–2 个数量级),但不是"drop-in 完美"。** 适合预算敏感、不依赖 Claude 专属特性的场景;当主力生产 agent、要榨干 harness 全部能力时会明显"不对口"。

## 3. 用 Claude Agent SDK 当框架的好处 vs 其他框架 vs 自建

定位:它是 **Claude Code 的同一套 harness**(官方把 Claude Code SDK 改名 Agent SDK 就是说"这套驱动 Claude Code 的 harness 也能驱动别的 agent"),跑在你自己进程/基础设施里,agent loop、上下文管理、工具都给你包好。

| 维度 | Claude Agent SDK | 主流编排框架(LangGraph / OpenAI Agents SDK / AutoGen / CrewAI / Pydantic AI) | 自建 |
|---|---|---|---|
| agent loop | 内置、生产级、**与模型协同调优** | 给积木(节点/状态机/工具抽象),loop 和 prompt 自己拼 | 从零写 |
| 上下文压缩/续跑 | 内置(JSONL resume、compaction、context editing) | 部分有,需配置 | 自己实现 |
| 工具 / MCP | 原生 MCP、in-process 函数工具、文件/bash 工具 | 看框架 | 自己接 |
| 权限 / hooks | 内置权限模式 + 生命周期 hooks | 一般没有 | 自己做 |
| 多模型可移植 | **弱**(Claude 优先,第三方二等公民、降级) | **强**(模型中立) | 看你怎么写 |
| 上手成本 | 低(约定优于配置) | 中 | 高 |
| 最适合 | Claude 驱动的编码 / 文件系统 / 生产自动化 agent | 需要精细图控制、多模型混排 | 有特殊架构需求 |

**vs 自建**:你要从零实现并调优的——工具调用循环、上下文压缩、权限门控、MCP 接入、子 agent 编排、文件编辑工具、会话续跑——恰好是难做对、且 Anthropic 在 Claude Code 上磨了很久的部分。除非有特殊架构诉求,自建性价比低。

**vs 主流编排框架**:它们大多是**模型中立的编排层**,给你积木让你自己拼 loop 和 prompt;Claude Agent SDK 给的是**开箱即用、与模型一起调优的完整 harness**。最大差异是 harness 和模型协同调优(系统提示、上下文策略都为 Claude 量身定)。**代价是可移植性弱**——要真正多模型、按节点选模型、或拿非 Claude 当主力,LangGraph + LiteLLM 这类更顺。

**选型建议:**
- 主力是 Claude、做编码/文件系统类 agent、想快上生产 → **Claude Agent SDK 最省事**。
- 想用 DeepSeek 降本跑跑、不依赖 Claude 专属特性 → 走 base_url 接 DeepSeek 可以,但当**实验/降本档**,别当"完美替代"。
- 要多模型混排、按步骤换模型、或把开源模型当主力 → **LangGraph / OpenAI Agents SDK + LiteLLM** 这类中立框架更合适。

---

**参考**(便于你自己核):DeepSeek Anthropic API 文档 `api-docs.deepseek.com/guides/anthropic_api`、Claude Code 环境变量 `code.claude.com/docs/en/env-vars`、Agent SDK overview `code.claude.com/docs/en/agent-sdk/overview`、LiteLLM 接 Claude Code 非 Anthropic 模型教程。

需要的话,我可以给你写一个**最小可跑的 Python Agent SDK 例子**,分别指向 Claude 原生 和 DeepSeek 兼容端点,你直接对比两边的工具调用/缓存表现。

---

### Turn 2  *(2026-06-09 06:26)*

**User:**

网上有没有deepseek兼容的测试？生产代码要放到服务器上，但服务器上没有vpn，连不上claude模型啊

**Assistant:**

好问题,而且这两件事其实是连在一起的——你服务器连不上 Claude,正好是该认真看 DeepSeek(及国产兼容端点)实测的理由。让我查一下当前的兼容性实测,以及"无 VPN 服务器"这条路的具体坑。

有,而且测试不少;更关键的是——**你"服务器连不上 Claude"这件事,正好被接 DeepSeek 这一招一起解决了**,因为 DeepSeek 的 Anthropic 兼容端点在国内不用 VPN 就能直连。下面分开说。

## 1. 兼容性测试:结论速览

| 来源 / 场景 | 结论 |
|---|---|
| **fundaai 38 任务横评** | V4-Pro 加权 8.27,和 Opus 4.7 同一档但略低(Opus 4.7=8.72);输出更省 token、废话少 |
| **kilo.ai FlowGraph(20 端点基建任务,在 Kilo CLI 里跑 agent loop)** | V4-Pro 夹在 Opus 4.7 和 Kimi K2.6 之间;**工具调用比预期稳**——会先读文件再改、在合理点跑测试、不卡死在重试循环。V4-Flash 偏弱 |
| **SWE-bench Verified** | V4-Pro 80.6% vs Opus 4.8 88.6%(约 8 分差距);但降价后便宜约 29× |
| **verdent.ai 的 Claude Code + V4 实测指南** | 明确列出**更弱项**:复杂架构推理、长会话指令一致性、Claude Code 工具循环的边角 case;**完全缺失**:`/ultrareview`、task budgets、xhigh effort、Skill 签名等一切依赖 Anthropic 后端的功能 |
| **NVIDIA NIM 论坛(一个真实可复现 bug)** | V4-Pro/Flash 经 **NIM 的 OpenAI 兼容端点 + 自建代理**时,工具调用发出但 agent 不继续到 tool_result;同样环境换 MiniMax 就正常 |

**两个能直接拿走的判断:**
- **别自己搭 OpenAI→Anthropic 的转译代理**(那个 NIM bug 就是栽在这)。用 **DeepSeek 官方的 `/anthropic` 原生端点**——它是 DeepSeek 自己维护、专门对 Claude Code 调的,保真度明显更高。
- 社区共识很一致:**脏活累活(后端逻辑、单测、批处理、脚本、代码审查)交给 DeepSeek 完全够;视觉/UX 设计、超长程复杂 agentic 还是 Claude 强。** 必须拿你自己的真实工作流跑一遍再定。

## 2. "服务器连不上 Claude"——不是问题,是该换思路

Agent SDK 认的是 **Anthropic Messages API 协议**,不是 `api.anthropic.com` 这个地址。所以**你根本不需要连上 Claude**:把 `ANTHROPIC_BASE_URL` 指向一个国内可直连的兼容端点即可。这些都**不用 VPN、国内直接解析**(fazm.ai 2026-05 实测确认):

- **DeepSeek** `https://api.deepseek.com/anthropic` —— V 系列
- **智谱 GLM(z.ai)** —— GLM-5 系列,主打 Claude Code 平替
- **Kim(Moonshot)** —— K2.5,注意有 `.ai` 和 `.cn` 两套域名
- **通义 Qwen(阿里云百炼 Coding Plan)** —— Qwen + 还能转 GLM/Kimi

一招解决三件事:**不用 Anthropic 账号/境外信用卡 + 不用 VPN + 便宜十几到几十倍**。

## 3. 服务器上落地的几个坑(这些不踩会卡住)

把下面这套设成进程环境变量(SDK 直接读 env;CLI 也读):

```bash
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic   # 末尾不要加斜杠
ANTHROPIC_AUTH_TOKEN=<你的 DeepSeek Key>
ANTHROPIC_API_KEY=                       # 必须置空,否则和 AUTH_TOKEN 抢,报 401
ANTHROPIC_MODEL=deepseek-v4-pro[1m]      # [1m] = 1M 上下文
ANTHROPIC_DEFAULT_OPUS_MODEL=deepseek-v4-pro[1m]
ANTHROPIC_DEFAULT_SONNET_MODEL=deepseek-v4-pro[1m]
ANTHROPIC_DEFAULT_HAIKU_MODEL=deepseek-v4-flash
CLAUDE_CODE_SUBAGENT_MODEL=deepseek-v4-flash
CLAUDE_CODE_EFFORT_LEVEL=high            # max 会映射到 DeepSeek 思考模式,更费 token
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1   # 关键:掐掉发往 Anthropic/Statsig 的遥测/特性开关心跳
DISABLE_AUTOUPDATER=1                     # 服务器上别让它自己联网更新
API_TIMEOUT_MS=600000
```

额外两个**服务器专属坑**:
- **首次启动会 ping `api.anthropic.com`**(交互式 onboarding 检查),连不上就转圈。CLI 场景在 `~/.claude/settings.json` 顶层加 `"hasCompletedOnboarding": true` 跳过(对应 issue #36998/#26935)。headless 跑 SDK 的 `query()` 一般不触发,但配上更稳。
- **装包**:`npm install -g @anthropic-ai/claude-agent-sdk`(或 `claude-code`)走淘宝镜像 `--registry=https://registry.npmmirror.com` 更快,npmjs 国内能连但不稳。

## 4. 如果是"把 Agent SDK 嵌进生产服务"——这几件事先想清楚

你说"生产代码放服务器上",我理解是 **agent 作为产品运行时组件**上线,不只是远程敲代码。那么有几点不是配置能盖住的:

- **Anthropic 服务端工具不支持**:DeepSeek 兼容表里 `code_execution`、`container_upload`、原生 MCP 连接器、`thinking` 字段都是 Ignored/Not Supported。**但**——SDK 侧的 MCP / 自定义工具 / bash / 文件编辑工具**照常工作**(它们跑在你自己进程里,对模型只是普通 `tool_use`)。受影响的只是"由模型 API 在 Anthropic 那侧执行"的那类工具。(DeepSeek 倒是单独特判支持了 Claude Code 的 Web Search。)
- **工具循环可靠性低于 Claude**(测试一致),长程多步任务尤甚——生产里要加重试/校验/护栏,别假设和 Claude 一样稳。
- **数据出境**:你的代码、prompt、工具输出会经 DeepSeek 基础设施。你做的是润滑油 B2B(中越)、库里还有客户/流失数据——**生产代码+客户数据喂给第三方 API 这件事得先过合规这关**。要彻底不出境,只能自托管(V4 是 MIT 开源权重,但 1T 参数要重型 GPU;更实际是自托管 Qwen/GLM 中小模型 + vLLM 的 Anthropic 兼容 shim)。
- **别"两头不讨好"**:Agent SDK 是为 Claude 协同调优的重型 harness。拿它喂 DeepSeek = 付出 harness 对 Claude 的假设,却拿不到 Claude。如果只是想在国内服务器跑一个产线 agent,也可以考虑**直接用 DeepSeek 原生 SDK + 轻量自写 loop**,或 **LangGraph/中立框架 + DeepSeek**,可移植性更好。选 Agent SDK 的理由应当是"就是要它那套现成的上下文管理/子 agent/工具编排",而不是惯性。

## 我的建议

1. 先用 DeepSeek 原生 `/anthropic` 端点 + 上面那套 env,在服务器上跑通一个**你真实工作流的小样本**(比如那个流失客户跟进的脚本/SQL 类任务),量一下工具调用成功率和结果质量。
2. 走得通就上;走不顺先试 **GLM-5**(智谱对 Claude Code 兼容调得很积极,工具调用历史口碑好)。
3. 涉及客户数据的链路单独评估出境合规,必要时这部分自托管。

需要的话我可以给你:① 一个**最小 Python Agent SDK 例子**,同一份 agent 代码切 DeepSeek/GLM 两个端点对比;或 ② 一个**冒烟测试脚本**,自动在你的服务器上验证端点连通 + 工具循环是否完整跑通(就是去复现 NIM 那类"工具调用断在半路"的问题)。要哪个?

---
