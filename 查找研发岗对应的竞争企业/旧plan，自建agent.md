# 研发岗品类竞企图片解析流水线 —— 验收计划

## Context（为什么做）

业务目标：为公司 7 个研发岗位找到对应的"竞争企业"，用途有二——主动挖人 / 招聘时优先关注从这些公司离职的人员。

入口资料：董事长在 2 场展会现场（浮选药剂展、电子展）拍的 87 张照片，已 HEIC→JPG、按展会分组完成。`demo.xlsx` 是用户手填的样板（武汉泽电、江苏双江各一条），定义了最终交付格式。

本轮范围（用户确认）：完成原始计划的**第 1-3 步**（品类持久化 → 图片解析 → 网搜验证 → Excel 汇总），**不做**第 4 步"通过品类反搜展会扩展参展企业"——它工作量是前 3 步的数倍，需要独立立项。

预期产出：一个 `results_{timestamp}.xlsx`（双 sheet）+ 完整的中间产物（图片解析报告、人工审核清单）+ 可复跑的 Python 流水线代码。

---

## 已确认的事实（输入资源现状）

- 项目根：`E:\category-based-rd-talent-source-mapping\`（WSL: `/mnt/e/category-based-rd-talent-source-mapping/`）
    
- 图片：`manual_image_screening/浮选/`（35 张 JPG）+ `manual_image_screening/电子展/`（52 张 JPG），共 87 张
    
- HEIC 原图备份在 `_photo_backup/展会图片/{浮选药剂,电子展化学品}/`
    
- 模板：`demo.xlsx`，结构 = A 列岗位 + B/C/... 横向铺公司，单元格用 HYPERLINK（display 文本"简称+网址"，target 是官网 URL）
    
- 7 岗位已在 demo.xlsx A3:A8 + A2 = 全部 7 行
    
- **⚠️ demo.xlsx 当前被 Excel 打开**（存在 `~$demo.xlsx` 锁文件）—— 写入策略：**永不写 demo.xlsx，始终输出新文件**
    

## 已确认的方案选型（用户拍板）

|维度|选型|
|---|---|
|视觉模型|**阿里 Qwen3-VL-Plus**（DashScope 模型 ID：`qwen3-vl-plus`，2026 Qwen 系列旗舰，复杂光线/模糊/倾斜场景文字识别强化）|
|网搜验证|**阿里 Qwen-Plus 联网版**（`qwen-plus` + `enable_search=true`，模型自带联网搜索能力，无需另配搜索 API）|
|Excel 输出|双 sheet：Sheet1 横向汇总（沿用 demo 格式）+ Sheet2 纵向明细|
|**公司名**|**必须用全称**（如"武汉泽电新材料有限公司"），不再用简称；display 文本 = `{full_name}{official_url}`|
|本轮范围|第 1-3 步（图片方向收口）|
|品类映射表|我起草、用户审|
|API 凭据|用户已有阿里云 DashScope API key|

---

## 推荐实现方案（模块化流水线）

### 流水线总览

[7 岗位] → M1.bootstrap → categories.yaml  
                              ↓  
[87 张 JPG] → M2.vision_parser → vision_raw.jsonl + 每张图缓存  
              （调 DashScope qwen3-vl-plus）  
                              ↓  
              M3a.normalize → companies.jsonl + manual_review.csv  
                              ↓（低置信走人工通道）  
              M3b.web_verify → verified.jsonl  
              （调 DashScope qwen-plus 联网版 enable_search=true，  
               让模型自己搜公司官网 + 主营品类）  
                              ↓  
              M4.assign → assignments.jsonl（纯规则评分匹配岗位）  
                              ↓  
              M5.excel_writer → output/results_{timestamp}.xlsx

### 关键决策与理由

1. **公司名归一化**：纯规则（后缀剥离 `有限公司/股份/集团/科技/新材料`、地名前缀分离、拼音首字母分桶、RapidFuzz `token_set_ratio≥88` 合并）。便宜、可复跑、不消耗 API。预计 87 张图的 ~100-150 个 raw 名压到 40-60 unique，网搜量直接砍半。
    
2. **置信度量化**：让 Qwen3-VL-Plus 在结构化 JSON 输出里自评，prompt 内显式锚点（`0.9+ = 全称清晰可读；0.6-0.9 = 简称/部分裁切；<0.6 = 拼写不确定`），强制"低分理由"字段。`temperature=0` + 用 `response_format={"type":"json_object"}` 强制 JSON。低置信阈值 `<0.6` 一律不入网搜、直接进 `manual_review.csv`。**重要**：图片识别出的可能是简称（"武汉泽电"），prompt 要明确要求模型"尽量补全为公司全称"，并在 `candidate_full_name` 和 `raw_text` 两个字段分开记录。
    
3. **网搜验证**：用 DashScope 的 `qwen-plus` 模型 + 联网开关（`extra_body={"enable_search": True}`）。一次调用内模型自己执行：搜公司全称 → 命中官网 → 读首页/产品页 → 对照 categories.yaml 判定主营品类 → 输出官网 URL + 全称 + 品类匹配。同 DashScope key，无额外凭据。注意：联网版 token 计费比纯文本略贵（~2 倍），但对 40-60 家公司可控。
    
4. **岗位匹配**：**纯规则评分**，不让 LLM 直接打"该公司属于岗位 X"的标签（避免幻觉、保留可解释性）。评分公式：
    
    score = 1.0·核心词命中(产品文本) + 0.6·核心词命中(URL文本)  
          + 0.3·关联词命中 + 0.2·图上可见品类重合
    
    `score≥0.6` 即匹配；允许一家匹配多岗位（业务现实，如绝缘油厂可能也做润滑脂）。
    
5. **断点续传**：`sha1(image_bytes)` 和 `dedup_key` 当 cache key，写在 `cache/vision/{sha1}.json` 和 `cache/web/{dedup_key}.json`。重跑零额外 token。API 调用前写 `inflight` 占位、成功后原子改名（防半写）。CLI 参数 `--max-api-calls` 控成本。
    
6. **Excel 输出**：永不覆盖 demo.xlsx；时间戳新文件 + `results_latest.xlsx` 软拷贝。HYPERLINK 用 `openpyxl.worksheet.hyperlink.Hyperlink`，**display = "{公司全称}{官网URL}"**（如 `武汉泽电新材料有限公司https://www.zdoil.cn/`，与用户在 demo 里看到的 sharedString 一致；不用简称），target = 官网 URL，蓝色下划线字体显式设置。Sheet3 额外加 `unmatched` 备查。
    

### 文件目录组织（建议）

/mnt/e/category-based-rd-talent-source-mapping/  
├── demo.xlsx                          # 模板，只读  
├── manual_image_screening/             # 已有  
├── _photo_backup/                      # 已有  
├── pipeline/                           # 新建  
│   ├── category_bootstrap.py  
│   ├── vision_parser.py  
│   ├── normalize.py  
│   ├── web_verify.py  
│   ├── assign.py  
│   ├── excel_writer.py  
│   ├── common/                         # io / logging / dashscope_client / image_utils  
│   └── run_all.py                      # 一键编排  
├── config/categories.yaml              # 品类映射表  
├── work/                               # 中间产物（jsonl + manual_review.csv）  
├── cache/                              # 断点续传（vision/, web/）  
├── output/results_{timestamp}.xlsx  
└── logs/run_{timestamp}.log

### Python 依赖

dashscope >= 1.20   Pillow   openpyxl >= 3.1   pypinyin   rapidfuzz  
PyYAML   pydantic >= 2   tenacity   python-dotenv   rich   pytest

环境变量：`DASHSCOPE_API_KEY`（`.env`，gitignore）

### 预算预估

- Vision (Qwen3-VL-Plus)：87 次调用，每张约 1.5-2k input tokens（含 image tokens）+ 600 output tokens；按 ¥8/百万 input tokens、¥24/百万 output tokens 计 **≈ ¥3-5 人民币**
    
- Web verify (Qwen-Plus 联网版)：40-60 次调用，每次 3-5k input + 800 output，联网版 token 计费 ~2x ≈ **¥5-10 人民币**
    
- **单跑总预算约 ¥10-20 人民币**，可接受
    

---

## 7 岗位 → 品类/工艺关键词草案（请审）

按 `主品类（强匹配）/ 关联品类（弱匹配）/ 典型工艺 / 典型应用 / 英文别名` 五维组织。会持久化到 `config/categories.yaml`。

### P01 植物基绝缘油研发工程师

- **主品类**：植物基绝缘油、天然酯绝缘油、生物基绝缘油、环保变压器油
    
- **关联**：矿物绝缘油、硅油绝缘液、变压器油
    
- **工艺**：油脂精炼、抗氧化处理、酯交换、脱水脱气
    
- **应用**：配电变压器、电力变压器、新能源变压器、干式→油浸替代
    
- **英文**：natural ester、vegetable oil transformer fluid、FR3、bio-based dielectric
    
- **行业标签**：电力设备 / 变压器 / 新能源
    

### P02 热处理工艺工程师（按用户补充已收紧）

**用户补充的招聘画像**：三家以上热处理企业经验、40 岁以下、金属材料及热处理及相关专业、熟悉热处理介质和工艺、能按工件/材质/设备设计热处理工艺。**结论：目标是承接金属热处理外协加工业务的工厂，不是把热处理当辅助工序的下游用户企业**。

- **主品类（强匹配）**：金属热处理加工服务、热处理外协、第三方热处理、表面处理服务、渗碳处理、渗氮处理、淬火/回火、调质处理
    
- **关联品类（弱匹配）**：热处理设备制造（多用炉、网带炉、真空炉、感应淬火机床）、热处理介质供应（淬火油、淬火盐、回火液、冷却介质）、表面强化技术服务
    
- **工艺**：真空淬火、离子/气体渗氮、可控气氛渗碳、高频/中频感应淬火、激光淬火、低温回火、深冷处理
    
- **应用线索（可作为佐证、不作为主匹配）**：轴承、齿轮、模具、紧固件、弹簧、汽车零部件——见到这些只能说明用过热处理，不是目标企业
    
- **英文**：heat treatment service、metal heat treating、carburizing、nitriding、quenching、tempering、case hardening、vacuum furnace、induction hardening
    
- **negative_keywords（排除）**：塑料热处理、食品热处理、热处理设备代理商（只卖不做）、热处理咨询培训
    
- **行业标签**：金属加工服务 / 热处理产业链
    

### P03 三防漆研发工程师

- **主品类**：三防漆、防潮漆、防霉漆、防盐雾漆、保形涂料、PCB 涂覆
    
- **关联**：电子涂覆胶、丙烯酸涂层、聚氨酯涂层、UV 涂层
    
- **工艺**：喷涂、浸涂、刷涂、选择性涂覆、UV 固化、热固化
    
- **应用**：PCBA 防护、汽车电子、消费电子、军工电子
    
- **英文**：conformal coating、PCB coating、acrylic coating、urethane coating、silicone coating
    
- **行业标签**：电子化学品 / PCB / 汽车电子
    

### P04 特种润滑脂研发工程师

- **主品类**：特种润滑脂、锂基脂、复合锂基脂、聚脲脂、硅脂、全氟聚醚脂（PFPE）
    
- **关联**：钙基脂、钠基脂、极压润滑脂、高温润滑脂、食品级润滑脂
    
- **工艺**：皂化反应、高温分散、稠化剂合成、后处理工艺
    
- **应用**：轴承润滑、高温环境、真空环境、汽车、风电、食品机械
    
- **英文**：grease、lithium grease、polyurea grease、PFPE grease、silicone grease、high temperature grease
    
- **行业标签**：润滑剂 / 工业润滑
    

### P05 矿产浮选药剂研发工程师

- **主品类**：浮选药剂、捕收剂、起泡剂、调整剂、抑制剂、活化剂
    
- **关联**：选矿药剂、矿用化学品、絮凝剂、磨矿助剂
    
- **工艺**：化学合成、表面活性剂合成、浮选选矿配套
    
- **应用**：铜矿、钼矿、铅锌矿、黄金、稀土、磷矿、煤炭
    
- **英文**：flotation reagent、collector、frother、depressant、activator、xanthate
    
- **行业标签**：选矿 / 矿冶 / 化工
    

### P06 灌封胶/导热胶研发工程师

- **主品类**：灌封胶、导热胶、有机硅灌封胶、环氧灌封胶、聚氨酯灌封胶、导热硅脂、导热凝胶
    
- **关联**：电子胶粘剂、结构胶、UV 胶、导热界面材料 (TIM)、导热垫片
    
- **工艺**：双组分配方、缩合固化、加成型固化（铂金催化）、热界面材料填料
    
- **应用**：电源模块封装、LED 灌封、动力电池、电机灌封、IGBT 模块、5G 基站
    
- **英文**：potting compound、TIM (thermal interface material)、encapsulant、thermal gel、gap filler
    
- **行业标签**：电子胶粘剂 / 新能源 / 电力电子
    

### P07 助焊剂研发工程师

- **主品类**：助焊剂、免清洗助焊剂、水溶性助焊剂、无卤助焊剂、焊膏、锡膏
    
- **关联**：锡线、焊接材料、清洗剂、表面活性剂
    
- **工艺**：配方研发、表面活性剂筛选、防腐处理、活化剂调控
    
- **应用**：SMT 焊接、波峰焊、选择性焊接、半导体封装、PCB 后处理
    
- **英文**：flux、soldering flux、no-clean flux、water-soluble flux、solder paste、solder wire
    
- **行业标签**：电子化学品 / SMT / 半导体
    

---

## 风险点与缓解

|#|风险|缓解|
|---|---|---|
|R1|Qwen3-VL 自评置信度漂移|prompt 加锚点示例；跑完 87 张后人工抽 20 张校准；< 0.6 一律人工审|
|R2|同公司不同变体未合并|RapidFuzz 阈值 88 + 保留 aliases[] + 输出 dedup 报告人工抽检|
|R3|多公司同框/合影|prompt 强制 companies[] 数组 + evidence_quote 锁区域|
|R4|联网搜索找错同名公司|把展会名（浮选/电子展）和图上可见品类一起塞 prompt 做歧义消解；要求输出 url_confidence < 0.5 时显式标 `needs_manual_search=true`|
|R5|官网首页无品类信息|prompt 允许模型搜次级页面/产品页；多轮失败兜底直接保留全称无 URL，标 needs_manual|
|R6|DashScope 限流/网络抖动|tenacity 指数退避 + 失败不写 cache 可自动续；联网版 QPS 一般比纯文本低，并发 ≤2|
|R7|demo.xlsx 文件锁|永远输出新文件，不就地写|
|R8|API 成本失控|`--max-api-calls` 上限 + cache + 低置信先离线审|
|R9|"热处理工艺工程师"过宽|在 categories.yaml 里加 `scope_note` 字段，匹配时用 negative_keywords 兜底（如排除"塑料热处理"）|

---

## 关键文件清单

待创建：

- `/mnt/e/category-based-rd-talent-source-mapping/config/categories.yaml`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/run_all.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/vision_parser.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/normalize.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/web_verify.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/assign.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/excel_writer.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/pipeline/common/*.py`
    
- `/mnt/e/category-based-rd-talent-source-mapping/.env`（放 `DASHSCOPE_API_KEY`）
    

只读模板：

- `/mnt/e/category-based-rd-talent-source-mapping/demo.xlsx`
    

---

## 端到端验证方法

1. **冒烟测试**：浮选目录手挑 3 张图（清晰 / 模糊 / 多公司同框）做 fixture，跑完全流程肉眼对比
    
2. **单测**（pytest）：
    
    - `normalize`：`["武汉泽电","武汉泽电新材料","武汉泽电新材料有限公司"]` 必须合并成 1 条
        
    - `assign`：构造 verified 数据，断言匹配岗位与人工标注一致
        
    - `excel_writer`：打开输出文件断言 `Sheet1.A2 == "植物基绝缘油研发工程师"`，`B2.hyperlink.target` 是 https 开头
        
3. **集成 dry-run**：3 张图全流程，对比格式 vs demo.xlsx
    
4. **生产跑分批**：浮选 35 张 → 用户审一轮 manual_review.csv → 电子展 52 张
    
5. **回归基线**：demo.xlsx 已填的武汉泽电、江苏双江当 ground truth，看 pipeline 能否独立识别
    

---

## 验收检查清单（已全部确认）

- [x]  视觉模型 = **Qwen3-VL-Plus**（取代原 Claude Opus 4.7 vision；2026 国内最强中文视觉模型）
- [x]  网搜验证 = **Qwen-Plus 联网版**（`enable_search=true`，取代原 Anthropic web_search）
- [x]  公司名 = **必须全称**（不能简称）
- [x]  P02 热处理 = 收紧为"金属热处理加工服务企业"，关联热处理设备/介质供应商
- [x]  其余 6 岗位品类草案采纳
- [x]  输出 = `results_{timestamp}.xlsx` 时间戳新文件，**不动 demo.xlsx**
- [x]  单跑预算 ≈ ¥10-20 人民币（vision ¥3-5 + 联网搜 ¥5-10）
- [x]  manual_review.csv 工作流：低置信图（<0.6）写 csv → 用 Excel 打开 → 看图、对照、填 Y/N 和修正名 → 跑 `--merge-manual` 回灌
- [x]  API 凭据：DashScope API key 已就绪

下一步：按本方案落地代码（M1→M2→M3a→M3b→M4→M5），分批跑（先浮选 35 张 → 用户审 → 再电子展 52 张）。