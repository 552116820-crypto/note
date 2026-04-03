http://dify.amer.cn/v1/datasets/04ffc526-1ae9-4a79-af9c-227e7ad5ec78/document/create-by-text


Bearer dataset-rTJy08bc1IW2lTVaUCDyPdja



• 结论

  - “一个销售一份客户快照，不要每轮对话都打 CRM”这个目标，最稳的实现其实不是用 Dify 知识库，而是用你已经在 Chatflow 里搭好的“进入会话初始化 → 写入会话变 量”的
    缓存；这样既不会触发知识库索引延迟，也不会有跨销售检索串数据的风险。
  - 如果你坚持用知识库做缓存：创建/更新（upsert）逻辑可以做，但要额外处理 2 件事：
      1. 判断“存在则更新，不存在则创建”；
      2. 必须做检索隔离（否则知识检索默认会搜整个知识库，可能把别的销售的客户信息检索出来）。

  ———

  推荐方案（不用知识库，直接满足“不每轮 HTTP”）

  1. CRM 登录/进入 AI 面板时：你已经能拿到 employee_code + 30 个客户数据，那就把“客户索引快照”整理成一个文本/JSON（只放常用字段，别放 passport 这种敏感字
     段）。
  2. CRM（PHP 后端）调用 Dify 的 POST /chat-messages，在 inputs 里传：
      - force_refresh=true
      - customer_snapshot=<你整理好的文本/JSON>
      - cache_version=<例如 CRM 数据更新时间 or hash>
      - user=<employee_code>（确保一个销售固定会话）
  3. Chatflow 顶部增加判断：
      - if force_refresh == true OR conversation.cache_version != inputs.cache_version
      - 然后用 Variable Assigner 把 customer_snapshot 写入 conversation.customer_snapshot，把 cache_version 写入 conversation.cache_version
  4. 后续对话全部只用 conversation.customer_snapshot 做客户列表/基本信息回答；只有用户明确要“最新/详情”时才按需调用 CRM。

  这条路的核心收益：你实现的“缓存”是在会话变量里，每次进入系统只更新一次，完全符合你“不每次对话都 HTTP”的目标，而且不会被知识库检索的“全库搜索”坑到。

  ———

  如果你坚持用知识库（一个销售=一个文档）怎么做
  建议把“同步知识库”这件事放在 CRM(PHP) 后端做，而不是在 Chatflow 里用 HTTP 节点去调用 Dify 自己的 Knowledge API（原因：Knowledge API token 权限很大，且更 新
  触发索引会拖慢首轮体验）。(docs.dify.ai (https://docs.dify.ai/versions/3-0-x/en/user-guide/knowledge-base/knowledge-and-documents-maintenance/maintain-
  dataset-via-api))

  ### 1) Upsert 逻辑（存在则更新，不存在则创建）

  你可以用“文档名唯一化 + 查询文档列表 keyword”来判断是否存在：

  - 文档名建议固定：sales_customers_{employee_code}（确保唯一且可搜索）
  - 查是否存在：GET /v1/datasets/{dataset_id}/documents?keyword={docName}&page=1&limit=20（API 支持按文档名 keyword 搜索）。(docs.dify.ai (https://
    docs.dify.ai/api-reference/documents/get-the-document-list-of-a-knowledge-base))
  - 找到后拿 document_id → 更新
  - 找不到 → 创建

  创建/更新接口（官方 API Reference）：

  - 创建文本文档：POST /v1/datasets/{dataset_id}/document/create-by-text (docs.dify.ai (https://docs.dify.ai/api-reference/documents/create-a-document-from-
    text))
  - 更新文本文档：POST /v1/datasets/{dataset_id}/documents/{document_id}/update-by-text (docs.dify.ai (https://docs.dify.ai/api-reference/documents/update-a-
    document-with-text?utm_source=openai))

  ### 2) 强烈建议：加“内容 hash/version”避免每次都更新

  你说每个销售 30 个客户——如果你每次进入都更新文档，会带来：

  - 反复 embedding/索引（成本+延迟）
  - 文档 indexing 期间检索到旧数据/不稳定

  做法：

  - CRM 端把生成的 doc_text 做一个 sha1(doc_text) 或用 CRM 的 updated_at 作为 cache_version
  - 你自己建表保存：employee_code -> document_id -> last_hash(last_version)
  - 登录时如果 hash 没变化：直接跳过知识库更新（最快、最省）

  ### 3) 处理索引异步：不要“更新完立刻就问”指望一定生效

  创建/更新接口会返回一个 batch，可以用它查索引进度：

  - GET /v1/datasets/{dataset_id}/documents/{batch}/indexing-status (docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-document-embedding-status-progress?utm_source=openai))

  推荐策略：

  - 同步更新走异步：先让销售进系统正常用（仍可用上一次索引内容）
  - 或者你在 CRM 里显示“客户数据同步中（约10~30秒）”，等 indexing 完成再提示“已更新”

  ### 4) 文档内容怎么组织（决定检索质量）

  把 30 个客户做成 30 个“可被切分的块”，例如用 ### 分隔（并在 Dify 的处理规则里用自定义分段 separator=###）：

  - 每个客户一段：### 客户ID: 73408 开头
  - 只保留“问得最多的字段”：星级、地区、行业、联系人、电话、开票状态、收货地址摘要等
  - 不要把 passport、内部鉴权 token、过多个人隐私字段写入知识库（安全/合规风险）

  ———

  必须提醒：共享知识库会“串数据”的风险（你现在方案的最大坑）
  如果你把所有销售的文档都放进同一个知识库，然后 Chatflow 里用 Knowledge Retrieval 默认检索：它会搜全库，可能检索到别的销售的客户信息。

  Dify 的 Knowledge Retrieval 节点支持“Metadata Filtering”来限制只检索满足条件的文档。(docs.dify.ai (https://docs.dify.ai/versions/3-7-x/en/user-guide/
  workflow/node/knowledge-retrieval?utm_source=openai))
  但前提是：你的每个销售文档都得能被打上 employee_code 之类的元数据，并且检索节点配置 employee_code == sys.user_id。

  问题在于：你现在“创建/更新文档”的公开 API 里并没有直接看到“写入文档自定义 metadata”的字段（至少在 Create/Update-by-text 的文档里不包含），所以你要么：

  - 额外找到/实现“文档 metadata 写入”的可用方式（UI 手工不现实）；或
  - 放弃“一个知识库多销售文档”的做法，回到我上面推荐的“会话变量缓存”（最稳）。

  ———

  我建议你选哪条

  - 你要“登录更新一次、后续不 HTTP、还要严防串数据” → 选 会话变量缓存（推荐方案）。
  - 你强依赖“知识库检索”能力（模糊搜客户/字段） → 可以做知识库 upsert，但你必须先确认你能否给文档自动打 metadata 并在检索节点做过滤，否则不建议上线。

  你回复我 3 个信息，我就能把方案落到“字段/节点配置级别”：

  1. 你 Dify 是哪个版本（以及 Knowledge Retrieval 节点里是否能配置 metadata 条件）
  2. 你现在知识库的 dataset_id 是哪个、你准备用一个 dataset 还是多个
  3. 你更希望客户问法是“按客户ID精确查”还是“按名称/电话模糊查”（决定 doc_text 的组织方式）



• 你这个需求（每个销售 30 个客户；销售进系统时同步一次；对话里模糊按客户名查；不想每轮都打 CRM HTTP）在 Dify 1.11.2 下可以做成一个很稳定的“CRM 负责同步，
  Chatflow 负责检索与回答”的闭环。

  下面按“我建议你怎么做（推荐方案）”→“如果你坚持在 Chatflow 里同步（不推荐但可行）”给你落地步骤。

  ———

  ## 一、我更推荐的方案：同步逻辑放 CRM(PHP) 后端，不放 Chatflow

  原因有三点：

  1. Dataset API Key 权限很大：Dify 官方明确说一个 Knowledge Base API token 可能对同账号下所有可见知识库都有操作权限，必须按“后端秘钥”级别管理，别放在工作 流
     里。(docs.dify.ai (https://docs.dify.ai/en/guides/knowledge-base/knowledge-and-documents-maintenance/maintain-dataset-via-api))
  2. 同步 KB 会触发 embedding/indexing 异步流程，不适合卡在首轮对话里（体验会抖）。更新接口会返回 batch 用于跟踪索引进度。(docs.dify.ai (https://
     docs.dify.ai/api-reference/documents/update-a-document-with-text?utm_source=openai))
  3. 你之前遇到的 192.168.* 内网不可达问题，本质上就是“Dify worker 不在内网”；把同步放到 CRM 后端就天然解决（CRM 后端能访问内网接口）。

  ———

  ## 二、数据设计：一个销售一篇文档（但检索隔离要靠 metadata）

  ### 1）文档命名（强烈建议用内置 metadata 过滤，省掉“写 metadata”的 API）

  Dify 有内置 metadata 字段（例如 document_name / uploader / upload_date / last_update_date / source），可以在知识库里启用。(docs.dify.ai (https://
  docs.dify.ai/en/guides/knowledge-base/metadata?utm_source=openai))

  因此你可以直接把文档名做成唯一：

  - crm_customers_{employee_code}
    例如：crm_customers_000424

  然后在 Knowledge Retrieval 节点里用 metadata filter：

  - document_name == crm_customers_{{sys.user_id}}

  这样你就不必再额外调用“设置文档 metadata”的接口。

  > 如果你现在已经建了自定义 metadata（比如 employee_code），也能用；但因为创建/更新文档 API 里并没有直接带 metadata 字段，通常还得再调一次 metadata 更新接口
  > （后面我给你备选）。

  ### 2）文档内容格式（决定“模糊按客户名查”的命中率）

  你这份 CRM 返回是：顶层 key 是 customer_id（如 73408），每个 customer 里有 base/contact/address/invoice 等。

  为了让“按客户名字模糊查”稳定，你的文档里每个客户块必须包含：

  - 客户名称（公司名/客户简称/别名）——你贴的数据里没看到“客户名称字段”，但你一定要把它补进去，否则“按名字查”没意义
  - customer_id
  - 常用检索字段：地区、行业、主联系人姓名、电话尾号（可选，注意脱敏）、星级等

  并且用固定分隔符把 30 个客户切成 30 段，推荐 ### 做分段符，方便 Dify 以自定义规则切 chunk：(docs.dify.ai (https://docs.dify.ai/api-reference/documents/
  create-a-document-from-a-file))

  示例（每个客户一段）：

  ### CUSTOMER 73408 | NAME=湖北斯万隆流体控技术有限公司 | ALIAS=斯万隆
  星级=二星级  类别=普通  行业=面板玻璃加工
  地区=中国/北京/北京市
  开户地址=...
  主联系人=李总(财务部/经理) 手机=136****2030
  收货地址摘要=湖北 鄂州市 鄂城区 ...

  ### CUSTOMER 363147 | NAME=华泰晨光药业 | ...
  ...

  ———

  ## 三、同步（Upsert）逻辑：存在就更新，不存在就创建，并且“没变化就不更新”

  你要的“判断逻辑”，推荐在 CRM(PHP) 做成标准 upsert + hash 去重。

  ### Step 0：CRM 侧建一张映射表（强烈建议）

  表：dify_sales_customer_doc

  - employee_code（唯一索引）
  - dataset_id
  - document_id
  - content_hash（md5/sha1 都行）
  - last_synced_at
  - last_batch（可选，用于查索引进度）

  ### Step 1：计算内容 hash（避免每次进入都更新）

  - 从 CRM 内网接口取到 30 个客户数据
  - 生成 doc_text
  - hash = sha1(doc_text)（或 md5）

  如果 hash 与表里 content_hash 一致：直接跳过更新（0 次 Dify API 调用）。

  ### Step 2：找 document_id（第一次需要）

  优先用你映射表里的 document_id；没有的话再去 Dify 查一次“按文档名 keyword 搜索”：

  - GET /v1/datasets/{dataset_id}/documents?keyword={docName}&page=1&limit=20(docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-the-document-
    list-of-a-knowledge-base?utm_source=openai))

  拿到后存表（以后就不需要每次查列表了）。

  ### Step 3：创建 or 更新文档

  - 创建（不存在）：
    POST /v1/datasets/{dataset_id}/document/create-by-text，body 至少包含 name + text + 可选 process_rule。(docs.dify.ai (https://docs.dify.ai/api-reference/
    documents/create-a-document-from-text))
  - 更新（存在）：
    POST /v1/datasets/{dataset_id}/documents/{document_id}/update-by-text，body 填 text（name 可不改），同样建议带 process_rule 固定分段。(docs.dify.ai
    (https://docs.dify.ai/api-reference/documents/update-a-document-with-text?utm_source=openai))

  两者都会返回 batch，你可以用它查索引进度：

  - GET /v1/datasets/{dataset_id}/documents/{batch}/indexing-status(docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-document-embedding-status-
    progress?utm_source=openai))

  > 注意：文档在 indexing 时可能会报“正在处理不能编辑”这类错误（例如 document_indexing），你需要在 CRM 侧做一个简单的并发保护：同一个 employee_code 同一时 间
  > 只允许一个同步任务。(docs.dify.ai (https://docs.dify.ai/en/guides/knowledge-base/knowledge-and-documents-maintenance/maintain-dataset-via-api?
  > utm_source=openai))

  ### Step 4：同步完成后更新映射表

  - 写回 document_id、content_hash、last_batch、last_synced_at

  ———

  ## 四、Chatflow 里怎么挂“客户查询”分支（只检索，不更新）

  你现在要做的是：对话时只从知识库检索 + LLM 组织答案，不打 CRM HTTP。

  1. Knowledge Retrieval 节点：

  - Dataset：选择你这个 dataset_id
  - TopK：建议 3~5（30 个客户不大）
  - Metadata filter：
      - 方案 A（推荐）：document_name == crm_customers_{{sys.user_id}}（前提：知识库启用 built-in metadata）(docs.dify.ai (https://docs.dify.ai/en/guides/
        knowledge-base/metadata?utm_source=openai))

  2. 后接一个 LLM 节点：

  - 规则：如果检索返回多个客户片段都像目标客户，让模型输出候选列表（客户名+customer_id+地区），让用户选；选中后你可以把 selected_customer_id 存会话变量，后续
    追问直接围绕它回答。

  ———

  ## 五、你坚持用“自定义 metadata(employee_code)”怎么办（备选）

  因为官方“创建/更新文档”API body 里没有 metadata 字段(docs.dify.ai (https://docs.dify.ai/api-reference/documents/create-a-document-from-text))，所以你需要额
  外再调一次“更新文档 metadata”的接口。

  社区里有人在 Dify 1.9.x 之后用过：

  - POST /v1/datasets/{dataset_id}/documents/metadata
  - body 类似：

    {
      "operation_data": [
        {
          "document_id": "xxx",
          "metadata_list": [
            {"id":"<metadata_field_id>", "name":"employee_code", "value":"000424"}
          ]
        }
      ]
    }

  这个接口目前不一定在公开 API reference 里完整列出，但在社区讨论里被验证过。(github.com (https://github.com/langgenius/dify/discussions/26688))

  所以：如果你不想依赖“未公开/可能变动”的接口，最稳就是用 built-in 的 document_name 来隔离。

  ———

  ## 六、我认为“更好的方案”（比知识库更稳、更快）

  你只有 30 个客户，其实完全可以不用 embedding：

  - 登录时 CRM 把 30 个客户做成一个 customers_index（只含 id+name+地区+行业+联系人）传给 Dify（inputs），Chatflow 用会话变量缓存；
  - 对话里用 Code/LLM 做字符串相似度匹配就能完成“模糊查”，命中后再按需去 CRM 拉详情（如果你能打公网 BFF）。

  这个方案优点：无索引延迟、无知识库串数据风险、性能更可控。

  ———

  如果你把下面 3 个值给我，我可以按你实际环境把 PHP 的 upsert 请求（curl/Guzzle）写成可直接用的函数，并给你一份 Chatflow 节点配置清单：

  1. 你的 Dify Base URL（比如 https://dify.amer.cn）和 dataset API 的 base path（你们实例里是 /v1/... 还是有前缀）
  2. 这个客户快照 dataset 的 dataset_id
  3. 客户“名称字段”在你 CRM JSON 里实际是哪一个 key（你贴的片段里没看到公司名字段）



## 优化

• 你现在这个“每个销售一篇文档（30个客户快照）→ 进入系统时创建/更新 → 对话里只做知识库检索”在 Dify 1.11.2 是能落地的，但我建议你把“创建/更新知识库文档” 的逻辑
  放在 CRM(PHP) 后端做，而不是在 Chatflow 里用 HTTP 节点去调用 Dify Dataset API。

  原因很现实：

  - 你之前已经遇到过 192.168.* 内网地址 Dify HTTP 节点访问不了的问题（Dify worker 不在你内网）；同步逻辑放 CRM 后端能直接访问内网 CRM 接口。
  - Dataset API Key 权限很大，放在工作流里风险高，放在后端最安全（也方便做 hash 去重、并发控制）。

  下面给你一套“最少坑、可稳定上线”的做法（含 upsert 判断逻辑 + Chatflow 挂载方式）。

  ———

  目标方案（推荐）

  - CRM 登录/进入 AI 面板时：CRM 后端取到该销售 30 个客户数据 → 生成 doc_text → 计算 hash → “存在则 update，不存在则 create” → 同步到 Dify dataset。
  - Chatflow：只做 Knowledge Retrieval，开启 Metadata Filtering，只检索当前销售那篇文档。

  这能做到：每次进入系统最多同步 1 次；对话中不再请求 CRM（只走 Dify 内部检索）。(docs.dify.ai (https://docs.dify.ai/en/use-dify/nodes/knowledge-retrieval?
  utm_source=openai))

  ———

  1) 文档如何唯一定位（最关键：别让检索串数据）
  你说 Dify 1.11.2 支持 metadata filter，那最省事的隔离方式是用“文档名”当过滤条件：

  - 文档名：crm_customers_{employee_code}
      - 例：crm_customers_000424

  Chatflow 的 Knowledge Retrieval 节点里：

  - 打开 Enable Metadata Filtering
  - 条件：document_name == crm_customers_{{sys.user_id}}
  - 这样每个销售只会检索到自己那篇文档，不会串到别人的客户快照。(docs.dify.ai (https://docs.dify.ai/en/use-dify/nodes/knowledge-retrieval?utm_source=openai))

  ———

  2) 文档内容怎么组织（决定检索“按客户ID/联系人/电话”能不能命中）
  你说当前 JSON 没有客户“名称字段”，只有客户 6 位数字 id，那“按客户名字模糊查”暂时做不到——检索只能匹配你文档里存在的字段（客户ID、联系人姓名、电话、地 址、行
  业等）。

  建议你把每个客户做成一个 chunk，强制用 ### 分隔（后面 process_rule 会用这个 separator 切段）：

  - 每个客户一段，第一行必须带 customer_id（方便模型回答时引用）：
      - ### CUSTOMER 73408
      - 然后把常用字段平铺出来（不要原样塞整个 JSON，太乱且冗余）

  示例结构（你自己拼接文本即可）：

  - ### CUSTOMER 73408
  - 星级=二星级 类别=普通 行业=面板玻璃加工 地区=中国/北京/北京市
  - 主联系人=李总 部门=财务部 职位=经理 手机=136****2030
  - 收货地址=湖北 鄂州市 鄂城区 ...
  - 开票=...

  ———

  3) Upsert 判断逻辑（存在则更新，不存在则创建 + hash 去重）
  你已经有固定 dataset（存所有销售的文档），所以对每个 employee_code 做如下流程：

  A. 生成 doc_text + 计算 hash

  - hash = sha1(doc_text)（或 md5）
  - 如果 hash 没变：直接跳过更新（避免每次进入都触发 embedding/indexing）。

  B. 找 document_id（强烈建议你自己存映射表）
  在 CRM 数据库建一张表，比如 dify_sales_doc_map：

  - employee_code（唯一）
  - dataset_id
  - document_id
  - content_hash
  - updated_at

  第一次没有 document_id 时，用 Dify “按 name keyword 查文档列表”接口找：

  - GET /datasets/{dataset_id}/documents?keyword=crm_customers_000424&page=1&limit=20(docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-the-
    document-list-of-a-knowledge-base?utm_source=openai))

  C. Create or Update

  - 创建文本文档：
      - POST /datasets/{dataset_id}/document/create-by-text
      - body 至少要 name + text；可选带 indexing_technique/doc_form/process_rule。(docs.dify.ai (https://docs.dify.ai/api-reference/documents/create-a-
        document-from-text?utm_source=openai))
  - 更新文本文档：
      - POST /datasets/{dataset_id}/documents/{document_id}/update-by-text
      - body 填 text（name 可不改），也可带 process_rule。(docs.dify.ai (https://docs.dify.ai/api-reference/documents/update-a-document-with-text?
        utm_source=openai))

  D. 可选：跟踪 indexing 状态（不强制）
  创建/更新会返回 batch，你可以用它查 embedding/indexing 进度：

  - GET /datasets/{dataset_id}/documents/{batch}/indexing-status(docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-document-embedding-status-
    progress?utm_source=openai))

  建议策略：

  - 不要阻塞销售进入系统（不然会卡）；你可以异步同步，然后 UI 提示“客户快照同步中”。

  ———

  4) 你在 CRM(PHP) 后端要调用的 Dify API（按你给的 base URL）
  你给的 dataset base：http://dify.amer.cn/v1

  - 查文档列表（用于首次定位 document_id）：

  curl -sS "http://dify.amer.cn/v1/datasets/{dataset_id}/documents?keyword=crm_customers_000424&page=1&limit=20" \
    -H "Authorization: Bearer {DATASET_API_KEY}"

  (docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-the-document-list-of-a-knowledge-base?utm_source=openai))

  - 创建文档：

  curl -sS "http://dify.amer.cn/v1/datasets/{dataset_id}/document/create-by-text" \
    -H "Authorization: Bearer {DATASET_API_KEY}" \
    -H "Content-Type: application/json" \
    -d '{
      "name": "crm_customers_000424",
      "text": "### CUSTOMER 73408\n...\n### CUSTOMER 363147\n...\n"
    }'

  (docs.dify.ai (https://docs.dify.ai/api-reference/documents/create-a-document-from-text?utm_source=openai))

  - 更新文档：

  curl -sS "http://dify.amer.cn/v1/datasets/{dataset_id}/documents/{document_id}/update-by-text" \
    -H "Authorization: Bearer {DATASET_API_KEY}" \
    -H "Content-Type: application/json" \
    -d '{
      "text": "### CUSTOMER 73408\n...\n### CUSTOMER 363147\n...\n"
    }'

  (docs.dify.ai (https://docs.dify.ai/api-reference/documents/update-a-document-with-text?utm_source=openai))

  - 查询索引进度（可选）：

  curl -sS "http://dify.amer.cn/v1/datasets/{dataset_id}/documents/{batch}/indexing-status" \
    -H "Authorization: Bearer {DATASET_API_KEY}"

  (docs.dify.ai (https://docs.dify.ai/api-reference/documents/get-document-embedding-status-progress?utm_source=openai))

  ———

  5) Chatflow 里你要怎么挂（你现在骨架后面只需要接一个 Retrieval 分支）
  在你的“客户查询”分支里：

  - Knowledge Retrieval 节点：
      - dataset：选你这个 dataset_id
      - query：用户输入（例如 userinput.query）
      - 开启 Metadata Filtering：
          - document_name == crm_customers_{{sys.user_id}}（或你实际传入的 sales id 变量）
      - top_k：先用 3 或 5
  - 后接 LLM 节点：
      - 提示词要求：只基于检索结果回答；如果检索片段里出现多个 customer_id，则输出候选让用户选择。

  （这一步只走 Dify 内部检索，不会再打 CRM HTTP。）(docs.dify.ai (https://docs.dify.ai/en/use-dify/nodes/knowledge-retrieval?utm_source=openai))

  ———

  6) 一个更“稳”的小优化（强烈建议你加）
  因为你现在用的是 http://：

  - 知识库里会存客户电话、地址等敏感信息时，强烈建议尽快切到 https://，否则链路明文风险很大。

  ———

  我还缺一个关键信息，才能把“过滤条件”和 doc_name 规范帮你定死

  - 你准备在 Dify 的 App API 调用时，把 user 传成什么？是 employee_code（如 000424）还是内部 sales_id（如 73408）？
      - 我建议：user = employee_code，这样 sys.user_id 就能直接拼 crm_customers_{sys.user_id} 做 metadata filter，最省事。

  你把 dataset_id 发我一下，我可以按你现在的字段给你：

  - 文档内容生成模板（把 customer_base/contact/address 哪些字段写进去最适合检索）
  - CRM(PHP) 的 upsert 伪代码/接口封装（含 hash 去重 + 首次查 doc id + 更新/创建）
