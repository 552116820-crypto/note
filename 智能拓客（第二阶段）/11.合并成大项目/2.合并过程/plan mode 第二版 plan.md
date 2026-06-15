# 深度合并 v2.2.0：共享层去重 + 脚手架 + 开发文档（rev2，经 Codex 审查修订）

## Context

2026-06-11 churn（流失预警）+ lead（智能拓客）已机械合并进 /mnt/e/amer_sales_assist（524 测试全绿、docker 四服务冒烟通过），但只是"并排放"：

- **真重复**：CRM passport md5 两份（`shared/crm_passport.py` vs `lead/clients/crm.py:compute_passport`）；日期工具两份（`shared/date_utils` vs lead `crm_today`）；lead 内部 `db/pool.py`（193 行）与 `db/log_pool.py`（157 行）为合计 ~350 行的近似拷贝。
- **lead 独有但属平台级**：错误 envelope 体系、TTL/Redis 缓存（singleflight）、DeepSeek 客户端、请求体限制中间件——未来新模块都需要，应上提 `app/shared/`。
- **脚手架空白**：无 git、无 ruff、无 Makefile、无 pre-commit；`/healthz` 的 lead db ping 无独立超时（DB 不可达最坏 ~41s：3 次重试 × 10s connect timeout + 每次失败后退避 sleep 2+4+5，含末次失败后的无意义尾部 sleep）。
- **文档缺口**：无平台级架构/开发/模块开发指南；docs/lead/README.md 多处过时（见阶段 4）。

目标：去重、低耦合、脚手架、开发文档，为后续功能模块化开发铺路。

**用户已拍板**：① 先 git init 锁基线（仅本地，不建远程不推送）；② 本轮不动 churn 内部结构；③ ruff 全库一次性格式化。
**本计划已经 Codex 只读审查并修订**（修正事实错误 6 处、补盲点 7 处、消歧 5 处）。

## 硬约束（全程不可违反）

1. `app/shared/` 永远不 import `app.lead.*` / `app.churn.*`——上提模块全部改构造注入，lead 侧保留薄装配文件传 settings 值。
2. 不留 shim：lead 的 infra 文件重写为"真实装配内容"（域异常子类、业务 key builder、单例实例化），不做纯转发层。
3. churn 业务代码零改动（仅 ruff 格式化触碰）；shared 现有 12 个模块的符号位置与签名不变（churn 13 处 import 锁死）。**唯一允许的例外**：`compute_crm_passport` 追加向后兼容的 keyword-only `salt` 参数（churn 两处调用与 tests/shared/test_crm_passport.py 零影响）。
4. 61 个既有 env 变量名一个不改（生产切换要直接复用旧 .env；含 `_flag()` 类布尔键如 CRM_HTTP_TRUST_ENV/EMPLOYEE_NOT_FOUND_404/ENABLE_API_LOGGING）；新增 key 必须带默认值。
5. Redis 分库 db0(broker)/db1(result)/db2(lead 缓存) 不动；celery 命令 `-s /tmp/celerybeat-schedule` 保留；API 契约不变（churn 422/500 vs lead 200-envelope）。
6. 524 测试基线恒等：搬家允许（lead 减 = shared 增），断言行为不改。
7. 本机验证：系统 Python 3.12 直跑 pytest；curl 本地端口必须 `--noproxy '*'`（WSL 代理假 502）。

## 阶段 0 — git 基线（1 commit）

`git init` → 确认 `git status` 不含 `.env`/`.pytest_cache`/`*.egg-info`（现 .gitignore 已覆盖）→ `git add -A` → commit `chore: baseline — merged churn + lead, 524 tests green (v2.1.0)`。
验证：`python3 -m pytest -q` 524 绿；`git log --stat` 抽查无密钥。

## 阶段 1 — ruff 引入 + 全库清洗（2 commit：格式与修复分离）

`pyproject.toml`：
- `[dev]` extra 加 `ruff>=0.14,<0.15`（pin 与 pre-commit rev 对齐）。
- `[tool.ruff]` line-length=110（存量 p99=89、超 110 仅 ~22 行）、target-version="py310"。
- `[tool.ruff.lint]` select = E,F,W,I,UP,B,C4,SIM,PERF,BLE；ignore E501。选 PERF/BLE 是因为存量已有 `# noqa: PERF203`/`BLE001`。per-file-ignores: `tests/**` = BLE001,SIM117。isort known-first-party=["app","tests"]。
- **Dockerfile 同步改**：`pip install --prefix=/install .[all]` → `.[mysql,celery]`——否则 ruff（连同现状的 pytest）进生产镜像依赖闭包（Codex 盲点 #7）。rebuild + 冒烟验证。

执行（两个独立 commit，便于审查）：
- **1a 纯格式**：`pip install --user ruff` → `ruff format app tests scripts` → 全量绿 → commit `chore: ruff format entire repo (pure formatting)`。
- **1b lint 修复**：`ruff check --fix`（仅 safe fixes）→ 残余手工修，**B/SIM 等涉行为规则的修复逐条人工确认** → ruff check 0 错 → 全量绿 + `docker build` 过 → commit `chore: ruff lint fixes + dockerfile installs runtime extras only`。

## 阶段 2 — shared 上提（4 个原子 commit）

**布局决策：shared 保持平铺**（不建子包）。硬原因：`shared/db.py` 已存在，`shared/db/` 同名包冲突；churn 13 处 import 锁死现有模块名。14 → 19 个单一职责模块仍可读。

### 2a errors + middleware

- 新增 `app/shared/errors.py`：从 `lead/errors.py` 原样上提 `envelope()` + `DomainError` 基类（code/message/http_status + `__init__`）。**7 个业务子类**（EmployeeNotFound/InvalidCoordinates/TooManyCandidatesError/InvalidCustomerId/DBUnavailableError/LogDBUnavailableError/InternalError）不上提。
- 扩充 `app/shared/middleware.py`：上提 `RequestBodyLimitMiddleware`（含 `RequestBodyTooLarge`/`_content_length`/`_send_413`），构造签名改 `__init__(self, app, max_bytes: int, *, code: int = 1002)`——max_bytes 必填（删 settings 缺省）、413 业务码改可配置（缺省 1002 = 现行为，消除 lead 专属语义硬编码，Codex 盲点 #6）；`_send_413` 用 `app.shared.errors.envelope`（shared 内部依赖合法）。
- `lead/errors.py` 瘦身：`from app.shared.errors import DomainError` + 7 个子类；删 `lead/middleware/request_size.py`。
- `app/main.py`：envelope 改自 `app.shared.errors`（main.py:27 import 块拆分，DomainError 子类仍来自 lead）；**新增 `from app.lead.config.settings import settings as lead_settings`**；`_LeadScopedBodyLimit`（main.py:141）改继承 shared 类，`add_middleware(_LeadScopedBodyLimit, max_bytes=lead_settings.max_request_body_bytes)`；`lead/routers/leads.py:8`、`customer_detail.py:10` 的 envelope import 同改。
- **决策注记（Codex 盲点 #1）**：main.py:212 的 `DomainError` handler 维持全局（不加 LEAD_PREFIX 分流）——envelope 即平台契约，凡采用 `shared.errors` 的模块自动获得 200-envelope 语义；churn 不抛 DomainError，行为不受影响。此约定写进 architecture.md 与 module-guide.md。
- 测试：`test_request_body_limit.py` 拆分——纯 ASGI 用例搬 `tests/shared/`，HTTP envelope 用例留 tests/lead。

### 2b cache

- 新增 `app/shared/cache.py`：上提 `_LOCK_STRIPES`、`InProcessTTLCache`（已纯参数注入，原样）、`_RELEASE_LUA`、`RedisTTLCache`——构造改 `__init__(self, url, ttl, *, lock_ttl_seconds=20, lock_wait_seconds=15.0, lock_poll_ms=50)`，体内 3 处 `settings.cache_lock_*` 改读构造参数；**缺省值必须等于 settings.py:67-69 的字段缺省**（保住 `test_redis_singleflight` 裸构造 `RedisTTLCache(url, ttl=60)` 语义）。`_build_cache` 改名 `build_cache(ttl, *, redis_url="", max_keys=…, lock_*…, label="cache")`，**ping+回退逻辑原样（store.py:216-225）：Redis 不可达回退进程内缓存、绝不阻断 import**。
- `lead/cache/store.py` 重写（~60 行）：保留 3 个业务 key builder（仍读 settings.score_version/reason_model_version）+ `cache`/`detail_cache` 两单例 = `build_cache(settings.…)` 全参传入。单例仍在模块 import 时构建，不挪 lifespan。
- 测试：conftest.py:16 等 4 处 `InProcessTTLCache` import 改 shared；`test_capacity_cache.py`→`tests/shared/test_cache_capacity.py`、`test_redis_singleflight.py`→`tests/shared/test_cache_singleflight.py`。

### 2c db 统一 + healthz fail-fast

- 新增 `app/shared/db_mysql.py`（替代合计 ~350 行的两份拷贝）：`MySQLUnavailable(RuntimeError)`；`@dataclass(frozen=True) MySQLConfig(host, port, user, password, database, connect_timeout, read_timeout, write_timeout, connect_retries, retry_sleep_max=5)`；`MySQLDatabase(config, *, unavailable_exc=MySQLUnavailable, label="DB")` 含：
  - `_connect_once(可覆写超时)`/`connect_with_retry`/`_map_query_error`/`_SafeCursor`/`_SafeConnection`（pool.py 逐行照搬，异常类取自实例）/`get_conn`/`fetch_all`/`fetch_one`/`execute`；
  - `connect_with_retry` **末次失败后不再 sleep**（修正现实现的无意义尾部 sleep；纯加速失败，调用方语义不变）；
  - `fetch_all_fast(sql, args, *, connect_timeout, read_timeout)` **保持"单次直连 `_connect_once`、无重试、失败上抛"语义**——industry_resolver 启动降级（industry_resolver.py:52-60）依赖此契约（Codex 歧义 #3）；
  - `ping(*, connect_timeout=None, read_timeout=None, retries=None) -> bool`：**返回 bool、吞所有异常、绝不上抛**（healthz "DB 挂仍 200 + body.db=false" 契约，Codex 歧义 #2）。
- `lead/db/pool.py` 重写（~40 行）：`DBUnavailable(MySQLUnavailable)` + `_db = MySQLDatabase(MySQLConfig(settings.db_*…), unavailable_exc=DBUnavailable, label="DB")` + 模块级绑定 `get_conn/fetch_all/fetch_one/fetch_all_fast`；`ping()` **保持零参签名**——共 **5 处**测试 monkeypatch `app.main.ping`（tests/lead/test_healthz.py:10,28、tests/shared/test_middleware.py:11,19、tests/shared/test_auth.py:12）——内部传 `connect_timeout=read_timeout=settings.healthz_db_ping_timeout, retries=1`。**时延论证**：单次尝试 + 3s 超时 + 无尾部 sleep → DB 不可达最坏 ~3s，落入 compose healthcheck `urlopen timeout=4` / `timeout: 5s` 窗口（docker-compose.yml:43-47），不再假 unhealthy。
- `lead/db/log_pool.py` 重写（~30 行）：`LogDBUnavailable(MySQLUnavailable)` + 实例（`retry_sleep_max=3`，label="LOG_DB"）+ 绑定 `get_conn/execute/fetch_all`。
- `lead/config/settings.py` 加新可选 key：`healthz_db_ping_timeout = int(os.getenv("HEALTHZ_DB_PING_TIMEOUT", "3"))`；`.env.example` lead DB 分节加注释行。
- 测试：`test_db_query_error_mapping.py` 重写为 `tests/shared/test_db_mysql.py`——**query-stage 错误映射用例（execute/fetchall/executemany → unavailable_exc）必须 6 用例一一对应保留**（现 test_db_query_error_mapping.py:55-95），模块级 patch `connect_with_retry` 改 patch 实例方法；加 1 个 ping fail-fast 用例（断言单次连接 + 超时 kwargs + 无 sleep）。

### 2d llm + passport/date 归一 + load_env

- 新增 `app/shared/llm.py`：上提 `LLMError` + `DeepSeekClient`，构造 `__init__(self, *, base_url, api_key, model, timeout, temperature=0.2, max_tokens=2000)`，体内 settings.deepseek_* 改 self。**职责分布事实（Codex 更正）**：重试循环在 recommendation_service.generate_reason（service 层用 `settings.deepseek_max_retries`，recommendation_service.py:165-176），客户端本身无重试——上提保持此分布，shared 客户端不加重试。
- `lead/clients/deepseek.py` 重写（~12 行）：仅 `deepseek_client = DeepSeekClient(settings 全参)`；不再转发 LLMError。**LLMError 异常同一性**：service 捕获点（recommendation_service.py:22,173）与 2 个测试（test_recommendation_service.py:4、test_api_errors.py:11）必须同 commit 切到 `app.shared.llm`，收尾 `grep -rn "deepseek import" app tests` 验证旧路径零残留——否则 `except LLMError` 捕不到新客户端抛的异常（Codex 风险 #2）。
- `shared/date_utils.py` 追加 `today_in_tz(tz_name) -> date`；`shared/crm_passport.py` 签名改 `compute_crm_passport(reference_date=None, *, salt="amer_fl_output")`。
- `lead/clients/crm.py`：CRM 业务全保留（CRMClient/CRMSourceError/crm_client 单例）；`crm_today(tz=None)` 体改 `today_in_tz(tz or settings.crm_passport_tz)`；`compute_passport(today=None)` 体**必须**写成 `compute_crm_passport(today or crm_today(), salt=settings.crm_passport_salt)`——不能把 None 透传给 shared 缺省（那会绕过可配置的 CRM_PASSPORT_TZ，退化成固定 Asia/Shanghai，Codex 歧义 #1）；`date_bucket` 原样；删 hashlib/ZoneInfo import。归一后 `grep hashlib app/lead/clients/crm.py` 应为空。
- `shared/config.py` 追加 `load_env(dotenv_path=None)`（定位仓库根 .env、幂等加载）；settings.py 顶部 `_ENV_PATH`/`load_dotenv` 两行换成它。**同时在 `app/main.py` 与 `app/celery_app.py` 顶部显式调用 `load_env()`**（Codex 盲点 #4：现状 .env 加载靠"main 恰好先 import lead settings"的隐式时序，celery 进程根本不加载 .env；容器内无 .env 文件时 no-op，compose 注入优先，行为安全）。
- 测试：`test_crm_time_boundary.py` 的 `crm.datetime` monkeypatch 目标改 `app.shared.date_utils`。

**阶段 2 业务层零改动**：timeline_service、lead_service、industry_resolver、scoring、5 个 repo、schemas、api_log 均不动（app 侧 import 改动仅 main.py + 2 routers + recommendation_service + celery_app 共 5 文件）。

**阶段 2 收口验证**：全量 524 绿对账 → `docker compose build && up -d` → `curl --noproxy '*'` 打 8000/8080 healthz、lead score-and-info POST（envelope 形状）、churn pending-enrich（X-API-Key）→ `docker compose logs celery | grep beat`。

## 阶段 3 — 脚手架（1 commit）

- `Makefile`（无 venv → 一律 `python3 -m`）：install / test / test-churn / test-lead / test-shared / lint / format / run / celery（含 `-s /tmp/celerybeat-schedule`）/ docker-build / up / down / logs / smoke（curl 串**写死 `--noproxy '*'`**）。
- `.pre-commit-config.yaml`：astral-sh/ruff-pre-commit（rev 同 [dev] pin），ruff-check --fix + ruff-format；文档注明可选启用。
- `.dockerignore` 追加 Makefile/.pre-commit-config.yaml（卫生）。
- **不写** `scripts/new_module.py` 生成器：下个模块形态未知，模板文档足够，第三个模块落地后再固化。

## 阶段 4 — 文档 + 版本 2.2.0（1 commit）

新增：
- `docs/architecture.md`：模块化单体；依赖方向图（churn→shared←lead，模块间禁横向 import）；main.py 装配点（路由/异常前缀隔离/中间件栈顺序）+ celery 装配；**DomainError→200-envelope 为平台契约**（采用 shared.errors 即采用该语义）；数据面（amer_crm_v2 只读 / amer_sales_assist 可写）；Redis 分库铁律；双错误契约；shared 19 模块职责表 +"构造注入、不读业务 settings"规则。
- `docs/development.md`：环境（系统 Python3.12 直跑、无 venv）、make 全目标、ruff/pre-commit、测试布局与 524 基线、docker 冒烟（WSL `--noproxy '*'`）、commit 规范、.env 管理（61 个 key 不可改名、新增须带默认值并同步 .env.example）。
- `docs/module-guide.md`：新模块目录骨架（routers/services/repositories/schemas/errors.py/config/settings.py 模板 + 薄装配模式）；装配 checklist（main.py include_router + 异常处理器前缀、中间件作用域、celery beat + register_tasks、`load_env()`、.env 分节、tests/<module>/、migrations/mysql/<module>/、docs/<module>/、表前缀）；shared 使用守则。
- 根 `CLAUDE.md`：一句话定位、make 命令表、目录速览、五条铁律（churn 契约与 shared 符号不可动 / env 名不可改 / Redis 分库不可动 / shared 禁止 import 业务模块 / 524 基线与 `--noproxy` 冒烟）。

更新：
- README（目录表 + make 快速上手）；AGENTS.md（**删净"lead 暂不依赖 shared/去重留后续"旧表述**）。
- **`docs/lead/README.md` 修复过时内容**（Codex 盲点 #2/#3，不止加一行注记）：接口日志"待做"→已实现（api_log.py）；Redis 库 `/0`→`/2`（与 compose 对齐）；旧路径 `app/schemas/`、`app/config/` → `app/lead/schemas/`、`app/lead/config/`；另加"基础设施已上移 shared"注记。
- CHANGELOG 加 2.2.0 条目（含 healthz 行为变化：DB 挂时容器 healthy + body.db=false、~41s→~3s，监控口径按 body.db）；版本号 bump：`pyproject.toml` + `app/main.py:130` → 2.2.0。

## 阶段 5 — 终验

`docker compose down && up -d --build` → `make smoke` + 阶段 2 收口全部 curl → `git log --oneline` 核对 9 个 commit → 重构 diff 跑一次 /code-review 找回归。

## 本轮明确不做（记录在案，防丢失）

- `shared/crm_api.py:134` 用 `date.today()`（系统时区）初始化默认订单窗口——容器 TZ=UTC 时与业务时区在午夜前后可能差一天。属 churn 行为面（本轮 churn 零改动），归入后续 churn 轮次，届时换 `app_today()` 并核对 tests/shared/test_crm_api.py。
- churn 内部分层重构（routers/services/repositories 化）。
- mypy 类型检查引入。
- `scripts/new_module.py` 模块生成器。

## 验证手段汇总

- 全量：`cd /mnt/e/amer_sales_assist && python3 -m pytest -q`（恒等 524 passed；每次搬家后对账 lead 减 = shared 增）。
- 依赖可解性：`docker build`（本机无 venv，以镜像构建验证 pyproject；阶段 1 改 extras 后必须重验）。
- 冒烟：compose 起栈后 `curl --noproxy '*' -fsS http://127.0.0.1:8000/healthz` 与 8080 同款；lead POST score-and-info；churn GET 带 X-API-Key；celery beat 日志。
- 静态：`ruff check` / `ruff format --check`；`grep -rn "from app.lead" app/shared/` 必须为空（依赖方向）；2d 后 grep 验证 LLMError/hashlib 旧路径零残留。

## 关键风险

1. **singleflight 语义**（最高危）：`_singleflight` 锁协议（token/双检/deadline/_RELEASE_LUA）逐行照搬；RedisTTLCache 构造参数缺省必须 = settings 缺省（20/15.0/50）。
2. **单例 import 时序**：cache 两单例保持模块加载期构建（Redis 不可达回退、不阻断 import），不挪 lifespan。
3. **异常类同一性**：DBUnavailable/LogDBUnavailable/LLMError 的抛出点与捕获点（main.py handler、service except、测试 import）必须每个 commit 内同源切换。
4. **monkeypatch 路径**与对应 commit 同步：`app.main.ping` 5 处零参 patch（签名不可变）；test_db_query_error_mapping 重写；test_crm_time_boundary 改 patch shared。
5. **extras 边界**：db_mysql 不碰 SQLAlchemy；Dockerfile 改装 `.[mysql,celery]` 后 [mysql]/[celery]/[all] 结构不变，dev 件不进镜像。
6. healthz 行为变化是特性不是回归（~41s→~3s、容器不再假 unhealthy），文档与 CHANGELOG 注明。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Skill code-review：阶段 5 对重构 diff 做正确性审查（git 基线建立后 diff 可审）
- Skill codex:rescue：计划已审（本份 rev2 即审查产物）；执行中如遇疑难回归可再调一次二诊

**考虑过但不用**：
- verify / run：冒烟流程已知且固定（pytest + docker compose + curl --noproxy），直接 Bash 更可控
- simplify：本任务本身就是精简重构，终检用 code-review 找回归即可，不再叠加
- claude-api：涉及的 LLM 客户端是 DeepSeek（OpenAI 兼容、非 Anthropic），按该 skill 自身 SKIP 规则跳过
- init：CLAUDE.md 在阶段 4 手写，与重构后的新结构对齐，比按当前状态生成更准
