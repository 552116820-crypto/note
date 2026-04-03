• 这些 .tmp 都是“验收/调试留痕文件”，不是业务运行依赖文件。
  1. .tmp_req263.txt
     含义：一次 POST /api/chat-messages 的 SSE 原始流（workflow_started/node_started/... 事件日志）。
     用途：排查某次链路节点命中和 run_id。
     结论：仅调试证据，可删。
  2. .tmp_req668_draft_post.resp.json
     含义：一次 POST /console/api/apps/.../workflows/draft 的返回体。
     内容：result/hash/updated_at（保存草稿成功信息）。
     结论：仅记录保存结果，可删。
  3. .tmp_req745_sse_success_after_rollback.txt
     含义：回滚后再跑 success 的 SSE 原始流。
     用途：证明“回滚后链路恢复正常”。
     结论：仅验收证据，可删。

  补充：

  - 这些文件不会被 Dify 工作流自动读取，不影响线上逻辑。
  - 你现在如果不需要审计留痕，可以全部清理 .tmp_*。
