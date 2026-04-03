• 是，核心思想非常像，但不能简化成“只是把类和方法封装了一下”。

  更准确地说，这些框架大多都是把你那个 prototype 里的几个基础部件做成了有运行时语义的通用系统：

  - skill registry -> 技能/工具/子代理/工作流注册
  - router -> 路由、规划、handoff、图分支
  - prompt assembler -> system/instructions/context/history 组装
  - runtime state -> memory、checkpoint、message history、thread state
  - tool executor -> schema 生成、调用、重试、审批、错误处理
  - references/RAG -> retriever、query engine、knowledge source

  所以结论是：

  - 底层结构和自建 agent 很像
  - 很多框架本质上就是这套结构的产品化实现
  - 但它们通常还多了一层真正重要的东西：runtime semantics
      - 状态持久化
      - 恢复/重放
      - 并发和事件流
      - human-in-the-loop
      - tracing / eval / deployment
      - typed tools / structured output / middleware

  这些不是“语法糖”，而是执行模型本身。

  按框架看

  - LangGraph
      - 最接近“你自己写 agent runtime”。
      - 官方定位就是底层 orchestration，强调 durable execution、streaming、human-in-the-loop、memory。
      - 你可以把它理解成：你那个 prototype 的 runtime/store/router 被做成了通用图执行引擎。
      - 来源：
          - https://docs.langchain.com/oss/python/langgraph/overview
          - https://docs.langchain.com/oss/python/langgraph/durable-execution
  - LangChain
      - 比 LangGraph 更高层。
      - 官方文档明确说 create_agent 构建的是基于 LangGraph 的 graph runtime，而且支持 tools、middleware、state。
      - 所以 LangChain 更像“帮你把常见 agent loop 先封好了”，你不一定要自己写 router/runtime。
      - 来源：
          - https://docs.langchain.com/oss/python/langchain/overview
          - https://docs.langchain.com/oss/python/langchain/agents
  - AutoGen
      - 很能说明这个问题。
      - 官方 Core 文档直接说：Core API 是极简的，你需要自己围绕 model clients 和 tools 构建 agent logic。
      - 也就是说，AutoGen Core 本来就鼓励你写自己的 runtime；而 AgentChat 是它的高层封装。
      - 来源：
          - https://microsoft.github.io/autogen/dev/user-guide/core-user-guide/components/tools.html
          - https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/agents.html
  - LlamaIndex
      - 它把这套东西分成 agents、tools、workflows、query engines。
      - 官方 workflow 是 event-driven、step-based；AgentWorkflow 又把 agent、state、tool-calling 预封装了一层。
      - 所以它和你的 prototype 也很像，只是更强调数据/RAG/查询引擎那一侧。
      - 来源：
          - https://developers.llamaindex.ai/python/llamaagents/workflows/
          - https://developers.llamaindex.ai/python/framework/understanding/agent/multi_agent/
  - PydanticAI
      - 如果你说的 pi 是 PydanticAI，那它很像“类型安全版的自建 agent”。
      - 它把 instructions、tools、dependencies、message history、history processors 做成强类型抽象，还支持 durable agents、tool approval。
      - 这很接近你 prototype 里的 prompt assembler + runtime state，只是工程化更强。
      - 来源：
          - https://ai.pydantic.dev/
          - https://ai.pydantic.dev/message-history/

  “skill” 这件事本身也不完全一样

  - 这些框架大多没有像 Codex 那样统一的 SKILL.md 一等概念。
  - 在这些框架里，“skill”通常会被拆成几部分：
      - instructions/system prompt
      - tools
      - retrievers/knowledge sources
      - sub-agents
      - workflow nodes / graph branches
  - 所以你写的那个 SKILL.md，在这些框架里通常不会原样直接成为一个“skill 对象”，而是会被拆进 agent 配置和 runtime。

  最接近真实情况的一句话

  - 这些框架的底层确实和自建 agent 框架同构。
  - 区别主要不在“有没有那几个类”，而在它们把 runtime、state、tooling、恢复、观测这些难点做到了什么程度。

  我的判断里，LangGraph 和 AutoGen Core 最像“把自建 runtime 系统化”；LangChain、AutoGen AgentChat、LlamaIndex、PydanticAI 更像在这之上再加一层更高阶抽象。

  如果你要，我可以下一步直接给你画一张：

  自建 prototype -> LangGraph / LangChain / AutoGen / LlamaIndex / PydanticAI 的一一映射表。



• 按我在 2026-03-12 查的官方文档，结论是：

  是，同构。
  你做的 prototype 和这些框架的核心结构很像。

  但不只是“把类和方法封装了一下”。
  真正拉开差距的是它们把这些部件做成了正式 runtime：状态持久化、恢复、流式事件、人审、分布式执行、观测、typed tools、tool approval 等。

  以下把你说的 pi 按 PydanticAI 理解。这个映射表里，“skill”都是概念映射，不是说这些框架都原生有 SKILL.md 这种对象。

  | 你的 prototype | LangGraph | LangChain | AutoGen | LlamaIndex | PydanticAI |
  |---|---|---|---|---|---|
  | skill registry | 通常自己维护；技能更像 subgraph、prompt 片段、tools 配置 | 通常自己维护；高层 agent 配置 + middleware + tools | Core 里自己维护；AgentChat 里更像 agent/teams 注册 | 通常映射成
  AgentWorkflow、sub-agents、tools | 最接近 Agent + toolsets + instructions；还可接第三方 Agent Skills |
  | router.py | 图里的分支/边/节点决策 | create_agent + middleware + dynamic model/tool routing | Core 里自己写 agent logic；AgentChat 提供 teams/patterns | AgentWorkflow / orchestrator / custom planner | 通过
  instructions、toolsets、history processors、上层编排来做 |
  | prompt_assembler.py | 你自己拼 prompt 或放到各节点 | 框架帮你拼一部分，middleware 还能改请求 | Core 自己拼；AgentChat 预置 agent 帮你拼 | agent prompt + workflow state + tools | Agent(instructions=...) +
  history processors + toolsets |
  | runtime_state.py | LangGraph state + checkpointer | LangGraph state/persistence 被高层封装 | Agent runtime / message state / lifecycle | Workflow Context / workflow state | message history + RunContext +
  durable execution |
  | tools | 节点/任务里调用工具 | 一等公民，agent loop 原生支持 | Core tools / AgentChat tools | FunctionAgent tools / sub-agent tools | tools / toolsets / built-in tools / MCP |
  | references / RAG | 你自己接 retriever 节点 | 常和 retriever/tool 一起用 | 自己接知识源或工具 | 这块最强，天然偏 data/RAG/workflow | 可走 tools/toolsets/RAG 方案，但不以 RAG 为中心 |
  | 长任务/恢复 | 强，LangGraph 的主卖点之一 | 继承 LangGraph 能力 | Core runtime 很强，且可分布式 | Workflow 有状态、事件流、可扩展 | 官方有 durable execution |
  | human-in-the-loop | 强 | 强 | 有 | 有 | 有 |
  | 和你 prototype 的相似度 | 最高 | 高 | 高 | 高 | 中高 |

  一句话理解每个框架

  - LangGraph：最像“把你的 prototype 做成通用执行引擎”。
  - LangChain：最像“把 LangGraph 再包装成更好用的 agent API”。
  - AutoGen Core：最像“消息驱动/运行时化的自建 multi-agent 系统”。
  - LlamaIndex：最像“数据/RAG 友好的 workflow + agent 编排系统”。
  - PydanticAI：最像“强类型、工具化、可持久化的 agent runtime”；而且官方文档里明确列了第三方 Agent Skills 支持。

  你那套 prototype 和这些框架真正共享的底层骨架

  request
  -> route
  -> assemble instructions/context
  -> call model
  -> call tools / sub-agents
  -> update state
  -> loop or stop

  真正不是“仅仅封装类方法”的部分

  - 运行时状态模型
  - 持久化与恢复
  - 并发/事件流/分布式
  - 工具 schema、校验、审批
  - 中断与继续执行
  - tracing / observability
  - 多 agent handoff 语义

  一个更准确的判断

  - 如果你自己继续做下去，确实会逐步长成这些框架的样子。
  - 这些框架本质上就是把“自建 agent 架构”产品化、稳定化、可运维化了。
  - 差别主要不在“有没有 registry/router/store”，而在它们把这些东西做到了多深。

  我建议你怎么选

  - 想保留最大控制权：LangGraph 或 AutoGen Core
  - 想快速做单/多 agent：LangChain 或 AutoGen AgentChat
  - 强依赖 RAG / 文档 / data workflow：LlamaIndex
  - 强调 Python 类型、安全工具调用、durable agent：PydanticAI