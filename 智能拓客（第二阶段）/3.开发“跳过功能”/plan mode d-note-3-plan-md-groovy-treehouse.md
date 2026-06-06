# 审查 + 落地方案：智能拓客"跳过功能"

## Context（为什么做这个）
前端推荐客户界面有"跳过"按钮，目标是把"看过/跳过"这类**会变的用户状态**叠加到推荐结果上，让业务员下次进来优先看未看过的客户。硬约束：业务库 `amer_crm_v2` **只读**，用户状态绝不能写进去；写入落到可写的日志库 `amer_sales_assist`(`.env` 里的 LOG_DB，root，主机 `8.138.104.14:23356`)。

本文件是对你手写 plan（`D:\note\智能拓客（第二阶段）\3.开发"跳过功能"\人工plan.md`）逐条对照真实代码后的**审查 + 修订落地方案**。

口径（你已拍板）：软分层**沉底** / **只认跳过按钮** / **粘住(永久沉底)** / 误点撤销 **MVP 先不做** / 可见性 **最终一致 + 前端乐观移除**。

> **状态更新**：`lead_skip` 表你已建好。已与你确认：销售标识列 = **`employee_code`**（与接口二/缓存键一致，repo 无需映射）、含 **`UNIQUE(employee_code, customer_id)`**（幂等 upsert 可用）。故下方步骤 3 不再建表，仅作确认；其余步骤照常。

## 审查结论
**核心思路对**：软分层 + 重算时冻结 `[未看过, 跳过]` + per-业务员 skip 集合 + 小 upsert 接口。插入点 `app/services/lead_service.py:_compute_blob`(:62) 是真实函数；列表项 `customer_id`(:85) 可直接匹配；表放 `amer_sales_assist` 正确。

**plan 里没覆盖、会卡住落地的缺口**（下方步骤已逐一补上）：
1. **LOG_DB 代码里还没接**——`settings.py` 只加载 `DB_*`，`app/db/pool.py` 只有业务库一个池，全仓零写操作。这是大头，不是"写个小接口"那么轻。
2. **skip 读取要可降级**——推荐是核心、跳过是增强；异地 LOG_DB(+ README 记录过该网段 MTU 黑洞史) 读失败不能拖垮整个推荐。
3. **"翻页稳定"非真保证**——Redis 是 `allkeys-lru`(docker-compose:16)，blob 可能 TTL 前被淘汰；多 worker 退回进程内缓存各存一份。需作为"接受的取舍"写明。
4. **customer_id 类型**——列表里是 `str`(:85)、表里是 `BIGINT`、接口传字符串，比对必须统一成 str，否则 skip 集合**悄悄匹配不上**。
5. **分层结果存成单一拼接列表**，避免改动现有分页数学。

**文档名字漂移**：讨论稿写 `churn_db`/`lead_gen`，实际是 `amer_sales_assist`——把文档统一过来，并确认它是**拓客专用状态库**（别和流失预警产品库混用；注意它与业务库**不同主机**，跨库 SQL JOIN 不可行，将来联动需应用层做）。

## 落地步骤（含文件与复用点）

**1. 配置：加 LOG_DB**（`app/config/settings.py`，仿 24–33 行 `db_*` 块）
新增 `log_db_host/port/user/pwd/name` + 独立 `log_db_connect_timeout`（建议更短，异地主机）。`.env`/`.env.example` 已有 LOG_DB_* 变量，去掉"本期不连"注释。

**2. 第二连接池**（新建 `app/db/log_pool.py`，仿 `app/db/pool.py`）
独立 PyMySQL 连到 `8.138.104.14:23356`，`autocommit=True`，复用 `pool.py` 的 `connect_with_retry`/`_RETRYABLE`/`get_conn()` 写法。**不要**复用业务库池（不同主机/库/账号）。

**3. `lead_skip` 表 — 已建好，无需再建表（schema 已确认）**
你已在 `amer_sales_assist` 建表，确认结构如下（与讨论稿一致；`employee_code` + `UNIQUE(employee_code, customer_id)` 已核对）：
```sql
CREATE TABLE lead_skip (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  employee_code VARCHAR(16) NOT NULL,
  customer_id   BIGINT UNSIGNED NOT NULL,
  action        VARCHAR(16) NOT NULL DEFAULT 'skip',
  created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_emp_cust (employee_code, customer_id),
  KEY idx_emp (employee_code)
);
```
`action` 留作未来扩展（收藏/已联系/撤销）。repo 只读写 `employee_code`/`customer_id` 两列，其余列与默认值由建表保证。

**4. 仓库层**（新建 `app/repositories/lead_skip_repo.py`）
- `fetch_skip_set(employee_code) -> set[str]`：`SELECT customer_id FROM lead_skip WHERE employee_code=%s`，结果 **cast 成 str** 返回（解决类型坑#4）。
- `upsert_skip(employee_code, customer_id)`：`INSERT ... ON DUPLICATE KEY UPDATE updated_at=NOW()`（幂等）。customer_id 入库 cast 成 int。

**5. 分层**（`app/services/lead_service.py:_compute_blob`，在 :104 排序之后）
```python
skip = set()
try:
    skip = lead_skip_repo.fetch_skip_set(employee_code)   # 降级#2：失败兜底空集合
except Exception:
    logger.warning("skip set fetch failed; treating as empty")
unseen = [it for it in items if it["customer_id"] not in skip]
sunk   = [it for it in items if it["customer_id"] in skip]
items  = unseen + sunk            # 单一拼接列表#5，两层已各自有序
# blob 里多存 unseen 计数
```
列表推导**保序**，无需二次 sort。**skip 集合直接在 `_compute_blob` 读，不要再单独缓存**（否则多一层 staleness）；它天然受 lead 缓存 TTL(≤5min) 约束。

**6. 分页字段**（`query_leads` :137 + `app/schemas/response.py:Pagination` :40）
`total` 仍 = `len(items)`（拼接后总数，分页数学完全不变）。新增 `pagination["unseen_total"]`、`pagination["skipped_total"] = total - unseen_total`；Pagination 模型加 `unseen_total: Optional[int]=None`、`skipped_total: Optional[int]=None`。
（建议用 `skipped_total` 而非你写的 `seen_total`——只认跳过时 seen≡skipped，叫 skipped 更准；前端"还有 N 个新客户"取 `unseen_total`。）

**7. 写接口**（`app/routers/leads.py`，仿 :18 `score-and-info`）
```
POST /api/v1/leads/skip   { "employee_code": "000068", "customer_id": "198" }
```
- 新 `LeadSkipRequest`(`app/schemas/request.py`)：`employee_code: min_length=1`、`customer_id: str`（非空 + 可转 int）。
- 调 `lead_skip_repo.upsert_skip`，成功 `return envelope(data=None)`。
- LOG_DB 写失败 → 新 DomainError(`app/errors.py`，仿 `DBUnavailable`，如 code 1505 "状态库暂不可用")，沿用全局异常 envelope。
- 校验范围（MVP）：只做 Pydantic 基础校验，**不**跨库查 customer 是否存在（异地、加延迟，不值）。

**8. 可见性：最终一致 + 前端乐观移除（已确认）**
后端只写库，列表在缓存过期/刷新后(≤5min)重排沉底。**不做** per-业务员主动失效：缓存键含定位(`store.py:build_cache_key`)，一个业务员多把键、无法廉价失效；而"skip 版本号入键"会让每次跳过触发一次 **7–10s 重算**，连续跳过很卡，不划算。前端配合：点击即**本地隐藏该卡片**（乐观更新）+ 提供"刷新"强制重算。

## 接受的取舍 / 风险
- **冻结顺序非强保证**：allkeys-lru 可能 TTL 前淘汰 blob；多 worker 进程内缓存各存一份。淘汰/换 worker 后重算会重排——同一会话内可能轻微抖动。MVP 接受。
- **跨主机非跨库**：`amer_crm_v2` 与 `amer_sales_assist` 不同主机，无法 SQL JOIN；将来"流失↔拓客"联动需应用层做。
- **误点恢复**：粘住 + MVP 不做 un-skip；靠软分层兜底（翻完未看过后跳过的仍会出现）——这正是选软分层而非硬过滤的理由。表已留 `action`+唯一键，后续加 un-skip 成本低。
- **（前瞻）合并 churn 引擎共用 Redis**：见 [[merge-with-churn-engine]]——noeviction vs allkeys-lru 冲突会改变本 blob 的淘汰行为，合并时单独评估。

## 验证
> 连真实 LOG_DB（`8.138.104.14`）的读写会触发 prod-DB 守卫——集成/HTTP/手测需你授权或你用 `!` 自己跑；纯函数单测离线即可，我来跑。
- **单元**：把分层抽成纯函数（items + skip_set → unseen+sunk + unseen_total），加 `tests/` 用例（含类型边界：int/str customer_id 都能匹配），仿现有 30 用例风格。
- **集成**(`scripts/integration_check.py`)：upsert 一个 customer → 强制重算 → 断言其落到 unseen 之后、`unseen_total/skipped_total` 正确。**用例不依赖 LOG_DB**：repo 读取可注入/mock，保证核心读路径离线可测。
- **HTTP**(`scripts/http_smoke.py`)：POST /skip 校验三段式 envelope + 幂等（连发两次不报错）；并验证 LOG_DB 不可用时 score-and-info 仍正常（降级#2）。
- **手测**：`curl` POST /api/v1/leads/skip → 再打 score-and-info 看分页字段变化。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent Explore：已用于摸清推荐链路/缓存/DB层/路由（本审查的依据）。
- Skill verify 或 run：实现后起 uvicorn 打真实 HTTP，端到端验证 /skip 与分层重排。
- Skill code-review：实现后对 diff 做正确性/简化审查。

**考虑过但不用**：
- Agent Plan：实现模式已被本仓库既有写法（pool.py/repositories/routers/schemas）确定，设计分析也已由 3 个 Explore + 直接读关键文件完成，无需再开 Plan agent。
- Skill claude-api：跳过功能不涉及 LLM/Claude（那是接口① AI 推荐理由的范围，本期不做）。
- MCP jina/tavily/firecrawl（网络检索）：本任务是本地代码审查，无需外网。
