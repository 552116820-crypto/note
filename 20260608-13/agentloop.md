# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `92fd375b-b2c8-4767-8746-36c2d88de5de`
- **范围**: Turn 1-2
- **导出时间**: 2026-06-11 10:09:42

---

### Turn 1  *(2026-06-11 02:00)*

**User:**

你知道最近新出的概念吗，loop engineering和agentloop

**Assistant:**

这两个词听起来像是我知识截止（2026 年 1 月）前后才热起来的说法，我先搜一下确认现在社区里指的是什么，再给你讲。

知道，而且这俩正好是这两周（2026 年 6 月初）刚被吵热的，我刚搜完一圈，给你讲讲现在社区里的实际用法。

## Loop engineering —— 真·新概念

引爆点之一是 Addy Osmani 6 月 7 日的同名博文（[addyosmani.com/blog/loop-engineering](https://addyosmani.com/blog/loop-engineering)），他的一句话定义：**"Loop engineering 就是把'给 agent 写 prompt 的那个人'换掉——你改去设计那个替你 prompt 的系统。"**

关键是分清两层循环：

- **内环（agent loop / agentic loop）**：agent 每轮本来就在跑的"推理 → 调工具 → 观察结果 → 再推理"循环，也就是 ReAct 那套，2022 年就有，不新。Claude Code、Codex、Devin 的核心都是它。
- **外环（loop engineering 的对象）**：在内环之上再建一个系统——定时醒来、自己找活、派活给 agent、验收、记账、决定下一步，跨很多个 agent 会话自动运转，你不在驾驶座上。

谱系上它是这条线的最新一环：prompt engineering 调单次调用，context engineering 调一个上下文窗口，harness engineering 调 agent 运行的环境，loop engineering 再往上一层，调"环绕 harness 的外环"——Osmani 原话是 loop 在 harness 的"楼上一层"。常见构件：cron/定时器当心跳、git worktree 隔离并行 agent、sub-agent 把"干活的"和"验收的"分开、skills + memory 免得每次重新解释项目、MCP 连真实系统（issue tracker、数据库、Slack）。

反方声音也同步出现了：Langfuse 6 月 9 日发了 ["AI is eating the AI engineering loop"](https://langfuse.com/blog/2026-06-09-ai-is-eating-ai-engineering)，论点是整个环技术上都能闭合，但自动化超过"你还能为产出担保"的那个点，就开始量产 **agent slop**（agent 批量生产的劣质 agent 和产出）。Osmani 自己也警告 token 成本会失控——杠杆点上移了，不等于活变轻松了。

## AgentLoop —— 看你指哪个，这词撞名很严重

- **当概念用**（最可能，如果你是和 loop engineering 一起看到的）：就是上面说的内环 "the agent loop"，在 loop engineering 的文章里被当作基础概念反复引用。
- **当项目名用**：GitHub 上有个 AgentLoopAI org，README 自称 "AgentLoop (formerly AutoGen)"，新闻条目（Chi Wang、ICLR 2024 best paper）和当年 AutoGen 社区分支一致，看起来是原 AutoGen/AG2 谱系改的名。另外还有 agentloop.run（一个本地 orchestrator CLI，带 `/orchestrator run --infinite` 和 daemon 模式，本质就是个 loop engineering 工具）和至少两家无关的同名 SaaS。

## 和你已有认知的衔接

这其实就是你之前了解的"养 agent"那套玩法（本地 daemon + 持久记忆 + 自喂任务，OpenClaw/Hermes 那个圈子）拿到了一个体面的工程学命名。而且你天天在用的东西里已经有这些原语了：Claude Code 的 `/loop`、cron routine、后台 Workflow，就是官方提供的外环构件。

如果你是在某篇具体文章或推文里看到这两个词并列，丢个链接给我，我帮你对一下那个语境里 agentloop 到底指概念还是指某个产品。

---

### Turn 2  *(2026-06-11 02:05)*

**User:**

https://addyosmani.com/blog/loop-engineering这篇博文具体讲的什么，https://langfuse.com/blog/2026-06-09-ai-is-eating-ai-engineering反方声音又体现在哪里

**Assistant:**

我把两篇都完整读一遍再给你讲。

两篇都读完了。先说一个修正：Langfuse 那篇并不是冲着 Osmani 来的对台戏——它说的甚至不是同一个 loop——但它确实是对"全自动闭环"叙事的明确泼冷水。下面分开讲。

## Osmani《Loop Engineering》（6 月 7 日）具体讲了什么

**核心命题**就是开头那句加粗的话：loop engineering 是把"给 agent 写 prompt 的你"替换掉，你改去设计那个替你 prompt 的系统。loop 在他的定义里是一个"递归目标"：你定义目的，AI 迭代到完成为止。文章由两条引语撑起来：Peter Steinberger 的"你不该再 prompt coding agent 了，你该设计 prompt 它们的 loop"，和 Boris Cherny（Anthropic 的 Claude Code 负责人）的"我不再 prompt Claude 了，我有一堆 loop 在跑着 prompt Claude、决定下一步干什么。我的工作是写 loop"。

他给的**定位**和我上次转述的一致：loop engineering 在 harness engineering 楼上一层——"还是那个 harness，但它跑在定时器上、会孵化小助手、会自己喂自己活"。

文章最有信息量的观察是：**这不再是自己写一堆 bash 维护到死的事，五个原语已经在 Codex app 和 Claude Code 里都产品化了，而且形状几乎一一对应**——所以别争用哪个工具，直接设计一个在哪个工具里都成立的 loop。五件套加一个记忆：

1. **Automations（心跳）**——定时自动跑发现和分诊。Codex 有 Automations 标签页加 Triage 收件箱（没发现东西的运行自动归档）；Claude Code 用 `/loop`、cron、hooks、GitHub Actions 到达同一个地方。他还单独强调了 `/goal`：一直跑到你写的可验证条件为真为止，且每轮由一个**独立的小模型**判断是否完成——写代码的不给自己打分。
2. **Worktrees（并行隔离）**——多 agent 一跑文件就互相踩，worktree 让每个 agent 有自己的检出。但他补了一句要害：机械碰撞解决了，**你仍然是天花板**——你的 review 带宽决定能跑几个，不是工具决定。
3. **Skills（项目知识固化）**——不写下来，loop 每个周期都从零重新猜你的项目；写下来，loop 开始复利。这是他之前"intent debt"论点的延续。
4. **Plugins / Connectors（MCP）**——区别在于"这是修复方案"和"自己开了 PR、关联了 Linear 工单、CI 绿了以后去 Slack 吱一声"。
5. **Sub-agents（maker 和 checker 分离）**——写代码的模型给自己的作业打分时太手软，必须用另一套指令（甚至另一个模型）来查。他说这是 loop 里**最有用的结构性设计**，因为 loop 在你不看着的时候跑，一个你真正信任的 verifier 是你敢走开的唯一理由。
6. 第六件是**记忆/状态**：一个 markdown 文件或 Linear 看板，活在单次对话之外，记录做完了什么、接下来是什么。"听起来蠢到不值一提"，但这是所有长期运行 agent 共同依赖的招——模型每次运行之间全忘光，所以记忆必须在磁盘上而不是上下文里。他的金句："The agent forgets, the repo doesn't."

然后他给了**一个完整 loop 的样子**：每天早晨一个 automation 跑起来，调用 triage skill 读昨天的 CI 失败、开放 issue、最近提交，把发现写进状态文件；值得做的发现各开一个 worktree，一个 sub-agent 起草修复，第二个 sub-agent 对照项目 skills 和现有测试来审；connector 负责开 PR、更新工单；loop 搞不定的落进收件箱等你。状态文件是脊柱，明早的运行从今天停的地方继续。"你只设计了一次，没有 prompt 其中任何一步。"

值得注意的是**文章后三分之一全在自我泼冷水**，他列了三个"loop 越好反而越尖锐"的问题：(1) 验证仍然在你头上——无人值守跑的 loop 也在无人值守地犯错，"done"是一个声称不是一个证明；(2) comprehension debt——loop 出货越快，"存在的代码"和"你实际理解的代码"差距越大；(3) cognitive surrender——loop 自己转起来后，你很容易不再有自己的判断。结尾的对照很狠：两个人搭一模一样的 loop，一个用它在自己深刻理解的工作上提速，另一个用它逃避理解，"loop 分不出区别，你分得出"。所以他说 loop design 比 prompt engineering **更难**而不是更容易——Cherny 的意思不是活变轻松了，是杠杆点挪了。最后一句："Build the loop. But build it like someone who intends to stay the engineer, not just the person who presses go."

## Langfuse 那篇的"反方"体现在哪

先把范围说清：它讨论的是 **AI engineering loop**——改进 AI 应用的那个环（trace 进来、监控、建数据集、做实验、评估、部署，循环往复），不是 Osmani 说的"coding agent 工作流外环"。它反对的也不是 loop 本身，而是**把整个环交出去、人完全退出**。

论证结构是"先承认、再划线"：

- **承认**：环里每一步现在技术上都不需要人了——读 trace 标异常 ✅、从生产错误建数据集 ✅、跑实验对着 eval 爬坡 ✅、新模型发布后自动适配并开 PR ✅。行业也确实在往这走，引了 swyx 的话：2026 年每家 evals/analytics 创业公司都在做"一次性的代际升级"，变成 continual learning 平台。Langfuse 自己说"方向上我们认为是对的"。
- **然后划线**：但把整个环交出去是错的。代价有名字——**agent slop**："由 agent 量产的低质量 agent，通常是 agent 对着不完美的 eval 和数据集做优化的结果。"机制在这句："agent 是顺着 evaluator 的梯度优化的，目标函数指错了，它会非常快地朝错误方向跑。"而"什么是好"的那些 nuance 活在你脑子里、还在持续演化——全自动化后 agent 优化的目标既不完整又会过期。他们还自曝了一个例子：对自家 Langfuse skill 跑 autoresearch 时就栽过这个跟头。
- **具体留给人的是哪一步**：手动读 trace。理由很精确——如果你只看 agent 或既有 evaluator 标记出来的 trace，你永远只看到"系统已经被告知要看的那个切片"，预料之外的行为全部漏掉；而且"你的 opinion 就是在这里形成的"，你读 trace 时留下的反馈正是后续喂给 agent 的方向。其余步骤（建数据集、跑实验、evaluator 自更新）默认自动化，但前提是 agent 拿到了你给的 context。

它的金句和 Osmani 殊途同归："**自动化越过'你还能为产出担保'的那个点，你就在出货 agent slop**"，以及"在你的判断就是产品的地方，把自己留在环里"。

## 合起来看

这两篇相隔两天，结论其实是同向的——Osmani 的 "stay the engineer" 和 Langfuse 的 "your judgment is the product" 是一回事。真正的"正方"是行业里"全自动闭环 / continual learning"的最大化叙事，两篇分别从两侧给它划边界：Osmani 从 coding agent 工作流侧，留给人的是 review 和对代码的理解；Langfuse 从评估改进环侧，留给人的是读 trace 和定义"什么是好"。所以我上一轮说 Langfuse 是"反方声音"，更准确的说法是：它是对 loop 全闭合的反方，不是对 loop engineering 的反方——它甚至默认你应该把环里大部分机械活都自动化掉。

---
