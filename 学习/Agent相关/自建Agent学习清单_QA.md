# 自建 Agent 学习清单（Q&A）

## Q
我愿意

## A
很好，下面给你一份可以直接执行的“源码到文件级”学习清单。  
我按你前面选的主线做了排序：`smolagents -> LangGraph -> OpenHands`，对应从“最小抽象”到“生产级工程”。

这份清单按 **2026-03-02** 的仓库结构整理。

### 学习顺序
1. `smolagents`：先理解 Agent 最小闭环（最少抽象）
2. `langgraph`：再理解可控编排（状态机/图/中断/持久化）
3. `OpenHands`：最后看完整工程（控制器、事件总线、运行时、LLM 适配）

### 一、smolagents（3-4 天）
1. `README.md`  
   目标：看清它强调的“少抽象 + code agent”定位。
2. `docs/source/zh/guided_tour.md`（或 `docs/source/en/guided_tour.md`）  
   目标：先跑通官方心智模型。
3. `src/smolagents/agents.py`  
   重点：`MultiStepAgent`（约 268 行）、`run`（约 436 行）、`step`（约 782 行）、`ToolCallingAgent`、`CodeAgent`。  
   你要回答：一次任务从输入到最终答案经过了哪些状态。
4. `src/smolagents/tools.py`  
   重点：`Tool`、`tool` 装饰器、参数校验。  
   你要回答：工具 schema 如何限制模型乱调用。
5. `src/smolagents/models.py`  
   重点：统一模型接口、tool call 格式转换。  
   你要回答：不同模型提供商差异是怎么被“抹平”的。
6. `src/smolagents/memory.py`  
   重点：`ActionStep/PlanningStep/FinalAnswerStep`。  
   你要回答：会话状态如何结构化沉淀。
7. `src/smolagents/local_python_executor.py`  
   目标：看安全执行边界（AST 执行、超时、受限导入）。
8. `src/smolagents/mcp_client.py`  
   目标：理解 MCP 工具接入点。
9. `src/smolagents/monitoring.py`  
   目标：token/time 监控怎么落地。
10. `tests/test_agents.py`、`tests/test_tools.py`、`tests/test_models.py`、`tests/test_local_python_executor.py`  
   目标：用测试反推设计约束。

实操任务：
1. 新增一个你自己的 `Tool`（比如查天气/查库存），接到 `CodeAgent`。
2. 在 `agents.py` 打点记录每 step 的 token 和耗时。
3. 给工具调用失败加“可重试 + 最大步数保护”。

### 二、LangGraph（4-5 天）
1. `README.md`  
   目标：先明确它不是“现成 Agent”，而是“编排内核”。
2. `libs/langgraph/langgraph/graph/state.py`  
   重点：`StateGraph`（约 113 行）、`add_node`、`add_edge`、`compile`（约 1036 行）、`CompiledStateGraph`。  
   你要回答：图是如何被编译成可执行对象的。
3. `libs/langgraph/langgraph/graph/_node.py`  
   目标：节点函数签名与运行时注入规则。
4. `libs/langgraph/langgraph/graph/_branch.py`  
   目标：分支路径如何决策。
5. `libs/langgraph/langgraph/types.py`  
   重点：`Command`、`Send`、`Interrupt`、`RetryPolicy`、`StateSnapshot`。  
   你要回答：控制流和数据流如何解耦。
6. `libs/langgraph/langgraph/runtime.py`  
   目标：运行时上下文与调用期能力。
7. `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py`  
   重点：`create_react_agent`（约 278 行）、`call_model`、`should_continue`。  
   你要回答：预置 ReAct Agent 在图上是如何循环的。
8. `libs/prebuilt/langgraph/prebuilt/tool_node.py`  
   目标：工具节点执行模型调用后的 tool calls。
9. `libs/prebuilt/langgraph/prebuilt/tool_validator.py`  
   目标：工具参数校验链路。
10. `libs/langgraph/tests/test_state.py`、`test_runtime.py`、`test_interruption.py`、`libs/prebuilt/tests/test_react_agent_graph.py`  
   目标：看中断/恢复/边界行为。

实操任务：
1. 写一个 4 节点图：`plan -> retrieve -> act -> summarize`。
2. 在中间加 `interrupt` 做人工确认。
3. 开启 checkpoint（sqlite 或内存）并验证“中断后恢复”。

### 三、OpenHands（7-10 天）
1. `README.md`  
   目标：看产品形态和组件边界（SDK/CLI/GUI）。
2. `Development.md`  
   目标：确认本地开发运行方式（后端/前端/runtime）。
3. `openhands/README.md`  
   重点：官方给的核心类和控制流伪代码。  
   你要回答：为什么它用 EventStream 做主通信骨架。
4. `openhands/controller/agent.py`  
   目标：Agent 抽象接口（`step` 合约）。
5. `openhands/controller/agent_controller.py`  
   重点：`AgentController`（约 112 行）、`step`、`_step`、`on_event`、`_handle_action/_handle_observation`。  
   你要回答：主循环怎么处理异常、恢复和安全检查。
6. `openhands/events/event.py` + `openhands/events/stream.py`  
   重点：`Event`、`EventSource`、`EventStream.add_event/subscribe`。  
   你要回答：动作和观测如何被统一事件化。
7. `openhands/runtime/base.py` + `openhands/runtime/action_execution_server.py`  
   重点：`Runtime.run_action`、`ActionExecutor.run_action`。  
   你要回答：工具执行和沙箱边界如何拆分。
8. `openhands/agenthub/codeact_agent/codeact_agent.py`  
   重点：`CodeActAgent.step`、`_get_messages`、`_get_tools`。  
   你要回答：它如何把任务映射成函数调用。
9. `openhands/agenthub/codeact_agent/function_calling.py`  
   目标：工具定义、参数、响应转 action 的规则。
10. `openhands/llm/llm.py`、`async_llm.py`、`streaming_llm.py` + `tests/unit/controller/test_agent_controller.py`、`tests/unit/events/test_event_stream.py`、`tests/unit/agenthub/test_function_calling.py`  
   目标：看 LLM 适配层与核心回归测试。

实操任务：
1. 给 `CodeActAgent` 增加一个自定义工具（比如内部 API 查询）。
2. 在 `EventStream` 增加 trace_id 透传到 runtime。
3. 给 `AgentController` 新增一个“高风险 action 需人工确认”策略。

### 每天固定产出模板（强烈建议）
1. 今天读了哪 3-5 个文件（文件名 + 作用一句话）。
2. 今天确认了哪条主链路（例如 `agent.step -> event -> runtime -> observation`）。
3. 今天发现了哪些扩展点（tool、memory、checkpoint、policy）。
4. 今天改了什么小实验（最小可运行代码）。
5. 明天要验证的一个问题（只写 1 个）。

如果你愿意，我下一步可以直接给你一版“14 天打卡版”，每天精确到“看哪些文件 + 做哪一个 commit 级小实验”。
