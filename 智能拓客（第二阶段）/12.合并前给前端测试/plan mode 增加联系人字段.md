# POST /api/v1/leads/score-and-info 增加客户联系人字段

## Context（背景）
`POST /api/v1/leads/score-and-info` 返回的每个客户对象（`data.list[]` 里的 `LeadItem`）目前缺少联系人信息。需补充 4 个来自业务库 `amer_crm_v2` 的字段，让前端能直接展示主要联系人及联系方式：
- `contact` — 主要联系人
- `contact_position` — 主要联系人职位
- `phone` — 主要联系人手机
- `tel` — 主要联系人固话

要求：模块化、最小动作、不影响其他代码。

> **澄清一处前提**：用户提到的 `app/clients/crm.py` 实际是远程 CRM 行为时间线（订单/拜访/服务）的 HTTP 客户端，**不连 amer_crm_v2、也不含客户档案**；其当前 git 改动（重试退避）与本需求无关。客户档案字段来自 `app/repositories/geo_repo.py` 中查询 `crm_customer` 的 `_GEO_SQL`。
>
> 数据流：`leads.py` router → `lead_service.query_leads` → `_compute_blob` → `geo_repo.fetch_candidates(_GEO_SQL)` → 组装 dict（lead_service.py:95-112）→ `envelope()`。该接口**无 `response_model`**，dict 里多出的 key 会原样返回。
>
> 因此改动落在 **geo_repo + lead_service + response schema 三处，不改 crm.py**。

## 会用到的能力

### 一、扫描范围（本次会话可见）
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

### 二、会用
- Agent Explore：已用于定位接口数据流与 SQL 来源（调研阶段，已完成）。
- Skill verify（或 run）：实现后拉起应用、实际请求接口确认 4 个字段进入 `data.list[]`；连业务库前需按 [[db-tunnel-mtu-gotcha]] 修 MTU。

### 三、考虑过但不用
- MCP chrome-devtools / firecrawl / jina / tavily：纯后端 SQL+JSON 字段改动，无前端调试或联网检索需求。
- Skill deep-research / claude-api：无外部资料或 LLM 调用需求。
- Skill code-review / simplify：可在改完后可选跑一遍，但改动极小且照搬现有字段模式，不列为必需。

## 改动（3 个文件，照搬现有字段模式）

### 1. `app/repositories/geo_repo.py` — `_GEO_SQL`（27-53 行）增加 4 列
在 SELECT 列表里、`distance_km` 计算之前加入 4 列。字段在主表 `crm_customer`（别名 c，已确认）：
```sql
    c.contact          AS contact,
    c.contact_position AS contact_position,
    c.phone            AS phone,
    c.tel              AS tel,
```
不动 FROM/JOIN/WHERE/HAVING/LIMIT，对其他列零影响。

### 2. `app/services/lead_service.py` — `_compute_blob` 组装 dict（95-112 行）增加 4 个 key
沿用现有 `_dash()`（None/空白 → `"-"`，与 `address`/`actual_capi`/`regist_capi` 完全一致）：
```python
            "contact": _dash(c.get("contact")),
            "contact_position": _dash(c.get("contact_position")),
            "phone": _dash(c.get("phone")),
            "tel": _dash(c.get("tel")),
```

### 3. `app/schemas/response.py` — `LeadItem`（16-31 行）增加 4 个字段（仅 OpenAPI 文档）
```python
    contact: str                               # 主要联系人, or "-"
    contact_position: str                      # 主要联系人职位, or "-"
    phone: str                                 # 主要联系人手机, or "-"
    tel: str                                   # 主要联系人固话, or "-"
```
该 schema 运行时不强制（router 无 `response_model`），加它只为文档/未来一致性，不改变运行行为。

## 缓存注意
组装结果会冻进缓存 blob，TTL 默认 300s（`app/cache/store.py`）。改动后旧 blob 会缺这 4 个 key，最多 5 分钟自然过期。为遵循"最小动作"，默认**等 TTL 过期**即可；如需立即生效可临时 bump `SCORE_VERSION` 环境变量（会使全部缓存键失效、影响面更大，非必要不做）。

## 验证
1. 离线单测：`python3 -m pytest tests/`（约 266 个，无需库/Redis，见 [[smart-lead-test-workflow]]）确认未破坏现有逻辑。注意：离线测试不连库，**不会**覆盖新增 SQL 列的真实取值。
2. 连库实测（覆盖 SQL）：先按 [[db-tunnel-mtu-gotcha]] 修 MTU，再 `docker compose up -d --build`（见 [[docker-dev-gotchas]]）起服务，localhost+禁代理 curl 一次该接口，确认 `data.list[]` 每个对象含 contact/contact_position/phone/tel，有值时正确、空值显示 `"-"`。
3. `git diff` 确认仅改上述 3 个文件。
