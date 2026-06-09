# 智能拓客服务 · Docker 部署配置

## Context

要把 `smart_lead_generation`（FastAPI 智能拓客服务）的源码放到服务器上，用 Docker 部署。
当前仓库**没有 Dockerfile / .dockerignore**，只有一个本地开发用的 `docker-compose.yml`（仅起一个
绑 `127.0.0.1` 的 Redis，app 仍跑在宿主机 uvicorn 上）。目标是补齐一套可在服务器上一键拉起
**「app + 专用缓存 Redis」**的 Docker 文件，且保留本地开发工作流。

服务事实（已读码确认）：
- 纯 FastAPI ASGI，入口 `uvicorn app.main:app --host 0.0.0.0 --port 8000`，无 Celery/无独立 worker
  （仅 `app/middleware/api_log.py` 内一个 daemon flusher 线程池，按需懒启动）。
- 依赖全是纯 Python wheel（`PyMySQL`/`httpx`/`uvicorn[standard]`…，无需编译器），见 `requirements.txt`。
- 无状态：状态在两个**外部** MySQL（业务库 `47.114.77.122:3306` 只读；日志库 `8.138.104.14:23356` 读写）
  + 可选 Redis + 外部 CRM/DeepSeek HTTPS。随仓库带只读快照 `app/data/amer_industry.csv`。
- 配置：`app/config/settings.py` 用 `load_dotenv()` 读项目根 `.env`，**真实环境变量优先于 .env**
  （`load_dotenv` 不覆盖已存在的 env）→ compose 的 `env_file`/`environment` 注入即生效，镜像里不需要 `.env`。
- 健康检查 `GET /healthz` 返回 `{status, db, industries, industries_source}`，**始终 HTTP 200**，db 连通性写在 JSON 里。
- 缓存：`app/cache/store.py` 在 `REDIS_URL` 连不上时**自动回退进程内缓存**（dev 不开 Redis 也能跑）。

本次部署已确认的四个选择：
1. **缓存**：随 compose 起一个专用、仅缓存的 Redis（app 通过服务名 `redis` 连接）。
2. **连库网络**：服务器**直连公网**到 Aliyun 公网 IP（→ bridge 网络即可，无 MTU 问题；真正前置是 Aliyun 白名单）。
3. **对外暴露**：容器只绑 `127.0.0.1:8000`，由宿主机 nginx 反代 + TLS。
4. **compose 布局**：**全栈设为默认 `docker-compose.yml`**（服务器 `docker compose up` 即全起）；
   现有 dev 用的 Redis-only 配置**改名搬到 `docker-compose.dev.yml`**。
   —— 这样消除「在服务器裸敲 `docker compose up` 只起一个 Redis、没有 app」的静默陷阱，两套流程都保留。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Skill `security-review`：可选，复核 `.env`/密钥没被 `COPY` 进镜像层、`.dockerignore` 生效（本任务涉及 DB 密码、DeepSeek key、CRM salt）。
- 验证主要用内置 Bash：`docker compose build/up` + `curl /healthz`（见下方「验证」）。

**考虑过但不用**：
- `run` vs `verify`：两者都能拉起应用确认改动；容器场景我直接用 Bash `docker compose up` + `curl` 更直接，故都不调用。

（其余 skills 与 4 个 web/浏览器类 MCP 与「写 Docker 配置文件」明显无关，不列。）

## 实施：Docker 文件（重排 compose 命名 + 新增 Dockerfile/.dockerignore + 2 处文档微调）

### 1. `Dockerfile`（新增；多阶段、slim、非 root）

要点：
- 基础镜像 `python:3.12-slim-bookworm`，与开发 Python 3.12 对齐；glibc + 纯 wheel，无需 Alpine/编译器。
- builder 阶段先 `COPY requirements.txt` 再 `pip install --prefix=/install`，利用层缓存；runtime 阶段 `COPY --from=builder /install /usr/local`。
- 只 `COPY app/ ./app/`（含 `app/data/amer_industry.csv` 兜底快照），**不** copy `.env`/`tests`/`scripts`。
- 建非特权用户运行；`ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PYTHONPATH=/app TZ=Asia/Shanghai`
  （`tzdata` wheel 已在 requirements，保证 `ZoneInfo("Asia/Shanghai")` 在 slim 里可用，passport 北京日期不出错）。
- **不在 Dockerfile 里写 HEALTHCHECK**：`/healthz` 会真连业务库，baked 健康检查会因 Aliyun 侧（白名单/网络）问题把「降级但仍可服务」的容器反复重启。健康检查放到 compose 层。
- `CMD` 单进程 + 优雅停机窗口（给 lifespan 排空 api-log 队列）：
  `uvicorn app.main:app --host 0.0.0.0 --port 8000 --proxy-headers --forwarded-allow-ips "*" --timeout-graceful-shutdown 20`

### 2. `docker-compose.yml`（**重写现有文件** → 默认 = 全栈：app + 专用 Redis）

- `redis` 服务：`redis:7-alpine`，`--save "" --appendonly no --maxmemory 512mb --maxmemory-policy allkeys-lru`，
  **不 publish 端口**（只同网络的 app 能连），保留 `redis-cli ping` healthcheck。
- `app` 服务：
  - `build: { context: ., dockerfile: Dockerfile }`，`restart: unless-stopped`。
  - `env_file: [.env]` 读服务器上的真实凭据（gitignored，不进镜像）。
  - `environment:` 显式覆盖 `REDIS_URL: redis://redis:6379/0`（**关键**：`.env` 里默认是 `127.0.0.1`，在容器内指向自身；不覆盖会静默回退进程内缓存）、`TZ`、`CRM_PASSPORT_TZ`。
  - `depends_on: { redis: { condition: service_healthy } }`。
  - `ports: ["127.0.0.1:8000:8000"]`（只绑本机，nginx 反代）。
  - compose 层 `healthcheck` 用 stdlib（slim 无 curl）：`python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/healthz',timeout=3).status==200 else 1)"`，`start_period: 20s` 覆盖行业词典预热；**只判 HTTP 200（进程存活），不判 db:true**，避免外部库抖动触发重启。
  - `stop_grace_period: 25s`（> uvicorn 的 20s 优雅超时，确保 api-log 排空）。
- **默认命令即全栈**：服务器 `docker compose up -d --build` 直接拉起 app+redis，无需 `-f`。

### 3. `docker-compose.dev.yml`（新增 = 现有 Redis-only 内容搬家）

- 把现有 `docker-compose.yml` 的 `redis` 服务（256mb、绑 `127.0.0.1:6379`、healthcheck、所有中文注释）**原样**移到此文件，内容不变。
- 本地开发：`docker compose -f docker-compose.dev.yml up -d redis` + 宿主机 `uvicorn app.main:app ...`（app 不进容器，保留热重载内环）。
- 因 app 在 `REDIS_URL` 连不上时自动回退进程内缓存，此文件**仅在本地想实测 Redis 代码路径**（`RedisTTLCache`/跨进程 singleflight）时才必需。

### 4. `.dockerignore`（新增）

排除：`.git`/`.claude`/IDE、`.env` 与 `.env.*`（但 `!.env.example`）、`__pycache__`/`*.pyc`/各类 cache、
`tests/`/`scripts/`/`docs/`/`*.md`、`Dockerfile`/`.dockerignore`/`docker-compose*.yml`、`*.log`。
**保留** `app/data/amer_industry.csv`（随 `app/` 进镜像）。目的：密钥绝不进镜像层 + 瘦身。

### 5. 文档微调（2 处指向旧命令的地方）
- `README.md`：把 dev 启动那行 `docker compose up -d redis` → `docker compose -f docker-compose.dev.yml up -d redis`；
  再加一小节「服务器 Docker 部署」：前置（白名单/.env）+ `docker compose up -d --build`。
- `.env.example`（第 36 行注释）：`docker compose up -d redis` → `docker compose -f docker-compose.dev.yml up -d redis`。

### 关键设计取舍（已定，写进文件）
- **单 uvicorn 进程**起步：app 是 I/O 密集，单进程已并发；进程内缓存是 per-process 全局，单进程下天然正确。
  将来要 `--workers N` 或多副本，**必须**先有 `REDIS_URL`（本方案已起专用 Redis，前提已满足）——否则 N 份进程内缓存 = N 倍冷算（含付费 DeepSeek）。文件里加注释说明扩展路径。
- **bridge 网络**（默认）：只需出站到达公网 IP，NAT 正常；不用 `network_mode: host`。
- **MTU 不在 Docker 层处理**：那是开发机 `198.18.x` 隧道的属性；服务器直连公网无此问题，不 baked 任何 MTU。

## 服务器端前置（部署前必须确认，非本仓库代码能解决）
1. **Aliyun 白名单/安全组**：把服务器**出口公网 IP** 加进**两个**库——业务库 `47.114.77.122:3306` 与日志库
   `8.138.104.14:23356`（**注意非标准端口 23356**，别漏）。否则 `/healthz` 显示 `db:false`、查询全 503。
2. 服务器上从 `.env.example` 复制出 `.env` 填真实密码/`DEEPSEEK_API_KEY`/`CRM_PASSPORT_SALT`，`chmod 600 .env`。
3. 服务器装好 Docker Engine + Compose v2。
4. nginx 反代：`proxy_pass http://127.0.0.1:8000;`（app 已带 `--proxy-headers`），TLS 在 nginx 终结。（nginx 配置本身不在本次 Docker 文件范围内，按需另配。）

## 验证（端到端）
1. 构建：`docker compose build`
2. 确认无密钥进镜像：`docker run --rm smart-lead-generation:latest ls -la /app`（应无 `.env`）；必要时 `docker history` 复核。
3. 起服务：`docker compose up -d`，`docker compose ps`（redis healthy、app 起来）。
4. 健康检查：`curl -s http://127.0.0.1:8000/healthz`
   - 期望 `{"status":"ok","db":true,"industries":<N>,"industries_source":"db"|"snapshot"}`。
   - `db:false` → 白名单/网络没通；`industries_source:"snapshot"` → 业务库行业表没读到（走了本地 CSV 兜底）。
5. 真接口冒烟：`POST /api/v1/leads/score-and-info`（带一组合法 `employee_code/lng/lat/radius_km`）确认返回 `{code:0,...}`。
6. 日志：`docker compose logs -f app`，确认无栈、优雅启动。
7. 重启容器验缓存命中：同一查询第二次应为毫秒级（Redis 命中）。
8. dev 流程回归：`docker compose -f docker-compose.dev.yml up -d redis` 仍能为宿主机 uvicorn 提供 Redis。

## 交付物
- 新增 `/mnt/e/smart_lead_generation/Dockerfile`
- 新增 `/mnt/e/smart_lead_generation/.dockerignore`
- 新增 `/mnt/e/smart_lead_generation/docker-compose.dev.yml`（搬入旧 Redis-only 内容）
- **重写** `/mnt/e/smart_lead_generation/docker-compose.yml` → 全栈 app + redis（默认文件）
- 文档微调：`/mnt/e/smart_lead_generation/README.md` 与 `/mnt/e/smart_lead_generation/.env.example` 里
  `docker compose up -d redis` → `docker compose -f docker-compose.dev.yml up -d redis`；README 增「服务器 Docker 部署」一节。
