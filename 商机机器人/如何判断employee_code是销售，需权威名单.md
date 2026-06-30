# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `3fb91b3e-57ae-4462-9bee-f85eb419f3e8`
- **范围**: Turn 1-3
- **导出时间**: 2026-06-30 15:24:20

---

### Turn 1  *(2026-06-30 07:16)*

**User:**

在这个项目中，我是怎么哪些employee_code是销售?

---

### Turn 2  *(2026-06-30 07:16)*

**User:**

在这个项目中，我是怎么知道哪些employee_code是销售?

**Assistant:**

我来探查代码库，看看这个项目里 `employee_code` 是怎么和"销售/业务员"挂钩的。

`employee_repo.py` 名字最像答案所在。我并行看几个关键文件。

`employee_repo.py` 只是"给定一个 employee_code 取上下文"，没回答"这个 code 凭什么算销售"。真正的来源应该在别处——churn 是任务驱动闭环，得遍历销售名单。我继续找"销售名单从哪来"。

churn 用 `sales_id`、lead 用 `employee_code`，两套来源不同。我再确认几处：lead 的 employee_code 从哪进来、churn 的销售名单是配置还是全量派生。

churn 的 HTTP 实现（生产路径）怎么列销售名单，我确认下。

关键一环：churn HTTP 路径优先用 `configured_sales_ids`。我确认它从哪配进来。

摸清了。这个项目里**没有"销售名册表"或"角色=销售"的判定** —— 两个模块各用各的口径，而且都不是去查"谁的职位是销售"，而是反过来从"有没有客户/成交"来定义。

## lead 模块：`employee_code` 由调用方传入，系统不自己判定

- `employee_code` 是**请求体里带进来的**，前端/上游传谁就是谁 —— `app/lead/schemas/request.py:27` `employee_code: str = Field(..., min_length=1, description="业务员编号")`。
- 系统只做 **strip + 非空** 校验（`request.py:17-22`、各 `_strip_employee_code`），**不查任何名册**。
- 唯一的"隐式校验"在取数那一步：`app/lead/repositories/employee_repo.py:37-43` 查 `crm_sale_issue_detail WHERE employee_code=%s AND is_del=0`（出库明细=成交）。这个 code 有成交 → 拿到行业上下文；查不到 → 空上下文（`is_empty`）。**它从不断言"你是销售"，只断言"这个 code 有没有成交历史"。**

## churn 模块：`sales_id` 名单两个来源，优先级如下

入口是 `list_sales_ids()`（`app/churn/service.py:213` 遍历它跑闭环）：

1. **配置覆盖（灰度/排障用）** —— `CRM_API_SALES_IDS` 环境变量，逗号分隔，经 `app/churn/factory.py:56-57` 解析成 `configured_sales_ids`，命中则直接返回（`crm_api.py:500-501`）。`.env.example:22` 原话："适合灰度或临时排障；频繁变化的全量名单不建议维护在 env"。**这就是你那 ~50 业务员灰度的开关。**
2. **数据派生（没配 env 时的默认）** —— "谁名下挂着客户/有成交，谁就是销售"：
   - MySQL 实现：`app/churn/crm_mysql.py:56-59` `SELECT DISTINCT sales_id FROM crm_customers`。
   - HTTP 实现：`crm_api.py:502` 走 `_ensure_loaded()`，从订单/客户 API（`getSalesmanCustomer` / `getSalesmanSaleOrder`）拉数后取出现过的 `sales_id` 集合。

## 一个要留意的点

两条路都**没有"在职/离职"或"角色=销售"的 gating**：离职但名下仍挂客户、或历史有成交的人照样会被算进来。真要精确控制范围，churn 侧靠 `CRM_API_SALES_IDS` 手工圈定，lead 侧则完全取决于上游传什么 `employee_code` 进来。

简言之：**这个系统不"认识"谁是销售，它只认"谁名下有客户/有成交"**——churn 可用 env 手工圈名单，lead 则信任请求方传入。

---

### Turn 3  *(2026-06-30 07:22)*

**User:**

也就是并没有销售人员清单，只靠订单/客户 API间接获取销售人员id

**Assistant:**

对，基本就是这样，churn 侧默认就是这么干的。但有两个地方值得拧精确一点:

**1. 严格说不是"完全没有清单",而是"没有维护一份权威名册"。** 有一个**可选的手工清单**入口 `CRM_API_SALES_IDS`(env),命中时直接用、不去 API 派生(`crm_api.py:500`)。它的定位是灰度/排障的临时圈定,不是长期维护的全量名册——`.env.example:22` 自己也说"频繁变化的全量名单不建议维护在 env"。所以优先级是:**手工 env 清单(若配) > 否则从订单/客户 API 间接派生**。

**2. "间接派生"只适用于 churn;lead 连派生都没有。** 这点区别要分清:

- **churn**:确实是间接拿 id —— `getSalesmanCustomer`/`getSalesmanSaleOrder` 拉回数据后,把出现过的 `sales_id` 收集成集合(HTTP 实现),或 `SELECT DISTINCT sales_id FROM crm_customers`(MySQL 实现)。**"谁名下有客户/有成交,谁就是销售"。**
- **lead**:不从任何 API"获取销售 id"。`employee_code` 是**请求方直接传进来的**,lead 只是拿这个已知的 code 去 `crm_sale_issue_detail` 查它的成交上下文。换句话说,lead 不负责"发现谁是销售",它信任上游已经知道。

所以更准确的一句话:**churn 默认靠订单/客户数据间接派生销售 id(可被 env 手工清单覆盖);lead 则根本不判定,由调用方传入。** 全项目都没有一张"角色=销售"的权威表。

---
