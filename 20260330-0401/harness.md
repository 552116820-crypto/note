• harness engineering 可以理解成：不是继续打磨“神提示词”，而是去设计一整套让 AI agent 稳定干活的运行环境。

  先纠正一个时间线。截止 2026-03-30，Anthropic/Claude 官方近几个月确实一直在讲这个方向，但更常用的说法其实是 agent harness、long-running agent harness，不完全是把 “harness engineering” 当成 Claude 独有的新品牌词。Anthropic 在 2025-11-26
  发了 Effective harnesses for long-running agents
  (https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)，2026-01-09 在 Demystifying evals for AI agents (https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) 里正式解释了 agent harness，2026-02-05 又用 Building a C compiler with a team of parallel Claudes
  (https://www.anthropic.com/engineering/building-c-compiler) 展示了并行 Claude 的 harness。完整标题直接叫 “Harness engineering” 的，公开最明确的是 OpenAI 2026-02-11 的 Harness engineering: leveraging Codex in an agent-first world
  (https://openai.com/index/harness-engineering/)。

  如果用一条主线来理解，大概是：

  - prompt engineering：把提示词写好
  - context engineering：把上下文、记忆、检索喂对
  - harness engineering：把 agent 周围整套系统搭对

  这里的 “harness” 本质上就是 agent 的脚手架/运行框架，通常包括：

  - 系统提示、CLAUDE.md、规则文件
  - 工具接口，比如 bash、浏览器、数据库、MCP
  - 记忆和跨会话交接，比如进度文件、git 提交、压缩总结
  - 验证回路，比如测试、eval、CI、自检
  - 隔离和权限，比如容器、沙箱、工作树
  - 多 agent 协作，比如任务拆分、锁、合并策略
  - 文档和知识库，让 agent 能自己找信息而不是靠人补上下文

  所以它为什么最近火？因为大家发现，agent 做不好复杂任务，很多时候不是模型“不够聪明”，而是外部环境没设计好。Anthropic 那
  篇长任务文章里，核心做法就很典型：第一个 session 先做初始化，生成 init.sh、进度文件、初始提交；后面的 Claude 每次只做
  增量推进，结束时必须留下清晰交接物。这样 agent 才能跨多个 context window 连续工作。Anthropic 后面做并行 Claude 编译
  器，也是同一思路：靠 harness 让多个 agent 分工、同步、验证，而不是只靠一次 prompt。

  一句话总结：harness engineering 关注的不是“怎么把模型说服”，而是“怎么把 agent 放进一个能持续产出正确结果的工程系统
  里”。

  如果你愿意，我下一条可以直接用一个具体例子讲：

  1. Claude Code 项目里的 harness 长什么样
  2. 普通开发者怎么给自己仓库做一个最小可用 harness

---

## Claude Code 视角下的 Harness Engineering（补充）

**Harness Engineering**（harness 工程）是 Anthropic 近期提出的概念，指的是**为 AI agent 构建运行环境和脚手架的工程实践**。

核心思想：模型本身的能力只是一部分，真正决定 agent 效能的是围绕模型搭建的**运行框架（harness）**——它赋予 AI "手、眼睛和工作空间"。

### 为什么需要它？

AI agent（如 Claude Code）面临几个关键限制：
- **上下文窗口有限** — 长任务会导致模型"遗忘"
- **自我评估偏差** — 模型倾向于给自己的输出打高分
- **跨会话状态丢失** — 每次新对话都从零开始

Harness engineering 就是解决这些问题的系统方法。

### 核心组件

| 组件 | 作用 |
|------|------|
| **CLAUDE.md 文件** | 项目级指令，定义编码规范、上下文、规则 |
| **Hooks（钩子）** | 在 agent 生命周期的关键节点自动执行动作（如 `PreToolUse`、`PostToolUse`、`SessionStart`） |
| **Memory（记忆系统）** | 跨会话持久化用户偏好、项目状态等 |
| **Skills（技能）** | 可复用的专业化 prompt，封装特定工作流 |
| **Subagents（子代理）** | 多 agent 架构，分工并行处理复杂任务 |
| **Progress tracking** | 通过文件和 git 提交记录进度，实现会话间交接 |

### 核心原则

1. **增量推进** — 不要试图一次完成所有事，每个会话完成一个可工作的增量
2. **干净的状态管理** — 每个会话结束时应有：可运行的代码 + git commit + 进度记录
3. **结构化知识传递** — 用文件（而非上下文）在会话间传递状态，弥补上下文窗口限制
4. **评估与执行分离** — 让专门的 agent 来测试和评审，避免"自己给自己打分"

### 类比理解

可以把它想象成**工厂的工装夹具**：

- 模型 = 工人的技能
- Harness = 工装夹具 + 操作流程 + 质检工位

再好的工人，没有合适的工装夹具也无法高效、精确地完成复杂工件。Harness engineering 就是为 AI agent 设计这套"工装"。

### 参考资料

Anthropic 发布了两篇深度工程博客：
- **Effective Harnesses for Long-Running Agents** — 讲如何为跨多个上下文窗口的 agent 设计架构
- **Harness Design for Long-Running Applications** — 讲多 agent 系统（planner/generator/evaluator）的结构化设计
- https://www.anthropic.com/engineering/harness-design-long-running-apps

这个概念本质上是在说：**用好 AI agent 不只是写好 prompt，更是工程问题**。
