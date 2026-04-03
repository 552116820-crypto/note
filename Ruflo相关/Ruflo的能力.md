• Ruflo 可以先把它理解成一个“给 Claude Code 加一层多智能体编排/记忆/路由能力”的 CLI + MCP server + hooks。按它当前官方
  README，它已经从 claude-flow 更名为 Ruflo，定位不是单纯的 agent SDK，而是直接把 Claude Code / Codex 变成一个更强的协作
  式开发运行时。相对地，LangGraph 的定位是一个你自己写代码来构建 agent 工作流的“低层编排框架”。
  来源: Ruflo README (https://github.com/ruvnet/ruflo), Claude Code MCP 文档 (https://code.claude.com/docs/en/mcp), Lan
  gGraph README (https://github.com/langchain-ai/langgraph)

  怎么开始用 Ruflo
  先满足前置条件: Node.js 20+，并且先装好 Claude Code。Ruflo README 给的最简单启动方式是：

  npx ruflo@latest init

  如果你想全局安装也可以：

  npm install -g ruflo@latest
  ruflo init

  它的 README 还提到，init 之后一个常见用法是“继续正常用 Claude Code”，让 Ruflo 的 hooks 在后台做任务路由、记忆和协调。

  把 Ruflo 的 MCP 接进 Claude Code
  最稳妥的加法，建议显式写 stdio：

  claude mcp add --transport stdio --scope user ruflo -- npx -y ruflo@latest mcp start
  claude mcp list

  如果 Ruflo 需要自己直连模型提供商，再按它 README 提供环境变量，例如：

  claude mcp add --transport stdio --scope user \
    --env ANTHROPIC_API_KEY=你的key \
    ruflo -- npx -y ruflo@latest mcp start

  然后进 Claude Code 里用：

  /mcp

  看 server 是否在线。Claude Code 官方文档也说明了，MCP 工具现在默认支持按需搜索和延迟加载，所以像 Ruflo 这种工具很多的
  server，不会一上来就把上下文塞满。

  接进去以后能怎么用
  两种用法最实用。

  第一种，直接自然语言让 Claude 调 Ruflo 的工具。Ruflo README 里提到的核心 MCP 工具有这些：

  - swarm_init
  - agent_spawn
  - task_orchestrate
  - memory_search
  - memory_usage
  - repo_analyze
  - code_review

  你可以直接这样说：

  先用 Ruflo 做一次 memory_search，找和 auth 相关的历史模式；
  然后用 hierarchical 拓扑初始化一个 swarm；
  再把当前登录模块重构任务拆开，并给出执行顺序。

  或者：

  用 Ruflo 分析这个仓库的测试缺口，再给我一个修复计划。

  第二种，把它当“后台协调层”。也就是你不手动点名工具，只是正常让 Claude Code 干活，Ruflo 的 hooks 和记忆系统在背后参与路
  由、缓存和协作。

  实战上我会建议你再加一个限制，避免工具太多太杂。Ruflo README 提供了这两个环境变量：

  export CLAUDE_FLOW_TOOL_MODE=develop
  # 或
  export CLAUDE_FLOW_TOOL_GROUPS=implement,test,fix,memory

  这样更适合日常开发，不然全量工具面太大。

  它和 LangGraph 的区别
  最核心的一句：

  - Ruflo 更像“现成的 agent 编排运行时/插件层”，重点是增强 Claude Code 这类现成 AI coding client。
  - LangGraph 更像“你自己写 agent 系统的框架”，重点是用代码定义 graph、state、durable execution、human-in-the-loop，然后
    部署成你的应用。

  如果换成工程视角：

  - 你想“马上让 Claude Code 更会拆任务、记忆上下文、做多 agent 协调”，选 Ruflo。
  - 你想“自己开发一个可部署、可恢复、可审计的客服/工作流/业务 agent 系统”，选 LangGraph。
  - 两者也不冲突: 你完全可以用 LangGraph 做生产 agent 服务，再把那个服务暴露成 MCP；而 Ruflo 则继续服务你的本地开发流。

  顺手提醒一点，Claude 官方对第三方 MCP server 有安全提示: 只接你信任的 server，尤其是能拉取外部内容的，因为会有 prompt
  injection 风险。
  来源: Claude Code MCP 安全提示 (https://code.claude.com/docs/en/mcp)

  如果你要，我下一条可以直接给你一份：

  1. Windows/WSL 可直接复制执行的 Ruflo + Claude Code 安装步骤
  2. 或者一份 Ruflo vs LangGraph 的架构级对照图和选型建议



• 架构图
  可以先把两者放在不同层级看。

  Ruflo
  用户/开发者
    ↓
  Claude Code / Codex / Cursor 等 MCP Client
    ↓  (MCP)
  Ruflo Runtime
    ├─ CLI / MCP Server
    ├─ Hooks / Router / Swarm Coordination
    ├─ Agent roles / task orchestration
    ├─ Memory / pattern store / cache
    └─ LLM providers
    ↓
  主要结果：增强“现成 AI 编程客户端”的协作、记忆、路由能力

  LangGraph
  你的应用 / API / Worker / Agent Service
    ↓
  LangGraph Runtime
    ├─ StateGraph / Nodes / Edges
    ├─ Checkpointer / Persistence
    ├─ Human-in-the-loop / Interrupt / Resume
    ├─ Memory / State / Tasks
    └─ Model + Tools + Business logic
    ↓
  部署到服务环境 / LangSmith Deployment / 自建运行时
    ↓
  主要结果：构建“你自己的生产级 agent 系统”

  核心差异
  | 维度 | Ruflo | LangGraph |
  |---|---|---|
  | 产品形态 | 现成运行时 + MCP server + hooks | 低层编排框架 |
  | 入口 | 先接到 Claude Code/Codex，再让它帮你工作 | 先写代码定义 graph/state，再部署 |
  | 抽象中心 | agent/swarm/coordination | state/node/edge/checkpoint |
  | 编排方式 | 运行时路由 + 工具调用 + hooks 协调 | 显式图结构和状态转移 |
  | 状态持久化 | 偏“记忆/模式/上下文复用” | 偏“工作流状态可恢复、可重放” |
  | 人工介入 | 有，但不是它最核心卖点 | 是一等公民，支持 interrupt/resume |
  | 与 Claude Code 关系 | 原生增强 Claude Code 工作流 | 不是 Claude Code 扩展层，本质是应用框架 |
  | 适合对象 | 开发者工作台、AI coding 协作 | 业务系统、长期运行 agent、生产服务 |
  | 上手成本 | 较低，接 MCP 即可开始 | 较高，要自己设计 graph 和运行逻辑 |
  | 可控性 | 高于普通插件，但低于自己写框架 | 最高，状态机和流程都自己定义 |

  怎么选
  如果你的目标是这几类，结论很明确：

  - 想把 Claude Code 立刻变强，选 Ruflo
  - 想做“可部署、可恢复、可审计”的业务 agent，选 LangGraph
  - 想做本地研发协作、多 agent 拆任务、记住项目上下文，优先 Ruflo
  - 想做审批流、长任务、暂停后几小时/几天再恢复，优先 LangGraph
  - 想要最强可控性和确定性，优先 LangGraph
  - 想少写框架代码、先把现有 coding workflow 提效，优先 Ruflo

  一句话边界

  - Ruflo 是“给现有 AI 编程客户端加 orchestrator”
  - LangGraph 是“让你自己造 agent runtime”

  推荐决策
  如果你是个人开发者或小团队，主要场景是“写代码、改仓库、做 review、让 Claude Code 更像一个会协作的开发团队”，先上
  Ruflo。
  如果你是要给公司做正式 agent 产品，比如客服、审批、研究助理、后台异步任务、带人工确认的长流程，直接上 LangGraph，不要
  拿 IDE 增强工具代替生产编排框架。

  混合方案
  很多团队最终会是这套：

  开发阶段：Claude Code + Ruflo
  生产阶段：LangGraph Agent Service
  必要时：把 LangGraph 服务再暴露成 MCP，供 Claude Code 调用

  这样本地开发效率和线上可靠性都能拿到。

  一个务实提醒
  Ruflo README 里很多“自学习、提速、对比其他框架”的表述，更多是项目方自述；LangGraph 在“durable execution / human-in-
  the-loop / persistence”这些核心能力上，文档定义更清晰、工程边界也更稳。所以如果你是做架构评审，LangGraph 更像正式基础
  设施，Ruflo 更像开发工作台增强层。


• 是，通常这个判断没问题。

  对团队里的 AI 工程师来说，Ruflo 更像“给 Claude Code/Codex 加一层编排和协作增强”的工具，而不是你真正拿来做生产 agent 系
  统的底座。你真正关心的往往是这些：

  - 可控的状态机和流程定义
  - 可恢复执行、重试、人工审批
  - 评测、观测、trace、成本治理
  - 和业务系统、队列、数据库、权限体系集成
  - 可部署、可测试、可审计

  这些点上，LangGraph 这类框架通常更对口；Ruflo 的价值主要偏向本地开发工作流增强，而不是生产架构本身。换句话说：

  Ruflo 解决的是“怎么让 AI coding client 更会协作”
  LangGraph 解决的是“怎么把 agent 系统做成工程产品”

  所以如果你的角色是：

  - 设计团队的 agent 架构
  - 落地生产工作流
  - 做评测、监控、恢复、HITL、服务化部署

  那 Ruflo 很容易显得“锦上添花，但不是刚需”，甚至会多一层抽象和黑盒。

  它仍然可能有价值的场景只有少数几个：

  - 你要提升团队日常用 Claude Code 写代码的效率
  - 你想快速做一个本地多 agent 原型
  - 你要给非 AI 工程师提供一个开箱即用的 agent 协作层

  如果不是这些场景，你把精力放在 LangGraph、自定义 orchestration、eval/observability、MCP 工具层设计上，通常更划算。
