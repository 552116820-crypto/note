# Review by Codex 3

日期: 2026-04-10

项目路径: `/mnt/e/churn-warning-engine`

## 本次会话目标

本次会话围绕 `churn-warning-engine` 项目做了三件事:

1. 阅读并核验 Claude 4 的 review 报告。
2. 修复 Claude 报告中确认成立的问题。
3. Codex 再次复审当前代码，继续修复新发现的状态机、并发、配置和 MySQL 升级风险。

最终验证结果:

```bash
pytest -q
# 56 passed in 0.43s

python3 -m py_compile app/*.py tests/*.py
# passed
```

---

## 对 Claude 4 报告的判断

Claude 4 报告总体正确，没有发现明显误报。

确认成立的问题:

1. `Order` 使用 `frozen=True`，但 `items` 字段是可变 list，语义冲突。
2. MySQL `list_pending_warnings` 缺少 `ORDER BY`，返回顺序不稳定。
3. `InMemoryRepository` 的去重锁原来为空实现，开发/演示环境存在并发竞态。
4. `detect_product_anomaly` 重复过滤订单是防御性冗余，不是 bug，保留即可。

结论:

- Claude 报告中的第 1、2 点应修复。
- 第 3 点可顺手修。
- 第 4 点无需修改。

---

## 第一轮修复

### 1. 修复 `Order` frozen 语义

文件: `app/models.py`

修改:

- `Order.items` 从 `list[OrderItem]` 改为 `tuple[OrderItem, ...]`。
- 在 `__post_init__` 中将传入的 list 自动规范化为 tuple。

目的:

- 保持 `Order(frozen=True)` 的不可变语义。
- 兼容现有测试和调用方继续传 list 的写法。

### 2. 修复 MySQL 构造订单时修改 frozen 对象的问题

文件: `app/db.py`

修改:

- MySQL JOIN 查询结果不再创建 `Order(items=[])` 后 append。
- 改为先在中间 dict 中收集订单明细，再一次性构造 `Order`。

目的:

- 避免对 frozen dataclass 的内部可变字段做原地修改。

### 3. 修复 pending warning 排序不稳定

文件:

- `app/db.py`
- `app/repository.py`

修改:

- MySQL `list_pending_warnings` 增加:

```sql
ORDER BY biz_date DESC, warning_id ASC
```

- 内存仓储也按同样规则排序。

目的:

- 让 MySQL 与 InMemory 行为一致。
- 避免不同数据库执行计划导致 pending 列表顺序不确定。

### 4. 修复 InMemory 去重锁为空实现

文件: `app/repository.py`

修改:

- `InMemoryRepository` 增加 `RLock`。
- 覆写 `warning_dedup_lock()`，让 service 层已有锁逻辑在内存仓储下也生效。

---

## Codex 再次复审发现的问题

第一轮修复后，Codex 再次审查当前代码，发现以下问题。

### 1. resolved warning 可以被 enrich 重新激活

严重程度: 中

文件: `app/service.py`

问题:

- `resolve_warning()` 会把 warning 状态设为 `resolved`。
- 后续如果再调用 `enrich_warning()`，原逻辑会无条件把它改成 `enriched`。

影响:

- 破坏“已关闭预警不可再激活”的状态语义。
- 在内存仓储下可能出现同一个客户、同一类预警、同一个 dedup key 同时存在两个活跃 warning。
- 在 MySQL 下可能触发 `uq_warning_active_dedup` 唯一约束冲突。

复现流程:

1. 检测生成 warning A。
2. resolve warning A。
3. 再次检测生成 warning B。
4. 对 warning A 调 enrich。
5. 结果 warning A 重新变成 enriched，warning B 仍是 pending_enrich。

### 2. enrich/resolve 未使用 dedup 锁

严重程度: 中

文件: `app/service.py`

问题:

- 检测任务保存/刷新 warning 时会使用 `warning_dedup_lock`。
- 但人工 `enrich_warning()` 和 `resolve_warning()` 原来没有使用同一把锁。

影响:

- 并发场景下，检测任务可能覆盖人工 enrich 的内容。
- 例如检测任务先读到 pending warning，销售随后 enrich，检测任务再用 stale 数据刷新回 pending。

### 3. MySQL 旧库升级缺少迁移/回填路径

严重程度: 中低

文件: `app/db.py`

问题:

- `Base.metadata.create_all()` 只会创建不存在的表，不会修改已有表。
- 如果生产库早于 `dedup_key` / `active_slot` 字段和唯一约束，启动时不会自动补齐。

影响:

- 旧活跃 warning 的 `active_slot` 可能为空。
- 依赖 `active_slot == "1"` 查询时可能查不到旧 warning，导致重复预警。

### 4. 配置缺少边界校验

严重程度: 低到中

文件: `app/config.py`

问题:

- `DEFAULT_BATCH_SIZE=0` 时，`range(..., size)` 会崩溃。
- 阈值类环境变量没有范围校验。

影响:

- API 层的 `batch_size` 有 Pydantic 校验，但 Celery/内部调用使用的是 `Settings.default_batch_size`，可被环境变量绕过。

---

## 第二轮修复

### 1. 禁止 resolved warning 再 enrich

文件:

- `app/service.py`
- `app/main.py`

修改:

- 新增 `InvalidWarningStateError`。
- `enrich_warning()` 在锁内重新读取 warning。
- 如果当前状态是 `resolved`，抛出 `InvalidWarningStateError`。
- API 将该错误映射为 HTTP `409 Conflict`。

行为:

```text
POST /api/warnings/{warning_id}/enrich
```

如果 warning 已 resolved，返回:

```json
{
  "detail": "resolved warning cannot be enriched"
}
```

状态码: `409`

### 2. enrich/resolve 使用同一把 dedup 锁

文件: `app/service.py`

修改:

- 新增 `_dedup_parts()`，统一从 warning 中提取:
  - `product_id`
  - `direction`
  - `dedup_key`
- `detect_batch()`、`enrich_warning()`、`resolve_warning()` 都使用同一套 dedup 信息。
- `enrich_warning()` 和 `resolve_warning()` 都进入 `repository.warning_dedup_lock()`。
- 进入锁后重新读取 warning，避免 stale-write。

效果:

- 检测刷新、人工 enrich、人工 resolve 对同一 warning key 串行化。
- 降低人工处理内容被检测任务覆盖的风险。

### 3. 配置边界校验

文件: `app/config.py`

修改:

- `Settings.__post_init__()` 增加校验:
  - `min_order_count >= 2`
  - `frequency_multiplier > 0`
  - `volume_drop_threshold` 在 `(0, 1]`
  - `volume_rise_threshold > 0`
  - `product_multiplier > 0`
  - `key_product_threshold` 在 `(0, 1]`
  - `analysis_window_days > 0`
  - `default_batch_size > 0`
  - `max_pending_enrich >= 0`
  - `completed_status_codes` 非空

效果:

- 非法环境变量会在配置加载阶段失败。
- 避免运行到 service 层才出现难定位的异常。

### 4. 降低 MySQL 旧库 active_slot 回填风险

文件:

- `app/db.py`
- `migrations/mysql/001_warning_records_active_dedup.sql`

修改:

- `find_active_warning()` 不再依赖 `active_slot == "1"` 查询活跃 warning，而是以 `status in ACTIVE_WARNING_STATUSES` 为准。
- 新增手动迁移 SQL:

```text
migrations/mysql/001_warning_records_active_dedup.sql
```

迁移内容:

- 添加 `dedup_key` 字段。
- 添加 `active_slot` 字段。
- 回填活跃状态:
  - `pending_enrich`
  - `enriched`
- 创建 `ix_warning_dedup_lookup` 索引。
- 创建 `uq_warning_active_dedup` 唯一约束。

说明:

- 该迁移文件是手动迁移脚本，未自动执行。
- 原因是不同 MySQL 版本对 `ALTER TABLE ... IF NOT EXISTS` 支持不完全，生产库升级应先审阅现有表结构。

---

## 新增测试

### `tests/test_models.py`

覆盖:

- `Order.items` 会被规范化为 tuple。
- `Order.items` 不再有 `append`。

### `tests/test_service.py`

新增/扩展覆盖:

- pending warning 按 `biz_date DESC, warning_id ASC` 排序。
- resolved warning 不能再 enrich。
- enrich/resolve 都会使用 dedup lock。

### `tests/test_api_and_factory.py`

新增覆盖:

- 对 resolved warning 调 enrich API 返回 `409`。

### `tests/test_config.py`

新增覆盖:

- `default_batch_size=0` 被拒绝。
- `key_product_threshold=1.5` 被拒绝。
- `load_settings()` 会校验环境变量。

---

## 最终验证

执行:

```bash
pytest -q
```

结果:

```text
56 passed in 0.43s
```

执行:

```bash
python3 -m py_compile app/*.py tests/*.py
```

结果:

```text
通过
```

说明:

- 当前环境没有真实 MySQL 服务和 SQLAlchemy 运行依赖，因此没有执行 MySQL 集成测试。
- MySQL 相关风险通过代码路径调整和迁移 SQL 降低，但生产环境仍建议在真实 MySQL 上补一轮集成验证。

---

## 本次修改文件清单

运行时代码:

- `app/models.py`
- `app/repository.py`
- `app/db.py`
- `app/service.py`
- `app/main.py`
- `app/config.py`

测试:

- `tests/test_models.py`
- `tests/test_service.py`
- `tests/test_api_and_factory.py`
- `tests/test_config.py`

迁移:

- `migrations/mysql/001_warning_records_active_dedup.sql`

---

## 当前结论

本次会话没有发现阻断级或高优先级问题。

已完成:

- 修复 Claude 4 报告中确认成立的中优先级问题。
- 修复 Codex 再次复审发现的 warning 状态机问题。
- 修复人工 enrich/resolve 与检测刷新之间的锁一致性问题。
- 增加配置边界校验。
- 降低 MySQL 旧库 active_slot 未回填带来的重复预警风险。
- 补充对应测试，测试总数从 50 增加到 56。

剩余建议:

1. 如计划启用 MySQL，执行迁移前先审阅现有 `warning_records` 表结构。
2. 在真实 MySQL 环境补充集成测试，重点验证:
   - active warning 去重
   - resolved 后允许新 warning
   - resolved warning enrich 返回冲突
   - 唯一约束行为
3. 如果后续引入更多 warning 状态，建议把状态机显式抽成单独模块，避免状态转换散落在 service 方法里。
