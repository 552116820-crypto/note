• 能，而且通常会更快、更稳定（也更省 token/费用）。前提是：你现在在 LLM 节点里做的事情主要是“从固定结构的 JSON 里取字段、拼接成文档内容/标题/元数据”，而不 是
  依赖大模型去“理解、总结、归纳、推断”。

  什么时候适合把 LLM 换成代码节点

  - 适合：字段提取、字段重命名/映射、数组展开、去重、格式化（Markdown/HTML）、清洗（去标签/去空白）、拼接文档、生成元数据、做简单规则（if/else）。
  - 不太适合（或需要保留 LLM）：需要摘要、主题归类、意图判断、相似合并、从自然语言备注里抽取“隐含信息”等。

  典型改造方式（推荐的工作流结构）

  1. HTTP 请求节点：调用 CRM API，拿到原始 JSON（尽量不要传给 LLM 原封不动的大 JSON）。
  2. 代码节点：把 CRM JSON 转成你知识库“创建/更新文档”节点需要的结构，例如：
      - doc_title：文档标题
      - doc_content：文档正文（建议 Markdown）
      - external_id：用于幂等更新的外部 ID（比如 CRM 的 customer_id / lead_id）
      - metadata：行业、地区、负责人、更新时间等（如果你下游支持）
  3. 下游继续用你现有的“创建/更新知识库文档”的方式（知识库节点或再次 HTTP 调 Dify 的相关接口）。
  4. 如果 CRM 返回的是列表（多条客户/线索）：用 迭代/循环节点 按条处理；代码节点输出一个数组，然后迭代逐条创建/更新。

  代码节点示例（Python，做“提取 + 拼文档”）

  > 你需要把入参名改成你代码节点里配置的输入变量名（例如 crm_response），字段路径也要按你的 CRM JSON 调整。

  import json

  def main(crm_response):
      data = json.loads(crm_response) if isinstance(crm_response, str) else crm_response
      r = data.get("data", data)  # 兼容 {data:{...}} 或直接 {...}
      title = f"客户-{r.get('name','')}".strip("-")
      content = "\n".join([
          f"# {title}",
          "",
          f"- CRM_ID: {r.get('id','')}",
          f"- 电话: {r.get('phone','')}",
          f"- 状态: {r.get('status','')}",
          f"- 最近跟进: {r.get('last_contact_at','')}",
      ])
      return {"doc_title": title, "doc_content": content, "external_id": str(r.get("id",""))}

  关于“更新”怎么做（避免每次都新建）

  - 关键是要有一个稳定的唯一键：external_id = CRM记录ID。
  - 你下游如果支持“按 external_id 更新”，最好直接用它。
  - 如果下游只能“按文档 ID 更新”，你需要在创建后保存映射关系（CRM_ID ↔ 文档ID），或者先查找是否存在同名/同 external_id 的文档再决定走更新还是创建。

  性能会提升的原因

  - 代码节点是本地规则计算：毫秒~秒级；LLM 节点通常是秒~几十秒（还可能受输入体积、模型队列、超时影响）。
  - 你还能在代码里把 JSON “裁剪”成最必要字段，后面即使仍要用 LLM（做摘要），输入也会小很多。

  我需要你确认 3 件事，才能给你一份“可直接粘贴用”的代码节点实现

  1. CRM API 返回 JSON 的一个实际样例（可脱敏），以及你需要抽取的字段清单/规则（例如：字段路径、数组要取第几个/全部）。
  2. 你知识库文档创建/更新节点现在吃什么输入（title/content/metadata/分段规则/是否支持批量）。
  3. CRM 一次返回是一条记录还是多条列表？如果是列表，你希望“一条 CRM 记录 = 一篇文档”，还是“合并成一篇文档”。

  把样例 JSON（脱敏）贴出来后，我可以按你的 Dify 工作流变量命名，给你完整的代码节点版本（含列表/迭代、幂等更新策略、字段缺失容错、内容清洗）。




• 把代码节点的输入变量命名为 body（映射上个节点的 body 字段），然后直接粘贴下面这段 Python；它会解析 body 里的 JSON，并把 data 下所有客户生成 Markdown（多个
  客户用 --- 分隔），输出变量名是 markdown。

  import json
  import re


  def _to_obj(x):
      if x is None:
          return {}
      if isinstance(x, (dict, list)):
          return x
      if isinstance(x, str):
          s = x.strip()
          if not s:
              return {}
          try:
              return json.loads(s)
          except json.JSONDecodeError:
              # 兜底：有些场景可能是被再次 JSON 包了一层字符串
              try:
                  return json.loads(json.loads(s))
              except Exception:
                  return {"raw": s}
      return {"raw": x}


  def _norm_text(value):
      if value is None:
          return "—"
      if isinstance(value, bool):
          return "是" if value else "否"
      if isinstance(value, (int, float)):
          return str(value)
      if isinstance(value, str):
          text = value.strip()
          if text == "" or text == "-":
              return "—"
          text = re.sub(r"\s+", " ", text)
          return text
      return str(value)


  def _join_nonempty(values, sep=" "):
      out = []
      for v in values:
          t = _norm_text(v)
          if t != "—":
              out.append(t)
      return sep.join(out) if out else "—"


  def _md_cell(value):
      t = _norm_text(value).replace("\n", " ")
      return t.replace("|", "\\|")


  def _dedupe_dict_list(items, key_fields):
      seen = set()
      out = []
      for it in items or []:
          if not isinstance(it, dict):
              continue
          k = tuple(_norm_text(it.get(f)) for f in key_fields)
          if k in seen:
              continue
          seen.add(k)
          out.append(it)
      return out


  def _contacts_table(contacts):
      contacts = _dedupe_dict_list(contacts, ["name", "phone", "tel", "email", "identity"])
      if not contacts:
          return "（无）"

      headers = ["姓名", "部门", "职位", "手机", "电话", "邮箱", "身份", "标签"]
      lines = [
          "|" + "|".join(headers) + "|",
          "|" + "|".join(["---"] * len(headers)) + "|",
      ]

      flag_map = [
          ("is_verify", "已验证"),
          ("is_finance", "财务"),
          ("is_pur", "采购"),
          ("is_user", "用户"),
          ("is_legal_person", "法人"),
          ("is_consignee", "收货人"),
      ]

      for c in contacts:
          tags = []
          for k, tag in flag_map:
              if c.get(k) in (1, True):
                  tags.append(tag)

          row = [
              _md_cell(c.get("name")),
              _md_cell(c.get("department")),
              _md_cell(c.get("position")),
              _md_cell(c.get("phone")),
              _md_cell(c.get("tel")),
              _md_cell(c.get("email")),
              _md_cell(c.get("identity")),
              _md_cell("、".join(tags) if tags else "—"),
          ]
          lines.append("|" + "|".join(row) + "|")

      return "\n".join(lines)


  def _addresses_table(addresses):
      addresses = _dedupe_dict_list(addresses, ["consignee", "phone", "tel", "region", "street"])
      if not addresses:
          return "（无）"

      headers = ["收货人", "手机", "电话", "地区", "详细地址"]
      lines = [
          "|" + "|".join(headers) + "|",
          "|" + "|".join(["---"] * len(headers)) + "|",
      ]

      for a in addresses:
          row = [
              _md_cell(a.get("consignee")),
              _md_cell(a.get("phone")),
              _md_cell(a.get("tel")),
              _md_cell(a.get("region")),
              _md_cell(a.get("street")),
          ]
          lines.append("|" + "|".join(row) + "|")

      return "\n".join(lines)


  def _build_customer_md(customer_id, customer_obj):
      base = (customer_obj.get("customer_base") or {}) if isinstance(customer_obj, dict) else {}
      invoice = (customer_obj.get("customer_invoice") or {}) if isinstance(customer_obj, dict) else {}
      contacts = customer_obj.get("customer_contact") or []
      addresses = customer_obj.get("customer_address") or []

      display_name = base.get("contact") or base.get("star_name") or str(customer_id)
      title = f"客户档案：{_norm_text(display_name)}（{customer_id}）"

      md = []
      md.append(f"# {title}")
      md.append("")

      md.append("## 基本信息")
      md.append(f"- 客户ID：{_norm_text(customer_id)}")
      md.append(f"- 客户星级：{_norm_text(base.get('star_name'))}")
      md.append(f"- 客户类别：{_norm_text(base.get('customer_category_text'))}")
      md.append(f"- 行业：{_norm_text(base.get('industry_name'))}")
      md.append(f"- 统一社会信用代码：{_norm_text(base.get('creditNo'))}")
      md.append(f"- 注册资本：{_norm_text(base.get('registCapi'))}")
      md.append(f"- 实缴资本：{_norm_text(base.get('actualCapi'))}")
      md.append(f"- 社保人数：{_norm_text(base.get('social_security_number'))}")
      md.append(f"- 主营销售：{_norm_text(base.get('main_sales'))}")
      md.append(f"- 角色：{_norm_text(base.get('role_name'))}")
      md.append(f"- 描述：{_norm_text(base.get('description'))}")

      location = _join_nonempty(
          [base.get("country"), base.get("province"), base.get("city"), base.get("district"), base.get("town")],
          sep=" ",
      )
      md.append(f"- 所在地：{location}")
      md.append(f"- 地址：{_norm_text(base.get('street'))}")
      md.append(f"- 注册地址：{_norm_text(base.get('zc_address'))}")

      md.append(f"- 联系人：{_norm_text(base.get('contact'))}")
      md.append(f"- 手机号：{_norm_text(base.get('phone'))}")
      md.append(f"- 电话：{_norm_text(base.get('tel'))}")
      md.append(f"- 传真：{_norm_text(base.get('fax'))}")
      md.append("")

      md.append("## 付款与开票规则")
      md.append(f"- 发票类型：{_norm_text(base.get('invoice_type'))}")
      md.append(f"- 付款方式：{_norm_text(base.get('payment_method_name'))}")
      md.append(f"- 付款周期：{_norm_text(base.get('payment_time'))}")
      md.append(f"- 对账周期：{_norm_text(base.get('bill_time'))}")
      md.append(f"- 开票起始：{_norm_text(base.get('bill_start'))}")
      md.append(f"- 开票截止：{_norm_text(base.get('bill_end'))}")
      md.append(f"- 开票章：{_norm_text(base.get('bill_invente_seal_text'))}")
      md.append(f"- 是否开票：{_norm_text(base.get('is_invoice_text'))}")
      md.append(f"- 打印模板：{_norm_text(base.get('print_template'))}")
      md.append("")

      md.append("## 开票信息")
      md.append(f"- 开户行：{_norm_text(invoice.get('account_bank'))}")
      md.append(f"- 账号：{_norm_text(invoice.get('account_num'))}")
      md.append(f"- 开户地址：{_norm_text(invoice.get('address'))}")
      md.append(f"- 开票电话：{_norm_text(invoice.get('invoice_tel'))}")
      md.append(f"- 寄送地址：{_norm_text(invoice.get('inv_send_address'))}")
      md.append(f"- 收件人：{_norm_text(invoice.get('inv_receiver'))}")
      md.append(f"- 收件人电话：{_norm_text(invoice.get('inv_receiver_tel'))}")
      md.append(f"- 收件人邮箱：{_norm_text(invoice.get('inv_receiver_email'))}")
      md.append(f"- 状态：{_norm_text(invoice.get('status_text'))}")
      md.append("")

      md.append("## 联系人")
      md.append(_contacts_table(contacts))
      md.append("")

      md.append("## 收货地址")
      md.append(_addresses_table(addresses))

      return "\n".join(md).strip() + "\n"


  def main(body):
      payload = _to_obj(body)

      data = payload.get("data") if isinstance(payload, dict) else None
      if not isinstance(data, dict):
          return {"markdown": "（未在 body JSON 中找到 data 字段，或 data 不是对象）\n"}

      def sort_key(k):
          s = str(k)
          return (0, int(s)) if s.isdigit() else (1, s)

      blocks = []
      for customer_id in sorted(data.keys(), key=sort_key):
          customer_obj = data.get(customer_id) or {}
          if not isinstance(customer_obj, dict):
              continue
          blocks.append(_build_customer_md(str(customer_id), customer_obj))

      markdown = "\n\n---\n\n".join([b.rstrip() for b in blocks]).strip() + "\n"
      return {"markdown": markdown}

  如果你后面要“一客户一文档”去更新知识库（而不是合并成一段 Markdown），我可以把返回改成 documents 数组（每个元素包含 external_id/title/markdown），配合 Dify
  的迭代节点就能实现真正的批量创建/更新。