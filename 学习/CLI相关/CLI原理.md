• 已帮你下载。

  你问的核心：为什么 CLI 能“长时间运行”，明明上下文有 token 限制？

  关键不是“无限上下文”，而是“有限上下文 + 持久化状态 + 自动压缩 + 按需重建上下文”。

  从 Kimi CLI 代码里可以直接看到这套机制：

  - 会话持久化到磁盘，支持恢复：context.jsonl、wire.jsonl、state.json
    参考 data-locations.md:77、data-locations.md:79、session.py:115、session_state.py:28
  - 支持 --continue / --session / /sessions 恢复会话，而不是一直依赖单次 prompt
    参考 sessions.md:13、sessions.md:21、sessions.md:29
  - 自动压缩上下文（不是等爆 token 才处理）
    触发条件：token_count >= max_context_size * ratio 或 token_count + reserved >= max_context_size
    参考 compaction.py:56、compaction.py:70、config.py:81
  - 主循环每步都检查是否该压缩，并更新 token 计数
    参考 kimisoul.py:408、kimisoul.py:559、kimisoul.py:567
  - 支持 checkpoint/revert，避免长链路执行崩掉后全盘丢失
    参考 context.py:68、context.py:80

  所以“长时间运行”本质上是：

  - 把长期记忆放在文件/状态存储，不放在单次上下文窗口里
  - 每一轮只塞“最有用的当前上下文”
  - 超预算时压缩、裁剪、回放、重建

  Codex CLI / Claude Code CLI 是不是类似？

  是，工程上通常都遵循同一套路（具体私有实现细节各家不会完全公开）：

  - 会话状态持久化（本地或服务端）
  - prompt 不是全历史，而是“近期对话 + 压缩摘要 + 当前任务相关代码片段”
  - token budget 管理（预留输出 token）
  - 自动摘要/压缩
  - 工具输出截断与结构化日志
  - 失败重试、超时、幂等和 checkpoint

  怎么从零实现“工程级”代码代理（而不只是 demo）
  可以按 5 层做，不要一开始追大而全：
 
  1. 最小闭环（MVP）
     用户输入 -> LLM -> 工具调用 -> 工具结果 -> 写入历史。
     先把消息协议定死（user/assistant/tool/system + usage）。
  2. 会话与持久化
     用 context.jsonl + state.json，支持 resume(session_id)。
     没有这层就做不了长期任务。
  3. 上下文预算管理
     维护 token_count，加 reserved_context_size 和 compaction_trigger_ratio。
     超阈值触发 compact()，保留最近 N 轮 + 摘要。
  4. 可靠性与安全
     工具权限审批、命令白名单/黑名单、超时、重试、取消、崩溃恢复、原子写文件。
  5. 工程化质量保障
     单元测试（消息/压缩/恢复）、集成测试（多步工具链）、回归快照测试（同输入同输出结构），再加可观测日志与 tracing。
 
  一个实用的主循环大概就是：
 
  for each turn:
    append(user_message)
    while step_count < max_steps_per_turn:
      if should_auto_compact(token_count, max_context): compact()
      result = llm.step(context)
      append(result.message)
      append(tool_results)
      update_token_count(result.usage)
      if no_tool_calls: break
  persist_session()
 
  如果你愿意，我下一步可以直接给你在当前目录搭一个“极简可运行版”CLI Agent骨架（Python），包括：
 
  6. session/context 持久化
  7. 自动压缩触发器
  8. 工具执行与审批钩子
  9. 一组最小测试，让它从 Day 1 就是工程化结构。
