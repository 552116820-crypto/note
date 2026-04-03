查了下官方资料，OpenClaw 底层主要用的是 Pi 这套 agent/runtime 生态，不是常见的 LangChain、CrewAI、AutoGen 那一路。

更准确地说：

OpenClaw = 自己的 Gateway / 多渠道 / 会话 / 工具编排层 + Pi 作为底层 agent runtime

官方文档里有几处关键信息：

- Agent Runtime 写的是：OpenClaw 的运行时“derived from pi-mono”
- Pi Integration Architecture 写的是：它集成了 pi-coding-agent、pi-ai、pi-agent-core、pi-tui
- 首页和 README 也反复提到 Pi agent runtime / AI coding agents like Pi

所以如果你问“OpenClaw 用的什么 agent 框架”，最简短的回答就是：

它主要是基于 Pi（pi-mono / pi-agent-core / pi-coding-agent）这套框架，再加上 OpenClaw 自己封装的一层平台能力。



这里的 Pi，可以理解成一套偏“底层 + 可扩展”的 AI agent 工具栈，不只是一个单独的聊天程序。

更准确地说，Pi 指的是 pi-mono 这一整套项目，主要分几层：

- pi-ai：统一接 OpenAI、Anthropic、Google 等模型，处理流式输出、tool calling、模型适配
- pi-agent-core：agent 的核心循环，负责“调用模型 -> 执行工具 -> 把结果喂回模型 -> 继续跑”
- pi-coding-agent：面向用户的编码 agent，带内置工具，比如 read、write、edit、bash，还有 session、compaction、skills、extensions
- pi-tui：终端界面层

所以一句话概括：

Pi = 一套用来构建和运行 AI coding agent 的分层框架 / runtime

它的风格和很多“大而全”的 agent 框架不太一样，官方强调的是：

- minimal：尽量轻
- hackable：容易改、容易嵌入
- terminal-first：以终端工作流为中心
- extensible：靠扩展、技能、模板、包来长功能

跟 OpenClaw 的关系也可以这么理解：

OpenClaw 不是直接把原版 pi 命令行原封不动跑起来，而是嵌入了 Pi 的一些核心能力来当 agent runtime，用 Pi 的模型层和 agent 层，但会自己接管一部分 session 管理、工具接线和消息网关逻辑。