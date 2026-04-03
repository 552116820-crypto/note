MD 模板（每企业一块）
# 企业名称：{customer_qizhidao_info.customer_name}
## 企业基本信息
企业类型：{customer_qizhidao_info.econKind}
注册地址：{customer_qizhidao_info.address}
注册资金：{customer_qizhidao_info.registCapi}
实缴资本：{customer_qizhidao_info.actualCapi}
统一社会信用代码：{customer_qizhidao_info.creditNo}
社保人数：{customer_qizhidao_info.social_security_number}
参保人数：{customer_qizhidao_info.insured_persons}
法人：{customer_qizhidao_info.operName}
邮箱：{customer_qizhidao_info.email}
官网：{customer_qizhidao_info.website}
经营范围：{customer_qizhidao_info.scope}
企业简介：{customer_qizhidao_info.company_profile}
## 企业业务信息
主营产品标签：{customer_qizhidao_info.main_product_tags}
行业赛道标签：{customer_qizhidao_info.industry_track_tags}
国行一级分类：{customer_qizhidao_info.industry_level_1}
国行二级分类：{customer_qizhidao_info.industry_level_2}
国行三级分类：{customer_qizhidao_info.industry_level_3}
## 企业联系人信息
姓名：{customer_contact[i].name}｜职位：{customer_contact[i].position}｜手机：{customer_contact[i].phone}
## 企业额外信息
荣誉资质：{customer_qizhidao_info.honors}
是否上市：{customer_qizhidao_info.is_listed}
裁判文书：{customer_qizhidao_info.judgment_documents}
失信被执行人：{customer_qizhidao_info.dishonest_person}
被执行人：{customer_qizhidao_info.executed_person}
限制高消费：{customer_qizhidao_info.consumption_restriction}
行政处罚：{customer_qizhidao_info.administrative_penalty}



额外规则：
  - 若 customer_contact 为空：整段 ## 企业联系人信息 + 列表不输出（你已确认）
  - 除“风险/计数类字段”（你可能希望保留 0）以外：值为空就整行不输出
  - 计数类字段：即使是 0 也输出（你之前选了“全部保留含0”）

  - 有联系人时，输出“联系人”标题 + 列表（每个数组元素一行）
  - 无联系人时，整段不输出
 
  例如（块内无空行版）：

  ## 企业联系人信息
  - 姓名：{name}｜职位：{position}｜手机：{phone}
  - 姓名：{name}｜职位：{position}｜手机：{phone}

  其中每一条 - ... 就对应 customer_contact[i]，顺序一致。



  1. 从输入里拿到的JSON是http请求后的返回body变量
  2. 遍历 data 里每个企业对象，生成 块内无空行 的 Markdown，并用 \n\n 把企业块拼接起来（保证你后续按 \n\n 分块时“一企业一块”）
