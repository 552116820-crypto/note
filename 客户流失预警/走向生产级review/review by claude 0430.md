我已经把整个仓库扫了一遍：跑了一次 `pytest`（**245 个用例全部通过，0.96s**），读了 `pyproject.toml`、`AGENTS.md`、`docs/technical-design.md`（1500+ 行的核心设计文档）、`docs/runbook.md`，以及 `service.py`、`mysql_repo.py`、`engine` 算法层、`factory.py`、`routes.py`、`crm_api.py`、`warning_export.py`、`config_sync.py`、`Dockerfile`、`docker-compose.yml` 等关键文件。下面是按多维度的评价。

---

## 总体一句话

> 这是一个**业务边界清晰、文档密度高、工程化程度明显高于一般项目**的中小规模生产级 Python 服务。已经具备「能上生产、能交接、能扩展」的素质；主要遗留问题是**少数核心文件过大**、**手写迁移与 `create_all` 并存的风险**、**无外部可观测性**这几项工程性短板。

整体我会给到 **B+ ~ A-**。

---

## 1. 架构 / 模块化（A-）

**亮点**

- 「共享基础设施 + 功能垂直切片」的目录结构非常清晰：`app/shared/`（CRM、auth、middleware、db）+ `app/churn/`（功能切片），`AGENTS.md` 还显式约定了未来新增 feature 的目录骨架（`config/engine/service/repository/...`）。
- 严格的分层：`routes → service → engine + repository`。`routes.py` 只做协议转换/错误映射，业务逻辑都在 service 和 strategy 里，能在不同入口（FastAPI、Celery）复用。
- `engine.py` 是兼容导出层，真正的检测拆到了 `app/churn/anomaly_strategies/{frequency,volume,product,support}.py`，每种异常一个文件，新增异常类型成本低。
- 工厂层 `factory.py` 用 `lru_cache(maxsize=1)` 做服务单例，并提供 `clear_cached_service()` 给测试用——这是非常正确的生产/测试解耦写法。

**问题**

- `app/churn/service.py` **1323 行**，`mysql_repo.py` **1943 行**，`tests/churn/test_api_and_factory.py` **1545 行** —— 这是项目里最大的三个「上帝模块」。
  - `WarningService` 同时承担了「Job 编排 / 配置优先级解析 / AI 上下文构建 / 状态流转 / 持久化协调」五种职责。可以再拆出 `WarningConfigService`、`AiAnalysisService`，让 `WarningService` 只做检测编排。
  - `MySQLChurnRepository` 把 7 张表的 ORM、查询优化、分块读取、UPSERT、列表分页、AI 缓存全塞在一个文件，建议至少按表/职责拆成 `warnings/jobs/runs/configs/ai/lookup` 子模块。
- `engine.py` 只剩 28 行兼容壳，但还在被 `service.py` 使用——属于过渡期遗留，可以排期清掉。

---

## 2. 领域建模与算法（A）

**亮点**

- 三类异常的口径在 `technical-design.md` 第 10 节有非常完整的数学描述（EWMA、IQR、置信度、严重度分级、业务日历），算法和文档**对齐度高**，这点比绝大多数同类项目要好。
- `predict_cycle` 用了：
  - 同日数量合并 + IQR 异常间隔过滤（`_remove_iqr_outliers`）
  - 指数权重 EWMA（`weights[i] = 2^i`）
  - 置信度 = 0.45·样本分 + 0.55·稳定分（基于变异系数）
  
  这个组合简单、稳定、不依赖 numpy/pandas，适合纯 Python 场景。
- `volume` 检测的**全额退款反向匹配**（`_match_fully_refunded_orders`，按客户+产品+数量+金额匹配最近一笔正向单）是真实业务里很容易被忽略的细节，做得到位，且在 `feature_summary` 里保留 `refunded_order_matches` / `unmatched_refund_order_ids` 等可解释字段。
- 业务日历支持「法定节假日 + 春节制造业停工 + 自定义停工 + 调休工作日」四类，并区分「自然天数」和「有效采购天数」——这是 B2B 流失场景最容易被忽视但很关键的一层。
- 去重设计严谨：`(customer_id, anomaly_type, dedup_key, active_slot)` 唯一约束 + `active_slot=NULL` 只对 resolved 生效，允许多个历史 resolved 共存——这是利用 MySQL 唯一约束语义的标准技巧。

**可改进**

- `_ewma` 的权重选择是 `2^i`，权重衰减很激进，本质上几乎只取最近 1~2 个间隔。如果客户采购节奏波动大，会导致预测偏向「最近的尖刺」。可以暴露一个 `ewma_alpha` 配置或者用经典 EWMA 公式 `α=2/(N+1)`，让运营可以软调。
- `_prediction_confidence` 的 0.45/0.55 权重和 6 条满分阈值是硬编码魔术数，可以提到 `ChurnSettings`。
- `volume` 严重度阈值（`60% / 40%`、`100% / 60%`）也是硬编码；`product` 只有触发倍数和占比阈值能配，下游业务方不能自己调严重度边界。

---

## 3. 数据访问与持久化（B+）

**亮点**

- `CrmRepository` / `ChurnRepository` 抽象接口 + 三种实现（`memory / api / mysql`）→ 算法层完全不知道数据来自哪里。这是这套架构能跑测试又能上生产的关键。
- MySQL 仓储里几个细节做得很好：
  - 分块 `SUBSTRING` 读取 `MEDIUMTEXT`（绕过部分驱动/网络下大文本不稳定）。
  - 列表分页采用 **seek-based cursor**，避免深分页大 offset 扫描。
  - 索引设计有针对性：`ix_warning_dedup_lookup`、`ix_warning_records_active_customer_type_status_date`、`ix_warning_records_status_biz_date_desc_warning_id_asc` 等，从字段顺序看就是按「活跃去重 / 列表排序 / 待处理列表」三类查询定制的。
  - MySQL `GET_LOCK` 做并发临界区，并在释放后 dispose 连接，避免带锁连接被池复用。
- `CrmApiRepository` 的分页防错很扎实：会校验后续页是否前进、是否有重复 row key、最终条数和唯一 row key 是否等于 `total`，任何一项不满足直接报错——避免「静默丢单/重复单」是 CRM 对接最痛的坑。

**问题**

- `MySQLChurnRepository.__init__` 里 `Base.metadata.create_all(self._engine)` 与 `migrations/mysql/churn/000~015.sql` 手写迁移共存（`technical-design.md` 23.5 自己也坦白了这是风险）。在生产首启或新增表时，可能出现「create_all 创建了 ORM 默认表，但漏掉手写迁移里的索引/约束/回填」，导致索引/约束「看起来有表但跑得慢」。建议生产环境把 `create_all` 关掉（用 env flag 控制），或彻底切到 。
- 没有 Alembic/版本表，靠人手记录脚本编号执行，迁移指南 (`docs/migration-guide.md`) 写得再好，长期也容易出现「测试库执行了，生产库忘执行」的故事。
- `_TEXT_CHUNK_SIZE = 800` 这种「为了绕开某种网络/驱动问题」的兼容性代码，建议在源码上方注释清楚是哪个具体 issue/环境触发的，否则三年后没人敢删。

---

## 4. API 设计 / 契约（A-）

**亮点**

- 端点拆分清晰，资源命名一致（`/api/warning-jobs`、`/api/warnings`、`/api/warning-configs/global`、`/api/detection-runs`），完全 RESTful。
- 同一个列表既支持 GET（query params）也支持 POST（结构化体），并且**显式标注游标和 offset 互斥**——这是真打过分页坑的项目才会做的事。
- 错误映射统一：`ValueError → 400`、`KeyError → 404`、`InvalidWarningStateError → 409`、`ChurnAiAnalysisNotConfigured → 503`、`ChurnAiAnalysisError → 502`、`WarningExportError → 502`，含义稳定。
- `response_model_exclude_none=True` 在 `/api/warnings` 上做按 `anomaly_type` 的稀疏字段返回，前端拿到的不是「全字段+一堆 null」。
- `/api/warning-configs/global?recompute=true` 同时返回 `recompute_tasks`（包含 task_id / 派发失败原因），同步操作和异步派发耦合得很轻。

**可改进**

- `/healthz` 只检查 FastAPI 是否能响应，**不检查 MySQL/Redis/CRM API/DeepSeek**。文档里也提到了「应该新增 `/readyz`」，建议尽早补上，否则容器编排做就绪探针会失真。
- `routes.py` 中部分路由的注册顺序里 `/api/warnings/{warning_id}` 放在 `/api/warnings/pending-enrich` 等具体路径之后，FastAPI 是按声明顺序匹配的，这个顺序目前正确，但建议加注释或重排，避免后续不小心被覆盖。
- 没有 OpenAPI 标签分组（`tags=["churn"]` 是统一打的），按业务/管理/AI 分组在 `/docs` 上会更易读。

---

## 5. 配置 / 可运维性（B+）

**亮点**

- `ChurnSettings` 是 `dataclass(slots=True)`，`__post_init__` 里逐字段做边界校验（正数、`(0,1]`、停工窗口字符串语法）。配置错误**在启动期就抛**，这比运行时崩好得多。
- `app.main._validate_startup_config()` 强制生产环境必须 `REPOSITORY_TYPE in {mysql, api}`、`CHURN_REPOSITORY_TYPE=mysql`、`DATABASE_URL` 和 `API_KEY` 必填——并且在 `_warm_up_services()` 里直接构建一次 `WarningService`，配置错误立刻在容器启动阶段失败，不会等到第一个请求才暴露。
- `docs/runbook.md` 已经覆盖到上线检查、灰度、回滚、巡检、6 种常见故障的处理步骤——这是很多团队后期才补的东西。
- 三个 CRM 默认 URL（订单/客户/拜访/预警同步/预警导出）在 `config.py` 和 `factory.py` 都有 default value，单机本地起服务配置很轻。

**问题**

- 配置入口分散：`os.getenv` 直接散落在 `factory.py`、`crm_api.py`、`mysql_repo.py`、`celery_app.py`、`db.py`，没有统一收口到 `ChurnSettings`。`ChurnSettings` 主要管业务配置，但「连接超时」「pool_recycle」「Celery concurrency」「nginx 端口」散落多处，调一个值要查多个文件。
- `factory.py` 中默认硬编码 `https://crm.amer.cn/...`，对开源/复用是反模式（vendor lock-in），开源版应该把这些默认值留空或放到一个 `crm_defaults.py`。
- `lru_cache(maxsize=1)` 的服务缓存依赖进程内单例，**前提是配置一辈子不变**。Celery worker 和 API 进程各自 lru_cache 各自结果。如果哪天改成 K8s reload env 不重启进程，缓存不会自然失效。可以加一个内部 `/admin/reload-config` 或基于配置版本号失效。
- 无 `/readyz`、无 metrics endpoint（这条上面也提了），生产监控目前只能靠日志 + 业务表轮询。

---

## 6. 安全（B-）

**亮点**

- API Key 走 `X-API-Key` header，路由层 `dependencies=[Depends(require_api_key)]` 集中挂载，遗漏成本低。
- 生产强制 `API_KEY` 必填。
- `deploy.sh` 自动生成 API_KEY，避免出现「弱默认 key 被部署到线上」的事故。

**问题 / 风险**

- 只有**单一 API Key**：没有 user/role/scope 概念，没有按销售或管理员区分权限，任何持 key 的客户端都可以读所有客户的预警。如果前端要给销售用，需要再加一层网关或细粒度权限。
- **没有限流**：单 key 单端点都没限制，AI 分析（`POST /api/warnings/ai-analysis`）一旦被滥用会直接导致 DeepSeek 账单飞涨。
- `CORS_ORIGINS` 默认 `*` + `allow_credentials=True` ——这是**违反 CORS 规范**的组合（浏览器会拒绝带 cookie 的跨域请求，但更重要的是这个搭配本身就是配置异味），生产应该强制具名域名。
- `compute_crm_passport`：MD5(`YYYY-MM-DD + "amer_fl_output"`) 是和 CRM 协商的固定算法，但**密钥常量直接写死在源码**里。如果代码泄露 == passport 算法泄露。建议把盐值挪到 env，方便 vendor 侧也跟着轮换。
- AI 分析会把客户订单/拜访摘要发到外部 DeepSeek，`technical-design.md` 14 节里提到了，但代码里没有显式的数据脱敏选项（比如客户名/产品名脱敏）。如果客户合规要求高，需要补。
- `.env` 出现在仓库目录里（git status 没显示 `??` 也没显示 `M`，需要确认 `.gitignore` 真把它排除了——快速检查可参考 `.gitignore`）。

---

## 7. 测试（A-）

**亮点**

- **245 个测试，0.96 秒跑完**，速度本身就是设计质量的体现：所有测试都跑在内存仓储上，没有真依赖 MySQL/Celery/CRM。
- 覆盖按模块切：算法层（`test_engine.py` 702 行）、服务编排（`test_service.py` 1144 行）、API+工厂（`test_api_and_factory.py` 1545 行）、MySQL 仓储（`test_mysql_repo.py` 1422 行）、CRM API（`test_crm_api.py` 694 行）、business calendar、auth、middleware、warning_keys、config_sync、warning_export、job_dates 全都有。
- `WarningService` 测试要求分别构造 `InMemoryCrmRepository` 和 `InMemoryChurnRepository`（`AGENTS.md` 第 36 行写明了）—— 这是测试纪律，不是巧合。

**可改进**

- 测试文件普遍偏大，最长的 `test_api_and_factory.py` 1545 行。可以按场景拆（比如 `test_warnings_listing.py`、`test_ai_analysis.py`、`test_warning_config.py`），便于定位失败用例。
- 缺少 **基于 testcontainers 的 MySQL 集成测试**。`test_mysql_repo.py` 走 SQLAlchemy 但用的是 SQLite 还是 MySQL 没看到 fixture（如果是 SQLite，则 `MEDIUMTEXT`、`GET_LOCK`、`UNIQUE 多 NULL` 这些 MySQL 特有语义就跑不到）。
- 仓库根目录还有一个 `test_crm_api.py`（不是 `tests/` 下），是手写的真 API 集成脚本，建议挪到 `scripts/smoke_crm_api.py` 或 `tests/integration/`，避免被 pytest 误收（虽然现在 `[tool.pytest.ini_options].testpaths = ["tests"]` 排除了它，但目录污染感还在）。

---

## 8. 文档（A）

**亮点**

- `README.md` 简洁，主要做总览/启动；
- `docs/technical-design.md` **24 章 1500+ 行**，覆盖技术栈、架构、模块职责、算法口径、CRM 接入、Celery 调度、MySQL 设计、迁移、部署、环境变量、测试、可观测性、风险、扩展约定——完全可以作为「人离职后接手的唯一文档」。
- `docs/runbook.md` 写到「现象 → 排查 → 处理」3 段式 SOP，事件记录模板都给了。
- `docs/migration-guide.md` 区分新库和老库的执行顺序。
- `AGENTS.md` 给 AI 编码助手单独做上下文，并指明项目结构、命名约定、测试要求——这是让 AI 协作不跑偏的关键。
- `CHANGELOG.md` 区分 `Breaking / Added / Changed / Fixed`，semver 思路清晰。

**问题**

- `docs/api-guide.md` 没读到（git status 显示存在，但我没强制打开），如果它和路由的最新签名脱节就比较麻烦——建议未来用 `pytest --collect-only` + 自动生成 OpenAPI 比对作 CI 检查。
- `docs/technical-design.md` 单文件 1500+ 行已经有点超出阅读舒适区，可以拆成 `01-architecture.md`、`02-algorithms.md`、`03-persistence.md` 等。

---

## 9. 性能（B+）

**亮点**

- N+1 已规避：`get_orders_for_customers` 批量加载、`_preload_warning_settings` 预加载客户配置和全局配置，避免在每个客户循环里再去查数据库。
- CRM API 并发：`CRM_API_MAX_PARALLEL_SALES_FETCHES` + `CRM_API_MAX_PARALLEL_PAGE_FETCHES` 两层并发可调。
- 列表查询两阶段：先按过滤条件查 `warning_id`，再按 ID 拉轻量列；不加载大文本字段（`feature_summary` / `metadata` / `sales_message`），列表页瘦身。
- MySQL 索引按真实查询路径设计（已在「数据访问」一节展开）。

**风险点**

- `ThreadPoolExecutor` 用于 IO 并发是合理的，但 `httpx.Client` 没有启用 keep-alive 连接池（`httpx.Client` 默认有，但 `crm_api.py` 是不是每次新建 client 需要确认；`warning_export.py` 用的是 `httpx.post(url, ...)` 模块级函数，**每次都新建连接**——批量推送预警时会重复 TLS 握手，规模大时会成为瓶颈）。建议改成长连接 client，或者至少在 `CrmWarningExporter` 里持有 `httpx.Client`。
- AI 分析 (`DeepSeekChurnAnalyzer.analyze`) 每次新建一个 `httpx.Client`，同上。
- Celery 当前**所有任务共用默认队列**（`technical-design.md` 15.2 自己也提到了），AI 分析任务（耗时长、成功率受外部 API 影响）和 detection 任务（耗时短、稳定）混在一起，AI 阻塞会拖死定时检测。
- `MySQLChurnRepository.__init__` 调用 `Base.metadata.create_all`，每个 process boot 都要 introspect schema，进程冷启动多一点开销（小，但稍微脏）。

---

## 10. 部署 / 生产化（A-）

**亮点**

- Dockerfile 简洁，固定基础镜像 `python:3.11-slim`，使用清华 PyPI 镜像加速国内构建。
- `docker-compose.yml` 把 `nginx / redis / api / celery` 一把起齐，`depends_on` + redis healthcheck 保证启动顺序正确。
- `deploy.sh` 在没有 `.env` 时自动从 `.env.example` 复制并生成 API_KEY，把 placeholder 检查作为硬性 gate——这是把「上线前必须改的事」做成代码。
- `nginx` 容器只暴露 HTTP，明确说明 HTTPS 由外部反代承担，这是更现实的部署假设（避免在容器里搞证书）。

**问题**

- `Dockerfile` 中 `COPY . .` 会把 `.git`、`.env`、`tests` 都打进镜像（虽然 `.dockerignore` 存在，但需要确认它包含了关键路径）。生产镜像应该只装运行所需。
- `Dockerfile` 没用 multi-stage build；`pip install --no-cache-dir .[all]` 会把 dev 依赖也带进生产镜像。可以拆成 `builder` + `runtime` 两阶段，最终镜像小且不含 pip 缓存。
- 容器**没用非 root 用户运行**（缺 `USER appuser`），并且没显式设 `WORKDIR` 之外的安全约束。
- `docker-compose.yml` 里 `celery` 服务把 worker 和 beat 放一起（`--beat`）。文档第 15.2 节也提醒了：扩容到多 worker 时必须只跑一个 beat，但 compose 没有内置防呆。建议在 README/runbook 里贴「扩容路线」的具体 compose override 示例。
- 没有看到 `restart: unless-stopped` / `restart: always` 之外的 `deploy.resources.limits`，单机被打爆会传染整个 compose。

---

## 11. 代码质量与一致性（A-）

**亮点**

- 全项目 `from __future__ import annotations` 一致开启，类型标注覆盖率高。
- `Protocol`、`@dataclass(slots=True)`、`replace` 用得很自然，体现了对现代 Python 的熟练度。
- 命名一致（snake_case / PascalCase / UPPER_SNAKE_CASE 均按 `AGENTS.md` 约定）。
- 几乎没看到重复代码（除了一些数据访问层的样板代码，这种很难避免）。
- 提交信息看起来是个小团队/个人作者维护，主题清晰（「solve empty employee_code」、「add crm api」、「Refactor app structure」），符合 `AGENTS.md` 里的提交规范。

**问题**

- 没有配置 `ruff` / `black` / `mypy` / `pre-commit`。`pyproject.toml` 的 `[tool.*]` 段只有 `pytest`，缺少格式化和静态检查的统一约定，长期容易出现风格漂移。
- `tests/__pycache__/` 和 `app/**/__pycache__/` 大量出现在 git status 的未跟踪文件里——`.gitignore` 应该已经忽略，但仓库里出现 `churn_warning_engine.egg-info/` 也是未跟踪状态，说明 `.gitignore` 不够全。
- `service.py` 部分方法已经超过 60 行 + 5 个参数，例如 `_warning_effective_config_payload`、`_settings_for_warning`、`detect_batch`，可以再切。

---

## 12. 主要风险 & 改进建议（按 ROI 排序）

| 优先级 | 项目 | 原因 |
|---|---|---|
| P0 | 加 `/readyz` 检查 MySQL/Redis/CRM API | 当前 `/healthz` 不能反映真实就绪，K8s/反代探针会误判 |
| P0 | 切到 Alembic（或至少做 schema_migrations 版本表）+ 关闭生产的 `Base.metadata.create_all` | 当前手写 SQL + create_all 并存是潜在静默故障源 |
| P0 | `CORS_ORIGINS` 在生产强制白名单，禁掉 `*` + `allow_credentials=True` 的组合 | 这是显式漏洞 |
| P1 | 拆分 `service.py` 和 `mysql_repo.py` | 1300+/1900+ 行单文件，长期维护痛 |
| P1 | 拆 Celery 队列：`churn_detection` vs `churn_ai` | 防止 AI 抖动拖死定时检测 |
| P1 | API Key 升级到「多 key + scope」或最少加限流（slowapi 或网关） | 保护 AI 端点和成本边界 |
| P1 | `httpx.Client` 改为长连接复用（在 `CrmWarningExporter` / `DeepSeekChurnAnalyzer` 里持有 client） | 高频导出/AI 时握手开销可观 |
| P2 | 增加 Prometheus 指标 + 检测耗时/CRM/AI/MySQL 分维度埋点 | 当前可观测性只有日志 |
| P2 | 多阶段 Dockerfile + 非 root 用户 + 完善 `.dockerignore` | 镜像体积/安全 |
| P2 | 引入 `ruff` + `pre-commit`，pytest 加覆盖率门槛 | 防止风格/质量漂移 |
| P3 | 严重度阈值/EWMA 衰减系数等魔术数提到 `ChurnSettings` | 让运营能调，不必改代码 |
| P3 | 把 `crm.amer.cn` 默认 URL 抽出为可选环境变量、源码里默认空字符串 | 开源/复用更友好 |
| P3 | 把 `passport` 盐值挪到 env | 减少代码泄露 = 协议泄露的风险 |
| P3 | `docs/technical-design.md` 拆分 | 单文件已超出阅读舒适区 |

---

## 13. 项目画像（一句总结每个维度）

| 维度 | 评分 | 一句话 |
|---|---|---|
| 架构与模块化 | A- | 切片清晰，但 service / mysql_repo 已经「太胖」 |
| 领域建模与算法 | A | EWMA + IQR + 业务日历 + 退款匹配，业务洞察足 |
| 数据访问与持久化 | B+ | 索引、分页、并发锁都做了，但迁移机制是阿喀琉斯之踵 |
| API 设计 | A- | RESTful、错误映射、稀疏字段、双分页都到位，缺 `/readyz` |
| 配置/运维 | B+ | 启动校验扎实，但配置入口分散、缺 metrics |
| 安全 | B- | 单 API Key + CORS `*` + 无限流，能用但不够生产 |
| 测试 | A- | 245 个用例 < 1s，纪律好；但缺真 MySQL 集成 |
| 文档 | A | technical-design + runbook + migration-guide 三件套齐全 |
| 性能 | B+ | 批处理/缓存/索引到位，httpx 短连接是个洞 |
| 部署 | A- | docker-compose 一把启动，但镜像与多实例策略可强化 |
| 代码质量 | A- | 类型标注 + Protocol + dataclass 用得熟，但缺 lint 工具链 |

如果你想，我可以挑其中**任意 1~2 项**深入做一份具体改造方案（比如「Alembic 接管迁移」「拆 service.py」「加 `/readyz` 和 Prometheus 指标」），你说一声就行。