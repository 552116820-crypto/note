# 第 1 轮

## 用户

具体聊聊，Agent 框架 = 这个仓库自己写的那套 coordinator/tasks/tools 架构

## 助手

可以把它理解成一套自写的 LLM agent runtime，不是“Bun 帮它做 agent”，而是它自己把这些能力拼起来了：

- 会话状态机
- 工具注册与权限系统
- 子代理递归执行
- 异步任务/通知回流
- coordinator / worker 角色分工

**核心是什么**
最核心的是 [src/QueryEngine.ts:176](/mnt/e/claude-code-snapshot-backup/src/QueryEngine.ts#L176) 和 [src/query.ts:219](/mnt/e/claude-code-snapshot-backup/src/query.ts#L219)。

`QueryEngine` 负责“这次对话”的状态，持有消息、文件缓存、累计 usage、权限拒绝记录等；`submitMessage()` 先处理用户输入，再进入 `query()` 主循环。[src/QueryEngine.ts:209](/mnt/e/claude-code-snapshot-backup/src/QueryEngine.ts#L209) [src/QueryEngine.ts:416](/mnt/e/claude-code-snapshot-backup/src/QueryEngine.ts#L416) [src/QueryEngine.ts:675](/mnt/e/claude-code-snapshot-backup/src/QueryEngine.ts#L675)

`query()` 则是典型 agent loop：
1. 把当前 messages + system prompt + tools 发给模型
2. 流式接收 assistant 输出
3. 一旦出现 `tool_use`，进入工具执行
4. 把 `tool_result` 再塞回上下文
5. 继续下一轮，直到模型给出终态回答

这条链路在 [src/query.ts:1363](/mnt/e/claude-code-snapshot-backup/src/query.ts#L1363) 到 [src/query.ts:1408](/mnt/e/claude-code-snapshot-backup/src/query.ts#L1408) 很清楚。

**Tool 层是它的 agent 能力底座**
这里的 `Tool` 不是普通函数，而是统一契约。看 [src/Tool.ts:362](/mnt/e/claude-code-snapshot-backup/src/Tool.ts#L362)：

- `inputSchema`
- `description()`
- `call()`
- `isConcurrencySafe()`
- `isReadOnly()`
- `checkPermissions()`
- `validateInput()`

这说明框架不是“模型说要调用啥就直接调用”，而是每个工具都要经过 schema、权限、并发、安全边界。

工具池也不是固定全开放的：
- 全量内建工具列表在 [src/tools.ts:193](/mnt/e/claude-code-snapshot-backup/src/tools.ts#L193)
- 当前会话可见工具由 [src/tools.ts:271](/mnt/e/claude-code-snapshot-backup/src/tools.ts#L271) 动态裁剪
- 内建工具和 MCP 工具最终在 [src/tools.ts:345](/mnt/e/claude-code-snapshot-backup/src/tools.ts#L345) 统一组装

更关键的是，工具执行层自己做了调度：
- `runTools()` 会把工具调用分成“可并发的只读批次”和“必须串行的写操作批次” [src/services/tools/toolOrchestration.ts:19](/mnt/e/claude-code-snapshot-backup/src/services/tools/toolOrchestration.ts#L19) [src/services/tools/toolOrchestration.ts:91](/mnt/e/claude-code-snapshot-backup/src/services/tools/toolOrchestration.ts#L91)
- 单个工具调用由 `runToolUse()` 执行 [src/services/tools/toolExecution.ts:337](/mnt/e/claude-code-snapshot-backup/src/services/tools/toolExecution.ts#L337)

这已经很像一个小型 agent OS 了，不只是“function calling”。

**多 Agent 是怎么搭起来的**
多 agent 的关键不是单独一个模块，而是 `AgentTool + runAgent + Task` 这三层。

`AgentTool` 本身就是暴露给模型的一个工具。模型如果要“委派任务”，调用的不是外部框架 API，而是这个工具。[src/tools/AgentTool/AgentTool.tsx:226](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L226)  
它的输入里直接有：
- `description`
- `prompt`
- `subagent_type`
- `run_in_background`
- `isolation`
- `cwd`

见 [src/tools/AgentTool/AgentTool.tsx:81](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L81) 和 [src/tools/AgentTool/AgentTool.tsx:239](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L239)。

真正厉害的地方在 `runAgent()`：它不是简单“起一个子进程”，而是给子代理重新组一套运行上下文，然后再跑同一个 `query()` 循环。

- 重新生成 agent id、上下文、缓存、权限态 [src/tools/AgentTool/runAgent.ts:332](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/runAgent.ts#L332)
- 创建 subagent 专属 `ToolUseContext` [src/tools/AgentTool/runAgent.ts:700](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/runAgent.ts#L700)
- 再次进入 `query()`，也就是“子代理复用同一套 runtime” [src/tools/AgentTool/runAgent.ts:748](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/runAgent.ts#L748)

所以这里的 worker 不是特别神秘的东西，本质上就是“同一套 agent runtime 的递归实例”。

**Task 层负责把异步 agent 变成可管理对象**
一旦子代理不是前台同步跑完，就会落到 `Task` 体系里。

`TaskStateBase` 定义了统一任务元数据：`id/type/status/outputFile/notified` 等。[src/Task.ts:44](/mnt/e/claude-code-snapshot-backup/src/Task.ts#L44)

具体任务类型有：
- `local_agent` [src/tasks/LocalAgentTask/LocalAgentTask.tsx:116](/mnt/e/claude-code-snapshot-backup/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L116)
- `remote_agent` [src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:22](/mnt/e/claude-code-snapshot-backup/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L22)
- 还有 bash/workflow/monitor 等 [src/Task.ts:6](/mnt/e/claude-code-snapshot-backup/src/Task.ts#L6)

任务框架统一做三件事：
- 注册任务 [src/utils/task/framework.ts:77](/mnt/e/claude-code-snapshot-backup/src/utils/task/framework.ts#L77)
- 更新状态 [src/utils/task/framework.ts:48](/mnt/e/claude-code-snapshot-backup/src/utils/task/framework.ts#L48)
- 轮询/回收 [src/utils/task/framework.ts:154](/mnt/e/claude-code-snapshot-backup/src/utils/task/framework.ts#L154)

任务完成后，不是直接“回调父函数”，而是转成 `<task-notification>` 这种内部消息，再回流给主会话或 coordinator。`LocalAgentTask` 和 `RemoteAgentTask` 都这么干。[src/tasks/LocalAgentTask/LocalAgentTask.tsx:195](/mnt/e/claude-code-snapshot-backup/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L195) [src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:163](/mnt/e/claude-code-snapshot-backup/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L163)

**Coordinator 模式其实是“角色化的同一引擎”**
`coordinatorMode` 不是另一套 runtime，而是在同一 runtime 上切换 system prompt 和工具边界。

- coordinator system prompt 在 [src/coordinator/coordinatorMode.ts:111](/mnt/e/claude-code-snapshot-backup/src/coordinator/coordinatorMode.ts#L111)
- 它明确要求 coordinator 做研究拆分、并行调度、综合结果 [src/coordinator/coordinatorMode.ts:120](/mnt/e/claude-code-snapshot-backup/src/coordinator/coordinatorMode.ts#L120)
- coordinator 自己只保留少数管理工具：`Agent / TaskStop / SendMessage / SyntheticOutput` [src/constants/tools.ts:107](/mnt/e/claude-code-snapshot-backup/src/constants/tools.ts#L107)
- worker 能拿到的是另一组工具白名单 `ASYNC_AGENT_ALLOWED_TOOLS` [src/constants/tools.ts:55](/mnt/e/claude-code-snapshot-backup/src/constants/tools.ts#L55)
- coordinator 还会给自己注入“worker 有哪些工具可用”的上下文 [src/coordinator/coordinatorMode.ts:80](/mnt/e/claude-code-snapshot-backup/src/coordinator/coordinatorMode.ts#L80)

所以 coordinator / worker 不是 Bun 自带角色，也不是某个第三方 agent framework 的魔法，而是这套代码自己用“prompt + tool pool + task notification”硬搭出来的。

一句话总结：  
这项目的 “Agent 框架” 本质上是 `QueryEngine/query` 作为对话运行时，`Tool` 作为能力接口，`AgentTool/runAgent` 作为递归子代理机制，`Task` 作为异步控制平面，`coordinatorMode` 作为角色化策略层。

如果你愿意，我下一条可以继续把它画成一张“调用链时序图”，从用户输入一路画到 `AgentTool -> runAgent -> task-notification`。

# 第 2 轮

## 用户

我愿意

## 助手

**时序图**
```text
用户输入
  -> QueryEngine.submitMessage()
  -> processUserInput()
  -> query() 主循环
  -> 模型输出 assistant + tool_use(name="Agent")
  -> runTools()
  -> runToolUse()
  -> AgentTool.call()

AgentTool.call() 分两条路:

1. 本地子代理
   -> 计算 shouldRunAsync / workerTools / isolation
   -> registerAsyncAgent() 把它注册成 local_agent task
   -> 后台启动 runAsyncAgentLifecycle()
   -> runAsyncAgentLifecycle() 里再调用 runAgent()
   -> runAgent() 重新构造 subagent context
   -> runAgent() 再跑一遍 query()
   -> 子代理完成
   -> completeAgentTask()/failAgentTask()
   -> enqueueAgentNotification()
   -> 往全局 command queue 塞入 <task-notification>

2. 远程子代理
   -> teleportToRemote()
   -> registerRemoteAgentTask() 把它注册成 remote_agent task
   -> startRemoteSessionPolling()
   -> 轮询远程 session
   -> 完成后 enqueueRemoteNotification()
   -> 往全局 command queue 塞入 <task-notification>

之后:
  -> 主线程 query() 在下一轮 drain command queue
  -> 把 task-notification 转成 attachment / user-role 内部信号
  -> coordinator 读到结果，决定继续 SendMessage / 再派新 worker / 直接答用户
```

**关键节点**
入口是 [src/QueryEngine.ts:209](/mnt/e/claude-code-snapshot-backup/src/QueryEngine.ts#L209)。它先走 [src/utils/processUserInput/processUserInput.ts:85](/mnt/e/claude-code-snapshot-backup/src/utils/processUserInput/processUserInput.ts#L85)，再进入 [src/query.ts:219](/mnt/e/claude-code-snapshot-backup/src/query.ts#L219) 的主循环。模型一旦产出 `tool_use`，就会在 [src/query.ts:1380](/mnt/e/claude-code-snapshot-backup/src/query.ts#L1380) 调 `runTools()`，然后进 [src/services/tools/toolOrchestration.ts:19](/mnt/e/claude-code-snapshot-backup/src/services/tools/toolOrchestration.ts#L19) 和 [src/services/tools/toolExecution.ts:337](/mnt/e/claude-code-snapshot-backup/src/services/tools/toolExecution.ts#L337) 执行具体工具。

`AgentTool` 本身就是“派子代理”的工具。它在 [src/tools/AgentTool/AgentTool.tsx:239](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L239) 处理调用，在 [src/tools/AgentTool/AgentTool.tsx:567](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L567) 决定是否异步，在 [src/tools/AgentTool/AgentTool.tsx:577](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L577) 给 worker 重新组装工具池。这里很关键，因为它说明“子代理不是拿父代理剩下的上下文凑合跑”，而是重新获得一套自己的 `tools + permission mode + context`。

**本地子代理链**
如果走本地异步分支，`AgentTool` 会先在 [src/tools/AgentTool/AgentTool.tsx:688](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L688) 调 `registerAsyncAgent()`，对应实现是 [src/tasks/LocalAgentTask/LocalAgentTask.tsx:466](/mnt/e/claude-code-snapshot-backup/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L466)。这一步把子代理注册成 `local_agent` task，写进统一 task state。

随后它在 [src/tools/AgentTool/AgentTool.tsx:733](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L733) 后台启动 `runAsyncAgentLifecycle()`，而这个生命周期函数内部再去调用 `runAgent()`，见 [src/tools/AgentTool/agentToolUtils.ts:508](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/agentToolUtils.ts#L508)。`runAgent()` 又会重新构造 subagent 的 `ToolUseContext`，见 [src/tools/AgentTool/runAgent.ts:700](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/runAgent.ts#L700)，然后在 [src/tools/AgentTool/runAgent.ts:748](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/runAgent.ts#L748) 再跑一次 `query()`。

这就是整套架构最重要的点：子代理不是特殊黑盒，而是“同一套 runtime 的递归实例”。

子代理结束后，状态更新走 [src/tasks/LocalAgentTask/LocalAgentTask.tsx:412](/mnt/e/claude-code-snapshot-backup/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L412) 和 [src/tasks/LocalAgentTask/LocalAgentTask.tsx:437](/mnt/e/claude-code-snapshot-backup/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L437)。通知不是直接回调父函数，而是在 [src/tasks/LocalAgentTask/LocalAgentTask.tsx:197](/mnt/e/claude-code-snapshot-backup/src/tasks/LocalAgentTask/LocalAgentTask.tsx#L197) 组装 `<task-notification>` XML，再 `enqueuePendingNotification()` 入队。

**远程子代理链**
如果是 remote isolation，分支在 [src/tools/AgentTool/AgentTool.tsx:435](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L435)。它先 `teleportToRemote()`，然后在 [src/tools/AgentTool/AgentTool.tsx:456](/mnt/e/claude-code-snapshot-backup/src/tools/AgentTool/AgentTool.tsx#L456) 调 `registerRemoteAgentTask()`，对应实现是 [src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:386](/mnt/e/claude-code-snapshot-backup/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L386)。

远程任务不会本地递归跑 `query()`，而是启动轮询器 [src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:538](/mnt/e/claude-code-snapshot-backup/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L538)，通过 [src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:564](/mnt/e/claude-code-snapshot-backup/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L564) 不断拉远端事件。任务完成后在 [src/tasks/RemoteAgentTask/RemoteAgentTask.tsx:755](/mnt/e/claude-code-snapshot-backup/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L755) 入队远程 `task-notification`。

**通知怎么回流**
所有这些通知都先进统一 command queue。队列定义在 [src/utils/messageQueueManager.ts:41](/mnt/e/claude-code-snapshot-backup/src/utils/messageQueueManager.ts#L41)，task-notification 入队函数在 [src/utils/messageQueueManager.ts:142](/mnt/e/claude-code-snapshot-backup/src/utils/messageQueueManager.ts#L142)。

主线程或子代理自己的 `query()`，会在每轮结束前从队列取“发给自己”的命令，逻辑在 [src/query.ts:1570](/mnt/e/claude-code-snapshot-backup/src/query.ts#L1570)。主线程只吃 `agentId === undefined` 的通知，子代理只吃发给自己 `agentId` 的通知。取到后在 [src/query.ts:1580](/mnt/e/claude-code-snapshot-backup/src/query.ts#L1580) 转成 attachment，最后在 [src/query.ts:1642](/mnt/e/claude-code-snapshot-backup/src/query.ts#L1642) 从队列移除。

Coordinator 模式只是把这条链再“角色化”了一层。它的 system prompt 明确规定：`<task-notification>` 是 worker 结果，不是普通用户消息，见 [src/coordinator/coordinatorMode.ts:111](/mnt/e/claude-code-snapshot-backup/src/coordinator/coordinatorMode.ts#L111) 和 [src/coordinator/coordinatorMode.ts:142](/mnt/e/claude-code-snapshot-backup/src/coordinator/coordinatorMode.ts#L142)。

一句话总结：  
这套框架的本质是“父代理通过 `AgentTool` 生成一个新的 task；本地 task 会递归跑同一套 `query()` runtime，远程 task 会进入轮询器；两者最后都统一落成 `<task-notification>`，再通过 command queue 回流给 coordinator”。

如果你要，我下一条可以继续画“类图/模块图”，把 `QueryEngine / query / Tool / AgentTool / Task / coordinatorMode` 这几个模块的关系一次性梳理出来。
