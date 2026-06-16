# 同步「主要联系人」4 字段到合并项目 amer_sales_assist

## Context（背景）
已在 `smart_lead_generation` 给 `POST /api/v1/leads/score-and-info` 的每个客户对象补了 4 个来自 `crm_customer` 的字段（`contact` 主要联系人 / `contact_position` 职位 / `phone` 手机 / `tel` 固话，缺失显示 `"-"`）。现要把**同一改动**镜像到合并项目 `E:\amer_sales_assist`（= `/mnt/e/amer_sales_assist`，拓客 + churn 合并，分支 `main`），代码 + 说明文档都同步。用户已选：**文档全更，跟随该仓规范**（含 CHANGELOG 与接口文档头注）。

经两个 Explore agent 核实：改动与 sibling **1:1 对应**，仅代码位于 **`app/lead/`** 子包下；4 个字段当前在该仓**全不存在**（无半成品）。

## 会用到的能力

### 一、扫描范围（本次会话可见）
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

### 二、会用
- Agent Explore：已用于在合并项目定位代码 3 处 + 文档 3 处（调研阶段，已完成）。
- Skill verify（或 run）：实现后在合并项目起服务实测接口确认 4 字段进入 `data.list[]`；连库前按 [[db-tunnel-mtu-gotcha]] 修 MTU。

### 三、考虑过但不用
- MCP chrome-devtools / firecrawl / jina / tavily：纯后端 SQL+JSON+Markdown 改动，无前端调试或联网检索需求。
- Skill deep-research / claude-api：无外部资料或 LLM 调用需求。
- Skill code-review / simplify：改动极小且照搬现有模式，可选；非必需。

## 与 sibling 那次的关键差异（你猜测"位置一致"，实际几处偏移）
- 代码在 **`app/lead/`** 子包（不是 `app/`），其余锚点逐字一致：`crm_customer c` + 两个 LEFT JOIN、`_dash()` 助手已存在、路由**无 `response_model`**（同样的刻意选择，schema 改动仅同步文档、运行时不强制）。
- 本仓风格略不同：`response.py` 用 PEP 604（`int | str`）+ 短注释；`lead_service.py` 是 black 格式（dict 缩进 16 空格）。新增行**按本仓风格写**，别照抄 sibling 的对齐/`Union`。
- 接口文档在 **`docs/lead/接口文档-对接前端.md`**（单份，无 dated 变体、无外部 note 文件）。
- 根 `README.md` 不枚举字段→不动；但 **`docs/lead/README.md:17`** 与 sibling README 那行同构，**补同样的一行注**。
- 本仓多了 **`CHANGELOG.md`（`## Unreleased`）**，AGENTS.md 要求 API 契约变化留记录 → 加一条。

## 改动清单（代码 3 + 文档 3，共 6 文件）

### 代码

**1. `app/lead/repositories/geo_repo.py`** — `_GEO_SQL` 在 `st.star AS star,`（L37）后插 4 列：
```sql
    c.contact          AS contact,
    c.contact_position AS contact_position,
    c.phone            AS phone,
    c.tel              AS tel,
```

**2. `app/lead/services/lead_service.py`** — `_compute_blob` dict 在 `"regist_capi": ...,`（L114）后、`"_dist": dist,`（L115）前插（**16 空格缩进**，沿用 `_dash()`）：
```python
                "contact": _dash(c.get("contact")),
                "contact_position": _dash(c.get("contact_position")),
                "phone": _dash(c.get("phone")),
                "tel": _dash(c.get("tel")),
```

**3. `app/lead/schemas/response.py`** — `LeadItem` 在 `regist_capi: str  # or "-"`（L30）后插（PEP 604 + 短注释风格）：
```python
    contact: str  # 主要联系人, or "-"
    contact_position: str  # 主要联系人职位, or "-"
    phone: str  # 主要联系人手机, or "-"
    tel: str  # 主要联系人固话, or "-"
```

### 文档

**4. `docs/lead/接口文档-对接前端.md`** — 三处：
- 字段表在 `| `regist_capi` | ... |`（L153）后插 4 行：
  ```
  | `contact` | string | 主要联系人；缺失为 `"-"` |
  | `contact_position` | string | 主要联系人职位；缺失为 `"-"` |
  | `phone` | string | 主要联系人手机；缺失为 `"-"` |
  | `tel` | string | 主要联系人固话；缺失为 `"-"` |
  ```
- JSON 示例（L187 `... "regist_capi": "5000万人民币"`）**末尾补逗号**并在 `}` 前加一行（**合成占位值**，与 sibling 文档一致，避免把实测真实 PII 写进文档）：
  ```
  "contact": "张伟", "contact_position": "采购经理", "phone": "13800138000", "tel": "022-58000000"
  ```
- 头注 L3 更新为 2026-06-16：`> 版本：v1 ・ 更新日期：2026-06-16（本次变更：`list[]` 每个客户增加主要联系人字段 `contact` / `contact_position` / `phone` / `tel`，缺失为 `"-"`）`

**5. `docs/lead/README.md`** — L17 句末追加（镜像 sibling README）：
> `list[]` 每个客户除基本信息外含**主要联系人**字段 `contact`/`contact_position`/`phone`/`tel`（均 string，缺失为 `"-"`）。

**6. `CHANGELOG.md`** — `## Unreleased`（L3）下、`### Hardened (churn...)` 前插一条（匹配本仓 Unreleased 的"括号标题 + 散文 + 列表"风格）：
```markdown
### Added (lead score-and-info 客户对象补充主要联系人字段)

`POST /api/v1/leads/score-and-info` 返回的 `list[]` 每个客户增加 4 个来自 `crm_customer` 的主要联系人字段
（`contact` 主要联系人 / `contact_position` 职位 / `phone` 手机 / `tel` 固话），缺失统一 `"-"`；路由无 `response_model`，schema 改动仅同步 OpenAPI 文档。

- `app/lead/repositories/geo_repo.py`：`_GEO_SQL` SELECT 增加 4 列。
- `app/lead/services/lead_service.py`：`_compute_blob` 组装 dict 增加 4 个 key（沿用 `_dash()`）。
- `app/lead/schemas/response.py`：`LeadItem` 增补 4 个字段。
- `docs/lead/接口文档-对接前端.md`：字段表 + 返回示例同步。
```

## 不动的地方（边界）
- 根 `README.md`、`docs/handover.md`、`docs/lead/TEST_REPORT.md`：只提端点名/公式，不枚举字段 → 不动。
- 接口文档 L392 的 `score_context` 建议字段（接口四 recommend-reason 回传）：与评分/理由无关 → 不掺（与 sibling 一致）。
- churn 侧、`app/shared/`、migrations：与本改动无关 → 不动。

## 验证
1. 离线测试：在 `/mnt/e/amer_sales_assist` 跑 `python3 -m pytest`（或该仓 `make test`，见 Makefile），确认现有用例不破。
2. 连库实测（覆盖新增 SQL 列）：按 [[db-tunnel-mtu-gotcha]] 修 MTU → 按该仓 `docker-compose.yml`/README 起服务（端口以其为准）→ localhost+禁代理 curl `/api/v1/leads/score-and-info`（payload 同 sibling：`{"employee_code":"000068","lng":117.860197,"lat":39.966853,"radius_km":30,"page":1,"page_size":20}`），确认每条含 contact/contact_position/phone/tel、空值为 `"-"`。注意缓存 TTL（同 sibling，旧 blob 最多 5 分钟自愈）。
3. `git -C /mnt/e/amer_sales_assist diff --stat` 确认仅改上述 6 个文件。
