• hooks 本质上就是“事件触发的回调/拦截器”。

  意思是：当某个固定时机发生时，系统自动执行你预先注册好的逻辑。
  很像：

  - git hooks 里的 pre-commit
  - Web 框架里的 middleware
  - IDE 插件里的 lifecycle callbacks

  放到 Claude Code 里，hooks 就是在这些时机自动运行的脚本、HTTP handler、prompt hook 或 agent hook，例如：

  - SessionStart：会话开始时
  - UserPromptSubmit：你发出提示词时
  - PreToolUse：工具调用前
  - PostToolUse：工具调用后
  - Stop：回答结束时
  - PreCompact：上下文压缩前

  所以一句话理解：

  hook = 在某个生命周期节点自动插进去的一段逻辑

  在 Ruflo 里，hooks 的作用通常是：

  - 自动做任务路由
  - 自动恢复/保存记忆
  - 自动分析编辑结果
  - 自动阻止上下文丢失
  - 自动给出模型或工具选择建议

  它不是“增强模型本体”，而是“在模型工作流周围加自动化”。

  一个直观例子：

  - 你让 Claude 改文件
  - PreToolUse hook 先判断这是不是简单改动
  - 如果是，Ruflo 提示“别走大模型，直接 Edit 更快”
  - 改完后 PostToolUse hook 再记录这次改动是否成功
  - 下次类似任务就能复用这个模式





• Ruflo 的 hooks 不是 shell 的 bash/zsh hooks，而是挂在 Claude Code 生命周期事件上的一组命令。
  它的用法本质上是：

  1. ruflo init 把 hook 配置和 helper 脚本装进项目
  2. Claude Code 在 SessionStart、PreToolUse、PostToolUse、UserPromptSubmit、PreCompact 这类事件发生时自动执行这些 hook
  3. Ruflo 用这些 hook 做路由、记忆、上下文归档、模式学习

  怎么用
  最常见的是自动模式，不是你每次手敲 hook。

  # 在项目里初始化
  npx ruflo@latest init

  初始化后，按 Ruflo README 的说法，它会把 hooks system 配到项目里，核心落点是：

  - .claude/settings.json
  - .claude/helpers/

  然后你就正常打开 Claude Code 干活。Ruflo hooks 会在后台自动触发，例如：

  - 开始任务前：分析复杂度、推荐 agent / model
  - 编辑文件前后：做校验、记录成功模式
  - 跑命令前后：记录风险和结果
  - 会话开始/结束：恢复和持久化上下文
  - 上下文快满时：做 compaction / archive / restore

  你平时怎么确认它在工作
  有三个最实用的入口：

  # 看学习/记忆/图谱统计
  node .claude/helpers/hook-handler.cjs stats

  # JSON 格式
  node .claude/helpers/hook-handler.cjs stats --json

  在 Claude Code 里还可以直接输入：

  /hooks

  官方 hooks 菜单会显示当前项目实际注册了哪些 hook、来自哪个 settings 文件。

  如果你想手动调试 hooks
  Ruflo 也提供 CLI 子命令，不一定非得等 Claude 自动触发：

  # 看有哪些 hooks
  npx ruflo@latest hooks list

  # 手动做一次任务路由
  npx ruflo@latest hooks route "refactor authentication to use JWT" --include-explanation

  # 模拟一次编辑前/后钩子
  npx ruflo@latest hooks pre-edit ./src/auth.ts
  npx ruflo@latest hooks post-edit ./src/auth.ts --success true --train-patterns

  # 手动管理会话
  npx ruflo@latest hooks session-start --session-id "feature-auth" --start-daemon
  npx ruflo@latest hooks session-end --export-metrics --persist-patterns

  它到底帮你做什么
  对 AI 工程师来说，hook 形态最核心的价值就四个：

  - 自动把任务路由到合适 agent / model
  - 自动记录“什么改法成功过”
  - 自动管理长上下文和归档恢复
  - 自动在编辑/命令生命周期里插入校验和学习

  边界
  如果不在 Claude Code 这类支持 hooks 的宿主里，Ruflo 的 hooks 不会“全局自动接管你的终端”。
  那时它只是一些你可以手动调用的 CLI 命令，或者你自己把它接到别的事件系统里。

  临时关闭
  Claude Code 官方支持在 settings 里关掉全部 hooks：

  {
    "disableAllHooks": true
  }
