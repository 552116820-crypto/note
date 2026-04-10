# Churn Warning Engine 代码审查记录

日期：2026-04-10  
仓库：`E:\churn-warning-engine`  
审查人：Codex 5

## 本轮用户请求

用户要求：

> 再次审查这个项目代码

随后要求：

> 把这次会话内容存进E:\note\客户流失预警\review by codex 5.md

## 审查范围

本轮按代码审查视角重新阅读了运行时代码和测试，重点关注：

- 异常检测逻辑：`app/engine.py`
- 任务编排与去重：`app/service.py`
- 内存仓储：`app/repository.py`
- MySQL 可选仓储：`app/db.py`
- API 入口与响应序列化：`app/main.py`
- 配置、任务日期、Celery 入口：`app/config.py`、`app/job_dates.py`、`app/tasks.py`
- 现有测试：`tests/`

工作区状态检查结果：当前没有未提交变更。

## 执行过的验证

运行测试：

```bash
pytest -q
```

结果：

```text
63 passed in 0.46s
```

额外做了几组针对性复现：

- 直接调用 `WarningService.start_job(..., batch_size=0)` 会使用默认批次大小。
- 直接调用 `WarningService.start_job(..., batch_size=-1)` 会创建零批次 job。
- 直接调用 `WarningService.start_job("unknown", ...)` 会先保存非法 job，后续 `detect_batch` 才抛出 `ValueError: unsupported job type: unknown`。
- 构造“原关键产品停采、其他产品继续大量采购”的数据时，当前产品异常逻辑可能因为窗口累计占比被稀释而不检测原关键产品。

## 审查结果

### 1. Medium：产品异常可能漏报“原关键产品停采/被替代”的场景

位置：

- `app/engine.py:380`
- `app/engine.py:396`

问题：

`detect_product_anomaly` 先用整个分析窗口累计各产品金额，再按这个累计占比筛选关键产品。这样如果原关键产品停止采购后，客户继续大量采购其他产品，原关键产品在整个窗口内的占比会被稀释到阈值以下，后续周期偏离判断不会执行。

影响：

如果业务目标包含“关键产品停采”或“替代品牌风险”，当前算法可能漏报真正需要关注的产品。

建议：

用“停采前的历史基线窗口”识别关键产品，再对最近缺口做检测。补充一条回归测试，覆盖“历史关键产品停采、近期其他产品继续采购”的场景。

### 2. Medium：MySQL CRM 表契约和实际 SQL 不一致

位置：

- `app/db.py:11`
- `app/db.py:229`

问题：

文件注释列出的 `crm_orders` 字段是：

```text
order_id, customer_id, sales_id, order_date, total_amount
```

但实际 SQL 查询强制使用：

```sql
o.status IN (...)
```

如果按当前文档建表或映射 CRM 表，启用 `REPOSITORY_TYPE=mysql` 后会直接 SQL 报错。

建议：

把 `status` 明确写进 CRM 表契约，或让完成状态过滤可配置/可关闭。最好为 `MySQLRepository.get_orders_for_customers` 增加最小集成测试，至少覆盖生成 SQL 所依赖的字段。

### 3. Low：`WarningService.start_job` 依赖 API 层做输入校验，服务层自身仍能创建无效任务

位置：

- `app/service.py:74`
- `app/service.py:79`
- `app/service.py:153`

问题：

API 层通过 `StartJobRequest` 约束了 `batch_size`，但 service 层直接调用时没有同等保护：

- `batch_size=0` 被 `batch_size or default` 当成未传值。
- `batch_size=-1` 会创建没有任何批次的 job。
- 非法 `job_type` 会先保存 job，之后执行批次检测时才失败。

影响：

HTTP API 路径目前相对安全，但 Celery、测试、脚本或其他内部调用路径可能绕过 API 校验，留下无效任务数据。

建议：

把 `job_type` 和 `batch_size >= 1` 校验下沉到 `WarningService.start_job`，使服务层本身保持不变量。

## 总体结论

主路径、内存仓储、去重和状态流转测试覆盖较完整；现有测试全部通过。主要风险集中在两处：

- 产品检测的业务口径可能漏掉“历史关键产品停采”。
- MySQL 生产路径缺少真实覆盖，且文档化表结构与 SQL 使用字段不一致。

本轮审查没有修改项目代码。
