# Churn Warning Engine — Codex 会话记录（二）

> 会话日期: 2026-04-10  
> 工作目录: `/mnt/e/churn-warning-engine`  
> 说明: 以下记录保存本次会话中用户与 Codex 的可见交互内容、关键判断、代码修改和验证结果。系统/开发者隐藏指令未写入。

---

## 一、用户提供的仓库说明

用户提供了 `/mnt/e/churn-warning-engine` 的 `AGENTS.md` 指令，主要内容如下：

### 项目结构与模块组织

`app/` 存放运行时代码。`app/engine.py` 负责异常检测逻辑，`app/service.py` 负责任务编排，`app/repository.py` 提供内存仓储，`app/db.py` 提供可选的 MySQL 仓储，`app/main.py` 是 FastAPI 入口，`app/tasks.py` 定义 Celery 定时任务。`app/config.py` 统一管理环境变量驱动的配置。`tests/` 存放 pytest 测试。`churn_warning_engine.egg-info/` 和 `__pycache__/` 属于生成产物，不应作为源码修改。

### 构建、测试与开发命令

`pip install -e .[dev]` 安装项目及本地开发所需的 pytest。  
`pip install -e .[all]` 额外安装 MySQL 和 Celery 可选依赖。  
`uvicorn app.main:app --reload` 在本地启动 API 服务。  
`pytest -q` 运行测试套件。  
`celery -A app.tasks worker --loglevel=info --beat` 启动定时检测任务。  
示例：`REPOSITORY_TYPE=mysql DATABASE_URL=mysql+pymysql://user:pass@host/db uvicorn app.main:app --reload`。

### 编码风格与命名约定

目标运行环境为 Python 3.10+，遵循现有 PEP 8 风格：4 空格缩进、显式类型标注、函数职责单一。模块、函数、变量使用 `snake_case`，类名使用 `PascalCase`，环境变量使用 `UPPER_SNAKE_CASE`。导入应分组清晰，注释保持简洁。新增逻辑应沿用现有的 service/repository 分层，不要直接把业务逻辑塞进 API 处理函数。

### 测试指南

使用 `pytest`，测试文件命名为 `tests/test_*.py`，测试类使用 `Test*`，测试方法使用 `test_*`。每次变更行为时都应补充测试，尤其是阈值边界、去重逻辑和不同仓储实现路径。目前没有覆盖率门禁，因此提交前应确保 `app/engine.py`、`app/service.py` 和 `app/db.py` 的关键分支有足够测试。

### 提交与合并请求规范

当前工作区快照不包含 `.git` 历史，因此无法验证仓库特有的提交规范。建议使用简短的祈使句标题，例如 `Add product anomaly regression test`，并保持单次提交只处理一个主题。PR 需要说明行为变化、列出新增或修改的环境变量、关联对应 issue；如果 API 契约发生变化，还应附上示例请求或响应。

### 安全与配置提示

新增配置项时同步更新 `app/config.py`。`DATABASE_URL`、`CELERY_BROKER_URL`、`CELERY_RESULT_BACKEND` 等敏感信息不得提交到版本库。只有在已安装可选 MySQL 依赖且数据库表结构可用时，才应启用 `REPOSITORY_TYPE=mysql`。

---

## 二、用户让 Codex 评价 Claude 第二轮 Review

用户：

> `E:\note\客户流失预警\review by claude 2.md` 我让 claude 进行了第三轮的 review，你觉得它说的怎么样

Codex 先读取了 Claude 的报告、当前代码和测试。关键核实结果：

- Claude 报告中的 “`job_dates.py` 是死代码，`tasks.py` 没接入” 在当前工作区不成立。
- 当前 `app/tasks.py` 已经 import `resolve_run_detection_date`，并在 `run_detection` 中调用它。
- 测试当时通过：`34 passed`。

Codex 的结论：

> 我看了 Claude 的 review，也对照了当前代码并跑了测试：`pytest -q` 结果是 `34 passed`。
>
> 总体判断：**Claude 的大方向不错，但第 1 条在当前工作区已经不成立；真正最值得修的是第 2 条。其余多数是维护性、语义一致性或性能建议，不是立刻会导致错误结果的 bug。**

逐条判断如下：

1. `job_dates.py` 死代码：**当前不成立 / 过时了**  
   当前 `app/tasks.py` 已经 import 了 `resolve_run_detection_date`，并且已经调用它。  
   所以 Claude 说的 `biz_date = date.fromisoformat(...) if biz_date_str else None` 已经不是当前代码。这个问题如果之前存在，现在已经修掉了。

2. Volume 部分月比较：**成立，而且是最值得处理的一条**  
   `app/engine.py` 确实是“本月截至 biz_date”对比“完整上月”。  
   例如 `biz_date=2026-04-07` 时，就是 4 月 1-7 日对比 3 月整月，容易误报下降。  
   Codex 建议修在 service/API 层：`volume` 类型要么默认使用上一个完整月月末，要么拒绝非月末 `biz_date`。这比在 engine 里做日均折算更稳，因为月度订单通常不是均匀分布的，日均会引入新的误差。

3. `db.py` ORM / Raw SQL 混用：**成立，但严重度偏高**  
   现在能跑，不是 correctness bug。  
   但 `_warning_from_row` 目前按 raw SQL 列名读，而保存/更新走 ORM 字段别名，确实有维护成本。可以后续统一成 ORM，但不应排在部分月问题前面。

4. Volume `direction` 不在 `metadata`：**功能没坏，建议可采纳**  
   当前去重从 `feature_summary["direction"]` 读取，InMemory 和 MySQL 也按这个逻辑找。  
   这能工作。只是产品预警用 `metadata.product_id`，volume 用 `feature_summary.direction`，语义不统一。  
   如果改，建议 `metadata={"direction": direction}`，但查找时保留 fallback：先读 `metadata`，没有再读 `feature_summary`，避免旧数据失效。

5. `Order` frozen 但 `items` 可变：**成立，低优先级**  
   `Order` 是 frozen，但 `db.py` 又 append `items`。这是 Python frozen dataclass 的常见语义陷阱。  
   最简单的处理是去掉 `Order` 的 `frozen=True`。如果想要真正不可变，就把 `items` 改成 tuple，但那会牵涉更多构建代码。

6. Python 版本上限 `<3.13`：**可以讨论，不算 bug**  
   如果没有 CI 覆盖 Python 3.13+，贸然放宽也不是纯收益。  
   建议等 CI 或本地至少跑过 3.13 后，再改成 `>=3.10` 或更明确的上限。

7. MySQL 缺少复合索引：**成立，性能建议**  
   `find_active_warning` 查询适合加 `(customer_id, anomaly_type, status)` 复合索引，甚至可以考虑把 `biz_date` 也放进去辅助排序。  
   但注意 `Base.metadata.create_all()` 不会自动给已有表补索引，生产库需要 migration 或手工 DDL。

8. `predict_cycle` 的 `> 0` 过滤：**成立但很小**  
   已经 `sorted(set(order_dates))`，所以 `> 0` 基本永远为真。删掉可以更干净，但不影响行为。

Codex 给出的优先级建议：

1. 先修 **volume 部分月问题**，并补 service/API 测试。
2. 顺手把 volume `direction` 放进 `metadata`，查找逻辑保留兼容 fallback。
3. 加 MySQL 复合索引，但生产要配 migration。
4. 再处理 ORM/raw 统一和 `Order.frozen` 这种维护性问题。
5. Python 版本上限和 `> 0` 死代码可以最后顺手做。

结论：

> **Claude 第 2 条很有价值，第 1 条已经过时，其余建议多数合理但优先级不应太高。**

---

## 三、用户要求处理 Claude 第 2 条：Volume 部分月比较

用户：

> 那就帮我把第2条处理了

Codex 执行思路：

> 我会把约束放到 `WarningService.start_job`，这样 API、Celery、测试直接调用 service 都走同一条规则。具体做法是：volume 未传日期时默认上一个完整月月末；传了日期时必须是月末，否则抛 400 级别的 `ValueError`。
>
> 我准备做两层保护：`start_job` 先拒绝/规范化 volume 日期，避免保存出部分月 job；`detect_volume_anomaly` 再加一次防御性校验，防止有人绕过 service 直接调用 engine。然后更新现有 volume 单测到月末日期。

实际修改：

- 新增 `app/date_utils.py`，提供 `previous_month_end()` 和 `is_month_end()`。
- 修改 `app/job_dates.py`，复用 `previous_month_end()`。
- 修改 `app/service.py`：
  - `volume` 未传 `biz_date` 时使用上一个完整月月末。
  - `volume` 显式传入 `biz_date` 时必须是月末，否则抛 `ValueError`。
- 修改 `app/engine.py`：
  - `detect_volume_anomaly()` 增加防御性校验，拒绝非月末日期。
- 修改 `app/main.py`：
  - `start_job` 捕获 `ValueError` 并返回 HTTP 400。
- 修改测试：
  - volume engine 测试改用 `2026-04-30`。
  - 增加部分月拒绝测试。
  - 增加 service 默认月末、拒绝部分月、接受月末测试。

验证结果：

```bash
pytest -q
38 passed in 0.21s
```

Codex 回复：

> 已处理第 2 条。
>
> 改动点：
>
> - 新增 `app/date_utils.py`，统一提供 `previous_month_end` 和 `is_month_end`。
> - `app/service.py` 现在对 `volume` job 做日期规则：
>   - 不传 `biz_date` 时，默认使用上一个完整自然月月末。
>   - 显式传 `biz_date` 时，必须是月末，否则抛 `ValueError`。
> - `app/engine.py` 加了防御性校验，避免有人绕过 service 直接用部分月日期调用 `detect_volume_anomaly`。
> - `app/main.py` 把 `start_job` 的 `ValueError` 转成 HTTP 400，API 不会变成 500。
> - 更新了 volume 相关测试，原来用 `2026-04-07` 的 engine 单测改为月末 `2026-04-30`，并新增部分月拒绝测试和 service 默认月末测试。
>
> 验证结果：
>
> ```bash
> pytest -q
> 38 passed in 0.21s
> ```

---

## 四、用户要求 Codex 再次 Review 当前项目

用户：

> 你再次review这个项目的代码，看看还有哪些地方需要改进

Codex 执行：

- 重新阅读 `app/db.py`、`app/service.py`、`app/engine.py`、`app/repository.py`、`app/main.py`、`app/config.py`、`app/schemas.py`、`app/models.py`、`app/tasks.py`、测试文件等。
- 运行测试，结果通过：

```bash
pytest -q
38 passed in 0.17s
```

Codex 的 Review 结论：

### Findings

- **High**: enriched 预警会永久压住后续同类预警。  
  `app/service.py` 遇到 `pending_enrich` 会刷新，但遇到 `enriched` 只复用旧 `warning_id`，不更新 `feature_summary`、`severity`、`biz_date`。同时 InMemory 和 MySQL 都把 `enriched` 当 active。当前 API 只有 enrich，没有 close/resolve。结果是：销售一旦 enrich 某个客户的 frequency/product/volume 预警，未来几个月同一 dedup key 的真实新异常都会被旧记录吞掉。建议增加 `closed/resolved` 生命周期，或者 dedup 只复用“当前业务周期内”的 active warning。

- **Medium**: MySQL 仓储会无条件截断订单历史，且行为和 InMemory 不一致。  
  `app/db.py` 用 `analysis_window_days` 算 `since`，然后过滤所有 job 类型的订单。InMemory 只按客户和 `biz_date` 过滤，不裁剪下界。这个差异会让 MySQL 下的 frequency 周期预测只看最近窗口，和测试路径不同；如果 `ANALYSIS_WINDOW_DAYS` 被调小，volume 甚至可能拿不到完整上月数据。建议把查询窗口从 service 按 job_type 明确传入，或者 repository 不做全局下界过滤，让 engine 自己负责各场景窗口。

- **Medium**: 去重依赖应用层逻辑，没有数据库级唯一约束。  
  service 先查再写，MySQL 侧虽然有 `GET_LOCK`，但没有唯一索引兜底；外部写入、历史脏数据、锁逻辑变更都可能产生重复 active warning。产品维度和 volume 方向还藏在 JSON 里，需要把候选行全部取出再 Python 过滤。建议增加显式 `dedup_key` 列，例如 `product:P1`、`volume:down`、`frequency`，并加 `(customer_id, anomaly_type, dedup_key, status)` 索引或唯一约束策略。

- **Low / Medium**: MySQL 的 warning 读写混用 ORM 和 raw SQL，后续改字段容易漏。  
  保存和更新走 ORM 属性名，读取走 raw SQL 列名，转换函数也只适配 raw SQL row。现在能工作，但维护成本偏高。建议统一成 ORM 读写，或者让转换函数同时兼容 ORM row 和 SQL row。

- **Low**: pending enrich 队列没有稳定排序，也缺少时间字段。  
  InMemory 依赖 dict 插入顺序；MySQL 没有 `ORDER BY`。当 pending 数量超过 limit 时，销售侧拿到哪批预警不可预测。建议给 `WarningRecord` 增加 `created_at` / `updated_at`，并按明确规则排序，比如最早创建优先或最新业务日期优先。

- **Low**: Celery 显式 volume 日期的校验分散，测试语义已经有点过时。  
  `app/job_dates.py` 会直接接受 `"2026-05-01"`，但 service 会拒绝非月末日期。当前行为没错，但测试名 “takes precedence” 容易让人误以为显式非月末也合法。建议 job_dates 测试改成月末日期，并补一个任务级 invalid volume date 测试。

- **Low**: `Order` 是 frozen dataclass，但 `items` 是可变 list。  
  `Order` 标了 `frozen=True`，但 `db.py` 又依赖 `items.append(...)`。这不是运行错误，但语义不干净。要么去掉 `Order` 的 `frozen=True`，要么把构建逻辑改成先收集明细再一次性构造不可变 tuple。

- **Very Low**: `predict_cycle` 里还有一段死过滤。  
  已经 `sorted(set(order_dates))`，所以 `if (cur - prev).days > 0` 永远为真。删掉能减少阅读噪音。

Codex 的优先级建议：

> 先处理 warning 生命周期，其次处理 MySQL 订单窗口一致性，再做 dedup_key 数据库约束。其它项可以作为维护性清理排后面。

---

## 五、用户要求处理三个重点：生命周期、MySQL 窗口、dedup_key

用户：

> 你帮我，先处理 warning 生命周期，其次处理 MySQL 订单窗口一致性，再做 dedup_key 数据库约束

Codex 执行思路：

> 我会把这次改成一个小的产品级闭环：先给 warning 增加可关闭/解决的生命周期，再把 MySQL 订单查询窗口改成由 job 类型控制，最后把去重键落成显式字段，避免 JSON 扫描和并发脏数据。

任务计划：

1. 梳理需要变更的数据模型、仓储接口和 API 面。
2. 实现 warning 生命周期和测试。
3. 实现按 job_type 的 MySQL 订单窗口一致性。
4. 实现 dedup_key 字段、索引和兼容逻辑。
5. 跑完整测试并修正回归。

主要修改：

### 1. 新增 `app/warning_keys.py`

集中定义：

- `ACTIVE_WARNING_STATUSES = ("pending_enrich", "enriched")`
- `RESOLVED_WARNING_STATUS = "resolved"`
- `ACTIVE_WARNING_SLOT = "1"`
- `make_warning_dedup_key()`
- `warning_dedup_key()`
- `active_slot_for_status()`

dedup key 规则：

- frequency: `frequency`
- volume: `volume:up` / `volume:down`
- product: `product:{product_id}`

### 2. 修改模型

`WarningRecord` 增加字段：

```python
dedup_key: str = ""
```

### 3. 修改 engine

- `detect_frequency_anomaly()` 生成 `dedup_key="frequency"`。
- `detect_volume_anomaly()` 生成 `dedup_key="volume:{direction}"`，并把 `direction` 同步写入 `metadata`。
- `detect_product_anomaly()` 生成 `dedup_key="product:{product_id}"`。

### 4. 修改 service

- 增加 `_orders_since_date()`，按 job 类型决定订单窗口：
  - `frequency`: `None`，不截断历史。
  - `product`: `biz_date - analysis_window_days`。
  - `volume`: 上月月初。
- `detect_batch()` 调用 repository 批量查订单时传 `since_date`。
- 去重时统一计算 `dedup_key` 并传给 repository。
- 新增 `resolve_warning()`：
  - 把 warning 状态更新为 `resolved`。
  - `resolved` 不再参与 active warning 去重。

### 5. 修改 repository

- `get_orders_for_customer()` 和 `get_orders_for_customers()` 增加 `since_date` 参数。
- InMemory 支持 `since_date` 过滤。
- InMemory `save_warning()` 在缺失 `dedup_key` 时自动补齐。
- InMemory `find_active_warning()` 用 `warning_dedup_key()` 对比，而不是分别读 JSON 字段。

### 6. 修改 MySQL repository

- `WarningRecordRow` 增加：
  - `dedup_key = Column(String(120), nullable=False, default="")`
  - `active_slot = Column(String(1), nullable=True, default="1")`
- 增加索引和唯一约束：

```python
Index(
    "ix_warning_dedup_lookup",
    "customer_id",
    "anomaly_type",
    "dedup_key",
    "active_slot",
    "biz_date",
)

UniqueConstraint(
    "customer_id",
    "anomaly_type",
    "dedup_key",
    "active_slot",
    name="uq_warning_active_dedup",
)
```

- `find_active_warning()` 改用 ORM 查询和 `dedup_key`。
- MySQL `GET_LOCK` 改成基于 dedup key 的短 hash，避免锁名过长。
- 保留 legacy fallback：如果按新 `dedup_key` 找不到，则扫描 `dedup_key=""` 的旧 active 记录并重算 key。

### 7. 修改 API

新增：

```http
POST /api/warnings/{warning_id}/resolve
```

用于把 warning 标记为 `resolved`。

### 8. 测试修改

新增/更新测试覆盖：

- resolved 后同类 warning 能重新生成。
- enrich 后人工内容仍被保护。
- frequency 不截断历史。
- product 使用 analysis window。
- volume 使用上月月初作为订单下界。
- warning 保存后有 `dedup_key`。
- volume warning 同时有 `metadata["direction"]` 和 `dedup_key`。
- `test_job_dates` 中显式 volume 日期改成月末。

中间测试失败一次：

```text
FAILED tests/test_engine.py::TestDedup::test_volume_direction_kept_separate
AssertionError: dedup_key: 'volume:up' != ''
```

原因：现在仓储保存时会补齐 `dedup_key`，返回对象比原始测试对象多了字段。Codex 将断言改为检查同一个 warning id 和补齐后的 key。

最终验证：

```bash
pytest -q
43 passed in 0.17s

python3 -m compileall -q app tests
```

Codex 回复：

> 已按你指定的三个优先级处理完。
>
> **已改内容**
>
> - Warning 生命周期：
>   - 新增 `resolved` 状态逻辑，`resolved` 不再参与 active warning 去重。
>   - 新增 `WarningService.resolve_warning()`。
>   - 新增 API：`POST /api/warnings/{warning_id}/resolve`。
>   - 已 enrich 的 warning 仍会保护人工内容；只有 resolve 后，同类新异常才会重新生成新 warning。
>
> - MySQL 订单窗口一致性：
>   - 仓储接口增加 `since_date` 参数。
>   - service 按 job 类型决定订单窗口：
>     - `frequency`: 不截断历史，保留完整周期样本。
>     - `product`: 使用 `analysis_window_days`。
>     - `volume`: 只取上月月初到当前月末所需窗口。
>   - MySQL 不再无条件用 `analysis_window_days` 截断所有检测类型。
>
> - `dedup_key` 数据库约束：
>   - `WarningRecord` 增加 `dedup_key` 字段。
>   - 新增 `warning_keys.py`，统一生成 dedup key 和 active slot。
>   - engine 现在直接生成明确的 dedup key：
>     - frequency: `frequency`
>     - volume: `volume:up` / `volume:down`
>     - product: `product:{product_id}`
>   - MySQL 表模型增加 `dedup_key`、`active_slot`、复合索引和唯一约束 `uq_warning_active_dedup`。
>
> **测试**
>
> 已补充生命周期、订单窗口、dedup key 的测试。验证通过：
>
> ```bash
> pytest -q
> 43 passed in 0.17s
>
> python3 -m compileall -q app tests
> ```
>
> **部署注意**
>
> MySQL 现有表需要迁移；`create_all()` 不会给已有表自动补列和约束。上线前需要先为 `warning_records` 增加 `dedup_key`、`active_slot`，回填现有 active warning 的 dedup key，再创建唯一约束和索引，否则已有库会缺列或建唯一约束失败。

---

## 六、用户让 Codex 过目 Claude 第三份 Review

用户：

> 我让claude再次review这个项目代码，`E:\note\客户流失预警\review by claude 3.md` 这是claude生成的代码审查报告，请你过目

Codex 读取了 Claude 报告。报告主要问题：

1. `Order` frozen dataclass 的可变性矛盾。
2. 模块级别副作用初始化。
3. Celery task 每次执行都重建连接池。
4. `batch_id` 解析不够健壮。
5. 缺少 `warning_records.status` 索引。
6. `_serialize` 手动映射字段。
7. Legacy dedup fallback 增加复杂度。
8. `get_orders_for_customers` 基类默认实现是 N+1。
9. 缺少 API 层测试。
10. 无 `.gitignore`。
11. 无日志。
12. API 无认证。

Codex 运行测试：

```bash
pytest -q
43 passed in 0.20s
```

Codex 结论：

> 我过了一遍 Claude 这份报告，也对照了当前代码和测试。结论：**整体判断靠谱，但严重度排序需要调整。当前最该优先处理的是模块级初始化和 Celery service 重建，不是 `Order frozen`。**

认同的重点：

- **模块级初始化确实是高优先级问题。**  
  `app.main` 在 import 时加载配置并实例化 repository。`create_engine()` 本身不一定马上连库，但 `Base.metadata.create_all()` 会触发数据库连接和 DDL 检查。  
  这会导致 `REPOSITORY_TYPE=mysql` 时，只要数据库不可用，`import app.main` 就可能直接失败。这个问题会影响 API 测试、CI、启动稳定性，应该排第一。

- **Celery 每次 task 重建 service/engine 也成立。**  
  每次执行都会 `_build_service()`，MySQL 模式下会创建新 SQLAlchemy engine/sessionmaker。任务频率目前不高，所以不一定马上炸，但设计上不理想。建议把 repository/service 工厂抽成公共函数，并在 Celery worker 进程内缓存。

- **`Order frozen` 问题成立，但 Codex 不认为是严重。**  
  `Order` 是 frozen，但 `db.py` 又 append。语义确实别扭，但目前只在构造订单聚合结果时变更，风险比启动/连接池问题低。最简单修法是去掉 `Order` 的 `frozen=True`。

- **`status` 索引问题成立。**  
  `list_pending_warnings` 用 `WHERE status = 'pending_enrich' LIMIT :lim`，但当前复合索引是 dedup 查询用的，不覆盖这个 pending 查询。建议加 `Index("ix_warning_pending", "status", "biz_date")`，更好是同时补 `created_at/updated_at` 后按明确顺序取。

- **API 层测试缺失成立。**  
  当前测试主要覆盖 engine/service/job_dates，确实没有 FastAPI TestClient，也没有 MySQL repository 和 Celery task 测试。尤其刚加了 `/resolve`，最好补 API 级测试验证 400/404/status 流程。

需要降级看待的点：

- `batch_id.split(":")` 改 `rsplit(":", 1")`：合理但低优先级。当前 job_id 来自 `uuid4()`，不会包含冒号。严格 split 现在没实际 bug。
- `_serialize` 手动映射：不完全是坏事。它也避免把内部字段如 `dedup_key`、`metadata` 直接暴露出去。不能简单说应该全量自动映射，除非先明确 API 是否要暴露这些字段。
- legacy dedup fallback：这是刚引入 `dedup_key` 后的迁移兼容代码。短期合理。真正要注意的是上线前必须做数据迁移。
- Repository 默认批量查询 N+1：可以改，但不是当前风险点。两个现有实现都已经覆盖了批量方法。

Codex 建议顺序：

1. 抽 `app/service_factory.py`，统一 main 和 tasks 的 repository/service 构建。
2. FastAPI 改 lifespan 或依赖注入，避免 import 时连库。
3. Celery 用 worker 进程内缓存的 service，避免每次 task 建 engine。
4. 补 API TestClient 测试，覆盖 start/detect/enrich/resolve 和 400/404。
5. 给 MySQL pending 查询加索引和稳定排序，同时准备迁移脚本。
6. 最后做 `Order frozen`、`batch_id.rsplit`、Repository abstract、日志等清理项。

结论：

> 这份报告可以采纳，但我会把优先级改成：**模块初始化/工厂复用 > Celery 缓存 > API/DB 测试和迁移 > Order frozen 语义清理**。

---

## 七、用户要求处理高优先级问题

用户：

> 把你认为的重点，高优先级问题给处理了

Codex 执行思路：

> 我会优先处理我认为真正影响启动和运行稳定性的两件事：抽公共 service 工厂，避免 `app.main` import 时初始化 MySQL；同时让 Celery worker 缓存 service，避免每次任务创建新的 SQLAlchemy engine。然后补对应测试，确保 import 和复用行为不会回退。

任务计划：

1. 确认当前 main/tasks 构建路径和测试能力。
2. 抽取 service_factory 并改 main 为延迟初始化。
3. 改 Celery 使用进程内缓存的 service。
4. 补充测试覆盖延迟初始化和缓存复用。
5. 运行完整测试和编译检查。

### 1. 新增 `app/service_factory.py`

内容：

- `build_repository(settings)`：统一选择 InMemory 或 MySQL。
- `build_service(settings=None)`：统一构建 `WarningService`。
- `get_cached_service()`：使用 `lru_cache(maxsize=1)` 缓存 service。
- `clear_cached_service()`：测试清理缓存。

### 2. 修改 `app/main.py`

删除模块导入时的：

- `settings = load_settings()`
- 根据 `REPOSITORY_TYPE` 初始化 repository
- `service = WarningService(...)`

改为：

- 引入 FastAPI `Depends`
- 每个业务端点通过 `Depends(get_cached_service)` 获取 service

结果：

- `import app.main` 不再触发 MySQL 初始化。
- 数据库不可用时不会在 import 阶段直接失败。
- 测试可以通过 `app.dependency_overrides` 注入测试 service。

### 3. 修改 `app/tasks.py`

删除重复的 `_build_service()` 逻辑，改为：

```python
def _build_service() -> WarningService:
    """返回当前 worker 进程缓存的 WarningService。"""
    return get_cached_service()
```

结果：

- Celery worker 进程内复用同一个 service。
- MySQL 模式下不会每次 task 创建新的 SQLAlchemy engine/sessionmaker。

### 4. 新增 API/工厂测试

新增 `tests/test_api_and_factory.py`：

- `test_main_import_does_not_build_service`
  - monkeypatch `service_factory.build_service` 为抛异常函数。
  - 设置 `REPOSITORY_TYPE=mysql`。
  - import `app.main`。
  - 验证 import 阶段不会构建 service。

- `test_cached_service_reuses_instance`
  - monkeypatch `build_service`。
  - 连续调用 `get_cached_service()` 两次。
  - 验证返回同一对象，构建次数为 1。

- `test_tasks_build_service_uses_cached_service`
  - 当前环境没有安装 celery。
  - 用 fake Celery 模块模拟 `celery` 和 `celery.schedules`。
  - import `app.tasks`。
  - 验证 `tasks._build_service()` 返回 cached service。

- `test_start_volume_partial_month_returns_400`
  - 使用 TestClient 和 dependency override。
  - 验证 volume 非月末日期返回 HTTP 400。

- `test_resolve_warning_api_allows_new_warning`
  - 通过 API 完整跑 start → batches → detect → resolve → detect。
  - 验证 resolve 后同类新 warning 会重新生成。

执行过程中确认：

```bash
python3 - <<'PY'
try:
    import celery
    print('celery ok')
except Exception as exc:
    print(type(exc).__name__, exc)
PY
```

输出：

```text
ModuleNotFoundError No module named 'celery'
```

因此测试中使用 fake Celery，而不是强行要求安装可选依赖。

最终验证：

```bash
pytest -q
48 passed in 0.42s

python3 -m compileall -q app tests
```

Codex 回复：

> 已把我认为的高优先级问题处理完，重点是启动稳定性和 Celery 连接池复用。
>
> 改动如下：
>
> - 新增公共工厂 `service_factory.py`：
>   - `build_repository()` 统一处理 `REPOSITORY_TYPE`。
>   - `build_service()` 统一构建 `WarningService`。
>   - `get_cached_service()` 用 `lru_cache(maxsize=1)` 做进程内缓存。
>   - `clear_cached_service()` 方便测试清缓存。
>
> - 改造 `main.py`：
>   - 删除模块 import 时的 `load_settings()`、repository 初始化和 service 初始化。
>   - 所有业务端点改用 `Depends(get_cached_service)`。
>   - 现在 `import app.main` 不会触发 MySQL 初始化，也不会在数据库不可用时直接 import 崩溃。
>
> - 改造 `tasks.py`：
>   - 删除重复的 `_build_service()` 逻辑。
>   - Celery task 现在通过 `get_cached_service()` 复用当前 worker 进程里的 service/engine，不再每次任务新建 SQLAlchemy engine 和 session pool。
>
> - 新增 `test_api_and_factory.py`：
>   - 验证 `app.main` import 不会构建 service。
>   - 验证 cached service 只构建一次。
>   - 用 fake Celery 验证 `tasks._build_service()` 走缓存入口，不要求本地安装 celery。
>   - 补了两个 API 层测试：volume 部分月返回 400，以及 resolve 后同类新 warning 可以重新生成。
>
> 验证通过：
>
> ```bash
> pytest -q
> 48 passed in 0.42s
>
> python3 -m compileall -q app tests
> ```
>
> 这轮没有动 `Order frozen`、status 索引、legacy fallback 这些中低优先级项。当前最关键的 import 副作用和 Celery 重建连接池问题已经处理掉了。

---

## 八、本次会话最终状态

截至本记录写入前，项目当前关键状态：

- 测试数量从 34 增加到 48。
- 最新验证结果：

```bash
pytest -q
48 passed in 0.42s

python3 -m compileall -q app tests
```

本次会话新增或修改的核心文件包括：

- `app/date_utils.py`
- `app/warning_keys.py`
- `app/service_factory.py`
- `app/engine.py`
- `app/service.py`
- `app/repository.py`
- `app/db.py`
- `app/main.py`
- `app/tasks.py`
- `tests/test_engine.py`
- `tests/test_service.py`
- `tests/test_job_dates.py`
- `tests/test_api_and_factory.py`

仍未处理但已识别为后续可做的事项：

- `Order` frozen dataclass 与 mutable `items` 的语义清理。
- MySQL pending warning 查询增加 status/时间索引和稳定排序。
- MySQL 现有库 migration：新增 `dedup_key`、`active_slot`、回填数据、创建唯一约束。
- `batch_id` 可改为 `rsplit(":", 1)`。
- Repository 默认批量查询可改为 abstract。
- 增加更多 DB/Celery 集成测试。
- 增加结构化日志。
- 明确 API 认证或部署网关要求。
