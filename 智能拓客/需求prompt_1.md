**需求解释**
核心技术栈python，fastapi

我要开发“智能拓客”功能，目的是为了给业务员提供高价值，高匹配度的公海客户。公海客户已经在数据库中，我需要计算公海客户的匹配度，即客户分数（有一套计算规则），按总分倒序，分页返回（每页20条）

最终我要交付三个接口给前端，第一个是AI推荐理由，第二个是客户分数与客户基本信息，第三个是行为时间轴信息。还要做日志，记录接口的调用情况

现在先完成第二个接口，客户分数与客户基本信息。

下面是两个数据库登录信息，把它们存进.env里

第一个数据库，用于查询业务数据：
host=47.114.77.122
port=3306
user=huangchuhao
password=
database=amer_crm_v2

第二个数据库，用于记录日志数据，查看接口调用次数和请求体信息，暂时不做
host=8.138.104.14
port=23356
user=root
password=
database=churn_db

**接口设计**

请求体
```json
{
  "employee_code": "000068",
  "lng": 117.860197,
  "lat": 39.966853,
  "radius_km": 60,      // 可选，默认 60，留口子方便以后调
  "page": 1,
  "page_size": 20
}
```

返回体（三段式：员工 / 分页 / 列表）
```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "employee_context": {                  // 员工级，整页共用一份
      "employee_code": "000068",
      "deal_customer_industries": ["金属冶炼热轧", "纸浆造纸行业"],
      "industry_top_customers": {          // ← 注意这里是对象 {}，不是数组 []
        "金属冶炼热轧": "唐山金马启新水泥有限公司",
        "纸浆造纸行业": "金东纸业（江苏）股份有限公司"
      }
    },
    "pagination": {                        // 分页级
      "page": 1,
      "page_size": 20,
      "total": 137,                        // 60km 内算分后的候选总数
      "total_pages": 7,
      "has_more": true
    },
    "list": [                              // 客户级，数组！一页 20 条
      {
        "customer_id": "198",
        "customer_name": "唐山市玉田金州实业有限公司",
        "total_score": 85,
        "score_detail": {                  // 销售要知道为什么这么分 ✅ 这个设计很好
          "industry_match": 40,
          "scale": 10,
          "tags": 10,
          "distance": 25
        },
        "amer_industry_id": 43,
        "amer_industry": "金属冶炼热轧",
        "distance_km": 9,
        "customer_lng": 117.860197,        // 改名：和请求里销售的 lng/lat 区分开
        "customer_lat": 39.966853,
        "address": "河北省唐山市玉田县郭家屯镇王乐庄村北",
        "actual_capi": "7500 万人民币",
        "tag_name": ["战略客户","2022-央企名单"],
        "star": 5,
        "social_insurance_count": 50,
        "regist_capi": "7500 万人民币"
      }
    ]
  }
}
```

**返回字段在数据库的位置**

实缴资金：amer_enterprise_baseinfo这张表， 有相应字段`actualCapi`'实缴资本'
客户标签：amer_enterprise_baseinfo这张表， 有相应字段`tags`'企业标签'
距离：crm_customer这张表，有相应的字段  `lng` '经度', `lat`'纬度'

### 员工级
"deal_customer_industries"：去重后的历史成交客户所在行业
"industry_top_customers"：同行业的历史成交数量最多的客户

历史成交客户，用出库明细表（crm_sale_issue_detail）来查，这个业务员有哪些客户有出库 ，出库就是成交 。
按crm_sale_issue_detail 的 employee_code 业务员编号，is_del=0 有效为条件，查出的 cust_ncccode 客户编码 （联表crm_customer 的 ncc_code 可得到full_name，amer_industry_id），
拿到amer_industry_id还需要查行业对应关系表amer_industry

### 客户级
主要看三张表，`crm_customer` 是**主表**，另外两张是**两个独立数据源的企业基础信息表**，各自通过 `customer_id` 外键挂到主表：
crm_customer.id == amer_enterprise_baseinfo.customer_id crm_customer.id == amer_qizhidao_baseinfo.customer_id

"customer_lng"对应crm_customer.lng
"customer_lat"对应crm_customer.lat
"tag_name"客户标签对应amer_tag_record.tag_name。在amer_tag_record这张表，也有customer_id
"tag_name": ["战略客户","2022-央企名单"],有的客户存在多个客户标签，取最近更新的两个标签
"actual_capi"实缴资金对应amer_enterprise_baseinfo.actualCapi


总分"total_score"的计算规则如下（满分100分）：
1.**行业匹配**：40分 （跟该账号历史客户所在行业完成匹配40分，不匹配0分）
2.**企业规模**：10分（实缴资金500万以上10分，实缴资金500万以下0分）
3.**客户标签**：10分（有标签10分，无标签0分）
4.**离定位距离**：40分（0-20KM）；1-40分（21-60KM，每增加1KM少一分）；0分（超过60KM）

计算逻辑如下：
1.**先用经纬度筛**：crm_customer.type = 0 为公海客户，只对公海客户进行筛选，SQL 里直接卡 60KM 范围，筛掉 90%+ 数据
Bounding Box 矩形预筛 + Haversine 精算
2.**剩下几千条算分**：在内存里算行业/规模/标签/距离
3.**排序分页**：按总分倒序，分页返回（比如每页 20）
因为分数不在数据库里，SQL 没法按总分排序取第 2 页。如果你每次翻页都重新"拉候选→算分→排序→切片"，那翻到第 7 页就把同样的活干了 7 遍。
正确做法：**第一次请求时把候选全量算分排序，结果（有序的 customer_id 列表 + 分数）缓存起来**（按 `employee_code + lng + lat + radius` 做 key，存 Redis，设个 5~10 分钟过期），翻页时直接从缓存切片


"customer_id"对应crm_customer.id
"customer_name"对应crm_customer.full_name
"amer_industry_id"对应crm_customer.amer_industry_id
"amer_industry"对应amer_industry.name，拿到amer_industry_id查行业对应关系表amer_industry，amer_industry_id == amer_industry.id
"distance_km"通过客户经纬度与业务员经纬度计算出来的
"address"对应amer_enterprise_baseinfo.address
"star"对应amer_customer_star.star，设置条件：字段 is_last =1，amer_customer_star.customer_id
"social_insurance_count"对应crm_customer.insurance_count
"regist_capi"对应amer_enterprise_baseinfo.registCapi
有些字段在对应的表中找不到数据的就标"-"
