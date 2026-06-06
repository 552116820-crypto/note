# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `04568e01-de42-4b21-91c6-31d0b4776c1b`
- **范围**: Turn 1-3
- **导出时间**: 2026-06-06 10:51:03

---

### Turn 1  *(2026-06-06 01:13)*

**User:**

D:\note\智能拓客（第二阶段）\4.行为时间轴和AI推荐接口\人工plan.md，仔细阅读这个文档

**Assistant:**

I'll read that document. The Windows path `D:\note\...` maps to `/mnt/d/note/...` in WSL.

I've read the requirements document. Let me get oriented on the project structure and the broader note folder before launching exploration agents.

The project is a FastAPI app (routers/services/schemas/repositories/db/cache/config). This is folder 4 of a multi-part stage-2 effort. Let me launch parallel Explore agents to understand the existing patterns I'll need to build on.

Excellent — I now have a clear architectural picture. Let me read the most critical files directly to ground the plan precisely before designing.

Now the service orchestration, schemas, scoring, deps, and env files.

Now scoring, dependencies, env files, and the geo repo (to see if a single-customer fetch exists for score recomputation).

Let me check the live `.env` for any already-configured CRM/AI keys (names only), and confirm the customer_id ↔ CRM mapping.

Confirmed the key facts: no existing HTTP/CRM/LLM code (clean slate for integrations), `.env` has no AI/CRM keys yet, and crucially the app's `customer_id` is literally `crm_customer.id` — exactly what the CRM APIs expect as `customer_ids`, so **no id mapping is needed**.

I have a strong design forming. Let me launch one Plan agent to pressure-test the trickiest decisions and surface risks before I finalize.

The Plan agent's design aligns with and sharpens my thinking. I now have a complete picture. Before finalizing, there are three genuine forks where your intent changes the architecture — and one of them pushes back on the literal wording of your doc. Let me confirm these with you.

Your answers are clear, and your Q3 custom answer actually improves the design. Let me lock in the decisions:

1. **时间轴构建** → 代码确定性拼装；AI 只写推荐理由（精确字段零幻觉）。
2. **AI 输出** → v1 只产出一段 `recommend_reason`（不做逐条总结、不做联网爬取）。
3. **分数来源** → 你的思路比"传一个数字"更好：前端把 `/score-and-info` 已加载的**业务员上下文 + 单个客户信息**随请求带上；后端**再补充"分数计算规则"文本**一起喂给大模型，让它基于评分逻辑 + 行为时间轴说出推荐理由。无需后端重算分。

Now writing the final plan.

Plan is written. Let me load the ExitPlanMode tool and present it.

I've finished reviewing your document and have a complete, hardened plan written to the plan file. Here's the gist of what I'm proposing — and where I optimized/supplemented your draft:

**核心优化（相对你的原始需求）**
- **时间轴改为代码确定性拼装**，大模型只写推荐理由 —— 你要的是"最可靠"，而订单号/金额/日期过大模型恰恰最不可靠，这样改零幻觉、可单测、低延迟。
- **采纳你 Q3 的思路并落地**：前端回传业务员+单客户信息，后端再补一段"评分规则说明"，让 AI 基于评分逻辑+时间轴说理由（而非只塞一个分数）。
- **必返合法 JSON 的三层保证**：DeepSeek json 模式 → Pydantic 单字段校验 → 带错误反馈重试 1 次 → 确定性兜底模板，绝不报错。
- **全链路降级**（沿用你"跳过库连不上就当无跳过"的既有风格）：CRM 三接口任一失败只标 `degraded/sources`，仍 `code:0`。
- **补强**：passport 时区固定 Asia/Shanghai + salt 入配置；缓存 key 含日期桶（正确性必需）；提示注入防护 + 时间轴喂模型截断 Top-N；新建独立 `recommendation.py` 路由不污染纯 DB 模块。
- **上线前探针**确认 6 个 CRM/DeepSeek 未知项（passport 时区、返回结构、空 employee_code 语义、出网代理、deepseek v4 确切 model id 等）。

你提出的 `sales_ids`/`employee_code` 是否填 —— 建议**只按 `customer_ids` 查、另两者留空**（拿该客户完整历史），但需探针确认空 employee_code 的语义。

完整设计（含文件清单、时间轴映射表、缓存/配置/验证、以及"会用到的能力"盘点）已写入计划文件。

You're right — and it's a good catch. Once the timeline is deterministic code (fast, no LLM) and the reason is LLM-backed (slow, can fail), the two halves have **completely different latency, failure, and caching profiles**. Bundling them forces the fast half to wait on the slow half. Splitting is the better design:

- **渐进式渲染**：`/timeline` 毫秒级返回先把时间轴画出来；`/recommend-reason` 跑大模型（可能 2–5s）单独异步加载，用户不用干等。
- **故障隔离**：时间轴（CRM）挂了是硬伤；理由（大模型）挂了无所谓。拆开后时间轴接口根本不背大模型的失败语义。
- **缓存粒度不同**：时间轴按 `customer_ids` 查、`employee_code` 留空 → **与业务员无关**，可按（客户+日期）全员共享缓存；理由依赖业务员成交上下文 → 必须按（客户+业务员+日期）缓存、且每次 miss 要花钱。拆开后两种缓存各自干净。

**唯一要定的点**：理由接口仍需要时间轴做输入。我的方案是——理由接口**内部复用同一个带缓存的时间轴取数函数**（`timeline_service.get_timeline(...)`），命中前一步 `/timeline` 写入的缓存即可，**不必让前端把时间轴回传**（避免请求臃肿 + 重复抓 CRM）。前端两个接口可并行发起。

我据此把计划改成"两个接口 + 共享时间轴取数"。更新计划文件：

计划已更新为**两个接口 + 共享时间轴取数内核**的设计。要点：

- **接口1** `POST /api/v1/leads/timeline`：纯 CRM、毫秒级、无大模型，返回全量行为时间轴。
- **接口2** `POST /api/v1/leads/recommend-reason`：内部复用接口1的带缓存取数函数拿到时间轴，叠加评分规则+业务员上下文喂大模型，返回一段 `recommend_reason`，全程可降级。
- 两接口共享 `timeline_service.get_timeline()`，时间轴缓存与业务员无关全员共享，理由缓存按（客户+业务员+日期）。
- 其余（确定性时间轴、JSON 三层校验兜底、passport 时区/salt、提示注入防护、上线前探针、端到端验证、能力盘点）均保留。

完整内容在计划文件里。

Good — both are important. Let me answer the timezone question directly, because it's the crux of the CRM auth:

**服务器是 UTC —— 影响很大，但我的设计已经规避了。** passport = `md5(当天日期 + salt)`，而 `crm.amer.cn`（cn 主机）几乎一定按**北京时间**校验。如果用 UTC 服务器的本机日期：

- 北京时间 **00:00–08:00** 这 8 小时，UTC 还停留在**前一天**（UTC = 北京 − 8h）。
- 于是这段时间我们算出的 passport 是"昨天"的，CRM 期望"今天"的 → **每天约 1/3 时段 CRM 调用全部鉴权失败**。

所以**绝不能用本机 `date.today()`**。`compute_passport` 用 `datetime.now(ZoneInfo("Asia/Shanghai"))` 显式取北京日期，与服务器 TZ 彻底解耦——服务器是 UTC 也不受影响。passport 日期、订单窗口结束日、缓存日期桶都用这同一个北京日期。

唯一还需实测的：crm.amer.cn 到底用哪个时区（它也用 `datetime.now()` 算期望值）。探针会在**北京 00:00–08:00 边界窗口**分别用"UTC 日期 / 北京日期"各打一次，看 CRM 认哪个，最终敲定 `CRM_PASSPORT_TZ`。

另外把**时间轴最多返回 5 条**（`TIMELINE_MAX_EVENTS=5`，可配）落进设计——倒序后取最近 5 条返回前端，喂大模型的也是这同 ≤5 条。更新计划：

两处都已落进计划：

- **时间轴最多 5 条**：`get_timeline` 倒序取最近 5 条（`TIMELINE_MAX_EVENTS=5` 可配），返回前端与喂大模型用同一份 ≤5 条。
- **passport 时区**：`compute_passport` 显式用 `ZoneInfo("Asia/Shanghai")` 算北京日期，与 UTC 服务器解耦；探针在北京 00:00–08:00 边界实测 CRM 认 UTC 还是北京日期，敲定 `CRM_PASSPORT_TZ`。

计划其余部分（两接口拆分、共享时间轴取数、确定性时间轴、JSON 三层校验兜底、提示注入防护、缓存、能力盘点、端到端验证）保持不变。

Plan approved. Let me set up progress tracking and read the remaining convention-defining files before I start writing code.

Now the test style, the proxy note in the smoke script, and the employee-context shape (which the frontend will echo back as `score_context`).

I now have all the conventions I need (retry loop in `pool.py`, singleton pattern in `industry_resolver.py`, exception-handler wiring in `main.py`, pure-function test style, the proxy-bypass trick, and the `employee_context` shape). Let me set up a task list and start building bottom-up.

Let me check the Python environment (version for `zoneinfo`, whether httpx is present) before writing config.

Environment is ready: Python 3.12, `zoneinfo` resolves `Asia/Shanghai`, httpx 0.28 and pytest 8.4 already present. Starting Task 1 (config, deps, error type).

Now extend the cache module to add a longer-TTL `detail_cache` (so LLM results aren't re-computed every 5 min) and centralize the new cache-key builders alongside the existing one.

Task 1 complete. Moving to the CRM client + passport (Task 2).

Task 3 done. Now the timeline service — the field mapping is written defensively with candidate key-name lists since the exact CRM field names await the probe.

Task 4 done. First a small tweak to the reason cache key (fold in a fingerprint of the passed score context so a moved-location/changed-score recomputes), then the recommendation service.

Task 5 done. Now schemas, router, and wiring (Task 6). I'll extract the digit-coercion validator to a shared helper so all three request models reuse it.

Now the router and wiring it into `main.py`.

Task 6 done. Let me verify the whole app imports cleanly (catches circular imports / syntax errors) before writing tests.

The app imports cleanly with **no circular imports** — and a strong signal: `compute_passport()` for today (2026-06-06) produces `cc5d55757f805503f9485d8bacc74a62`, which **exactly matches the passport in your CRM doc examples**. The passport algorithm and the timezone-pinned date are confirmed correct. Now the tests (Task 7).

Now let me run the full test suite (new + existing) to confirm everything passes.

All 53 tests pass (35 existing + 18 new). Task 7 done. Now the probe script (Task 8) — to be run on the server to confirm the CRM contract before trusting field-name assumptions.

Let me compile-check the probe and all new modules (the probe makes external calls only when run with a customer_id, so I'll byte-compile rather than hit the live CRM from here).

Before verifying, I spotted a robustness gap: `get_or_compute` would cache a **degraded** result (e.g., transient CRM outage → empty timeline) for the full hour. Let me make caching conditional — only cache healthy results — so a blip doesn't get stuck for a TTL.

Now sync the tests with the new get/set caching and add tests that prove degraded results aren't cached (and healthy ones are).

Let me re-run the suite to confirm the caching changes and new tests pass.

Task 8 done — probe compiles and its CLI works. Now Task 9: live end-to-end smoke via FastAPI TestClient. CRM/DeepSeek aren't reachable from here, so this exercises the **graceful-degradation paths** (which is exactly what I want to confirm works end-to-end). Short CRM timeout, no Redis, no DB warmup.

**Major finding:** the CRM is actually reachable from here and the calls **succeeded with real data** — the passport (`cc5d55757f805503f9485d8bacc74a62`) was accepted. The `/timeline` endpoint returned 5 real orders, correctly sorted desc and capped at 5, `degraded:false`. The template fallback fired (no DeepSeek key) and correctly folded in "近一年有5 笔订单记录".

But notice orders show `detail:"-"` and `status:"-"` — my guessed field names for product/status didn't match the real ones. Since CRM is reachable, I can run the probe **now** and fix the mappings precisely instead of guessing.

Now I have the **real orders contract**: envelope is `{"state":200,"message":"success","data":[...]}` (my extractor handled it — 18 records). Order fields: `order_id`/`id`, `order_date` (both matched ✓), but **product is nested in `details[].product_name`** (not top-level — that's why detail was "-"), and **there's no status field** at all. Visits/services were empty for 3599. Let me probe a customer likely to have visits/services to pin those field names too.

Now I have the **complete real contract for all three** — and visits/services use a totally different envelope than orders:

- **Orders**: `{"state":200,"data":[ {order_id, order_date, total_amount, details:[{product_name}]} ]}` — flat list, product nested in `details[]`, no status field.
- **Visits**: `{"state":200,"data":{"3599":{"customer_CallReport":[[ {time, summary, visit_way} ]]}}}` — keyed by customer_id, double-nested.
- **Services**: `{"state":200,"data":{"3599":{"customer_service_apply":[[ {apply_no, use_conditions, time, status_text} ]]}}}` — same nested shape.

That's why visits/services extracted 0. Let me fix the client extraction (per-type) and the timeline field mappings to the **real** field names, then re-verify.

Now update the timeline field mappers to the real field names (orders: `order_id`/`order_date`/`details[].product_name`/`total_amount`, no status; visits: `time`/`summary`/`visit_way`; services: `apply_no`/`time`/`use_conditions`/`status_text`).

Now sync the probe (it imported the old `_extract_records`) and update the one timeline test that used guessed field names to the real shapes.

Now re-run the test suite, the probe (confirm visits/services extract now), and the live `/timeline` (should now show products + visits + services with statuses).

All 55 tests pass and extraction now works: **orders 18, visits 3, services 1**. Let me run the definitive live smoke — the merged, sorted, capped-to-5 timeline with real products/visits/services + the recommend-reason.

The probe surfaced a real bug. Comparing the two runs: visits for 3599 with `employee_code="000068"` returned **0**, but with `employee_code=""` returned **3** — so the CRM **filters visits/services by employee_code**. Since the timeline must be the customer's *full* history (and its cache key is rep-independent), I must call CRM with **empty** employee_code regardless of the request. Currently `_compute_timeline` forwards the request's employee_code → a rep-scoped, wrong (and cache-poisoning) timeline. Fixing it:

Let me verify the fix the way that would expose it: in a **fresh process**, call `/recommend-reason` first with `employee_code=000068` (no prior `/timeline` to warm the cache) and confirm visits are now included.

The fix is verified — `/recommend-reason` (emp=000068, called first, fresh process) now shows "2 次拜访", confirming visits are no longer filtered out by employee_code. All 55 tests pass.

Last item: update the README. Let me read it to match its style.

Now I'll update the README to document the new endpoints, the CRM/DeepSeek config, the probe, and the verified contract.

All code is verified; the README edits don't touch code. Let me do one final clean test run to confirm no regressions, then wrap up.

Done. The behavior-timeline + AI-recommendation feature (接口四) is implemented, and — because the CRM turned out to be reachable from here — **verified end-to-end against the real CRM with live data**.

## 做了什么

**两个接口**（`app/routers/customer_detail.py`）：
- `POST /api/v1/leads/timeline` — 纯 CRM、毫秒级、无大模型，返回合并倒序、最多 5 条的行为时间轴。
- `POST /api/v1/leads/recommend-reason` — 复用时间轴 + 评分上下文喂 DeepSeek，返回一段 `recommend_reason`，可降级。

**分层**：`clients/crm.py`（passport + 三接口取数）、`clients/deepseek.py`（JSON 模式调用）、`services/timeline_service.py`（纯函数拼装 + 带缓存取数）、`services/recommendation_service.py`（评分规则 + 提示词 + Pydantic 校验 + 重试 + 模板兜底）。配置/错误/缓存键复用原有约定。**55 个单测全通过**。

## 真机验证发现并修正的 3 件事（探针 `scripts/probe_crm.py` 立的功）

1. **passport 完全正确** — `compute_passport()` 今天算出 `cc5d5575…74a62`，与你文档示例**逐字一致**，真实 CRM 接受。
2. **三接口结构各不相同**，我按实测改了抽取与字段映射：订单是扁平数组、产品在 `details[].product_name`、**无状态字段**；拜访/服务是 `{customer_id:{customer_CallReport/customer_service_apply:[[…]]}}` 的"按客户分组+双层数组"，字段是 `time/summary/visit_way` 与 `apply_no/time/use_conditions/status_text`。
3. **`employee_code` 会过滤拜访/服务**（带 000068 → 0 条拜访；留空 → 3 条）。要客户完整历史就**必须留空**，已修正 `_compute_timeline` 一律不传 employee_code（否则还会污染与业务员无关的时间轴缓存）。

实跑 `/timeline`（客户 3599）：18 订单+3 拜访+1 服务 → 合并倒序取 5 条，订单显示"￥1200｜安美防锈型加工冷却液SF30、…"，`degraded:false`。`/recommend-reason` 无 DeepSeek key 时正确回退模板："…综合评分 80 分，所属行业「电器制造」与您成交行业匹配…近一年有3 笔订单、2 次拜访记录，建议优先跟进。"

## 你需要补两件事
1. **`.env` 配 `DEEPSEEK_API_KEY`** 并确认 `DEEPSEEK_MODEL`（"deepseek v4" 的确切 id）— 不配则推荐理由一直走模板兜底（接口仍正常）。
2. **跨天边界复测 passport 时区**：今天 UTC 与北京同为 2026-06-06，没覆盖到差异时段；在北京 00:00–08:00 跑 `python scripts/probe_crm.py 3599` 与 `--utc` 各一次，确认 CRM 认北京日期，最终敲定 `CRM_PASSPORT_TZ`。

---

### Turn 2  *(2026-06-06 02:40)*

**User:**

我在.env.example上配好了DEEPSEEK_API_KEY 和 DEEPSEEK_MODEL

**Assistant:**

Good — let me verify it actually works against the live DeepSeek API. One thing first though: the app loads **`.env`**, not `.env.example` (`settings.py` reads `parents[2]/".env"`). So I'll load the DeepSeek values from `.env.example` for this test, then flag the placement.

Excellent — `deepseek-v4-flash` is valid, the key works, and JSON mode returns a clean `{"recommend_reason":"我在线，一切正常。"}`. Now the **full pipeline live** (real CRM timeline + real DeepSeek reason) for customer 3599:

Confirmed working **fully live, end to end**:

```
reason_source = llm   degraded = False
推荐理由： 该客户是电器制造行业，与您的成交行业完全匹配，实缴资本1000万规模较大，
          距您仅12公里，且近期有多次拜访和订单，值得优先跟进。
```

`deepseek-v4-flash` 是有效 model id，JSON 模式正常，理由质量很好——它正确用上了评分维度（行业匹配/实缴1000万/距离12km，来自 score_context + 评分规则）和真实行为时间轴（"近期有多次拜访和订单"）。校验 + 兜底链路都在。

## ⚠️ 但有一个会让它"线上不生效"的坑

**应用运行时读的是 `.env`，不是 `.env.example`**（`settings.py` 里 `_ENV_PATH = parents[2]/".env"`）。我刚才是特地把 `.env.example` 的值加载进来才测通的；你直接 `uvicorn` 起服务时，`.env.example` 不会被读取 → `DEEPSEEK_API_KEY` 为空 → 推荐理由会一直回退模板（接口仍 `code:0`，只是 `reason_source` 永远是 `template`）。

另外 **`.env.example` 是要提交进 git 的模板**（只有 `.env` 被 `.gitignore` 忽略），把真实 `sk-...` 写在那里有泄露风险。

**正确做法**：把这两行（连同需要覆盖的项）放进 `.env`，并把 `.env.example` 里的 key 还原成占位：
```
# .env （和 DB 密码放一起，不进 git）
DEEPSEEK_API_KEY=sk-fcd7...（你的真实 key）
DEEPSEEK_MODEL=deepseek-v4-flash
```
CRM_* 那几项用代码默认值即可，不配 `.env` 也能跑（刚才实测就是走默认）。

要我帮你把 DeepSeek 这几行写进 `.env`、并把 `.env.example` 里的 key 改回占位（`DEEPSEEK_API_KEY=`）吗？因为涉及密钥和你的凭据文件，我先等你确认再动。

---

### Turn 3  *(2026-06-06 02:44)*

**User:**

写进.env

**Assistant:**

On it. Let me read the current `.env` so I append cleanly.

`.env` has no DeepSeek keys yet. Let me append the DeepSeek block to `.env`, and re-read `.env.example` (you modified it) before sanitizing the key there.

Now sanitize the real key out of the committed template `.env.example` (keeping the model id, which is useful, non-secret documentation):

Now verify the **runtime** picks it up from `.env` (no test harness loading `.env.example` this time) — start the app normally and check the reason path is `llm`:

完成。现在**正常启动（只读 `.env`）就能用真实 DeepSeek**了：

- `settings.deepseek_api_key` 读到 ✓，`model: deepseek-v4-flash`
- `reason_source = llm`，理由："该客户行业完全匹配，距离仅12公里，近期有多次订单和拜访记录，显示活跃且关系良好，优先跟进有望扩大合作。"

两处改动：
- **`.env`** — 追加了 DeepSeek 配置块（key + model + 超时/温度/重试），和 DB 密码同处、不进 git。
- **`.env.example`** — 真实 key 已抹成占位 `DEEPSEEK_API_KEY=`（模型 id 保留作文档），避免提交模板时泄露。

接口四到此全链路打通并真机验证：CRM 三接口取数 → 确定性时间轴（倒序、≤5 条）→ DeepSeek 出推荐理由（JSON 校验 + 重试 + 模板兜底）→ 全程降级、必返 `code:0`。55 个单测通过。

> 那两条 `Redis unavailable … falls back to in-process` 是因为本机没起 Redis，自动回退进程内缓存，属正常；线上配了 `REDIS_URL` 就走 Redis。

剩一件按你节奏做即可：**北京 00:00–08:00 跨天时段**用 `python scripts/probe_crm.py 3599` 和 `--utc` 各跑一次，确认 CRM 认北京日期（敲定 `CRM_PASSPORT_TZ`）——今天 UTC 与北京同日，没覆盖到那个边界。

---
