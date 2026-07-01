• 下面给你一套在 Dify Chatflow 里可直接用的“抽取客户信息 → 生成知识库文档正文（适合检索）”提示词（包含 System 提示词 和 User 提示词模板）。它能处理你这种 结
  构：外层是 HTTP 节点输出 JSON，里面 body 又是一段“字符串形式的 JSON”（而且带 \\uXXXX 转义）。

  ———

  System 提示词（粘到 LLM 节点的 System/角色提示词里）

  你是“客户资料抽取与知识库文档生成器”。

  目标：从输入的 HTTP 响应 JSON 中，抽取客户信息并生成适合向量检索的 Markdown 文档正文，用于写入知识库。你必须严格遵守以下规则：

  1. 数据来源与解析

  - 输入可能是一个对象，包含 status_code、headers、body 等字段；其中 body 可能是：
    a) 已经解析好的 JSON 对象；或
    b) 一个 JSON 字符串（可能带转义，如 \"、\\uXXXX）。
  - 你必须先解析出真正的业务数据对象（优先从 body 解析）。
  - 你必须将 \\uXXXX 还原为可读中文再输出。


  2. 客户数据结构

  - 业务 JSON 一般形如：{ state, message, data }
  - data 可能是一个对象，key 是客户ID（例如 "73408"），value 内包含：
      - customer_base
      - customer_invoice
      - customer_contact（数组）
      - customer_address（数组）
  - 你必须遍历 data 的所有客户ID，逐个输出。

  3. 输出格式（必须是 Markdown，分块友好）

  - 只输出 Markdown（不要输出解释、不要输出多余前后缀）。
  - 文档最开头必须有一个摘要块，包含：
      - 文档名：（来自 user 提供的 doc_name）
      - 员工编码：（来自 user 提供的 employee_code）
      - 生成时间：（用输入中的 date 头或你生成时的自然语言时间；拿不到就写“未知”）
      - 客户数量：
      - 关键词：（用逗号分隔，尽量覆盖：客户ID、统一社会信用代码、主销售、行业、省市、联系人姓名、电话后4位等；不要编造）
  - 每个客户必须用二级标题分隔：## 客户ID：{customer_id}
  - 每个客户下必须包含这些小节（缺失就写“未知/空”，不要猜）：
    A. ### 基本信息
    B. ### 业务与结算
    C. ### 开票信息
    D. ### 联系人
    E. ### 收货地址
    F. ### 检索友好摘要（用 2-4 句自然语言，把关键事实串起来，便于语义检索）

  4. 字段映射（尽量覆盖）
     在对应小节中尽量输出这些字段（存在就写值，不存在写“未知/空”）：

  - 基本信息（customer_base）：
      - 客户星级（star_name）
      - 客户类别（customer_category_text）
      - 统一社会信用代码（creditNo）
      - 注册资本（registCapi）
      - 实缴资本（actualCapi）
      - 所属行业（industry_name）
      - 主销售（main_sales）
      - 描述/备注（description）
      - 国家/省/市/区县/街道（country/province/city/district/town/street）
      - 注册地址/工商地址（zc_address）
      - 默认联系人（contact）
      - 手机/电话/传真（phone/tel/fax）
  - 业务与结算（customer_base）：
      - 付款方式（payment_method_name）
      - 付款周期/账期（payment_time）
      - 对账时间/开票时间相关（bill_time、bill_start、bill_end）
      - 是否有印章（bill_invente_seal_text）
      - 是否开票（is_invoice_text）
      - 打印模板（print_template）
  - 开票信息（customer_invoice）：
      - 开户行（account_bank）
      - 银行账号（account_num）
      - 开票地址/电话（address、invoice_tel）
      - 邮寄信息（inv_send_address、inv_receiver、inv_receiver_tel、inv_receiver_email）
      - 审核状态（status_text）
  - 联系人（customer_contact 数组）：
      - 每人一条，输出：姓名、部门、岗位、手机、电话、邮箱、身份(identity)
      - 同时根据标记补充角色标签（如：is_finance=1 → “财务”；is_pur=1 → “采购”；is_legal_person=1 → “法人”；is_consignee=1 → “收货人”；is_verify=1 → “已验
        证”）
      - 允许同名多条，但要尽量去重完全重复记录。
  - 收货地址（customer_address 数组）：
      - 每条输出：收货人(consignee)、手机(phone)、电话(tel)、地址(street)、区域(region)
      - 尽量去重完全相同的地址记录。

  5. 安全与真实性

  - 严禁编造公司名称、客户名称等输入里不存在的信息。
  - 严禁把原始整段 JSON 原样贴回（会污染检索）；只能抽取后的结构化文本。

  ———

  User 提示词模板（粘到 LLM 节点的 User/用户提示词里）

  把下面的 {{...}} 换成你在 Dify 里实际选择的变量（用变量选择器点选即可）：

  文档名：{{doc_name}}
  员工编码：{{employee_code}}

  以下是 HTTP 节点输出（其中 body 可能是 JSON 字符串，也可能是对象）：
  {{http_response}}

  请严格按 System 规则输出 Markdown 文档正文。

  ———

  变量怎么传（Dify 里最常见的接法）

  - doc_name：建议你在前面用“模板”节点生成 sales_customers_{{employee_code}}，然后传给 LLM。
  - http_response：建议直接传“HTTP 请求节点的整个输出对象”（包含 status_code/body/headers），而不是只传 body。这样模型能判断失败情况并取 date。

  ———

  参数建议（让抽取更稳）

  - temperature: 0 ～ 0.2（减少胡编）
  - 如果你的 data 里客户很多，输出可能超 token：可以在 User 里加一句“若客户数>50，只输出前50个并在摘要注明已截断”，避免失败。

  如果你愿意，我也可以按你当前 Chatflow 的节点名，帮你把 {{http_response}} / {{doc_name}} / {{employee_code}} 精确到“从哪个节点选哪个字段”，并给你一个“输 出直
  接是 {doc_name, doc_text} 的 JSON 版本”，方便你下一步用 HTTP API 写入知识库。



## 文档名称模板

• 按你现在的情况（employee_code 来自开始节点输入；模板节点输出字段叫 output），做法就是：

  1. 用「模板」节点把字符串拼出来
  2. 后面所有地方都引用「模板节点.output」即可（可选：再用“变量赋值”存成会话变量 doc_name）

  下面是可直接照做的步骤。

  1）开始节点：确认有输入变量

  - Start / 开始 节点里有一个输入变量：employee_code（用户每次对话输入）

  2）添加「模板」节点：生成 doc_name

  - 在 Start 后面加一个 模板 / Template 节点
  - 模板内容这样写（建议用“插入变量”点选，不要手打花括号）：
      - 先输入固定前缀：sales_customers_
      - 然后点「插入变量」
      - 选择：开始节点 -> employee_code
  - 最终模板看起来会像：sales_customers_{{employee_code}}（你界面可能显示成别的括号格式，但只要是用“插入变量”插进去的就对）

  这个模板节点的输出就是：模板节点.output，值例如：sales_customers_73408（取决于用户输入的 employee_code）。

  3）在后续节点里使用 doc_name（两种方式）

  A. 直接用（最省事）

  - 例如你在 LLM 节点 User Prompt 里写：
      - 文档名： + 插入变量选择 模板节点 -> output
  - 或者你在 HTTP 节点参数里要用它（比如 keyword、title），同样插入 模板节点.output

  B. 存成会话变量 doc_name（更干净，后面复用方便）

  - 新建一个 Conversation Variable：doc_name（String）
  - 在模板节点后面加一个 变量赋值 / Variable Assigner 节点：
      - 目标变量：doc_name
      - 值：插入变量选择 模板节点 -> output
  - 之后你在任何节点都用会话变量 doc_name（不用记“哪个模板节点”）

  小提醒（常见坑）

  - 如果用户输入的 employee_code 可能带空格/换行（比如复制粘贴），建议在模板前加一个“代码节点”做 trim() 再拼，不然文档名会多出空格导致检索/匹配失败。

  如果你把“模板节点”的节点名（例如叫 T1_生成文档名）告诉我，我可以把你后面 LLM 提示词里的变量引用写成你界面里能直接点选到的路径（例如：T1_生成文档名.output /
  doc_name）。
