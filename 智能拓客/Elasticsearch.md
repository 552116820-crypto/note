• 可以把 Elasticsearch/OpenSearch 定位成搜索与检索体验层，不是必须的推荐核心。

  在你这个架构里：

  MySQL：业务主库，客户、公海、领取、成交等事实数据
  PostgreSQL：推荐库，经纬度、PostGIS、pgvector、相似度、推荐分
  Elasticsearch/OpenSearch：搜索引擎，负责复杂搜索、模糊搜索、筛选聚合、地图搜索体验

  它能拿来做什么

  1. 企业搜索
     比如销售输入：

  利辛 纺织
  向阳中路
  面料加工
  某某科技

  Elasticsearch/OpenSearch 很适合做：

  - 企业名称模糊搜索
  - 地址模糊搜索
  - 经营范围搜索
  - 主营产品标签搜索
  - 拼写不准、简称、关键词命中

  PostgreSQL 也能做全文搜索，但如果你要做类似“CRM 搜索框”的体验，ES/OpenSearch 更顺手。

  2. 附近 + 关键词组合搜索
     比如：

  找我 30 公里内，经营范围包含“面料”，星级大于 2 的公海客户

  ES/OpenSearch 可以同时处理：

  - geo_point 附近查询
  - 文本关键词
  - 标签过滤
  - 星级过滤
  - 排序

  这适合做“销售主动搜索客户”，而不是系统自动推荐。

  3. 筛选聚合
     比如页面左侧筛选器：

  区域：
  - 亳州市 230 家
  - 阜阳市 180 家
  - 淮北市 96 家

  星级：
  - 3星 820 家
  - 2星 530 家

  行业标签：
  - 纺织 1200 家
  - 面料 340 家
  - 印染 90 家

  这种叫聚合统计，ES/OpenSearch 很擅长。
  适合做客户地图、客户池看板、销售筛选页。

  4. 地图列表/热力图
     如果你要在地图上展示客户点位、区域客户密度、框选地图区域搜索客户，ES/OpenSearch 也能做一部分地理检索和聚合。

  但如果是复杂地理边界、片区、多边形判断，PostGIS 仍然更强。

  5. 推荐结果检索缓存
     PostgreSQL 算出推荐结果后，可以把结果同步到 ES/OpenSearch：

  sales_id
  customer_id
  recommend_score
  distance_km
  match_score
  star
  industry_tags
  address
  reason

  前端列表页就可以从 ES/OpenSearch 查：

  - 分页
  - 搜索
  - 筛选
  - 高亮
  - 排序
  - 聚合

  这样 PostgreSQL 负责“算”，ES/OpenSearch 负责“查和展示”。

  6. AI 语义搜索
     如果你后面要让销售搜索：

  找附近做高端面料、可能有出口业务的企业

  OpenSearch/Elasticsearch 都支持向量检索能力，可以做语义搜索。
  不过如果你已经用 PostgreSQL + pgvector 做相似客户，第一版没必要再把向量也放 ES/OpenSearch，避免重复建设。

  我的建议

  第一版不要急着上 ES/OpenSearch。先这样落地：

  MySQL + PostgreSQL/PostGIS/pgvector

  等你出现这些需求，再引入 ES/OpenSearch：

  - 销售要按企业名称、地址、经营范围模糊搜索
  - 筛选条件很多，页面要实时统计数量
  - 地图上拖动/框选/关键词搜索客户
  - 推荐结果列表需要复杂分页、排序、高亮
  - 公海客户查询 QPS 很高，不想都压 PostgreSQL

  一句话
  PostgreSQL 适合做“推荐计算核心”；Elasticsearch/OpenSearch 适合做“搜索、筛选、聚合和前端检索体验”。
  不是必须三套都上，第一版可以先不上 ES/OpenSearch。

---



# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019dd693-4460-77f0-a0a2-560a21c15ad3`
- **范围**: Turn 23
- **导出时间**: 2026-04-30 11:15:43

---

### Turn 1  *(2026-04-30 03:11)*

**User:**

ES是什么，怎么用，我把PostgreSQL算出来的结果同步到ES，然后通过ES把推荐结果返回到前端，多了写入ES这一步对吗。ES的模糊搜索怎么实现的，我用PostgreSQL不可以实现吗，我是不是还要把客户信息上传到ES

**Assistant:**

ES 通常指 **Elasticsearch**，可以理解成“专门做搜索的数据库/搜索引擎”。OpenSearch 是它的开源分支，用法和定位很接近。

在你这个架构里，三者职责可以这样分：

```text
MySQL：业务事实库
- 客户主档
- 公海状态
- 领取、分配、成交
- 联系人、电话等主数据

PostgreSQL：推荐计算库
- 经纬度
- PostGIS 附近查询
- pgvector 相似度匹配
- 客户评分、推荐分计算

ES/OpenSearch：搜索展示库
- 模糊搜索
- 关键词检索
- 标签筛选
- 聚合统计
- 推荐结果列表的快速筛选和分页
```

是的，如果你要通过 ES 把推荐结果返回前端，**会多一步写入 ES**。但这一步通常是异步的，不应该在用户请求时临时写。

推荐链路可以是：

```text
MySQL 客户数据
    ↓ 同步
PostgreSQL 计算坐标、画像、相似度、推荐分
    ↓ 同步推荐结果/客户搜索文档
Elasticsearch
    ↓
前端查询列表、筛选、搜索、分页
```

你需要同步到 ES 的不是全部业务数据，而是“搜索和展示需要的字段”，比如：

```json
{
  "customer_id": 12345,
  "customer_name": "安徽某某纺织有限公司",
  "public_pool_status": "可领取",
  "province": "安徽省",
  "city": "亳州市",
  "district": "利辛县",
  "address": "安徽省亳州市利辛县城关镇向阳中路128号",
  "industry": "纺织业",
  "product_tags": ["纺织", "面料", "针织"],
  "star": "3星",
  "customer_score": 68.5,
  "match_score": 82.0,
  "recommend_score": 78.6,
  "location": {
    "lat": 33.143,
    "lon": 116.208
  },
  "recommend_reason": "主营标签与历史成交客户相似，距离较近，风险较低"
}
```

注意：**MySQL/PostgreSQL 仍然是准数据源，ES 只是搜索副本**。客户被领取、分配、冻结后，要同步更新 ES，否则前端可能看到已经不可领取的客户。

**ES 的模糊搜索怎么实现**

ES 的核心是倒排索引。简单说，它会把文本拆成词或片段，然后建立索引：

```text
安徽省亳州市利辛县城关镇向阳中路128号
```

可能被拆成：

```text
安徽省
亳州市
利辛县
城关镇
向阳中路
128号
```

用户搜：

```text
利辛 向阳中路
```

ES 就可以很快找到包含这些词的客户。

常见实现方式：

1. **match / multi_match**
   搜多个字段：

```json
{
  "query": {
    "multi_match": {
      "query": "利辛 面料",
      "fields": [
        "customer_name^3",
        "address^2",
        "product_tags^2",
        "business_scope"
      ]
    }
  }
}
```

`^3` 表示企业名称权重更高。

2. **fuzziness**
   适合英文、拼音、编码、公司名里的拼写误差。ES 官方 fuzzy query 是按编辑距离匹配，比如少字、错字、插入、调换字符。

```json
{
  "query": {
    "match": {
      "customer_name": {
        "query": "huaxin fangzhi",
        "fuzziness": "AUTO"
      }
    }
  }
}
```

3. **ngram / edge_ngram**
   更适合中文“包含式搜索”和输入联想。比如用户只输入：

```text
向阳
```

也能搜到：

```text
向阳中路128号
```

中文场景下，单纯 `fuzziness` 不一定好用，通常要配中文分词、ngram、同义词、拼音字段。

**PostgreSQL 不可以实现吗？**

可以。PostgreSQL 也能做不少搜索能力：

- `LIKE / ILIKE`
- 全文搜索 `to_tsvector`
- `pg_trgm` 三元组相似度搜索
- GIN/GiST 索引
- PostGIS 地理查询
- pgvector 向量相似度

比如 PostgreSQL 可以这样做相似文本搜索：

```sql
CREATE EXTENSION pg_trgm;

CREATE INDEX idx_customer_name_trgm
ON public_customer_profile
USING GIN (customer_name gin_trgm_ops);

SELECT *
FROM public_customer_profile
WHERE customer_name % '华鑫纺织'
ORDER BY similarity(customer_name, '华鑫纺织') DESC
LIMIT 20;
```

所以第一版你完全可以先不用 ES，只用：

```text
PostgreSQL + PostGIS + pgvector + pg_trgm
```

这对 100 万客户已经能做很多事。

**什么时候才值得上 ES**

如果你只是做：

```text
附近推荐 + 相似度匹配 + 简单筛选
```

先用 PostgreSQL 就可以。

如果你后面要做这些，再考虑 ES/OpenSearch：

- 企业名称、地址、经营范围的复杂模糊搜索
- 搜索结果高亮
- 输入联想
- 多字段相关性排序
- 标签/城市/星级的实时聚合统计
- 地图拖动搜索
- 搜索 QPS 很高
- 前端列表有很多筛选、排序、分页需求

**是否要把客户信息上传到 ES**

如果前端要直接从 ES 拿搜索结果，就需要把必要客户信息同步到 ES。不是上传全量敏感业务数据，而是同步搜索文档。

推荐两种模式：

**模式 A：ES 只返回 ID，后端再查 PostgreSQL/MySQL 补详情**
```text
优点：数据一致性好
缺点：多一次查询，列表性能差一点
```

**模式 B：ES 存列表展示字段，直接返回前端**
```text
优点：快，适合列表页
缺点：要处理同步延迟和数据冗余
```

你的场景我建议：

```text
第一阶段：不用 ES，先用 PostgreSQL 做推荐和简单搜索
第二阶段：搜索体验复杂后，把客户搜索文档和推荐结果同步到 ES
```

这样不会一开始就把系统搞重。

Sources: [Elasticsearch fuzzy query](https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-fuzzy-query), [PostgreSQL pg_trgm](https://www.postgresql.org/docs/17/pgtrgm.html)

---
