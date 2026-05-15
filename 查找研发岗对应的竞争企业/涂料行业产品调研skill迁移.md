# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `25c68c4e-32d2-4d65-a61c-65e47896c531`
- **范围**: Turn 1-7
- **导出时间**: 2026-05-13 16:22:18

---

### Turn 1  *(2026-05-13 07:02)*

**User:**

E:\coatings_industry_competitor_research这个是涂料行业产品调研，和现在这个项目相比，它构建了完整的skills文档，现在这个项目是否也需要skills；能从涂料行业产品调研项目中学习到什么，能用在现在这个项目的

**Assistant:**

我把两个项目都看了一遍，对比如下。

## 涂料项目的 skill 长什么样

```
.claude/skills/research-one-company/
├── SKILL.md                          # 648 行，Phase 0→9 + trace_token
├── references/site-patterns.md       # 259 行，8 类站点判定手册
└── templates/template_*.py           # 5 个可跑骨架（static_aspx / wordpress_dt / spa_api / image_codes / jz_listing）
```

关键机制：
- **trace_token** (`SKILL-RESEARCH-ONE-COMPANY-v11-20260511`) — subagent 返回 JSON 必带，主会话校验；SKILL.md 升一版 token 升一版，能识别"没读最新规则的 subagent"
- **CLAUDE.md vs SKILL.md 分工严判** — CLAUDE.md 只写主会话职责（5 条），subagent 流程全沉淀进 SKILL.md，避免规则两处维护
- **Phase 0→9 硬规则 + 💡 软建议**显式标注；每条规则带"实测踩坑案例（公司名+日期）"做反例教学
- **Phase 8 自检 checklist**（10+ 项）写 JSON 前必过
- **失败模式专章**（Phase 9）：超时/局部失败/BLOCKED 判定/abort 必落产物

## 当前项目要不要也做 skill — 建议**做**

你已经有 `.claude/agents/exhibition-verifier.md`（165 行 subagent 定义），结构上跟涂料项目 SKILL.md 的角色一样，但目前还偏"prompt 模板"而非"playbook"。

**做的理由**：剩 6 岗（P01/P02/P03/P04/P06/P07）每岗位要派多个 batch subagent，每个 batch 30-35 家。N×6 倍的复用面，跟涂料项目 100+ 公司调研是同级别。模式 B 的踩坑（推断池 / TLS 错 / 同集团 / 大矿企客户混淆 / fuzz 漏 dedup）已经够多够具体，值得固化。

## 具体能搬过来的 6 件事

| # | 涂料项目做法 | 搬到本项目怎么做 |
|---|---|---|
| 1 | **trace_token 版本化** | exhibition-verifier.md frontmatter 加 `trace_token: SKILL-EXH-VERIFIER-v1-20260513`，要求 subagent JSONL 第一行或 final message 回带；主对话校验，升级踩坑规则跟着升版 |
| 2 | **CLAUDE.md / SKILL.md 角色分工** | CLAUDE.md 只留主会话职责（Step 1 展会发现 / Step 5 跨批次合并 + manual dedup / audit 三层防护 / xlsx build）；模式 B 当前埋在 CLAUDE.md 200+ 行的"subagent 干什么"细节迁到 exhibition-verifier.md |
| 3 | **Phase 0→N 硬规则结构 + 踩坑实例** | 现 agent 文件是平铺的"三种模式 + JSON schema + 工作纪律"，可以改成 Phase 1 任务输入校验 → Phase 2 品牌简称搜索 → Phase 3 url_check → Phase 4 cross-position scan → Phase 5 exhibit_sources 三层证据 → Phase 6 self-check & 输出。每条规则带"(FL02 推断池 2026-05-13)"这种带日期案例 |
| 4 | **写 JSONL 前 self-check checklist** | 现在没有；可以加：`[]is_inferred 标了吗 / []exhibit_sources.evidence_quote 不是空的 / []company_type 都填了 / []大矿企集团没误标 main_categories / []阿里巴巴/慧聪 URL 配 url_confidence ≤ 0.6 / []TLS 错 notes 里有 重校提示` |
| 5 | **references/ 资源** | 新建 `.claude/skills/exhibition-verifier/references/`：① `exhibition-sources.md`（4 类入口可达性排序：协会页 / 展会官网 SSR / 第三方汇总站 / 行业头部公司池）② `audit-three-layer.md`（Wayback + 本地 evidence/<tag>.md + 原 URL 流程） |
| 6 | **templates/ 资源** | ① `template_cross_batch_merge.py`（fuzz 70-99 scan + MERGE_PAIRS/LINK_PAIRS 骨架，现在还是 inline script）② `template_pre_filter.py`（Group A/B/C/N 分类器骨架） |

## 我的建议起手

最值钱的两步先做：

1. **把 exhibition-verifier.md 加 trace_token + Phase 结构 + self-check checklist**（涂料项目里这是最防错的机制，且改动小，1 次 edit 能搞定）
2. **把 CLAUDE.md 「模式 B」章节里的 subagent 流程细节迁到 agent 文件**，CLAUDE.md 留主会话职责索引 + 工作产物表 — 防止后续踩坑修一处忘一处

`references/` 和 `templates/` 是次优先级，等真跑 P01/P02 时如果发现展会入口又踩新坑再开。

要不要我直接开始改 `.claude/agents/exhibition-verifier.md` 加 trace_token 和 Phase 结构？还是先把 CLAUDE.md「模式 B」章节迁移做了？

---

### Turn 2  *(2026-05-13 07:46)*

**User:**

把这最值钱的两步先做了

**Assistant:**

把当前 `exhibition-verifier.md` 重写成 Phase 结构 + trace_token + checklist，然后把 CLAUDE.md 模式 B Step 4 段对应清掉。

更新 CLAUDE.md Step 4 加上 skill_trace 校验逻辑；已知坑表加固化提示。

两步都做完了。

**1. `.claude/agents/exhibition-verifier.md` 重写（165 → 263 行）**
- frontmatter 加 `trace_token: SKILL-EXH-VERIFIER-v2-20260513`
- Phase 0→8 硬规则结构：Phase 0 必读 → 1 任务卡 → 1.5 scrape 专用 → 2/3 verify 主流程 → 3.r retry 6 fallback → 4 url_check → 5 品类命中 + company_type 严判 → 6 exhibit_sources → 7 schema + checklist → 8 失败模式
- **TL;DR 顶部 3 条"不能违反"**：品牌简称铁律 / company_type 严判 / 推断池真实标
- **Phase 7.2 写盘前 self-check checklist (11 项)**：full_name 一字不差 / is_inferred 真标 / company_type 不为空 / 大矿企 main=[] / 设备厂品类正确 / 聚合站 url_confidence ≤ 0.6 / TLS 错不预判 dead / product_text 不含 P0X main 字面 / JSONL 合法 / skill_trace 必带
- 把 CLAUDE.md「assign 子串否定语境陷阱」「SSL 反爬误杀」等 subagent 视角的踩坑搬进 Phase 4/5
- final message 必带 `skill_trace` 字段，主对话校验用

**2. `CLAUDE.md` Step 4 升级 + 已知坑表注归属**
- Step 4 加 Phase 0→8 索引表（一眼看 agent 内部到底固化了什么）
- 新增 **Step 4.1 skill_trace 校验** + 4 种不匹配处置策略 + 升级流程
- 已知坑表加表头说明（主对话职责 vs subagent 视角），逐行标 `→agent Phase N`：推断池(1.5/6) / SPA SSL 反爬(4) / 子串否定语境(5) / 同集团(8) / 暗星(3.r) / TLS(4)

下次升 agent 内规则只要 bump `trace_token` 末尾日期，主对话 `EXPECTED_TRACE` 同步即可。要不要试着派一个 P01 的 batch 1 验证流程跑通？或继续做 references/templates？

---

### Turn 3  *(2026-05-13 07:54)*

**User:**

继续做

**Assistant:**

继续做 references + templates 之前，先看一下能拿来当样板的现有资源。

按 4 个资源做：playwright 骨架 / evidence MD 模板 / 展商入口手册 / audit 三层流程。先建目录写文件。

把 references 链接接到 agent 和 CLAUDE.md，trace_token 升 v3。

现在在 Phase 0 加 references 引用：

现在更新 CLAUDE.md：升 EXPECTED_TRACE、加 docs/ 到文件结构、Step 2.5 引用 template、Audit 段引用 reference。

4 个文件落盘，5 处 CLAUDE.md + 2 处 agent 引用已接上。

**做了的事**

```
docs/mode-b/
├── references/
│   ├── exhibition-sources.md      # 5 类入口可达性 + 决策树 + 4 个实战踩坑（cnmn/productronica/china-mining/武汉矿业展）
│   └── audit-three-layer.md       # 三层防护 / Wayback 触发命令 / 哪些核心 evidence 必做 / 命名约定速查
└── templates/
    ├── template_playwright_scrape.py   # SPA 抓取骨架，从 productronica 实战提炼；TODO 4 处占位 + 无限滚动退化路径
    └── template_evidence_md.md         # evidence MD 模板，按 ⚠️ 填空；附填写提示
```

**接入位置**
- `.claude/agents/exhibition-verifier.md` Phase 0「按需 Read」段加 2 个 references（trace_token 升 v3）
- `CLAUDE.md` Step 2 入口列表末尾、Step 2.5 模板段、Audit 三层防护段、文件结构表、`EXPECTED_TRACE` 字符串各 1 处

**还没做（用户当时判断"演化中"，暂不抽）**
- `template_cross_batch_merge.py` — 跨批次 fuzz dedup + MERGE_PAIRS/LINK_PAIRS 骨架
- `template_pre_filter.py` — Group A/B/C/N 分类骨架

下次跑新岗位（P01/P02/P04）时如果 cross-batch merge 又踩同样的坑，再抽出来不迟。

---

### Turn 4  *(2026-05-13 08:01)*

**User:**

  - template_cross_batch_merge.py — 跨批次 fuzz dedup + MERGE_PAIRS/LINK_PAIRS 骨架
  - template_pre_filter.py — Group A/B/C/N 分类骨架
这是做什么的

**Assistant:**

两个都是**主对话**用的本地脚本（不是 subagent 用的），对应 CLAUDE.md 模式 B Step 3 和 Step 5 里现在还 inline 写在会话里的零散代码。

## `template_pre_filter.py`（Step 3 用）

**输入**：scrape 模式抓到的展商 raw jsonl（如 `work/vision_raw_exh_FL02.jsonl`，每行 `{full_name, ...}`）
**输出**：4 个 jsonl，把展商按名字分到 A/B/C/N 四组

```
Group A = 名字含品类强关键词（化工/化学/药剂/材料/试剂/精细/捕收/起泡）  → verify 高优
Group B = 明显不对口（XX 集团 / XX 矿业 / 活性炭 / 贸易 / 设备 / 服务）   → 跳过 verify，直接 unmatched
Group C = 与已有 verified.jsonl fuzzy >= 90（跨批次重复家）              → 跳过 verify，复用旧 URL
Group N = 名字模糊（既不像 A 也不像 B）                                  → 仍需 verify
```

**作用**：减少 subagent 工作量 ~50%。P05/FL02 实数：原始 252 家 → A 66 + N 60 = 126 家进 verify，**省 50%**。Group B 118 家直接归 unmatched 但 HR 仍可见。

骨架长这样：

```python
A_KEYWORDS = ["化工", "化学", "药剂", "材料", "试剂", "精细", ...]  # 按岗位调整
B_KEYWORDS = ["集团", "矿业", "活性炭", "贸易", "科技服务", ...]
EXISTING_VERIFIED = "work/verified_all_v4.jsonl"  # 跨批次比对

def classify(name, existing):
    if any(fuzz.ratio(name, e["full_name"]) >= 90 for e in existing): return "C"
    if any(kw in name for kw in A_KEYWORDS): return "A"
    if any(kw in name for kw in B_KEYWORDS): return "B"
    return "N"
```

## `template_cross_batch_merge.py`（Step 5 用）

**输入**：多个 batch verify 产物（如 `work/p05_verify_batch1.jsonl` + `batch2.jsonl` + ... + Group C 复用）
**输出**：单一 `work/verified_all_v<N>.jsonl`（合并 + dedup + 累加 source_exhibits）

**做的事**：
1. 把所有 batch jsonl 加载成 list
2. 对全 list pairwise 跑 `rapidfuzz` 算公司名相似度
3. 把分数落在 **70-99** 区间的对吐出来让人审（100 已经在 `pipeline/normalize.py` 处理；< 70 不太可能是同公司）
4. 人审分两类：
   - **MERGE_PAIRS**：同一家不同写法（"陶氏化学 Dow DOWSIL" + "陶氏化学 Dow Chemical Company"）→ 合并成一条，aliases 累加，source_exhibits 累加
   - **LINK_PAIRS**：同集团不同子公司（"爱森中国" + "河南爱森"；"武汉朗英" + "武汉强盛"）→ **不合并**，但在双方 `related_companies` 字段标关系
5. 写出合并后的 verified.jsonl

**为什么单独抽**：`pipeline/normalize.py` 跑的是自动 pinyin bucket dedup，对全角/半角混合、英文+中文混写、子公司命名差异这种边缘场景识别不到。手审环节绕不开但流程固定。

骨架长这样：

```python
# 1. 合并加载
all_entries = []
for f in BATCH_FILES:
    all_entries.extend(json.loads(l) for l in open(f))

# 2. fuzz scan 70-99 区间
suspects = []
for i, a in enumerate(all_entries):
    for j, b in enumerate(all_entries[i+1:], i+1):
        score = fuzz.token_set_ratio(a["full_name"], b["full_name"])
        if 70 <= score < 100:
            suspects.append((score, a["full_name"], b["full_name"]))

# 3. 人审填表 (主对话每次跑新批次都要写)
MERGE_PAIRS = [("陶氏化学 Dow DOWSIL", "陶氏化学 Dow Chemical Company"), ...]
LINK_PAIRS  = [("爱森中国", "河南爱森", "SNF 子公司"), ...]

# 4. 执行合并
for a, b in MERGE_PAIRS:
    merge(a, b)  # aliases 累加 / source_exhibits 累加 / url 取 url_confidence 高的
for a, b, rel in LINK_PAIRS:
    add_related(a, b, rel)  # 不合并，双方 related_companies 标关系
```

---

## 当时为什么不抽

CLAUDE.md Step 5 末尾备注：

> 参考实现：本会话留下的 inline scripts（cross_batch_merge + manual dedup），**未抽成 `pipeline/*.py` 模块化是因为这部分还在演化中**。

"演化中"指 MERGE 判定规则在 P05 实操中持续变化（举例：什么时候算"同集团"？事业部 vs 母公司算合并还是 link？外资中国子公司算独立还是合并到全球母公司？）。抽得太早 = 模板里写死的规则下次就要改。

抽 template 的合理时机：跑完 P03 同模式后再跑 1-2 个岗位（P06/P07/P01），观察 3-4 个岗位下的 MERGE 决策共性，再抽。

要现在抽吗？现在抽的价值：你跑新岗位时不用重写 fuzz scan + 人审流程，省 30-60 min/批次。代价：模板可能写死 P05 偏好（如某些化工集团的合并规则），P03/P06 用时要改。

要不就先抽个 minimal 版本（只做 fuzz scan + suspects 输出，不内置 MERGE 规则）？规则部分让用户每次跑时根据当批数据 inline 填？

---

### Turn 5  *(2026-05-13 08:04)*

**User:**

只做 fuzz scan + suspects 输出，不内置 MERGE 规则，这是什么意思。规则部分让用户每次跑时根据当批数据 inline 填，这又是什么意思

**Assistant:**

意思是 template 只做"机械工作"的部分，把"判断工作"留给跑脚本的人临时填。具体长这样：

## "fuzz scan + suspects 输出"（template 做的）

输入：合并后的 `all_entries.jsonl`（所有 batch 拼起来）
输出：一个 `suspects.txt`，列出所有相似度 70-99 的公司对：

```
score=88  | 陶氏化学 Dow DOWSIL          | 陶氏化学 Dow Chemical Company
score=85  | 爱森（中国）絮凝剂           | 河南爱森化学
score=78  | 江西铜业集团（铅山）         | 江西铜业（贵溪）冶炼厂
score=92  | 山东金安化工有限公司         | 山东金安化工
score=73  | 紫金矿业集团                 | 福建紫金选矿药剂
```

脚本就到此为止 — **它不知道这些对里哪些该合并、哪些该 link、哪些根本无关**。

## "inline 填规则"（每次跑的人做的）

人打开 `suspects.txt` 肉眼扫一遍，然后在**当次跑的脚本里**直接写两个 list：

```python
# === 跑这批时人审填的 ===
MERGE_PAIRS = [
    ("陶氏化学 Dow DOWSIL", "陶氏化学 Dow Chemical Company"),  # 一家不同写法
    ("山东金安化工有限公司", "山东金安化工"),                   # 一家有/无"有限公司"后缀
]

LINK_PAIRS = [
    ("爱森（中国）絮凝剂", "河南爱森化学", "SNF 集团子公司"),
    ("武汉朗英", "武汉强盛", "事业部 vs 母公司"),
]

# 没列进 MERGE 或 LINK 的 suspects 默认 = 无关，不动
# 例如 "江西铜业（铅山）" vs "江西铜业（贵溪）" 是两个不同分公司，都参展过，保留为独立 entries
# "紫金矿业" vs "福建紫金选矿药剂" 母公司是矿企客户、子公司是药剂厂，关系不大，保留独立
```

这两个 list **不写进 template**，是每次跑新批次时根据当批的 suspects 现填的。填完调 template 里的合并函数：

```python
merge_apply(all_entries, MERGE_PAIRS, LINK_PAIRS)
```

## 对比：如果"内置 MERGE 规则"长什么样

就是把判断逻辑写进 template 自己：

```python
# template 内置（坏做法）
def should_merge(a, b):
    # 后缀去掉后一样就合
    if strip_suffix(a) == strip_suffix(b): return True
    # 都含"陶氏" / "Dow" 就合
    if "陶氏" in a and "陶氏" in b: return True
    # ...
```

问题：
- "都含陶氏就合"对吗？"陶氏化学" 和 "陶氏家居建材"是两家完全无关公司，规则误合
- P05 实战发现"江西铜业(铅山)" vs "江西铜业(贵溪)"是两个独立分公司，但规则可能误判同公司
- P03 跑下来"3M 中国" vs "3M Electronic Solutions" 是同一家，但 P05 没这种 case，规则没覆盖到

**判断"两家公司算不算同一家"依赖外部知识**（集团结构、地理分布、业务线划分），规则写得再细也覆盖不全 — 而且每个岗位的边缘 case 不一样。

## 总结对比

| 方式 | template 包含 | 人每次做 |
|---|---|---|
| **内置规则** | fuzz scan + 合并判断 + 执行 | 跑脚本，盲信结果 |
| **suspects 输出（推荐）** | fuzz scan + 合并执行函数 | 看 suspects.txt → inline 填 MERGE_PAIRS / LINK_PAIRS → 跑合并 |

第二种把**机械活**（pairwise 比较 + 累加 aliases / source_exhibits）固化进 template，把**判断活**（这两家是不是一家）留给人，每次跑现填。这样跨岗位复用稳。

你刚才"演化中所以暂不抽"的担心，主要担心的就是"内置规则"那种 — 那种确实会被边缘 case 反复打脸。"suspects 输出"版本没这风险，要抽吗？

---

### Turn 6  *(2026-05-13 08:07)*

**User:**

suspects 输出，那做这个吧

**Assistant:**

更新 CLAUDE.md Step 5 + 文件结构表把新 template 接上。

Template 落盘（263 行，语法 OK），CLAUDE.md Step 5 + 文件结构表已接上。

**`template_cross_batch_merge.py` 长这样**

两步命令：

```bash
python3 work/_p06_merge.py scan    # → 产 _p06_suspects.txt
# 人审 → 填 MERGE_PAIRS / LINK_PAIRS
python3 work/_p06_merge.py merge   # → 产 verified_all_p06_v1.jsonl
```

**scan 段做的事**：
- 加载多个 batch jsonl，先做 exact-name dedup（同 full_name 累加 source_exhibits + exhibit_sources）
- pairwise 跑 `fuzz.token_set_ratio`，先 `strip_suffix` 再比（避免"X 有限公司" vs "X" 拉低分）
- 收集 70-99 区间的对，按分数倒序写 `_suspects.txt`

**merge 段做的事**：
- 读 `MERGE_PAIRS` / `LINK_PAIRS`（人填的）
- merge：aliases / source_images / source_exhibits / exhibit_sources（按 tag 去重）累加；main/related/visible_categories 取并集；url 取 url_confidence 高的；product_text 拼接
- link：双方 `related_companies` 互标关系，不合并
- 写出最终 jsonl

**与 `pipeline/normalize.py` 的边界**：
- normalize.py 跑自动合并（FUZZ_THRESHOLD=88，pinyin bucket + token_set_ratio），处理 >= 88 的同公司
- 本 template 跑 70-99 区间，捕 normalize.py 自动合并漏掉的边缘 case（"陶氏 Dow DOWSIL" vs "陶氏 Dow Chemical Company" 这种 token 差异大但是同公司的）
- 区间不重叠，互补

剩 `template_pre_filter.py` 还没做，要继续吗？

---

### Turn 7  *(2026-05-13 08:12)*

**User:**

继续

**Assistant:**

Template 落盘（137 行，语法 OK），CLAUDE.md Step 3 + 文件结构表已接上。

**`template_pre_filter.py` 长这样**

```bash
# 改完 TODO 1 路径 + TODO 2 关键词后单步跑
python3 work/_p06_pre_filter.py
```

**核心逻辑**：

```python
def classify(name, existing_names):
    # 1. C 优先（跨批次重复不重复 verify）
    if fuzz.token_set_ratio(name, ex) >= 90 for ex in existing_names: return "C"
    # 2. B 优先于 A（避免"宝钢化工"误归 A — 钢铁客户而非药剂厂）
    if any(kw in name for kw in B_KEYWORDS): return "B"
    if any(kw in name for kw in A_KEYWORDS): return "A"
    return "N"
```

**4 个岗位的实战关键词示例已内联在注释里**（P05 浮选 / P03 三防漆 / P01 绝缘油 / P04 润滑脂），照着改即可。

**末尾打印统计**：每组数量 + 占比 + 下一步指示。例如：

```
=== vision_raw_exh_FL02.jsonl → p05_pre_filter/ ===
total: 252
  Group A:   66 ( 26.2%)  高概率对口 → 派 subagent verify（高优）
  Group B:  118 ( 46.8%)  低概率 → 跳过 verify，直接 unmatched
  Group C:    8 (  3.2%)  跨批次重复 → 跳过 verify，复用旧 URL
  Group N:   60 ( 23.8%)  模糊 → 派 subagent verify（次优）

下一步：派 subagent verify Group A + N，合计 126 家（节省 126 家 verify quota）
```

---

## 整轮做下来的全貌

```
.claude/agents/exhibition-verifier.md              # Phase 0→8 + trace_token v3 + 11 项 self-check
docs/mode-b/
├── references/
│   ├── exhibition-sources.md                      # 5 类入口可达性 + 决策树 + 实战踩坑
│   └── audit-three-layer.md                       # 三层防护 / Wayback 触发 / 哪些必做
└── templates/
    ├── template_playwright_scrape.py              # SPA 抓取（productronica 实战提炼）
    ├── template_pre_filter.py                     # Group A/B/C/N 分类（B 优先于 A）
    ├── template_cross_batch_merge.py              # fuzz 70-99 suspects + MERGE/LINK 人审
    └── template_evidence_md.md                    # evidence MD 模板
```

CLAUDE.md 全部更新到位（Step 2 / 2.5 / 3 / 5 / Audit 三层 / 文件结构 / `EXPECTED_TRACE`）。

下次跑 P01/P02/P04/P06/P07 任一岗位时，从 4 个 template 各拷一份到 `work/_<P>_*.py` 改 TODO 占位即可走完整流程。

要不要顺手把这一轮的改动 commit？

---
