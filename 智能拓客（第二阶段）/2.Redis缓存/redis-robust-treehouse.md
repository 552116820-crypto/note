# 启用并跑通 Redis 缓存（smart_lead_generation）

> 本版已并入 Codex 交叉评审意见（修正了运行期降级描述、重排验证顺序、改稳 dotenv 写法与 compose）。

## Context（为什么做这个）

你要求"添加 Redis 缓存"。但摸查后发现：**Redis 缓存代码其实已经写好了**，只是没启用——
所以本任务不是"写缓存代码"，而是**把已有的 Redis 后端真正接上一个实例、跑通、验证它确实生效**。

现状（已存在、已为 Redis 准备好）：
- `app/cache/store.py:82` `RedisTTLCache`：JSON 值、`setex` 带 TTL、连接/读超时容错。
- `app/cache/store.py:118` `_build_cache()`：**只要 `settings.redis_url` 非空就切 Redis 并 `ping`，连不上自动降级回进程内缓存**（`InProcessTTLCache`）。**注意：此选择在模块 import 时定型（`store.py:130` 模块级 `cache = _build_cache()`），运行期不再改变。**
- `app/config/settings.py:50` 读 `REDIS_URL`；`requirements.txt` 有 `redis>=5.3`（系统已装 `5.3.1`，无需再装）。
- `app/services/lead_service.py` 两处热点（员工上下文 `emp:*`、整份评分 blob `lead:*`）都走 `cache.get_or_compute()`，数据全 JSON-safe。

你的选择：**①启用并跑通现有缓存 ②用 docker-compose 起本地 Redis**。
目标产出：一个独立的本地 Redis（docker-compose）+ 填好 `REDIS_URL`，并验证缓存真的落到了 Redis（而非进程内兜底）。

## 改动清单

> 代码层**不需要改**——`_build_cache()` 已处理切换与降级。只新增部署文件 + 改配置。

### 1. 新增 `docker-compose.yml`（项目根，仅 Redis；app 仍用宿主机 uvicorn）
app 跑在宿主机，所以必须把 6379 发布出来，但只绑 `127.0.0.1` 不外露。
纯缓存：不挂 volume、关 RDB/AOF、限内存 + LRU。`command` 用**列表（exec）形式**避免 `--save ""` 的引号在折叠标量里被误解析；不设固定 `container_name`（跨项目易撞名，用 `docker compose exec redis` 即可）。

```yaml
# 本地开发用 Redis（仅缓存；app 用宿主机 uvicorn，故只发布到 127.0.0.1）。
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    # 纯缓存：不挂 volume，关 RDB/AOF；重启丢缓存是预期行为。本实例独立、无 Celery，allkeys-lru 安全。
    command:
      - redis-server
      - --save
      - ""
      - --appendonly
      - "no"
      - --maxmemory
      - 256mb
      - --maxmemory-policy
      - allkeys-lru
    ports:
      - "127.0.0.1:6379:6379"   # 只绑本机，别暴露到局域网
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 5s
```

### 2. 启用 `REDIS_URL`（注释单独成行，**不写行内注释**）
python-dotenv 只有在 `#` 前**有空白**时才把它当行内注释；一旦后续编辑漏了那个空格，`#...` 会并进值里，导致实际 URL 与肉眼所见不符。**最稳写法：注释独立一行、值裸写。**
- `.env`（实际生效）：把现有 `# REDIS_URL=...` 那一行替换为：
  ```dotenv
  # 本地 docker-compose Redis 缓存；要回到进程内缓存，请注释掉下一行或留空。
  REDIS_URL=redis://127.0.0.1:6379/0
  ```
- `.env.example`（模板，第 31 行）：同样取消注释、独立注释行：
  ```dotenv
  # 本地 docker-compose Redis 缓存；启动前先 docker compose up -d redis。留空=进程内缓存兜底。
  REDIS_URL=redis://127.0.0.1:6379/0
  ```
- **实施时只定位 Redis 行、不全文打印 `.env`（含业务库凭据）**：`rg -n '^\s*#?\s*REDIS_URL\b' .env .env.example`，确认实际文本后再精确 `Edit`。

### 3.（可选）README「运行」补一句
在 `README.md` §运行 下加：`docker compose up -d redis` 起本地缓存；省略则自动用进程内缓存。

## 与 churn 合并的前瞻注记（本次不做，但写进注释/文档防踩坑）

将来与 `churn-warning-engine` **共用一个 Redis** 时——`maxmemory-policy` 是**整实例全局**的、不是分库的。
Celery（churn 的 broker/backend，DB 0/1）要求 **`noeviction`**（不能丢任务），缓存默认想要 `allkeys-lru`，**光"分库"解决不了淘汰策略冲突**。

正确做法二选一：
- **共用同一实例**：实例策略设 `noeviction`，本缓存用**独立 DB**（如 `redis://.../2`）+ 现有 `lead:`/`emp:` 前缀做命名隔离。
  本缓存是 TTL 自过期（`setex`）+ key 数量小，本身不依赖淘汰、放进 `noeviction` 也安全。
  **但要清楚**：DB 只隔离 keyspace、**不隔离内存**，无法给某个 DB 单独限内存——缓存与 Celery 共享同一内存池，
  缓存涨上去可能让 Celery 写入触发 Redis OOM 失败。需按「Celery + 缓存」峰值预留 `maxmemory` 并监控 OOM/写失败。
- **若确实要给缓存用 `allkeys-lru`**：正确隔离方式是**单独起一个 Redis 实例**，而不是只拆 DB。

本次是独立本地实例、无 Celery，故用 `allkeys-lru`，不涉及上述冲突。

## 前置条件 / 注意

- **docker 当前在此 WSL 不可用**（`docker: command not found`）。要 `docker compose up`，先在
  Docker Desktop → Settings → Resources → **WSL Integration** 打开对本发行版的集成；或见"不依赖 docker 的兜底"。
- 端到端验证会连业务库 `47.114.77.122`：每次 WSL 重启后先 `sudo ip link set dev eth2 mtu 1400`，
  否则地理大结果集会被隧道黑洞丢弃、请求挂起（README §MTU；**非代码问题**）。
- **真实环境变量优先于 `.env`**（`load_dotenv` 不覆盖已存在的 env）。验证前先排查陈旧覆盖：
  `env | rg '^REDIS_URL='`；若有残留，先 `unset REDIS_URL` 或显式设成 `redis://127.0.0.1:6379/0`。
- 改完 `.env`、起完 Redis、或 Redis 曾不可用之后，**必须重启已在跑的 uvicorn 进程**——`settings` 和 `cache` 都在 import 时创建。

## 验证（必须证明"切到了 Redis"而非进程内兜底）

**关键时序：Redis 必须早于任何 `from app.main import app` / `from app.cache.store import cache`。** 因为 `cache` 在 import 时定型。

1. **先起 Redis**（且必须早于下面任何 Python import）：
   `docker compose up -d redis` → `docker compose exec -T redis redis-cli ping` 得 `PONG`。
2. **fail-fast 验工厂选择**（全新 Python 进程，项目根执行以加载 `.env`）：
   `python3 -c "from app.cache.store import cache; n=type(cache).__name__; print(n); raise SystemExit(n!='RedisTTLCache')"`
   必须打印 `RedisTTLCache`；否则**别往下跑**，先修 `REDIS_URL`/连接/陈旧 env。
3. 清空本地验证库（**仅限本地独立 compose 实例**，别在共享实例上做）：
   `docker compose exec -T redis redis-cli FLUSHDB`
4. 跑既有 smoke：`python3 scripts/http_smoke.py`（page1 未命中 ~7–10s → page2 命中 ~2ms）。
   注：smoke 把 uvicorn 日志设为 `warning`，**看不到** `store.py:123` 的 `INFO Using Redis cache backend`——别拿这行或"page2 变快"当证据（进程内缓存同样会变快）。
5. **硬证据——键确实落到 Redis**：
   ```bash
   docker compose exec -T redis redis-cli --raw --scan --pattern 'lead:*' | head
   docker compose exec -T redis redis-cli --raw --scan --pattern 'emp:*'  | head
   key="$(docker compose exec -T redis redis-cli --raw --scan --pattern 'lead:*' | head -n1)"
   docker compose exec -T redis redis-cli TTL "$key"   # 应满足 0 < TTL <= 300（CACHE_TTL_SECONDS）
   ```
6. **降级行为（区分两种，别搞混）**：
   - **启动时降级**：Redis 没起就启动新进程 → `_build_cache()` 打 `WARNING Redis unavailable ...; falling back to in-process cache`，`type(cache)` 为 `InProcessTTLCache`。这是唯一会"切回进程内"的路径。
   - **运行期宕机**：进程已选定 `RedisTTLCache` 后再 `docker compose stop redis` → **不会**切后端、**不会**打"falling back"，只会每次打 `redis get/set failed` 并重算（功能正常、无缓存加速）。

**不依赖 docker 的兜底**（开不了 docker 集成时）：
- 仅验"接线 + 启动时降级"：把 `REDIS_URL` 临时指向连不上的端口（如 `redis://127.0.0.1:6399/0`）起新进程，
  应见 `falling back to in-process cache` 且功能照常——证明接线/降级正确（但不证明真用 Redis）。
- 要硬验证：WSL 内 `apt-get install -y redis-server` 后 `redis-server --port 6379 --save "" --appendonly no &`
  代替 compose，再做第 2、5 步。

## 关键文件
- 新增：`docker-compose.yml`
- 改：`.env`、`.env.example`（、可选 `README.md`）
- 不改但是核心机制：`app/cache/store.py:118`（`_build_cache` 切换+降级）、`:130`（import 时定型）、`app/config/settings.py:50`（`redis_url`）
- 复用验证脚本：`scripts/http_smoke.py`

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent `Explore`：已并行派 3 个，摸清技术栈/缓存热点/Redis 现状与 churn 合并上下文（规划阶段已完成）。
- Skill `codex:rescue` → Agent `codex:codex-rescue`：已把本 plan 交给 Codex 交叉评审（本轮完成，结论已并入；首跑后台崩溃，已用前台 `--wait --fresh` 重跑成功）。
- 内置 Write/Edit/Bash：加 compose、改 env、起 docker、跑 `http_smoke.py` + `redis-cli` 验证。

**考虑过但不用**：
- Agent `Plan`：改动小且设计明确，相关代码已读完、又有 Codex 评审兜底，直接成文，不再派 Plan agent。
- Skill `verify` / `run`：验证直接复用项目自带 `scripts/http_smoke.py` + `redis-cli`（经 Bash）即可，不必再包一层；你若想要也可改用 `/verify`。
