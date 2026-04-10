# 项目代码审查报告（第五轮）

> 审查时间：2026-04-10
> 审查范围：churn-warning-engine 全部源码 + 测试
> 测试状态：56 passed in 0.43s（修复前后均全部通过）

## 整体评价

架构分层清晰（engine → service → repository → API），EWMA 预测算法设计合理，去重机制（active_slot NULL 技巧）巧妙。代码质量整体较高，以下按优先级列出发现的问题。

---

## 一、Bug / 正确性问题

### 1. `service.py:49` — 局部变量遮蔽导入的函数名

```python
from app.date_utils import is_month_end, previous_month_end  # 导入了函数
...
def _orders_since_date(self, job_type: JobType, biz_date: date) -> date | None:
    if job_type == "volume":
        current_month_start = biz_date.replace(day=1)
        previous_month_end = current_month_start - timedelta(days=1)  # 遮蔽了同名函数
        return previous_month_end.replace(day=1)
```

局部变量 `previous_month_end` 遮蔽了从 `date_utils` 导入的同名函数。当前不会导致运行错误，但如果后续在此方法中调用 `previous_month_end()` 函数会触发 `AttributeError`。建议将局部变量重命名为 `prev_month_end`。

### 2. `db.py:421-457` — 连接池容量极小时存在死锁风险

`warning_dedup_lock` 在 session `s`（连接 C1）上持有 `GET_LOCK`，然后 `yield` 期间 `find_active_warning`/`save_warning` 等操作各创建新 session（需要新连接）。默认连接池大小为 5，正常无问题。但如果生产环境配置 `pool_size=1` 且无 overflow，C1 持锁等待 yield 返回、yield 内的操作等待 C1 释放 → **死锁**。

建议在 `create_engine` 时显式设置 `pool_size>=2`，或在文档中注明最低池大小要求。

### 3. `schemas.py` — `WarningResponse` 缺少 `biz_date` 字段

`WarningRecord` 有 `biz_date`，但 API 响应 `WarningResponse` 中未暴露。消费者无法直接获知预警对应的业务日期，需要从 `feature_summary` 中间接推断。如果这是有意为之，建议在 schema 注释中说明；否则建议添加该字段。

### 4. `migrations/001_warning_records_active_dedup.sql` — 迁移可能在存量数据上失败

新增 `dedup_key VARCHAR(120) DEFAULT ''` 后，所有存量活跃记录的 `dedup_key` 全部为 `''`，`active_slot` 全部为 `'1'`。若同一客户存在两条同类型活跃预警，`UNIQUE (customer_id, anomaly_type, dedup_key, active_slot)` 约束会添加失败。建议在 `CREATE CONSTRAINT` 前增加去重/清理步骤，或在迁移注释中明确说明前置条件。

---

## 二、设计 / 健壮性问题

### 5. 状态字符串字面量散落各处

`"pending_enrich"` 和 `"enriched"` 在 `service.py`、`repository.py`、`db.py` 中以字面量出现。`warning_keys.py` 已定义了 `RESOLVED_WARNING_STATUS` 和 `ACTIVE_WARNING_STATUSES`，但缺少对应的 `PENDING_ENRICH_STATUS = "pending_enrich"` 和 `ENRICHED_STATUS = "enriched"` 常量。若某天需要修改状态值，遗漏某处会导致逻辑不一致。

### 6. `repository.py:153` — 冗余的 `setdefault`

```python
orders_by_customer = {customer_id: [] for customer_id in customer_ids}  # 已初始化
...
orders_by_customer.setdefault(order.customer_id, []).append(order)  # setdefault 永远命中已有 key
```

`db.py:253` 也存在同样的冗余。不是 bug，但会误导阅读者以为字典可能不包含该 key。直接使用 `orders_by_customer[order.customer_id].append(order)` 即可。

### 7. `db.py:384-410` — `find_active_warning` 查询未利用 `active_slot` 索引

查询条件用 `status.in_(ACTIVE_WARNING_STATUSES)` 过滤，但索引 `ix_warning_dedup_lookup` 包含的是 `active_slot` 而非 `status`。可以额外加上 `WarningRecordRow.active_slot == ACTIVE_WARNING_SLOT` 条件，让查询走索引更高效。

### 8. 无 API 认证和 CORS 配置

所有端点公开访问，无鉴权中间件。若仅作为内部服务可接受，但作为生产 API 应至少添加 token 校验。同样缺少 CORS 中间件，浏览器端调用会被阻断。

---

## 三、测试覆盖缺口

| 缺失覆盖                                          | 风险 |
|-------------------------------------------------|------|
| `MySQLRepository` 完全无测试                        | 高   |
| `_shorten_key()` 超长 key 的 SHA1 截断路径            | 中   |
| 严重程度分级函数 (`_severity_from_ratio/drop/rise`) 边界值 | 中   |
| 多个关键产品同时异常的场景                              | 低   |
| `date_utils` 的 `is_month_end` / `previous_month_end` | 低   |
| re-enrich（从 enriched 再次 enrich）                | 低   |

---

## 四、小问题

- `engine.py:218` — `if predicted_days else 0.0` 是冗余防御，因为上方 `predicted_days <= 0` 已 early return。
- `InMemoryRepository._seed_demo_data()` 在每次实例化时运行，测试中需要 `demo_reference_date` 来保证数据稳定。目前所有测试都正确设置了该参数，但无测试验证遗漏的情况。
- `tasks.py` 的 `run_detection` 返回值包含 `warning_ids` 列表，如果异常量极大，Celery result backend 序列化可能成为瓶颈。

---

## 五、已执行的修复

审查后对可直接修复的代码问题进行了修改，修复前后 56 项测试全部通过。

| # | 修复内容 | 涉及文件 |
|---|---------|---------|
| 1 | 局部变量 `previous_month_end` → `prev_month_end`，消除对导入函数的遮蔽 | `service.py` |
| 2 | 新增 `PENDING_ENRICH_STATUS` / `ENRICHED_STATUS` 常量，替换全部散落的字面量 | `warning_keys.py`, `engine.py`, `repository.py`, `service.py`, `db.py` |
| 3 | `WarningResponse` 添加 `biz_date` 字段 + `_serialize` 同步映射 | `schemas.py`, `main.py` |
| 4 | 冗余 `setdefault` → 直接下标访问 | `repository.py`, `db.py` |
| 5 | `find_active_warning` 用 `active_slot == "1"` 替代 `status.in_(...)` 以命中索引 | `db.py` |
| 6 | 移除 `deviation_ratio` 的冗余 `if predicted_days` 防御 | `engine.py` |

### 未在代码中修改的项（需运维/架构层面决策）

- **连接池死锁风险**（第 2 项）：取决于生产环境 `pool_size` 配置，建议部署文档中注明最低池大小要求。
- **迁移脚本存量数据冲突**（第 4 项）：需根据实际存量数据情况决定是否在迁移前添加清理步骤。
- **API 认证和 CORS**（第 8 项）：属于部署架构决策，可能由网关层统一处理。

---

## 总结

代码质量良好，核心算法和去重逻辑经过充分测试。主要风险点是 **MySQL 仓储层完全没有测试覆盖** 和 **连接池死锁可能性**。其余问题为命名、常量管理、索引优化等可维护性改进，已在本轮审查中完成修复。
