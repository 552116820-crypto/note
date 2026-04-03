• 不是。

  更准确地说，Ruflo 当前是用 CLI + hooks + MCP server 这三层在增强“agent 运行时/开发工作流”，不是只能以 MCP 一种形态存
  在。

  - CLI：可以直接跑命令，比如 init、agent spawn、hive-mind spawn，不接 MCP 也能用。
  - hooks：README 明确写了，init 之后可以“正常使用 Claude Code”，由 hooks 在后台做路由、记忆和协作。
  - MCP：这是它接入 Claude Code 或 Codex CLI 的主要接口，让客户端把 Ruflo 的能力当工具来调用。

  但如果你问的是“它能不能直接增强底层模型本身”，那答案基本也是：不能直接改模型能力本体。它更像外部 orchestration layer，
  而不是模型微调层。它做的是这些外围增强：

  - 任务路由
  - 多 agent 协调
  - 记忆检索
  - hooks 自动化
  - 多模型/多 provider 选择
  - 对简单任务直接走 WASM/规则处理，绕过 LLM

  所以可以这样理解：

  底层模型能力：Claude / GPT / Gemini 自己的事
  Ruflo 做的事：在模型外面加一层 runtime / orchestration / memory
  MCP 的作用：把这层能力接进 Claude Code / Codex

  对你这种团队里的 AI 工程师，这也正是它“价值有限”的原因：
  它更像 IDE/agent client 增强层，不是你拿来定义生产 agent 架构的主框架。你不用它的 MCP，也能用它的 CLI/hook；但如果你想
  让 Claude Code 直接“会用 Ruflo”，那 MCP 基本就是它的标准接入面。


