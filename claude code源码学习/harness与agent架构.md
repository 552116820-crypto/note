# 这个项目是怎么实现harness的

## Claude Code Harness 实现架构

整个 harness 是一个基于 React/Ink 的终端应用，核心是一条 **异步生成器驱动的工具循环**。以下是关键层次：

### 1. 启动编排 — `src/main.tsx`

```
main() 
  → 并行预取 (MDM、Keychain、模块加载)
  → setup() 初始化插件/技能
  → getCommands() + getAgentDefinitions() 并行加载
  → 权限上下文初始化
  → launchRepl() 启动 React 渲染树
```

`launchRepl()`（`src/replLauncher.tsx`）动态导入 `App` 和 `REPL` 组件，通过 Ink 渲染到终端：

```
<App>  →  <AppStateProvider>  →  <REPL />
```

### 2. REPL 主循环 — `src/screens/REPL.tsx`

这是事件循环的核心，约 3800 行。每次用户提交消息后：

```typescript
for await (const event of query({
  messages, systemPrompt, canUseTool, toolUseContext, ...
})) {
  onQueryEvent(event)  // 更新 UI 状态
}
```

用 `useQueueProcessor()` 钩子消费排队的命令和消息，`onQueryEvent()` 处理流事件并更新 React 状态。

### 3. 查询循环（核心调度） — `src/query.ts`

`query()` 是一个 **async generator**，内部维护状态机：

```
normalizeMessagesForAPI()  →  API 调用 (withRetry)
    ↓
StreamingToolExecutor 接收 tool_use 块
    ↓
对每个工具调用:
  1. canUseTool() — 权限判定
  2. executePreToolUseHook() — 前置钩子
  3. runTools() — 实际执行
  4. executePostToolUseHook() — 后置钩子
    ↓
检查停止条件 (end_turn / maxTurns / stopHook)
  → 继续循环 或 yield Terminal 退出
```

### 4. 工具并发控制 — `src/services/tools/toolOrchestration.ts`

```typescript
export async function* runTools(toolUseMessages, canUseTool, toolUseContext)
```

关键设计：**只读工具并行，写入工具串行**。`partitionToolCalls()` 将工具调用分为并发安全和非安全两组，读操作用 `Promise.all()` 并发，写操作按序执行。`StreamingToolExecutor` 维护 `queued → executing → completed → yielded` 的状态队列。

### 5. 权限系统 — `src/hooks/useCanUseTool.tsx`

每次工具调用必经权限门控：

```
hasPermissionsToUseTool() → 'allow' | 'deny' | 'ask'
  ├─ allow → 直接放行，记录日志
  ├─ deny  → 记录拒绝，通知用户
  └─ ask   → 三种处理器优先匹配：
       1. handleCoordinatorPermission() — 协调器模式
       2. handleSwarmWorkerPermission() — Swarm 工作线程
       3. handleInteractivePermission() — REPL 交互对话框
```

权限上下文（`ToolPermissionContext`）包含 `mode`（default/bypass/auto）、`alwaysAllowRules`、`alwaysDenyRules` 等规则集。

### 6. Hooks 系统 — `src/utils/hooks.ts`

用户可通过 `settings.json` 配置在特定事件触发 shell 命令：

**事件类型**：`PreToolUse`、`PostToolUse`、`Stop`、`SessionStart`、`PermissionRequest` 等

**钩子类型**（`src/schemas/hooks.ts`）：
- `BashCommandHook` — 执行 shell 命令
- `PromptHook` — LLM 评估
- `HttpHook` — HTTP POST
- `AgentHook` — Agent 验证

钩子可以返回 `decision`（覆盖权限判定）、`stopReason`（中止执行）、或 `{ async: true }` 异步运行。超时默认 10 分钟。

### 7. 状态管理 — `src/state/AppState.tsx`

类 Zustand 的自建 store，通过 `useSyncExternalStore` 与 React 集成：

```typescript
AppState = {
  settings, messages, toolPermissionContext,
  mainLoopModel, verbose, ...20+ fields
}
```

`useAppState(selector)` 切片订阅避免无关渲染，`useSetAppState()` 提供无渲染的更新器。

### 整体流程图

```
CLI argv → main() → setup() → launchRepl()
                                    ↓
                              <REPL /> (React/Ink)
                                    ↓
                         用户输入 → query() async generator
                                    ↓
                              ┌─ API 调用 ─┐
                              │  模型采样   │
                              └─────┬──────┘
                                    ↓
                            tool_use 块列表
                                    ↓
                     ┌──────────────┼──────────────┐
                     │              │              │
                  权限检查      前置 Hook      工具执行
                canUseTool()  preToolUse    runTools()
                     │              │         ↓    ↓
                     │              │      并行  串行
                     │              │     (只读) (写入)
                     └──────────────┼──────────────┘
                                    ↓
                              后置 Hook
                            postToolUse
                                    ↓
                         停止条件检查 → 循环或退出
```

核心设计思想是 **async generator 驱动的事件流** + **React 状态响应式 UI** + **分层权限门控** + **可扩展 Hook 系统**，形成一个高度模块化的 CLI harness。

---

# 这个项目的agent架构是怎样的

## Claude Code 多 Agent 架构

整个系统支持四种 Agent 运行模式，统一通过 `AgentTool` 入口分发。

---

### 1. Agent 生命周期总览

```
用户/模型调用 AgentTool
        ↓
    路由判定
   ┌────┼────┬──────────┐
   ↓    ↓    ↓          ↓
 Fork  普通  Teammate  Remote
子agent 后台  (组队)    (云端)
   ↓    ↓    ↓          ↓
注册 Task → 执行 query() 循环 → 完成/失败 → 通知 → 清理
```

---

### 2. AgentTool — 统一入口（`src/tools/AgentTool/AgentTool.tsx`）

输入参数决定路由：

| 参数组合 | 路由 |
|----------|------|
| `subagent_type` 省略 + FORK_SUBAGENT 开启 | Fork 子 agent（继承父上下文） |
| `subagent_type` = 具名 agent | 匹配 agent 定义，后台执行 |
| `team_name` + `name` | Teammate 模式（组队协作） |
| `isolation: 'worktree'` | Git worktree 隔离 |
| `isolation: 'remote'` | 传送到远程 CCR 环境 |

---

### 3. 四种 Agent 模式

#### A. Fork 子 Agent（`src/tools/AgentTool/forkSubagent.ts`）

最轻量的分支方式：子 agent **继承父 agent 的完整 system prompt**（字节相同，利用 prompt cache），获得父 agent 的助手消息上下文，拥有与父 agent 完全相同的工具集。

```typescript
FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],       // 与父 agent 相同
  maxTurns: 200,
  model: 'inherit',   // 继承父模型
  permissionMode: 'bubble'  // 权限冒泡到父级
}
```

防止递归：`isInForkChild()` 检查阻止 fork 套 fork。

#### B. 后台 Agent（`src/tasks/LocalAgentTask/LocalAgentTask.tsx`）

标准异步 agent，注册为 `local_agent` 类型的 Task：

```typescript
LocalAgentTaskState = {
  type: 'local_agent',
  agentId: string,
  prompt: string,
  selectedAgent?: AgentDefinition,
  abortController?: AbortController,
  result?: AgentToolResult,
  progress?: ProgressTracker,    // 工具调用计数、token、最近活动
  isBackgrounded: boolean,
  pendingMessages: string[],     // SendMessage 排队的消息
}
```

进度追踪器记录最近 5 次工具调用、累计 token 消耗，实时更新到 AppState。

#### C. In-Process Teammate（组队协作）

这是最复杂的模式，多个 agent 在同一进程内并发运行。

**创建流程**（`src/utils/swarm/spawnInProcess.ts`）：
1. 生成 `agentId = formatAgentId(name, teamName)`
2. 创建独立的 `AbortController`（不随 leader 中止）
3. 用 `AsyncLocalStorage` 创建 `TeammateContext` 隔离上下文
4. 注册 `InProcessTeammateTaskState` 到 AppState

**执行循环**（`src/utils/swarm/inProcessRunner.ts`）：
```
runWithTeammateContext(context)
    ↓
  runAgent() — 与普通 agent 相同的 query() 循环
    ↓
  每轮工具执行后: 检查 mailbox（收件箱）
    ↓
  drainPendingMessages() — 注入排队消息
```

**Team 文件结构**（`src/utils/swarm/teamHelpers.ts`）：
```typescript
TeamFile = {
  name: string,
  leadAgentId: string,
  members: [{
    agentId, name, agentType, model,
    cwd, worktreePath?,
    backendType: 'in-process' | 'tmux',
    isActive: boolean
  }],
  teamAllowedPaths?: [...]
}
```

#### D. Remote Agent（`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`）

Agent 被传送到 Anthropic 云端（CCR）执行：
- `teleportToRemote()` 将 agent 发送到远程环境
- 主进程通过 `pollRemoteSessionEvents()` 轮询状态
- 结果写入本地 task 输出文件

---

### 4. Agent 间通信 — SendMessageTool（`src/tools/SendMessageTool/SendMessageTool.ts`）

支持三种消息类型：

| 类型 | 用途 |
|------|------|
| 纯文本 `{ to, message, summary }` | 普通通信 |
| `shutdown_request` | 请求 teammate 退出 |
| `plan_approval_response` | Team lead 审批 teammate 的计划 |

**路由逻辑**：

```
SendMessage(to, message)
    ↓
目标是 in-process agent？
  ├─ 运行中 → queuePendingMessage()，下一轮工具执行后注入
  └─ 已停止 → resumeAgentBackground() 唤醒后投递
    ↓
广播到 team？
  → 遍历所有 teammates（排除发送者）
    ↓
跨会话？
  → postInterClaudeMessage() 或 UDS socket
```

**关闭协议**：
```
Leader 发送 shutdown_request
    → Teammate 收到，决定 approve/reject
    → approve: abortController.abort() → agent 循环退出
    → reject: 返回拒绝原因，继续工作
```

---

### 5. 三层隔离机制

| 层级 | 机制 | 隔离范围 |
|------|------|----------|
| **进程内** | `AsyncLocalStorage` | 身份、邮箱、权限上下文（同进程多 teammate） |
| **文件系统** | `git worktree` | 每个 agent 独立分支和工作目录 |
| **环境** | CCR 远程沙箱 | 完全隔离的云端环境 |

Worktree 创建流程（`src/tools/EnterWorktreeTool/`）：
```
findCanonicalGitRoot()
  → createWorktreeForSession(sessionId, slug)
  → process.chdir(worktreePath)
  → 清除缓存（system prompt、memory、plans）
```

---

### 6. Coordinator 模式（`src/coordinator/coordinatorMode.ts`）

当 `COORDINATOR_MODE` 开启时，主 agent 变为 **编排器**，不直接执行工具，而是通过 AgentTool 派发 worker：

```
Coordinator（编排器）
    ├─ AgentTool → Worker A (worktree 隔离)
    ├─ AgentTool → Worker B (worktree 隔离)  ← 可并行
    └─ AgentTool → Worker C
                      ↓
            Worker 独立运行 query() 循环
            结果通过 task output 返回 Coordinator
```

Coordinator 的 system prompt 会注入 worker 可用工具列表，并指示：
- Worker 不得嵌套 AgentTool
- 尽量并行派发独立任务
- 使用 scratchpad 目录共享跨 worker 知识

---

### 7. Agent 定义系统（`src/tools/AgentTool/loadAgentsDir.ts`）

Agent 定义来源：内置 → 用户设置 → 项目 `.claude/agents/` → 插件

```typescript
BaseAgentDefinition = {
  agentType: string,
  whenToUse: string,        // 描述何时使用
  tools?: string[],         // 可用工具白名单
  disallowedTools?: string[],
  mcpServers?: [...],       // 需要的 MCP 服务器
  hooks?: HooksSettings,
  model?: string,
  maxTurns?: number,
  isolation?: 'worktree' | 'remote',
  background?: boolean,     // 始终异步
  permissionMode?: PermissionMode,
}
```

内置 agent 类型包括：`general-purpose`、`explore`、`plan`、`verification` 等。

---

### 8. Task 生命周期框架（`src/utils/task/framework.ts`）

```
registerTask()                    ← 注册到 AppState.tasks
  → emit task_started
    ↓
updateTaskState()                 ← 运行中更新进度
    ↓
completeAgentTask() / failAgentTask()   ← 终态
  → enqueuePendingNotification()  ← 通知主循环
  → emit task_terminated_sdk
    ↓
evictTaskOutput() + evictTerminalTask()  ← 延迟清理
```

Task 状态流：`pending → running → completed | failed | killed`

所有 task 类型共享这套框架：`local_bash`、`local_agent`、`remote_agent`、`in_process_teammate`、`dream`、`local_workflow`、`monitor_mcp`。

---

### 架构总结

整个多 Agent 系统的核心设计思想：

1. **统一入口**：所有 agent 操作通过 `AgentTool` 分发
2. **Task 抽象**：不同运行模式统一为 Task 生命周期管理
3. **三层隔离**：AsyncLocalStorage（进程内）→ git worktree（文件系统）→ CCR（远程）
4. **异步通信**：SendMessageTool + 邮箱机制，支持排队、唤醒、广播
5. **协调模式**：Coordinator 不执行工具，只编排 Worker，Worker 并行独立运行
6. **渐进式复杂度**：从最轻的 Fork（继承上下文）到最重的 Remote（云端隔离），按需选择

---

# harness和agent架构有什么不同

两者是 **层次关系**，不是并列关系。

## Harness = 单个 Agent 的运行引擎

Harness 是每一个 agent 实例内部的执行框架：

```
query() async generator
  → API 调用
  → 接收 tool_use 块
  → 权限检查 (canUseTool)
  → 前置 Hook
  → 工具执行 (runTools)
  → 后置 Hook
  → 循环或退出
```

它负责的是 **一个** agent 的：启动、消息规范化、API 交互、工具分发、权限门控、Hook 触发、token 预算、自动压缩、停止判定。

## Agent 架构 = 多个 Harness 实例的编排层

Agent 架构建立在 harness **之上**，管理的是多个 harness 实例之间的关系：

```
Coordinator (自己跑一个 harness)
    ├─ AgentTool → Worker A (独立的 harness 实例)
    ├─ AgentTool → Worker B (独立的 harness 实例)
    └─ SendMessageTool → 跨 harness 通信
```

它负责的是：agent 生成路由、Task 生命周期、隔离策略、进程间通信、关闭协调。

## 一句话区分

| | Harness | Agent 架构 |
|---|---|---|
| **是什么** | 单 agent 的执行循环 | 多 agent 的协调层 |
| **核心文件** | `query.ts`、`REPL.tsx`、`useCanUseTool` | `AgentTool.tsx`、`SendMessageTool`、`coordinator/` |
| **关注点** | API 调用、工具执行、权限、Hook | 生成、通信、隔离、生命周期 |
| **实例数** | 每个 agent 各一份 | 整个会话共享一套 |

**每个被派出的 agent（无论 Fork、后台、Teammate、Remote）内部都在跑同一套 harness 的 `query()` 循环。** Agent 架构只是决定了「启动几个 harness 实例、给它们什么上下文、怎么互相通信、怎么收回结果」。
