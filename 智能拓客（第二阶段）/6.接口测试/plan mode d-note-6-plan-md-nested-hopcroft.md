# 智能拓客 · 四接口边界测试 + 承接能力测试 + 前端对接文档

## Context（为什么做这件事）

`D:\note\智能拓客（第二阶段）\6.接口测试\人工plan.md` 要求：项目现有 **4 个接口**，补**边界测试 + 承接能力（并发/吞吐）测试 + 相关稳定性测试**，确保上线稳定；并产出一份可**直接交付前端**的接口文档（每个接口含：测试用例、请求体、返回体、注释）。

本计划已经过 **Codex 只读审查 + 我对源码逐条复核**。复核结论：Codex 的 A 类 10 项改进属实、已纳入；B 类 2 项被高估、已澄清（见下）。审查还挖出 2 个需改 app 代码才能根治的潜在隐患，用户已确认**一并修复 + 补测试**。

四接口（源码事实，文档与测试均以此为准）：

| # | 路径 | 用途 | 关键依赖 | 走大模型 |
|---|------|------|----------|----------|
| 1 | `POST /api/v1/leads/score-and-info` | 客户分数+基本信息，分页 | 业务库 amer_crm_v2、状态库 lead_skip、缓存 | 否 |
| 2 | `POST /api/v1/leads/skip` | 标记跳过（幂等 UPSERT） | 状态库 amer_sales_assist.lead_skip | 否 |
| 3 | `POST /api/v1/leads/timeline` | 行为时间轴（快路径） | CRM 三接口（串行）、detail 缓存 | 否 |
| 4 | `POST /api/v1/leads/recommend-reason` | AI 推荐理由（慢路径，可降级模板） | DeepSeek、复用 timeline、detail 缓存 | 是 |

### 经 Codex 审查的修正（评估结论）
- **B 类·已澄清（Codex 高估）**：① "EmployeeNotFound 是 404、与契约矛盾"——`grep` 证实 `EmployeeNotFound`/`InvalidCoordinates`/`InvalidCustomerId`/`EMPLOYEE_NOT_FOUND_404` **全是死代码，从未 raise/读取**；实际可达契约 = 业务结果→HTTP 200、**仅两库不可用(1503 业务库/1505 状态库)→503**。② "无 catch-all→裸 500"——属实但 services 已尽力不抛，仅"未预期 bug"可达；属加固范畴 → 已列入交付物 0 修复。

### 决策（已与用户确认）
- 测试范围：**两者都要**（tests/ pytest 隔离 + scripts/ 真实压测）。
- 压测工具：**标准库线程脚本**（沿用 `scripts/http_smoke.py`：线程内 uvicorn + urllib + `ProxyHandler({})`），零新增依赖。
- 压测环境：**现在就跑**（业务库/状态库可达、`eth2` MTU 已设 1400、`DEEPSEEK_API_KEY` 已配）。
- 修复范围：**两个隐患都修 + 补测试**（见交付物 0）。

---

## 交付物 0：app 代码小修（低风险，含回归测试）

1. **`app/schemas/request.py`** —— 给 `LeadQueryRequest`、`LeadSkipRequest` 的 `employee_code` 加 **strip + 拒空白**校验器（复用 `RecommendReasonRequest._strip_employee_code:110-116` 的现成模式）。**根治缓存投毒**：现状缓存键 strip（`cache/store.py:26`）、SQL 用原始值（`employee_repo.py:44`、`lead_service.py:147`），导致 `" 000068 "` 先到会把空结果写进 strip 后的键、污染干净请求。input 端归一化后两边口径一致。
2. **`app/main.py`** —— 加 `@app.exception_handler(Exception)`：`logger.exception(...)` 记 traceback，返回 `JSONResponse(status_code=500, content=envelope(code=1500, message="服务内部错误，请稍后重试", data=None))`，**不泄露异常细节**。未预期异常也守住 envelope 合约。（注：该 handler 由最外层 ServerErrorMiddleware 触发，api_log 对这类请求会记 `biz_code=null`，可接受。）
3. **`app/errors.py`** —— 新增错误码 **1500（服务内部错误，http 500）**；文档错误码表同步。

---

## 交付物 A：pytest 隔离测试（tests/，全 mock、CI 可跑、零新增依赖）

范式：`TestClient(app)` + `monkeypatch` 桩服务层（校验错误发生在服务层之前，可断言"服务未被调用"）；缓存类直接 new `InProcessTTLCache`，不碰全局缓存。

1. **`tests/test_request_schema_bounds.py`** —— `app/schemas/request.py` 校验器边界：lat/lng ±90/±180 边界、radius(0/负/>1e5/>200 保留)、`page_size=500`→截到 100 不报错、默认值；`customer_id` int→str/前导零/非数字拒；**employee_code（含 0 修复）**：query/skip 现 strip+拒空白、`" 000068 "`→`"000068"`、`"   "`→1002；timeline→None；recommend→拒空白。
2. **`tests/test_api_boundary.py`** —— 4 接口 HTTP 输入边界（→1002）+ 合法边界（→200）+ envelope 形状；非法输入断言服务桩**未被调用**，合法断言参数已归一化。**（Codex）补**：畸形 JSON、非对象 body（`[]`/`"x"`）、`score_context` 类型错 → 均 1002。
3. **`tests/test_pagination_radius.py`** —— `query_leads` 承接边界（monkeypatch `cache.get_or_compute` 返回构造 blob）：分页数学（total/total_pages=ceil/has_more/越界页→空）、半径 clamp 回显（500→requested/actual；≤200→缺省）、unseen/skipped 计数。
4. **`tests/test_api_errors.py`** —— 异常处理器→错误码映射：`DBUnavailable`→503/1503、`TooManyCandidates`→1003、`LogDBUnavailable`→1505；**（0 修复）未预期异常→HTTP 500 + envelope code 1500**（`TestClient(raise_server_exceptions=False)`）；降级透传（timeline degraded、recommend reason_source）；**（Codex）补 1 个端到端降级用例**：只 fake CRM+DeepSeek 客户端、不桩 service，使 TestClient 真正走通 `recommendation_service` 内部 template 兜底；unknown employee→软 code 0（EmployeeNotFound 死代码不测）。
5. **`tests/test_capacity_cache.py`** —— `InProcessTTLCache` 真线程并发：singleflight（同键 compute 恰好 1 次、各线程同值）、不同键各算 1 次、超 max_keys→LRU 淘汰且 size 有界、过期→miss、`get_or_compute_cacheable` 降级(cacheable=False)每次重算不污染。**（Codex）补**：`_key_locks` 过期不回收的观察性断言（`get()` 过期只 pop `_data`，`store.py:76-78`）；**显式限定 InProcessTTLCache**，注释 Redis 后端无 singleflight（`store.py:118-120`，按设计）。
6. **`tests/test_recommend_cache_fingerprint.py`（新，Codex §3.7）** —— 指纹稳定性：相同稳定字段→同 key 命中；混入易变 extra（如时间戳）→`_ctx_fingerprint`（`recommendation_service.py:58-62` 哈希全量 context）变→miss。佐证 token 成本风险，呼应文档强约束。
7. **`tests/test_api_log_backpressure.py`（新，Codex §3）** —— hermetic：`_pending≥api_log_max_pending` 时 `_schedule_write` 丢弃且不抛、正常时入队（复用 `test_api_log_*.py` 的同步记录器 monkeypatch 范式）。

---

## 交付物 B：真实压测脚本 `scripts/load_test.py`（标准库，打真实后端）

沿用 http_smoke：线程内起 uvicorn(127.0.0.1:8078) + urllib + `ProxyHandler({})`、自清理、`argparse`/env 调并发与轮次（默认温和控 DeepSeek 花费）。

- **Preflight 闸口（Codex §2，硬性）**：先 GET `/healthz` 断言 `db==true`、`industries_source=="db"`，再跑一次小半径 score-and-info 确认候选>0；任一不满足→打印 MTU/冷启/快照回退提示并**中止**（避免 resolver 静默回退快照 `industry_resolver.py:41-43` 导致数据失真）。
- **进程内计数器（Codex §2）**：wrap `lead_service._compute_blob`（线程安全计数），击穿阶段直接断言"冷算==1"，不靠时延推断。

阶段（量化承接能力）：
1. 预热/冷启（记冷延迟）。
2. 热路径吞吐（同缓存键，QPS/p50/p95/p99/错误率）。
3. 击穿/singleflight（并发打新键 + 计数器断言冷算==1）。
4. **冷路径吞吐（新，单列）**：多组不同 (emp,coord,radius) 并发强制 miss → 量化真实 DB/算法承载，与热路径分开汇报。
5. timeline 吞吐（**冷/热分列**；degraded 率、sources 健康；注明冷路径=串行 3 次 CRM×5s）。
6. recommend 吞吐（**变 score_context 破缓存 + 断言首批 `reason_source=="llm"`**，确保真测到 LLM 而非静默落模板/缓存；次数可配默认小、注明 token 成本；preflight 选一个有 timeline 的 customer）。
7. skip 并发幂等（并发跳同一 (emp,cid)→均 code 0、仅一行，用完删行自清理）。
8. **API 日志开/关两态对比（新，Codex §3）**：各跑一轮，报告吞吐差 + 丢弃率（`_pending` 背压 `api_log.py:162-167`）。
9. 混合负载（四接口按比例，总 QPS/错误率）。

输出：每阶段表（请求/并发/成功/错误/QPS/p50/p95/p99）+ 阈值 PASS/FAIL；开头 MTU 提醒。

---

## 交付物 C：前端对接文档

写入 `/mnt/d/note/智能拓客（第二阶段）/6.接口测试/接口文档，对接前端.md`（即 `D:\note\...\接口文档，对接前端.md`）；**并在 repo 内放一份副本** `docs/接口文档-对接前端.md`（Codex §5，便于 review/CI，低成本）。中文、面向前端、字段逐条注释。

- **全局约定（修正后契约）**：Base URL 占位；三段式 envelope（code=0 成功）；业务接口 `POST`+`application/json`，`GET /healthz`；当前**无鉴权**（注明上线网关另议）；HTTP 状态：**业务结果恒 200；仅两库不可用→503（1503 业务库/1505 状态库）；未预期错误→500（code 1500）**；错误码表 **0/1002/1003/1500/1503/1505**（1001/1004 为源码保留、当前不会返回，注明以免前端误判）；降级约定（接口 3/4 全程降级，靠 `degraded`/`sources`/`reason_source` 标记不报错）。
- **每个接口**：路径+方法+用途+依赖+冷/热耗时；**请求体**字段表（名/类型/必填/默认/取值校验/注释）+示例；**返回体**完整字段表（嵌套 employee_context/pagination/list[]/score_detail/timeline[]，逐字段注释，标注 `"-"`/`[]` 兜底、`number|"-"` 联合、可选字段何时出现）+示例；错误/降级返回；**测试用例表**（正常/边界/异常/降级，与交付物 A 对应）；**对接注意（Codex 强化）**：
  - score-and-info：employee_code 会 strip（前后空格无影响）；`page_size>100` 自动截断不报错。
  - timeline：**冷路径=串行 3 次 CRM、各 5s 超时（最坏数十秒）**，命中缓存才毫秒级；与业务员无关可不传 employee_code。
  - recommend-reason：**`score_context` 只回传接口二原样的稳定字段，切勿带时间戳/随机值**，否则每次 cache miss、徒增延迟与 LLM 成本；`degraded`/`reason_source` 含义。
- **附录**：四接口一览、错误码总表、`"-"`/`[]` 约定速查、**压测结论摘要（交付物 B 实测）**。

---

## 验证（端到端）

1. 隔离测试：`python3 -m pytest tests/ -q` → 现有 + 新增 + 2 个 app 修复回归全绿。
2. 既有冒烟回归：`python3 scripts/http_smoke.py`、`python3 scripts/integration_check.py`（确认 strip 校验器 + catch-all 未破坏现有路径）。
3. 真实压测：`sudo ip link set dev eth2 mtu 1400` 后 `python3 scripts/load_test.py` → preflight 通过→各阶段报告，摘要写回文档 C 附录并回报用户。
4. 文档 C：每接口请求/返回字段与源码逐一核对、示例可直接复制联调。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent `Explore`：Phase 1 已用于盘点 4 接口契约与测试基建。
- Agent `codex:codex-rescue`：已用于对本计划做对抗式审查（结论已并入本计划）。
- 内置 Bash：跑 `pytest`、`load_test.py`、`http_smoke.py`、`integration_check.py`。
- （可选）Skill `code-review`：新写测试/脚本/app 小修完成后自审一遍。

**考虑过但不用**：
- `claude-api`：接口四用 **DeepSeek（OpenAI 兼容协议），非 Anthropic**；按该 skill 自身规则跳过。
- `run` / `verify`：压测脚本自身在线程内起 uvicorn，无需拉起/验证应用。
- `deep-research` / MCP `tavily`·`jina`·`firecrawl` / `chrome-devtools`：本任务不需联网检索或浏览器自动化（契约全部来自本地源码）。
