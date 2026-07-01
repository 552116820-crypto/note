• 先把你遇到的现象“为什么一样 / 为什么拿不到 employee_code”讲清楚，再展开你问的“深度集成”。

  你现在 /jump?employee_code=000424 vs /jump?employee_code=000425 看起来一样的原因

  - /jump 最后跳的是同一个 Dify WebApp 地址：/chatbot/<token>，所以“页面 UI”一定一样；差异只可能体现在“它认为你是谁（user）”以及“落到哪段会话（conversation）”。        
  - 在同一个企业微信 WebView 里，Dify WebApp 往往会复用同一个浏览器存储/会话（localStorage/cookie）。如果 Dify 没有明确拿到一个新的 sys.user_id（或不会因为 URL 参数而刷
    新用户身份），它就会继续复用旧的会话，所以你换了 employee_code 也还是那段对话。
  - 你说“工作流拿不到 employee_code”，常见原因是：Dify 工作流只会读取它认可的输入变量/系统变量。纯粹把 employee_code=... 放在 URL query 上，并不一定会被 WebApp 当成工作
    流 inputs；即使第一次生效了，后续同会话也可能不再覆盖（inputs 通常在会话创建时固定）。w
  结论：你怀疑“Dify 通过 conversation_id 判断不同对话”没错，但conversation_id 是 Dify 生成并由它管理的；在 WebApp 模式下，你很难可靠地靠“换 URL 参数”来强制切换 user/   
  conversation。

  ———

  ## 深度集成形态（CRM 自己做聊天 UI + 后端转发 Dify API）到底是什么

  一句话：你不再打开 Dify 的 Web 页面，而是 CRM 自己画聊天窗口；每次发消息都由 CRM 后端去调用 Dify 的 Service API，拿到回复后再返回给前端显示。

  ### 1) 系统架构拆分

  - CRM 前端（企业微信内的 H5/PC 页）
      - 负责：聊天 UI、消息列表渲染、输入框、附件上传、流式输出展示
      - 不直接碰 Dify 的任何 Key
  - CRM 后端（你的服务）
      - 负责：鉴权（企业微信用户/CRM 登录态）、组装上下文（客户/线索/工单）、保存会话映射、调用 Dify API、把流式结果转发给前端
  - Dify（你的工作流/Agent）
      - 负责：LLM 推理、工作流编排、知识库、工具调用（可选）

  你仓库里的 amer_ketan/test.py 其实就是这种模式的“最小雏形”（它在后端用 DIFY_SERVICE_API_KEY 调 /v1/chat-messages）。可以从它直接演进。

  ### 2) “用户是谁”和“会话怎么分”是深度集成的核心
  深度集成里你要自己决定两件事：

  A. user（用户标识）用什么

  - 建议用企业微信 userid 或你们内部 employee_code（如 000424），但必须是“稳定且唯一”的。
  - 在调用 Dify API 时，把它放到请求里的 user 字段（test.py 就是 user: f"user_{employee_code}" 这种做法）。

  B. conversation_id 存在哪里

  - Dify 会在第一次对话返回 conversation_id。
  - 你需要在 CRM 数据库建一张表保存映射，例如：
      - (tenant_id, crm_user_id, entity_type, entity_id) -> dify_conversation_id
  - 这样就能做到：
      - 同一个销售在同一个客户卡片里：永远是同一段对话
      - 同一个销售切换到另一个客户：自动进入另一段对话（新的或历史的）

  这是 WebApp /jump 很难做到的“按业务对象分会话”。

  ### 3) 一次消息的典型调用流程（你需要实现什么）

  以“客户详情页右侧有个AI助手”为例：

  1. 前端打开客户 customer_id=123，请求后端：GET /assistant/session?customer_id=123
  2. 后端查库是否已有 (user, customer_id=123) 的 conversation_id
  3. 用户发消息 “帮我总结这个客户最近跟进”
  4. 前端 POST /assistant/message：
      - conversation_id（可空）
      - customer_id
      - message
  5. 后端组装 Dify 请求并转发：
      - user：稳定用户 id（employee_code / wecom userid）
      - conversation_id：如果已有就带上；没有就不带（让 Dify 创建新会话）
      - inputs：建议只在“创建新会话时”传入固定上下文（例如 customer_id、employee_code）
      - query：用户输入
      - response_mode：blocking 或 streaming
  6. Dify 返回：
      - conversation_id（若新建）
      - answer / 流式事件
  7. 后端把回复回传给前端，并在新建会话时把 conversation_id 落库

  ### 4) “为什么工作流拿不到 employee_code”在深度集成里怎么彻底解决

  深度集成你有两条可靠路子（通常两条都做）：

  - 把 employee_code 作为 Dify 的 user：这样工作流里通常能直接用 sys.user_id（系统变量）识别用户
  - 把 employee_code 定义成 Dify App 的输入变量 key（比如就叫 employee_code），然后在第一次消息里通过 inputs: { employee_code: "000424" } 传进去，在工作流里用输入变量引 
    用它

  > 注意：如果你复用同一个 conversation_id，很多情况下 inputs 不会被后续消息“更新覆盖”。所以“用户/客户上下文会变”的场景，要么每个客户独立一个会话，要么把动态上下文通过工
  > 具节点实时拉取（见下一条）。

  ### 5) 动态 CRM 上下文怎么喂给 Dify（两种主流做法）

  - 方式 1：后端每次调用前，把客户摘要拼进 query
      - 简单粗暴，但 prompt 可能变长、成本高、也更容易把敏感字段带进去
  - 方式 2（更推荐）：Dify 工作流里加一个 HTTP 工具节点回调 CRM
      - 你的后端提供一个 GET /crm/context?customer_id=123（带鉴权）
      - Dify 在需要时拉取最新客户信息/跟进记录
      - 好处：上下文永远最新，且更可控（可按字段脱敏）

  ———

  ## 这种深度集成的优缺点（你做决策用）

  - 优点
      - UI/交互完全在 CRM 内，体验最好
      - 能做到“按客户/线索/工单分会话”
      - DIFY_SERVICE_API_KEY 永远不出后端，安全合规更好
      - 更容易做审计、权限控制、敏感字段脱敏
  - 成本/坑
      - 你要自己做流式转发（SSE/WebSocket）、消息存储、附件、重试、限流
      - 需要自己维护“会话映射表”和“用户身份体系”
      - Service API 创建的会话不一定和 Dify WebApp 的会话互通（所以要选定一种渠道为主）

  ———

  如果你愿意我直接按你们 CRM 的实际形态给一套“落地接口设计 + 数据表 + 最小可跑代码”，我需要你回答 3 个信息：

  1. CRM 前端是 H5 还是 PC web（企业微信内）？要不要流式输出？
  2. 会话要按什么维度隔离：仅按员工，还是按（员工 + 客户/线索/工单）？
  3. Dify 你用的是 Chat App 还是 Workflow App（工作流）？目前输入变量里有没有定义 employee_code 这个 key？
