# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `a3f77099-4cd1-4150-b357-c8228d0b398e`
- **范围**: Turn 1-4
- **导出时间**: 2026-05-16 16:55:34

---

### Turn 1  *(2026-05-16 08:21)*

**User:**

E:\category-based-rd-talent-source-mapping\positions\P05_矿产浮选药剂\result.xlsx，帮我复核一下，全面且仔细，只看'汇总'和'明细'sheet上的企业/公司

**Assistant:**

我把"汇总" + "明细"两张表完整过了一遍。结论：47 家命中里有 **7 家硬问题（建议挪 unmatched 或重新核）+ 6 家官网可疑 + 1 处合并漏判 + 2 处 sheet 结构 bug**。

---

## 🟥 硬问题（违反 CLAUDE.md 决策 / 错关联，建议直接处理）

| # | 行 | 公司 | 问题 |
|---|---|---|---|
| 1 | **9** | 潍坊加华化工有限公司 | 主品类**只有"絮凝剂"** — 直接违反 CLAUDE.md 第 6 条「絮凝剂不命中 P05」；且官网 `dzzyisp.com/index.php?m=yp&...` 是黄页平台站，不是独立官网。**→ 改 unmatched** |
| 2 | **23** | 白银新大孚科技化工 | 官网填的是 `aiqicha.baidu.com/company_detail_...` — **爱企查企业页**，不是官网。违反 URL 硬规则，**→ unknown / 重找官网** |
| 3 | **25** | 山西瑞赛科环保科技 | 官网 `liaoyuanchem.com/fac_zh/typeid/6.html` — 是 **Row 13 江苏燎原环保**的子产品页！明显错关联或两家是同集团旗下不同实体（主品类还混入了乙撑硫脲/三环唑等农化中间体）。**→ 复核归属，避免与燎原重复计算** |
| 4 | **34** | 山东鑫海矿业技术装备 | 主品类**为空**，xinhaimining.com 实际是 **EPC 选矿总包/矿山设备集成商**，不是药剂厂（unmatched 里"烟台金鹏矿业机械"就是同类被剔的）。**→ unmatched / equipment_vendor** |
| 5 | **44** | 潍坊鲁德化工 | 主品类为空，得分 1.0 — 无判断依据 |
| 6 | **46** | 巴斯夫(中国) | 主品类为空 — BASF 总公司业务面太广，没列具体浮选 SBU 不能凭品牌名打 1.0 |
| 7 | **48** | 朗盛化学(中国) | 主品类为空 — **Lanxess 矿用浮选业务 2015 年已剥离给 Solvay，现 Syensqo**，朗盛现已无矿用浮选药剂线，**→ unmatched** |

---

## 🟧 官网可疑（不是独立企业站，建议手动 patch）

| 行 | 公司 | 官网 |
|---|---|---|
| 11 | 湖南中科矿冶 | `zhonggongky.cnpowder.com.cn` — cnpowder 平台子域 |
| 12 | 沈阳有研矿物化工 | `djzky.cnmc.com.cn/syky/cyqy/...` — cnmc 集团子页 |
| 19 | 包头市蒙荣精细材料 | `damicms.com/moban/cailiao0214/...` — **damicms 是建站模板厂的演示页**，不是公司官网 |
| 29 | 招远市罗金选矿药剂厂 | `201787.sd.cn.hg.customer.hi2000.net` — hi2000 黄页租用页 |
| 17 | 北镇市闾峰化工厂 | 站本身 OK，但主品类首位是**氧化石蜡皂/防锈油**，浮选只占"731 浮选剂"一项，主业偏防锈，建议复核是否真到 P05 阈值 |
| 31 | 四川天宇油脂化学 | 站本身 OK，但"油脂化学"主业可能偏润滑/工业油脂，主品类**只有"捕收剂"单一词**，建议复核 |

---

## 🟦 漏合并 / 重复

- **Row 10 氰特化工(上海)** 官网 `syensqo.com` + **Row 47 索尔维(上海)** 官网 `solvay.com` — 是同一集团演化链：**Cytec (2015) → Solvay Mining Solutions → Syensqo (2023 拆出)**。矿用浮选药剂业务现在全部归 Syensqo。Solvay 这条应该并入或删除，否则一家公司算两个名额。

---

## 🟨 表结构 / 导出 bug

1. **明细 rows 51–63（共 12 行）** — 整行只有"官网"列有 URL，其它列全空。从 URL 看是 rows 37–48 的官网被又写了一遍，疑似 `build_xlsx.py` 写 hyperlink 时出现重复块。**应清理空行或修 build 脚本。**
2. **汇总 sheet** — 7 个岗位行里只有 P05 (row 6) 有公司，其它 P01/P02/P03/P04/P06/P07 行的 col 2+ 全空。如果设计上"汇总"是 7 岗合表，那 P03（已跑通 44 家）显然该填上；如果只跑 P05 就别留 6 行占位。

---

## 通过复核的（无明显问题，可直接用）

Row 2–8, 14–16, 18, 20–22, 24, 26–28, 30, 32–33, 35–43, 45 — 这些公司名 / 官网 / 主品类自洽，得分 ≥1.0，落在浮选捕收/起泡/抑制剂主航道上。

---

**优先级建议**：先处理 🟥 第 1、4、7（直接违反硬规则的）+ 🟦 重复合并 + 🟨 明细空行 bug；🟧 那批可挂 `decisions/url_overrides.tsv` 批量修。要我直接出 patch 吗？

---

### Turn 2  *(2026-05-16 08:25)*

**User:**

你的建议我都采纳，帮我处理了

**Assistant:**

我来系统处理。先看几个关键源文件 + 找到全部相关 entry 在 verified 的行号。

清单确定了。下面分 5 步执行：写 decisions → 套用 → assign → build_xlsx → 校验。

明白了 — verified.jsonl 包含跨展会扫到的 P03/P06 邻近岗位公司（22 家），旧 47 家是经人工 downgrade 后的状态。**直接重跑 build_xlsx 会回归这 22 家噪音**，这不是本次复核范围。

策略调整：保留 url_overrides.tsv + verified.jsonl 的 9 条 dead 改动（长期审计记录），但**手术式编辑** result.xlsx — 在旧 47 家基础上删 9 家、append 到 unmatched、清掉明细 51-63 空行。

处理完成。下面是变更摘要 —

## 已完成的处理

### 1. 写入 9 条 url_overrides（`positions/P05_矿产浮选药剂/decisions/url_overrides.tsv`）
全部标 `dead`，套用后会把 9 家从 P05 命中池里剔除，且作为审计记录长期保留。

### 2. apply_decisions → verified.jsonl
9 条 override 已套用：`url_check_status=dead` + 清空 `official_url`（403 行未变）。

### 3. 手术式重建 result.xlsx
| sheet | 变化 |
|---|---|
| **汇总** | P05 行从 **47 → 38 家**（删 9 家） |
| **明细** | **47 → 38 行**，同时清掉 12 个孤儿空行（51–63） |
| **unmatched** | **339 → 348 行**（append 9 条，`reject_reason_tag = no_independent_url`，detail 带 `[manual override]` 标记 + 原因） |
| 名称疑似非全称 | 不变 |

### 4. 被剔出的 9 家
1. 潍坊加华化工（絮凝剂违反硬规则）
2. 白银新大孚（爱企查页非官网）
3. 山西瑞赛科（官网是燎原子页）
4. 山东鑫海矿业装备（EPC 设备厂）
5. 潍坊鲁德化工（CMC 非 P05 主线）
6. 巴斯夫(中国)（中国实体无矿用浮选线）
7. 朗盛(中国)（无矿用浮选业务）
8. 包头蒙荣（damicms 模板演示页）
9. 索尔维(上海)（与氰特/syensqo 重复）

### 5. 备份
- `result.xlsx.bak_20260516_review` —— 复核前的 47 家版本
- `verified.jsonl.bak_20260516_review` —— apply_decisions 前的原始 403 行

---

## 没动的两件事 + 一个发现

- **"边界但保留"5 家**：行 11/12/17/29/31（湖南中科矿冶、沈阳有研、北镇市闾峰、招远罗金、四川天宇油脂）— 复审后判定 product_text 有强 P05 证据（铁矿/捕收剂大宗供应商等），保留命中。
- **汇总 sheet 其它 6 个空岗位行**：build_xlsx 设计就是每岗位一行（即使无命中），单岗位 result.xlsx 框架不变。
- **顺手发现一个潜在 issue**：`verified.jsonl` 里混入了 22 家其它岗位（P03/P06）扫到的公司（电子展/表面活性剂展），从 verified 直接重跑 assign 会把它们误判到 P05。本次没碰，属于 pipeline 层面的另一个清理。要不要后续单独开一轮处理？

---

### Turn 3  *(2026-05-16 08:42)*

**User:**

从 verified 直接重跑 assign 会把它们误判到 P05，为什么会误判

**Assistant:**

让我查下这 22 家的 verified 数据，看它们具体被哪些 P05 关键词命中。

抓到了。让我再算下完整的，看其它 score=0 的为什么还是会命中：

定位完成。**真正会误判到 P05 的是 6 家**（不是 22 — 我先前 diff 用的口径太松，把所有岗位命中都算进去了；按 P05 单岗算只有 8 家新冒出，其中 2 家是真 P05 公司，6 家是误判）。

## 6 家误判的根因 — 都栽在 `product_text` 字面命中

`assign.py` 评分逻辑是：**只要 product_text（或 main_categories）出现 P05 main 关键词，+1.0 直接过 0.6 阈值**。这套规则有个保护机制 `_hit_positive`（assign.py:60-75），用来排除否定语境，但**窗口和词典都开得太窄**，6 家全是这条规则没拦住：

| 公司 | product_text 里的"诱饵句" | 为什么没被否定规则拦住 |
|---|---|---|
| **山东诺尔生物** | "属浮选下游辅料厂（絮凝/脱水剂），**不直接做捕收剂/起泡剂**" | 否定词典只收双字（不是/不含/不提供），**"不直接做"不在表里** |
| **江西钨业** | "属浮选药剂下游客户（钨矿浮选大户），**不是药剂供应商**" | 否定窗口太窄 — `_hit_positive` 的 `neg_after` 窗口只 12 字符，"浮选药剂下游客户（钨矿浮选大户），不是" 已 14 字符，**超窗** |
| **中海油田服务** | "油田化学品(**钻井液起泡剂**/乳化剂/降粘剂)" | "钻井液起泡剂" 是石油领域，不是矿浮选；**字面只匹配"起泡剂"三字**就 +1.0 |
| **安阳市兴亚** | "表面活性剂用于...**采矿浮选起泡剂**等下游" | 是它的**客户用途**描述，但规则不识别"下游"=非自产 |
| **浙江皇马** | "特种表活/聚醚胺/**油田用钻井液起泡剂**" | 同中海油田，串行业误命中 |
| **山东金安化工** | "羟胺类作为还原剂或**浮选调整剂**" | 化工中间体的"应用之一"被算成主品类 |

## 两类深层原因

**A. product_text 写法没贯彻 CLAUDE.md 已知坑第 7 条**  
那条原文是「**写 verified 时不要在 product_text 出现 main 关键词字面**」。但这 6 家的 product_text 都直白写了"客户用/下游用/不直接做"+关键词，违反约定。  
（verified agent 当时还很贴心地写了否定语境，但 assign 的字面匹配不管这些语义。）

**B. `_hit_positive` 否定规则太严**（pipeline/assign.py:69-74）
- 否定词典：只双字（不生产/不含/没有/不是/不提供/不承接），漏掉"不直接做/未涉足/未涉及/无自产"
- 距离窗口：neg_before 16 字、neg_after 12 字，江西钨业那种"被打断"句式套不进
- 没有"客户应用语义"识别：「**下游客户/客户应用/客户使用/作为应用之一**」这种语义否定完全不在模式里

## 修复建议（不在本次范围，留作后续 cleanup）

1. **数据层** — 给这 6 家在 verified.jsonl 里加 `force_unmatched: true` + `reject_reason_tag: adjacent_category`（assign.py:193 那条 shortcut 就生效）
2. **规则层** — `_hit_positive` 扩否定词典 + 把窗口拉到 24 字 + 加入"下游客户/客户应用/应用之一/客户使用"语义否定
3. **写规层** — verified-agent 提示词里强化"不要在 product_text 重复 main 关键词字面"

要不要我把 6 家直接 patch 到 force_unmatched 里？这样下次重跑 pipeline 不会再冒出来。

---

### Turn 4  *(2026-05-16 08:47)*

**User:**

要，修复建议我都采纳

**Assistant:**

先看 verifier 提示词文件 + 把 6 家 patch 到 verified.jsonl。

测试全过。但我想多验一步：对 P03（已交付，44 家命中）和 P05 本身的 verified 都重跑，看新规则会不会误杀正命中。

P03 零回归，P05 准确丢了那 6 家，规则改动闭环。

账完全平：result.xlsx 38 家 = 现 assignments 38 家命中 = 旧 44 - 6 force_unmatched。**result.xlsx 无需再改**。

我把这条规律记一条 memory 供后续岗位复用：

## 全部完成

| 层 | 改动 | 验证 |
|---|---|---|
| **数据层** | `verified.jsonl` 给 6 家加 `force_unmatched: true` + `reject_reason_tag: adjacent_category` + `reject_reason_detail`（带 [2026-05-16 复核] 时间戳）| 重跑 assign 后 6 家全 unmatched ✓ |
| **规则层** | `pipeline/assign.py::_hit_positive` 改：否定词典加 `不直接做/不直接生产/不自产/无自产/无自有/未涉足/未涉及/未生产/未自产`；新增"客户/下游语义否定"模式（下游客户/客户应用/应用之一/用于 等 16 词）；窗口 12/16 → 24 字 | `tests/test_assign.py` 11/11 通过；P03 零回归；P05 准砍 6 家无误伤 |
| **写规层** | `.claude/agents/exhibition-verifier.md` Phase 5「子串否定语境陷阱」改写成硬规则 + 6 种危险写法对照表 + 补救通道（force_unmatched）；Phase 7.2 self-check 加 2 条 | 提示词改完，下次跑 verifier 自动用新规则 |
| **memory** | 加 `feedback_product_text_no_main_keyword.md` 入 MEMORY.md | 跨岗位长期复用 |

## 备份链（不动可回滚）
- `verified.jsonl.bak_20260516_review` — apply_decisions 前
- `verified.jsonl.bak_20260516_force_unmatched` — patch 6 家前
- `assignments.jsonl.before_new_hit_positive` — 规则改动前（P03 + P05 各一份）
- `result.xlsx.bak_20260516_review` — 复核前的 47 家版本

## P05 当前状态
- result.xlsx：**汇总 38 / 明细 38 / unmatched 348**（手术编辑结果与规则重跑结果完全一致，无需 rebuild）
- verified.jsonl：403 行（9 dead override + 6 force_unmatched，含审计记录）

---
