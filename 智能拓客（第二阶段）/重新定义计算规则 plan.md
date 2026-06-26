# 重定义 lead 客户得分计算规则

## Context

`/api/v1/leads/score-and-info`（接口二）给每个候选客户打分（满分 100），用于排序拓客线索。 业务方调整了四个维度的计分规则，需要按新规则重算。改动面要求尽可能小。

**现行规则 → 新规则**（满分仍 100）：

|维度|现行|新规则|
|---|---|---|
|行业匹配 industry_match|0 / **40**|0 / **30**（语义不变：客户行业 ∈ 业务员历史成交行业 → 30）|
|企业规模 scale（实缴资本）|0 / **10**（≥500万二元）|**0–20 分档**（见下，左闭右开）|
|客户标签 tags|0 / 10|**0 / 10（不变）**|
|离定位距离 distance|0–40（≤20km=40，20–60km 线性，>60km=0）|**0–40：每公里 −1**（[0,1)km=40、[1,2)km=39…，≥40km=0）|

**企业规模分档**（实缴资本，左闭右开，每 500 万一档、逐级 +1，≥1 亿封顶 20）： `[0,500万)=0、[500万,1000万)=1、[1000万,1500万)=2 …… [9500万,1亿)=19、[1亿,∞)=20` 统一公式：`min(floor(实缴元 / 5_000_000), 20)`。（用户 2026-06-26 校正：各档比初稿低 1，满分 20 仅在 ≥1 亿出现。）

**两项已确认的决策**：

- 实缴**缺失 / 无法解析 / 外币** → **0 分**（无从比较，沿用现有语义；与最低档 `[0,500万)=0` 同分，唯 ≥500万 可解析者才 ≥1）。
    
- **不** bump `SCORE_VERSION`：blob 缓存 TTL 300s / 详情 3600s 自然自愈，零 `.env` 改动；窗口期内可能短暂新旧分混排（已接受）。
    

## 会用到的能力

**扫描范围**（本次会话可见）：

- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
    
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily
    

**会用**：

- Agent Explore：已在规划阶段定位 scoring 引擎 / 数据模型 / 测试与文档同步点。
    
- Skill code-review：实现后对 diff 做一次正确性复查（评分是排序核心，值得过一遍）。
    
- 门禁命令（Bash）：`pytest -q` / `ruff` / `mypy`（验证见末节）。
    

**考虑过但不用**：

- deep-research 及 web MCP（jina / tavily / firecrawl）、chrome-devtools：本地纯代码改动，无需联网检索或浏览器。
    
- Skill simplify：被 code-review 覆盖（后者兼顾正确性 + 简化，更贴合排序逻辑改动）。
    
- Skill verify / run：评分是纯函数 + 已有离线 HTTP 不变量测试覆盖，`pytest` 即可验证，无需起全栈应用观测。
    
- Agent codex:rescue：改动明确、范围小，不需要第二实现/救援。
    

## 改动清单

### 1. 核心引擎 `app/lead/services/scoring.py`（单一真源）

- **模块 docstring**（line 3）：权重改为 `industry_match 30 | scale 20 | tags 10 | distance 40`；line 9 距离说明改为"每公里 −1（default）；normalized 变体保持不动、未启用"。
    
- **常量**（line 43）：保留 `SCALE_THRESHOLD_RMB = 5_000_000`（现兼作规模分"每档步长 = 500 万"，更新注释），新增 `SCALE_MAX_SCORE = 20`。保留名字是为不动 `recommendation_service` 的 import。
    
- **`score_industry`**（line 95 + docstring line 85）：`40` → `30`，其余逻辑（空行业判定、id 匹配）不变。
    
- **`score_scale`** 改为分档：
    
    def score_scale(actual_capi_raw: object) -> int:  
        """实缴资本分档 0–20（左闭右开，每 500 万一档）：[0,500万)=0、[500万,1000万)=1…逐档 +1，  
        [9500万,1亿)=19、≥1亿封顶 20。外币/缺失/无法解析 → 0。"""  
        val = parse_capital_to_rmb(actual_capi_raw)  
        if val is None or val < 0:  
            return 0  
        return min(int(val // SCALE_THRESHOLD_RMB), SCALE_MAX_SCORE)
    
    复用现有 `parse_capital_to_rmb`（已处理万/亿/外币/全角/逗号）—— 不改解析。
    
- **`distance_score_default`**（line 107-113）改为每公里递减：
    
    def distance_score_default(d_km: float, radius_km: float) -> float:  
        """每公里 −1：[0,1)km=40、[1,2)km=39…逐公里 −1，≥40km 封底 0（左闭右开）。  
        radius_km 不参与（保持与 normalized 一致的注册表签名）。"""  
        return float(max(40 - math.floor(d_km), 0))
    
    `radius_km` 现状本就未被 default 使用，无 lint 影响；`score_candidate` 里 `round_half_up` 包裹整数结果不变。
    
- **`distance_score_normalized` / `score_candidate` / `parse_capital_to_rmb` / `is_empty_industry` / `round_half_up`**：**不动**（normalized 未启用、保留为遗留变体；score_candidate 仅调用更新后的子打分器，求和与 detail 键名不变）。
    
    - ⚠️ 部署前确认生产 `DISTANCE_SCORER=default`（未设 `normalized`）—— normalized 仍是旧 20/60km 曲线，若被启用则距离分不走新规则。默认即 `default`（settings.py:59 / .env.example），常态安全。
        

### 2. 同步点 `app/lead/services/recommendation_service.py`（接口四，喂给 LLM 的规则镜像）

- **`SCORING_RULES_TEXT`**（line 38-43）：重写为新规则文案（行业 30 / 规模 0–20 分档 / 距离每公里 −1）；继续复用 `_SCALE_WAN`（=500）描述"每 500 万一档"，故 `SCALE_THRESHOLD_RMB` 的 import 保留。
    
- **`template_reason` 规模高亮**（line 167）：`"规模达标（实缴≥500万）"` 现在 scale≥1 即触发会失真，改为反映分档得分（如 `f"实缴规模评分 {detail['scale']}/20"`）。其余兜底逻辑不动。
    

### 3. 测试（仅改期望值/夹具，**不增删测试函数 → 基线 559 不变**）

- `tests/lead/test_scoring.py`：
    
    - `test_score_scale_boundary`（单函数，断言数可加但不计数）：重写覆盖分档边界 —— `"499万"→0`、`"500万"→1`、`"999万"→1`、`"1000万"→2`、`"9500万"→19`、`"9999万"→19`、`"1亿"→20`、`"5亿"→20`、`"0"→0`、`"100万"→0`、`None→0`、`"100万美元"→0`。
        
    - `test_score_industry`：`40` → `30`。
        
    - `test_distance_default`（保持 8 个参数）：新曲线 `[(0,40),(0.5,40),(1,39),(2,38),(39,1),(40,0),(41,0),(80,0)]`。
        
    - `test_score_candidate_sums_to_total`：入参改 `actual_capi_raw="1亿"`、`distance_km=0` → detail `{30,20,10,40}`、total 100。
        
    - `test_score_candidate_low`：重算 `industry=999/"100万"/无标签/45km` → detail `{0,0,0,0}`、total 0（"100万" 落 [0,500万)=0、45km>40 距离 0）。
        
    - `test_parse_capital` / `test_score_tags` / `test_is_empty_industry` / `test_distance_normalized` / `test_round_half_up_not_bankers`：**不变**。
        
- `tests/lead/test_score_invariants_http.py`：重设 `CANDS` 让排序与原叙事一致（A=100,B=100,C=60,D=39 → 序 `["B","A","C","D"]`、skip A → `["B","C","D","A"]` **均不变**）：
    
    - `A:("A",43,"1亿",0.5)`=100、`B:("B",43,"1亿",0)`=100(更近)、`C:("C",43,None,10)`=60、`D:("D",99,"100万",1)`=39（行业未匹配+无标签+"100万"落 [0,500万)=0 → 仅距离 1km=39）。
        
    - 更新维度断言：`B.score_detail == {30,20,10,40}`、`C.score_detail["distance"] == 30`、`D` 的 `industry_match==0 & total==39`（原为 40，因 "100万" 现计 0 分）。排序/skip/求和断言不动。
        
- `tests/lead/test_recommendation_service.py`：`SCORE_CTX.score_detail` 改为新量程实值（如 `{industry_match:30, scale:10, tags:0, distance:28}` + 对应 total）。无断言依赖具体分值 → 低风险，仅保夹具不撒谎。**须保持 `industry_match > 0`**：`test_template_reason_content` 靠行业高亮（recommendation_service.py:164）断言出现"数控机床"，归零会断。
    
- **内联注释随数字一起改**（reviewer 提示，别留误导旧算式）：`test_score_invariants_http.py` 的 `CANDS` 注释（33-39，如 "40+10+10+40 = 100"、"线性=20"）与 `test_scoring.py:116` 的 "40*(60-45)/40 = 15" 都编码旧公式，须同步重写。
    

### 4. 文档同步（人读镜像，与代码同改）

- `docs/lead/接口文档-对接前端.md`（三处，**reviewer 补了第三处**）：
    
    - `score_detail` 规则表（line 171-174）：行业 `0/30`、规模 `0–20` 分档、距离每公里 −1。
        
    - 接口二返回示例（line 194-195）+ **接口四请求示例 score_context（line 413-414）**：两处都把 `industry_match:40` 这种现已不可能的值，按新规则改成同一组自洽值（如客户 3599「5000万人民币」→ scale = floor(5000万/500万) = 10 → `{industry_match:30, scale:10, tags:10, distance:32}`、total 82；顺带修原例 tag_name 非空却 `tags:0` 的旧不一致）。
        
    - 可选：line 437 的 `reason_source:"llm"` 叙述串含"规模达标"措辞——属大模型自由文案示例（非模板输出），可留可轻改，不影响正确性。
        
- `docs/lead/README.md:23`：一行规则重写。
    
- `docs/handover.md:169`：打分公式重述。
    
- `docs/lead/capacity.md`（line 11-12, 34）：">60km 距离分恒 0" → ">40km 距离分恒 0（每公里 −1）"。
    

### 不改动（明确排除）

- `app/lead/config/settings.py` / `.env.example`：**不** bump `SCORE_VERSION`（已确认靠 TTL 自愈）。
    
- `app/lead/schemas/response.py`：`ScoreDetail` 四个 `int` 字段名/类型不变（只是量程变了）。
    
- 无 DB 迁移；`CHANGELOG.md` 不写 Unreleased（沿用 cut-release 时从 git log 回填的约定）；`CLAUDE.md` 测试基线 559 不变（未增删测试）。
    

## 提交粒度（铁律#2）

单一 lead 模块、一个完整主题。建议两笔、各自可独立回滚：

1. 运行时 + 测试：`scoring.py` + `recommendation_service.py`（同步镜像）+ 三个测试文件。
    
2. 纯文档：`docs/**`（纯文档按铁律#2 不必单独，可并入①，但分开更干净）。
    

## 验证

1. **靶向测试**：`python3 -m pytest -q tests/lead/test_scoring.py tests/lead/test_score_invariants_http.py tests/lead/test_recommendation_service.py`
    
2. **全量**：`python3 -m pytest -q`（应仍 **559** 全绿；离线 HTTP 不变量测试 `test_score_invariants_http.py` 已端到端覆盖 `/score-and-info` 的求和恒等、降序、分档与新距离曲线）。
    
3. **门禁**：`python3 -m ruff format app tests scripts && python3 -m ruff check --fix app tests scripts`；`python3 -m mypy`（须全绿）。
    
4. **可选人工冒烟**：`docker compose up -d --build` → `curl --noproxy '*' -X POST http://127.0.0.1:8000/api/v1/leads/score-and-info -d '{...}'`，眼检 `score_detail` 的 `industry_match≤30 / scale≤20 / distance` 每公里 −1 落点。离线测试已足够，此步非必需。
    

## 审查记录

Codex 审查当时不可用（用量限制，到 Jul 18 19:01 重置）。改由一个独立 general-purpose agent 做对抗式复核：从零重算了全部 10 个规模档位、10 个距离点、4 候选的排序/skip 顺序及两个 `score_candidate` 用例，**均与规格吻合，无正确性 bug**（1亿处无 off-by-one、封顶 20 正确、左闭右开与"0 vs 缺失"语义都对、测试数 559 不变、无 ruff/mypy 隐患）。本节上方 §3-§4 的三处补强（接口文档 line 413-414 接口四示例、测试内联注释、`SCORE_CTX.industry_match>0`）即采纳自该复核；另提示部署前确认生产 `DISTANCE_SCORER=default`（已并入 §1）。

**规模档位校正（用户 2026-06-26，复核之后）**：复核验证的是初稿公式 `min(floor(实缴/500万)+1, 20)`（含 +1，[0,500万)=1）。用户随后把各档下调 1 → `[0,500万)=0 … [9500万,1亿)=19 … ≥1亿=20`，即去掉 +1 得 `min(floor(实缴/500万), 20)`。已据此重算受影响处：`test_score_scale_boundary` 全部断言、`test_score_candidate_low`（{0,0,0,0}=0）、`test_score_invariants_http` 的 D（"100万"→0 → total 39）、文档示例。距离维度、`1亿→20`、排序/skip 顺序、测试数 559 均不受影响。新公式边界已自查：`[9500万,1亿)` floor=19→19、`1亿` floor=20→封顶 20，无 off-by-one、无缺口。