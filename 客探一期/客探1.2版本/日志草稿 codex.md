• 这个需求本质上不是“Dify 跨页面通信”，而是“你自己的业务系统里要有一份按销售员 + 日期维护的日志草稿”。
  最稳的做法是：Dify只负责生成拜访总结/代办，日志草稿放到你自己的后端或CRM，日志页和拜访页都读写这份草稿。

  为什么不能只靠 Dify 页面本身

  - 如果你现在用的是 Workflow 应用，它的变量是单次执行的，跑完就结束，不能天然支持“今天多次录入自动累加”。
  - 如果改成 Chatflow，会话变量只能在同一个会话里持续，不适合直接当“日志页”的正式数据源。
  - Dify 官方文档明确写了：Service API 创建的会话和 WebApp 里的会话是隔离的，所以你不能指望“嵌入的 Dify 页面”和“你后端再去调 Dify API”天然共享同一份会话数据。
    来源：
    https://docs.dify.ai/api-reference/chatflow/send-chat-message
    https://docs.dify.ai/api-reference/conversations/get-conversation-history-messages

  你这个需求的推荐实现

  1. 拜访总结页里，用户输入客户名称。
  2. 小程序把 wecom_userid/sales_id/customer_id 发给你自己的后端。
  3. 后端去调用 Dify 工作流/Chatflow，拿到结构化结果，不要只拿一段 Markdown。
  4. Dify 在输出里返回：

  {
    "customer_id": "C888",
    "customer_name": "某某公司",
    "visit_time": "2026-03-20 15:30:00",
    "summary": "本次拜访总结……",
    "todos": [
      {"title": "下周二回访", "due_at": "2026-03-25 10:00:00", "priority": "P2"}
    ],
    "source_run_id": "dify_workflow_run_id_or_visit_id"
  }

  5. 后端执行 append draft：
     以 sales_id + log_date + status=draft 找当天草稿，没有就创建，有就追加。
  6. 日志页打开时，请求 GET /api/log-drafts/today，按客户分组展示。
  7. 用户点“保存”时，再执行 POST /api/logs/save-from-draft，生成正式日志，草稿状态改成 saved。

  后端数据建议

  - daily_log_drafts
      - id
      - sales_id
      - log_date
      - status：draft/saved
      - content_json
      - version
      - saved_log_id
  - daily_log_draft_items
      - draft_id
      - customer_id
      - customer_name
      - summary
      - todos_json
      - visit_time
      - source_run_id
      - idempotency_key

  关键点

  - 分组不要只按 customer_name，内部最好按 customer_id，customer_name只用于展示。
  - 一定要做幂等，避免用户重复触发后重复追加。idempotency_key 可以用 sales_id + visit_id，或者 sales_id + source_run_id。
  - “当天已保存后又新增一条拜访总结”这条规则你要提前定清楚。更稳的是：保存后新建一个新的 draft version，不要直接改已保存正文。
  - 如果你现在是“直接把 Dify WebApp 嵌到企业微信小程序里”，这是能跑 MVP，但不适合长期做这种跨页面草稿能力。最终建议是：小程序/前端自建页面 + 后端代理 Dify API。
  - Dify API Key 只能放后端，不要放小程序里。

  如果你想最少改动先上线

  - 保留现在的 Dify 页面。
  - 在 Dify 工作流最后加一个 HTTP Request 节点，直接调你后端的 POST /api/log-drafts/append。
  - 日志页只从你后端读草稿，不去读 Dify 页面内容。

  和 Dify 相关的能力边界

  - 嵌入时可以传 user_id/conversation_id/inputs：
    https://docs.dify.ai/versions/3-0-x/en/user-guide/application-publishing/embedding-in-websites
  - Chatflow 会话变量可以跨多轮保存，但只适合同一会话内临时记忆：
    https://docs.dify.ai/versions/3-0-x/en/user-guide/workflow/variables
    https://docs.dify.ai/en/use-dify/nodes/variable-assigner

  如果你愿意，我下一步可以直接给你两份可落地内容：

  1. 这套“日志草稿”后端接口定义
  2. 你现有 Dify 工作流里该加哪些节点，怎么传字段
