# Churn Warning Engine 代码审查 (Claude Opus 4.6)

> 审查时间: 2026-04-10
> 项目路径: /mnt/e/churn-warning-engine
> 测试结果: 56 passed in 0.41s

---

## 项目结构

```
app/
  __init__.py
  config.py          — Settings dataclass + 环境变量加载
  models.py           — 领域模型 (Customer, Order, WarningJob, WarningRecord)
  schemas.py          — Pydantic API 契约
  engine.py           — 核心检测引擎 (EWMA, IQR, severity, 参数推荐)
  repository.py       — Repository 抽象 + InMemoryRepository
  db.py               — MySQLRepository (可选)
  service.py          — WarningService 任务编排/去重/enrich/resolve
  main.py             — FastAPI 入口
  tasks.py            — Celery 定时任务
  date_utils.py       — 日期工具
  job_dates.py        — Celery 任务日期解析
  warning_keys.py     — 去重 key 管理
  service_factory.py  — 工厂 + lru_cache 单例
tests/
  test_engine.py
  test_models.py
  test_service.py
  test_config.py
  test_api_and_factory.py
  test_job_dates.py
migrations/
  mysql/001_warning_records_active_dedup.sql
```

---

## 整体评价

这是一个设计良好的客户流失预警引擎。分层清晰（engine → service → repository → API），算法合理（EWMA + IQR + 双因子置信度），去重机制完善，测试覆盖率高。以下是按严重程度排列的发现：

---

## 一、Bug / 正确性问题

### 1. `_prediction_confidence` 空列表返回 0.0 但缺少调用方防护

`app/engine.py:84` — `_prediction_confidence([])` 返回 0.0，但在 `predict_cycle` 中，如果 `_remove_iqr_outliers` 返回非空列表后传给 `_ewma`，confidence 计算路径是安全的。不过 `_prediction_confidence` 的空列表分支实际永远不会被触发（因为 `predict_cycle` 在 `filtered` 为空时已提前返回），属于死代码。轻微问题，不影响运行。

### 2. `InMemoryRepository.get_job` / `get_warning` 错误信息不一致

`app/repository.py:168-169` — 直接 `self.jobs[job_id]` 抛出的 KeyError 没有描述信息，而 `app/db.py:306` 的 `MySQLRepository` 会抛出 `KeyError(f"job {job_id} not found")`。虽然都是 KeyError，但调试时内存仓储的错误信息更不友好。

---

## 二、安全与健壮性

### 3. `.gitignore` 不完整

当前 `.gitignore` 缺少以下条目：

- `*.egg-info/` — `churn_warning_engine.egg-info/` 已被提交到仓库
- `.env` — 防止敏感配置泄露
- `dist/`、`build/` — 构建产物

### 4. `MySQLRepository` session 无显式 rollback

`app/db.py` 中 `save_job`、`save_warning`、`update_warning` 等方法在 `commit()` 失败时依赖 Session 上下文管理器隐式回滚。虽然 SQLAlchemy 2.0+ 支持这种模式，但显式 try/except + rollback 会更稳健，特别是在 `warning_dedup_lock` 内的写操作。

### 5. `db.py:421` — SHA1 用于锁名生成

SHA1 在此场景下（非密码学用途、仅用于 MySQL GET_LOCK 名称生成）是可接受的，但建议在注释中说明这不是安全用途。

---

## 三、设计与架构建议

### 6. engine.py / service.py 缺少日志

`app/tasks.py` 有 logging，但核心检测引擎和服务层完全没有日志。在生产环境中，当检测产生预警或去重刷新时，缺少日志会增加排查难度。建议在关键路径（预警生成、去重命中、enrich/resolve 状态变更）添加 INFO/DEBUG 级别日志。

### 7. Celery beat 调度没有防重叠机制

`app/tasks.py` 中的定时任务（每周一频率检测、每月量级检测）如果上一次任务未完成就触发下一次，会产生并发检测。虽然去重锁能防止重复预警，但两个任务会做重复计算。可以考虑使用 Celery 的 `solo` pool 或任务锁来避免。

### 8. `batch_id` 格式依赖冒号分割

`app/service.py:107` — `batch_id` 格式为 `{job_id}:{index}`，而 `job_id` 是 UUID（不含冒号），所以 `split(":")` 有效。但如果将来 `job_id` 格式变化，这里会断裂。考虑用更明确的分隔符或结构化传参。

### 9. `list_pending_warnings` 缺少分页

`app/main.py:103` — 只用了 `max_pending_enrich` 限制条数，没有 offset/cursor 参数。当待丰富预警较多时，前端无法翻页查看。

### 10. `detect_batch` 返回的 `anomaly_count` 语义模糊

`app/service.py:200` — `anomaly_count` 是引擎检测到的原始异常数（含可能被去重合并的），而 `warning_ids` 是实际保存/复用的预警 ID。两者可能不一致（例如重跑时 anomaly_count > 0 但没有新预警）。建议命名为 `detected_count` 或在文档中说明。

---

## 四、测试覆盖缺口

当前 56 个测试覆盖了主要场景，以下是建议补充的边界用例：

| 缺失场景 | 说明 |
|---|---|
| `_severity_from_ratio` 边界值 | ratio 恰好等于 1.5 和 2.0 时的分级 |
| `_shorten_key` 长 key 路径 | product_id 超长导致 dedup_key > 120 字符 |
| `active_slot_for_status` | 各状态返回值的直接测试 |
| 多个关键产品同时触发 | `detect_product_anomaly` 返回多条预警 |
| `config.py` 更多边界 | `min_order_count=2` (最小合法值)、`volume_rise_threshold=0` (非法) |
| `InMemoryRepository.get_orders_for_customers` | 空 `customer_ids` 输入 |

---

## 五、代码质量小项

| 位置 | 描述 |
|---|---|
| `app/db.py:19` | `OrderedDict` 已导入但 Python 3.7+ 的普通 dict 已保序，可用普通 dict 替代 |
| `app/repository.py:246-251` | 演示数据 `add()` 闭包内 `nonlocal oid` 方式可行但嵌套较深，可考虑用 `enumerate` 重构 |
| `app/engine.py:116-119` | `zip(ordered[:-1], ordered[1:])` 中额外过滤 `days > 0` — 由于上面已做 `sorted(set(...))` 去重，相邻日期不可能相同，此过滤是死代码 |
| `explain.md` (29KB) | 这个文件看起来是生成的项目说明文档，如果仅供参考建议移出版本控制或放到 docs/ 目录 |

---

## 总结

项目整体质量**较高**，核心算法正确，分层清晰，去重/锁机制设计合理。主要改进方向：

1. 补全 `.gitignore`，清理 `egg-info` 产物
2. 为 engine/service 层添加关键日志
3. 补充上述边界测试用例
4. 考虑 `list_pending_warnings` 分页支持

没有发现严重的安全漏洞或阻塞性 Bug。
