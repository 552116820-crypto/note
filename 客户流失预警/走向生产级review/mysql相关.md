get_warning() 目前一次性 SELECT 全部 MEDIUMTEXT 列，在真实 MySQL 链路上同样会读超时。这会导致 Celery 里的 AI 派发任务也失败。我先把 get_warning() 改成窄列 + 分块读取长文本，再继续派发验证。

# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019d9f35-c2a8-7650-a530-579e1563d5e8`
- **范围**: Turn 1-2
- **导出时间**: 2026-04-18 16:10:25

---

### Turn 1  *(2026-04-18 06:23)*

**User:**

在warning_record表中sales_message信息哪来的？active_slot是什么？parameter_recommendation是什么

**Assistant:**

我先看一下 `warning_record` 相关的模型、迁移和写入逻辑，确认这些字段是在哪一层生成和落库的。

搜索结果显示这三个字段都集中在 `app/churn`：`sales_message` 和 `parameter_recommendation` 是预警对象自身的业务字段，`active_slot` 是 MySQL 去重用字段。我继续读生成和更新路径，避免只按字段名猜。

现在可以看到两条来源：检测引擎先生成基础话术；后续如果调用“丰富预警”接口，会用请求里的内容覆盖为 enriched 状态。我再看一下接口 schema，确认字段在 API 上是怎么暴露的。

你说的应该是表 `warning_records`。这三个字段含义如下。

**`sales_message` 从哪来**

它不是 CRM 原始字段，而是流失预警引擎生成的“给销售看的处理话术”。

生成路径是：

1. 检测引擎在发现异常时创建 `WarningRecord`，其中包含 `sales_message`。
   位置：[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):377

2. 不同异常类型有不同模板：
   - `frequency`：基于预测下单日期、逾期天数生成“建议确认项目进度、库存和采购计划”等话术。
     位置：[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):489
   - `volume`：基于月度数量上升/下降生成“核查停工、竞品替换、回款节奏”等话术。
     位置：[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):571
   - `product`：基于关键产品采购间隔拉长生成“确认项目状态和替代品牌情况”等话术。
     位置：[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):737

3. 保存到 MySQL 时直接写入 `warning_records.sales_message`。
   位置：[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):445

另外，`sales_message` 后续可能被人工/外部系统覆盖：`POST /api/warnings/{warning_id}/enrich` 会把请求里的 `sales_message` 写回预警，并把状态改为 `enriched`。
位置：[service.py](/mnt/e/churn-warning-engine/app/churn/service.py):323

所以判断来源可以看状态：

- `pending_enrich`：通常是检测引擎自动生成的基础话术。
- `enriched`：通常是 enrich 接口传入的人工/AI 增强话术。
- `resolved`：保留历史话术，但该预警已关闭。

**`active_slot` 是什么**

`active_slot` 是数据库去重字段，不是业务展示字段。

规则在代码里是：

- `pending_enrich`、`enriched` → `active_slot = '1'`
- `resolved` → `active_slot = NULL`

位置：[warning_keys.py](/mnt/e/churn-warning-engine/app/churn/warning_keys.py):142

它配合唯一约束使用：

```sql
UNIQUE (customer_id, anomaly_type, dedup_key, active_slot)
```

位置：[000_initial_schema.sql](/mnt/e/churn-warning-engine/migrations/mysql/churn/000_initial_schema.sql):42

作用是：同一个客户、同一种异常、同一个去重键，只允许存在一条“活跃预警”。但已解决的预警 `active_slot = NULL`，利用 MySQL 唯一索引中 `NULL` 不互相冲突的特性，允许历史已解决记录保留多条，也允许未来再次产生新的活跃预警。

简化理解：

| status | active_slot | 含义 |
|---|---:|---|
| `pending_enrich` | `'1'` | 活跃，待丰富 |
| `enriched` | `'1'` | 活跃，已丰富 |
| `resolved` | `NULL` | 已关闭，不参与活跃去重 |

**`parameter_recommendation` 是什么**

`parameter_recommendation` 是预警引擎给出的“参数调优建议”，以 JSON 字符串形式存进 `MEDIUMTEXT` 字段。

模型字段定义在：[models.py](/mnt/e/churn-warning-engine/app/churn/models.py):52  
MySQL 写入位置：[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):462

不同异常类型会写不同内容：

- `frequency`：
  ```json
  {
    "recommended_frequency_multiplier": 1.36
  }
  ```
  表示建议的频率异常倍数阈值。

- `volume` 下降：
  ```json
  {
    "recommended_volume_drop_threshold": 0.35
  }
  ```
  表示建议的数量下降阈值。

- `volume` 上升：
  ```json
  {
    "recommended_volume_rise_threshold": 0.5
  }
  ```

- `product`：
  ```json
  {
    "recommended_key_product_threshold": 0.3,
    "recommended_product_multiplier": 1.42
  }
  ```
  表示建议的关键产品占比阈值、产品周期异常倍数。

它也可能被 enrich 接口覆盖，接口要求请求体里必须带 `parameter_recommendation`。
位置：[schemas.py](/mnt/e/churn-warning-engine/app/churn/schemas.py):175

一句话总结：`sales_message` 是销售处理话术，`parameter_recommendation` 是检测参数建议，`active_slot` 是数据库用来保证“同类活跃预警不重复”的技术字段。

---

### Turn 2  *(2026-04-18 06:26)*

**User:**

表示建议的频率异常倍数阈值。这有什么用

**Assistant:**

我查一下这些推荐参数有没有被系统自动消费，避免把“建议值”和“生效配置”混在一起。

`recommended_frequency_multiplier` 的作用是“给人或配置系统看的阈值调优建议”，不是当前这条预警真正生效的阈值。

当前频率预警判断逻辑是：

```python
threshold_days = predicted_days * settings.frequency_multiplier
```

位置：[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):443

也就是：

- 系统预测客户正常下单周期是 `30` 天
- 当前配置 `FREQUENCY_MULTIPLIER = 2.0`
- 那么超过 `30 * 2.0 = 60` 天没下单，才触发频率异常

`recommended_frequency_multiplier` 是引擎根据这次预测的置信度，给出的“建议倍数”。例如：

```json
{
  "recommended_frequency_multiplier": 1.36
}
```

意思是：这类客户的下单周期判断可以考虑用 `1.36` 倍作为阈值。还是上面的例子，`30 * 1.36 = 40.8` 天，也就是大约 41 天没下单就可以提醒，比当前 60 天更早。

它的实际用途主要有三个：

1. 给运营/产品调参参考  
   如果很多预警都建议比当前 `2.0` 更低的倍数，说明系统可能提醒太晚；如果建议值偏高，说明当前阈值可能太敏感，容易误报。

2. 给前端或后台展示“为什么建议调整”  
   `feature_summary` 里有当前配置 `current_frequency_multiplier`，`parameter_recommendation` 里有建议值，两者可以对比展示。

3. 给人工 enrich 或后续配置系统使用  
   当前代码里没有自动把这个值写回 `FREQUENCY_MULTIPLIER`。它只是被存进 `warning_records.parameter_recommendation`，并通过 API 返回。真正生效的还是环境变量/配置里的 `FREQUENCY_MULTIPLIER`。

简单说：它现在是“诊断建议”，不是“自动生效参数”。如果没有前端展示、人工调参流程或配置中心消费它，那它对当前检测结果没有直接影响。

---
