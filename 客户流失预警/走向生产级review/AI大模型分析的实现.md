# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019d9a23-1aeb-7bc3-95b8-1828cce529fb`
- **范围**: Turn 1-9
- **导出时间**: 2026-04-17 15:37:43

---

### Turn 1  *(2026-04-17 06:35)*

**User:**

现在我需要用AI大模型分析客户流失的原因，给销售提供挽回策略和沟通技巧，主要基于客户拜访记录的内容，以及客户流失预警的简述，你觉得这可行吗

**Assistant:**

可行，而且和你现在的流失预警系统很契合。但要把它定位成“基于证据的销售辅助分析”，不要定位成“AI 准确判断真实流失原因”。真实原因往往需要销售验证，AI 更适合做线索整理、风险归因、挽回话术和下一步行动建议。

我建议这样设计：

**核心输入**

- 客户流失预警简述：例如风险等级、触发规则、异常指标、最近变化。
- 客户拜访记录：时间、拜访人、客户反馈、异议、竞品信息、预算、决策人态度、后续计划。
- 可选增强信息：客户基础信息、历史成交、合同到期时间、最近订单/使用变化、销售阶段。

**AI 输出不要太自由，最好结构化**

可以让大模型返回类似这种 JSON 或固定 schema：

```json
{
  "likely_churn_reasons": [
    {
      "reason": "客户近期对价格敏感度上升",
      "evidence": ["2026-03-12 拜访记录中提到预算缩减", "2026-03-28 客户询问竞品价格"],
      "confidence": "medium"
    }
  ],
  "customer_sentiment": "谨慎观望",
  "key_risks": ["竞品介入", "预算收缩", "决策人参与不足"],
  "recommended_recovery_strategy": [
    "先确认预算变化是否已经影响采购计划",
    "安排一次包含决策人的价值复盘会议",
    "准备 ROI 或成功案例材料"
  ],
  "talking_points": [
    "理解贵司今年预算更谨慎，我们可以先一起确认哪些投入必须保留。",
    "如果单看价格，我们未必最低，但可以把服务响应、交付风险和长期成本放在一起评估。"
  ],
  "questions_for_sales": [
    "客户是否已经拿到竞品报价？",
    "当前反对意见来自使用部门还是采购部门？"
  ],
  "missing_information": [
    "最近一次拜访没有明确记录客户是否已有替代方案"
  ]
}
```

**关键原则**

1. **必须要求 AI 给出证据来源**  
   每个判断都要引用拜访记录或预警简述中的具体内容。没有证据就标记为“推测”或“不足”。

2. **不要只让 AI 输出结论**  
   更有价值的是输出“原因假设 + 证据 + 销售验证问题 + 挽回动作”。

3. **先不用复杂 RAG**  
   如果每个客户拜访记录数量不大，直接取最近 90/180 天记录，按时间排序喂给模型即可。等记录很多时，再做摘要、分块检索或长期客户画像。

4. **异步生成更合适**  
   这个功能适合在预警生成后由 Celery 任务异步生成分析结果，销售打开客户详情时直接查看缓存结果，避免接口慢。

5. **需要人工确认闭环**  
   销售可以标记“有用/无用”、选择真实流失原因、记录最终挽回结果。后续这些反馈可以用于优化提示词、规则和评估集。

从你这个项目的结构看，比较自然的落点是：

- `app/churn/engine.py`：保留现有预警规则，不建议让大模型替代风险判断。
- `app/churn/service.py`：在预警详情中编排 AI 分析能力。
- 新增类似 `app/churn/ai_analysis.py` 或 `app/churn/llm_service.py`：封装大模型调用、提示词和输出校验。
- `app/churn/tasks.py`：预警生成后异步生成“流失原因与挽回建议”。
- 数据表新增 AI 分析结果缓存：保存输入摘要、模型输出、模型版本、生成时间、销售反馈。

我会建议先做一个 MVP：针对已触发预警的客户，读取最近 180 天拜访记录 + 当前预警简述，生成一份结构化“流失原因分析和挽回建议”。先让销售团队试用 20 到 50 个真实客户案例，看建议是否具体、是否能帮他们准备沟通，再决定是否扩大到自动化、批量生成和反馈学习。

---

### Turn 2  *(2026-04-17 06:37)*

**User:**

拜访记录只会返回客户最近三次的拜访记录

**Assistant:**

可以，最近三次拜访记录仍然可用，但功能定位要收窄成：

**基于最近沟通内容的流失近因分析和销售挽回建议**，而不是完整的客户历史流失归因。

最近三次拜访记录的优点是信息新、贴近销售下一步沟通，尤其适合生成：

- 客户当前主要异议
- 最近情绪和态度变化
- 是否出现竞品、预算、交付、服务、决策人变化等信号
- 下一次沟通策略
- 销售可以怎么问、怎么回应、怎么推进

但它的限制也很明显：

- 如果流失原因是长期积累的，三次记录可能看不出来
- 如果销售记录写得很简略，AI 只能给低置信度建议
- 不能让 AI 编造历史原因，必须允许输出“信息不足”
- 更适合分析“最近为什么危险”，不适合判断“客户过去完整流失路径”

我建议输入可以这样组织：

```json
{
  "customer_id": "C001",
  "warning_summary": "客户近30天下单频次下降60%，最近一次订单金额低于历史均值70%，触发高风险预警。",
  "recent_visit_records": [
    {
      "visit_time": "2026-04-10",
      "content": "客户反馈今年预算收紧，正在比较其他供应商报价。"
    },
    {
      "visit_time": "2026-03-28",
      "content": "客户认为售后响应速度不如预期，希望后续有明确联系人。"
    },
    {
      "visit_time": "2026-03-15",
      "content": "客户提到新负责人上任，采购流程可能调整。"
    }
  ]
}
```

输出建议控制成结构化结果：

```json
{
  "analysis_scope": "仅基于最近三次拜访记录和当前预警简述",
  "likely_reasons": [
    {
      "reason": "预算收紧并开始比较竞品报价",
      "evidence": ["2026-04-10 拜访记录提到预算收紧和比较其他供应商报价"],
      "confidence": "high"
    },
    {
      "reason": "售后响应体验影响客户信任",
      "evidence": ["2026-03-28 拜访记录提到售后响应速度不如预期"],
      "confidence": "medium"
    }
  ],
  "recovery_strategy": [
    "优先确认客户是否已经收到竞品正式报价",
    "准备价格以外的价值对比材料，例如响应时效、交付稳定性、历史合作成本",
    "安排售后负责人参与下一次沟通，明确后续服务联系人和响应机制"
  ],
  "talking_points": [
    "我们理解今年预算压力比较大，这次可以先帮您把必须保留的服务价值和可调整部分拆开看。",
    "关于响应速度的问题，我们希望这次直接明确后续负责人和响应时间，避免再让您反复跟进。"
  ],
  "questions_for_sales": [
    "竞品报价是否已经进入正式评估？",
    "新负责人对供应商选择最关注价格、稳定性还是服务响应？",
    "客户当前最不满意的是价格、服务，还是内部采购流程变化？"
  ],
  "missing_information": [
    "缺少客户历史长期不满点",
    "缺少竞品具体名称和报价信息",
    "缺少新决策人的采购偏好"
  ]
}
```

我会建议系统里明确加一条规则：**AI 的每个原因必须引用最近三次拜访记录或预警简述作为证据；没有证据只能放到待确认问题里，不能当成结论。**

如果你们将来想弥补“只有最近三次”的限制，可以加一个轻量方案：每次生成分析后，沉淀一份“客户风险画像摘要”。以后即使拜访接口只返回最近三次，也可以把历史摘要一起喂给模型：

- 最近三次拜访记录：反映最新情况
- 客户风险画像摘要：保留长期问题
- 当前流失预警简述：解释为什么系统判断有风险

这样效果会比单纯依赖三条拜访记录稳定很多。MVP 阶段只用最近三次也可以先做，重点看销售是否觉得建议具体、可用、不会胡说。

---

### Turn 3  *(2026-04-17 06:39)*

**User:**

"sales_id": "000455",
  "customer_ids":"115849",这是一组销售和流失客户，你帮忙看看

**Assistant:**

我先看一下项目里现有的客户流失查询和拜访记录接口怎么接，确认这组 `sales_id/customer_ids` 能直接通过现有代码或测试数据查到什么。

代码里已经有 CRM API 仓储和预警查询入口。我继续看配置与接口形状，判断这组 ID 是该跑一次检测，还是只能查现有预警记录。

现有代码没有“按一组客户 ID 直接分析拜访记录”的入口，只在检测时用拜访接口校验客户归属。我要直接调用 CRM 拜访/订单接口看原始返回，再给你判断这组客户能支撑什么分析。

环境里没有配置本地数据库或 API key，我会直接按项目默认的 CRM 接口和当前日期生成的 `passport` 请求。先只取这组销售/客户相关数据，不做任何文件修改。

拜访接口实际要求字段是 `employee_code`，`sales_id` 会返回“员工编号不能为空”；`customer_ids` 参数看起来会被忽略，接口仍返回该销售可见的客户集合。我现在从返回里只抽取客户 `115849`。

客户 `115849` 有三条拜访记录，但时间跨度很大，最新一条是 2026-03-17，另外两条在 2021 年。为了补上“流失预警简述”，我再查这位销售近一年订单里该客户的订单情况，看是否能解释当前预警。

按现有引擎用近一年订单跑了一遍：这组客户会触发 1 条整体频率高风险和 2 条关键产品高风险。为避免“只看一年”导致误判，我再扩展历史窗口核对一下该客户更早的订单节奏。

我看了这组数据：`sales_id=000455`，`customer_id=115849`。

客户是 **苏州鼎嘉工贸有限公司**，销售是 **黄阳霞**。按项目当前默认近一年窗口和 2026-04-17 业务日期看，这个客户确实是比较典型的高风险流失客户。

**关键事实**

最近三次拜访记录：

- `2026-03-17`：上门拜访杨总和老板，**谈涨价问题**
- `2021-12-07`：记录提到 2 年没去拜访，维护客情关系
- `2021-10-10`：微信联系杨总，问候生产情况并补合同

近一年订单：

- `2025-06-07`：10083 元，导轨油 T68 + 液压油
- `2025-07-28`：7144 元，导轨油 T68
- `2025-08-09`：252 元，洗手液
- `2025-08-11`：252 元，洗手液
- `2025-09-09`：13022 元，导轨油 T68 + 液压油
- `2025-10-09`：5878 元，液压油
- `2026-01-09`：252 元，洗手液

系统预警结果：

- 整体下单频率：高风险  
  预测周期约 `25.65` 天，上次正向下单是 `2026-01-09`，截至 `2026-04-17` 已经 `98` 天未下单，预计应在 `2026-02-04` 左右下单，已延后 `72` 天。

- 关键产品 1：安美优质导轨油 T68，高风险  
  近一年销售占比约 `58.1%`，上次采购是 `2025-09-09`，已经 `220` 天未采购。

- 关键产品 2：安美优质抗磨液压油 QH-HM68，高风险  
  近一年销售占比约 `39.8%`，上次采购是 `2025-10-09`，已经 `190` 天未采购。

**我的判断**

最可能的流失原因是：**涨价引发客户重新评估采购，主力油品采购已经中断，只剩低金额辅品订单维持关系。**

证据比较清楚：最新拜访唯一有效信息就是“谈涨价问题”；订单上看，客户不是完全没交易，而是主力产品停了，`2026-01-09` 只买了 252 元洗手液。这种情况比“客户彻底停摆”更像是：核心油品采购被延后、转向竞品、压库存，或者价格谈不拢。

**建议销售这样挽回**

第一步不要直接谈优惠，先确认状态：

- “杨总，上次我们聊到涨价问题，我想先确认一下，现在贵司是暂缓采购，还是已经在对比其他供应商？”
- “导轨油和液压油从去年 9、10 月之后就没有继续补货了，现在现场是库存还够，还是已经换了其他产品在用？”
- “这次价格调整主要卡在预算，还是老板对性价比有顾虑？”

第二步把沟通重点放在主力产品上，不要泛泛维护客情：

- 导轨油 T68 是第一优先级，因为近一年占比最高，且停采时间最长。
- 液压油 QH-HM68 是第二优先级，需要确认是否被替代。
- 洗手液订单金额太小，不能证明客户关系稳定。

第三步准备一个“涨价解释 + 成本对比”方案：

- 说明涨价原因，但不要只解释成本上涨。
- 对比停机、换油、品质不稳定、售后响应的隐性成本。
- 如果客户正在比价，建议拿“总使用成本”去比，而不是单价硬拼。

可用话术：

> 杨总，上次我们聊到涨价，我理解这个点会影响采购判断。我们这次不急着让您下单，想先把现场现在用油情况确认清楚：T68 和液压油现在是库存还够，还是已经在试其他品牌？如果是在比价格，我们可以一起把单价、使用周期、设备风险和售后响应放在一起算一遍，避免只看单价导致后面使用成本更高。

**需要销售补充确认的问题**

- 客户是否已经换用竞品？
- 涨价后客户能接受的价格区间是多少？
- 当前反对意见来自杨总，还是老板最终拍板？
- 主力产品停采是因为库存、生产下降、价格、质量，还是供应商替换？
- 能否安排一次现场用油检查或库存盘点？

结论：这组客户非常适合作为 AI 分析样例。拜访记录虽然只有一条近期有效内容，但和订单异常能形成闭环：**最新拜访提到涨价，订单显示主力产品停采，系统频率和产品预警都已高风险**。

---

### Turn 4  *(2026-04-17 06:45)*

**User:**

是的，分析的很好，我希望按照这样的方式进行AI分析，你看一下能否实现

**Assistant:**

可以实现，而且现有项目基础比较适合做这个功能。

我建议实现成一个新的“AI 流失分析”能力，不改动现有预警引擎本身。现有 `frequency/product/volume` 规则继续负责判断“是否异常”，AI 负责解释“可能为什么异常、销售怎么挽回”。

**建议实现方式**

新增一个接口，例如：

```http
POST /api/churn-ai-analysis
```

请求：

```json
{
  "sales_id": "000455",
  "customer_id": 115849
}
```

返回：

```json
{
  "customer_id": 115849,
  "customer_name": "苏州鼎嘉工贸有限公司",
  "sales_id": "000455",
  "analysis_scope": "仅基于最近三次拜访记录、当前预警简述和近期订单异常",
  "likely_reasons": [
    {
      "reason": "涨价导致客户重新评估主力油品采购",
      "evidence": [
        "2026-03-17 拜访记录提到谈涨价问题",
        "导轨油 T68 自 2025-09-09 后未再采购",
        "液压油 QH-HM68 自 2025-10-09 后未再采购"
      ],
      "confidence": "high"
    }
  ],
  "recovery_strategy": [
    "优先确认客户是否因涨价暂停采购或转向竞品",
    "重点挽回导轨油 T68 和液压油 QH-HM68 两个主力产品",
    "准备涨价说明和总使用成本对比，而不是直接进入降价谈判"
  ],
  "talking_points": [
    "杨总，上次我们聊到涨价，我想先确认现在是暂缓采购，还是已经在对比其他供应商？",
    "如果是在比价格，我们可以一起把单价、使用周期、设备风险和售后响应放在一起算。"
  ],
  "questions_for_sales": [
    "客户是否已经换用竞品？",
    "涨价后客户能接受的价格区间是多少？",
    "主力产品停采是因为价格、库存、生产下降，还是供应商替换？"
  ],
  "missing_information": [
    "缺少客户是否已收到竞品报价的信息",
    "缺少客户当前库存和现场实际用油情况"
  ]
}
```

**需要改造的点**

1. **拜访记录获取**

当前代码里 `app/shared/crm_api.py` 已经会调用拜访接口：

```python
CRM_VISIT_API_URL = getSalesmanCustomerVisit
```

但它现在只用于“校验客户是否还属于该销售”，没有把拜访内容暴露给业务层。

需要新增一个方法，例如：

```python
get_recent_visit_records(
    sales_id: str,
    customer_id: int,
    limit: int = 3,
) -> list[VisitRecord]
```

注意一点：我刚才实际验证过，拜访接口用的是：

```json
{
  "employee_code": "000455",
  "passport": "..."
}
```

而不是 `sales_id`。`customer_ids` 参数看起来会被接口忽略，所以实现时要先按 `employee_code` 拉取该销售的客户拜访集合，再在本地用 `customer_id` 过滤出 `115849`。

2. **预警简述生成**

现有预警记录已经有 `reason`、`sales_message`、`feature_summary`，可以直接作为 AI 输入。

例如这类内容：

- 整体频率：预测周期、上次下单、延迟天数、风险等级
- 产品异常：关键产品、上次采购日期、停采天数、销售占比
- 量级异常：环比下降比例

可以在 `WarningService` 里新增一个方法，把当前客户的活跃预警整理成简短上下文。

3. **AI 分析模块**

建议新增：

```text
app/churn/ai_analysis.py
```

职责：

- 拼接输入上下文
- 调用大模型
- 要求模型返回固定 JSON
- 校验输出字段
- 强制每个原因必须带 evidence
- 没证据的内容只能进入 `questions_for_sales` 或 `missing_information`

4. **缓存 AI 分析结果**

建议新增一张表，例如：

```text
churn_ai_analyses
```

字段大致包括：

- `analysis_id`
- `sales_id`
- `customer_id`
- `warning_ids`
- `input_hash`
- `model_name`
- `analysis_json`
- `created_at`
- `updated_at`

这样同一客户的拜访记录和预警没变化时，可以直接返回缓存，避免每次都调大模型。

5. **路由层新增接口**

可以在 `app/churn/routes.py` 增加：

```http
POST /api/warnings/ai-analysis
```

或：

```http
GET /api/customers/{customer_id}/churn-analysis?sales_id=000455
```

我更建议第一个，保持在流失预警模块下。

**推荐落地顺序**

1. 先做不落库版本：输入 `sales_id + customer_id`，实时拉拜访记录和预警，调用大模型返回分析。
2. 再加缓存表，避免重复调用。
3. 再加销售反馈字段，例如“有用/无用”“真实原因”“是否挽回成功”。
4. 最后再做批量异步任务，在预警生成后自动产出 AI 分析。

**结论**

可以实现，难度不高。这个项目已经有三块关键基础：

- 已有流失预警结果，可以提供“为什么系统认为有风险”
- 已能调用 CRM 拜访接口，可以拿到最近三次拜访记录
- 已有 `engine → service → repository → routes` 分层，适合把 AI 分析作为 `churn` 模块的增强能力接进去

真正需要新增的是：**拜访记录读取方法、AI 分析服务、结构化输出 schema、结果缓存表和接口**。我建议先按 MVP 做单客户实时分析接口，把刚才这种分析结果跑通。

---

### Turn 5  *(2026-04-17 07:03)*

**User:**

先按 MVP 做单客户实时分析接口，把刚才这种分析结果跑通。我选择deepseek-reasoner模型

**Assistant:**

我会先按 MVP 做成同步实时接口：读取该销售的拜访记录、按客户筛选，再用现有订单/预警规则生成上下文，最后调用 DeepSeek 返回结构化分析。先确认依赖和现有 schema，再动代码。

实现上我会把 LLM 调用做成可注入组件：生产环境走 `DEEPSEEK_API_KEY`，测试里用假的 analyzer，避免测试打真实大模型接口。接口本身会实时跑现有规则，不依赖预警已经落库。

我现在开始改代码：新增共享的 `VisitRecord`、CRM 拜访读取方法、DeepSeek 分析器、Service 编排方法和 `/api/warnings/ai-analysis` 接口，同时补测试覆盖接口和拜访解析。

主要代码路径已经接上了。我先跑相关测试，重点看路由、CRM 拜访解析、DeepSeek 配置默认值这些有没有破坏现有行为。

测试通过，只有一个 UTC 时间写法警告，我已经改成时区感知写法。接下来补 README 的接口和环境变量说明，避免部署时不知道 `DEEPSEEK_API_KEY` 要怎么配。

接口、服务和 CRM 解析都已完成，README 也补了配置说明。我现在跑全量测试，确认这次新增能力没有影响原来的检测和仓储逻辑。

我再给 `docs/api-guide.md` 加一个最小请求/响应示例，因为这是新增 API 契约，前端联调会直接用到。

文档示例也补上了。代码没有再动核心逻辑，但我会再跑一次全量测试确认最终状态。

已按 MVP 跑通单客户实时 AI 分析接口，使用 `deepseek-reasoner`，不落库缓存。

主要实现：

- 新增 `POST /api/warnings/ai-analysis`，请求体：
  ```json
  {
    "sales_id": "000455",
    "customer_id": 115849
  }
  ```
- 实时拉取客户最近三次拜访记录。
- 实时运行现有 `frequency / product / volume` 预警规则，生成“预警简述”。
- 汇总近期订单摘要、拜访记录、预警简述，调用 DeepSeek。
- 强制模型返回结构化 JSON，并过滤掉没有证据的 `likely_reasons`。
- 未配置 `DEEPSEEK_API_KEY` 时接口返回 `503`。

核心代码位置：

- DeepSeek 调用与 JSON 结果校验：[app/churn/ai_analysis.py](/mnt/e/churn-warning-engine/app/churn/ai_analysis.py):25
- 单客户分析编排：[app/churn/service.py](/mnt/e/churn-warning-engine/app/churn/service.py):412
- API 路由：[app/churn/routes.py](/mnt/e/churn-warning-engine/app/churn/routes.py):217
- 请求/响应 schema：[app/churn/schemas.py](/mnt/e/churn-warning-engine/app/churn/schemas.py):92
- CRM 拜访记录读取：[app/shared/crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py):726
- DeepSeek 配置项：[app/churn/config.py](/mnt/e/churn-warning-engine/app/churn/config.py):33
- API 文档示例：[docs/api-guide.md](/mnt/e/churn-warning-engine/docs/api-guide.md):280
- README 配置说明：[README.md](/mnt/e/churn-warning-engine/README.md):153

需要配置：

```bash
export DEEPSEEK_API_KEY="your-key"
export DEEPSEEK_MODEL="deepseek-reasoner"
```

默认 `DEEPSEEK_API_BASE_URL` 是 `https://api.deepseek.com`。实现按 DeepSeek 官方 OpenAI-compatible Chat API 接入，并使用 `/chat/completions` 与 `response_format: {"type":"json_object"}`。参考 DeepSeek 官方文档：  
https://api-docs.deepseek.com/  
https://api-docs.deepseek.com/api/create-chat-completion

验证结果：

```text
130 passed, 4 warnings in 1.07s
```

4 个 warning 是现有 MySQL/SQLite 测试里的 SQLAlchemy sqlite date adapter deprecation warning，和本次 AI 分析接口无关。

下一步比较自然的是加结果缓存表和销售反馈字段，这样同一客户的拜访记录和预警没有变化时不用重复调用大模型。

---

### Turn 6  *(2026-04-17 07:14)*

**User:**

继续，加结果缓存表和销售反馈字段

**Assistant:**

我会把缓存做进 churn 仓储层：实时接口先构造同一份输入上下文并计算 `input_hash`，命中缓存直接返回；未命中才调 DeepSeek 并保存。销售反馈单独做一个更新接口，避免反馈影响缓存命中。

现有仓储只有 job/warning/run 三类记录，我会新增 `ChurnAiAnalysisRecord` 模型和仓储方法，并给 MySQL/InMemory 两套实现保持测试和生产一致。

缓存设计会允许同一输入直接命中旧结果，同时支持 `force_refresh` 重新生成一条新分析；反馈绑定到具体 `analysis_id`，不会改变输入 hash。这样不会因为销售反馈导致重复调用模型。

Service 和路由已经接入缓存/反馈，接下来先跑相关测试看接口 shape 是否有不兼容，再补测试断言缓存命中和反馈回写。

测试已经补了三个关键行为：同输入命中缓存、`force_refresh` 重新生成、反馈回写后缓存结果能带回反馈；再加了 MySQL 仓储的缓存/反馈 round trip。现在跑这些新增测试。

缓存与反馈实现通过了新增测试。我接着把 README 和 API 文档里的 “不落库” 改成缓存表语义，并补上反馈接口的请求示例。

文档现在反映缓存和反馈接口。下一步跑全量测试，包括 MySQL 仓储的新增表结构和路由响应 schema。

我把缓存表里的上下文字段命名为 `input_context`，避免使用过于通用的 `context` 列名。最后再跑一次全量测试确认迁移映射和 API 测试都稳定。

已继续完成：AI 分析结果缓存表和销售反馈字段都接上了。

新增能力：

- `POST /api/warnings/ai-analysis`
  - 现在会先基于输入上下文计算 `input_hash`
  - 同一 `sales_id + customer_id + input_hash + model_name` 会直接返回缓存
  - 请求体支持 `force_refresh: true` 强制重新调用 DeepSeek 并生成新记录
  - 响应新增 `analysis_id`、`cached`、`input_hash`、`feedback`

- `POST /api/warnings/ai-analysis/{analysis_id}/feedback`
  - 回写销售反馈
  - 支持字段：
    - `is_helpful`
    - `actual_reason`
    - `recovery_result`: `pending` / `recovered` / `lost` / `no_action` / `unknown`
    - `sales_notes`

新增/修改位置：

- 缓存与反馈编排：[service.py](/mnt/e/churn-warning-engine/app/churn/service.py):468
- AI 分析与反馈路由：[routes.py](/mnt/e/churn-warning-engine/app/churn/routes.py):218
- 请求/响应 schema：[schemas.py](/mnt/e/churn-warning-engine/app/churn/schemas.py):92
- 缓存记录模型：[models.py](/mnt/e/churn-warning-engine/app/churn/models.py):32
- 仓储接口：[repository.py](/mnt/e/churn-warning-engine/app/churn/repository.py):66
- 内存仓储实现：[memory_repo.py](/mnt/e/churn-warning-engine/app/churn/memory_repo.py):137
- MySQL 仓储实现：[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):117
- MySQL 迁移：[007_churn_ai_analyses.sql](/mnt/e/churn-warning-engine/migrations/mysql/churn/007_churn_ai_analyses.sql):1
- API 文档：[api-guide.md](/mnt/e/churn-warning-engine/docs/api-guide.md):280
- README 说明：[README.md](/mnt/e/churn-warning-engine/README.md):151

反馈请求示例：

```json
{
  "is_helpful": true,
  "actual_reason": "涨价后客户暂停主力产品采购",
  "recovery_result": "pending",
  "sales_notes": "下次拜访带总使用成本对比表"
}
```

验证结果：

```text
133 passed, 4 warnings in 1.04s
```

4 个 warning 仍是现有 SQLite date adapter 的 deprecation warning，和这次缓存/反馈功能无关。

上线 MySQL 环境前需要执行新增迁移：

```bash
migrations/mysql/churn/007_churn_ai_analyses.sql
```

---

### Turn 7  *(2026-04-17 07:26)*

**User:**

现在最后一步是不是该做批量异步任务，在预警生成后自动产出 AI 分析

**Assistant:**

是的，最后一步就是做批量异步任务，但建议不要做成“每条预警都调一次 AI”，而是做成：

**预警生成后，按客户维度自动生成 AI 分析。**

原因是一个客户可能同时有：

- `frequency` 预警
- 多个 `product` 预警
- `volume` 预警

这些应该合并进同一份客户分析里。你现在的单客户接口已经是这个逻辑：`sales_id + customer_id` 一次性汇总拜访记录、实时预警简述和订单摘要。所以批量任务只需要复用现有 `analyze_customer_churn()`。

推荐实现方式：

1. `detect_batch()` 生成预警后拿到 `warning_ids`
2. 根据 `warning_ids` 查询 warning 详情
3. 提取唯一的 `(sales_id, customer_id)`
4. 后台异步调用 AI 分析
5. 因为现在已有 `input_hash` 缓存，所以重复客户不会反复调模型
6. 失败不影响预警生成，只记录失败原因或重试

我建议加两个 Celery 任务：

```text
app.churn.tasks.generate_ai_analysis_for_customer
app.churn.tasks.generate_ai_analysis_for_warnings
```

大致职责：

- `generate_ai_analysis_for_customer(sales_id, customer_id, biz_date=None)`
  - 调用 `service.analyze_customer_churn(...)`
  - `force_refresh=False`
  - 命中缓存就直接返回

- `generate_ai_analysis_for_warnings(warning_ids)`
  - 批量读取 warning
  - 按客户去重
  - 逐个派发 `generate_ai_analysis_for_customer`

触发点可以有两个：

- **首选**：在 Celery 的检测任务 `run_detection` 结束后，拿本轮 `all_warning_ids` 派发 AI 分析任务。
- **补偿**：加一个定时任务，扫描最近未生成 AI 分析的活跃预警，避免某次 DeepSeek 失败后永远漏掉。

需要注意几个控制点：

- 只在 `DEEPSEEK_API_KEY` 配置后启用，否则跳过并打日志。
- 控制并发，避免 DeepSeek 限流。
- 对高风险客户优先生成，低风险可以延后。
- 不要阻塞预警检测主流程。
- 任务失败要可重试，但不要无限重试。
- 仍保留前端手动调用单客户接口，用于即时刷新或补生成。

所以答案是：**是，该做批量异步任务了。**  
不过它应该是对当前单客户分析接口的异步批量封装，而不是重新写一套 AI 分析逻辑。当前缓存表、反馈字段和 `force_refresh` 已经为这一步铺好了。

---

### Turn 8  *(2026-04-17 07:29)*

**User:**

对的，你帮我做吧

**Assistant:**

我会把批量 AI 生成放进现有 Celery tasks：检测任务完成后按 warning 去重到客户，再派发客户级 AI 任务；同时保持主检测流程不被 DeepSeek 失败阻断。先看现有 task 写法和测试覆盖，再改。

我会先加一层任务辅助函数，方便测试：从 warning_ids 提取唯一客户、生成客户 AI 任务、检测任务结束后派发批量任务。这样 Celery 任务本身保持薄，失败不会影响预警检测主流程。

任务代码已加好两个 Celery 任务：客户级生成和 warning 批量派发。现在我补单元测试，覆盖按客户去重、未配置 AI 时跳过、配置 AI 时派发客户任务。

自动派发已经放在 `run_detection` 末尾，检测汇总返回里会带 `ai_analysis` 派发状态。现在补文档说明任务名和自动触发条件。

文档已补充批量异步生成任务和检测 summary 的 `ai_analysis` 字段。我现在跑全量测试，确认任务改动没有影响现有检测流程。

已做完批量异步生成。

现在 `app.churn.tasks.run_detection` 在完成预警检测、保存 `DetectionRun` 后，会自动按本轮 `warning_ids` 去重到客户维度，并派发客户级 AI 分析任务。一个客户即使同时有 `frequency` 和多个 `product` 预警，也只会派发一次 AI 分析。

新增内容：

- 客户级 Celery 任务：`app.churn.tasks.generate_ai_analysis_for_customer`
- 预警批量派发任务：`app.churn.tasks.generate_ai_analysis_for_warnings`
- `run_detection` 自动派发 AI 分析，并在返回 summary 里增加 `ai_analysis`
- 未配置 `DEEPSEEK_API_KEY` 时自动跳过，不影响预警检测
- AI 派发失败也不会阻断检测主流程

核心位置：

- 批量去重与派发逻辑：[tasks.py](/mnt/e/churn-warning-engine/app/churn/tasks.py):30
- 客户级 AI 任务：[tasks.py](/mnt/e/churn-warning-engine/app/churn/tasks.py):100
- 批量 warning 任务：[tasks.py](/mnt/e/churn-warning-engine/app/churn/tasks.py):151
- `run_detection` 完成后自动派发：[tasks.py](/mnt/e/churn-warning-engine/app/churn/tasks.py):257
- 文档说明：[api-guide.md](/mnt/e/churn-warning-engine/docs/api-guide.md):374
- README 说明：[README.md](/mnt/e/churn-warning-engine/README.md):160

检测任务返回里现在会带类似：

```json
{
  "ai_analysis": {
    "status": "dispatched",
    "customer_count": 2,
    "dispatched": [
      {"sales_id": "000455", "customer_id": 115849}
    ]
  }
}
```

如果没配 DeepSeek：

```json
{
  "ai_analysis": {
    "status": "skipped",
    "reason": "ai_not_configured",
    "customer_count": 0,
    "dispatched": []
  }
}
```

也可以手动补生成：

```python
generate_ai_analysis_for_warnings.delay(["warning-id-1", "warning-id-2"], "2026-04-17")
```

验证结果：

```text
136 passed, 4 warnings in 1.04s
```

这一步自动触发点放在 Celery 的 `run_detection` 里；如果是手动调用 HTTP 的单批检测接口，仍然可以用单客户 AI 分析接口或批量 Celery 任务补生成。

---

### Turn 9  *(2026-04-17 07:35)*

**User:**

现在AI分析是还用不了吧，在哪里配置deepseek API key

**Assistant:**

对，现在如果没有配置 `DEEPSEEK_API_KEY`，AI 分析接口和 Celery 自动生成都会跳过或返回不可用。

配置位置就是环境变量：

```bash
export DEEPSEEK_API_KEY="你的 DeepSeek API Key"
export DEEPSEEK_MODEL="deepseek-reasoner"
```

默认已经是 `deepseek-reasoner`，所以最少只需要配：

```bash
export DEEPSEEK_API_KEY="你的 DeepSeek API Key"
```

如果你是本地直接启动 API：

```bash
DEEPSEEK_API_KEY="你的 DeepSeek API Key" uvicorn app.main:app --reload
```

如果同时启动 Celery，也要给 Celery 进程配置同一个环境变量：

```bash
DEEPSEEK_API_KEY="你的 DeepSeek API Key" celery -A app.celery_app worker --loglevel=info --beat
```

如果是 Docker / docker-compose 部署，需要把它加到容器环境变量里。通常是在 `.env` 或 `docker-compose.yml` 的 `environment` 里加：

```env
DEEPSEEK_API_KEY=你的 DeepSeek API Key
DEEPSEEK_MODEL=deepseek-reasoner
```

相关配置项在代码里是：

- [config.py](/mnt/e/churn-warning-engine/app/churn/config.py):33
- [README.md](/mnt/e/churn-warning-engine/README.md):167

其他可选配置：

```bash
export DEEPSEEK_API_BASE_URL="https://api.deepseek.com"
export DEEPSEEK_TIMEOUT_SECONDS="60"
export DEEPSEEK_MAX_TOKENS="4096"
```

配好后重启 API 和 Celery 进程，AI 分析就会启用。未配置时：

- `POST /api/warnings/ai-analysis` 会返回 `503`
- Celery 自动生成会返回/记录 `ai_not_configured`，但不影响预警检测。

---
