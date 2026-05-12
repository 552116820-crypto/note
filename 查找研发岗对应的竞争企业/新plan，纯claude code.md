# 研发岗品类竞企图片解析 —— 纯 Claude Code 执行剧本

## Context（为什么写这份 + 为什么从旧 plan 大改）

旧 plan（`/home/huangch001002/.claude/plans/e-note-md-md-effervescent-kettle.md`）选了 **DashScope Qwen3-VL-Plus + Qwen-Plus 联网版** 当后端，理由是单次成本 ~¥15（vs Claude API ~¥150）。

但用户本次明确：**不要写完整 pipeline，就在这个 Claude Code 会话里直接干活**。这条决策推翻了旧 plan 的整套架构：

- ❌ 不要 `dashscope` SDK / API key / `dashscope_client.py`
    
- ❌ 不要 vision_parser.py / web_verify.py（这两件事**我自己**用 Read+Agent / WebFetch+WebSearch 就能做）
    
- ✅ 保留：旧 plan 所有**业务决策**（7 岗位品类映射、置信度策略、公司名归一化规则、评分公式、xlsx 双 sheet 结构、不动 demo.xlsx）
    
- ✅ 保留：旧 plan 的"分批跑 + 人工 review"工作流（浮选 35 → 审 → 电子展 52 → 审 → 合并）
    

复用频率：用户说**每年 1-2 次新展会**会重跑。这强度不值得写 pipeline，但**值得留一份 `CLAUDE.md`（repo 根）**——Claude Code 打开这个 repo 时会自动加载它，让一年后的我（或下一个 Claude）能立刻接上项目背景、按本剧本复刻流程。**不另写 RUNBOOK.md**（人/AI 同读一份就够）。

## Pre-flight 实测结论（已核实，沿用之前的表）

|项|状态|
|---|---|
|浮选 35 / 电子展 52 张 JPG|✓|
|demo.xlsx|只有 Sheet1，无锁文件，A2:A8 是 7 岗位|
|Hyperlink 写法|**走 B2 公式法** `=HYPERLINK(url, 全称)`，cell 只显示公司全称（用户拍板）|
|Excel 起点|**`Workbook()` 全新创建**，显式设列宽（用户拍板）|
|图片文件名|纯 IMG+时间戳，无元数据|
|Python 已装|openpyxl 3.1.5 / pillow 12.2 / pydantic 2.12 / pytest 8.4 / PyYAML 6 / rich 15|
|还要装|**只剩 `rapidfuzz` 和 `pypinyin`**（用于公司名归一化）。`dashscope/anthropic/tenacity/python-dotenv` 全部不用|

---

## 角色分工（关键认知）

┌─────────────────────────────────────────────────────────────┐  
│  Claude Code（本会话） = "vision + 网搜 + 编排" 的执行者    │  
│  ├─ Read 工具 → 看图（在 Agent subagent 内，避免主context爆） │  
│  ├─ WebFetch / WebSearch → 验证公司官网 + 主品类             │  
│  ├─ Write/Edit → 把中间结果写到 work/*.jsonl                 │  
│  └─ Bash → 跑 3 个 Python 小工具                             │  
├─────────────────────────────────────────────────────────────┤  
│  Python（仅 3 个文件）= "数学+IO" 的执行者                   │  
│  ├─ pipeline/normalize.py → 公司名归一化（rapidfuzz）        │  
│  ├─ pipeline/assign.py    → 评分匹配岗位（纯规则）           │  
│  └─ pipeline/build_xlsx.py → 读 jsonl 写 xlsx                │  
└─────────────────────────────────────────────────────────────┘

**核心原则**：所有 LLM 推理（看图、判定公司、找官网、匹配品类）都由**我在这个会话里完成**，不写 LLM 客户端代码。Python 只做确定性计算和文件 IO。

---

## 文件结构（最终）

/mnt/e/category-based-rd-talent-source-mapping/  
├── demo.xlsx                          # 只读模板  
├── manual_image_screening/             # 已有，87 张 JPG  
├── _photo_backup/                      # 已有，只读备份  
├── config/  
│   └── categories.yaml                 # 7 岗位品类映射（Phase A 落地）  
├── work/                               # 中间产物（每个 Phase 增量写入）  
│   ├── vision_raw.jsonl                # 每张图一行：sha1 / path / companies[]  
│   ├── companies.jsonl                 # 归一化后的 unique 公司列表  
│   ├── verified.jsonl                  # 加上官网+主品类  
│   ├── assignments.jsonl               # 加上命中岗位  
│   └── manual_review.csv               # 低置信图人工审清单  
├── output/  
│   └── results_{batch}_{ts}.xlsx  
├── pipeline/  
│   ├── normalize.py                    # rapidfuzz 合并  
│   ├── assign.py                       # 评分匹配  
│   └── build_xlsx.py                   # openpyxl 写 xlsx  
└── CLAUDE.md                           # 项目背景 + 复跑流程（CC 自动加载）

---

## 总执行序列：5 个 Phase

A: 脚手架 + categories.yaml + 3 个 Python 小工具  
   ↓ 闸门: build_xlsx 用假数据能产生合法 xlsx  
B: 3 张 fixture 图打通全流程  
   ↓ 闸门: 用户肉眼审 results_smoke.xlsx  
C: 浮选 35 张 + manual_review 回灌  
   ↓ 闸门: 用户审 csv → 浮选 xlsx 通过  
D: 电子展 52 张 + 跨批合并  
   ↓ 闸门: 最终 xlsx 通过  
E: CLAUDE.md + 单测 + git commit

跨闸门必须等用户确认，不连跑。

---

## Phase A — 脚手架（零 LLM 调用）

### 动作

1. `pip install rapidfuzz pypinyin`
    
2. 建目录：`pipeline/ config/ work/ output/ docs/`
    
3. 写 `config/categories.yaml`，按旧 plan §"7 岗位品类草案"七个段落落地，每岗位五维结构：
    
    P01:  
      position: 植物基绝缘油研发工程师  
      main_categories: [植物基绝缘油, 天然酯绝缘油, 生物基绝缘油, 环保变压器油]  
      related_categories: [矿物绝缘油, 硅油绝缘液, 变压器油]  
      processes: [油脂精炼, 抗氧化处理, 酯交换, 脱水脱气]  
      applications: [配电变压器, 电力变压器, 新能源变压器]  
      english_aliases: [natural ester, FR3, vegetable oil transformer fluid, bio-based dielectric]  
      industry_tags: [电力设备, 变压器, 新能源]  
      P02 特殊：额外加 scope_note + negative_keywords  
    P02: ...
    
4. 写 `pipeline/normalize.py`：
    
    - 输入 `work/vision_raw.jsonl` → 输出 `work/companies.jsonl`
        
    - 后缀剥离表：`["有限公司","股份有限公司","集团","科技有限公司","新材料","股份"]`
        
    - 地名前缀分离（用 pypinyin 拿首字母分桶后再合并），RapidFuzz `token_set_ratio>=88` 视为同一家
        
    - 保留 `aliases[]` 备查
        
    - 低置信图（`confidence<0.6`）单独写 `work/manual_review.csv`
        
5. 写 `pipeline/assign.py`：
    
    - 输入 `work/verified.jsonl` + `config/categories.yaml`
        
    - 按旧 plan §"关键决策"-4 评分公式：
        
        score = 1.0·主品类命中(产品文本) + 0.6·主品类命中(URL文本)  
              + 0.3·关联品类命中 + 0.2·图上品类重合
        
    - `score>=0.6` 命中，允许多岗位
        
    - 输出 `work/assignments.jsonl`
        
6. 写 `pipeline/build_xlsx.py`：
    
    - `Workbook()` 全新创建（不从 demo 拷）
        
    - 列宽：A 列 28，B+ 列 40
        
    - **Sheet1（横向）**：A 列 7 岗位（按 categories.yaml 顺序），公司从 B 列横向铺。单元格：
        
        cell.value = f'=HYPERLINK("{official_url}","{full_name}")'  
        cell.font = Font(color="0563C1", underline="single")
        
    - **Sheet2（纵向明细）**：列 = 公司全称 / aliases / 官网 / 主品类 / 命中岗位 / 来源图 / vision_confidence / url_confidence
        
    - **Sheet3（unmatched）**：公司全称 / 官网 / 主品类 / 来源图 / 漏匹配原因
        
    - 表头加粗，`ws.freeze_panes = "A2"`
        
    - 输出 `output/results_{batch}_{ts}.xlsx` + 拷贝一份 `output/results_latest.xlsx`
        

### 闸门 A

构造一个最小 fake `work/verified.jsonl` 和 fake `work/assignments.jsonl`（2-3 条），跑 `python -m pipeline.build_xlsx` 产出 xlsx，用 Excel 打开：

- 三 sheet 都在
    
- B2.公式正确，点击打开网页
    
- cell 显示只是公司全称（不带 URL）
    
- 列宽合适
    

**LLM 消耗**：0（纯 Python）

---

## Phase B — 3 张 Fixture 图打通全流程

### 角色操作清单（我做的事）

1. 用户先挑 3 张 fixture 图（清晰 1、模糊 1、多公司同框 1），路径告诉我
    
2. **我用 Agent subagent 并行读 3 张图**（每个 agent 读 1 张图，prompt 锚定：要求输出严格 JSON、公司全称尝试补全、低置信打 `low_confidence_reason`、多公司用 `companies[]` 数组）。Agent 返回 JSON 我写进 `work/vision_raw.jsonl`
    
3. 跑 `python -m pipeline.normalize` → `work/companies.jsonl`
    
4. **我用 WebFetch/WebSearch 串行验证 3-5 家公司**：搜全称 → 命中官网 → 读首页/产品页 → 对照 categories.yaml 判定主品类 → 写 `work/verified.jsonl`
    
5. 跑 `python -m pipeline.assign` → `work/assignments.jsonl`
    
6. 跑 `python -m pipeline.build_xlsx --batch smoke` → `output/results_smoke_{ts}.xlsx`
    

### Subagent prompt 模板（关键，会复用）

你是研发岗品类识别助手。读这张图，按 JSON 输出图里所有可见的"参展公司"信息。  
​  
输出严格 JSON 格式（response_format=json_object 思路，但你直接吐 ```json 块就行）：  
{  
  "companies": [  
    {  
      "raw_text": "图上看到的原文（包括简称、Logo 文字）",  
      "candidate_full_name": "尽量补全为公司全称（如'武汉泽电'补成'武汉泽电新材料有限公司'）",  
      "confidence": 0.0-1.0,  
      "low_confidence_reason": "如果<0.6，说明原因；否则空字符串",  
      "evidence_quote": "图里支持你识别的可见文字片段",  
      "visible_categories": ["图上可见的产品/品类关键词"]  
    }  
  ]  
}  
​  
锚点：  
- 0.9+ = 全称清晰可读  
- 0.6-0.9 = 简称/部分裁切，但能猜  
- <0.6 = 拼写不确定/识别困难

### 闸门 B

- 3 张图至少 1 张有合法的"公司全称 + 官网 + 主品类"
    
- xlsx 打开正常，hyperlink 公式渲染对，cell 只显示公司全称
    
- 用户口头确认：公式风格、列宽、sheet 划分都满意
    
- **不满意就在这里改 prompt / 改 build_xlsx，不带到 C/D**
    

**LLM 消耗**：3 张图 vision + ~5 次 web fetch（在 CC 订阅额度内）

---

## Phase C — 浮选 35 张 + 人工审

### 我的操作

1. 把 35 张图分 7-12 个 batch 给 Agent subagent 并行读（每个 agent 读 3-5 张，避免单 agent 上下文过载、又避免 87 个 agent 启动开销）
    
2. 每个 batch agent 把结果写回 `work/vision_raw.jsonl`（追加模式，sha1 做幂等）
    
3. `python -m pipeline.normalize` → companies.jsonl + manual_review.csv
    
4. **看 manual_review.csv 数量**：如果超过 10 条，**先停**——把 csv 给用户在 Excel 里审，他填 `Y/N/correct_full_name` 后我回灌
    
5. 对 unique 公司列表（预计 25-35 家）批量 WebFetch+WebSearch 验证 → verified.jsonl
    
6. assign + build_xlsx → results_浮选_{ts}.xlsx
    

### 闸门 C

- 浮选 xlsx 用户验收通过
    
- manual_review.csv 已审完并回灌
    

**LLM 消耗**：35 张 vision + ~30 次 web verify

---

## Phase D — 电子展 52 张 + 最终合并

### 我的操作

1. 同 C 流程，处理 52 张
    
2. **合并**：手动（或写个临时 Python 片段）把 `work/verified_浮选.jsonl` 和 `work/verified_电子展.jsonl` 合并到 `work/verified_all.jsonl`，公司全称去重、aliases 累加、保留"来源展会"列
    
3. `python -m pipeline.assign` + `python -m pipeline.build_xlsx --batch final` → `output/results_final_{ts}.xlsx`
    

### 闸门 D

最终 xlsx 用户验收。Sheet2 明细里能看到"来源展会"。

**LLM 消耗**：52 张 vision + ~40 次 web verify

---

## Phase E — 收尾

### 动作

1. 写 **repo 根的 `CLAUDE.md`**（Claude Code 打开 repo 时自动加载），结构：
    
    - **项目目的**（一段话）：7 个研发岗 × 87 张展会图 → 竞企图谱，给未来 Claude 一眼看懂
        
    - **架构约束**：α 模式（CC 会话本体执行 vision+网搜，不用外部 LLM SDK）
        
    - **6 条关键决策**（沿用 plan 末尾的拍板表）
        
    - **重跑流程**：a) 把新 JPG 放进 `manual_image_screening/{新展会名}/`；b) 在本 repo 开 CC 说"按本剧本 Phase B-D 跑新一批"；c) `python -m pipeline.build_xlsx --batch {名}`
        
    - **categories.yaml 改法**：新增/调整岗位的字段约束（main/related/processes/applications/aliases/tags + P02 类特殊岗位的 scope_note/negative_keywords）
        
    - **已知坑**：B2 公式 vs C2 拼接的区别、demo.xlsx 永远只读、低置信阈值 0.6 的来源、87 张图 ≠ 87 个 HEIC（一对多）
        
2. pytest 三个最易回归的点：
    
    - `tests/test_normalize.py`：`武汉泽电` / `武汉泽电新材料` / `武汉泽电新材料有限公司` 必须合并成 1 条
        
    - `tests/test_assign.py`：构造 verified 数据，断言匹配岗位与人工标注一致
        
    - `tests/test_build_xlsx.py`：断言 Sheet1.A2 是岗位名、B2 单元格值以 `=HYPERLINK(` 开头、Font color="0563C1"
        
3. git commit 一次（消息 `feat: 完成首版 87 张图全流程`），把 `pipeline/ config/ docs/ tests/` 一起入库；`work/ cache/ output/` 维持 gitignored
    

---

## 关键文件清单

|文件|Phase|来源|
|---|---|---|
|`config/categories.yaml`|A|抄旧 plan §"7 岗位品类草案"|
|`pipeline/normalize.py`|A|新写（rapidfuzz）|
|`pipeline/assign.py`|A|新写（评分公式）|
|`pipeline/build_xlsx.py`|A|新写（openpyxl）|
|`work/vision_raw.jsonl`|B/C/D|我读图后追加写|
|`work/manual_review.csv`|C/D|normalize 输出|
|`work/verified.jsonl`|B/C/D|我 web verify 后写|
|`output/results_*.xlsx`|B/C/D|build_xlsx 产出|
|`tests/test_*.py`|E|新写|
|`CLAUDE.md`（repo 根）|E|新写|

只读：

- `demo.xlsx`、`manual_image_screening/*`、`_photo_backup/*`
    

---

## 端到端验证

|验证|Phase|怎么测|
|---|---|---|
|categories.yaml 结构对|A|normalize+assign 拿假数据能跑通|
|build_xlsx 公式渲染|A+B|Excel 打开点链接|
|公司名归一化|A+E|normalize 模块 + pytest|
|Vision 抓得到中文标牌|B|3 张 fixture 肉眼审|
|网搜歧义消解|B|把展会名塞 prompt 防找错同名公司|
|岗位匹配规则评分|B+E|assign 模块 + pytest|
|断点续传|C|中途 ctrl+c，jsonl 追加模式自然续跑（sha1 幂等）|
|Manual review 回灌|C|csv 改完，我读 csv 把 correct_full_name 灌回 companies.jsonl|
|跨批合并去重|D|同公司在两展都出现 → 最终 xlsx 应只占 1 列|
|回归基线|C/D|demo.xlsx 已有的"武汉泽电"/"江苏双江"作 ground truth|

---

## 用户已拍板的关键决策

|#|议题|决策|
|---|---|---|
|1|LLM 后端|**不用 DashScope/Anthropic SDK**，由 CC 会话里的 Claude 直接读图+网搜|
|2|Excel 起点|`Workbook()` 全新创建，不从 demo 拷|
|3|Hyperlink 风格|`=HYPERLINK(url, 全称)` 公式法，cell 只显示公司全称|
|4|M1 位置|合进 Phase A 一起做|
|5|Sheet3 unmatched|Phase B 必交付|
|6|复用频率|每年 1-2 次 → 留 RUNBOOK.md，不留全自动脚本|

---

## 不在本次范围

- ❌ 第 4 步反搜（通过品类找新参展企业）
    
- ❌ HEIC→JPG 转换（已离线完成）
    
- ❌ 简历库 / 招聘系统对接
    
- ❌ Web UI / API 服务化
    
- ❌ DashScope 或 Anthropic API key 配置（这版彻底不用）
    

## 显式不做的"持久化基建"（已与用户对齐）

- ❌ **自定义 `.claude/skills/`**：单项目、年频复跑，复用度不够，维护开销大于收益
    
- ❌ **`settings.json` permissions 预配置**：执行时如有摩擦，用 `/fewer-permission-prompts` 扫一份就行，不预先建
    
- ❌ **hooks**：本项目没有"每次保存触发 X"这类自动化需求
    
- ❌ **单独 RUNBOOK.md**：已并入 `CLAUDE.md`