# score-and-info 行业筛选 + /industries 端点（含 DeepSeek 兜底确认）

## Context（为什么做）

用户两点：
1. **score-and-info 新增"按行业筛选"**。已敲定契约：**`industry_ids` 与 `industry_names` 都收，id 优先、name 兜底**；并新增轻量行业列表端点供前端做"按名筛选"的下拉。
2. **AI 推荐理由 DeepSeek 失败兜底**——**已实现，核心不改**（详见下方"要点2"，措辞已按 Codex 审查收紧）。

**筛选设计核心**：读时、内存内对**已缓存的打分 blob**收窄——复用 `(employee_code,lng,lat,radius)` 缓存键（`build_cache_key` 不含 filter，`store.py:23`），**一份 blob 服务所有筛选、不进缓存键、不触发 7–10s 重算**。`name` 经 IndustryResolver 反查成 id，最终**统一按候选 `amer_industry_id`(int) 做整数集合匹配**（候选数据本就带该 id，`geo_repo.py:31`；评分也按它匹配——id 是规范键）。

### 经 Codex 审查的修正（评估结论）
- **采纳并写死语义**：① `[]`/omit vs 非空列表（见 §2/§3，解决原"空列表"自相矛盾）；② **筛选只建新局部列表、绝不 mutate `blob["items"]`/`blob["unseen_total"]`**（进程内缓存 `get()` 返回引用、无深拷贝，`store.py:80`——in-place 改会串包）；③ `amer_industry_id=="-"` 候选在筛选时被排除、未知 id 被接受但匹配不到任何候选（均为**有意设计**，不报错）；④ name 归一化口径 + 前端应取自 /industries 的名以保证命中。
- **确认无误**：name→id 是真实缺口（resolver 现仅 `id_to_name`）、软分层重算正确、分页筛选后自洽、/industries 放 leads 前缀合理——均已在计划内。
- **要点2 措辞收紧**：DeepSeek 失败→模板**准确**；但 `timeline_service.get_timeline()` 在内层 try 之外（`recommendation_service.py:186`），其意外异常不被模板覆盖、会落全局兜底返回 **1500 envelope**（非裸 500、非模板）。见 §5 可选硬化。

## 改动

### 1. IndustryResolver 加 name→id 反查 + 列表导出 — `app/services/industry_resolver.py`
- 现仅 `_id_to_name`（值在 `_rows_to_mapping` 已 `.strip()`，`:80`）。新增 `_name_to_ids: dict[str, set[int]]`，**key = `name.strip().casefold()`**（一名多 id 用 set 兜底），在 `_rows_to_mapping` 内与 `_id_to_name` 一并构建（不额外 I/O，复用 `load()`/快照兜底）。
- 新增 `name_to_ids(name) -> set[int]`（同样 `strip().casefold()` 后查、缺失返 `set()`）、`all_industries() -> list[dict]`（`[{"id":int,"name":str}, ...]` 按 name 排序）。
- 归一化口径：只做 `strip().casefold()`（CJK 名 casefold 基本无副作用；不做全角/半角转换，避免过度工程）。**契约**：前端按名筛选时应使用 /industries 返回的名（逐字），以保证命中。

### 2. 请求 schema — `app/schemas/request.py`（`LeadQueryRequest`）
- 新增两可选字段：`industry_ids: Optional[list[int]] = None`、`industry_names: Optional[list[str]] = None`。
- 校验器：`industry_names` 逐项 `strip()`、丢空白项；两者各设长度上限（如 ≤100）防滥用。**`[]` 合法**（语义在服务层=不筛选，见 §3）。

### 3. 服务层筛选 — `app/services/lead_service.py`（`query_leads`，插入点在 `items = blob["items"]` 之后、`total = len(items)`(`:151`) **之前**）
- **解析 filter（id 优先，三态语义明确）**：
  - `industry_ids` 为**非空**列表 → `idset = {int(x) for x in industry_ids}`，`filter_active=True`。
  - 否则 `industry_names`（校验器 strip 后）为**非空**列表 → `idset = set().union(*(industry_resolver.name_to_ids(n) for n in industry_names))`，`filter_active=True`。
  - 否则（两者皆 omit 或 `[]`）→ `filter_active=False`（**不筛选、返回全量**，与现状一致）。
  - `filter_active=True` 但 `idset` 为空（如 names 全不可解析）→ 结果为**空集**（用户筛了匹配不到的行业，正确）。
- **按软分层 segment 收窄（建新局部列表，绝不 mutate blob）**，仅当 `filter_active`：
  ```python
  unseen  = [it for it in items[:unseen_total] if it["amer_industry_id"] in idset]
  skipped = [it for it in items[unseen_total:] if it["amer_industry_id"] in idset]
  items, unseen_total = unseen + skipped, len(unseen)
  ```
  （`amer_industry_id` 是 int 或 `"-"`，`lead_service.py:100`；`"-"` 永不命中整数集合→被排除；未知 id 被接受但匹配不到候选→空，均不报错。）
- 其余 `total`/`total_pages`/`has_more`/`unseen_total`/`skipped_total`/切片照旧（`:151-167`），自然反映筛选后集合。缓存键、employee_context、打分均不变。
- **性能**：O(n) 过候选列表，n ≤ `max_candidates`(20k，`geo_repo.py:68`)，<1ms，可忽略。

### 4. 行业列表端点 — `app/routers/leads.py`
- `GET /api/v1/leads/industries` → `envelope({"industries": industry_resolver.all_industries(), "source": industry_resolver.source})`。同步 def、纯读内存（启动已预热 `main.py:34`）、毫秒级。
- **降级行为（明确）**：DB+快照都失败时 `_id_to_name` 为空 → 返回**空列表**（不 503），`source` 字段（db/snapshot/none）供前端/排障识别。与全仓"resolver 优雅降级"一致。

### 5. 要点2：DeepSeek 兜底确认（核心不改）+ 可选硬化
- **确认（不改代码）**：`generate_reason` 重试耗尽后捕获 `(LLMError, ValidationError)` → `template_reason`（`recommendation_service.py:176`）；`get_recommend_reason` 对空 timeline 直接模板、并在内层 `except Exception` 兜底（`:189,196-198`）。**DeepSeek 任何失败 → `reason_source:"template"`**，已有 e2e 单测 + 压测实测。
- **（可选，Codex 指出）微硬化**：`get_timeline()` 调用在内层 try 之外（`:186`），其意外异常会落全局 1500 envelope 而非模板。如要 recommend **任何**情况都返模板，可把 `tl = get_timeline(...)` 也纳入兜底（失败则按"空时间轴 + degraded"走模板）。**默认不做**（get_timeline 内部各源已 `_safe_fetch`、实际不抛；且全局兜底已是 envelope 非裸 500）；按用户意愿决定是否加。

### 6. 测试 — tests/
- `test_industry_resolver_fallback.py`（扩充）：`name_to_ids` 精确/归一化(大小写/空白)/缺失→`set()`/一名多 id；`all_industries` 形状与排序。
- 新增 `test_industry_filter.py`（仿 `test_pagination_radius.py`，monkeypatch `lead_service.cache.get_or_compute` 返回带不同 `amer_industry_id`(含 `"-"`) 的构造 blob）：按 id 筛、按 name 筛（注入 resolver 反查）、**id 优先**、unseen/skipped 重算、`[]`/omit→全量、非空 names 解析为空→空集、`"-"` 被排除、未知 id 不报错→空。**并断言不 mutate 原 blob**（筛后再用同 blob 取全量仍是全量）。
- `test_api_boundary.py` 补：`industry_ids`/`industry_names` 合法→200、类型错(如 `industry_ids:["x"]`)→1002（服务桩）；`GET /industries` 返回列表 + source。

### 7. 文档 — `D:\note\…\接口文档，对接前端.md` + 仓库副本 `docs/接口文档-对接前端.md`
- score-and-info 请求体表加 `industry_ids`/`industry_names`（可选；**id 优先**；注释：name 反查成 id+归一化、应取自 /industries 的名、无行业 id 的候选会被排除、`[]`/不传=不筛选）；测试用例补两条筛选用例。
- 新增 `GET /api/v1/leads/industries` 小节（`data.industries=[{id,name}]` + `source`，供下拉显示名/传 id）。
- 接口四"对接注意"按 §5 收紧措辞：DeepSeek 失败自动回退模板（已实现）。

## 验证
1. `python3 -m pytest tests/ -q` 全绿（新增 + 回归）。
2. 起服务实测（http_smoke 风格，需 MTU=1400 + 业务库可达 + Redis）：`GET /industries` 返回 ~70 行业；score-and-info 带 `industry_ids=[43]` 与等价 `industry_names=["数控机床"]` 返回**同一筛选结果**、分页计数正确、命中同一缓存 blob（第二次 ~ms）；`industry_ids=[]`/不传 与现状全量一致（回归）；非空但不可解析的 name → 空集。

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent `codex:codex-rescue`：已用于审查本计划（结论已并入上方"经 Codex 审查的修正"）。
- 内置 Bash：跑 `pytest` 与起服务实测（/industries + 筛选）。
- （可选）Skill `code-review`：新代码完成后自审。

**考虑过但不用**：
- `claude-api`：要点2 涉及 DeepSeek（非 Anthropic）且核心不改；按该 skill 自身规则跳过。
- Agent `Explore`/`Plan`：改动文件已知、契约已敲定、本会话已读全相关代码，直接设计，无需再 fan-out。
- `run`/`verify`：复用现成 `scripts/http_smoke.py`/`load_test.py` 起服务即可。
- `deep-research` / MCP `tavily`·`jina`·`firecrawl`·`chrome-devtools`：纯本地后端改动，无需联网/浏览器。
