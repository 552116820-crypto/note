AI匹配度：在客户根据计算规则筛相应的客户，AI匹配度分数越高排的越前。 AI匹配度计算规则 行业匹配：40分 （跟该账号历史客户所在行业完成匹配40分，不匹配0分）

企业规模：10分（实缴资金500万以上10分，实缴资金500万以下0分，）

客户标签：10分（有标签10分，无标签0分）

离定位距离：

40分（0-20KM）

1-40分（21-60KM，每增加1KM少一分）  0分（超过60KM） 行为时间轴：展示跟公司历史相关的动作，AI一句话总结或是AI爬取的最新的动态。1、有拜访记录的展示拜访总结，拜访时间

2、有订单记录展示订单号，订单产品，下单时间，订单状态等

3、有服务记录的展示服务单号，服务单状态，服务需求描述



MySQL版本：5.6.47-log
host:47.114.77.122
port:3306
user:huangchuhao
password：$%asdF8976amer

现在我要开发“智能拓客”功能，目的是为了给业务员提供高价值，高匹配度的公海客户。公海客户已经在数据库中，我需要计算公海客户的匹配度，即客户分数（有一套计算规则），按总分倒序，分页返回（每页10条）

现在我要开发三个接口给前端，第一个是AI推荐理由，第二个是客户分数与客户基本信息，第三个是行为时间轴信息。

**客户分数与客户基本信息这个接口详情如下：**

crm_customer.id == amer_enterprise_baseinfo.customer_id crm_customer.id == amer_qizhidao_baseinfo.customer_id

接口请求
```json
{
"employee_code":"000068", // 员工编号
"lng":"00.000000", // 经度
"lat":"00.000000" // 纬度
}
```

amer_industry_id：26 => 纸浆造纸行业  # amer_industry表id

接口返回
```json
{
  "customer_id": "198",
  "customer_name": "唐山市玉田金州实业有限公司",
  "total_score": 85,
  "score_detail": {                 // 销售要知道为什么这么分
    "industry_match": 40,
    "scale": 10,
    "tags": 10,
    "distance": 25
  },
  "amer_industry_id":43,   
  "amer_industry":"金属冶炼热轧", 
  "distance_km": 9, 
  "lng":"117.860197",
  "lat":"39.966853",
  "address":"河北省唐山市玉田县郭家屯镇王乐庄村北",
  "actualCapi":"7500 万人民币",
  "tag_name":"战略客户", // 战略部提供的,有多条取最新，amer_tag_record
  "star":5, // amer_customer_star
  "social_insurance_count": 50,
  "registCapi":"7500 万人民币"
}
```



客户分数计算规则如下（满分100分）：
1.**行业匹配**：40分 （跟该账号历史客户所在行业完成匹配40分，不匹配0分）
2.**企业规模**：10分（实缴资金500万以上10分，实缴资金500万以下0分）
3.**客户标签**：10分（有标签10分，无标签0分）
4.**离定位距离**：40分（0-20KM）；1-40分（21-60KM，每增加1KM少一分）；0分（超过60KM）
计算逻辑如下：
1.**先用经纬度筛**：SQL 里直接卡 60KM 范围（用空间索引），筛掉 90%+ 数据
2.**剩下几千条算分**：在内存里算行业/规模/标签/距离
3.**排序分页**：按总分倒序，分页返回（比如每页 20）

这里计算规则的需要的数据，MySQL数据库都有

行业：crm_salesman_customer_history这张表存了客户与业务员历史跟进关系，可以通过员工编号`employee_code`来筛选
实缴资金：amer_enterprise_baseinfo这张表， 有相应字段`actualCapi`'实缴资本'
客户标签：amer_enterprise_baseinfo这张表， 有相应字段`tags`'企业标签'
距离：crm_customer这张表，有相应的字段  `lng` '经度', `lat`'纬度'








