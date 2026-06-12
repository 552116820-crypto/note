# 桥接服务（wecom-deepexi-aibot）服务器压测方案

## Context

服务已用 docker compose 部署到服务器，需要测它的并发量与响应时间。探索确认：服务**不是 HTTP 服务器**，而是企微 WebSocket 客户端（主动连 `wss://openws.work.weixin.qq.com`，每个 bot 仅允许一条连接），消息由企微推送 —— 所以 ab/wrk/JMeter 用不上，也不能另起连接模拟企微推送（会把线上连接踢下线）。

采用**注入式压测**：新增独立 harness 脚本，导入项目自身的 `MessageHandler`，用带时间戳记录的 fake WS client 替代真实连接，并发注入合成消息，统计延迟与吞吐。已确认：**Mock + 真实滴普都测；真实流量打 .env 现有生产 agent**。

## 处理链路与打点位（已读码核实）

- 装配方式镜像 `wecom_deepexi_aibot/__main__.py:32-47`：`Settings.from_env()` → `ThreadPoolExecutor(max_workers=settings.agent_stream_max_workers)`（默认 8）→ `MessageHandler(settings=, ws_client=, executor=)`。
- 入口 `handle_message(headers={"req_id":...}, body={...})`（`message_handler.py:463`）；body 只需 `msgtype/msgid/chattype/from.userid/text.content`。
- 回复路径即打点位（`message_handler.py:811-949`）：
  1. 进入流式阶段立即 `reply_stream(content="正在思考...", finish=False)` → **t_dispatch**（事件循环调度延迟）
  2. 之后才 `run_in_executor` 提交滴普请求（线程池排队发生在这之后）
  3. 首条真实内容 `reply_stream(finish=False)` → **t_first**（首响；受 0.8s/80 字符节流，线上同样存在）
  4. `reply_stream(finish=True)` → **t_done**；没等到即失败（失败路径走 `reply_text(reply_error_prefix)`）
- fake WS 需实现的全部方法已有现成范本：`tests/test_message_handler.py:83-141` 的 `FakeWsClient`（`new_stream_id / reply_stream / reply_text / reply_welcome / reply_file / reply_image / download_encrypted_media`），照抄加打点即可。
- 上游切换：`config.py:25` 的 `load_dotenv` 用 `os.environ.setdefault`，所以 harness 内**直接赋值 `os.environ["DEEPEXI_BASE_URL"]` 即可覆盖** .env / compose env_file。
- mock 必须兼容的 SSE 解析（`agent_chat.py:792-858`）：`data: {JSON}` 行，`answer` 字段作增量，`event: message_end` 结束，`data: [DONE]` 兜底；JSON 里 `code != 0` 会被当错误抛出。

## 实现：新增 `scripts/loadtest.py`（唯一新文件，服务本体零改动）

单文件自包含（stdlib + 项目包），四部分：

1. **RecordingWsClient** — 按 `FakeWsClient` 结构加 `time.monotonic()` 打点，按 req_id 归档 t_dispatch / t_first / t_done / 错误文本。
2. **MockDeepexiServer**（仅 `--upstream mock`）— `http.server.ThreadingHTTPServer` 后台线程，`POST /chat/chat-messages` 返回 SSE：
   ```
   data: {"answer": "<第i段>", "conversation_id": "mock-<uuid>"}
   ...
   data: {"event": "message_end", "conversation_id": "..."}
   data: [DONE]
   ```
   参数：`--mock-first-delay`（首字延迟，默认 1.0s）、`--mock-chunk-interval`（0.05s）、`--mock-chunks`（30）。启动后覆盖 `DEEPEXI_BASE_URL=http://127.0.0.1:<port>` 再 build settings。
3. **驱动器** — 对每个并发档 L（`--levels`，mock 默认 `1,4,8,16,32`，real 默认 `1,4,8`）：`asyncio.Semaphore(L)` 注入 `--per-level` 条消息（mock 默认 20，real 默认 5），每条：req_id=uuid、**msgid=uuid（绕过去重 `message_handler.py:478`）**、独立 userid=`loadtest-{L}-{i}`（每条新会话、样本独立；`--reuse-conversation` 测多轮续聊）、content=`--prompt`（real 默认"请只回复：收到"省 token）。
4. **统计输出** — 每档：成功/失败、吞吐(msg/s)、dispatch 延迟、首响、完整耗时的 p50/p95/max；终端表格 + 原始样本 JSONL 写 `scripts/loadtest_results/`（经挂载留在宿主机）。`--workers` 可覆盖线程池大小预演扩容；日志默认压到 WARNING（`--verbose` 恢复）。

另：`.gitignore` 加一行 `scripts/loadtest_results/`。

## 服务器上运行（不影响线上容器）

镜像只 COPY 了包目录（`Dockerfile:18`），scripts/ 不在镜像内 → 挂载方式起一次性容器，线上容器照常运行：

```bash
# 服务器上先 git pull 拿到脚本
# ① mock：测桥接自身极限（零成本）
docker compose run --rm --no-deps -v "$(pwd)/scripts:/app/scripts" \
  wecom_deepexi_aibot python scripts/loadtest.py --upstream mock --levels 1,4,8,16,32 --per-level 20

# ② real：小规模真实端到端（消耗滴普配额，默认仅 ~15 次调用）
docker compose run --rm --no-deps -v "$(pwd)/scripts:/app/scripts" \
  wecom_deepexi_aibot python scripts/loadtest.py --upstream real --levels 1,4,8 --per-level 5

# 旁路观察：另开终端 docker stats，看 harness 容器与线上容器的 CPU/内存
```

real 模式必须在服务器上跑（出口 IP 8.138.104.14 已加白名单，本地 WSL 出口没有）。

## 验证

1. 本地 WSL 先小规模 mock 干跑：`python scripts/loadtest.py --upstream mock --levels 1,4 --per-level 5`（repo 根目录有 .env，必填项齐全）。
2. 核对拐点特征验证打点正确性：mock 首字 1.0s、单条全程约 2.5s 时，L≤8 首响 p50 ≈ 1s；L=16 应明显抬升、L=32 进一步翻倍（8 线程排队）。
3. `pytest` 全量跑一遍，确认服务本体零改动、全绿。
4. 服务器执行 ①②，交付两张汇总表 + JSONL 原始数据。

## 结果解读要点

- 有效并发上限 ≈ `AGENT_STREAM_MAX_WORKERS`（8）；吞吐 ≈ workers ÷ 单条平均耗时。要扩容：.env 调大该值重启线上容器；可先 `--workers 16` 在 harness 里预演。
- real 模式 408/429/500 会被 `agent_chat.py` 自动重试（`agent_chat.py:99,272-298`），表现为长尾延迟而非失败；p95 异常大时查 harness 日志里的 retriable 告警。
- 局限：不含真实企微 WS 网络段/ack 往返（fake 即刻返回）与企微侧限流；首响有服务自身 0.8s 节流下限（线上同样存在）。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- （无 —— 实现与验证全部用内置 Read/Write/Edit/Bash 完成；服务器上的两条命令由你在服务器执行，或告知我 ssh 方式由我代跑）

**考虑过但不用**：
- verify / run：面向"启动本项目应用看效果"，本任务的验证就是直接运行新增的压测脚本，Bash 足够
- code-review：单文件新增脚本，写完自查；需要时可在合入前单独跑
