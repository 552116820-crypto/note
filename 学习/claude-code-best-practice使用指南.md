# Claude Code Best Practice 使用指南

整理时间：2026-04-03  
来源：本次会话整理稿

---

## 会话记录

### 1. 用户提问

`https://github.com/shanraisshan/claude-code-best-practice这个网站讲了什么`

### 1.1 助手说明

我先看一下这个 GitHub 仓库的 README 和目录，确认它主要讲的内容，再给你一个简洁总结。

### 1.2 助手回答

这个 GitHub 仓库本质上是在做一件事：把 **Claude Code 的用法、最佳实践、工作流模板和社区经验** 系统整理成一个“实战手册”，不是单篇文章。仓库名就叫 *claude-code-best-practice*，核心目标是教你怎么把 Claude Code 用得更工程化、更高效。  
来源：<https://github.com/shanraisshan/claude-code-best-practice>

它主要讲这几类内容：

- **Claude Code 的核心能力**：subagents、commands、skills、hooks、MCP servers、plugins、settings、memory、checkpointing 这些分别是什么、放在哪、适合解决什么问题。
- **推荐开发流程**：把日常开发收敛成 `Research → Plan → Execute → Review → Ship`，并强调 `Command → Agent → Skill` 这种编排模式。
- **大量实操建议**：比如先做 plan mode、把常用流程写成 slash commands / skills、用 feature-specific subagents、用 hooks 自动格式化和权限处理、保持小 PR、常 squash merge、用截图和浏览器/MCP 做调试。
- **社区工作流对比**：它还汇总了多个流行 Claude Code 工作流/仓库，比较各自特色。
- **资料库性质很强**：README 里还整理了推文、文章、视频、报告，以及一些“Claude Code 可能替代哪些创业方向”的讨论。

一句话概括：**这是一个面向 Claude Code 重度用户/团队的“最佳实践导航站 + 示例仓库”**，重点不是讲 AI 原理，而是讲怎么把 Claude Code 接进真实的软件开发流程里。

如果你愿意，我可以继续帮你把这个仓库压缩成：

1. `10 条最值得抄的实践`
2. `适合个人开发者的用法`
3. `适合团队落地的用法`

---

### 2. 用户提问

`我愿意`

### 2.1 助手回答

基于这个仓库的 README，我会把它压缩成这三部分。先说结论：**最值得抄的不是某个单独技巧，而是“把 AI 使用方式产品化、流程化、可复用化”**。  
来源：<https://github.com/shanraisshan/claude-code-best-practice>

## 10 条最值得抄的实践

1. 先 `Plan` 再实现。不要一上来就让模型写代码，先让它把方案、阶段和验证方式讲清楚。
2. 给模型明确的验收标准。比如“证明它能工作”“先别提 PR，先过测试”。
3. 把高频提示词做成 `commands`。重复做的事不要每次重写 prompt。
4. 把复杂领域知识做成 `skills`。重点写约束、坑点和触发条件，不要写废话。
5. 用 feature-specific subagents，不要只建“后端工程师/QA”这种泛角色。
6. 主上下文保持干净。能丢给 subagent 的就丢，避免一个会话塞太多中间过程。
7. `CLAUDE.md` 要短、准、可执行。尤其要写清楚怎么安装、测试、构建、运行。
8. 小 PR、单功能 PR、常提交、最好 squash merge。这样 AI 和人都更容易 review。
9. 调试时尽量给模型“看见”真实环境。截图、控制台日志、浏览器自动化都比纯口述强。
10. 经常把成功模式固化。凡是一天会做两次以上的动作，都值得变成 command、skill 或 hook。

## 适合个人开发者的用法

- 最先落地的三样东西：`CLAUDE.md`、`/commands`、一个你常用的 `skill`。
- 最实用的个人工作流是：`描述问题 -> 让它先出计划 -> 让它执行 -> 让它自审 -> 你再看 diff`。
- 如果你经常修 bug，就做两个模板：`/bugfix-plan` 和 `/bugfix-review`。
- 如果你经常重构，就做一个 `/simplify` 或 `/techdebt` 命令。
- 不要一开始搭很重的 agent system。个人项目里，普通 Claude Code 加少量 commands，往往已经够强。
- 代码库要保持一致性。半迁移状态、两套框架并存、测试命令不统一，都会让模型更容易跑偏。

## 适合团队落地的用法

- 团队先统一最小规范：目录结构、构建命令、测试命令、提交流程，都写进 `CLAUDE.md` 或规则文件。
- 把团队共识变成工具，而不是靠口头提醒。比如格式化、权限、审查规则可以用 hooks 或 settings 固化。
- 给不同领域做专用 agent/skill，例如支付、前端验证、回归测试，而不是一个万能 agent。
- 把 review 前移。让 AI 在 PR 前先做一次多代理检查，重点抓回归、边界条件和安全问题。
- 鼓励小 PR 和频繁提交，否则上下文太大，AI 审查和人审查都会下降。
- 如果多人并行开发，可以用 agent teams + worktrees 这种思路，把上下文和代码修改范围隔离开。
- 最后把“最佳实践”当成活文档维护。模型、工具和工作流更新很快，半年不动就会过时。

这仓库还有一个要点值得提醒：**它是“社区最佳实践汇编”，不是纯官方规范**。所以最好的用法不是照搬全套，而是先挑 2 到 4 条放进你现在的项目里验证。

来源：

- GitHub 仓库 README：<https://github.com/shanraisshan/claude-code-best-practice>

如果你要，我下一条可以直接给你一份“适合你自己项目的最小落地清单”，只保留真正值得马上加的 5 项。

---

### 3. 用户提问

`我要`

### 3.1 助手回答

按一个通用代码仓库来做，先落地这 5 项就够了。目标不是“把 Claude Code 配满”，而是先把高频动作标准化。  
来源：<https://github.com/shanraisshan/claude-code-best-practice>

1. `CLAUDE.md`  
只写最关键的内容：
   - 项目怎么启动
   - 测试怎么跑
   - 构建怎么做
   - 提交前必须检查什么
   - 哪些目录/模块最容易出错  
建议控制在 60 到 200 行内。仓库里反复强调这类记忆文件要短、准、可执行。

2. 两个高频命令  
先做这两个就行：
   - `/plan`: 先出方案、拆阶段、列验证方式
   - `/review`: 按“回归风险、边界条件、测试缺口”审一遍  
这样你不用每次重写提示词，也能把流程固定下来。

3. 一个项目专用 skill  
不要一开始做通用“前端工程师”或“后端工程师”。  
直接做一个最贴你项目的，比如：
   - 支付改动检查
   - 表单提交流程验证
   - 数据库迁移注意事项
   - 登录鉴权改动检查  
skill 里重点写“坑点”和“约束”，不要写常识。

4. 固定开发节奏  
每次都按这条线走：
   - 先让它出 plan
   - 再让它实现
   - 再让它自测
   - 再让它自审 diff
   - 最后你看结果  
这个仓库的主线就是把开发流程收敛成 `Research → Plan → Execute → Review → Ship`。

5. 小 PR + 高频提交  
这是最容易被忽略、但实际收益最大的部分：
   - 一个 PR 只做一件事
   - 任务完成就提交
   - 尽量 squash merge  
这样 AI 更容易审，人也更容易回滚和定位问题。

如果你现在就要开始，建议顺序是：

1. 先写 `CLAUDE.md`
2. 再做 `/plan`
3. 再做 `/review`
4. 最后补 1 个 skill

如果你愿意，我下一条可以直接给你一份“可直接改”的最小版 `CLAUDE.md` 模板。

---

### 4. 用户提问

`给我写一份适合团队的`

### 4.1 助手回答

下面这份是给团队用的 `CLAUDE.md` 模板，重点是短、可执行、能统一团队和 AI 的工作方式。你可以直接放到仓库根目录，再把命令替换成你们自己的。

```md
# CLAUDE.md

## Project
- Project: <repo-name>
- Purpose: <一句话说明这个仓库解决什么问题>
- Primary stack: <例如 Next.js + Node.js + PostgreSQL>
- Package manager: <pnpm/npm/yarn/bun>
- Main apps/packages:
  - `<path-1>`: <作用>
  - `<path-2>`: <作用>
  - `<path-3>`: <作用>

## Source Of Truth
- Prefer existing project patterns over inventing new ones.
- Reuse existing utilities, components, hooks, and test helpers before adding new abstractions.
- If documentation and code disagree, trust code first and update docs in the same task.
- If a change affects API, DB schema, auth, billing, or infra, call it out explicitly in the plan.

## Setup
- Install: `<install-command>`
- Start dev: `<dev-command>`
- Build: `<build-command>`
- Run tests: `<test-command>`
- Run lint: `<lint-command>`
- Run typecheck: `<typecheck-command>`

## Required Checks Before Finishing
- Always run the smallest relevant verification first.
- For code changes, run all that apply:
  - `<lint-command>`
  - `<typecheck-command>`
  - `<test-command>`
- If you cannot run a required check, say exactly why.

## Team Workflow
- Start substantial work with a short plan.
- Keep changes focused. Do not mix unrelated refactors into the same task.
- Prefer small PRs with one feature or one fix per PR.
- Update tests when behavior changes.
- Update docs when setup, behavior, or public interfaces change.
- Summarize user-visible impact and technical risk in the final handoff.

## Coding Rules
- Follow the existing code style and naming in each folder.
- Prefer simple, local changes over broad abstractions.
- Do not introduce new dependencies unless necessary.
- Do not rename or move files unless it materially improves the task.
- Preserve backward compatibility unless the task explicitly allows breaking changes.

## Safety Rules
- Never delete production data, secrets handling, migrations, or env usage without explicit justification.
- Never hardcode secrets, tokens, or credentials.
- Never change CI, deployment, auth, billing, or database migration behavior silently.
- If a command is destructive or irreversible, stop and explain the risk first.

## Testing Guidance
- Add or update tests close to the changed behavior.
- Prefer regression tests for bug fixes.
- Prefer integration tests when multiple layers are involved.
- Do not mark work complete if the main path changed and there is no reasonable verification.

## Repo-Specific Gotchas
- `<example: This repo uses server actions in app routes; do not move logic to client components unless required>`
- `<example: Shared UI components live in packages/ui and should stay framework-agnostic>`
- `<example: DB schema changes require migration files plus generated types>`
- `<example: E2E tests are flaky on CI; prefer targeted local runs first>`

## Directory Notes
- `<path>`: <editing guidance>
- `<path>`: <testing guidance>
- `<path>`: <common pitfalls>
- `<path>`: <ownership or review expectation>

## PR Expectations
- Include:
  - what changed
  - why it changed
  - how it was verified
  - any follow-up work
- Call out:
  - breaking changes
  - schema changes
  - env var changes
  - manual migration or rollout steps

## When Unsure
- Ask for clarification only if the choice is high-risk or changes product behavior.
- Otherwise make the safest reasonable assumption, proceed, and state it clearly.
```

这份模板适合团队的原因是它把重点放在 4 件事上：`统一命令`、`统一交付标准`、`统一风险边界`、`统一 PR 习惯`。这比写很多“人格化提示词”更有用。

如果你愿意，我下一条可以继续给你一份：

- `适合前端团队的版本`
- `适合后端/API 团队的版本`
- `适合 monorepo 团队的版本`

来源：<https://github.com/shanraisshan/claude-code-best-practice>

---

### 5. 用户提问

`我要的是面向 Claude Code 重度团队的最佳实践指南`

### 5.1 助手说明

我把范围收窄到官方 Claude Code 文档和这个仓库本身，整理成一份偏团队落地的重度使用指南，不只给模板。

### 5.2 助手回答

截至 **2026-04-03**，如果你要做的是“面向 Claude Code 的重度团队落地”，我的建议很明确：**把 Claude Code 当成可治理的工程平台，而不是每个工程师各写各的 prompt**。社区仓库 `claude-code-best-practice` 很适合找灵感，但和当前官方文档有一个关键差异要先纠正：**官方现在把自定义 commands 基本并入了 skills**，`.claude/commands/` 仍兼容，但团队新建设计应优先围绕 `skills` 展开。  
来源：  
- <https://code.claude.com/docs/en/skills>  
- <https://github.com/shanraisshan/claude-code-best-practice>

## 核心架构

- 组织接入层：多数团队优先用 **Claude for Teams / Enterprise**。大团队真正需要的是 SSO、role-based permissions、managed policy，而不是每人一套本地野生配置。  
  来源：<https://code.claude.com/docs/en/team>
- 仓库契约层：固定共享这几样东西：`CLAUDE.md`、`.claude/settings.json`、`.claude/skills/`、`.claude/agents/`、`.claude/hooks/`、`.mcp.json`。
- 分工要清楚：`CLAUDE.md` 写项目真相，`settings.json` 写权限和工具策略，`skills` 写可复用流程，`subagents` 写隔离上下文的专职角色，`hooks` 写强制验证，`MCP` 只接外部系统。
- 作用域不要混：**managed** 放组织硬规则，**project** 放团队默认，**user/local** 放个人偏好。不要把公司政策写进个人目录，也不要把个人习惯塞进项目规则。  
  来源：  
  - <https://code.claude.com/docs/en/settings>  
  - <https://code.claude.com/docs/en/memory>

## 怎么设计团队资产

- `CLAUDE.md` 只放稳定、共享、可执行的信息：启动/测试/构建命令、目录职责、禁区、评审标准。不要把临时任务和个人碎碎念塞进去。  
  来源：<https://code.claude.com/docs/en/memory>
- 高复用流程优先做成 `skills`，例如 `/plan-feature`、`/review-pr`、`/migration-check`、`/release-note`。skill 的 `description` 是触发路由，不是产品文案。  
  来源：<https://code.claude.com/docs/en/skills>
- monorepo 不要只在根目录堆一层技能。官方支持从嵌套 `.claude/skills/` 自动发现，前端包、后端包、数据包都可以各自维护本地技能树。  
  来源：<https://code.claude.com/docs/en/skills>
- `subagents` 只做“专精且隔离”的事，不做“全能后端工程师”。好的子代理像 `payment-auditor`、`db-migration-reviewer`、`frontend-regression-checker` 这种能明确路由的角色。  
  来源：<https://code.claude.com/docs/en/sub-agents>
- 给 subagent 限权：限制 `tools`、`permissionMode`、`model`、`mcpServers`，必要时预载 `skills`。重度团队最值钱的不是“更多代理”，而是“更小权限、更清晰职责”。  
  来源：<https://code.claude.com/docs/en/sub-agents>
- 一个简单判断：**同上下文复用流程，用 skill；需要独立上下文、独立权限、独立 MCP 或独立模型时，用 subagent。**  
  来源：  
  - <https://code.claude.com/docs/en/skills>  
  - <https://code.claude.com/docs/en/sub-agents>

## 怎么运行日常工作流

- 我建议把团队默认节奏固定成：**Explore / Plan -> Implement -> Verify -> Review -> Ship**。大改动先出计划，再实现，再让 Claude 自测和自审 diff，最后人审。
- `PreToolUse` hook 用来拦危险 Bash、敏感目录写入、未批准 MCP。现在它不只是 allow/deny，还支持 `ask` 和 `defer`，足够接内部策略服务。  
  来源：<https://code.claude.com/docs/en/hooks>
- `PostToolUse` hook 用来做自动格式化、lint、定向测试，把机械检查从人手里拿掉。  
  来源：<https://code.claude.com/docs/en/hooks>
- `Stop` / `SubagentStop` hook 用来防止 Claude 过早宣布完成，要求它在停下前确认测试、文档、review 都过了。  
  来源：<https://code.claude.com/docs/en/hooks>
- CI 再接一层 **Claude Code GitHub Actions**。`@claude` 很适合做 PR 分析、issue 到 PR、批量 review，但不应该直接变成无人值守 merge bot。  
  来源：<https://code.claude.com/docs/en/github-actions>

## 安全、成本、可观测

- 安全基线先写死：`.claude/settings.json` 直接 deny `.env`、`secrets/**`、凭证文件；敏感仓库再加 sandbox 或 devcontainer，不要靠“大家小心一点”。  
  来源：  
  - <https://code.claude.com/docs/en/settings>  
  - <https://code.claude.com/docs/en/security>
- 第三方 MCP 默认不信任。官方明确提示第三方 MCP 有安全和 prompt injection 风险；团队应该把允许的 MCP 列表纳入版本控制，并优先自建或选可信提供方。  
  来源：  
  - <https://code.claude.com/docs/en/mcp>  
  - <https://code.claude.com/docs/en/security>
- 如果某个 MCP 只该给特定角色用，就挂在 subagent 的 `mcpServers` 上，不要把工具暴露给整个主会话。  
  来源：<https://code.claude.com/docs/en/sub-agents>
- 成本上，API 团队应启用 workspace spend limits 和 `/cost`；Teams / Enterprise 更适合看 adoption 和工程产出。官方 analytics 能看 DAU、accept rate、PR/LOC 贡献，但 **Zero Data Retention** 下没有 contribution metrics。  
  来源：  
  - <https://code.claude.com/docs/en/costs>  
  - <https://code.claude.com/docs/en/analytics>
- 截至当前官方文档，Sonnet 4.6 的经验值大约是 **$100-200 / 开发者 / 月**，但并行实例和自动化会把成本明显拉高。  
  来源：<https://code.claude.com/docs/en/costs>

## 最常见的 5 个反模式

- 一个超长 `CLAUDE.md` 试图装下所有规则。
- 一个万能 agent 管所有事。
- 每个人自由接 MCP，团队没有准入和审计。
- 长期开 `bypassPermissions`。
- 只看“Claude 会不会写代码”，不看 adoption、贡献、成本和 review 质量。

真要重度落地，我建议顺序是：先 `CLAUDE.md + settings.json`，再做 `3-5` 个高频 `skills`，再做 `2-3` 个高价值 `subagents`，最后才上 hooks、MCP 和 GitHub 自动化。

如果你要，我下一条可以直接给你一份 **“重度团队版 Claude Code 仓库骨架”**，包括目录、文件名和每个文件该写什么。

---

### 6. 用户提问

`我要`

### 6.1 助手说明

我补一下官方文档里的几个落地点，确保仓库骨架里的目录和职责跟当前 Claude Code 文档对齐。

### 6.2 助手回答

这套骨架假设两层治理同时存在：仓库里放 `Project scope` 共享资产，组织层用 `Managed scope` 管硬性权限和 MCP 白名单。重度团队默认**不用** `.claude/commands/` 起步，优先用 `skills`，因为官方当前推荐 `skills` 并支持 supporting files。  
来源：  
- <https://code.claude.com/docs/en/settings>  
- <https://code.claude.com/docs/en/skills>

## 骨架

```text
repo/
├── CLAUDE.md
├── .claude/
│   ├── settings.json
│   ├── settings.local.example.json
│   ├── skills/
│   │   ├── plan-feature/
│   │   │   ├── SKILL.md
│   │   │   ├── templates/plan-template.md
│   │   │   └── references/risk-checklist.md
│   │   ├── implement-change/
│   │   │   ├── SKILL.md
│   │   │   ├── references/coding-standards.md
│   │   │   └── scripts/collect-context.sh
│   │   ├── review-pr/
│   │   │   ├── SKILL.md
│   │   │   └── references/review-checklist.md
│   │   ├── migration-check/
│   │   │   ├── SKILL.md
│   │   │   └── references/schema-and-rollout.md
│   │   └── release-note/
│   │       └── SKILL.md
│   ├── agents/
│   │   ├── planner.md
│   │   ├── implementer.md
│   │   ├── reviewer.md
│   │   └── production-auditor.md
│   ├── hooks/
│   │   ├── pretool-policy.sh
│   │   ├── posttool-format-and-test.sh
│   │   ├── stop-require-evidence.sh
│   │   └── lib/policy.sh
│   └── rules/
│       ├── frontend.md
│       ├── backend.md
│       ├── data.md
│       └── security.md
├── .mcp.json
├── docs/ai/
│   ├── repo-map.md
│   ├── architecture.md
│   ├── service-boundaries.md
│   ├── testing-matrix.md
│   ├── release-playbook.md
│   └── incident-checklist.md
└── .github/workflows/
    ├── claude-pr-review.yml
    └── claude-issue-to-pr.yml
```

## 每个文件写什么

- `CLAUDE.md`：只写启动命令、测试命令、目录职责、危险区、交付标准。别塞个人偏好。控制在 200 行内，必要时 `@docs/ai/...` 导入少量高价值文档。  
  来源：<https://code.claude.com/docs/en/memory>
- `.claude/settings.json`：写团队共享权限、deny 规则、hooks 绑定、sandbox、`claudeMdExcludes`、共享插件和项目级 MCP。不要放密钥。  
  来源：<https://code.claude.com/docs/en/settings>
- `.claude/settings.local.example.json`：只做团队示例，告诉成员哪些配置允许本地覆盖；真正的 `.claude/settings.local.json` 不入库。
- `.claude/skills/plan-feature/SKILL.md`：输出阶段计划、风险、验证矩阵、回滚点。它是团队默认入口。
- `.claude/skills/implement-change/SKILL.md`：约束实现方式，强调复用现有模式、控制 diff、先跑最小验证。
- `.claude/skills/review-pr/SKILL.md`：只关注 bug、回归、安全、测试缺口，不写空泛总结。
- `.claude/skills/migration-check/SKILL.md`：专门审 schema、env、兼容性、灰度和回滚。
- `.claude/skills/release-note/SKILL.md`：把 merged diff 变成 release summary、风险提示和手动步骤。
- `templates/`、`references/`、`scripts/`：模板、长文档、辅助脚本放这里，别把 SKILL.md 写成大杂烩。  
  来源：<https://code.claude.com/docs/en/skills>
- `.claude/agents/planner.md`：`permissionMode: plan`，只读探索，预载 `plan-feature`。
- `.claude/agents/implementer.md`：负责改代码和跑定向验证，工具范围收窄，不给它默认全仓库高权限。
- `.claude/agents/reviewer.md`：只读 reviewer，检查 diff、测试和风险。
- `.claude/agents/production-auditor.md`：只给日志/监控类 MCP，不给编辑权。敏感 MCP 尽量内联到 agent，而不是放全局 `.mcp.json`。  
  来源：<https://code.claude.com/docs/en/sub-agents>
- `.claude/hooks/pretool-policy.sh`：拦危险 Bash、拦 secrets 路径、拦未批准网络/外部工具。`PreToolUse` 现在可以 `allow / deny / ask / defer`。  
  来源：<https://code.claude.com/docs/en/hooks>
- `.claude/hooks/posttool-format-and-test.sh`：写完文件后自动格式化、lint、跑最小相关测试。  
  来源：<https://code.claude.com/docs/en/hooks>
- `.claude/hooks/stop-require-evidence.sh`：Claude 想结束时，强制它给出验证证据、未验证项和风险说明。
- `.claude/rules/*.md`：按目录或技术栈拆细则，避免根 `CLAUDE.md` 失控。
- `.mcp.json`：只放团队批准的通用 MCP，比如 GitHub、Jira、内部文档、只读观测。第三方 MCP 不默认信任。  
  来源：  
  - <https://code.claude.com/docs/en/mcp>  
  - <https://code.claude.com/docs/en/security>
- `docs/ai/*.md`：给 AI 读的结构化文档，不是面向人的长篇 PRD。重点写 repo map、边界、测试矩阵、发布步骤。
- `.github/workflows/claude-pr-review.yml`：让 `@claude` 或自动触发做 PR 分析、issue triage、批量 review，但不要直接无人值守 merge。  
  来源：<https://code.claude.com/docs/en/github-actions>

## 重度团队的默认约束

- 组织级硬规则放 Managed settings，不要只靠仓库内提示词；Enterprise 支持 role-based permissions 和 managed policy。  
  来源：  
  - <https://code.claude.com/docs/en/team>  
  - <https://code.claude.com/docs/en/settings>
- 根目录只放跨团队共识；monorepo 的包级能力下沉到 `packages/*/.claude/skills/` 和 `packages/*/CLAUDE.md`，官方支持嵌套自动发现。  
  来源：  
  - <https://code.claude.com/docs/en/skills>  
  - <https://code.claude.com/docs/en/memory>
- 所有敏感路径先 deny，再按需 allow。官方也强调网络请求默认需要批准，MCP 需要信任校验。  
  来源：<https://code.claude.com/docs/en/security>

## 第一批先建这 8 个

1. `CLAUDE.md`
2. `.claude/settings.json`
3. `plan-feature`
4. `implement-change`
5. `review-pr`
6. `planner.md`
7. `reviewer.md`
8. `pretool-policy.sh`

如果你要，我下一条可以直接给你一份 **可落地的重度团队版 `CLAUDE.md` 正文**，不是模板提示，而是按这套骨架写好的初稿。

---

## 主要参考链接

- GitHub 仓库：<https://github.com/shanraisshan/claude-code-best-practice>
- Claude Code Skills：<https://code.claude.com/docs/en/skills>
- Claude Code Subagents：<https://code.claude.com/docs/en/sub-agents>
- Claude Code Hooks：<https://code.claude.com/docs/en/hooks>
- Claude Code Settings：<https://code.claude.com/docs/en/settings>
- Claude Code Memory：<https://code.claude.com/docs/en/memory>
- Claude Code MCP：<https://code.claude.com/docs/en/mcp>
- Claude Code Security：<https://code.claude.com/docs/en/security>
- Claude Code Team：<https://code.claude.com/docs/en/team>
- Claude Code Analytics：<https://code.claude.com/docs/en/analytics>
- Claude Code Costs：<https://code.claude.com/docs/en/costs>
- Claude Code GitHub Actions：<https://code.claude.com/docs/en/github-actions>
