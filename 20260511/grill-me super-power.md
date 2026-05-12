# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `85766f61-08ed-4a53-8d0c-d1d6ec964b87`
- **范围**: Turn 1-3
- **导出时间**: 2026-05-11 14:59:52

---

### Turn 1  *(2026-05-11 06:01)*

**User:**

superpower和grill-me这两个skill

**Assistant:**

这两个 skill 都不在当前已加载的 skill 列表里。当前可用的是：skill-vetter、agentic-harness-patterns-zh、frontend-design、update-config、keybindings-help、simplify、fewer-permission-prompts、loop、schedule、claude-api、init、review、security-review。

`superpower` 和 `grill-me` 是社区里比较出名的第三方 skill：

- **superpower**（obra/superpowers）：一套元 skill 集合，给 Claude Code 加上 brainstorming、planning、TDD、debugging 等"超能力"，以及一个动态加载其他 skill 的机制。
- **grill-me**：通过苏格拉底式追问帮你"压力测试"想法/计划/代码的 skill，作者也是 obra。

你是想：

1. 安装它们到本地？（一般通过 `git clone` 到 `~/.claude/skills/` 或用插件市场）
2. 了解它们具体做什么、怎么用？
3. 在安装前用 `skill-vetter` 审一遍安全性？

告诉我方向，我再帮你下一步。安装第三方 skill 之前我建议先跑 skill-vetter 过一遍。

---

### Turn 2  *(2026-05-11 06:56)*

**User:**

了解它们具体做什么、怎么用？

**Assistant:**

先纠正我上一条消息里的一个错误：**grill-me 的作者是 Matt Pocock，不是 obra**。

---

## 1. superpowers（obra/superpowers）

**它是什么**：一个"agentic skills 框架 + 软件开发方法论"。本质是给 Claude Code 装一整套相互组合的 skill，并通过初始指令让 agent 在合适时机自己调用它们。

**核心能力（20+ skills）**：
- `/brainstorm` — 写代码前先头脑风暴，逐条提问、列备选、分段确认设计
- `/write-plan` — 把想法落成结构化的实施计划
- `/execute-plan` — 让 agent 自主推进计划（作者声称常能连续自主工作几小时）
- TDD、调试、协作模式等工程实践 skill
- 内置 `skills-search` 工具，自动发现可用 skill

**怎么用**：
- 通过 Superpowers Marketplace 当作 Claude Code 插件安装：`/plugin marketplace add obra/superpowers-marketplace`，然后 `/plugin install superpowers`
- 装完后，遇到合适场景直接打 `/brainstorm`、`/write-plan` 等斜杠命令；或者让 agent 自己决定何时调用
- 相关仓库：`superpowers-skills`（社区可编辑版本）、`superpowers-lab`（实验性）、`superpowers-chrome`（浏览器控制）

**适合谁**：想让 Claude 走完整"先想清楚 → 写计划 → TDD 执行"流程，而不是一上来就开写的人。

---

## 2. grill-me（mattpocock/skills）

**它是什么**：一个"苏格拉底式追问"skill。你把想法/设计/计划丢给 Claude，它会**反复追问**直到决策树的每个分支都被走完，最后给你一份总结。

**核心行为**：
- 一问一答地推进，逼你把模糊的地方说清楚
- 当某个问题答案显然时，它会**直接推荐答案**，你说"yes"就行（这是后来的一次重要改进，否则太累）
- 重点是暴露依赖关系和隐含假设

**怎么用**：
- 安装：仓库是 `mattpocock/skills`，把 `skills/grill-me/` 拷进 `~/.claude/skills/`，或用插件市场
- 使用：当你有一份计划/架构方案想被"压一下"时，让 Claude 用 grill-me skill 来面试你
- 典型场景：架构决策、产品 roadmap、提交方案前自检

**适合谁**：方案自己写得差不多但拿不准、想被人挑刺的时候。和 superpowers 的 `/brainstorm` 有点像，但 grill-me 更"狠"、更聚焦于已有方案的压力测试。

---

## 一句话对比

- **superpowers** = 全套开发方法论（很多 skill 的集合）
- **grill-me** = 单一 skill，专注"逼你把方案想透"

两个可以**同时装**，grill-me 可以补 superpowers 在"质疑既有计划"上的位置。

要不要我用 `skill-vetter` 给这两个仓库都过一遍安全审查？第三方 skill 装之前过一遍比较稳。

Sources:
- [obra/superpowers (GitHub)](https://github.com/obra/superpowers)
- [obra/superpowers-marketplace (GitHub)](https://github.com/obra/superpowers-marketplace)
- [obra/superpowers-skills (GitHub)](https://github.com/obra/superpowers-skills)
- [obra/superpowers-lab (GitHub)](https://github.com/obra/superpowers-lab)
- [mattpocock/skills (GitHub)](https://github.com/mattpocock/skills)
- [My 'Grill Me' Skill Went Viral — Matt Pocock](https://www.aihero.dev/my-grill-me-skill-has-gone-viral)
- [Grill Me — Claude Code Marketplaces](https://claudemarketplaces.com/skills/mattpocock/skills/grill-me)

---

### Turn 3  *(2026-05-11 06:58)*

**User:**

这两个skill好用吗

**Assistant:**

我的判断（按场景说，不一概而论）：

## superpowers

**好的地方**：
- `/brainstorm` 和 `/write-plan` 确实能逼 Claude 在动手前先想清楚，对**陌生领域 / 较大改动**特别有用
- 作者 obra（Jesse Vincent）做事比较系统，skill 写法本身就是学习 prompt 工程的好材料
- 社区活跃，迭代频繁

**坑 / 不适合的场景**：
- **小任务会变重**。改个 bug、写个一次性脚本，它会让你先 brainstorm → plan，节奏被拖慢
- skill 多了之后，agent 会过度调用，主上下文被吃掉
- 方法论比较"硬"，如果你已经有自己的思考流程，可能会觉得碍事
- 装的是一整套，不是只挑一两个

**适合**：写中大型功能、不熟的代码区、需要让 agent 长时间自主工作的场景。

## grill-me

**好的地方**：
- **单一 skill，零负担**，不会污染其他工作流
- "推荐答案 + yes 就行"那次改进之后，体验明显更好（不会问到你烦）
- 真的能发现思维漏洞，特别是依赖关系和隐含假设

**坑**：
- 还是会比较啰嗦，**只适合"我有方案，但想被挑刺"的时刻**，不要拿它做日常对话
- 如果你方案本身很糙，它会问到崩溃；适合方案七八成熟时用
- 本质上 Claude 自己也能做这个事，只是 grill-me 把节奏和结构固化了

**适合**：架构决策、PR 提交前自检、考研复习时给自己出题压力测试（对你来说这点可能有用）。

---

## 我的建议

给你的画像（在职 AI 工程师 + 考研清华 AI），我会这么选：

1. **grill-me 值得装** — 轻量，对**工程方案评审**和**学习时检查理解深度**都直接有用
2. **superpowers 看你工作方式** — 如果你日常多是改既有代码 / 调模型 / 写小脚本，它的收益有限；如果你常做新模块设计，值得试

折中方案：先只装 grill-me 跑一周，确认风格能接受，再考虑要不要上 superpowers。两个**不冲突**，可以共存。

要的话我可以现在帮你：
- 用 skill-vetter 审 grill-me 仓库的安全性
- 或者直接给出安装步骤

---
