# 客户流失预警系统 — Claude 会话记录

> 会话日期: 2026-04-09
> 内容: 两个项目的异常检测分析 → 对比评价 → 合并最佳版本 → Codex 点评 → 修复

---

## 一、两个项目的 Python 异常检测对比分析

两个项目都是**客户流失预警系统**，检测三种相同类型的异常，但架构和算法实现有明显差异。

### 1.1 `churn-warning-ai` — 分离式架构（预测 + 预警解耦）

**架构特点：** 预测和预警是两个独立的定时任务，通过数据库衔接。

```
[周日凌晨] batch_predict.py → 写入 churn_prediction 表
[周一早8点] warning_check.py → 读取预测结果 → 判断是否触发预警 → 推企微
```

#### 场景一：频率异常（下单间隔异常拉长）

**核心文件：** `src/services/cycle_predictor.py`

算法流程（4 步）：

| 步骤 | 方法 | 说明 |
|------|------|------|
| 1. 计算间隔 | `sorted_dates[i+1] - sorted_dates[i]` | 相邻下单日期之差（天数），过滤同天订单 |
| 2. **IQR 去异常值** | `_remove_outliers_iqr()` :89行 | Q1-1.5×IQR ~ Q3+1.5×IQR 范围外的间隔剔除（如春节长假） |
| 3. **EWMA 指数加权移动平均** | `_ewma()` :116行 | 权重 `2^i`（越近期权重越大），最近一次间隔权重是最早的 2^(n-1) 倍 |
| 4. 置信度 | `_calculate_confidence()` :134行 | 基于**变异系数 CV** = std/mean，置信度 = 1-CV，再乘以样本量修正系数 |

**预警触发（`warning_engine.py:18`）：**

```python
实际未下单天数 = today - last_order_date
阈值 = predicted_cycle_days × multiplier（可配置）
如果 实际天数 >= 阈值 → 触发预警
```

#### 场景二：量级异常（下单数量环比剧变）

**核心文件：** `src/services/volume_detector.py`

纯规则检测（非 ML）：

```python
change_rate = (当月量 - 上月量) / 上月量 × 100%

if change_rate <= -50%  → 异常(下降)   # VOLUME_DROP_THRESHOLD = 50%
if change_rate >= +50%  → 异常(上升)   # VOLUME_RISE_THRESHOLD = 50%
```

#### 场景三：产品异常（关键产品采购间隔异常）

**核心文件：** `src/services/product_analyzer.py`

1. **识别关键产品**：年度采购额占比 >= 30%（`KEY_PRODUCT_PCT`）
2. **对每个关键产品**复用 `predict_order_cycle()` 做独立的周期预测
3. 预警逻辑同场景一：实际未采购天数 >= 预测周期 × 倍数

**关键参数：**

| 参数 | 默认值 |
|------|--------|
| `MIN_ORDERS_FOR_PREDICTION` | 3 |
| `VOLUME_DROP_THRESHOLD` | 50% |
| `VOLUME_RISE_THRESHOLD` | 50% |
| `KEY_PRODUCT_PCT` | 30% |
| `LOW_SAMPLE_CONFIDENCE_CAP` | 0.6 |

---

### 1.2 `crm-warning-engine` — 一体化引擎架构

**架构特点：** 单个 `engine.py` 包含全部检测逻辑，通过 REST API（`service.py`）按批次调度，支持 Job→Batch→Detect→Enrich 流水线。

```
POST /jobs → start_job() → 按 sales_id 分批
POST /detect → detect_batch() → 调 engine.detect_*() → 生成 WarningRecord
POST /enrich → enrich_warning() → AI/人工丰富预警内容
```

#### 场景一：频率异常

**核心：** `engine.py:56` `predict_cycle()` + `engine.py:115` `detect_frequency_anomaly()`

| 步骤 | 差异点 |
|------|--------|
| IQR 去异常值 | `_remove_iqr_outliers()` :23行 — **自实现分位数**（不依赖 numpy） |
| 加权平均 | `_weighted_average()` :36行 — **线性递增权重** `[1,2,3,...,n]`（而非指数 2^i） |
| 置信度 | `_prediction_confidence()` :42行 — **双因子加权**：`0.45×样本分 + 0.55×稳定分`，范围锁定 [0.1, 0.95] |
| 触发条件 | `days_since <= predicted × 1.5`（`frequency_multiplier=1.5`） |
| **额外功能** | 自动计算 **severity**（high/medium/low）和**参数推荐**（建议的最优 multiplier） |

#### 场景二：量级异常

**核心：** `engine.py:163` `detect_volume_anomaly()`

```python
change_rate = (当月金额 - 上月金额) / 上月金额
if change_rate > -0.35 → 正常（不预警）  # volume_drop_threshold=0.35 即 35%
```

与 churn-warning-ai 的差异：
- 只检测**下降**，不检测上升
- 比较的是**金额**而非数量
- 阈值更敏感：**35%**（vs churn-warning-ai 的 50%）
- 有 severity 分级：下降>=60% 高危、>=40% 中危、其余低危

#### 场景三：产品异常

**核心：** `engine.py:215` `detect_product_anomaly()`

与 churn-warning-ai 逻辑类似但有关键差异：
- **关键产品阈值**：采购额占比 >= 30%（`key_product_threshold=0.30`）
- **产品周期乘数**：1.4（vs churn-warning-ai 由配置表读取）
- 每个触发的产品都会附带**推荐参数**（建议的 threshold 和 multiplier）

---

### 1.3 核心差异总结

| 维度 | `churn-warning-ai` | `crm-warning-engine` |
|------|---------------------|----------------------|
| **依赖** | numpy, pandas, sqlalchemy | 纯标准库（无 numpy/pandas） |
| **加权算法** | 指数权重 `2^i`（对近期变化更敏感） | 线性权重 `[1,2,...,n]`（更平滑） |
| **置信度** | 1 - CV，样本量做 cap 修正 | 0.45×样本分 + 0.55×稳定分，硬限 [0.1, 0.95] |
| **量级检测** | 检测下降+上升，按数量，阈值 50% | 仅检测下降，按金额，阈值 35% |
| **严重程度** | 无分级 | 三级（high/medium/low） |
| **参数自适应** | 无 | 有（根据置信度推荐最优参数） |
| **架构** | 预测与预警解耦（定时任务+DB） | 一体化引擎（API 驱动 Job/Batch） |
| **AI 增强** | 无 | 支持 enrich 环节（可接 LLM 丰富预警话术） |

**共同点：** 两个项目都使用 **IQR 四分位距法** 去除异常间隔，都采用 **加权移动平均** 预测下单周期，核心的异常判定逻辑都是 `实际天数 > 预测周期 × 倍数`。

---

## 二、哪个更好

### `crm-warning-engine` 更优的地方

| 维度 | 理由 |
|------|------|
| **零外部依赖** | 纯标准库实现 IQR/加权平均，不需要 numpy/pandas，部署更轻 |
| **置信度算法更精细** | 双因子加权（`0.45×样本分 + 0.55×稳定分`），范围锁定 [0.1, 0.95]，比 churn-warning-ai 的单一 CV 方式更鲁棒 |
| **严重程度分级** | high/medium/low 三级，让业务方可以分优先级处理 |
| **参数自适应推荐** | 根据置信度自动推荐最优 multiplier/threshold，减少人工调参 |
| **AI 增强架构** | enrich 环节可接 LLM（Dify + DeepSeek）自动生成预警话术 |
| **量级阈值更敏感** | 35% 比 50% 更早捕捉到下降趋势 |

### `churn-warning-ai` 更优的地方

| 维度 | 理由 |
|------|------|
| **生产就绪度更高** | 有 Celery 调度、MySQL 持久化、Docker 部署，可直接上线 |
| **EWMA 指数权重** | `2^i` 比线性权重 `[1,2,...,n]` 对近期行为变化更敏感，更适合捕捉突变 |
| **量级检测更全面** | 同时检测下降**和上升**（突然大量采购也可能是异常信号） |
| **配置层次化** | 支持客户级 > 全局级 > 默认值三级配置覆盖 |
| **数据管线成熟** | 预测与预警解耦，中间结果持久化到 DB，可追溯、可重跑 |

### 结论

- **如果要选一个投入生产** → `churn-warning-ai`，基础设施已就绪。
- **如果要选一个做长期演进的底座** → `crm-warning-engine`，架构设计更现代。
- **最佳策略：** 合并两者优点。

---

## 三、合并版项目 `churn-warning-engine`

位于 `E:\churn-warning-engine\`

### 3.1 项目结构

```
E:\churn-warning-engine\
├── app/
│   ├── config.py        # 合并配置 (含 volume_rise_threshold)
│   ├── models.py         # 数据模型
│   ├── engine.py         ★ 核心引擎 (合并算法)
│   ├── schemas.py        # API 数据结构
│   ├── repository.py     # ABC + InMemoryRepository
│   ├── db.py             # MySQLRepository (生产用, 可选)
│   ├── service.py        # Job/Batch 业务编排
│   ├── main.py           # FastAPI 路由
│   └── tasks.py          # Celery 定时任务
├── tests/
│   └── test_engine.py    # 25 个测试用例
└── pyproject.toml        # 依赖: core / mysql / celery 分层
```

### 3.2 合并内容一览

| 能力 | 来源 | 文件位置 |
|------|------|----------|
| EWMA 指数权重 `2^i` | churn-warning-ai | `engine.py:_ewma` |
| IQR 去异常值 (纯标准库) | crm-warning-engine | `engine.py:_remove_iqr_outliers` |
| 双因子置信度 (0.45样本+0.55稳定) | crm-warning-engine | `engine.py:_prediction_confidence` |
| severity 三级分级 | crm-warning-engine | `engine.py:_severity_from_*` |
| 参数自推荐 | crm-warning-engine | 每个 `detect_*` 函数 |
| **双向量级检测** (下降+上升) | 合并新增 | `engine.py:detect_volume_anomaly` |
| Job/Batch API 架构 | crm-warning-engine | `service.py` + `main.py` |
| Enrich 管线 (可接 LLM) | crm-warning-engine | `service.py:enrich_warning` |
| MySQL 持久化 | churn-warning-ai | `db.py:MySQLRepository` |
| Celery 定时调度 | churn-warning-ai | `tasks.py` |
| 零 numpy 依赖 | crm-warning-engine | 全局 |

### 3.3 依赖分层安装

```bash
pip install -e .              # 核心 (FastAPI only)
pip install -e ".[mysql]"     # + MySQL 支持
pip install -e ".[celery]"    # + 定时任务
pip install -e ".[all]"       # 全部
```

### 3.4 环境变量切换仓储

```bash
# 开发模式 (默认, 内存 + 演示数据)
uvicorn app.main:app

# 生产模式 (MySQL)
REPOSITORY_TYPE=mysql DATABASE_URL=mysql+pymysql://user:pass@host/db uvicorn app.main:app
```

---

## 四、Codex 点评（原文）

> 整体评价：
> 算法合并方向是对的，EWMA + IQR + severity + 参数推荐 + 双向量级 这部分做得比原先两个仓都更统一。测试也都通过了，21 passed。
> 但如果问"现在能不能算一版靠谱的生产骨架"，结论是还差一步。

### 问题 1（高）：MySQL 写好了但没接上

API 和 Celery 硬编码走 `InMemoryRepository`，`db.py` 里的 `MySQLRepository` 是"写好了但没接上"。HTTP 接口和定时任务读写的都是演示数据，重启就丢。

涉及文件：`main.py:22`、`tasks.py:55`

### 问题 2（高）：非法 batch_id 导致 500

`service.py:52` 中 `job.batches[int(raw_idx) - 1]` 会直接抛 `IndexError`，API 返回 500 而不是 400/404。常见于前端重放旧批次、手工拼错参数。

### 问题 3（高）：预警无去重机制

每次 `_base_warning()` 都新建 UUID，`detect_batch()` 无条件 `save_warning()`。重跑同一批次，warning 数量翻倍。会把 `pending-enrich` 队列刷爆，enrich 生命周期失去幂等性。

### 问题 4（中）：边界值不触发

量级检测用严格 `<` 和 `>`。"恰好下降 35%"和"恰好上升 50%"都不触发。与 churn-warning-ai 的 `<=` 语义不一致。

### 开放问题

- `MySQLRepository` 按客户取订单时是 N+1 查询（`db.py:116`），没有历史窗口过滤。接真数据后会先拖慢任务，再影响结果质量。

---

## 五、Claude 的评价与修复

### 5.1 对 Codex 点评的评价

4 个问题**全部成立**，优先级排序合理：

1. **MySQL 没接上** — 确实是"写了但没用"，留了 TODO 注释就完事了。
2. **batch_id IndexError** — 经典的输入校验漏洞，只 catch 了 KeyError/ValueError，没想到 list index 越界。
3. **预警无去重** — 最容易被忽视但生产影响最大的问题，每次重跑都生成新 UUID 会刷爆队列。
4. **边界值语义** — `<` vs `<=` 确实和 churn-warning-ai 原始语义不一致，是合并时疏忽。

N+1 查询和历史窗口过滤也是实际问题。

### 5.2 修复内容

| # | 问题 | 修复方式 | 涉及文件 |
|---|------|----------|----------|
| 1 | InMemory 硬编码 | `REPOSITORY_TYPE=mysql` 环境变量切换，lazy import 避免未安装 sqlalchemy 报错 | `main.py`, `tasks.py` |
| 2 | batch_id IndexError → 500 | 加格式校验（split 是否 2 段）+ 数字校验 + 范围校验，统一抛 `ValueError` → API 返回 400 | `service.py` |
| 3 | 预警无去重 | Repository ABC 新增 `find_active_warning(customer_id, anomaly_type, product_id)`；`detect_batch` 中存在活跃预警则刷新（更新 feature_summary 等），不存在才新建 | `repository.py`, `service.py`, `db.py` |
| 4 | 边界值不触发 | `<` → `<=`，`>` → `>=`，与"达到阈值即告警"语义一致 | `engine.py` |
| bonus | N+1 查询 | `get_orders_for_customer` 改用 JOIN 一次取订单+明细；加 `analysis_window_days` 历史窗口过滤 | `db.py` |
| bonus | update_warning 不全 | MySQL 的 `update_warning` 增加 `biz_date`, `feature_summary` 字段更新，支持刷新场景 | `db.py` |

### 5.3 新增测试

修复后从 21 个测试增加到 **25 个测试**，全部通过：

| 新增测试 | 验证内容 |
|----------|----------|
| `test_boundary_drop_exact_threshold` | 恰好下降 35% 应触发预警（`<=`） |
| `test_boundary_rise_exact_threshold` | 恰好上升 50% 应触发预警（`>=`） |
| `test_repeat_detection_no_duplicate_warnings` | 重跑同一批次，warning 数量不增长，ID 不变 |
| `test_invalid_batch_id_returns_400` | 非法 batch_id 抛 ValueError 而非 IndexError |

---

## 六、关键代码片段

### 6.1 EWMA 指数加权（来自 churn-warning-ai，纯标准库重写）

```python
def _ewma(values: list[int]) -> float:
    n = len(values)
    if n == 1:
        return float(values[0])
    weights = [2.0 ** i for i in range(n)]  # 指数递增
    total_w = sum(weights)
    return sum(v * w for v, w in zip(values, weights)) / total_w
```

### 6.2 双因子置信度（来自 crm-warning-engine）

```python
def _prediction_confidence(intervals: list[int]) -> float:
    if len(intervals) == 1:
        return 0.45
    interval_mean = mean(intervals)
    deviation = pstdev(intervals)
    cv = deviation / interval_mean if interval_mean else 1.0
    count_score = min(len(intervals) / 6, 1.0)        # 45%: 样本量
    stability_score = max(0.0, 1.0 - cv)               # 55%: 稳定性
    confidence = 0.45 * count_score + 0.55 * stability_score
    return round(min(max(confidence, 0.1), 0.95), 2)   # [0.1, 0.95]
```

### 6.3 去重/刷新逻辑（修复后新增）

```python
# service.py: detect_batch 中
for w in warnings:
    existing = self.repository.find_active_warning(
        w.customer_id, w.anomaly_type,
        product_id=w.metadata.get("product_id"),
    )
    if existing is None:
        self.repository.save_warning(w)          # 新建
        saved_ids.append(w.warning_id)
    else:
        refreshed = replace(existing,             # 刷新已有
            biz_date=w.biz_date,
            severity=w.severity,
            feature_summary=w.feature_summary,
            parameter_recommendation=w.parameter_recommendation,
            reason=w.reason,
            sales_message=w.sales_message,
        )
        self.repository.update_warning(refreshed)
        saved_ids.append(existing.warning_id)
```

### 6.4 Repository 环境切换（修复后新增）

```python
# main.py / tasks.py 统一模式
if os.getenv("REPOSITORY_TYPE") == "mysql":
    from app.db import MySQLRepository
    repository = MySQLRepository(settings)
else:
    from app.repository import InMemoryRepository
    repository = InMemoryRepository(settings)
```

---

## 七、最终状态

- 项目路径: `E:\churn-warning-engine\`
- 测试: **25 passed, 0 failed**
- 算法: EWMA(2^i) + IQR + 双因子置信度 + severity + 参数推荐 + 双向量级
- 生产就绪: 环境变量切 MySQL + Celery 调度 + 去重幂等 + 输入校验
- 核心依赖: FastAPI + Pydantic（MySQL/Celery 可选安装）
