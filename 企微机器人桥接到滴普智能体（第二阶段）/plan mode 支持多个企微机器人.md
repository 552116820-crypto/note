# 支持多个企微机器人（多部门）— 多容器部署方案

## Context（为什么做这个改动）

不同部门要用**不同的企业微信机器人**（各自独立的 `WECOM_BOT_ID`/`WECOM_BOT_SECRET`），并连**各自的滴普知识库**（`DEEPEXI_AGENT_ID` 不同，滴普平台/凭据相同）。

当前架构是**单机器人**：`__main__.py` 只起一条 WebSocket 长连接，靠全局唯一的一对 `WECOM_BOT_ID/SECRET` 鉴权（`__main__.py:32-54`，`config.py:535-536`）。已有的 `DEEPEXI_GROUP_ROUTES_JSON` 路由解决的是"同一个机器人、不同群连不同知识库"，**不是多个企微机器人**，因此不满足诉求。

本服务无监听端口、是纯出站长连接客户端，`docker-compose` 单服务、不映射端口，**天然支持横向复制**。经确认（用户选定）：规模 4-8 个机器人、后端仅 `agent_id` 不同。因此采用**多容器部署**——为每个部门跑一个独立容器，各自一份配置。

**预期结果**：零代码改动；每个部门一个容器、一条长连接、一个知识库；去重/会话/日志/健康检查天然按部门隔离，一个部门故障不影响其他。

## 方案概览

- **不改任何 Python 代码**。只动部署/配置文件 + 文档。
- 所有容器**复用同一个镜像**（`wecom_deepexi_aibot:latest`，build 一次）。
- 配置**分两层**，避免 4-8 份重复：
  - `.env.common`：所有部门共享的滴普凭据 + 行为配置。
  - `.env.<dept>`（每部门一份）：该部门的 `WECOM_BOT_ID/SECRET` + `DEEPEXI_AGENT_ID`，以及可选的部门级覆盖。
  - 利用 docker compose v2 的 `env_file` 列表语义——**列表中靠后的文件覆盖靠前的同名变量**。
- `docker-compose.yml` 用 YAML 锚点抽公共段，每个部门一个 service。

## 具体改动

### 1. 企微侧准备（人工，前置）
在企微管理后台为**每个部门**创建一个"智能机器人"，连接方式选**长连接**，各拿一对 `Bot ID` / `Secret`。
> 关键约束：每个 `bot_id` 全局只能有一条活连接。务必确保各部门用**不同** bot_id，且没有别的进程占用同一 bot_id，否则企微会反复互踢、疯狂重连（`wecom_ws.py:211-216` 的 `disconnected_event` 处理）。

### 2. 新建 `.env.common`（共享层）
从现有 `.env.example` 抽出**所有部门一致**的部分（不含 bot 凭据、不含 `DEEPEXI_AGENT_ID`）：
```
DEEPEXI_BASE_URL=http://knowledge.amer.cn/api/openapi
DEEPEXI_APP_ID=<共享 app_id>
DEEPEXI_SECRET_KEY=<共享 secret_key>
DEEPEXI_FIXED_INPUTS_JSON={}
DEEPEXI_RESPONSE_MODE=blocking
DEEPEXI_TIMEOUT_SECONDS=120
DEEPEXI_VERIFY_SSL=true
DEEPEXI_INPUT_FILE_MODE=upload
DEEPEXI_FILE_UPLOAD_PATHS=/chat/files/upload
DEEPEXI_APP_USER=amer
DEEPEXI_APP_USER_PREFIX=wecom:
CONVERSATION_TTL_SECONDS=1800
RESET_CONVERSATION_COMMANDS=/reset,重置会话,清空会话,清空上下文
RESET_CONVERSATION_REPLY=会话已重置，请开始新的提问。
REPLY_ERROR_PREFIX=处理失败，请稍后再试
WECOM_STREAM_REPLY_MAX_BYTES=20480
STRIP_LEADING_MENTIONS=true
DEDUPE_TTL_SECONDS=7200
LOG_LEVEL=INFO
```

### 3. 每部门一份 `.env.<dept>`（差异层）
仅放该部门独有的值。最小三项即可（`DEEPEXI_AGENT_ID` 是 `from_env` 必填项 `config.py:540`，必须在此提供）：
```
# .env.sales（销售部，示例）
WECOM_BOT_ID=<销售部机器人 bot_id>
WECOM_BOT_SECRET=<销售部机器人 secret>
DEEPEXI_AGENT_ID=<销售知识库 agent_id>

# 可选的部门级覆盖（按需）：
# PASSIVE_WELCOME_MESSAGE=你好，我是销售助手
# DEEPEXI_GROUP_ROUTES_JSON={"<该部门某群chatid>":"<另一个agent_id>"}
# DEEPEXI_GROUP_ALIASES_JSON={"<chatid>":"销售日报群"}
# DEEPEXI_APP_USER=sales   # 若要在滴普侧按部门区分用户归属
```
合并后必填项齐全：`WECOM_BOT_ID/SECRET`（dept）+ `DEEPEXI_BASE_URL/APP_ID/SECRET_KEY`（common）+ `DEEPEXI_AGENT_ID`（dept）。

### 4. 改造 `docker-compose.yml`
用锚点抽公共段，每部门一个 service（示例给 2 个，按同样模式扩到 4-8 个）：
```yaml
x-aibot-base: &aibot-base
  build:
    context: .
    dockerfile: Dockerfile
  image: wecom_deepexi_aibot:latest
  restart: unless-stopped
  init: true
  stop_signal: SIGINT

services:
  bot_sales:
    <<: *aibot-base
    container_name: wecom_aibot_sales
    env_file:
      - .env.common
      - .env.sales

  bot_ops:
    <<: *aibot-base
    container_name: wecom_aibot_ops
    env_file:
      - .env.common
      - .env.ops
```
要点：
- 现有写死的 `container_name: wecom_deepexi_aibot` 必须**改成每个 service 不同**（否则容器名冲突）。
- 多个 service 共享同一 `image` tag：`docker compose build` 只实际构建一次，其余命中缓存。
- 各容器内仍是单进程单连接 → Dockerfile 已有的进程级 `HEALTHCHECK`（`Dockerfile:26-27`，扫 `/proc` 找 `-m wecom_deepexi_aibot`）**精准且够用**，无需改动。

### 5. 补全凭据忽略（安全，必做）
当前 `.gitignore:1` 与 `.dockerignore:1` 只忽略精确的 `.env`。新增的 `.env.common`/`.env.<dept>` 含多份 bot secret，必须一并忽略：
- `.gitignore`：把 `.env` 一行改为同时覆盖 `.env`、`.env.*`（保留 `.env.example` 不被忽略，便于提交模板）。
- `.dockerignore`：同样补 `.env.*`（这些靠 `env_file` 运行时注入，不应进镜像）。
> 附带提醒：现有 `.env.example`/`.env` 里是**真实凭据明文**（含 bot secret、滴普 secret_key）。多部门会新增更多 secret，建议借此机会轮换已暴露的凭据，并确认它们从未被提交到 git 历史。

### 6. 更新文档（`.env.example` / README）
- `.env.example` 顶部加一段说明：多机器人请用 `.env.common` + `.env.<dept>` 分层，并保留单 bot 的原样示例（向后兼容，单容器用户不受影响）。
- README 部署章节补"多部门多机器人"小节：上面 4 步 + 启停命令。

## 关键文件
- `docker-compose.yml` — 改为多 service（锚点 + per-dept `env_file`），改 `container_name`。
- `.env.common`（**新建**）— 共享滴普凭据 + 行为配置。
- `.env.<dept>`（**新建，4-8 份**）— 各部门 bot 凭据 + 知识库 `agent_id`。
- `.gitignore` / `.dockerignore` — 补 `.env.*` 忽略。
- `.env.example` / `README.md` — 文档说明（可选但建议）。
- **不改**：`wecom_deepexi_aibot/` 下任何 `.py`（`config.py`/`__main__.py`/`message_handler.py`/`wecom_ws.py` 全部保持现状）。

## 验证（端到端）
1. **构建**：`docker compose build`（一次）。
2. **启动**：`docker compose up -d`。
3. **健康**：`docker compose ps` —— 所有 service 显示 `healthy`（约 20s start-period 后）。
4. **连接成功**：`docker compose logs -f bot_sales` 等，逐个确认：
   - `Starting WebSocket mode (bot_id=<该部门 bot_id> ...)`（`__main__.py:49`）—— 核对 bot_id 是该部门的。
   - 鉴权成功日志（`wecom_ws.py` auth 成功分支）。
5. **实连**：在每个部门机器人对应的群/单聊里发一条消息，确认**对应容器**有响应日志、且回复内容来自**该部门知识库**（可问一个只有该部门知识库才知道的问题验证 agent_id 路由正确）。
6. **隔离抽查**：给 A 部门机器人发消息，确认 B 部门容器日志无任何记录（验证去重/会话/连接互不串）。

可选的**离线配置预检**（不连企微，验证某份合并配置能被解析）：
```
docker compose run --rm --no-deps bot_sales python -c \
  "from wecom_deepexi_aibot.config import Settings; s=Settings.from_env(); print('OK', s.wecom_bot_id, s.agent_agent_id)"
```
> 注意：**不要**在宿主机 shell 里 `source .env.*` 后本地跑——记忆中本机 `.bashrc` 会自动 source `.env` 导致 JSON 引号被吃、loopback 被代理劫持（见 [[bashrc-sources-env-mangles-json]]）。预检请在容器内做（上面命令），环境干净。

## 风险与注意
- **bot_id 唯一性**：各部门必须不同 bot_id；切勿让同一 bot_id 在多个容器/进程并存（企微互踢）。
- **资源**：4-8 个容器 = 4-8 个 Python 进程，各自默认 8 线程池。压测基线（[[loadtest-capacity-baseline]]）单实例 workers=16 扛 16 路零失败；各容器独立互不争用，常规问答负载下宿主机压力可控。如某部门并发高，可单独给该容器调大 `AGENT_STREAM_MAX_WORKERS` 或加 `deploy.resources.limits`。
- **env_file 覆盖语义**：依赖 docker compose v2「后文件覆盖前文件」。若环境是更老的 compose v1，需改为每份 `.env.<dept>` 自包含全量配置（牺牲 DRY 换确定性）。
- **非容器备选**（如用 systemd）：等价做法是为每个部门起一个进程、各自指向自己的 env 文件，机制相同，本方案不展开。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (0)：本次会话未注入可调用 skills 清单
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily
- Agent types (7): claude, claude-code-guide, codex:codex-rescue, Explore, general-purpose, Plan, statusline-setup

**会用**：
- 本方案为零代码改动，执行阶段只用内置工具（Read/Edit/Write 改 compose 与 env，Bash 跑 `docker compose` 验证）。无需 Skill/MCP/专门 Agent。

**考虑过但不用**：
- MCP jina / tavily / firecrawl（三个相似的联网检索/抓取）：本任务是本地配置/部署改动，不需要联网，均不用。
- MCP chrome-devtools：无浏览器/前端调试需求。
- Agent Explore / Plan：已在调研、设计阶段使用；执行阶段是机械的配置编辑，不再 fan-out。
- 单进程多连接方案（改 `config.py` 引入 `WECOM_BOTS_JSON`、改 `__main__.py` 用 `asyncio.gather` 跑多连接）：已与多容器方案对比，用户选定多容器（零代码、天然隔离、单部门故障不波及他人），故不采用。
