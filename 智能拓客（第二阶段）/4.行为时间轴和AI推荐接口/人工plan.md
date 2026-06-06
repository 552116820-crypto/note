
行为时间轴：展示跟公司历史相关的动作，AI一句话总结或是AI爬取的最新的动态。
1、有拜访记录的展示拜访总结，拜访时间
2、有订单记录展示订单号，订单产品，下单时间，订单状态等
3、有服务记录的展示服务单号，服务单状态，服务需求描述，服务时间

现在紧接着，就做AI推荐理由，行为时间轴信息，这两个信息我打算放在一个接口，因为AI推荐理由，它是基于行为时间轴，客户分数等信息，喂上下文给deepseek v4大模型，调用大模型API给出推荐理由。
行为时间轴，也需要根据拜访记录，订单记录，服务记录，喂给AI，调用API整理出最可靠的行为时间轴信息，整理规则可以讨论或给出你的方案。

有一个点需要注意，大模型的每次回复格式都不是固定，但接口返回一定是json，要给大模型结果做规范，并校验。

拜访记录，订单记录，服务记录这三方面信息你需要调用crm接口，获取相应的信息。

以上是我的需求和规划，请你审查，并做好优化和补充

crm接口汇总：
全是post请求：
请求体中“passport”字段，通过md5计算得到，代码如下：
import hashlib
import datetime

def main(employee_code: str) -> dict:
    current_date = datetime.datetime.now().strftime("%Y-%m-%d")
    fixed_str = "amer_fl_output"
    raw_string = current_date + fixed_str
    signature = hashlib.md5(raw_string.encode('utf-8')).hexdigest()
    return {
        "passport": signature
    }

接口url与请求体示例如下：customer_ids支持多个客户id查询，当customer_ids不为空时，sales_ids和employee_code可为空，需不需要填sales_ids和employee_code可以探讨一下

订单记录接口，只看一年内的订单
https://crm.amer.cn/v2/api/OutPut/getSalesmanSaleOrder
{
  "start_date": "2025-06-06",
  "end_date": "2026-06-06",
  "sales_ids": "",
  "customer_ids": "3599",
  "passport":"cc5d55757f805503f9485d8bacc74a62"
}


拜访记录接口
https://crm.amer.cn/v2/api/OutPut/getSalesmanCustomerVisit
{
"employee_code": "",
"customer_ids":"3599,73408",
"passport": "cc5d55757f805503f9485d8bacc74a62"
}

服务记录接口
https://crm.amer.cn/v2/api/OutPut/getSalesmanCustomerServiceApply
{
"employee_code": "",
"customer_ids":"3599",
"passport": "cc5d55757f805503f9485d8bacc74a62"
}

