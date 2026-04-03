• sys.user_id 本质上是 Dify 运行时注入的“应用用户 ID”：每个用户在使用工作流应用时，系统会给他分配/带入一个唯一标识，用来区分不同用户。(docs.dify.ai
  (https://docs.dify.ai/versions/3-0-x/zh/user-guide/workflow/variables?utm_source=openai))

  它的取值由“你从哪里/怎么触发这次运行”决定：

  - 通过 Service API 执行工作流（POST /workflows/run）时，请求体里必填的 user（User identifier）就是用户标识；因此你在工作流里读到的 sys.user_id 会由你传的
    这个 user 决定。(docs.dify.ai (https://docs.dify.ai/api-reference/workflow-execution/execute-workflow))
  - 通过发布的 WebApp / 网页嵌入时，如果你在嵌入配置里显式传 systemVariables.user_id，就会用你传的值做追踪；如果不传，则由系统自动给访客分配一个唯一 ID。
    (docs.dify.ai (https://docs.dify.ai/en/use-dify/publish/webapp/embedding-in-websites))
  - 想让 sys.user_id 对齐你业务系统的用户：用 API 调用时把 user 设成你自己的稳定用户 ID（或其哈希/匿名化值），或在嵌入配置里设置 systemVariables.user_id。
    (docs.dify.ai (https://docs.dify.ai/api-reference/workflow-execution/execute-workflow))

  补充：在 API 文档里也明确提到，Service API 创建的会话与 WebApp 创建的会话不共享；所以要复用同一用户的历史，入口要一致，并且持续使用同一个 user 标识。
  (docs.dify.ai (https://docs.dify.ai/api-reference/chatflow/send-chat-message))