# 开发"日志功能"：API 流量日志（amer_sales_assist.lead_api_log）

## Context（为什么做）

`D:\note\...\5.开发"日志功能"\人工plan.md` 的需求：做一个**接口流量日志**，用于**汇报内容和阶段审查**——看清"谁、调了哪个接口、多少次、多快、成不成、返回多少候选"。你已在 `amer_sales_assist` 建好 `lead_api_log` 表（现 **0 行**，全新功能），并邀请我按更好的方案做改动。

调研结论：表设计得很到位，**无需改表**；功能本身缺的是"把每次调用写进去"的那一层。落点是一个 **HTTP 中间件**，因为要同时拿到①请求体里的 `employee_code`②**最终响应** envelope 的 `code`/`total`（失败响应由 `main.py` 的 5 个全局异常处理器产出，只有读最终响应才能捕获失败）。

已与你确认的三项决策：
- **记录范围**：4 个业务 POST 接口的**成功 + 失败**都记，跳过 `/healthz`。
- **request_body**：**精简存储**——剥掉 recommend-reason 里庞大的 `score_context`，超限再截断为合法 JSON 标记。
- **汇报产出**：暂不做统计脚本/接口，先只把日志落表。

## 目标表（已存在，保持不变）

```sql
lead_api_log(
  id           bigint unsigned PK auto_increment,
  employee_code varchar(16) NULL,   -- 业务员工号（取自请求体）
  endpoint     varchar(64) NOT NULL,-- = request.url.path
  request_body json NULL,           -- 精简后的请求体快照
  biz_code     int  NULL,           -- = 响应 envelope.code（0=成功；失败如 1002/1503/1505）
  result_total int  NULL,           -- = 响应 data.pagination.total（仅 score-and-info 有，余为 NULL）
  cost_ms      int  NULL,           -- 实测耗时
  created_at   datetime default CURRENT_TIMESTAMP,
  KEY (employee_code, created_at), KEY (endpoint, created_at)
)
```
不加 `http_status` 列：`biz_code` 已能区分（1503/1505 即 HTTP 503），冗余。`request_body` 体积问题在应用层解决（精简），非改表。

## Codex 审查结论（已并入）

Codex 独立对照源码审查：**核心方案 SOLID**（纯 ASGI 中间件 + 请求体重放 + `send` 包裹捕获最终响应字节取 `biz_code`，以及与 `upsert_skip` 不同的降级策略，均判定正确）；判 **NEEDS-CHANGES** 的是 5 项工程化加固，已全部并入：

| 级别 | 问题 | 并入位置 |
|---|---|---|
| P2 | 默认 executor 无背压、故障写最坏 ~15s（非 5s）会积压 | 技术决策 2 + 步骤 2（专用有界池 `_LOG_EXECUTOR` + `API_LOG_MAX_PENDING` 丢弃背压 + 更正耗时） |
| P2 | fire-and-forget 的持久性/关停语义未声明 | 步骤 4（关停短超时排空 + 显式 **best-effort、非审计级** 声明） |
| P3 | 前缀匹配比"4 个接口"宽，会误收新路由/404/错方法 | 步骤 2（显式 `{method,path}` 白名单 `_LOGGED`） |
| P3 | `result_total` 按响应形状取，将来别的接口会误记 | 步骤 2（`_RESULT_TOTAL_PATHS` 按字面路径门控） |
| P3 | 依赖 Starlette 0.47.3 行为但 `requirements.txt` 没钉版本 | 步骤 7（钉 fastapi/starlette 区间；且纯 ASGI 只依赖稳定规范） |

## 关键技术决策（已用 Plan 子代理核实 Starlette 0.47.3 源码）

1. **用纯 ASGI 中间件**（`async def __call__(self, scope, receive, send)`），**不用 `BaseHTTPMiddleware`/`@app.middleware("http")`**。原因：核实到 BaseHTTPMiddleware 里给响应挂的 `BackgroundTask` 会在 flush **之前**执行（`await response(...)` 先于 `response_sent.set()`），所谓"响应发完再写、零延迟"在它身上**不成立**；纯 ASGI 可"先把每个 `send` 消息转发出去，再调度写库"，真正零延迟，且绕开 `body_iterator` 重建/`Content-Length` 一类坑。注：上面那条 flush 时序是 Starlette 0.47.3 的实现细节；而**最终选的纯 ASGI 方案只依赖稳定的 ASGI 规范语义、不依赖该版本细节**，所以更稳健。为防未来安装漂移到未实测的版本，按步骤 7 在 `requirements.txt` 钉住实测版本对（已并入 Codex P3）。
2. **写库绝不阻塞 event loop，且有背压**（已并入 Codex P2）：中间件跑在事件循环上，而 `log_pool.execute()` 是阻塞 PyMySQL；连不上最坏约 **~15s**（connect_timeout 5s × `log_db_connect_retries`(2) + 退避 sleep，见 `log_pool.py:54,60`、`settings.py:42`），**不是 5s**。因此用**专用有界线程池** `_LOG_EXECUTOR = ThreadPoolExecutor(max_workers=2)` 跑写库——与 sync 路由用的**默认** anyio 线程池**隔离**，避免日志写挤占请求处理线程或拖累其他默认 executor 用户；`loop.run_in_executor(_LOG_EXECUTOR, _safe_write, record)` fire-and-forget、不 `await`。加**丢弃式背压**：用一个原子计数 `_pending`（提交 +1，future 完成回调 -1）跟踪在途写；超过上限（`API_LOG_MAX_PENDING`，默认 200）就**直接丢这条日志 + 一条限频 warning**，绝不无限堆积（持续故障下避免内存/线程积压）。这优于 `run_in_executor(None,…)`（会和请求线程抢默认池）与 `asyncio.create_task(run_in_threadpool(...))`（无背压、且易出"Task exception never retrieved"噪声）。
3. **绝不因日志失败而 500**：与 `lead_skip_repo.upsert_skip`（故意让 `LogDBUnavailable` 上抛）**相反**——流量日志是辅助，必须吞掉一切异常。两层 `try/except`：`_safe_write` 内一层、调度处一层。遵循全仓"优雅降级"铁律。
4. **不经 Redis**：按 memory「合并时 Redis 的坑」，共享实例若开了淘汰策略会**静默丢键**，拿它缓冲日志会丢数据。日志直写 MySQL（持久），Redis 只留作缓存。

## 实现步骤（文件级）

**1. `app/config/settings.py`** —— 加 kill-switch（与既有 `_flag` 风格一致，默认关）：
```python
enable_api_logging: bool = _flag("ENABLE_API_LOGGING", "0")
```
并在 `.env` 写 `ENABLE_API_LOGGING=1`（本部署开启），`.env.example` 注明"默认 0，置 1 开启"。每请求读 `settings.enable_api_logging`（`settings` 是 `@lru_cache` 单例，测试沿用既有 monkeypatch 属性的方式）。

**2. `app/middleware/api_log.py`（新建 + `app/middleware/__init__.py`）** —— 纯函数 + ASGI 壳，逻辑全放可单测的自由函数里：
- **显式接口白名单**（已并入 Codex P3，取代"前缀匹配"——避免误收未来新路由、误用的方法、`/api/v1/...` 的 404）：
  ```python
  _LOGGED = {
      ("POST", "/api/v1/leads/score-and-info"),
      ("POST", "/api/v1/leads/skip"),
      ("POST", "/api/v1/leads/timeline"),
      ("POST", "/api/v1/leads/recommend-reason"),
  }
  _RESULT_TOTAL_PATHS = {"/api/v1/leads/score-and-info"}   # 仅此接口取 result_total
  ```
- `MAX_BODY_BYTES = 16*1024`、`_DROP_KEYS = ("score_context",)`、`API_LOG_MAX_PENDING = 200`、`_LOG_EXECUTOR = ThreadPoolExecutor(max_workers=2)`
- `trim_request_body(raw: bytes) -> str | None`：解析一次 → 浅拷贝去掉 `_DROP_KEYS` → `json.dumps(ensure_ascii=False)`；仍超限则整体换成合法 JSON 标记 `{"_truncated": true, "_bytes": n}`（**不要**对 JSON 字节切片，会变非法 JSON）；空/解析失败 → `None`，绝不抛。
- `extract_employee_code(parsed) -> str|None`：防御式取 `employee_code`，强转 str 截到 16 字符，缺/空/非标量 → `None`。
- `extract_biz_code(resp_body, content_type) -> int|None`：从**最终响应字节**解析 envelope `code`；非 JSON/缺字段 → `None`。
- `extract_result_total(resp_body, content_type, endpoint) -> int|None`：**先按接口门控**（已并入 Codex P3）——`endpoint not in _RESULT_TOTAL_PATHS` 直接返回 `None`，只有 `/score-and-info` 才解析 `data.pagination.total`；这样将来别的接口即便也返回 `pagination` 也不会被误记。非 JSON/缺字段 → `None`（`result_total` 永不默认 0——0 个候选与"无此字段"含义不同）。
- `build_log_record(*, endpoint, request_raw, resp_body, content_type, cost_ms) -> dict`：组装列值，纯函数、吞一切。
- `class ApiLogMiddleware`：`scope["type"]!="http"` 或 `not settings.enable_api_logging` 或 `(scope["method"], scope["path"]) not in _LOGGED`（即排除 `/healthz`、`/docs`、未来新路由、误用方法、`/api/v1/` 404）→ 直接放行。否则：把 `receive` 的 body 全缓冲、再用自定义 `receive` **重放**（下游 sync 路由照常解析）；包 `send`，遇 `http.response.start` 记 status/content-type、遇 `http.response.body` 追加副本，**先转发**再累积；`finally` 里算 `cost_ms`、`build_log_record`，再调 `_schedule_write(record)`。
- `_schedule_write(record)`（背压闸，已并入 Codex P2）：`if _pending >= API_LOG_MAX_PENDING: logger.warning(限频) ; return`（丢弃）；否则 `_pending += 1`，`fut = loop.run_in_executor(_LOG_EXECUTOR, _safe_write, record)`，`fut.add_done_callback(lambda _: _dec_pending())`。整段再包一层 `try/except`（调度本身也绝不抛）。
- `_safe_write(record)`：`try: lead_api_log_repo.insert_log(record) except Exception: logger.warning(...)`（跑在 `_LOG_EXECUTOR` 线程里）。

**3. `app/repositories/lead_api_log_repo.py`（新建）** —— 仿 `lead_skip_repo`，但写库可上抛（由 `_safe_write` 兜底）：
```python
from app.db import log_pool
def insert_log(rec: dict) -> None:
    log_pool.execute(
        "INSERT INTO lead_api_log "
        "(employee_code, endpoint, request_body, biz_code, result_total, cost_ms) "
        "VALUES (%s,%s,%s,%s,%s,%s)",
        (rec["employee_code"], rec["endpoint"], rec["request_body"],
         rec["biz_code"], rec["result_total"], rec["cost_ms"]))
```
`id`/`created_at` 由库默认。

**4. `app/main.py`** —— 在 `include_router` 之后注册（`add_middleware` 最后加 = 最外层，才能看到异常处理器产出的最终响应字节）：
```python
from app.middleware.api_log import ApiLogMiddleware
app.add_middleware(ApiLogMiddleware)
```
并在现有 `lifespan` 的关停段（`main.py:34` `yield` 之后）**短超时排空在途写**（已并入 Codex P2）：等 `_LOG_EXECUTOR` 的在途 future ≤ 一个小超时（如 `asyncio.wait(..., timeout=2.0)` 或轮询 `_pending` 几百 ms），再 `_LOG_EXECUTOR.shutdown(wait=False)`；超时就放弃剩余（best-effort，不拖慢关停）。整段 `try/except` 包好，关停永不报错。

> **语义声明（已并入 Codex P2）**：本流量日志是 **best-effort（尽力而为）**，**非审计级**——进程被强杀/重载、或持续故障触发背压丢弃时，可能丢少量日志；写入顺序也不保证（`created_at` 仍单调可用于排序/统计）。用途是流量汇报与阶段审查，这一可靠性档位足够；若日后要"零丢失"，再上"有界 `asyncio.Queue` + 单 drain 协程 + 关停 flush"，仍不经 Redis。

**5. `.env.example`** —— 加 `ENABLE_API_LOGGING=0` 一行 + 注释。

**6. 测试**
- `tests/test_api_log_helpers.py`（纯函数、无 DB，仿 `test_skip_layering.py`）：`trim_request_body` 去 `score_context`/超限换标记且 `json.loads` 可往返/空→None；`extract_employee_code` 各分支；`extract_biz_code` 对成功 envelope、`1002` 失败 envelope、非 JSON 字节；`extract_result_total` 在 `/score-and-info` 取到 total、在**非** `/score-and-info`（`/skip`/`/timeline`/`recommend-reason`）一律 `None`（**接口门控**，已并入 Codex P3）。
- `tests/test_api_log_scoping.py`（或并入降级测试，`TestClient`）：打开 flag，断言**白名单内**接口落行、**白名单外**（`GET /healthz`、`/docs`、`/api/v1/` 的 404、错误方法）**不**落行（已并入 Codex P3）。
- `tests/test_api_log_degradation.py`（`fastapi.testclient.TestClient` 走完整 ASGI 栈）：monkeypatch `log_pool.execute` 抛 `LogDBUnavailable` + 打开 flag，POST 合法请求，**断言响应仍 `code==0`、HTTP 200、无异常冒泡**（写在 `_LOG_EXECUTOR` 线程里、`_safe_write` 吞掉异常）。

**7. `requirements.txt`（已并入 Codex P3）** —— 把实测可用的版本对从 `fastapi>=0.116` **钉成区间**，例如 `fastapi>=0.116,<0.117` + 显式加 `starlette>=0.47,<0.48`，避免将来安装漂移到未实测的 Starlette、踩到 ASGI 之外的行为变化。（纯 ASGI 方案本身只依赖稳定规范，钉版本是双保险。）

## 复用既有实现（不要新造）

- `app/db/log_pool.py`：`execute()`/`fetch_all()`/`LogDBUnavailable` + 短超时，直接用。
- `app/repositories/lead_skip_repo.py:19` `upsert_skip`：写库范式模板。
- `app/errors.py` `envelope()` + `app/main.py:53-80` 五个异常处理器：保证失败也输出统一 `{code,...}`，中间件读它即得 `biz_code`。
- `app/config/settings.py:19` `_flag()`：开关范式。
- 端点位置：`leads.py:19/27`（score-and-info/skip）、`customer_detail.py:18/25`（timeline/recommend-reason）。

## 正确性要点（实现时必须守住）

- **绝不抛 / 绝不阻塞循环**（见上技术决策 2、3）。
- **请求体读后必须重放**：否则下游 pydantic 读到空 body、每次 1002。
- **响应只观察不消费**：转发原 `send` 消息、只追加 body 副本——不重建响应、不动 headers/Content-Length。
- **专用有界线程池 + 丢弃式背压**：日志写走 `_LOG_EXECUTOR`（与请求线程隔离），在途超 `API_LOG_MAX_PENDING` 即丢；持续故障最坏单次写阻塞 ~15s（仅占 executor 线程，不碰请求）。
- **best-effort 语义**：强杀/重载/背压丢弃下可能丢少量日志，顺序不保证（非审计级）；关停短超时排空在途写。
- **接口白名单门控**：只记 `_LOGGED` 里的 4 个 `{method,path}`；`result_total` 仅 `/score-and-info` 解析。
- **`employee_code` 截到 16 字符**（`/timeline` 可为空 → NULL）。

## 验证（端到端）

1. `pytest tests/`：单元（helpers）+ 降级用例全绿。
2. 手动实跑：`ENABLE_API_LOGGING=1 uvicorn app.main:app`，分别 POST 四个接口各一次 + **故意发一个校验失败**的请求；`SELECT endpoint,employee_code,biz_code,result_total,cost_ms FROM lead_api_log ORDER BY id` 应看到 5 行：成功行 `biz_code=0`、score-and-info 行 `result_total` 有值、失败行 `biz_code=1002`、`cost_ms` 合理、`/healthz` **无行**、recommend-reason 行的 `request_body` 已无 `score_context`。
3. `scripts/http_smoke.py` 追加一段自清理断言（仿其 `/skip` 段：发 score-and-info → 轮询查到新行 → 校验后 `DELETE`，整段失败降级为 SKIP 不拖垮测试）。
4. **降级实测**：临时把 `LOG_DB_HOST` 指到不可达地址，重发接口，确认仍 `code:0` 且响应不变慢（写在后台线程池、不 `await`）。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent `Explore` / `Plan`：本次规划已用——Explore 摸清 HTTP/DB/约定三层，Plan 子代理核实 Starlette 0.47.3 中间件/后台任务时序（揪出 BaseHTTPMiddleware 的 flush 时序坑）。
- Skill `codex:rescue`：已用——让 Codex 独立对照源码复审本 plan，判 **SOLID core + 5 项加固**，已全部并入（见上「Codex 审查结论」）。（首跑 shared runtime 约 70s 崩溃，已 `--fresh --wait` 重跑成功。）
- Skill `code-review`：实现后过一遍新中间件 diff——body 缓冲/重放与"绝不抛"降级是最易错处，值得专门查 bug。

**考虑过但不用**：
- `verify` / `run`：两者都能起服务实跑，但本仓已有 `scripts/http_smoke.py`（real uvicorn + 落行断言 + 自清理 DELETE），复用它更贴合既有约定，故走脚本而非这两个 skill。
- `simplify`：与 `code-review` 重叠（都过 diff）；本任务正确性优先（中间件 body/降级易错），选查 bug 的 `code-review` 而非只做整洁的 `simplify`。

> 内置 Read/Edit/Write/Bash 及 `pytest`/`scripts/*` 经 Bash 运行属基础能力，不另列。MCP（chrome-devtools/firecrawl/jina/tavily）均为联网/浏览器检索，本任务纯后端、答案在本仓库，与任务无关。
