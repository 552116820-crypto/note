# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `47083a24-f0d0-4432-948e-f18c5650427d`
- **范围**: Turn 1-6
- **导出时间**: 2026-05-25 15:49:41

---

### Turn 1  *(2026-05-25 06:35)*

**User:**

CCAF考核的领域：Agentic 架构与编排，能否介绍一下，并给我一份学习计划

**Assistant:**

先查一下 CCAF 是哪个认证体系，再给你介绍领域内容和学习计划。

## CCAF 是 Anthropic 官方的 Claude Certified Architect – Foundations

考试格式：60 题选择题，120 分钟，远程监考，720/1000 通过。共 5 个领域，**Agentic Architecture & Orchestration 占 27%，是权重最大的一块**。

---

## 一、Domain 1 考的是什么

围绕 Claude Agent SDK 设计「能跑在生产环境的自主代理系统」。官方拆成 7 个子领域：

| #   | 子领域                       | 核心考点                                                                                                            |
| --- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 1.1 | **Agentic Loop**             | `stop_reason` 控制流：`tool_use` 继续 / `end_turn` 终止；tool_result 追加到对话历史                                 |
| 1.2 | **多代理编排**               | Hub-and-spoke（中央协调者 + 专门子代理）vs 扁平架构；三大收益：可观测、信息流可控、故障隔离                         |
| 1.3 | **子代理调用 / 上下文传递**  | `Task` 工具 spawn 子代理；**只传该子任务相关上下文**，不传 coordinator 全历史；多 `Task` 并行；`allowedTools` 配置 |
| 1.4 | **多步骤工作流强制**         | 程序化前置条件 vs 提示词指令；100% 强制 vs ~99% 概率执行                                                            |
| 1.5 | **Agent SDK Hooks**          | `PostToolUse`（数据规范化，如时间戳统一）/ 拦截 hook（策略拦截，如退款 > $500 自动升级）；hook 是确定性代码         |
| 1.6 | **任务拆解策略**             | 静态 prompt chaining vs 动态 adaptive 拆解；什么场景用哪个                                                          |
| 1.7 | **Session 管理**             | `--resume`、`fork_session`、命名 session；长会话状态保存                                                            |

**反模式（高频考点）**：用自然语言信号判循环终止、设硬性迭代上限当主要停机制、用文本内容判完成、子代理之间互通、把整个对话历史塞给子代理。

---

## 二、4 周学习计划

按你工作日 19:30 后空闲 + 周末来排，每周约 12–14h。

### Week 1 — Agentic Loop 基础 + Hooks
- **读**：Anthropic 官方 Agent SDK 文档「agentic loop」「stop_reason」「hooks」章节
- **做**：用 Agent SDK 写一个最小 loop（单工具：fetch_weather），手动观察 `stop_reason` 状态机
- **加码**：实现一个 `PostToolUse` hook 把 API 返回的 unix 时间戳转 ISO8601
- **自检**：能用一句话说清「hook vs prompt 指令」分别用于什么场景

### Week 2 — 多代理 + Subagent
- **读**：Claude Code 内置的 Task 工具文档；Anthropic blog 上的 multi-agent research 案例
- **做**：复刻官方场景 3「多代理研究系统」——一个 coordinator 派 3 个 subagents（web search / doc analysis / synthesis），最终输出**带引用**的报告
- **关键练习**：刻意试两种 context 传递方式（全历史 vs 仅任务上下文），观察 token 用量和回答质量差异

### Week 3 — Workflow 强制 + Session + 任务拆解
- **读**：Agent SDK 的 session 文档（`--resume`、`fork_session`）；任意一篇 task decomposition 论文/blog
- **做**：复刻官方场景 1「客服解决方案 Agent」——用 hook 强制 refund > $500 必须 escalate，对比纯 prompt 指令的「漏网率」（跑 50 次看几次绕过规则）
- **附加**：写一个能 `--resume` 的长会话 agent，测试中断恢复

### Week 4 — 综合实战 + 模考
- **做**：选官方 6 个场景里你最不熟的 1 个，完整复刻
- **刷题**：去 claudecertprep.com、claudecertifications.com、flashgenius.net 这类站做免费练习题（注意：第三方站质量不一，用来查漏不当真理）
- **复盘**：把 Domain 1 的 7 个子领域，每个能用 3 句话说清「是什么 / 反模式 / 关键 API」

---

## 三、推荐的权威资料

| 资料                           | 用途                            |
| ------------------------------ | ------------------------------- |
| Anthropic 官方课程（200-level）| 唯一官方背书的备考材料          |
| Claude Agent SDK GitHub + 文档 | 所有 API 细节的 source of truth |
| claudecertprep.com Domain 1    | 第三方但内容最贴合考纲          |
| dev.to「5 Domains, 6 Scenarios」| 通过者写的实战 takeaway         |

---

要不要我把其中某一周展开成具体的代码任务清单（比如 Week 2 的多代理系统），或者先聊聊你目前对 Agent SDK 哪部分最陌生、想从哪切入？

No response requested.

---

### Turn 2  *(2026-05-25 06:58)*

**User:**

hub-and-spoke 编排是什么意思

**Assistant:**

## Hub-and-spoke = 中心辐射式编排

字面意思来自自行车轮：**hub**（车轴）在中心，**spokes**（辐条）从中心向外发散。每根辐条只连到 hub，辐条之间不直接相连。

放到多代理系统里：

```
            ┌──────────────┐
            │ Coordinator  │   ← hub（中央协调者）
            │   (hub)      │
            └──┬────┬───┬──┘
               │    │   │
        ┌──────┘    │   └──────┐
        ▼           ▼          ▼
   [Web 搜索]  [文档分析]  [报告合成]   ← spokes（专门子代理）
   subagent    subagent    subagent
```

**规则**：
- 所有子代理只跟 coordinator 通信，**子代理之间不互相调用、不共享上下文**
- coordinator 决定每个子代理看到什么信息、何时启动、结果怎么用

---

## 为什么这种结构是默认推荐

| 维度           | Hub-and-spoke                      | 扁平架构（subagent 互通）                  |
| -------------- | ---------------------------------- | ------------------------------------------ |
| **可观测性**   | 所有动作过 coordinator，单点可日志/审计 | 子代理互相调，调用链像意大利面，难追踪      |
| **信息流控制** | coordinator 决定谁知道什么            | context 容易污染、扩散，权限边界模糊        |
| **故障隔离**   | 一个 subagent 挂了，coordinator 接住决定怎么办 | 故障在网状结构里级联传播                    |
| **Token 成本** | 每个 subagent 只拿任务相关 context     | 全历史共享，token 爆炸                      |

---

## 在 Claude Agent SDK 里怎么落地

用 `Task` 工具 spawn 子代理：

```python
coordinator = Agent(
    model="claude-sonnet-4-...",
    tools=[
        Task,              # 必须 — 用来 spawn subagent
        summarize_results, # coordinator 自己做合成
        format_report,
    ]
)
```

**关键纪律**：调 `Task` 时，prompt 里**只放该子任务需要的上下文**，不要把 coordinator 的对话历史整段塞进去。塞了的话：① 烧 token ② 子代理被无关信息干扰，回答质量下降。

并行：在一个回合里发**多个** `Task` 调用，子代理会并行跑。

---

## 一个直观例子

「研究 X 公司财务状况并写报告」：

- **Hub**（coordinator）：拆任务，分发，最后合成
- **Spoke 1**：只拿到「搜 X 公司 2024 年财报相关新闻」，不知道最终要写报告
- **Spoke 2**：只拿到「分析这份 10-K 文件的现金流部分」，不知道有其他子代理在搜新闻
- **Spoke 3**（合成）：拿到 spoke 1 和 2 的**结构化结果**（带引用源），写最终报告

每个子代理上下文极简，coordinator 是唯一知道全局的人。

---

需要我再展开「context passing 应该长什么样」（结构化 JSON vs 纯文本），或者「什么时候反而不该用 hub-and-spoke」吗？

---

### Turn 3  *(2026-05-25 07:16)*

**User:**

自定义 slash 命令是什么

**Assistant:**

## Slash 命令 = 在 Claude Code 输入框里以 `/` 开头的快捷指令

分两类：
- **内置**：`/help`、`/clear`、`/config`、`/init` 等，由 Claude Code 本身提供
- **自定义**：你自己写的 markdown 文件，命名后就能用 `/<文件名>` 调用，本质是「**可复用的 prompt 模板**」

---

## 怎么写一个自定义 slash 命令

把一个 markdown 文件放到下面任一位置：

| 位置                          | 作用域                | 何时用                           |
| ----------------------------- | --------------------- | -------------------------------- |
| `~/.claude/commands/<name>.md`| 用户级（跨所有项目）  | 个人常用流程，如 `/daily-review` |
| `.claude/commands/<name>.md`  | 项目级（团队共享）    | 项目特定流程，如 `/run-migrations` |

文件名就是命令名：`fix-flaky-test.md` → `/fix-flaky-test`。

**最简形式**就是纯 markdown：

```markdown
请审查当前 git diff，重点找出：
1. 边界条件遗漏
2. 错误处理缺失
3. 命名不一致

输出格式：按文件分组，每条问题标 [严重/建议]。
```

保存为 `~/.claude/commands/review-diff.md`，之后在 Claude Code 里敲 `/review-diff`，等价于把上面这段 prompt 发给我。

---

## 进阶：带参数 + frontmatter

```markdown
---
description: 把指定文件按 PEP 8 重写
argument-hint: <file-path>
---

请读取 $ARGUMENTS，按 PEP 8 规范重写：
- 行宽 88
- 双引号统一
- 类型注解补齐

修改后跑 ruff check 验证。
```

调用：`/format-python src/utils.py`

`$ARGUMENTS` 会被替换成 `src/utils.py`。`description` 和 `argument-hint` 会出现在自动补全里。

---

## 实用例子（按场景）

| 命令                  | 内容                                                |
| --------------------- | --------------------------------------------------- |
| `/pr-summary`         | 读 `git log main..HEAD` 和 diff，生成 PR 描述       |
| `/explain-test $ARGUMENTS` | 解释指定测试文件在测什么、为什么这么测              |
| `/security-audit`     | 扫当前 diff，按 OWASP Top 10 检查漏洞               |
| `/standup`            | 汇总最近 24h 的 git 活动，生成站会发言稿            |

---

## 在 CCAF 里的位置

属于 **Domain 3: Claude Code Configuration & Workflows（20%）**，常见考点：

- 自定义 slash 命令 vs CLAUDE.md 的分工：
  - **CLAUDE.md**：始终注入的「项目规则/上下文」（被动）
  - **Slash 命令**：用户主动触发的「一次性 prompt」（主动）
- 项目级 vs 用户级的取舍：团队共享放项目级（进 git），个人偏好放用户级
- 跟 CI/CD 集成：slash 命令可以在 headless 模式（`claude -p`）下用，做自动化 code review、自动文档生成等

---

要不要我帮你写一个针对你具体工作流（比如调研 / 长期产品代码 / 多个短期项目）的自定义命令模板？

---

### Turn 4  *(2026-05-25 07:36)*

**User:**

handoff 模式是什么

**Assistant:**

## Handoff 模式 = 一个 agent 把控制权（连同上下文）整体移交给另一个 agent

跟 hub-and-spoke 的区别一句话讲清：

| 模式             | 控制权          | 类比                              |
| ---------------- | --------------- | --------------------------------- |
| **Hub-and-spoke**| coordinator 始终持有，subagent 做完返回结果给 hub | 老板派活给员工，员工交报告回老板  |
| **Handoff**      | 移交给下一个 agent，前一个退出 | 一线客服转给二线工程师，自己挂线  |

---

## 在 CCAF Domain 1.4「multi-step workflows with enforcement and handoff」里考什么

### 1. 何时该 handoff，何时该 hub-and-spoke

| 选 handoff         | 选 hub-and-spoke                  |
| ------------------ | --------------------------------- |
| 任务是**线性阶段**，下一阶段需要前一阶段的**完整对话状态** | 任务可**并行**或可**独立完成** |
| 用户跟接手的 agent 继续多轮对话    | 子任务只需要单次结果              |
| 不同阶段需要不同**工具集 / 权限**  | 工具差异不大                      |
| 例：客服 triage → 退款专员 → 升级人工 | 例：研究 agent 派多个子代理并行搜 |

### 2. Handoff 的关键设计点

**a. 上下文转交格式**

不要把整段对话原文丢过去，要做**结构化压缩**：

```json
{
  "case_id": "C-12345",
  "user_intent": "申请退款 $450",
  "verified_facts": {
    "order_id": "O-789",
    "purchase_date": "2026-04-01",
    "amount": 450
  },
  "actions_taken": ["lookup_order", "verify_identity"],
  "reason_for_handoff": "金额超出 triage agent 权限",
  "next_agent_role": "refund_specialist"
}
```

为什么结构化：接手的 agent 要能在不重读全部历史的情况下立刻知道「我在哪、要做什么」。

**b. 触发条件要程序化，不靠模型自觉**

考点反复出现的对比：

```python
# ❌ 反模式：纯 prompt 强制
system_prompt = "如果退款超过 $500，请转给升级专员"  # ~99% 概率执行

# ✅ 正模式：hook 拦截
def pre_tool_use(tool_call):
    if tool_call.name == "process_refund" and tool_call.args["amount"] > 500:
        return handoff_to("escalation_agent", context=...)  # 100% 执行
```

**c. 不可逆性**

Handoff 通常是单向的：交出去之后 triage agent 就不再参与了。这意味着**转交前一定要把所有必要信息提取干净**，否则下游补不回来。

---

## 一个完整例子：客服三段式 handoff

```
用户："我要退款 $800，订单号 O-789"
          │
          ▼
┌──────────────────┐
│  Triage Agent     │  工具：lookup_order, classify_intent
│                   │  发现：退款 + 金额 > $500
└────────┬──────────┘
         │ handoff（结构化 JSON）
         ▼
┌──────────────────┐
│ Refund Specialist │  工具：process_refund (≤$500), check_policy
│                   │  发现：金额超权限
└────────┬──────────┘
         │ handoff
         ▼
┌──────────────────┐
│  Escalation Agent │  工具：create_ticket, notify_human
│                   │  动作：开工单转人工
└──────────────────┘
```

每一段 agent 的**工具集都不一样**——这正是用 handoff 而不是单 agent 的原因。

---

## 易错点（考试常陷阱）

1. **把 handoff 当 hub-and-spoke 写**：让 triage 「调用」refund specialist 等返回——这是 subagent 模式，不是 handoff
2. **handoff 时丢上下文**：只传一句 "用户要退款"，新 agent 还得再问一遍订单号
3. **handoff 触发器写在 prompt 里**：考点要求关键决策用 hook 程序化强制
4. **过度 handoff**：能一个 agent 搞定的简单任务硬拆成三段，徒增复杂度和 token

---

要不要我接着讲跟 handoff 经常一起考的「escalation pattern」（升级到人工），那个是 Domain 5 的高频题？

---

### Turn 5  *(2026-05-25 07:39)*

**User:**

要

**Assistant:**

## Escalation Pattern = Agent 主动「举手」把任务交还给人类

Handoff 是 agent → agent，escalation 是 **agent → human**。在 Domain 5 (Context Management & Reliability, 15%) 里反复考。

---

## 为什么必考：生产 agent 的「安全阀」

LLM 本质是概率系统。生产环境里有些动作**绝不能赌概率**：
- 不可逆操作（退款、删数据、发邮件）
- 高金额 / 高风险（合规、法律、医疗）
- 模型自己不确定（confidence 低）
- 用户明显不满 / 情绪激动

Escalation 就是「**当 agent 没把握时，宁可打扰人，也不要瞎执行**」的机制。

---

## 四种触发器（考点都在这里）

### 1. 阈值触发（最常考）

业务规则的硬边界，**必须用 hook 程序化强制**，不能靠 prompt：

```python
def pre_tool_use(tool_call):
    if tool_call.name == "process_refund":
        if tool_call.args["amount"] > 500:
            return escalate(reason="amount_exceeds_threshold")
    if tool_call.name == "delete_account":
        return escalate(reason="irreversible_action")
```

考试陷阱：题目给一段 prompt「请确保退款超过 $500 时升级」，问问题在哪——答案是「应该用 hook，不是 prompt」。

### 2. Confidence 校准触发

Agent 自评把握度，低于阈值就升级：

```json
{
  "answer": "建议执行 X",
  "confidence": 0.45,
  "reasoning": "用户描述模糊，可能是 A 也可能是 B"
}
```

如果 `confidence < 0.7` → escalate。要让模型**结构化输出 confidence**（Domain 4 的 structured output 知识点交叉考）。

### 3. 重试耗尽触发

Agent 试了 N 次都没成功，不要无限循环，升级：

```
工具调用失败 → 重试 1 (变参数)
            → 重试 2 (换工具)
            → 重试 3 (换策略)
            → 还失败 → escalate
```

反模式：硬编死循环、无限重试、把错误吞掉假装成功。

### 4. 意图模糊触发

用户请求多义，agent 不该擅自猜：

```
用户："帮我处理一下那个订单"
→ 系统里有 3 个未完成订单
→ 不要瞎选 → escalate（或先问澄清问题）
```

---

## Escalation 的「交接包」长什么样

跟 handoff 类似，要**结构化**，让人类一眼看懂：

```json
{
  "case_id": "C-12345",
  "escalation_reason": "amount_exceeds_threshold",
  "user_request": "申请退款 $800",
  "agent_progress": {
    "verified": ["identity", "order_exists"],
    "attempted": ["check_refund_policy"],
    "blocked_at": "process_refund (权限不足)"
  },
  "context_summary": "用户购买了 O-789，因质量问题申请全额退款。订单确认存在，身份已验证。",
  "suggested_action": "审批退款 OR 联系用户协商",
  "priority": "medium",
  "user_sentiment": "frustrated"
}
```

**关键设计**：人类接手时，**不应该需要重新问用户**任何已经确认过的事实。

---

## 升级路径的层次设计（考点）

不是「立刻找人」，而是**分层升级**：

```
Level 0: Agent 自己解决
   │ 失败 / 不确定
   ▼
Level 1: 切换到更强模型 / 更专门 agent（still autonomous）
   │ 仍解决不了
   ▼
Level 2: 升级到一线人工
   │ 超出权限
   ▼
Level 3: 升级到主管 / 工程师 / 合规
```

每一层都要有**明确的触发条件**和**结构化交接**。

---

## Escalation vs Handoff vs Hub-and-Spoke 一张表

| 维度         | Hub-and-Spoke           | Handoff                  | Escalation             |
| ------------ | ----------------------- | ------------------------ | ---------------------- |
| 目标对象     | 子代理（短期工人）      | 另一个 agent（接班人）    | **人类**               |
| 控制权       | 始终在 coordinator      | 移交                     | 移交（且通常不可挽回）  |
| 触发原因     | 任务可拆 / 可并行       | 进入新阶段，需要不同工具集 | 风险 / 不确定 / 卡死    |
| 上下文       | 仅子任务相关            | 结构化转交                | 结构化转交 + 人类可读摘要 |
| 强制机制     | 提示词足够              | 提示词 + hook            | **必须 hook**           |

---

## 常见错误（考试反模式）

1. **没有 escalation 路径**：agent 卡死后无限重试或编造答案
2. **escalation 触发器写在 prompt 里**：高风险动作必须用 hook
3. **交接包是纯文本**：人类要花 5 分钟读历史才能接手，应该结构化
4. **过度升级**：每次小不确定都喊人，agent 失去价值
5. **升级即放弃上下文**：用户被人工接手时还得再说一遍诉求

---

## 一个完整场景（综合 Domain 1 + 5）

「客服退款 agent」生产版：

```python
# Hook 层：硬性强制
def pre_tool_use(call):
    if call.name == "process_refund":
        if call.args["amount"] > 500:           # 阈值触发
            return escalate("over_limit", build_handoff_package())
        if confidence_score < 0.7:               # 置信度触发
            return escalate("low_confidence", ...)

# Agent 层：自评 + 主动 escalate
if retry_count >= 3:                             # 重试耗尽触发
    return escalate("exhausted_retries", ...)
if ambiguous_intent(user_msg):                   # 意图模糊触发
    return ask_clarifying_question() or escalate(...)
```

四种触发器一个不缺，hook 兜底，agent 自检，结构化交接——这就是考试想看到的「production-ready」答案。

---

要不要继续讲跟 Domain 5 也常考的「错误传播策略」（subagent 报错时 coordinator 怎么处理）？

---

### Turn 6  *(2026-05-25 07:40)*

**User:**

要

**Assistant:**

## Error Propagation = Subagent 出错时，coordinator 怎么知道、怎么决定下一步

在 hub-and-spoke 里，subagent 是黑盒：它要么成功返回结果，要么失败。失败信息怎么从 spoke 传回 hub、hub 怎么处理——这是 Domain 5 的高频考点，也是「demo agent」和「production agent」最大的分水岭。

---

## 三种传播策略（必考）

### 1. Fail-fast（快速失败）

第一个 subagent 报错 → coordinator 立刻终止整个任务。

**适用**：步骤强依赖，前置失败后续无意义。

**例子**：
```
研究 agent：搜资料 → 分析 → 写报告
        搜资料 subagent 全挂 → 没必要继续，fail-fast
```

### 2. Graceful Degradation（降级）

部分 subagent 失败，coordinator 用剩下的结果**继续但标注降级**。

**适用**：多个 subagent 并行做**互补但非互斥**的工作。

**例子**：
```
多源研究：4 个 subagent 各搜 1 个数据源
  → 2 个成功，2 个失败
  → coordinator 用 2 个成功的合成报告
  → 在报告里标注「数据源 X, Y 不可用，结论基于 A, B」
```

### 3. Retry-then-escalate（重试后升级）

失败 → 重试 N 次（最好换策略不只是换参数）→ 仍失败 → escalate。

**适用**：瞬时故障可恢复，永久故障应升级。

```python
for attempt in range(3):
    result = run_subagent(task)
    if result.success:
        break
    task = adjust_strategy(task, result.error)  # 换策略，不是死重试
else:
    escalate(reason="subagent_exhausted", history=...)
```

---

## Subagent 报错该返回什么（**结构化错误**，跨 Domain 2 考点）

不要返回纯文本 "something went wrong"。要：

```json
{
  "success": false,
  "errorCategory": "transient | permanent | policy_violation | ambiguous_input",
  "isRetryable": true,
  "message": "API rate limit exceeded, retry after 60s",
  "details": {
    "attempted_action": "fetch_url",
    "url": "https://...",
    "http_status": 429,
    "retry_after_seconds": 60
  },
  "partial_results": [...]   // 失败前已经拿到的部分数据，不要扔
}
```

**为什么字段这么设计**：
- `errorCategory` → coordinator 决定走哪种策略
- `isRetryable` → 直接告诉 hub 能不能重试
- `partial_results` → 即使整体失败，已经拿到的也别浪费

---

## Coordinator 的决策表（考点）

| Subagent 返回             | Coordinator 动作                 |
| ------------------------- | -------------------------------- |
| `transient + isRetryable` | 重试，换参数/换策略              |
| `permanent`               | 不重试，尝试 graceful degradation 或 escalate |
| `policy_violation`        | 立即 escalate，不要重试          |
| `ambiguous_input`         | 回到用户问澄清问题               |
| `partial_results` 存在    | 即便整体失败，也合并进最终输出   |

---

## 反模式（考试反复出现）

| 反模式                              | 为什么错                            |
| ----------------------------------- | ----------------------------------- |
| Subagent `try/except: pass`，假装成功 | Coordinator 拿到「成功但内容空」的结果，错误被吞 |
| Coordinator 看到错误就无限重试      | 永久故障会烧死 token 也不解决       |
| 把 stack trace 原样塞进 LLM 上下文  | 噪音多、token 贵，应抽成结构化字段  |
| 一个 subagent 挂了就整个 task 崩    | 没做 graceful degradation，可靠性差 |
| 错误信息只给 message 不给 category  | Coordinator 没法做策略决策，只能瞎试 |
| Subagent 之间互相传错误             | 违反 hub-and-spoke，错误应汇总到 hub |

---

## 综合场景（把 Domain 1 + 5 串起来）

「多代理研究系统」生产版：

```
Coordinator 派 3 个 subagent 并行：web 搜索 / arXiv 搜索 / 内部知识库

返回结果：
  ├─ web subagent:    success, 10 条结果
  ├─ arXiv subagent:  failure, errorCategory=transient, isRetryable=true
  └─ kb subagent:     failure, errorCategory=permanent, partial_results=3 条

Coordinator 决策：
  1. arXiv → 重试 1 次（换查询词）→ 成功，拿到 5 条
  2. kb → 不重试，用 partial 3 条
  3. 合成报告，注明「内部知识库部分可用，仅检索到 3 条」
  4. 如果 web + arXiv 都失败 → escalate 给用户「无法完成研究，是否换方向」
```

四种策略组合用：retry（arXiv）+ degradation（kb）+ partial 利用 + escalation 兜底。

---

## 一张图收尾：三种「移交」串起来

```
   Subagent 错误
       │
       ▼
  ┌──────────────────┐
  │ 结构化错误返回    │  ← Domain 2: tool 错误设计
  └────────┬─────────┘
           │
           ▼
  ┌────────────────────┐
  │ Coordinator 决策   │  ← Domain 5: 错误传播策略
  │ (retry/degrade/?)  │
  └────────┬───────────┘
           │ 解决不了
           ▼
  ┌────────────────────┐
  │ Handoff or Escalate│  ← Domain 1.4 / Domain 5
  └────────────────────┘
```

---

到这里，Domain 1 + 5 的「编排骨架」基本闭环：agentic loop → multi-agent (hub-and-spoke) → handoff → error propagation → escalation。

接下来还想深入哪块？
- **Domain 2**: Tool design（工具描述怎么写不会被 Claude 误用、structured error 怎么设计）
- **Domain 3**: Claude Code 配置（CLAUDE.md 分层、slash 命令、plan mode 时机）
- **Domain 4**: Prompt engineering + structured output（JSON schema、validation/retry loop）
- 或者把官方 6 个 scenario 挑一个做完整的设计推演

---
