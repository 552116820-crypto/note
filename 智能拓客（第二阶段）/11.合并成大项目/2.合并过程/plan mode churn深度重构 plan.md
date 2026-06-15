# B 组：churn 内部重构（删死码 + 拆大文件 + shared 去 churn 化）

## Context

2026-06-13 Claude×Codex 联合 review 的 B 组收尾。A 组已"关标"（docstring/架构表如实标注 legacy churn CRM），B 组做物理收尾。用户已明确选择**全做（含解冻铁律 #2）**：把 churn 单一消费者的 7 个文件从 `app/shared/` 物理搬回 `app/churn/`，让 shared 真正只剩平台件；并删退役死码、拆两个大文件。

**核心前提**：这是纯 app 内部 import 重构，**不碰** .env / DB / Redis 分库 / celery 命令 / 既有环境变量名 / 200-envelope 契约 → 运行时行为零变化，494+13=507 测试全程兜底。**解冻的是铁律 #2**（churn↔shared 既有 import 红线）——这条红线本就是为保护"未搬迁"现状而存在，B3 是在消解它，须同步改写 CLAUDE.md。

执行顺序 **B1 → B3 → B2**：先删死码（缩小要搬/拆的面）、再整文件搬迁归位、最后在最终位置拆大文件（避免在 shared 里拆出 churn 碎片）。每个原子步骤后跑 `pytest -q && ruff check && mypy`，独立 commit。

> **本版已过 codex 两次独立审查**（首次 WSL 卡死、重启后 ~4min 完成），吸收并验证 3 项修订：① 测试 `monkeypatch` 字符串路径改写（19 处，见 B3）② 文档同步扩到 `handover.md` + `technical-design.md` ③ `factory` 全路径 6 处。codex 另确认无误：celery 任务名稳定（`app.churn.tasks.*` 不随搬迁变）、B1 死码范围准、`__init__` 无 re-export 负担、循环 import 风险低、`pyproject` `include=["app*"]` 无需改。

---

## B1 — 删 churn 退役死码（low risk，1 commit）

- `app/churn/models.py:75-83`：删 `WarningSortField` / `WarningSortOrder`（grep 实证无消费方，退役 list 端点遗留）。
    
- `app/churn/mysql_repo.py:577-583`：删 `_lock_name`（无调用；现役是 `_customer_lock_name`，按客户锁）。
    
- `app/churn/config.py`（行 48 字段 / 81-82 校验 / 155 env 解析）：`max_pending_enrich` 加 `# Deprecated: configured-but-unused since v2.3.0 HTTP retirement` 注释。**字段+校验+env 读取全保留**（删字段会破坏 `load_churn_settings`；`MAX_PENDING_ENRICH` env 名按铁律 #3 不可删）。同步在 `.env.example` 对应行加 deprecated 备注。
    
- **不动** `PENDING_ENRICH_STATUS`（活的状态常量，别误删）。
    
- 验证：`pytest -q`（507）+ ruff + mypy。
    

## B3 — shared 去 churn 化（解冻铁律 #2，搬 7 文件，~3 commits）

**搬迁映射**（`git mv` 保留历史；留 shared 的 `date_utils/config/crm_passport` 不动）：

|源（app/shared/）|目标（app/churn/）|备注|
|---|---|---|
|`crm_api.py`|`crm_api.py`|同名|
|`crm_mysql.py`|`crm_mysql.py`|同名|
|`crm_memory.py`|`crm_memory.py`|同名|
|`crm_repository.py`|`crm_repository.py`|同名|
|`models.py`|**`crm_models.py`**|改名：避撞 `churn/models.py`(WarningRecord)|
|`db.py`|**`sqlalchemy_engine.py`**|改名：避撞 `churn/engine.py`(检测引擎)|
|`business_calendar.py`|`business_calendar.py`|同名|

**import 改写模式**（`from app.shared.X` → `from app.churn.X`，仅对搬迁符号；`date_utils/config/crm_passport` 保持 `app.shared`）：

- churn 业务侧（代表路径）：`anomaly_strategies/{frequency,product,refund_matching,support,volume}.py` + `service.py` 的 `app.shared.models` → `app.churn.crm_models`；**`app/churn/factory.py` 6 处**（`:12,13` 顶层 import + `:34,35,39,104` 函数内 lazy import）的 `crm_api/crm_mysql/crm_memory/crm_repository`→`app.churn.*`、`db`→`app.churn.sqlalchemy_engine`；`config.py` 的 `business_calendar`→`app.churn`。
    
- 搬迁文件内部互引：`crm_{api,mysql,memory}.py` + `crm_repository.py` 里 `app.shared.models`→`app.churn.crm_models`、`app.shared.crm_repository`→`app.churn.crm_repository`；`sqlalchemy_engine.py` 的 `app.shared.config` 保持不变；`crm_api.py` 的 `app.shared.crm_passport`/`date_utils` 保持不变。
    
- 测试搬迁：`tests/shared/{test_crm_api,test_models,test_business_calendar}.py` → `tests/churn/`（`test_models.py` import 改 `app.churn.crm_models`）；`tests/churn/{test_config_sync,test_engine,test_mysql_repo,test_service,test_warning_export}.py` 内 5 处 import 改路径。
    
- ⚠️ **monkeypatch 字符串路径（import 改写抓不到，codex 审出 / grep 实证 19 处）**：`tests/shared/test_crm_api.py` 有 19 处 `monkeypatch.setattr("app.shared.crm_api.…")`（httpx.post / time.sleep / ThreadPoolExecutor），随该文件搬到 `tests/churn/` 时**全部**改成 `app.churn.crm_api.*`——否则 setattr 到不存在路径直接报错。自检：搬后 `grep -rn "app.shared.crm" tests/` 须空。
    

**文档同步（解冻必做，1 commit）**：

- `CLAUDE.md` 铁律 #2 改写：去掉"churn 对 shared 既有 import 是兼容红线"（已消解），保留"churn 内部重构独立立项"；目录速览更新 churn 文件清单（+crm_*/crm_models/sqlalchemy_engine/business_calendar）与 shared 清单（-这些）。
    
- `docs/architecture.md`：依赖方向图（churn→shared 现仅平台件）、shared 职责表**删 legacy-churn-CRM 区**（搬走了）、数据面/注入说明。
    
- `docs/module-guide.md`：反模式那条"复用 legacy churn CRM 件"改为"churn 的 CRM 抽象在 app/churn/、非平台件"。
    
- **文档路径引用改写（codex 审出，斜杠/点/.py 各形式都要扫）**：`docs/handover.md`（`:53,54,150,201,202,213,215`）、`docs/churn/technical-design.md`（文件树 `:101-104,110` + `:260,620,890` + 测试路径 `:1205,1206`）里的 `shared/crm_*`、`shared/models`、`shared/db`、`shared/business_calendar` 全改为 `app/churn/` 新路径（注意 `models`→`crm_models`、`db`→`sqlalchemy_engine` 改名）。自检：`grep -rnE "shared/(crm_|models|db|business_calendar)" docs/` 须空。
    

**B3 验证**：`pytest -q`（507）+ ruff + mypy；铁律自检 `grep -rn "from app.lead\|from app.churn" app/shared/` 须空；`grep -rn "from app.shared" app/churn/` 只剩 date_utils/config/crm_passport。

## B2 — 拆两个大文件（在 churn 最终位置，~2 commits）

用 **mixin 拆法**（保留单一对外类与 public import 路径，私有方法/状态按职责分文件，纯函数可模块级）。**关键约束（codex 审出）**：mixin 共享 `self`（`_cache`/`_engine`/`_session` 等），用一个 `typing.Protocol`（或 typed base）显式声明各 mixin 依赖的 self 属性，**每拆完一块即跑 `mypy`**（不等全拆完），避免多继承 MRO / 类型推断坑：

- **`app/churn/crm_api.py`(1061) → 5 文件**（Explore 已给边界）：`crm_api_http.py`（`_post_with_retry`/`_fetch*` + `_HTTP_RETRY_DELAYS_SECONDS`）、`crm_api_parsing.py`（`_parse_*`/`_extract_customer_rows` + `_CUSTOMER_API_*_KEYS`）、`crm_api_visits.py`（访问子系统 395-599 + `_VISIT_*_KEYS`）、`crm_api_loading.py`（分页/并发/懒加载 655-945）、`crm_api.py`（`CrmApiRepository` = 多 mixin + `CrmRepository`，含 init/缓存/public 接口）。保留 `from app.churn.crm_api import CrmApiRepository`。
    
- **`app/churn/mysql_repo.py`(642) → 6 文件**：`mysql_repo_schema.py`、`mysql_repo_warnings.py`、`mysql_repo_configs.py`、`mysql_repo_detection.py`、`mysql_repo_locks.py` + `mysql_repo.py`（主类 mixin 组合）。**保留 39-45 行 Row 类 re-export**（`from app.churn.mysql_repo import DetectionRunRow/Base` 是 schema 校验+测试依赖的兼容路径）。
    
- **风险点**：mixin 共享 `self`（缓存/engine/session）——逐文件抽、每抽一块即跑 `pytest -q`，不要一次性大搬。
    

## 会用到的能力

**扫描范围**（本次会话可见）：

- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
    
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily
    

**会用**：

- Agent Explore：执行期核查 import 残留/改写完整性（已用于摸拆分边界）。
    
- Agent codex:codex-rescue：B2 mixin 拆分状态共享最易错，让 codex 二次 review 拆后正确性，或并行拆一块做对照。
    
- Skill code-review：B 组改完审整体 diff（重构正确性，非仅风格）。
    

**考虑过但不用**：

- Skill simplify：只做质量清理、不查 bug；B 组是大重构要查正确性，用 code-review。
    
- Skill verify / run：纯重构、运行时行为零变化，pytest 即足够验证，无需起 app。
    
- deep-research / jina / tavily / firecrawl / chrome-devtools：纯本地重构，无需联网/浏览器。
    

## 验证（最终）

1. `python3 -m pytest -q` → 507 passed（搬迁/拆分零行为变化，数不变）。
    
2. `python3 -m ruff format app tests scripts && ruff check app tests scripts` 全绿；`python3 -m mypy` 全绿。
    
3. 铁律自检：`grep -rn "from app.lead\|from app.churn" app/shared/` 为空；`grep -rn "from app.shared" app/churn/` 只剩 date_utils/config/crm_passport。
    
4. 可选 `docker compose up -d --build && make smoke`。
    
5. 全程在分支 `chore/churn-restructure-b` 上分 6-7 个原子 commit，最后 FF 合回 main。
    

## 风险与回滚

- 最易错：B2 mixin 状态共享 → 逐块抽、每块跑测试。B3 命名冲突（models→crm_models）→ 全局改 import 后 mypy 兜底。
    
- 生产无关：不改 .env/DB/Redis/celery；纯 import 重构。
    
- 回滚：每步独立 commit，分支隔离，`git revert`/`reset` 可逐步回退。