# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `69914ade-8691-41a1-a02b-46180e01afd0`
- **范围**: Turn 1-2
- **导出时间**: 2026-05-28 10:18:56

---

### Turn 1  *(2026-05-28 00:09)*

**User:**

steering和hook怎么搭配skill使用

**Assistant:**

我先确认一下 Claude Code 里的"steering"具体指什么——这词在官方文档里不算常用术语，可能你指的是 CLAUDE.md / output style / 还是别的。然后再去查 hook 和 skill 的协作模式。

# Steering / Hook / Skill 三者的协作

## 各自的"时机"和"地位"

| | 加载时机 | 谁执行 | 上下文成本 |
|---|---|---|---|
| **CLAUDE.md（steering）** | 每次会话开头**永久注入** | Claude 读后自行遵守 | 永远在占预算 |
| **Skill** | 触发词/语义匹配时**按需加载** SKILL.md | Claude 读后自行遵守 | 只在用到时占 |
| **Hook** | 生命周期事件（PreToolUse/PostToolUse/UserPromptSubmit/Stop/SessionStart/PreCompact 等） | **harness 确定性执行**，不经 Claude | 不占（除非主动注入文本） |

关键差异：CLAUDE.md 和 Skill 是"劝"，Hook 是"管"。前两者 Claude 可能误读或忽略，Hook 一定会跑。

## 三种典型搭配

### 1. CLAUDE.md 当"路由器"，Skill 当"程序"

CLAUDE.md 里只写一行：
```
做 X 类任务时，调用 /skill-x
```
Skill 的几百行 SOP 只在真用到时才进上下文。这是省 token 的标准做法——把"什么时候用"塞 CLAUDE.md，把"怎么做"塞 SKILL.md。

### 2. Hook 当 Skill 的"硬触发器"

Claude 自己路由 skill 不一定可靠。用 **UserPromptSubmit hook** 扫用户输入关键词，向 Claude 注入一条 system-reminder：
```
检测到 X，请使用 /skill-x
```
比"靠 Claude 看 CLAUDE.md 联想"稳得多——尤其当你已经被它"忘了用某个 skill"坑过。

### 3. Hook 当 Skill 的"出口质检"

Skill 跑完写了文件，但 Claude 不一定真按 skill 要求做了。用 **PostToolUse hook** 在 Edit/Write 之后跑 lint/test/grep 校验。**Stop hook** 可以在 Claude 想结束时检查 checklist 没走完就拦回去再干。

这条对你尤其重要——你记忆里的 [[feedback_tool_status_verify]] 就是"声称 ≠ 事实"，hook 是把"声称"变"事实"的唯一确定手段。

## 哪种组合用在哪

- **个人偏好/风格**（终身有效，跨项目）→ CLAUDE.md
- **可复用的多步流程/领域知识**（按需）→ Skill
- **"每次/只要 X 就 Y"的自动行为** → **必须 Hook**（CLAUDE.md 和记忆都做不到强制，[[update-config]] skill 的描述里也强调了这点）

## 一个常见反模式

把"每次提交前跑 X"写进 CLAUDE.md/记忆 → Claude 会随机忘。
正解：写成 Stop hook 或 PostToolUse hook，由 harness 执行。

要不要我帮你把某个具体场景配出来（比如"每次写完 Python 文件自动 ruff"或者"用户说调研就强制走 [[research_workflow]] 那五个 MCP"）？

---

### Turn 2  *(2026-05-28 00:18)*

**User:**

"steering" 在 Claude Code 里还可以指哪一种？

**Assistant:**

# Claude Code 里 "steering" 还可能指什么

按"什么时候、谁来注入"分四类：

## 一、静态注入（启动/会话开头一次性）

- **CLAUDE.md**（上轮聊的）—— 项目级 + 用户级 + 仓库根，全部拼成系统提示
- **Auto memory / MEMORY.md** —— 你这套 `~/.claude/projects/.../memory/` 就是；它把 MEMORY.md 当索引每次注入
- **Spec/Steering docs（Kiro 派）** —— AWS Kiro 的 `product.md` / `tech.md` / `structure.md` 三件套；社区有人把这套搬进 Claude Code 当"产品规范钉死 agent 行为"用，常被直呼"steering files"
- **Output Style**（`/output-style`）—— Explanatory / Learning / 自定义；切的是输出人格，本质也是系统提示拼接

## 二、动态注入（每轮或按事件）

- **`<system-reminder>` 标签** —— harness 在每轮塞的提醒（auto memory 的注入、CLAUDE.md 重申、todo 状态、可用 skill 列表都走这个通道）
- **Hook 注入文本** —— UserPromptSubmit/SessionStart/PostToolUse 等 hook 把 stdout 回灌给 Claude，是**最强的动态 steering**（确定性 + 可基于事件）
- **Skill SKILL.md** —— 触发后注入，本质也是动态 steering，只是触发器是语义匹配而非事件

## 三、模式切换（meta-steering，改的是 agent 的工作模式）

- **Plan mode** —— 强制只研究不动手，直到 ExitPlanMode
- **Permission mode** —— `acceptEdits` / `plan` / `default`，约束的是动作而不是表达，但也算 steering 的一种
- **`/model` 切换** —— Opus/Sonnet/Haiku，间接改风格

## 四、子作用域 steering（只对某段工作流生效）

- **Subagent 的 system prompt** —— 每个 agent definition 里的 frontmatter + 正文，是该 subagent 的局部 CLAUDE.md
- **Slash command 的 prompt 模板** —— `/foo` 触发时拼进当前轮，相当于一次性 steering
- **Tool description** —— 工具的 description 字段是给 Claude 看"什么时候用我"的微型 steering（你的 [[research_workflow]] 五个 MCP 钦定人设那条记忆，本质就是在补 tool description 不够清楚的缺）

## 关键区分：Anthropic 官方语 vs 社区黑话

- **Anthropic 文档** —— "steering" 多指系统提示 / Constitutional AI 这类**模型层**引导
- **Agentic 社区**（Kiro / Cursor rules / Cline rules）—— "steering" 常特指**项目级规范文档**（接近你上轮选的 CLAUDE.md 含义，但更强调"规格而非偏好"）

所以下次有人聊 "steering files" 时，先看上下文是 Kiro 派的 product/tech/structure 三件套，还是泛指 CLAUDE.md/记忆/spec 这一整类。

---
