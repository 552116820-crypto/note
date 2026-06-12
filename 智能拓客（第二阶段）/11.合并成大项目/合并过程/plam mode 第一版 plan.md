# 代码合并：churn-warning-engine + smart_lead_generation → /mnt/e/amer_sales_assist

## Context

两个同领域项目（客户流失预警 + 智能拓客）合并为一个模块化单体平台。依据是用户两份调研笔记（D:\note\智能拓客（第二阶段）\）已定的结论：

- churn 本身就是 `app/churn/` + `app/shared/` 的"一库多模块"结构，lead 进来就是 `app/lead/`
- 数据库已统一：churn 表已迁入 `amer_sales_assist` 库并带 `churn_` 前缀，migration 017 已建好 `lead_skip`/`lead_api_log` 表（DDL 与 lead 实际读写列**已核对一致**，无需修正迁移）
- Redis 单实例分库：db0=Celery broker、db1=Celery result、db2=lead 缓存；实例级 `maxmemory` + **`volatile-lru`**（broker key 无 TTL 天然免疫淘汰；lead 缓存 setex 带 TTL 可淘汰）。**禁止 allkeys-lru**（lead 现 compose 用的就是它，必须改）

**用户已拍板**：① 合并产物放到空目录 `/mnt/e/amer_sales_assist`；② **纯代码合并，不做任何 git 操作**（不 clone、不 init、不 commit——要不要纳管 git 之后用户自己定），原两个目录原封不动即天然备份/回滚；③ **机械并入**——lead 自包含进 `app/lead/`（含它自己的 cache/db/clients/config/middleware/errors），只改 import 路径和部署配置，共享层去重留下一阶段。

**对外契约不变**：churn 路由保持根路径（`/api/warnings/*`、`/api/detection-runs/*`，X-API-Key 保护），lead 保持 `/api/v1/leads/*`——两者无路径冲突（已验证）。

已核实消除的疑点：churn 的 DeepSeek/AI 分析代码**已全部删除**（grep 零命中，migration 019 已删表），其 `.env` 里 `DEEPSEEK_*`/`CHURN_AI_ANALYSIS_ENABLED` 是死配置 → lead 的 7 个 `DEEPSEEK_*` 键名无前缀直接用，无冲突。lead 的 `ApiLogMiddleware` 本身就按 4 个 `(method, path)` 白名单只记 lead 接口，churn 路由不受影响。

**Codex 二审结论**（已读实际代码逐项验证）：无 BUG 级问题；sed 映射完整、`app.main` 不重写正确、main.py 设计与两边测试断言吻合、依赖约束可解、compose 设计正确。两个 GAP 已并入下文：① churn `.env` 里 `API_KEY` 是注释状态（auth.py 与 production 启动校验都要求非空）→ 见 3.3；② `API_LOG_*` 在 lead `.env.example` 中只是注释示例（仅 `ENABLE_API_LOGGING` 是显式键，默认值由 settings 类提供）→ 见 3.3。（原 RISK"squash commit 记录 SHA"随 git 环节取消而失效。）

---

## Phase 0 — 体检基线（旧目录，只读）

```bash
cd /mnt/e/churn-warning-engine && pytest -q       # 应绿
cd /mnt/e/smart_lead_generation && pytest -q      # 应 323 绿
```

## Phase 1 — churn 作为基座复制到新目录，验证 churn-only 绿

```bash
rsync -a --exclude='.git' --exclude='.pytest_cache' --exclude='__pycache__' \
  --exclude='*.egg-info' --exclude='.claude' --exclude='cutover_*.sql' \
  /mnt/e/churn-warning-engine/ /mnt/e/amer_sales_assist/
cd /mnt/e/amer_sales_assist
python3 -m venv .venv && .venv/bin/pip install -e .[all]
.venv/bin/pytest -q                                # churn 基座绿 = Phase 1 完成
```

- `.env` 随 rsync 带过来（后续 Phase 3 改造）；`cutover_*.sql` 是已执行过的一次性生产脚本，**故意不带**（留在旧目录存档）

## Phase 2 — lead 并入 `app/lead/`，合并 main.py + pyproject，验证

### 2.1 文件搬迁

```bash
mkdir -p tests/lead scripts/lead docs/lead
rsync -a --exclude='__pycache__' --exclude='*.py[cod]' /mnt/e/smart_lead_generation/app/ app/lead/
rm app/lead/main.py                       # 融入根 app/main.py（见 2.4）
rsync -a --exclude='__pycache__' --exclude='*.py[cod]' /mnt/e/smart_lead_generation/tests/ tests/lead/
rsync -a --exclude='__pycache__' /mnt/e/smart_lead_generation/scripts/ scripts/lead/
cp /mnt/e/smart_lead_generation/docs/接口文档-对接前端.md docs/lead/
cp /mnt/e/smart_lead_generation/{README.md,TEST_REPORT.md} docs/lead/
```

- lead 工作区里 `app/data/amer_industry.csv` 的最新内容（含其未提交改动）随 rsync 一并带入
- **不带**：lead 的 main.py、requirements.txt（并进 pyproject）、Dockerfile、docker-compose*、.gitignore/.dockerignore/.env*（Phase 3 合并）、.claude/、.git/

### 2.2 import 重写（sed，幂等，`app.main` 故意不在映射里）

映射：`app.X` → `app.lead.X`，X ∈ {cache, clients, config, db, errors, middleware, repositories, routers, schemas, services}；路径写法 `app/X/` → `app/lead/X/`（含 data）。Codex 已对照 lead 实际 import 清单确认**无遗漏**；`data` 只是包数据路径、非模块，不进 import 映射。

```bash
grep -rl --include='*.py' -E '\bapp\.(cache|clients|config|db|errors|middleware|repositories|routers|schemas|services)\b' app/lead tests/lead scripts/lead \
  | xargs sed -i -E 's/\bapp\.(cache|clients|config|db|errors|middleware|repositories|routers|schemas|services)\b/app.lead.\1/g'
grep -rl --include='*.py' -E 'app/(cache|clients|config|data|db|errors|middleware|repositories|routers|schemas|services)' app/lead tests/lead scripts/lead \
  | xargs sed -i -E 's#app/(cache|clients|config|data|db|errors|middleware|repositories|routers|schemas|services)#app/lead/\1#g'
# 收尾审计，两条都必须为空：
grep -rn --include='*.py' -E '\bapp\.(cache|clients|config|db|errors|middleware|repositories|routers|schemas|services)\b' app/lead tests/lead scripts/lead   # 残留旧 import
grep -rn --include='*.py' 'app\.lead' app/churn app/shared tests/churn tests/shared            # churn 侧反向污染
```

`app.main` 不重写的原因：lead 的 `tests/test_healthz.py` 直接 import 并 monkeypatch `app.main`，合并后它要打到的就是根 main。

### 2.3 手工修改点

| 文件 | 改动 |
|---|---|
| `app/lead/config/settings.py` | `_ENV_PATH = parents[2]` → `parents[3]`（文件深了一层，.env 在仓库根）+ 注释 |
| `scripts/lead/{http_smoke,integration_check,load_test,probe_crm}.py` | sys.path 处再包一层 `os.path.dirname(...)`（深了一层） |
| `app/__init__.py` | docstring → 平台描述 |
| `app/main.py` | 整体重写（2.4） |
| `pyproject.toml` | 整体替换（2.5） |

### 2.4 合并版 `app/main.py` 关键约束（按此写）

- **模块级必须导入名 `ping`（来自 `app.lead.db.pool`）和 `industry_resolver`**——`tests/lead/test_healthz.py` monkeypatch 的是 `app.main.ping` / `app.main.industry_resolver`
- lifespan = churn 的 `_validate_startup_config()` + `_warm_up_services()`（从旧 main.py:32-81 **原样照抄**，保持工厂惰性导入——churn 测试有断言）→ lead 的 `industry_resolver.load()`（包 try，失败仅 warning 不阻断启动）；shutdown = `timeline_service.shutdown_crm_pool()` + `crm_client`/`deepseek_client`.close() + `await api_log.drain_and_shutdown()`
- 中间件 add 顺序（后加的在外层）：`ApiLogMiddleware`（最内，看到最终 envelope）→ `RequestBodyLimitMiddleware`（413 先于日志缓冲）→ `RequestIDMiddleware` → `CORSMiddleware`（churn 原配置照抄，默认 `*`，最外）
- 路由：churn router 根路径 + `leads.router` + `customer_detail.router`（前缀都在 router 自身，无需改）
- `/healthz` 合并为一个：`{"status":"ok","db":ping(),"industries":...,"industries_source":...}`，**不能出现 `code` 键**，去掉 churn 的 `HealthResponse` model
- 异常处理器**按路径限定**（`request.url.path.startswith("/api/v1/leads")`）：
  - `RequestValidationError`：lead 路径 → 200 + envelope code 1002（照抄 lead 原实现）；**非 lead 路径 → `request_validation_exception_handler` 默认 422**（churn 测试断言 422）
  - 兜底 `Exception`：lead → envelope 500；非 lead → `PlainTextResponse("Internal Server Error", 500)`
  - `TooManyCandidates`/`DBUnavailable`/`LogDBUnavailable`/`DomainError`：照抄 lead，不用限定（只有 lead 代码会抛）
- FastAPI 元信息：`title="AMER Sales Assist"`, `version="2.1.0"`
- lead 的 `logging.basicConfig` 丢弃，统一用 churn `setup_logging()`

### 2.5 合并版 `pyproject.toml`（核心内容）

```toml
[project]
name = "amer-sales-assist"
version = "2.1.0"
requires-python = ">=3.10,<3.13"
dependencies = [
  "fastapi>=0.116,<0.117",        # lead 的中间件实测 pin 收紧 churn 的 <1.0
  "starlette>=0.47,<0.48",
  "uvicorn[standard]>=0.35,<1.0",
  "pydantic>=2.12,<3.0",
  "httpx>=0.27,<1.0",
  "PyMySQL>=1.1,<2.0",            # lead 无条件 import → 升为核心
  "python-dotenv>=1.0,<2.0",
  "redis>=5.3,<6.0",              # 交集 >=5.3,<6.0，升为核心（lead 缓存运行期要）
  "tzdata>=2024.1",
]
[project.optional-dependencies]
mysql = ["sqlalchemy>=2.0,<3.0"]  # PyMySQL 已上移
celery = ["celery>=5.3,<6.0"]     # redis 已上移
dev = ["pytest>=8.0,<9.0"]
all = ["amer-sales-assist[mysql,celery,dev]"]
[tool.setuptools.package-data]
"app.lead" = ["data/*.csv"]
# build-system / packages.find include ["app*"] / pytest testpaths ["tests"] 沿用 churn 原文件
```

### 2.6 验证门（必须全过才进 Phase 3）

```bash
rm -rf .venv && python3 -m venv .venv && .venv/bin/pip install -e .[all]
.venv/bin/pytest tests/churn tests/shared -q     # churn 不回归
.venv/bin/pytest tests/lead -q                   # 323 绿
.venv/bin/pytest -q                              # 合跑（.env 泄漏金丝雀，见风险2）
.venv/bin/python -c "import app.main"
.venv/bin/uvicorn app.main:app --port 8001 &
curl -s localhost:8001/healthz                                                        # 含 status/db/industries
curl -s -X POST localhost:8001/api/v1/leads/skip -H 'content-type: application/json' -d '{}'   # 200 {"code":1002,...}
curl -s -o /dev/null -w '%{http_code}\n' 'localhost:8001/api/warnings?limit=501'              # 422（churn 契约未变）
kill %1
```

## Phase 3 — 部署/配置统一 + 命名/文档

### 3.1 Dockerfile（替换为 lead 的多阶段范式）

`python:3.12-slim-bookworm` 两阶段、清华 pip 源、非 root 用户 `app`、`pip install .[all]`、CMD 带 `--proxy-headers --forwarded-allow-ips "*" --timeout-graceful-shutdown 20`（churn requires-python <3.13 容许 3.12；本机 venv 3.12.3，测试即同版本验证）。

### 3.2 docker-compose.yml（替换）

- **redis**：`redis-server --maxmemory 512mb --maxmemory-policy volatile-lru` + healthcheck（注释写明 db0/1/2 分工与"禁 allkeys-lru"原因）
- **api**：env_file `.env`，environment 强制钉死 `REDIS_URL=redis://redis:6379/2`、`CELERY_BROKER_URL=redis://redis:6379/0`、`CELERY_RESULT_BACKEND=redis://redis:6379/1`、`TZ=Asia/Shanghai`；`ports: ["127.0.0.1:8000:8000"]`（**保住 lead 前端现有 host-nginx 上游**）+ expose 8000 给 compose 内 nginx；healthcheck 打 /healthz；`stop_grace_period: 25s`
- **celery**：worker `--beat` 命令**必须加 `-s /tmp/celerybeat-schedule`**——新镜像非 root，`/app` 写不进，beat 默认调度文件会让容器崩
- **nginx**：沿用 churn 的 8080→api:8000；`nginx/default.conf` 是 `location / proxy_pass`，已天然转发 `/api/v1/leads/*`，**无需改**
- lead 的 `docker-compose.dev.yml` 可选重建为 redis-only 开发辅助（用 volatile-lru）；`docker-compose.override.yml` 不带

### 3.3 .env / .env.example 合并

- `.env.example`：churn 原文件为骨架；删死键 `CHURN_AI_ANALYSIS_ENABLED`、`DEEPSEEK_API_BASE_URL`、`DEEPSEEK_TIMEOUT_SECONDS`；DeepSeek 块换成 lead 的 7 个键名；追加 lead 全部分节（`DB_*`、`LOG_DB_*`、`CACHE_*`、`RADIUS_*`、`CRM_BASE_URL`/`CRM_PASSPORT_*`、`DETAIL_*`/`TIMELINE_*`、`ENABLE_API_LOGGING` 等，照抄 lead .env.example；**`API_LOG_*` 保持 lead 原样的注释示例形态**——仅 `ENABLE_API_LOGGING` 是显式键，其余默认值由 settings 类提供，不显式写入，Codex GAP-2）；`REDIS_URL=redis://redis:6379/2`（注释裸机用 127.0.0.1:6379/2）
- 真 `.env`（手工同操作）：**`REDIS_URL` 必须从 lead 的 `/0` 改 `/2`**（compose 虽钉死，裸机/pytest 读的是 .env）；DEEPSEEK_API_KEY 取 lead 的活 key（churn 那个是死的）；`APP_ENV`、`DATABASE_URL` 沿用 churn；lead 的 `DB_*`/`LOG_DB_*` 原样并入
- **`API_KEY`（Codex GAP-1）**：churn 本地 `.env` 里它是**注释状态**（开发模式不强制，本地合并版保持现状即可）；但 `APP_ENV=production` 时启动校验（main.py）与 auth.py 都要求非空——**服务器部署前必须从旧生产环境的 .env 提取实际值，写成非注释、非空的 `API_KEY=<旧值>`**，且与旧值一致（CRM 调用方 X-API-Key 不变），否则合并版起不来

### 3.4 命名/文档/杂项

- 根 `README.md` 重写为平台总览（模块清单、目录约定、启动方式）；churn 旧 README + `docs/` 5 篇 → `docs/churn/`；lead 文档已在 `docs/lead/`
- `AGENTS.md`：补 `app/lead/` 自包含模块规则（"churn 与 lead 互不 import，仅在 app/main.py 组合；共享要提升须走 app/shared/"）
- `CHANGELOG.md` 加 2.1.0；`.gitignore` 并入 lead 条目（虽暂不建 git，留好规则）；`deploy.sh`/`setup-https.sh` 逻辑不变只改文案

## Phase 4 — 终验

```bash
cd /mnt/e/amer_sales_assist
.venv/bin/pytest -q                                  # 全量
docker compose build && docker compose up -d
curl -s localhost:8080/healthz                       # 走 nginx
curl -s -o /dev/null -w '%{http_code}\n' -H "X-API-Key: <旧值>" 'localhost:8080/api/warnings?limit=1'
curl -s -X POST localhost:8080/api/v1/leads/score-and-info -H 'content-type: application/json' -d '{...实参}'
curl -s http://127.0.0.1:8000/healthz                # lead 前端旧上游路径
docker compose exec redis redis-cli config get maxmemory-policy     # volatile-lru
docker compose exec redis redis-cli -n 2 --scan --pattern 'lead:*'  # 缓存落 db2
docker compose logs celery | grep -E "beat|ready"    # worker+beat 起来，3 个排程
docker compose down
```

## 上线切换清单（服务器侧，之后手工做，本次不执行）

1. 避开 beat 窗口（周一 07:00/07:15、每月 1 日 07:30），确认旧 broker 空：`redis-cli -n 0 keys '*'`
2. 把 `/mnt/e/amer_sales_assist` 整体同步到服务器（git 纳管与远程仓库之后由你自行决定）
3. 放置合并版 `.env`（重点核对：**`API_KEY` 从旧生产 .env 提取实际值、非注释非空、与旧值一致**；`DEEPSEEK_API_KEY` 取 lead；`REDIS_URL` db2；`APP_ENV=production`；`CRM_HTTP_TRUST_ENV=0`）
4. 旧 churn 与旧 lead 两个目录分别 `docker compose down`（释放 8080 和 127.0.0.1:8000）→ 新目录 `docker compose up -d --build` → 跑 Phase 4 冒烟
5. 路由不动：churn 调用方继续打 :8080；lead 前端的 host nginx 继续指 127.0.0.1:8000
6. 观察一个周期：`lead_api_log` 有新行、下周一 beat 写 `churn_detection_runs` 并推 CRM、`redis-cli info memory` 验证 512mb/volatile-lru 余量

## 已识别的行为变化/风险（接受项）

1. **celery beat 调度文件**：非 root 镜像下必须 `-s /tmp/...`，否则容器崩（已纳入 compose）
2. **合跑 pytest 时 .env 泄漏**：lead conftest import 链会 `load_dotenv()` 进 os.environ；已审计 churn 所有 env 敏感测试都显式 setenv/delenv，预期绿；合跑命令即金丝雀，分跑是兜底
3. **/healthz 变重**：合并后真实 ping 业务库（amer_crm_v2 不可达时最坏 ~35s、容器显示 unhealthy 但仍服务）——机械合并轮接受，后续可给 ping 加短超时
4. /healthz 响应多了 `db/industries` 键（增量，status 语义不变）；churn 全部请求多 1MB body 上限与 X-Request-ID（无实际影响）；lead 日志格式改走 churn `setup_logging`；lead 未设 CORS_ORIGINS 时从"不挂 CORS"变为 churn 默认 `*`（维持 churn 现状）
5. churn 运行时 3.11→3.12 + 依赖重新解析——本地 venv 3.12.3 的全量测试即验证
6. **无 git 保护**：本次不建仓库，回滚手段 = 原两个目录未动 + 可整删 `/mnt/e/amer_sales_assist` 重来；建议合并验收后尽早自行纳管 git

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent Explore：规划期已用（两仓库并行扫描）；执行期如需复核零散事实再用
- Skill codex:rescue：规划期已用（Codex 对计划做了二审，结论已并入）；执行后如需可再让它复查合并产物
- Skill code-review：合并完成后对手写合并文件（app/main.py、pyproject、compose）做一次正确性复查（注：无 git 时按目录 diff 审，若受限则降级为人工走查）

**考虑过但不用**：
- Skill verify / run：验收靠计划内显式的 pytest + uvicorn + compose + curl 命令（Bash 直接执行），不需要通用应用驱动技能
- MCP chrome-devtools：无浏览器界面要验，接口冒烟 curl 足够
- MCP jina / firecrawl / tavily、Skill deep-research：纯本地代码合并，无联网调研需求
