• 可以，从 CRM API 取检索结果不仅“能做”，而且对你这种“每个客户结构几乎一致、每条又很长（~2000 tokens）”的数据形态来说，通常比把它们做成 Dify 知识库更靠谱。

  为什么 CRM API 往往更靠谱

  - 你的客户档案是强结构化数据（字段固定、模板一致），这类数据用向量召回天然会“都很像”，相似度分不出谁是谁，于是出现你说的：召回分数低、相似度高、容易串客
    户。
  - CRM 是“精确查询/权限过滤/最新数据”的系统；RAG（知识库）是“语义近似检索 + 生成”，不擅长当数据库用（尤其是要唯一命中某个客户时）。
  - 你还有“销售只能看自己的”硬隔离需求：CRM 用 owner_id=sales_id 这种条件过滤是确定性的；知识库要靠 metadata + 向量检索，稍不注意就会在召回阶段混入别人的
    chunk。

  ———

  推荐架构（混合，但以 CRM 为主）

  - CRM API = 事实数据源（客户基本信息、阶段、联系人、跟进记录等）
  - Dify Chatflow = 对话编排 + 总结生成（把 CRM 返回的结构化 JSON 变成可读回答/话术/下一步建议）
  - Dify 知识库（可选）= 只放非结构化资料（会议纪要全文、合同条款、产品资料、行业报告、销售 SOP），不要把“同模板客户档案”作为主要检索对象

  ———

  在 Dify Chatflow 里怎么做（核心流程）

  1. 后端鉴权注入销售身份

  - 企业微信登录后，你后端拿到可信的 userid，映射成 sales_id（建议统一前缀，如 wecom:{userid}）。
  - 调用 Dify chat-messages 时，把 user 固定设置为这个 sales_id（前端不允许传 sales_id）。

  2. Chatflow：先“定位客户”，再“取详情”

  - 节点 A（解析/抽取）：从用户问题里抽取企业名称/客户名（例：中国铝业股份有限公司）。
  - 节点 B（HTTP/工具调用 CRM：搜索）：调用 CRM 的“客户搜索”接口，强制加过滤条件：
      - owner_id = sys.user_id（或你传入的 sales_id）
      - name = 中国铝业股份有限公司（先精确匹配，找不到再模糊）
  - 节点 C（分支处理）：
─      - 0 条：直接回复“你名下没有该客户/请确认名称”
      - 多条：把候选列表返回让用户选（或再问一个字段：地区/统一社会信用代码/最近跟进时间）
      - 1 条：拿到唯一 customer_id
  - 节点 D（HTTP/工具调用 CRM：详情+跟进）：用 customer_id 拉取完整详情、最近 N 次跟进、下一次计划等。
  - 节点 E（LLM 总结/生成）：把 CRM JSON 作为“上下文事实”，生成你要的回答（概况、风险点、下一步动作、话术）。

  这样做的好处是：命中客户靠“精确 ID”，不靠“文本相似度”。

  ———

  你提到的“知识库召回分数低、相似度高”怎么理解

  - 向量检索会把“文本整体语义”编码成向量；当 30k 个客户文档都长得像（同样字段、同样标题、同样段落），向量就会非常接近。
  - 用户只给了一个实体（如企业名），在 2000 tokens 的大块文本里，这个实体对向量的贡献可能不够“突出”，导致 top_k 里混入别的相似客户块。

  ———

  如果你仍然想把客户档案放进知识库（不推荐当主检索，但可做兜底），至少要这么改

  - 把 sales_id 做 metadata 过滤：先过滤到“当前销售名下”的文档集合，再做向量检索（否则 1000 人的客户全集太相似）。
  - 把“企业名称/客户ID”放在每个 chunk 的最前面，并重复一次（不是为了好看，是为了让向量更“抓住”唯一标识）。
  - 去模板化：尽量减少重复字段名/固定话术，保留值、差异化信息（需求、痛点、跟进摘要、行业关键词），否则 embedding 全被模板噪声淹没。
  - 优先启用关键词/混合检索或 rerank（如果你这版 Dify 支持）：企业名这种强关键词匹配，纯语义向量不占优势。
  - 把“客户唯一 ID”变成检索主键：让用户/流程先选中客户，再问细节（两段式交互），避免“一句问话直接从 30 个相似块里猜”。

  ———

  你们 CRM API 只要回答我两个点，我就能把“Chatflow 节点设计 + 请求参数 + 分支逻辑”写成一套可直接照做的流程：

  - CRM 是否有“按企业名称搜索并返回 customer_id”的接口？（例如 GET /customers?name=... 或 POST /customers/search）
  - CRM 的客户归属字段叫什么（owner_id/sales_id/user_id）以及是否支持按它过滤？



• 可以，而且用你给的这 2 个 CRM 接口做“确定性检索（按员工编号/客户ID）”会比把客户档案做向量知识库更稳定——也正好避开你之前提到的“结构太一致导致相似度分不出
  谁”的问题。

  下面给你一套 Dify Chatflow 可直接照做的节点流程（节点设计 + 请求参数 + 分支逻辑），默认你要实现：

  - “销售只能看自己的客户”（强隔离）
  - 用户可以：列出我的客户、按名称定位客户、查看详情、查看开票/联系人/收货地址、支持多轮追问“这个客户…”

  ———

  0) 前置约定（非常关键，否则隔离做不牢）

  - 你的后端（企业微信登录后）调用 Dify Chat API 时，把 user 强制设置为该销售的 employee_code（例：000424）。
      - 这样 Chatflow 内 sys.user_id 就等于 employee_code，你在 CRM 调用里直接用它。
      - 前端不要传 employee_code，也不要让销售在对话里填（都不可信）。

  ———

  1) Chatflow 变量（在 Start 节点里配置）
  建议建 2 个“会话变量 / conversation variables”（名字按你 Dify 界面支持的方式建）：

  - current_customer_id（string，默认空）
  - current_customer_name（string，默认空）

  不建议把“客户列表”存会话变量（数组可能大/不稳定），客户列表每次需要时从 CRM 拉一次即可。

  ———

  2) 节点总览（直接照这个顺序拖节点）

  流程图（文字版）：

  3. Start
  4. LLM - 解析意图&提取客户标识（JSON 输出）
  5. Switch/If - 按 action 分支
      - A. list_customers 分支：列出名下客户
      - B. clear_customer 分支：清空当前客户
      - C. customer_* 分支：需要客户详情（summary/invoice/contacts/address…）
          1. 拉取名下客户列表（用于“名称->ID”匹配 + 防越权）
          2. 匹配 0/1/多 个客户
          3. 唯一命中后拉详情并回答
          4. 命中多个就让用户选 ID

  ———

  6) 节点 1：LLM 解析意图（强烈建议用 JSON 模式）

  节点名：ParseIntent

  输入：用户消息 sys.query

  让 LLM 输出固定 JSON（示例 schema）：

  {
    "action": "list_customers | clear_customer | customer_summary | customer_invoice | customer_contacts | customer_address | select_customer",
    "customer_id": "",
    "customer_name": "",
    "include_contact_details": false
  }

  提取规则建议：

  - 用户说“我的客户/客户列表/名下客户” => list_customers
  - 用户说“清空/重置客户” => clear_customer
  - 用户说“开票/发票/开户地址/银行账号” => customer_invoice
  - 用户说“联系人/对接人/电话/邮箱” => customer_contacts
  - 用户说“收货地址/地址/收货人” => customer_address
  - 其它默认 customer_summary
  - 用户若明确要求“电话/邮箱”，把 include_contact_details=true（否则默认不返回联系方式，只返回姓名/部门/职位）

  ———

  4) 节点 2：生成 passport（Code 节点，Python）

  你每次调用 CRM 前都需要 passport = md5(当日日期 + amer_fl_output)。

  节点名：GenPassport（Python Code）

  输出：passport

  代码（可直接用；用 UTC+8 规避服务器时区问题）：

  from datetime import datetime, timedelta
  import hashlib

  date_str = (datetime.utcnow() + timedelta(hours=8)).strftime("%Y-%m-%d")
  raw = f"{date_str}amer_fl_output"
  passport = hashlib.md5(raw.encode("utf-8")).hexdigest()

  return {"passport": passport, "date_str": date_str}

  说明：你给的例子 md5("2025-07-18amer_fl_output") 的确是 59b9d153ed42f856ea2821ca20d5f85b，算法一致。

  安全建议：

  - amer_fl_output 尽量只放在 Code 节点里，不要出现在任何 LLM Prompt 里（防被“提示词注入”套走）。

  ———

  5) 节点 3：HTTP 调用 CRM（两个接口）

  统一配置：

  - Method：POST
  - Header：Content-Type: application/json
  - URL 用线上测试：https://crm.amer.cn/v2/api/OutPut/...

  ———

  ### 5.1 查询销售名下客户（getSalesmanCustomer）

  节点名：CRM_ListCustomers

  URL：https://crm.amer.cn/v2/api/OutPut/getSalesmanCustomer

  Body（JSON）：

  {
    "passport": "{{GenPassport.passport}}",
    "data": [
      {
        "employee_code": "{{sys.user_id}}"
      }
    ]
  }

  返回你会用到：

  - state 是否为 200
  - data[]（每项包含 customer_id、customer_name）

  ———

  ### 5.2 查询客户详情（getCustomerDetail）

  节点名：CRM_GetCustomerDetail

  URL：https://crm.amer.cn/v2/api/OutPut/getCustomerDetail

  Body（JSON）：

  {
    "passport": "{{GenPassport.passport}}",
    "data": [
      {
        "customer_ids": "{{selected_customer_id}}"
      }
    ]
  }

  注意点：

  - CRM 返回的 data 是个对象，key 是字符串形式的客户ID，例如：data["73408"]

  ———

  6) “客户分支”的匹配与防越权逻辑（重点）

  因为用户可能在聊天里乱填 customer_id，你必须做到：

  - 任何客户详情查询前，都先调用 CRM_ListCustomers
  - 只允许查询“列表里存在的 customer_id”

  建议用一个 Code 节点来做匹配和选中：

  节点名：PickCustomer（Python Code）

  输入：

  - customers = CRM_ListCustomers.data
  - customer_id = ParseIntent.customer_id
  - customer_name = ParseIntent.customer_name
  - current_customer_id = 会话变量
  - current_customer_name = 会话变量

  输出：

  - match_type：by_id | by_name | by_current | none | multi
  - selected_customer_id
  - selected_customer_name
  - candidates（多匹配时给前端展示）

  匹配逻辑（照做即可）：

  1. 如果用户给了 customer_id：
      - 在 customers 里找同 ID
      - 找到 => by_id，选中
      - 找不到 => none（提示“你名下没有该客户ID”）
  2. 否则如果用户给了 customer_name：
      - 先做精确匹配（完全相等）
      - 再做包含匹配（A 包含 B 或 B 包含 A）
      - 命中 1 个 => 选中
      - 命中多个 => multi（让用户选 ID）
  3. 否则如果会话里已有 current_customer_id：
      - 仍要验证它在 customers 里（防会话污染）
      - 在 => by_current
      - 不在 => 清空并走 none
  4. 其它 => none

  ———

  7) 分支输出（If/Else 直接照这个写）

  ### 7.1 action = list_customers

  节点序列：

  - GenPassport -> CRM_ListCustomers -> Answer

  回答建议：

  - 如果 data 为空：回复“你名下暂无客户”
  - 否则只列前 N 条（比如 20 条），并提示：
      - “回复客户ID（如 73408）或客户全名以查看详情”
      - “也可以说：查看 73408 的开票信息 / 联系人 / 收货地址”

  ### 7.2 action = clear_customer

  节点序列：

  - SetVar current_customer_id=""
  - SetVar current_customer_name=""
  - Answer：提示已清空

  ### 7.3 action in customer_summary / customer_invoice / customer_contacts / customer_address / select_customer

  节点序列（核心链路）：

  - GenPassport
  - CRM_ListCustomers
  - PickCustomer
  - If PickCustomer.match_type == "none"：
      - Answer：提示未找到 + 引导“发送：我的客户”
  - Else If match_type == "multi"：
      - Answer：输出候选列表（customer_id + customer_name），让用户回复要选的 ID
  - Else（唯一命中）：
      1. SetVar current_customer_id = selected_customer_id
      2. SetVar current_customer_name = selected_customer_name
      3. CRM_GetCustomerDetail
      4. 按 action 取字段并回答（建议用 Code 节点先裁剪 JSON，再交给 LLM/直接格式化）

  ———

  8) “取字段并回答”的建议（减少 2000 tokens 压力 + 降幻觉）
  你这个 CRM 返回 JSON 很完整，建议不要把全量直接丢给 LLM。

  做法：

  - 加一个 Code 节点 BuildContext，根据 action 只取需要的字段：
      - customer_summary：customer_base 的关键字段（星级、类别、信用代码、注册资金、行业、付款方式、地址、描述…）
      - customer_invoice：只取 customer_invoice
      - customer_contacts：取 customer_contact；默认只输出 name/department/position/identity，当 include_contact_details=true 时再加 phone/email
      - customer_address：取 customer_address

  然后：

  - 结构化字段（开票、地址、联系人）我建议直接用模板输出，不走 LLM（最稳、不会编造）。
  - 需要“总结/建议/话术”的，再把裁剪后的 context 给 LLM。

  ———

  9) 重要的部署建议（更安全、更省心）
  如果你不想把 amer_fl_output 这个“验签固定串”放在 Dify（哪怕在 Code 节点），更推荐：

  - 由你自己的后端封装 2 个接口：/tool/listCustomers、/tool/getCustomerDetail
  - Dify Chatflow 只调用你后端（后端去算 passport、去调 CRM、做防越权）
  - 这样 CRM 验签规则永远不出你内网/后端，风险最低

  ———

  你确认一下这件事，我就能把“PickCustomer/BuildContext”的 Code 节点代码给你写成可直接粘贴的版本（含多匹配候选输出格式）：

  - sys.user_id 你们后端传的是 employee_code（如 000424） 还是企业微信 userid？如果是企业微信 userid，你们是否已有 userid -> employee_code 映射表/API？
