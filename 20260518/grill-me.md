● 我的评价基本和你一致，再补一点角度：

  它定位本身就尴尬。 "反问到达成共识"这个目标，对一个已经会自己罗列决策树的用户来说是负价值——你脑子里已经走过那些分支了
  ，它再把分支念一遍只是延长对话。它真正能服务的是那种"我有个模糊想法，帮我问出来"的人，但那类用户通常也不知道自己要
  /grill-me。受众和能力错位。

  问题深度受限于通用 prompt。 Skill
  本质是塞进上下文的一段说明，没有领域知识注入。所以它问出来的永远是"你考虑过失败模式吗""边界条件是什么""谁是用户"这类通
  用清单——也就是你说的"我都想得到"。要问出让人停顿三秒的问题，得有领域上下文（代码、历史决策、约束），通用 skill
  给不了。

  它和 Claude 默认行为重叠。 不装这个
  skill，你直接说"帮我把这个方案的薄弱点挑出来"或者"扮演反方质询我"，效果不会差太多，甚至更聚焦——因为你能在 prompt
  里直接给上下文，而不是让 skill 走它预设的流程。

  真正有用的"被质询"得是不对称的：审稿人比你更懂这块、或者掌握你不知道的事实/历史。/grill-me
  两边都不占，所以体感就是"陪聊"。

  删掉是对的。

---

❯ skill-vetter是什么

  Searched for 1 pattern, read 1 file, listed 1 directory (ctrl+o to expand)

● 简短版：它是一个安装新 skill 前做安全审计的清单 skill——读 SKILL.md
  frontmatter、看权限范围、扫红旗字符串（~/.ssh、curl|bash、base64、prompt injection 关键词）、查
  typosquat，最后吐出一个"SAFE/WARNING/DANGER/BLOCK"格式的报告。

  但有几点需要直说：

  1. 它面向的生态不是 Claude Code。 里面反复提到 "OpenClaw / ClawHub / UseClawPro"，这不是 Anthropic 官方的东西，也不是
  Claude Code 内置的 skill 分发渠道。Claude Code 的 skill 就是放在 ~/.claude/skills/ 或 .agents/skills/ 下的
  SKILL.md，没有什么 "ClawHub 中心仓库" 概念。所以这个 skill 的应用场景在你这套环境里基本不存在。

  2. 元数据里的 trust-score: 97、last-audited: 2026-02-01 是自吹的。 SKILL.md 的 frontmatter 谁都能写，"trust-score 97"
  不连接任何外部权威系统，只是 skill 作者自己声明的数字。一个声称要做安全审计的
  skill，自己用不可验证的信任分给自己背书，气味不太对。

  3. 你本地装了两份：
  - /home/huangch001002/.claude/skills/skill-vetter/SKILL.md
  - /mnt/c/Users/55211/.agents/skills/skill-vetter/SKILL.md

  （system-reminder 里 skill-vetter 出现两次就是这个原因。）

  4. 内容本身的实用性也偏弱。 和 grill-me 同一个毛病：清单是通用的、你看一眼 SKILL.md 自己也能过一遍——查 curl/wget/bash
  -i、看权限、防 typosquat，这些都是你装东西时本能会做的。它没有比如"和已知恶意 skill
  库做指纹比对"或"调用真实的扫描工具"这种带外部能力的步骤，纯靠 prompt 走流程。

  建议：两份都可以删。真要审 SKILL.md，你直接让我读那个文件、把可疑点列出来就行，不用走这个 skill
  的固定流程。要删的话告诉我，我两个都清掉。

---


❯ agentic-harness-patterns-zh，这个skill在哪？这个skill有什么用，作用大吗？

  Read 2 files (ctrl+o to expand)

● 位置

  /mnt/c/Users/55211/.agents/skills/agentic-harness-patterns-zh/

  结构：
  - SKILL.md（250 行，主索引）
  - references/ 下 9 个模式文档（每个 100 行左右）
  - 总约 1110 行 / 160K

  它在讲什么

  不是"让 agent 帮你干活"的工具型 skill，而是给工程师讲怎么造 agent 运行时。六章：

  1. 记忆系统 — 指令记忆 / 自动记忆 / 会话提取三层，两步保存不变式，互斥锁防重写
  2. Skills 系统 — 懒加载、元数据预算、隔离 vs 内联执行
  3. 工具与安全 — fail-closed 默认、按调用粒度的并发分类、多源权限管道
  4. 上下文工程 — 把上下文当预算（select/write/compress/isolate 四操作）
  5. 多 Agent 协调 — Coordinator / Fork / Swarm 三种委派模式
  6. 生命周期与可扩展性 — Hook、后台任务、Bootstrap 序列

  每节都是「痛点 → 黄金法则 → 工作原理 → 从哪开始 → 权衡 → 深入阅读」的结构，权衡是带正反两面的（"省 token
  但首次激活多一轮延迟"这种），不是空话。例子拿 Claude Code 自身当案例。

  评价：和前面两个 skill 是两回事

  前面 grill-me、skill-vetter 我都建议删了，原因是通用脚手架、可被默认行为替代、缺乏领域内容。这个不一样：

  - 真有信息密度。讲"按调用粒度而非按工具类型分类并发安全"、"协调者不能委派理解"这种点，是从生产 runtime 里提炼的，不是
  prompt 模板能凑出来的
  - 触发词加载，常驻成本接近零。它只在你聊 harness/记忆/权限/上下文工程这类话题时被拉进上下文，平时不占预算
  - 和你的方向对得上。你是 AI 工程师 + 在准备清华 AI 考研——这恰好是 LLM
  系统工程方向的核心内容，无论是工作还是科研路线都用得上

  作用大不大，分两种情况

  对你大——如果你之后会自己搭 agent / 多 agent 系统，或者要在简历/面试里讲 agent 架构，这就是一份"现成的术语表 + 设计
  checklist"，能让你输出的内容上一个台阶。考研复试或者实习面试聊 agent 题的时候，能说出"fail-closed 默认 +
  按调用粒度的并发分类"和"我会用 prompt 让 agent 做工具调用"完全是两个层次。

  对你小——如果你只是用 Claude Code
  当编程助手，从不研究它怎么搭起来的，那它顶多在你问设计问题时偶尔触发，留着也不亏，删了也无感。

  建议：留着。它是这次清理里唯一一个我觉得"装得有道理"的 skill。
