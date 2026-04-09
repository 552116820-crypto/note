[User]
<environment_context>
  <cwd>/mnt/c/Users/55211</cwd>
  <shell>bash</shell>
  <current_date>2026-04-09</current_date>
  <timezone>Asia/Shanghai</timezone>
</environment_context>

[User]
E:\churn-warning-ai和E:\crm-warning-engine请问这两个项目中，是如何用python检测异常的

[Assistant]
我先在这两个项目里定位 Python 相关代码和“异常检测/告警判定”实现，重点看检测逻辑、阈值规则和调用入口。

[Assistant]
我已经定位到两边的核心 Python 文件了。下一步直接看检测入口和规则实现，确认它们到底是“统计阈值”、“规则引擎”还是“模型预测残差”这类方式。

[Assistant]
两个项目看起来都不是用机器学习异常检测库，而是 Python 里的“统计规则 + 阈值判断 + 周期预测”来做。现在我把核心函数和默认参数读出来，整理成可落地的说明。

[Assistant]
我再补一层运行方式差异：一个项目是“先批量算预测，再按配置出预警”，另一个是“批次执行时直接算异常并生成 warning”。这会影响你怎么接入现有 CRM。

[Assistant]
我顺手确认一下有没有用到 `sklearn`、`prophet` 之类真正的异常检测库；如果没有，就可以明确说这两边都是规则/统计法。

[Assistant]
结论先说：这两个项目都不是用 `sklearn`、`IsolationForest` 这类机器学习异常检测库来做的，而是用 Python 的统计方法加业务阈值来判定异常。我全仓检索了相关库，未发现使用；`churn-warning-ai` 的设计文档也明确写了 V1 不上重型 ML 框架，走统计法即可，[algorithm_design.md#L293](/mnt/e/churn-warning-ai/docs/algorithm_design.md#L293)。

**`churn-warning-ai`**
这个项目把异常分成 3 类：频率异常、量级异常、产品异常。

- 频率异常和产品异常的核心是“先预测正常下单周期，再判断是否超期”。预测函数在 [cycle_predictor.py#L16](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L16)：先算历史订单间隔，再用 IQR 去掉异常间隔值，再用 EWMA 指数加权平均做周期预测，最后用变异系数算置信度。默认最少 3 单才做预测、近期权重更高、低样本会压低置信度，[settings.py#L53](/mnt/e/churn-warning-ai/config/settings.py#L53)。
- 真正触发预警是在 [warning_engine.py#L18](/mnt/e/churn-warning-ai/src/services/warning_engine.py#L18) 和 [warning_engine.py#L112](/mnt/e/churn-warning-ai/src/services/warning_engine.py#L112)：如果“距上次下单天数 >= 预测周期 × 倍数”就报异常。默认倍数是 `2.0`，[settings.py#L53](/mnt/e/churn-warning-ai/config/settings.py#L53)。
- 量级异常在 [volume_detector.py#L13](/mnt/e/churn-warning-ai/src/services/volume_detector.py#L13)：直接比较“本月数量 vs 上月数量”的环比变化率。默认规则是下降超过 `50%` 或增长超过 `50%` 就算异常；如果上月为 0，直接跳过不判，[settings.py#L59](/mnt/e/churn-warning-ai/config/settings.py#L59)。
- 产品异常先在 [product_analyzer.py#L19](/mnt/e/churn-warning-ai/src/services/product_analyzer.py#L19) 识别关键产品：一年内产品金额占比达到阈值（默认 `30%`）的产品才进入检测；然后对每个关键产品复用同一套周期预测逻辑，[product_analyzer.py#L79](/mnt/e/churn-warning-ai/src/services/product_analyzer.py#L79)。
- 运行流程是“两阶段”：先批量把预测结果写入 `churn_prediction` 表，[batch_predict.py#L28](/mnt/e/churn-warning-ai/src/tasks/batch_predict.py#L28)；再由周/月任务读取预测结果和客户配置，生成预警，[warning_check.py#L71](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L71)、[warning_check.py#L149](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L149)、[warning_check.py#L272](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L272)。数据是从 CRM 库里查订单和订单明细来的，[data_fetcher.py#L46](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L46)、[data_fetcher.py#L118](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L118)。

**`crm-warning-engine`**
这个项目思路类似，但更像一个轻量 demo/starter，当前数据源是内存仓库，不是真实数据库，[repository.py#L10](/mnt/e/crm-warning-engine/app/repository.py#L10)。

- 核心检测都在 [engine.py#L56](/mnt/e/crm-warning-engine/app/engine.py#L56)。它同样先做周期预测：订单日期去重排序，算相邻间隔，用 IQR 去异常值，再做加权平均；不过它的加权不是 EWMA `2^i`，而是线性权重 `1..n`，[engine.py#L23](/mnt/e/crm-warning-engine/app/engine.py#L23)、[engine.py#L36](/mnt/e/crm-warning-engine/app/engine.py#L36)。
- 频率异常在 [engine.py#L115](/mnt/e/crm-warning-engine/app/engine.py#L115)：`days_since_last_order > predicted_days * frequency_multiplier` 就触发。默认倍数是 `1.5`，比前一个项目更敏感，[config.py#L8](/mnt/e/crm-warning-engine/app/config.py#L8)。
- 量级异常在 [engine.py#L163](/mnt/e/crm-warning-engine/app/engine.py#L163)：比较的是“本月金额 vs 上月金额”，不是数量；而且它只检测下降，不检测暴涨。默认阈值是下降 `35%`，[config.py#L8](/mnt/e/crm-warning-engine/app/config.py#L8)。
- 产品异常在 [engine.py#L215](/mnt/e/crm-warning-engine/app/engine.py#L215)：先统计近 `365` 天内各产品金额占比，超过 `30%` 的算关键产品，再对该产品复用周期预测；如果该产品距离上次采购天数超过 `预测周期 × 1.4`，就出预警。
- 触发时是“边跑批边算”，由 [service.py#L46](/mnt/e/crm-warning-engine/app/service.py#L46) 直接调用 `detect_frequency_anomaly / detect_volume_anomaly / detect_product_anomaly`，不是像前一个项目那样先落预测表再二次判定。

**两者最关键的区别**
- `churn-warning-ai`：更像生产版骨架，接 CRM/MySQL，先预测再预警，量级看“数量”，支持“下降/上升”双向异常。
- `crm-warning-engine`：更像演示版规则引擎，批处理时直接计算，量级看“金额”，只报“下降异常”。

如果你要，我可以下一步直接给你画出这两个项目各自的“异常检测流程图”，或者把“应该改哪个 Python 文件来调阈值”单独列出来。

[User]
哪个好

[Assistant]
如果按“更适合拿来接真实业务、继续上线”来选，`E:\churn-warning-ai` 更好。

原因很直接：

- 它已经有真实 CRM 数据读取层，不是纯 demo，[data_fetcher.py](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L15)。
- 它是完整链路：先批量做周期预测并落库，再按配置触发预警，[batch_predict.py#L28](/mnt/e/churn-warning-ai/src/tasks/batch_predict.py#L28)、[warning_check.py#L71](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L71)。
- 检测能力更完整：频率异常、量级异常、产品异常都齐，量级还支持“下降”和“暴涨”双向判断，[volume_detector.py#L13](/mnt/e/churn-warning-ai/src/services/volume_detector.py#L13)。
- 周期预测也更细一点，用了 IQR 去异常间隔、指数加权平均、置信度修正，[cycle_predictor.py#L16](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L16)。

`crm-warning-engine` 的优点是代码更干净、更容易读，但它现在更像 starter/demo。它的仓储层是内存实现，而且仓库里直接 seed 了演示数据，[repository.py#L10](/mnt/e/crm-warning-engine/app/repository.py#L10)。所以它适合快速原型、讲流程、重写参考，不太像可直接接生产的版本。

补一句：`crm-warning-engine` 默认阈值更激进，频率 `1.5x`、量级下降 `35%`、产品 `1.4x`，[config.py](/mnt/e/crm-warning-engine/app/config.py#L8)；`churn-warning-ai` 默认更保守，频率 `2.0x`、量级 `50%`、产品 `2.0x`，[settings.py](/mnt/e/churn-warning-ai/config/settings.py#L53)。所以后者“没那么爱报错”，前者“更容易报异常”，这属于策略差异，不是绝对优劣。

如果你只让我选一个：
- 要继续做成正式项目，选 `churn-warning-ai`
- 要拿来重构成你自己的轻量引擎，选 `crm-warning-engine`

如果你要，我可以下一步直接给你做一张“这两个项目的优缺点对比表”。

[User]
这是最佳策略吗？ 把两者的优点合并 —— 用 crm-warning-engine 的引擎架构（severity + 参数推荐 + enrich），换上 churn-warning-ai的指数权重 EWMA 算法和双向量级检测，再补上 Celery 调度和 MySQL 持久化。

[Assistant]
我先确认一下 `churn-warning-ai` 里 Celery 和持久化到底落到什么程度，避免把“设计上有”误当成“代码里已经完整接好”。

[Assistant]
我补看了一下调度落地情况，重点确认是不是已经有真正的 worker/beat 服务，还是只有依赖声明。

[Assistant]
方向对，但“最佳策略”我会改成一句更准确的话：

不是“以 `crm-warning-engine` 为底座补齐一切”，而是“以 `churn-warning-ai` 为主干，吸收 `crm-warning-engine` 的告警生命周期设计”。

原因很明确。`churn-warning-ai` 已经更接近真实系统：有 CRM/MySQL 数据读取、预测落库、配置覆盖和批量任务骨架，[data_fetcher.py#L46](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L46) [db.py#L51](/mnt/e/churn-warning-ai/src/models/db.py#L51) [batch_predict.py#L28](/mnt/e/churn-warning-ai/src/tasks/batch_predict.py#L28)。`crm-warning-engine` 的强项不在基础设施，而在告警记录模型、`severity`、参数推荐、`enrich` 工作流，[engine.py#L71](/mnt/e/crm-warning-engine/app/engine.py#L71) [models.py#L41](/mnt/e/crm-warning-engine/app/models.py#L41) [service.py#L90](/mnt/e/crm-warning-engine/app/service.py#L90)。  
另外，`churn-warning-ai` 虽然有 Celery/Redis 依赖和 Redis 容器，[requirements.txt#L14](/mnt/e/churn-warning-ai/requirements.txt#L14) [docker-compose.yml#L47](/mnt/e/churn-warning-ai/docker-compose.yml#L47)，但我没找到真正的 `Celery` worker/beat 代码，所以调度这块确实还没落完整。

我建议的最优合并方案是：

- 保留 `churn-warning-ai` 的周期预测内核：IQR 去异常 + 指数权重 EWMA，这部分比 `crm-warning-engine` 的线性加权更适合“近期变化更重要”的场景，[cycle_predictor.py#L64](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L64) [cycle_predictor.py#L116](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L116)。
- 引入 `crm-warning-engine` 的告警输出层：`severity`、`feature_summary`、`parameter_recommendation`、`pending_enrich/enriched` 状态机，[engine.py#L87](/mnt/e/crm-warning-engine/app/engine.py#L87) [service.py#L79](/mnt/e/crm-warning-engine/app/service.py#L79)。
- 量级检测不要只“换成双向”这么简单，最好直接抽象成 `metric=amount|quantity|both`。`churn-warning-ai` 现在看“数量”，[data_fetcher.py#L118](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L118)；`crm-warning-engine` 看“金额”，[engine.py#L173](/mnt/e/crm-warning-engine/app/engine.py#L173)。这两个业务含义不同，最好可配置。
- “上涨异常”不要默认和“流失预警”放同一优先级。对 churn 场景，下降异常是主信号；上涨更像运营波动或机会信号，建议单独类型或低优先级，否则会制造告警噪音。
- 真正要补的是工作流，不是算法本身：补 `warning_record` 持久化表、幂等去重、关闭/重开规则、Celery worker/beat、失败重试。

所以我的结论是：

你提的“合并两者优点”这个终态是对的，但最佳实施路径应当是 `churn-warning-ai` 做骨架，移植 `crm-warning-engine` 的告警治理能力，而不是反过来。

如果你要，我下一步可以直接给你出一版“合并后的目标目录结构 + 表设计 + 重构顺序”。

[User]
我要

[Assistant]
我按“能直接开工”的粒度给你一版合并方案，重点放目标结构、表设计和重构顺序，不讲空话。

[Assistant]
最佳落地方式是直接把 `E:\churn-warning-ai` 作为主仓重构，不新起第三个项目。保留它现有的数据接入、预测落库和配置体系，再迁入 `E:\crm-warning-engine` 的 `severity`、参数推荐和 `enrich` 生命周期。核心来源分别是 [cycle_predictor.py](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L16)、[warning_check.py](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L71)、[db.py](/mnt/e/churn-warning-ai/src/models/db.py#L51)、[engine.py](/mnt/e/crm-warning-engine/app/engine.py#L71)、[service.py](/mnt/e/crm-warning-engine/app/service.py#L46)。

**目标结构**
```text
src/
  api/
    predict_routes.py
    detect_routes.py
    warning_routes.py
    job_routes.py
  core/
    settings.py
    celery_app.py
  models/
    db.py
    schemas.py
  repositories/
    crm_order_repo.py
    prediction_repo.py
    warning_repo.py
    config_repo.py
  engines/
    cycle_predictor.py
    frequency_engine.py
    volume_engine.py
    product_engine.py
    severity.py
    recommendation.py
  services/
    prediction_service.py
    detection_service.py
    warning_service.py
    enrich_service.py
  tasks/
    prediction_tasks.py
    detection_tasks.py
```

- `cycle_predictor.py` 保留 [churn-warning-ai 的 EWMA + IQR](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L64)。
- `severity.py` 和 `recommendation.py` 提炼自 [crm-warning-engine 的 engine](/mnt/e/crm-warning-engine/app/engine.py#L71)。
- `warning_service.py` 取代现在 [warning_check.py 里的 `_warning_callback`](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L41)，改成直接持久化和去重。
- `volume_engine.py` 统一支持 `quantity`、`amount`、`both`，并支持 `drop` 和 `rise` 双向检测。
- API 不要一次性推翻现有 `/api/ai/*`，先兼容，再补 `/api/warnings/*`。

**表设计**
保留现有表：
- `churn_prediction`
- `prediction_job_log`
- `churn_warning_global_config`
- `churn_warning_customer_config`

改造现有表：
- `volume_anomaly_snapshot`
- 新增 `metric_scope`、`current_month_amount`、`last_month_amount`、`amount_change_rate`、`quantity_change_rate`、`anomaly_metric`
- 否则你后面一旦同时看数量和金额，这张表会不够用

新增核心表：
- `warning_record`
- 字段建议：`id`、`warning_id(uuid)`、`fingerprint`、`anomaly_type`、`metric_scope`、`anomaly_direction`、`customer_id`、`customer_name`、`salesperson_id`、`product_id`、`product_name`、`biz_date`、`severity`、`status`、`feature_summary(json)`、`parameter_recommendation(json)`、`reason`、`sales_message`、`metadata(json)`、`first_detected_at`、`last_detected_at`、`closed_at`
- `status` 建议枚举：`pending_enrich`、`enriched`、`closed`、`suppressed`
- `fingerprint` 用于幂等去重，建议由 `anomaly_type + customer_id + product_id + metric_scope` 组成

- `warning_event`
- 字段建议：`id`、`warning_id`、`event_type`、`operator_type`、`operator_id`、`payload(json)`、`created_at`
- `event_type` 建议：`created`、`refreshed`、`enriched`、`closed`、`reopened`
- 这张表是为了审计，不然 `enrich` 和关闭历史会丢

- `warning_job_log`
- 字段建议：`id`、`job_id`、`job_type`、`batch_no`、`biz_date`、`status`、`total_customers`、`success_count`、`fail_count`、`error_message`、`started_at`、`finished_at`
- `job_type` 建议：`prediction`、`frequency_detect`、`volume_detect`、`product_detect`

配置表建议补字段：
- `volume_metric_scope`
- `severity_enabled`
- `rise_threshold_enabled`
- 这样才能把双向量级和告警分级配透

**重构顺序**
1. 先在 [db.py](/mnt/e/churn-warning-ai/src/models/db.py#L51) 增加 `warning_record`、`warning_event`、`warning_job_log` ORM 和 migration，不动现有预测逻辑。
2. 把 [crm-warning-engine 的 `WarningRecord` 输出模型](/mnt/e/crm-warning-engine/app/models.py#L41) 映射进 `churn-warning-ai`，让频率、量级、产品检测都返回统一的富告警对象，而不是只返回简单消息。
3. 保留 [当前周期预测算法](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L16)，把 [crm-warning-engine 的 `severity` 和参数推荐逻辑](/mnt/e/crm-warning-engine/app/engine.py#L130) 拆成独立模块接进去。
4. 重写量级检测：从 [现有仅数量环比](/mnt/e/churn-warning-ai/src/services/volume_detector.py#L13) 升级成 `amount/quantity/both + rise/drop`，并把快照表一并升级。
5. 把 [warning_check.py](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L56) 的 callback 推送改成 `create_or_refresh_warning()`，先查活动告警，有则刷新 `last_detected_at`，无则新建，解决重复告警。
6. 补 `/api/warnings/pending-enrich`、`/api/warnings/{id}/feature-summary`、`/api/warnings/{id}/enrich`，接口形态可直接参考 [crm-warning-engine 的 main.py](/mnt/e/crm-warning-engine/app/main.py#L100)。
7. 最后再补 Celery。当前仓库只有依赖和 Redis 容器，[requirements.txt](/mnt/e/churn-warning-ai/requirements.txt#L14) [docker-compose.yml](/mnt/e/churn-warning-ai/docker-compose.yml#L47)，但没有真正的 worker/beat 代码，所以这一步要新建 `celery_app.py`、`prediction_tasks.py`、`detection_tasks.py`。
8. 测试顺序不要反过来。先补 detection 单测，再补 warning lifecycle 集成测试，最后补 Celery 任务测试。

**最终效果**
- 算法层用 `churn-warning-ai` 的预测质量
- 告警层用 `crm-warning-engine` 的治理能力
- 基础设施层补足 MySQL 持久化、幂等去重、批任务和 enrich 流程

最值得先落的第一批改动是第 `1-5` 步，因为这批做完，系统已经从“能检测”升级成“能稳定产出可运营告警”。下一步如果你要，我可以直接给你出 MySQL DDL 和文件级改造清单。

[User]
我要

[Assistant]
我先把现有建表脚本读出来，按你这两个项目现有字段风格给出一版可直接落库的 DDL，然后附上文件级改造清单。

[Assistant]
我把文档里关于 `churn_warning_record` 和配置表的约束也读一下，避免给你的 DDL 和现有设计冲突。

[Assistant]
下面这版可以直接作为 `MySQL 8` 的第一版 DDL，用在 `E:\churn-warning-ai\sql\schema.sql` 里，风格和你现有表一致，[schema.sql](/mnt/e/churn-warning-ai/sql/schema.sql#L1)。

**DDL**
```sql
-- ============================================================
-- 4. 预警记录表
-- 说明:
-- 1) 一条“当前有效”的预警占一行
-- 2) 关闭后 dedupe_key 置空，允许后续重新开新预警
-- 3) feature_summary / parameter_recommendation / metadata 用 JSON 承载扩展字段
-- ============================================================
CREATE TABLE IF NOT EXISTS churn_warning_record (
    id                          BIGINT PRIMARY KEY AUTO_INCREMENT,
    warning_id                  VARCHAR(64) NOT NULL COMMENT '业务预警ID(UUID)',
    dedupe_key                  VARCHAR(255) DEFAULT NULL COMMENT '活动预警去重键，关闭后置NULL',
    fingerprint                 VARCHAR(255) NOT NULL COMMENT '稳定指纹: type+customer+product+metric',
    anomaly_type                VARCHAR(20) NOT NULL COMMENT 'frequency / volume / product',
    warning_type                TINYINT NOT NULL COMMENT '1-频率 2-量级 3-产品',
    metric_scope                VARCHAR(20) DEFAULT NULL COMMENT 'quantity / amount / both / null',
    anomaly_direction           VARCHAR(10) DEFAULT NULL COMMENT 'drop / rise / null',
    customer_id                 BIGINT NOT NULL COMMENT '客户ID',
    customer_name               VARCHAR(200) DEFAULT NULL COMMENT '客户名称',
    salesperson_id              BIGINT DEFAULT NULL COMMENT '销售ID',
    product_id                  BIGINT DEFAULT NULL COMMENT '产品ID(产品异常时填写)',
    product_name                VARCHAR(200) DEFAULT NULL COMMENT '产品名称(产品异常时填写)',
    biz_date                    DATE NOT NULL COMMENT '业务日期',
    severity                    VARCHAR(10) NOT NULL DEFAULT 'low' COMMENT 'low / medium / high',
    status                      VARCHAR(20) NOT NULL COMMENT 'pending_enrich / enriched / closed / suppressed',
    feature_summary             JSON DEFAULT NULL COMMENT '特征摘要',
    parameter_recommendation    JSON DEFAULT NULL COMMENT '参数建议',
    reason                      TEXT DEFAULT NULL COMMENT '系统原因或人工修订原因',
    sales_message               TEXT DEFAULT NULL COMMENT '给销售的话术',
    metadata                    JSON DEFAULT NULL COMMENT '扩展元数据',
    first_detected_at           DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '首次发现时间',
    last_detected_at            DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '最近一次刷新时间',
    closed_at                   DATETIME DEFAULT NULL COMMENT '关闭时间',
    created_at                  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at                  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_warning_id (warning_id),
    UNIQUE KEY uk_dedupe_key (dedupe_key),
    INDEX idx_customer_status (customer_id, status),
    INDEX idx_salesperson_status (salesperson_id, status),
    INDEX idx_anomaly_type_status (anomaly_type, status),
    INDEX idx_biz_date (biz_date),
    INDEX idx_product (product_id),
    INDEX idx_last_detected (last_detected_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='客户流失预警记录表';


-- ============================================================
-- 5. 预警事件流水表
-- 说明: 审计每次创建、刷新、富化、关闭、重开
-- ============================================================
CREATE TABLE IF NOT EXISTS churn_warning_event (
    id                          BIGINT PRIMARY KEY AUTO_INCREMENT,
    warning_id                  VARCHAR(64) NOT NULL COMMENT '关联 warning_record.warning_id',
    event_type                  VARCHAR(20) NOT NULL COMMENT 'created / refreshed / enriched / closed / reopened / suppressed',
    operator_type               VARCHAR(20) NOT NULL DEFAULT 'system' COMMENT 'system / user / api / task',
    operator_id                 VARCHAR(64) DEFAULT NULL COMMENT '操作人/任务ID',
    payload                     JSON DEFAULT NULL COMMENT '事件附加内容',
    created_at                  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_warning_created (warning_id, created_at),
    INDEX idx_event_type (event_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='预警事件流水表';


-- ============================================================
-- 6. 预警任务日志表
-- 说明: 记录频率/量级/产品检测任务运行情况
-- ============================================================
CREATE TABLE IF NOT EXISTS churn_warning_job_log (
    id                          BIGINT PRIMARY KEY AUTO_INCREMENT,
    job_id                      VARCHAR(64) NOT NULL COMMENT '任务ID(UUID)',
    job_type                    VARCHAR(50) NOT NULL COMMENT 'frequency_detect / volume_detect / product_detect',
    batch_no                    INT DEFAULT NULL COMMENT '批次号',
    biz_date                    DATE NOT NULL COMMENT '业务日期',
    status                      VARCHAR(20) NOT NULL COMMENT 'running / success / partial / failed',
    total_customers             INT DEFAULT 0,
    success_count               INT DEFAULT 0,
    fail_count                  INT DEFAULT 0,
    skip_count                  INT DEFAULT 0,
    warning_count               INT DEFAULT 0 COMMENT '触发预警数',
    error_message               TEXT DEFAULT NULL,
    started_at                  DATETIME NOT NULL,
    finished_at                 DATETIME DEFAULT NULL,
    duration_seconds            INT DEFAULT NULL,
    created_at                  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_job_id (job_id),
    INDEX idx_job_type_biz_date (job_type, biz_date),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='预警检测任务日志表';


-- ============================================================
-- 7. 扩展量级快照表
-- 说明: 从“只看数量”升级到“可看数量/金额”
-- ============================================================
ALTER TABLE volume_anomaly_snapshot
    ADD COLUMN metric_scope         VARCHAR(20) DEFAULT 'quantity' COMMENT 'quantity / amount / both' AFTER snapshot_month,
    ADD COLUMN current_month_amount DECIMAL(14,2) DEFAULT 0 COMMENT '当月下单金额' AFTER last_month_qty,
    ADD COLUMN last_month_amount    DECIMAL(14,2) DEFAULT 0 COMMENT '上月下单金额' AFTER current_month_amount,
    ADD COLUMN quantity_change_rate DECIMAL(8,2) DEFAULT NULL COMMENT '数量环比变化率(%)' AFTER change_rate,
    ADD COLUMN amount_change_rate   DECIMAL(8,2) DEFAULT NULL COMMENT '金额环比变化率(%)' AFTER quantity_change_rate,
    ADD COLUMN anomaly_metric       VARCHAR(20) DEFAULT NULL COMMENT 'quantity / amount / both' AFTER anomaly_direction;

-- 如果已存在旧字段 change_rate，则约定:
-- 1) change_rate 先保留兼容老接口
-- 2) quantity 模式下 change_rate = quantity_change_rate
-- 3) amount 模式下 change_rate = amount_change_rate


-- ============================================================
-- 8. 配置表建议扩展字段
-- 说明: 让量级检测支持 数量/金额/双指标 + 双向阈值
-- 注意: 若你的配置表已存在，则执行 ALTER；若尚未建表，请合并到原始建表SQL
-- ============================================================
ALTER TABLE churn_warning_global_config
    ADD COLUMN volume_metric_scope  VARCHAR(20) DEFAULT 'quantity' COMMENT 'quantity / amount / both',
    ADD COLUMN volume_drop_pct_amount DECIMAL(5,2) DEFAULT NULL COMMENT '金额下降阈值(%)',
    ADD COLUMN volume_rise_pct_amount DECIMAL(5,2) DEFAULT NULL COMMENT '金额上升阈值(%)',
    ADD COLUMN rise_threshold_enabled TINYINT DEFAULT 1 COMMENT '是否启用上升异常',
    ADD COLUMN severity_enabled TINYINT DEFAULT 1 COMMENT '是否启用严重度计算';

ALTER TABLE churn_warning_customer_config
    ADD COLUMN volume_metric_scope  VARCHAR(20) DEFAULT 'quantity' COMMENT 'quantity / amount / both',
    ADD COLUMN volume_drop_pct_amount DECIMAL(5,2) DEFAULT NULL COMMENT '金额下降阈值(%)',
    ADD COLUMN volume_rise_pct_amount DECIMAL(5,2) DEFAULT NULL COMMENT '金额上升阈值(%)',
    ADD COLUMN rise_threshold_enabled TINYINT DEFAULT 1 COMMENT '是否启用上升异常',
    ADD COLUMN severity_enabled TINYINT DEFAULT 1 COMMENT '是否启用严重度计算';
```

有两个设计点要注意。

- `dedupe_key` 是为了替代“同客户同类型未关闭预警不重复发”的老回调约束，[data_requirements.md#L108](/mnt/e/churn-warning-ai/docs/data_requirements.md#L108)。活动预警时把它设成 `fingerprint`，关闭时置 `NULL`，这样 MySQL 的唯一索引就能保证“同一指纹只有一条活动预警”。
- `volume_anomaly_snapshot` 我保留了旧的 `change_rate`，这样你现有 [api.py](/mnt/e/churn-warning-ai/src/api.py#L130) 不会立刻炸，后面再逐步切到 `quantity_change_rate / amount_change_rate`。

**文件级改造清单**
按这个顺序改，风险最低。

1. `sql/schema.sql`
   - 加上上面的 DDL。
   - 目标是先把库结构铺平。

2. `src/models/db.py`
   - 新增 `ChurnWarningRecord`、`ChurnWarningEvent`、`WarningJobLog` ORM。
   - 扩展 `VolumeAnomalySnapshot` 字段。
   - 保留现有 `ChurnPrediction`、`PredictionJobLog` 不动。
   - 参考位置：[db.py](/mnt/e/churn-warning-ai/src/models/db.py#L48)

3. `src/models/schemas.py`
   - 现有 `WarningItem` 太薄，只够消息推送，[schemas.py](/mnt/e/churn-warning-ai/src/models/schemas.py#L49)。
   - 新增：
     - `WarningFeatureSummary`
     - `ParameterRecommendation`
     - `RichWarningItem`
     - `EnrichWarningRequest`
     - `WarningResponse`
   - 保留旧 `WarningItem` 一段时间兼容旧接口。

4. `src/services/cycle_predictor.py`
   - 算法基本保留。
   - 只补一个统一返回结构，方便频率和产品共用。
   - 核心逻辑不用动，当前这部分是优点，[cycle_predictor.py](/mnt/e/churn-warning-ai/src/services/cycle_predictor.py#L16)。

5. 新增 `src/services/severity.py`
   - 从 `crm-warning-engine` 提炼：
     - `severity_from_ratio()`
     - `severity_from_drop()`
   - 来源参考：[engine.py](/mnt/e/crm-warning-engine/app/engine.py#L71)

6. 新增 `src/services/recommendation.py`
   - 放参数推荐逻辑：
     - `recommended_frequency_multiplier`
     - `recommended_product_multiplier`
     - `recommended_volume_drop_threshold`
     - `recommended_key_product_threshold`
   - 来源参考：[engine.py](/mnt/e/crm-warning-engine/app/engine.py#L131)

7. `src/services/volume_detector.py`
   - 从现在的“数量环比 + 双向阈值”升级成统一引擎。
   - 建议新增参数：
     - `current_month_qty`
     - `last_month_qty`
     - `current_month_amount`
     - `last_month_amount`
     - `metric_scope`
     - `drop_threshold_qty`
     - `rise_threshold_qty`
     - `drop_threshold_amount`
     - `rise_threshold_amount`
   - 输出要能说明到底是哪种指标异常。
   - 参考当前实现：[volume_detector.py](/mnt/e/churn-warning-ai/src/services/volume_detector.py#L13)

8. `src/services/warning_engine.py`
   - 不再只返回“message + detail”。
   - 改成返回富对象，带：
     - `severity`
     - `feature_summary`
     - `parameter_recommendation`
     - `reason`
     - `sales_message`
     - `metadata`
   - 保留当前三个入口：
     - `check_frequency_warning`
     - `check_volume_warning`
     - `check_product_warning`
   - 参考当前入口：[warning_engine.py](/mnt/e/churn-warning-ai/src/services/warning_engine.py#L18)

9. 新增 `src/services/warning_service.py`
   - 新核心服务，取代 `_warning_callback`。
   - 建议提供：
     - `create_or_refresh_warning()`
     - `close_warning()`
     - `enrich_warning()`
     - `list_pending_warnings()`
     - `get_feature_summary()`
   - `create_or_refresh_warning()` 逻辑：
     - 算 `fingerprint`
     - 查 `dedupe_key`
     - 有活动记录则刷新 `last_detected_at`
     - 没有则新建记录 + 写 event

10. `src/tasks/warning_check.py`
    - 现在是检测后 `_emit_warnings()`，[warning_check.py](/mnt/e/churn-warning-ai/src/tasks/warning_check.py#L56)。
    - 改成检测后直接调 `warning_service.create_or_refresh_warning()`。
    - 同时把日志写入 `churn_warning_job_log`，不要再只打 log。

11. `src/api.py`
    - 现有 `/api/ai/detect/*` 保留。
    - 新增或拆分出：
      - `GET /api/warnings/pending-enrich`
      - `GET /api/warnings/{warning_id}/feature-summary`
      - `POST /api/warnings/{warning_id}/enrich`
      - `GET /api/warnings/{warning_id}`
    - 接口形态可以参考 `crm-warning-engine`，[main.py](/mnt/e/crm-warning-engine/app/main.py#L100)

12. `src/utils/data_fetcher.py`
    - 新增按月金额聚合函数，和现有数量聚合并存。
   - 当前只有数量聚合：[data_fetcher.py](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L118)
   - 建议补：
     - `get_monthly_order_amount(customer_id, year_month)`

13. 新增 `src/tasks/celery_app.py`
    - 当前仓库只有 Celery 依赖，没有 worker/beat 代码，[requirements.txt](/mnt/e/churn-warning-ai/requirements.txt#L14)。
    - 这里要补：
      - `Celery(...)`
      - beat schedule
      - broker/backend 配置

14. 新增 `src/tasks/prediction_tasks.py` 和 `src/tasks/detection_tasks.py`
    - 包装：
      - `run_batch_prediction`
      - `run_weekly_frequency_check`
      - `run_monthly_volume_check`
      - `run_weekly_product_check`
    - 让它们可被 Celery 调度。

15. `docker-compose.yml`
    - 现在只有 `app/mysql/redis`，[docker-compose.yml](/mnt/e/churn-warning-ai/docker-compose.yml#L3)。
    - 需要补：
      - `worker`
      - `beat`
    - 否则 Celery 只是依赖，不是可运行任务系统。

**建议的开发批次**
- 第一批：`schema.sql`、`db.py`、`warning_service.py`、`warning_engine.py`
- 第二批：`volume_detector.py`、`data_fetcher.py`、`warning_check.py`
- 第三批：`api.py` 的 enrich 接口
- 第四批：Celery worker/beat 和 `docker-compose.yml`
- 第五批：测试补齐

如果你要继续往下走，最有价值的下一步不是再讲方案，而是我直接给你出“第一批改造的具体文件 patch 方案”，也就是每个文件要新增哪些类、哪些函数、函数签名怎么定。

[User]
E:\churn-warning-engine我改进了一下，你看怎么样

[Assistant]
我先把 `E:\churn-warning-engine` 的结构和关键改动点扫一遍，重点看异常引擎、持久化、任务调度和测试覆盖，然后给你结论式 review。

[Assistant]
我已经看到你把 EWMA、双向量级、Celery 和 MySQL 都接进来了。现在进入 review 细看实现细节和边界条件，重点找会在真实运行中出错的地方。

[Assistant]
我已经看到几个结构性改进是成立的，但也发现几个可能在生产接入时直接出问题的点。再补看 API 和 schema 后，我给你按严重级别列出来。

[Assistant]
测试是全绿的，但覆盖还没碰到几个关键边界。我再核对一下测试集和实现是否存在“看上去没问题，实际上线上会偏”的地方。

[Assistant]
我确认到了一个边界条件风险，测试里还没覆盖。再跑两个最小例子把它坐实，然后我给你正式 review。

[Assistant]
我还要确认一个点：重复执行同一批任务时会不会无限新增同类预警。如果会，这在真实运营里是高噪音问题。

[Assistant]
主要问题有 4 个，优先级从高到低排。

1. 高: API 和 Celery 现在仍然硬编码走 `InMemoryRepository`，实际不会用到你新增的 MySQL 持久化层。[main.py](/mnt/e/churn-warning-engine/app/main.py#L22) [tasks.py](/mnt/e/churn-warning-engine/app/tasks.py#L55)  
这意味着 HTTP 接口和定时任务读写的都是演示数据，而且状态只在进程内存里，重启就丢。`app/db.py` 里的 `MySQLRepository` 目前是“写好了但没接上”。如果你是按生产可用来评估，这个是当前最大的阻塞点。

2. 高: 非法 `batch_id` 会直接抛 `IndexError`，API 现在会返回 500，而不是可控的 400/404。[service.py](/mnt/e/churn-warning-engine/app/service.py#L52) [main.py](/mnt/e/churn-warning-engine/app/main.py#L79)  
我本地用 `job_id:999` 复现了 `IndexError: list index out of range`。这类错误在线上很常见，比如前端重放旧批次、任务过期、手工拼错参数。

3. 高: 预警没有去重/刷新机制，重复跑同一批会不断生成新的 warning 记录。[engine.py](/mnt/e/churn-warning-engine/app/engine.py#L163) [service.py](/mnt/e/churn-warning-engine/app/service.py#L76) [db.py](/mnt/e/churn-warning-engine/app/db.py#L171)  
每次 `_base_warning()` 都新建 UUID，`detect_batch()` 又无条件 `save_warning()`。我本地重跑同一批次，两次都产出了新的 `warning_id`，已存记录数从 1 变 2。这会把 `pending-enrich` 队列刷爆，也让 `enrich` 生命周期失去幂等性。

4. 中: 量级异常的边界值现在不触发，和你合并前的 `churn-warning-ai` 语义不一致。[engine.py](/mnt/e/churn-warning-engine/app/engine.py#L286) [engine.py](/mnt/e/churn-warning-engine/app/engine.py#L305) [tests/test_engine.py](/mnt/e/churn-warning-engine/tests/test_engine.py#L130)  
你现在用的是严格 `<` 和 `>`。我本地验证了“恰好下降 35%”和“恰好上升 50%”都会返回空列表。若业务配置页把阈值理解为“达到阈值就告警”，这里会少报。

开放问题和后续风险：

- 我默认你这个仓是朝“真实 CRM + MySQL + Celery”方向走的。如果你只是做 demo，前 1、3 条的严重度会下降。
- `MySQLRepository` 现在还有一个明显的扩展风险：按客户取订单时是 N+1 查询，而且没有完成状态/历史窗口过滤。[db.py](/mnt/e/churn-warning-engine/app/db.py#L116) 这不会马上炸，但接真数据后会先拖慢任务，再影响结果质量。

整体评价：

算法合并方向是对的，`EWMA + IQR + severity + 参数推荐 + 双向量级` 这部分做得比原先两个仓都更统一，[engine.py](/mnt/e/churn-warning-engine/app/engine.py#L98)。测试也都通过了，我本地跑的是 `21 passed`。  
但如果问“现在能不能算一版靠谱的生产骨架”，结论是还差一步，先把 `MySQLRepository` 真接入、补 `batch_id` 校验、再做 warning 去重/刷新，这个项目的质量就会明显上一个台阶。

[User]
这和你之前的结论：“合并两者优点”这个终态是对的，但最佳实施路径应当是 churn-warning-ai 做骨架，移植 crm-warning-engine 的告警治理能力，而不是反过来。这个改进后的版本遵守了吗

[Assistant]
没有完全遵守，算是“终态方向对了，但实施路径还是偏向 `crm-warning-engine`”。

核心判断很简单：

- 算法层，你已经往我说的终态靠了。`EWMA + IQR + severity + 参数推荐 + 双向量级` 都合进来了，[engine.py](/mnt/e/churn-warning-engine/app/engine.py#L4)。
- 但骨架层，当前还是 `crm-warning-engine` 的骨架，不是 `churn-warning-ai` 的骨架。任务编排、Job/Batch API、`enrich` 流程、Repository 抽象，都是沿着原 `crm-warning-engine` 继续长出来的，[service.py](/mnt/e/churn-warning-engine/app/service.py#L18) [main.py](/mnt/e/churn-warning-engine/app/main.py#L56)。

最关键的几个证据是：

- 你现在没有 `churn-warning-ai` 那种“先预测落库，再按配置检测”的两阶段主链路。当前还是在检测时直接算异常，[service.py](/mnt/e/churn-warning-engine/app/service.py#L52)。
- 你没有接入 `churn-warning-ai` 的配置读取模式，也就是客户级/全局预警配置表驱动，[data_fetcher.py#L151](/mnt/e/churn-warning-ai/src/utils/data_fetcher.py#L151) 在新仓里没有对应实现。
- 你虽然写了 `MySQLRepository`，但主入口和 Celery 仍默认走 `InMemoryRepository`，[main.py](/mnt/e/churn-warning-engine/app/main.py#L22) [tasks.py](/mnt/e/churn-warning-engine/app/tasks.py#L55)。这说明“生产骨架”还没真正切到持久化主干。
- 你也没有继承 `churn-warning-ai` 那几张关键业务表思路，比如 `churn_prediction`、`volume_anomaly_snapshot`、`prediction_job_log` 这一套预测/快照体系；现在是 `warning_jobs` 和 `warning_records` 这套更偏 `crm-warning-engine` 的模型，[db.py](/mnt/e/churn-warning-engine/app/db.py#L37)。

所以更准确地说：

- 如果问“能力上是不是朝我建议的合并终态走了”：是，走对了。
- 如果问“是否遵守了我说的最佳实施路径，即以 `churn-warning-ai` 为主干吸收治理能力”：没有完全遵守，现在更像是“以 `crm-warning-engine` 为主干，补进了 `churn-warning-ai` 的算法和部分基础设施”。

我会给这个版本一个判断：

- 终态一致性：`7/10`
- 路径一致性：`4/10`

如果你想把它真正拉回我之前建议的路径，下一步最重要的不是再改算法，而是补这 4 件事：

1. 把主入口和 Celery 默认仓储切到 `MySQLRepository`，不要再默认内存仓。
2. 加上“预测落库 -> 检测读取预测结果”的两阶段链路。
3. 接入客户级/全局配置表读取。
4. 补 warning 去重/刷新机制，而不是每次生成新 UUID。

如果你要，我可以下一步直接告诉你：基于你这个 `E:\churn-warning-engine`，最少改哪 5 个文件，就能把它从“偏 crm 骨架”拉回“以 churn-warning-ai 为主干”的实现。


