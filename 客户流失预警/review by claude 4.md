# Review by Claude 4 (Opus 4.6)

日期: 2026-04-10

## 审查范围

对 `/mnt/e/churn-warning-engine` 项目全部源码和测试进行完整审查。

### 项目结构

```
app/
  __init__.py
  config.py          # 环境变量驱动的配置
  schemas.py         # Pydantic 请求/响应模型
  date_utils.py      # 日期工具函数
  job_dates.py       # Celery 任务日期解析
  warning_keys.py    # 去重 key 生成与状态管理
  models.py          # 领域数据模型 (dataclass)
  engine.py          # 核心检测引擎 (EWMA/IQR/置信度/三场景检测)
  service.py         # 任务编排与业务逻辑
  repository.py      # 仓储抽象基类 + InMemoryRepository
  db.py              # MySQL 仓储实现 (可选)
  service_factory.py # 工厂 + 单例缓存
  main.py            # FastAPI 入口
  tasks.py           # Celery 定时任务
tests/
  test_engine.py
  test_service.py
  test_job_dates.py
  test_api_and_factory.py
```

### 测试结果

```
48 passed in 0.39s
```

全部 48 个测试通过。

---

## 审查总结

项目整体质量很高：分层清晰（engine / service / repository / API）、参数化查询防 SQL 注入、去重逻辑完整、测试覆盖了关键路径和边界条件。以下按严重程度列出发现的问题。

---

### 1. `Order` frozen 语义与实际用法矛盾 (中)

**文件**: `models.py:25` + `db.py:253-259`

`Order` 标记了 `frozen=True`，但 `items: list[OrderItem]` 字段是可变 list。`db.py` 中直接通过 `order_map[oid].items.append(...)` 向已创建的 frozen Order 追加明细：

```python
# db.py:253-259
order_map[oid] = Order(..., items=[])
# ...
order_map[oid].items.append(OrderItem(...))  # 修改 frozen 对象的内部 list
```

Python 允许这样做（frozen 只阻止字段重赋值，不阻止 list 原地修改），所以**运行时不会报错**。但 `frozen=True` 暗示对象不可变，这里违反了语义。

**建议**: 要么去掉 `Order` 的 `frozen=True`，要么在 `db.py` 中先收集 items 再一次性构造 Order。

---

### 2. MySQL `list_pending_warnings` 缺少 ORDER BY (中)

**文件**: `db.py:432-441`

```python
"SELECT warning_id FROM warning_records "
"WHERE status = 'pending_enrich' LIMIT :lim"
```

没有指定排序，结果顺序取决于存储引擎的物理顺序，不确定且可能在不同执行间变化。对比 `InMemoryRepository` 走的是 dict 插入顺序——两个实现行为不一致。

**建议**: 加上 `ORDER BY biz_date DESC` 或 `ORDER BY warning_id`，确保结果确定性。

---

### 3. InMemoryRepository 去重无锁保护 (低)

**文件**: `repository.py:87-96`

基类 `warning_dedup_lock` 是空的 `yield`，`InMemoryRepository` 没有覆写。如果 uvicorn 以多线程模式运行，`service.py:133-164` 的 check-then-insert 去重逻辑存在竞态条件。

这对开发/演示环境不构成实际问题，但值得注意。MySQL 仓储通过 `GET_LOCK` 正确处理了并发。

---

### 4. `detect_product_anomaly` 重复过滤订单 (低)

**文件**: `engine.py:376`

```python
recent = [o for o in orders if o.order_date >= since]
```

`service.py:104` 已通过 `since_date` 在 repository 层过滤了历史订单，engine 再过滤一次。这是防御性冗余，不是 bug，但如果订单量大会浪费 CPU。

---

### 5. 其他观察（无需修改）

| 项 | 说明 |
|---|---|
| SQL 参数化 | `db.py` 全部使用 `:param` 绑定变量，无注入风险 |
| 幂等性 | `resolve_warning` 对已 resolved 的记录直接返回，正确 |
| month-end 双重校验 | `service._resolve_effective_date` 和 `engine.detect_volume_anomaly` 都检查，防御到位 |
| 量级检测边界 | drop 用 `<=` (含等于)，rise 用 `>=` (含等于)，测试已覆盖精确阈值 |
| SHA1 用途 | 仅用于 dedup key 缩短和锁名称，非加密场景，适用 |
| 批量查询 | `get_orders_for_customers` 批量加载 + JOIN 消除 N+1，性能合理 |
| `service_factory.lru_cache` | 单例 + `clear_cached_service` 可测试，设计干净 |
| 测试覆盖 | 48 个测试覆盖了三种异常检测、去重/刷新/resolve、batch_id 校验、API 400/404、工厂缓存等关键分支 |

---

## 结论

项目代码质量好，架构合理。第 1、2 点建议修复，其余为低优先级或仅供知晓。
