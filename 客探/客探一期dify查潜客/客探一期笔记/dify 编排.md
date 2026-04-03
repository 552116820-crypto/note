• 你这个诉求本质是“把 CRM 的动态数据当成上下文”，但又不想每轮对话都打一次 CRM API。这里我建议你不要用“Dify 知识库”来做这个缓存层，而是用 Chatflow 的会话 变量
  （Conversation Variables）+ 首轮初始化（Bootstrap）+ 按需懒加载 来编排。

  原因与编排方式我按可落地的方式给你。

  1) 先纠正一个认知：不建议用知识库缓存“每个销售的客户数据”

  - Dify 知识库（Dataset）是“全局共享的文档检索库”，适合产品资料、SOP、话术、政策等相对静态内容；不适合把“每个 sales 的客户清单/客户详情”塞进去当缓存。
  - 你确实可以用知识库 API 维护文档，但它的 API Key 一般对同账号下可见知识库有很大操作权限，需要非常注意数据安全隔离。(docs.dify.ai (https://docs.dify.ai/
    versions/3-0-x/en/user-guide/knowledge-base/knowledge-and-documents-maintenance/maintain-dataset-via-api?utm_source=openai))
  - 更关键的是：把 CRM 数据写入知识库意味着“文档更新→分段→Embedding→索引”，这条链路不适合“进会话就更新”的实时场景（有排队/索引耗时），也容易带来跨销售数据混
    入风险。

  结论：知识库用来放静态知识；CRM 动态数据用会话变量/CRM 后端缓存。

  ———

  2) Dify Chatflow 里做“只初始化一次、后续复用”的标准方法：会话变量 + Variable Assigner
  Dify Chatflow 支持 会话变量，在同一个 conversation 内跨多轮对话持久存在；写入需要 Variable Assigner 节点。(docs.dify.ai (https://docs.dify.ai/versions/3-0-
  x/en/user-guide/workflow/variables?utm_source=openai))
  同时你还能用 sys.dialogue_count 判断“这是第几轮对话”，从而在第 1 轮做初始化。(docs.dify.ai (https://docs.dify.ai/versions/3-0-x/en/user-guide/workflow/
  variables?utm_source=openai))

  ———

  3) 推荐的 Chatflow 编排骨架（节点级别）
  你可以按下面这个“骨架”搭起来，然后把“客户查询/代办/拜访总结”分支挂上去。

  A. 会话变量先建好（Conversation Variables）
  建议至少这些：

  - cv.cache_ready (Boolean) —— 是否已初始化
  - cv.cache_version (String/Number) —— CRM 数据版本号/时间戳（用于判断是否需要刷新）
  - cv.customers_index (Array[Object]) —— “客户索引”小数据（只放够用字段）
  - cv.todos_open (Array[Object]) —— 未完成代办摘要（可选）
  - cv.selected_customer_id (String) —— 当前会话聚焦的客户（销售常用）
  - cv.customer_detail_cache (Object) —— 客户详情缓存（map：{customer_id: {...}}，按需填充，别一次性拉全量）

  B. 顶部加一个“初始化/刷新判断”分支（If/Else）
  条件建议用三段逻辑（满足任一就刷新）：

  1. 首轮对话：sys.dialogue_count == 1 (docs.dify.ai (https://docs.dify.ai/versions/3-0-x/en/user-guide/workflow/variables?utm_source=openai))
  2. 没初始化过：cv.cache_ready != true
  3. 外部强制刷新：start.force_refresh == true（这个是你在 Dify 的 Start/User Input 里加一个布尔输入，CRM 后端每次“打开 AI 面板”可置 true）

  C. 初始化分支：HTTP Request → Variable Assigner

  4. HTTP Request 节点调用你的 CRM 后端（建议新做一个聚合接口）：

  - GET /internal/dify/bootstrap?sales_id={{sys.user_id}}
  - Header 带服务端固定 token：X-Dify-Tool-Token: ...
  - HTTP Request 节点支持在 URL、Header、Body 里做变量替换。(docs.dify.ai (https://docs.dify.ai/en/guides/workflow/node/http-request?utm_source=openai))

  2. CRM 返回一个“轻量快照”（重点：别返回客户全字段）
     建议返回结构：

  - version：比如客户数据最后更新时间或递增版本
  - customers_index：数组，每条只要 {id, name, alias, stage, tags, last_contact_at} 这种级别
  - todos_open：数组摘要（可选）
  - （可选）default_customer_id：如果 CRM 当前页面就在某客户详情，可以让 CRM 顺便返回当前客户

  3. Variable Assigner 节点把返回写入会话变量：(docs.dify.ai (https://docs.dify.ai/en/use-dify/nodes/variable-assigner?utm_source=openai))

  - cv.cache_ready = true（overwrite）
  - cv.cache_version = bootstrap.version（overwrite）
  - cv.customers_index = bootstrap.customers_index（overwrite）
  - cv.todos_open = bootstrap.todos_open（overwrite）
  - （可选）cv.selected_customer_id = bootstrap.default_customer_id（overwrite）
  - （可选）如果你怕旧详情过期：cv.customer_detail_cache 清空（clear）

  D. 非初始化分支：直接进入“意图识别/参数提取”
  这一步不要打 CRM。

  - 用一个 LLM 节点（或 Question Classifier）输出严格 JSON：
      - intent: customer_search | customer_detail | todo_create | todo_list | todo_done | visit_summary_generate | visit_summary_save | refresh
      - customer_name_or_hint
      - need_detail（是否必须查实时详情）
      - fields（想看的字段：手机号/地址/商机阶段等）

  E. “客户解析”优先用缓存索引，不命中再 HTTP

  - 先在 cv.customers_index 里做 name/alias 模糊匹配（可以用 Code 节点做简单匹配；或让 LLM 在提示词里只基于索引做匹配）
  - 命中唯一：写入 cv.selected_customer_id
  - 命中多个：让用户选（不要立刻查详情，避免无效 HTTP）
  - 完全不命中：再调用 CRM 的搜索接口（一次）补齐

  F. 只有需要实时详情时才打 CRM，并把结果写进 cv.customer_detail_cache
  例如当 intent 是 customer_detail 或 visit_summary_generate：

  - If：cv.customer_detail_cache[cv.selected_customer_id] 存在且未过期 → 直接用缓存
  - Else：HTTP Request：GET /internal/dify/customer/{{cv.selected_customer_id}}?sales_id={{sys.user_id}}
      - 返回后 Variable Assigner 写入 cv.customer_detail_cache 对应 key
        这样同一个客户连续追问 5 次，你只打一次 HTTP。

  G. 最终回答：静态知识用知识库，动态数据用变量/HTTP结果

  - “产品怎么介绍、价格策略、标准话术”→ 走 Dify 知识库检索
  - “客户归属/联系人/商机/代办/拜访总结”→ 走会话变量 + CRM 工具 API

  ———

  4) 你说“进入会话更新”，但你又是“一销售一个固定会话”——怎么处理刷新？
  这里要注意：如果你真的把一个 sales 永久绑定同一个 conversation_id，那 sys.dialogue_count == 1 只会发生一次。(docs.dify.ai (https://docs.dify.ai/versions/3-
  0-x/en/user-guide/workflow/variables?utm_source=openai))
  所以你需要“进入 CRM 面板即刷新”的等效机制，推荐两种：

  - 方式 1（最简单）：每次打开 AI 面板，CRM 后端给 Dify 这次调用带 inputs.force_refresh=true
      - 不需要新建会话
      - 也不需要你在 Dify 里知道“打开事件”
      - 缺点：用户每次第一次问话会触发一次 bootstrap HTTP（但你本来就是想“进来更新一次”，这正好）
  - 方式 2（更省）：加一个 TTL 刷新
      - 会话变量里存 cv.cache_version 或 cv.cache_updated_at
      - CRM bootstrap 接口支持返回 version
      - 如果 version 未变化就不重新下发大 payload（甚至返回空/304）
      - Dify 还是会发一次请求，但成本很小（而且 HTTP 节点支持 header/query/body 自定义，做这种版本判断很方便）。(docs.dify.ai (https://docs.dify.ai/en/
        guides/workflow/node/http-request?utm_source=openai))

  ———

  5) 给你一个“最少 HTTP”的落地建议（你可以直接照着做）

  - 只在 3 种情况打 CRM：
      1. 首次进入/强制刷新：/bootstrap（一次）
      2. 用户明确要看某客户实时详情：/customer/{id}（按需）
      3. 写操作：创建/完成代办、保存拜访总结（按需）
  - 其它“列我名下客户、我有哪些代办、客户名单里有没有XX、帮我找名字类似的客户”都只用 cv.customers_index / cv.todos_open，不走 HTTP。

  ———

  如果你愿意，我可以基于你现有的 CRM 数据（客户表字段、代办字段、拜访记录字段）把“bootstrap 返回哪些字段最合适 / Chatflow 每个节点提示词怎么写 / JSON schema
  怎么定”给你一套最小可用模板。

  你现在销售名下客户大概是一个量级：几十、几百、还是上千？（决定 customers_index 该多“瘦身”，以及是否需要分页/分片缓存）


## 功能添加

• 把 3 个业务分支“挂上去”的关键做法是：在你现有骨架（初始化/缓存判断 → 路由 → Switch）后面，每个分支都按同一套模式接节点：

  意图路由(Switch) → 槽位抽取(LLM结构化JSON) → 缓存命中判断(If) → 必要时HTTP Request → Variable Assigner写回会话变量 → 渲染回复(LLM/Answer)

  下面给你一套可以直接照着在 Dify Chatflow 里连线的“分支挂载模板”。

  ———

  ## 0）先统一：路由输出格式（让 Switch 好接）

  在路由 LLM 节点里输出一个统一 JSON（用 Structured Output / JSON Schema，避免跑偏），建议最少这些字段：

  {
    "intent": "customer_search|customer_detail|todo_list|todo_create|todo_done|visit_summary_generate|visit_summary_show|refresh|other",
    "customer_hint": "张三|C123|手机号后四位|空",
    "todo_hint": "完成第2条|T10086|空",
    "visit_hint": "今天拜访|V7788|空",
    "need_realtime": false
  }

  然后你用 Switch(intent) 把分支挂出去：

  - customer_search / customer_detail → 客户分支
  - todo_list / todo_create / todo_done → 代办分支
  - visit_summary_generate / visit_summary_show → 拜访总结分支
  - refresh → 强制刷新分支
  - other → 普通闲聊/静态知识库分支

  ———

  ## 1）客户查询分支怎么挂（Search / Detail 两层）

  ### 1.1 客户搜索（customer_search）

  目标：优先用 customers_index（会话缓存）解决；不命中再 HTTP；命中后写 selected_customer_id。

  节点链路建议：

  1. LLM：ExtractCustomerQuery

  - 输入：用户原话 + conversation.customers_index（注意：只给“瘦索引”，比如最近/高频100条，不要上千条塞进prompt）
  - 输出 JSON：

  {
    "query": "张三",
    "match_strategy": "local_first",
    "top_k": 5
  }

  2. Code(可选) / LLM：LocalMatchCustomers

  - 如果你有 Code 节点：用简单 contains/拼音首字母/手机号后四位匹配。
  - 输出：

  {
    "matched": [
      {"id":"C1","name":"张三公司","stage":"意向"},
      {"id":"C2","name":"张三(个体)","stage":"跟进"}
    ]
  }

  3. If：matched_count == 1 ?

  - 是 → Variable Assigner：set selected_customer_id，并回复“已选中客户XXX”
  - 否（>1） → Answer：让用户选一个（给出编号 + 关键信息），并把列表写入 conversation.last_customer_candidates
  - 否（=0） → 走 HTTP 搜索

  4. HTTP Request：CRM Customer Search

  - POST /internal/dify/customer_search
  - Body（关键：sales_id 用系统变量）：
      - sales_id: {{sys.user_id}}
      - keyword: {{extract.query}}
  - 返回后：
      - Variable Assigner：更新 conversation.last_customer_candidates
      - 如果结果唯一：同时写 conversation.selected_customer_id

  5. Answer：返回候选/已选中

  - 候选多：提示用户“回复 1/2/3 选中”
  - 这一步你可以让路由器支持一种 intent：customer_select_index（用户回“2”时走这个），或者在 customer_search 分支里加一个“如果用户输入是纯数字且存在
    last_customer_candidates 就直接选中”的小判断。

  ### 1.2 客户详情（customer_detail）

  目标：已有 selected_customer_id 时，先读 customer_detail_cache，没缓存才 HTTP。

  节点链路建议：

  1. If：conversation.selected_customer_id 为空？

  - 空 → Answer：让用户先指定客户（或提示“你要查哪个客户？”）
  - 不空 → 继续

  2. If：cache_hit？

  - 规则例子：conversation.customer_detail_cache[selected_id] 存在 && 未过期
  - 命中 → 直接用缓存渲染
  - 未命中 → HTTP 拉详情

  3. HTTP Request：CRM Customer Detail

  - GET /internal/dify/customers/{{conversation.selected_customer_id}}?sales_id={{sys.user_id}}

  4. Variable Assigner：写入缓存

  - conversation.customer_detail_cache[selected_id] = response.detail
  - 可选：conversation.customer_detail_cache_expire_at[selected_id] = now+10min

  5. LLM/Answer：渲染客户详情

  - 只输出必要字段（避免把大量隐私字段塞回模型上下文）

  ———

  ## 2）代办事项分支怎么挂（List / Create / Done）

  ### 2.1 列代办（todo_list）

  目标：优先用 conversation.todos_open（bootstrap来的摘要）直接回；只有用户说“刷新/最新”才 HTTP。

  节点链路：

  1. If：need_realtime == true OR todos_open为空 OR 过期？

  - 否 → 直接渲染 todos_open
  - 是 → HTTP Request：GET /internal/dify/todos?status=open&sales_id={{sys.user_id}}

  2. Variable Assigner

  - 更新 conversation.todos_open
  - 同时写 conversation.last_todo_list = todos_open（后面“完成第2条”要用）

  3. Answer：列表渲染

  - 建议输出格式固定：编号 + 标题 + 截止时间 + 客户简称 + 优先级
  - 并提示：“回复 完成 2 / 改期 2 明天下午 / 新增代办 ...”

  ### 2.2 新增代办（todo_create）

  目标：从话里抽字段；缺字段就追问；创建成功后本地把 todos_open append（避免再拉一次列表）。

  节点链路：

  1. LLM：ExtractTodoCreate

  - 输出 JSON：

  {
    "title": "下周二回访确认需求",
    "due_at": "2026-02-10 10:00:00",
    "priority": "P2",
    "customer_id": "AUTO|C123|空"
  }

  - 规则：没明确 customer_id 时，优先用 conversation.selected_customer_id，否则返回 AUTO 让后续节点决定。

  2. If：title 缺失？ or due_at 缺失？

  - 缺失 → Answer：追问（截止时间/关联客户）
  - 不缺失 → 继续

  3. HTTP Request：POST /internal/dify/todos

  - Body：
      - sales_id: {{sys.user_id}}
      - customer_id: {{resolved_customer_id}}
      - title / due_at / priority

  4. Variable Assigner：更新缓存

  - conversation.todos_open = [newTodo] + conversation.todos_open（或 append）
  - conversation.last_created_todo_id = newTodo.id

  5. Answer：确认创建

  - “已创建代办：… 截止…（关联客户…）”

  ### 2.3 完成代办（todo_done）

  目标：支持“按编号完成”和“按 todo_id 完成”。

  节点链路：

  1. LLM：ExtractTodoDone

  {"todo_id":"T123|空","todo_index":2}

  2. If：todo_id 为空且 todo_index 有值？

  - 用 conversation.last_todo_list[todo_index] 解析出真实 todo_id
  - 如果 last_todo_list 不存在 → 让用户先“列一下代办”或提供 ID

  3. HTTP Request：PATCH /internal/dify/todos/{todo_id}

  - Body：{"status":"done","sales_id":"{{sys.user_id}}"}

  4. Variable Assigner：从 todos_open 移除/标记 done

  - 这一步如果没 Code 节点做数组更新，就简单粗暴点：把 need_realtime=true 写入，让下一次 todo_list 触发刷新

  5. Answer：确认完成

  ———

  ## 3）拜访总结分支怎么挂（Generate / Show）

  ### 3.1 生成拜访总结（visit_summary_generate）

  目标：拿到材料（visit_id 或 notes）→ 生成结构化总结 JSON → 存 CRM →（可选）批量创建 follow-up 代办 → 更新本地缓存。

  节点链路：

  6. LLM：ExtractVisitSummaryRequest

  {
    "customer_id": "AUTO|C123|空",
    "visit_id": "V7788|空",
    "visit_at": "2026-02-03 15:00:00|空",
    "notes": "空或用户粘贴的纪要",
    "create_follow_up_todos": true
  }

  7. Resolve customer_id

  - AUTO → 优先用 conversation.selected_customer_id
  - 仍为空 → 追问“针对哪个客户？”

  3. If：visit_id 有？

  - 有 → HTTP Request：GET /internal/dify/visits/{visit_id}?sales_id={{sys.user_id}} 拿原始拜访记录
  - 没有但 notes 有 → 直接用 notes
  - 两者都没有 → Answer：让用户粘贴拜访纪要/或选择一次拜访记录

  4. LLM：GenerateSummaryJson（强制结构化输出）
     建议输出：

  {
    "title": "与XX客户拜访总结",
    "key_points": ["..."],
    "needs": ["..."],
    "risks": ["..."],
    "next_steps": ["..."],
    "follow_up_todos": [
      {"title":"发送方案报价","due_at":"2026-02-05 18:00:00","priority":"P1"}
    ],
    "summary_text": "一段给人看的总结"
  }

  5. HTTP Request：POST /internal/dify/visit-summaries

  - Body：
      - sales_id: {{sys.user_id}}
      - customer_id: ...
      - visit_id: ...
      - summary_json: {{summary_json}}

  6. If：follow_up_todos 非空 && create_follow_up_todos == true

  - HTTP Request：POST /internal/dify/todos/batch（sales_id + customer_id + todos数组）

  7. Variable Assigner：更新缓存

  - 把新建代办合并进 conversation.todos_open（至少把 title/due_at 摘要塞进去）
  - 可选：conversation.last_visit_summary_id = saved.id

  8. Answer：输出总结 + 已创建代办清单

  - 输出“总结要点 + 下一步 + 代办列表”，让销售能直接执行

  ### 3.2 查看最近总结（visit_summary_show）

  节点链路：

  - HTTP Request：GET /internal/dify/visit-summaries?customer_id=...&limit=1&sales_id={{sys.user_id}}
  - Answer：渲染

  （这个分支不建议缓存太久，因为总结本身不频繁；也可以缓存 conversation.last_summary_* 仅做“刚生成后立刻追问”场景。）

  ———

  ## 4）把“减少HTTP”做实：每个分支都加两条规则

  - 只缓存“索引/摘要”：客户只缓存轻字段（name/stage/last_contact），代办只缓存 open 的摘要；详情/拜访原文按需拉。
  - 写操作后本地回填：新增代办/生成总结后，把结果写回会话变量，避免用户下一句“再列一下”又触发 HTTP。

  ———

  如果你把你现在骨架里这几个变量名（你实际用的名字）和你 CRM 已经有的 6 个接口路径（bootstrap / customer_search / customer_detail / todos / visits / visit-
  summaries）贴出来，我可以按你现有命名直接给你“每个节点该填什么（URL、headers、body、输出映射、Variable Assigner 映射、Switch 条件）”的逐项清单。
