# 客户流失预警系统 — Claude 会话记录 (第二轮)

> 会话日期: 2026-04-10
> 内容: 用户完成了 Codex 第二轮 review 的 4 个修复 → Claude 评价修复质量 → 全面 review 找新问题

---

## 一、评价用户完成的 4 个修复

上次会话中 Codex 提出了 5 个问题，Claude 修了第 1 个 (IQR 保序)，用户自行完成了剩余 4 个。以下是对用户实现的逐个评价。

### Fix 2: biz_date 数据泄露 — 评价: 优秀

**问题**: `get_orders_for_customer` 未限制上界，可能读到 biz_date 之后的"未来"订单，导致检测结果不可复现。

**用户的实现**:
- `repository.py` ABC: 加了 `biz_date: date | None = None` 参数，向后兼容
- InMemory: `if biz_date is not None: orders = [o for o in orders if o.order_date <= biz_date]`
- MySQL (`db.py`): `until = biz_date or date.today()` + `AND o.order_date <= :until`
- `service.py`: 传 `biz_date=job.biz_date`

**加分项**: 用户额外加了 `get_orders_for_customers` 批量方法，InMemory 和 MySQL 都有实现，service 层直接一次查出整批客户的订单，消灭了 N+1。还写了 `test_future_orders_excluded_by_biz_date` 测试来验证，构造了一条未来订单确认被过滤掉。另外 MySQL 的 `customer_count_for_sales_ids` 也改成了 `COUNT(*)` 查询，不再加载全量数据计数。

### Fix 3: 订单状态过滤 — 评价: 好，设计选择合理

**问题**: MySQL 查询未过滤订单状态，草稿/取消的订单也会参与计算。

**用户的实现**:
- 用 `completed_status_codes: tuple[int, ...]` 而不是字符串列表，更贴合实际 CRM 数据库（状态码通常是整数）
- `_parse_status_codes` 解析函数干净，空值回退到 `(1,)`
- MySQL 查询用参数化占位符 `AND o.status IN (:status_0, :status_1, ...)` 防注入
- InMemory 演示数据里没有 `status` 字段，过滤只对 MySQL 生效——这是正确的，演示数据没有无效订单需要过滤

### Fix 4: batch_id 前缀校验 — 评价: 完美

**问题**: 用 job1 的 batch_id 去调 job2 的 detect，不会报错但结果错乱。

**用户的实现**:
```python
if parts[0] != job_id:
    raise ValueError(
        f"batch_id job_id mismatch: {batch_id} (expected prefix {job_id})"
    )
```

一行搞定，加了 `test_batch_id_job_prefix_must_match` 测试：创建两个 job，用 job2 的 batch_id 去调 job1 的 detect，确认抛 ValueError。

### Fix 5: Volume 方向去重 — 评价: 优秀

**问题**: volume 预警的 up 和 down 共享去重键，导致"下降"预警会被"上升"预警覆盖。

**用户的实现**:
- ABC 和两个实现都加了 `direction: str | None = None` 参数
- InMemory `find_active_warning` 对 volume 类型走 `feature_summary.get("direction")` 匹配
- MySQL 同样逻辑
- `service.py` 从 `w.feature_summary.get("direction")` 取值传入
- `test_volume_direction_kept_separate`: 先存一个 direction=up 的预警，验证查 up 能找到、查 down 返回 None

**加分项**: MySQL 实现里用户加了 `warning_dedup_lock` 用 `GET_LOCK`/`RELEASE_LOCK` 做分布式锁，防止并发检测时的去重竞态。ABC 层给了空 `yield` 的默认实现，InMemory 不需要锁。这是生产级的考虑，超出了 Codex 提的问题范围。

**设计决策**: 用户没有把 direction 冗余到 `metadata` 里，而是让去重逻辑直接从 `feature_summary` 读。这比冗余存储更干净——同一份数据一个来源。

### 总评

4 个修复全部正确，测试从 25 个增长到 34 个（新增 9 个测试，含 3 个新测试文件中的测试）。在修复 Codex 提出的问题之外，用户还自发做了三个改进：批量查询优化、MySQL 分布式去重锁、COUNT(*) 优化。代码风格和原有代码一致，没有过度设计。

---

## 二、Claude 全面 Review (第三轮)

在前两轮修复后，对完整代码库进行新一轮审查，34 个测试全部通过。

### 问题 1: `job_dates.py` 是死代码 — tasks.py 没接入 [严重]

`resolve_run_detection_date` 专门为 volume 任务设计了"用上月末而非今天"的逻辑，测试也写了 3 个，但 `tasks.py` 根本没调用它：

```python
# tasks.py:74 — 当前代码
biz_date = date.fromisoformat(biz_date_str) if biz_date_str else None
```

volume 的 Celery beat 配置是每月 1 号 08:00 跑，`biz_date_str` 为 None，所以 `biz_date=None` 传给 service，最终用 `date.today()` = 5月1日。此时：
- 当月 = 5月1日一天的金额
- 上月 = 整个4月的金额
- 几乎必然触发"下降"误报

应该改为：

```python
from app.job_dates import resolve_run_detection_date

biz_date = resolve_run_detection_date(job_type, biz_date_str)
```

### 问题 2: Volume 月度比较的部分月问题 [中]

即使修好了问题 1，通过 API 手动传 `biz_date=2026-04-07` 跑 volume 检测，仍然是 7 天 vs 31 天的不公平比较。

两种修法选一个：
- **A.** engine 里按日均金额对比：`current_daily = current / days_in_current_month`
- **B.** service 层对 volume 类型校验 biz_date 必须是月末（拒绝部分月）

B 更简单，和 `job_dates.py` 的设计意图一致。

### 问题 3: `db.py` ORM / Raw SQL 混用 [中]

`save_warning` 和 `update_warning` 用 ORM 对象（属性名是 `feature_summary_json`, `param_rec_json`），但 `get_warning` 和 `find_active_warning` 用 raw SQL（列名是 `feature_summary`, `parameter_recommendation`）。`_warning_from_row` 也按 raw SQL 列名来读。

如果将来改列名，需要改两套地方，容易遗漏。建议统一：要么全用 ORM（`s.get` + ORM attr），要么全用 raw SQL。统一 ORM 更自然：

```python
def get_warning(self, warning_id: str) -> WarningRecord:
    with self._session() as s:
        row = s.get(WarningRecordRow, warning_id)
        if not row:
            raise KeyError(...)
        return self._warning_from_row(row)  # 改为从 ORM 对象构建
```

### 问题 4: Volume 预警 `direction` 不在 `metadata` 里 [低]

产品预警用 `metadata={"product_id": pid}` 作为去重键，volume 预警的 `direction` 却只在 `feature_summary` 里，去重时从 `feature_summary.get("direction")` 读。

功能上没问题，但语义不一致。如果将来 `feature_summary` 结构变化，去重会静默失效。建议 engine.py volume 预警加上 `metadata={"direction": direction}`，去重改读 `w.metadata.get("direction")`。

### 问题 5: `Order` frozen 但 `items` 可变 [低]

`models.py:23` `Order` 标记了 `frozen=True`，但 `items: list[OrderItem]` 是可变列表。`db.py:214` 靠 `order_map[oid].items.append(...)` 工作——依赖的就是 frozen dataclass 无法拦截列表内部变异这个行为。

语义上有欺骗性。建议去掉 Order 的 `frozen=True`，或者改 db.py 构建方式（先收集 items 再一次性构建 Order）。

### 问题 6: `pyproject.toml` Python 版本上限过窄 [低]

```toml
requires-python = ">=3.10,<3.13"
```

Python 3.13 已发布，代码全是标准库 + FastAPI/Pydantic，没有不兼容的原因。建议改为 `">=3.10"`。

### 问题 7: MySQL 缺少复合索引 [低]

`find_active_warning` 查询条件是 `customer_id + anomaly_type + status`，但 `warning_records` 表只有 `customer_id` 和 `sales_id` 的单列索引。数据量大了以后这个查询会慢。

建议在 ORM 模型上加：

```python
from sqlalchemy import Index

class WarningRecordRow(Base):
    __tablename__ = "warning_records"
    __table_args__ = (
        Index("ix_warning_dedup", "customer_id", "anomaly_type", "status"),
    )
```

### 问题 8: `predict_cycle` 的 `> 0` 过滤是死代码 [极低]

```python
intervals = [
    (cur - prev).days
    for prev, cur in zip(ordered[:-1], ordered[1:])
    if (cur - prev).days > 0  # <- 永远为 True
]
```

`ordered = sorted(set(order_dates))` 已经去重排序，连续差值必然 > 0。这个 `if` 永远不会过滤掉任何东西。不影响正确性，删掉可以减少困惑。

### 汇总表

| # | 严重度 | 问题 | 文件 |
|---|--------|------|------|
| 1 | **高** | `job_dates.py` 死代码，volume Celery 任务会误报 | tasks.py |
| 2 | 中 | API 侧 volume 部分月比较不公平 | engine.py / service.py |
| 3 | 中 | ORM / Raw SQL 混用，改列名容易出错 | db.py |
| 4 | 低 | direction 去重键位置不一致 | engine.py / service.py |
| 5 | 低 | Order frozen 但 items 可变 | models.py / db.py |
| 6 | 低 | Python 版本上限卡 3.13 | pyproject.toml |
| 7 | 低 | 缺复合索引 | db.py |
| 8 | 极低 | 死代码 `> 0` 过滤 | engine.py |

---

## 三、当前代码文件清单

```
app/
├── __init__.py
├── config.py          — Settings dataclass + load_settings()
├── models.py          — Customer, Order, OrderItem, WarningJob, WarningRecord
├── schemas.py         — Pydantic API schemas
├── engine.py          — 核心检测引擎 (EWMA, IQR, 三场景检测)
├── repository.py      — Repository ABC + InMemoryRepository
├── db.py              — MySQLRepository (SQLAlchemy)
├── service.py         — WarningService 业务编排
├── job_dates.py       — Celery 任务日期解析 (当前未接入 tasks.py)
├── main.py            — FastAPI 路由
└── tasks.py           — Celery 定时任务

tests/
├── __init__.py
├── test_engine.py     — 29 个测试 (工具函数 + 三场景 + 去重 + batch校验)
├── test_job_dates.py  — 3 个测试 (日期解析)
└── test_service.py    — 2 个测试 (批量加载 + enriched 保护)

pyproject.toml         — 依赖 + 可选组件 (mysql, celery, dev)
```

测试总数: 34 个，全部通过。
