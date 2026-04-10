# Churn Warning Engine — 代码审查报告

## 项目概述

客户流失预警引擎，基于 FastAPI + Celery 构建，提供三种异常检测能力：
- **频率异常**：客户下单周期偏离（EWMA 指数加权预测）
- **量级异常**：月度采购金额环比大幅波动（双向：下降 + 上升）
- **产品异常**：关键产品采购间隔拉长

整体代码量约 1100 行（不含测试），测试约 400 行，属于小型但完整的业务系统。

---

## 一、架构设计 — 优点

1. **分层清晰**：`Repository → Service → API` 三层解耦，职责边界明确。Repository 抽象基类 + 两种实现（InMemory / MySQL），切换仅需一个环境变量。
2. **算法实现扎实**：EWMA 指数加权 + IQR 离群值过滤 + 双因子置信度评分，全部用纯标准库实现（零 numpy 依赖），可移植性好。
3. **去重机制完备**：`warning_keys.py` 按异常类型生成不同粒度的 dedup key（频率=全局、量级=方向、产品=产品ID），配合 MySQL 端的 `GET_LOCK` 实现并发安全。
4. **配置管理规范**：所有阈值均通过环境变量注入，`Settings` dataclass 提供类型安全的默认值。
5. **可选依赖分组**：`pyproject.toml` 中 MySQL/Celery/dev 依赖独立，部署可按需裁剪。

---

## 二、需要关注的问题

### 严重（建议优先修复）

#### 1. `Order` frozen dataclass 的可变性矛盾

`app/models.py:27` 声明 `Order` 为 `frozen=True`，但 `items: list[OrderItem]` 是可变列表。`db.py:259` 中直接 `order.items.append(...)` 修改了 "frozen" 对象的内部状态：

```python
# db.py:259 — 修改了 frozen dataclass 的 mutable 属性
order_map[oid].items.append(
    OrderItem(product_id=str(r[5]), ...)
)
```

`frozen=True` 只阻止属性重新赋值，不阻止列表内容变更。这在语义上具有误导性，且使 Order 实际上不是线程安全的。**建议**：要么去掉 `frozen=True`，要么在构造完成后不再修改 items（先收集 items 再一次性构建 Order）。

#### 2. 模块级别的副作用初始化

`app/main.py:23-32` 在模块导入时即执行 `load_settings()` 和 Repository 实例化：

```python
settings = load_settings()          # 导入时读环境变量
if os.getenv("REPOSITORY_TYPE") == "mysql":
    from app.db import MySQLRepository
    repository = MySQLRepository(settings)  # 导入时连数据库
```

问题：
- 数据库不可用时 import 即崩溃，影响测试和 CI
- 环境变量在导入后变更不生效
- `tasks.py:54-63` 中 `_build_service()` 重复了同样的选择逻辑

**建议**：使用 FastAPI 的 `lifespan` 或依赖注入机制延迟初始化；将 Repository 选择逻辑抽为公共工厂函数。

#### 3. Celery task 每次执行都重建连接池

`app/tasks.py:73` `_build_service()` 每次调用都会创建新的 `SQLAlchemy engine` + `sessionmaker`：

```python
@celery_app.task(name="app.tasks.run_detection")
def run_detection(job_type: str, ...):
    service = _build_service()  # 每次新建 engine + session pool
```

高频执行时会导致连接池泄漏或过度创建。**建议**：使用模块级单例或 `functools.lru_cache` 缓存 service 实例。

### 中等（建议计划修复）

#### 4. `batch_id` 解析不够健壮

`service.py:83` 使用 `batch_id.split(":")` 并假定恰好 2 段：

```python
parts = batch_id.split(":")
if len(parts) != 2:
    raise ValueError(...)
```

如果未来 job_id 格式包含冒号，`parts[0]` 将不是完整的 job_id。使用 `batch_id.rsplit(":", 1)` 更安全。

#### 5. 缺少 `warning_records.status` 索引

`db.py` 中 `list_pending_warnings` 用 `WHERE status = 'pending_enrich'` 查询，但 `WarningRecordRow` 没有在 `status` 列上建索引。预警记录增长后查询性能会下降。

#### 6. `_serialize` 手动映射字段

`main.py:41-54` 逐字段手动构造 `WarningResponse`。如果 `WarningRecord` 新增字段但忘记同步更新 `_serialize`，该字段会被静默丢弃。**建议**：使用 `WarningResponse.model_validate(warning, from_attributes=True)` 或等效方式自动映射。

#### 7. Legacy dedup fallback 增加了复杂度

`db.py:370-384` 中 `find_active_warning` 有一个 fallback 逻辑：当按 dedup_key 找不到时，扫描 `dedup_key=""` 的旧记录并重算 key。这是向后兼容代码，长期应清理。

#### 8. `get_orders_for_customers` 基类默认实现是 N+1

`repository.py:42-55` 基类提供的默认实现逐个调用 `get_orders_for_customer`。虽然两个子类都覆写了批量版本，但新增 Repository 实现时容易遗漏。**建议**：将此方法也改为 `@abc.abstractmethod`，强制子类提供批量实现。

### 轻微（建议逐步改善）

#### 9. 缺少 API 层测试

当前测试覆盖了 engine 和 service 层，但没有 FastAPI TestClient 的集成测试，也没有 `db.py`（MySQL Repository）和 `tasks.py`（Celery task）的测试。关键缺失：
- API 错误码映射（HTTPException 400/404）
- 请求/响应 schema 验证
- MySQL 仓储的 CRUD 和去重逻辑

#### 10. 无 `.gitignore`

项目目录中存在 `__pycache__/`、`.egg-info/`、`.pytest_cache/` 等生成物，且项目未初始化 git 仓库。

#### 11. 无日志

除 `tasks.py` 外，engine 和 service 层没有日志输出。异常检测的决策过程（命中/跳过/去重）对排查问题至关重要，应添加结构化日志。

#### 12. API 无认证

所有端点公开暴露，无认证/鉴权。对于内部服务可接受，但应在部署文档中注明。

---

## 三、测试评估

| 模块 | 覆盖情况 | 评价 |
|------|---------|------|
| `engine.py` | 工具函数、三种检测场景、边界值 | **良好** — 阈值边界测试到位 |
| `service.py` | 去重、批量加载、日期逻辑、enrich 保留 | **良好** — 关键业务流程覆盖 |
| `job_dates.py` | Celery 日期解析 | **良好** |
| `repository.py` | 通过 service 测试间接覆盖 | **一般** |
| `db.py` | 无 | **缺失** |
| `main.py` | 无 | **缺失** |
| `tasks.py` | 无 | **缺失** |
| `warning_keys.py` | 通过 service 测试间接覆盖 | **一般** |

---

## 四、总结

这是一个设计合理、实现清晰的小型业务系统。核心算法（EWMA + IQR + 双因子置信度）实现可靠，去重机制考虑周全，代码风格一致。主要改进方向是：

1. 修复 `Order` frozen 矛盾和模块级初始化问题（稳定性）
2. 补充 API/DB 层测试（可靠性）
3. 添加日志和 status 索引（可运维性）
4. 抽取公共工厂函数消除重复（可维护性）
