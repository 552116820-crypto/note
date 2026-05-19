
### Turn 5  *(2026-05-19 01:32)*

**User:**

所以skill graph 就是 overclaim了吧，官网即没有相关定义，也不符合skill的设计哲学。像"E:\coatings_industry_competitor_research\.claude\skills\research-one-company"，这个skill复杂度明显超出范围，难以做到跨项目

**Assistant:**

同意你的判断。让我先看一眼你举的那个 skill 的实际样子，给出有依据的具体分析。

你的判断完全成立。看了这个 skill 的实情，问题比"跨项目难"更深 — 它本质上**根本不是 skill，是项目 SOP 借了 skill 的壳**。

## 这个 skill 的具体反模式

### 1. SKILL.md 38KB，frontmatter description 几百字

正常的 description 是 1-3 行"做什么 + 何时触发"，这里写成了一段产品说明书。Skill 索引的预算被它一个人吃掉一大块。

### 2. 绝对路径硬编码到项目根目录

```
/mnt/e/coatings_industry_competitor_research/CLAUDE.md
/mnt/e/coatings_industry_competitor_research/findings/_shensaier_redo/METHOD.md
```

这一行就锁死了跨项目可能。把项目挪个位置、复制到别的机器都会断。

### 3. 依赖项目内的 Python 业务模块

```python
from scripts.lib.fetch import fetch, fetch_html, ...
from scripts.lib.ocr   import ocr_image, ocr_url
```

Skill 应该是"知识 + 流程指令"，不是"调用我们家自研业务库的人"。这种耦合一旦换项目，整个 skill 失效。

### 4. 把行业、数据 schema、工作流全焊死

涂料/油墨/化工行业判断、findings JSON schema、9 个 Phase 的固定顺序、trace_token、status JSON — 这些都是**这个项目独有的**协议，跨项目零意义。

### 5. 9 个 Phase + "违反就重做"

这是 SOP / 项目运行手册，不是 skill。Skill 是被模型按需召唤的工具，SOP 是被人/上层流程严格遵循的规章 — 强制顺序、强制重做、强制不能跳步，这些跟 skill 的"可发现、可组合、按需激活"是冲突的。

## 这种东西的正确形态

| 内容 | 应该放在哪 |
|---|---|
| 12 条硬性纪律、工具入口、坐标系约定 | 项目根 `CLAUDE.md` |
| 9 个 Phase 的具体流程、踩坑案例、JSON schema | 项目内 `docs/research-sop.md` 或 `playbooks/` |
| `fetch_html` / `ocr_url` 这些业务函数 | `scripts/lib/` 的 Python 库（项目内代码） |
| 5 类站点判定模板、4 个 template_*.py | 项目内 `templates/`，跟 skill 无关 |
| **"调研一家公司"作为 skill 的部分** | 一个 200-500 字的 SKILL.md，只写："本项目有 `docs/research-sop.md` 详细 SOP，调研单家公司请按它执行；产出落到 `findings/{批次}/{简称}.json`" |

也就是说 — skill 退化成一个**指针**：告诉模型项目里有个 SOP、入口在哪、产物落到哪。**真正的内容都在项目文档里**。这反而完全符合 skill 的设计哲学：薄、可发现、激活后引导模型去读真正的资料。

## 判断标准（你可以拿这个量任何 skill）

一个 skill 应该满足:

1. **跨项目能用** — 把它 cp 到另一个 repo，理论上还能工作（路径、数据 schema、业务库都不该硬编码）
2. **自包含** — 不要求"先去读项目里某个特定文件"才能开始工作（references 例外，但 references 是 skill 自己的目录）
3. **description ≤ 100 字** — 能在元数据列表里被模型一眼扫到
4. **主体 ≤ 几千字** — 详细内容拆到 references/
5. **召唤式，不是 SOP 式** — 模型在合适时机自然激活，不是"必须严格按步骤执行不能跳步"

满足 0-1 条：是项目工作流，**不是 skill**，写到项目 CLAUDE.md / docs。
满足 2-3 条：可以做成项目级 skill，但需要剥皮。
满足 4-5 条：才是健康的可跨项目 skill。

`research-one-company` 现在大概只满足第 5 条一半（在项目里能被召唤）。其余全部踩雷。这跟"复杂"无关 — 复杂的领域研究流程本来就该用文档 + 子 agent 组合解决，不该塞进 skill。

No response requested.

---

### Turn 6  *(2026-05-19 02:27)*

**User:**

有什么优秀的skill

**Assistant:**

继续我们刚才的判断标准来回答 — 但先说一个老实话：判断一个 skill 是否"优秀"必须看内部结构（SKILL.md 主体 + references 设计），光看 description 只能猜个轮廓。我能直接确认的样本只有刚才那一个 `agentic-harness-patterns-zh`，其他都得打开才知道。

## 已经确认的好样本：`agentic-harness-patterns-zh`

刚才你已经看过它的目录结构了。它的"好"在哪：

1. **主 SKILL.md 控制在 15KB**，没把所有细节硬塞进去
2. **references/ 用得到位** — 9 个模式各自一个文件，每个 ~12KB，互相不嵌套依赖，按主题切分
3. **再嵌套了 references/context-engineering/ 子目录** — 当某个主题本身就大（select / compress / isolate 三个 pattern），再下钻一层，而不是把这一层撑爆
4. **主文档里用"深入阅读"表格组织所有 references 链接**，模型能看完主册自己判断该翻哪本附录 — 真正的 progressive disclosure
5. **`不适用于`节** — 显式划清边界，告诉模型什么时候**别**用这个 skill

这是一个跨项目能用、自包含、薄、召唤式 — 五条全过的 skill。

## 从 description 形态可以**初步**推测可能优秀的（待验证）

按你上一轮认可的标准，几个 description 形态看起来不错的：

| Skill | description 的好特征 |
|---|---|
| `claude-api` | 罕见地显式写了 `TRIGGER when:` 和 `SKIP:` 两段，告诉模型何时该召唤、何时该忽略 — 这是避免"误触发"的高级写法 |
| `update-config` | 结构齐全：用途 + 触发模式（"from now on when X"）+ 多个 examples + 边界（"For simple settings, suggest /config instead"） |
| `init` | 极短 description，意图单一，没有任何模糊空间 |
| `keybindings-help` | "Use when..." + 例子，clean |

这几个的 description 写法是好的；但它们 SKILL.md 主体是否也保持薄、references 是否切得合理、有没有硬编码项目路径之类的坑，**没打开看就不能说**。

## description 形态可能值得警惕的

- `frontend-design` 的 description 偏宣传语 — "distinctive, production-grade, high design quality, creative, polished, avoids generic AI aesthetics"。这种**形容词堆叠**通常说明作者还没想清楚"何时该用 vs 何时不该用"，只是在自我推销。但这只是 description 层的怀疑，主体也可能写得很扎实。

## 你想怎么继续

我可以挑 1-2 个打开来拆，给出像 `research-one-company` 那样的具体分析。建议从 `claude-api` 或 `update-config` 开始 — 这两个的 description 形态最像"成熟设计的 skill"，看它们的内部能不能学到可借鉴的范式。要不要开一个？

---

### Turn 7  *(2026-05-19 02:30)*

**User:**

继续

**Assistant:**

先定位这两个 skill 的位置，再拆。

不在用户目录，应该是内置 skill。找一下 Claude Code 安装位置。

内置 skill 都打包在 JS bundle 里、没有独立 SKILL.md 文件 — 我没法像看 `agentic-harness-patterns-zh` 那样拆开它们的内部结构。诚实地说，对 `claude-api` / `update-config` 我只能停留在 description 层面的初步判断。

不过我刚发现你用户目录里还有一个我之前漏掉的 skill — `skill-vetter`。从名字看就是"审核 skill 用的 skill"，很可能是个高质量样本。先看它。

`skill-vetter` 是个断链 symlink — 目标目录已经不存在了，所以也没法看。`frontend-design:frontend-design` 名字里的冒号说明是 plugin 提供的，可能在 plugin 目录里有源文件，最后试一下。

找到了。看一下 frontend-design 的结构：

看了之后我得**修正上一轮对 `frontend-design` 的怀疑**。它实际上是一个非常好的样本，且形态跟 `agentic-harness-patterns-zh` 完全相反 — 把两个一起对比，能比较好地讲清"优秀 skill"不止一种形态。

## 三个样本横向对比

| | `frontend-design` | `agentic-harness-patterns-zh` | `research-one-company`（反例） |
|---|---|---|---|
| SKILL.md 体量 | 41 行 / 1.5KB | 250 行 / 15KB | 800 行 / 38KB |
| references/ | 无 | 9 个文件 + 1 子目录 | 有，但主体已经撑爆 |
| 外部依赖 | 零 | 零 | 项目内 Python lib + 绝对路径 |
| 内容形态 | 设计原则手册 | 多模式索引 + 渐进披露 | 9 Phase SOP |
| 跨项目可用 | 完全可以 | 完全可以 | 完全不行 |
| 调用方式 | 召唤式 | 召唤式 | SOP 式（强制顺序） |

## `frontend-design` 好在哪（修正版）

我上一轮怀疑它 description 是"形容词堆叠"，看了主体发现是错判。它的设计哲学：

1. **极度克制的体量** — 41 行就够了，因为它要传达的不是流程，是**品味**。激活成本几乎为零。
2. **手册型，不是 SOP 型** — 没有"Step 1 / Phase 0",  给的是决策框架（Design Thinking 那一节问 4 个问题）+ 审美原则（5 个维度的具体指导）。模型可以根据具体任务灵活应用。
3. **显式列举反模式**（这点最有价值）："NEVER use Inter / Roboto / Arial / purple gradients on white / Space Grotesk" — **告诉模型什么不能做**比告诉它什么能做有效得多，因为 LLM 默认会回归训练数据里的平均值（generic AI aesthetics）。
4. **description 里的形容词在主体里被兑现** — "distinctive" 不是空话，主体里给了"typography 要 distinctive"的具体定义。
5. **结尾激励语** "Don't hold back, show what can truly be created" — 这是 prompt 工程的小撬点，能真实抬升模型努力度。

## 优秀 skill 的共性（基于这两个正例）

无论体量大小，正例都满足：

- **零外部硬编码** — 没有任何路径、业务库、项目特定 schema 写死在 SKILL.md 里
- **激活成本与主题广度匹配** — 审美指南就 1.5KB，多模式索引就 15KB + references 树，不强求统一
- **是手册不是 SOP** — 告诉模型决策框架和原则，不是强制顺序的 step-by-step
- **显式划边界** — `agentic-harness-patterns-zh` 有"本技能不适用于"节，`frontend-design` 有"NEVER use X"清单。**告诉模型何时不该用、不该怎么用**，是大多数 skill 漏掉的步骤
- **description ≤ 100 字 / 信息密度高** — 不堆形容词，写清"做什么 / 何时用"

## 关于其他没法直接看到的内置 skill

`claude-api`、`update-config`、`init`、`review`、`security-review` 等都打包在 Claude Code 的 JS bundle 里，没有独立文件。它们的内容是 Anthropic 自己写的、被官方维护的，**大概率质量很高**，但我没法给你做内部拆解 — 这是一个客观限制。你想看的话，可以等以后某次激活它们（比如真的运行 `/init` 或 `/review`），看激活后模型展现的具体行为模式来反推。

## 一个建议

如果你想沉淀"会写 skill"的判断力，我觉得最有用的练习是 — 把那个 `research-one-company` 按我们刚才的标准重构成：

- **极薄的 SKILL.md**（< 1KB），只写 "调研一家公司用 `docs/research-sop.md` 里的 SOP"
- 把 9 个 Phase 全部挪到项目内 `docs/research-sop.md`
- 把硬纪律挪到项目 `CLAUDE.md`
- 业务 Python 库就留在 `scripts/`，跟 skill 解耦

这个过程能让你切身体会"skill 的边界 vs 项目文档的边界"在哪里。要不要从这个开始动手？

---

### Turn 8  *(2026-05-19 02:38)*

**User:**

极薄的 SKILL.md（< 1KB），只写 "调研一家公司用 docs/research-sop.md 里的 SOP"，什么意思，要怎么写

**Assistant:**

具体讲。核心思路一句话：**skill 主体不再"包含知识"，只"指向知识"**。它从"39KB 的执行手册"退化成"几行字的入口卡片"，真正的内容挪到项目文档里。

## 直接给一份草稿

把原来 38KB 的 SKILL.md 重写成这样（连 frontmatter 总共 ≈ 20 行 / < 1KB）：

```markdown
---
name: research-one-company
description: 调研一家涂料/油墨/化工同行公司，输出 findings JSON。触发：用户说"调研 X 公司"/"做 X 这家"/"重做 X 这家"/"检查 X 这家有没有漏"，或主会话派 subagent 做单家调研。10+ 家批量调度走主会话 Agent 工具，不进本 skill。
---

# research-one-company

## 入口

按项目固化 SOP 执行。开工前必读：

1. `CLAUDE.md` — 硬性纪律 + 工具入口
2. `docs/research-sop.md` — 9 Phase 完整流程 + 踩坑实录 + JSON schema + 站点判定矩阵

按 SOP 顺序走完，不跳步。

## 产物契约

- 落盘：`findings/{批次}/{简称}.json`
- 返回主会话：status JSON，`skill_trace` 字段必填

## 不在本 skill 范围

- 10+ 家批量调度 → 主会话用 Agent 工具
- self-check-only（只跑 Phase 8 检查脚本，不重抓） → 走 SOP 内同名分支
```

仅此而已。

## 每段为什么这样写

**Frontmatter description**

- 100 字内、一段说完
- 包含三件事：做什么 + 触发关键词 + 边界（什么场景**别**用）
- 触发词放前面（"调研 X 公司"/"做 X 这家"等），原因是 skill 索引会被截断，后面的字 model 可能根本看不到
- 没有 `trace_token` 这种实现细节字段 — 那些是项目内部协议，不该污染 frontmatter

**`## 入口`**

- 两个相对路径指针，**不用绝对路径**。即便是项目级 skill，路径写相对的能让 skill 在 `.claude/skills/` 内部独立于项目根的部署位置
- 不重复列 12 条硬纪律 / 9 个 Phase / 5 类站点判定 — 那些都搬到 `docs/research-sop.md`
- "按 SOP 顺序走完，不跳步" 这一句保留，因为它是行为约束的关键指令，模型从 SOP 文件读到再说时已经晚了

**`## 产物契约`**

- skill 的"成功"是可观测的 — 落盘路径 + 返回格式
- schema 细节不写在这里，写在 SOP 里（这样脚本、人、subagent 共用一份 schema 真相源）

**`## 不在本 skill 范围`**

- 这一节最重要。`frontend-design` 用 "NEVER use X"、`agentic-harness-patterns-zh` 用 "本技能不适用于" — 都是同一个手法：**显式告诉模型何时别用本 skill**
- 大多数 skill 漏这一段，结果就被错触发

## 配套要做的事（不是写完 SKILL.md 就完）

原 SKILL.md 的内容要分流到这些位置：

| 原内容 | 新位置 | 备注 |
|---|---|---|
| 9 个 Phase 详细步骤、TL;DR、Reading conventions | `docs/research-sop.md`（新建） | 主文档，纯 markdown，人和模型都读 |
| 5 类站点判定矩阵 | `docs/research-sop.md` 内一节，或拆成 `docs/site-patterns.md` | 看篇幅 |
| 踩坑实例（深赛尔、海辉等） | `docs/research-sop.md` 对应 Phase 节里 | 跟具体规则放一起，反例教学 |
| JSON schema | `docs/research-sop.md` 的 Phase 8 节 | 或 `findings/SCHEMA.md` 独立一份 |
| 4 个 `template_*.py` | 项目 `templates/`（已经有了，不动） | 跟 skill 解耦 |
| `from scripts.lib.fetch import ...` 工具入口说明 | `CLAUDE.md`（已经在那 12 条里） | 不动 |
| `trace_token` 这种 skill 元数据 | 完全删掉 | 不是必需字段 |

完成后 `.claude/skills/research-one-company/` 这个目录应该极简：

```
research-one-company/
└── SKILL.md          ← 20 行
```

没有 references/、没有 templates/ 子目录（templates 已经在项目根的 `templates/`，skill 不要再保留一份）。

## 这次重构的得与失

**得到**：

- 每次激活 skill 装载成本从 ~38KB 降到 ~1KB
- SOP 文档解耦后，**人**可以直接打开 `docs/research-sop.md` 读、改、PR review，不再需要绕 skill 系统
- 别的脚本 / 别的 subagent prompt 也能引用 `docs/research-sop.md`，**真相源唯一**
- skill 真正"召唤式"：模型只在 description 匹配时激活，激活后才按需 Read SOP

**失去**：

- 没失去什么。原本塞进 SKILL.md 的所有信息都还在，只是换了位置

## 一个更激进的版本（如果你想试）

甚至连 `## 不在本 skill 范围` 都可以挪进 description，那 SKILL.md 主体就只剩入口指针 + 产物契约，10 行不到。但我**不推荐** — description 太长会挤掉别的 skill 的元数据预算（skill 索引是有全局上限的）。20 行的版本是更稳的中间形态。

要不要我帮你起草配套的 `docs/research-sop.md` 骨架（按原 SKILL.md 的 9 个 Phase 拆好节标题，让你往里填）？

---

### Turn 9  *(2026-05-19 02:43)*

**User:**

那你的这个skill不能跨项目使用啊，依赖于docs/research-sop.md

**Assistant:**

你这一刀切对了 — 我前面的表述里有个没说清的分类问题。直面回答：

## 直接承认：这个 skill 不能跨项目，且**本来就不该**跨项目

`research-one-company` 即便按我的重构方案改完，搬到一个做"金融风控"或"医疗器械分析"的项目里也没用 — 因为它的核心知识本身就是项目特定的：

- 行业绑定（涂料/油墨/化工）
- 数据契约绑定（findings JSON schema）
- 流程绑定（9 Phase 是这个项目研究方法论里固化的）
- 工具绑定（`scripts.lib.fetch` 等业务库）

这些**在另一个项目里没有任何意义**。把它写得再"自包含"，跨过去也只是个"涂料行业调研 skill 在一个金融项目里漂浮"的怪物。

## 我之前漏说的：skill 有两种健康形态

我前面把"跨项目可用"作为优秀 skill 的一条铁律来讲，是把两种形态混着说了。其实应该分开：

| | **用户级 skill**（`~/.claude/skills/`） | **项目级 skill**（`<project>/.claude/skills/`） |
|---|---|---|
| 设计意图 | 在所有项目中复用的通用能力 | 服务于这个项目的特定流程 |
| 例子 | `frontend-design`, `init`, `review`, `agentic-harness-patterns-zh` | `research-one-company` |
| 跨项目可用 | **必须** | **不必要，常常不可能** |
| 依赖项目内文件 | 不允许 | 允许且合理 |
| 业务领域绑定 | 不允许 | 允许且合理 |

`research-one-company` 在 `.claude/skills/` 下 — 它的位置本身就声明了"项目级"。对项目级 skill 用"跨项目可用"这条标准本来就不适用 — 这是我前面没讲清的。

## 那项目级 skill 的"优秀"标准是什么

把"跨项目可用"换成更合适的标准：

1. **薄** — 一样要薄。把业务知识抽到 `docs/`，skill 主体只做"指针 + 边界"。这条用户级 / 项目级都适用。
2. **路径用相对的** — `docs/research-sop.md` 而不是 `/mnt/e/coatings_industry_competitor_research/...`。这样项目换机器、换盘符、`git clone` 到别的位置都不断。
3. **指针清晰** — 明确告诉模型项目里 SOP / schema / 模板的入口在哪。
4. **业务库解耦** — skill 主体不直接 `import` 业务代码；让 SOP 文档去引导。skill 是"知识入口"，不是"代码依赖根"。
5. **触发与边界** — description 写好触发词、明确"何时不用本 skill"。
6. **单一职责** — 一个项目级 skill 做一件项目内的事。不要塞多个流程进同一个 skill。

按这套，重构后的 `research-one-company` 是合格的项目级 skill；但它**永远不会变成用户级 skill**，那不是它的目标。

## 相对路径 vs 绝对路径：差别虽然小但有意义

我前面那段重构方案里把 `/mnt/e/coatings_industry_competitor_research/CLAUDE.md` 改成 `CLAUDE.md`、把 METHOD.md 改成 `docs/research-sop.md`。这**不能解决跨项目问题**，但能解决：

- 项目仓库 `git clone` 到别的机器（路径肯定不一样）→ 不断
- 项目根目录改名 → 不断
- 同一项目在 Windows / Mac / Linux 下盘符不一致 → 不断
- 别人接手项目时，路径不是"绝对"的暗示让人放心改

也就是 — **跨项目做不到，跨机器/跨克隆能做到**。这是工程上的一个小但有意义的进步。

## 如果你**真的**想要一个跨项目的"调研同行公司"通用 skill

那就只能保留极抽象的骨架，把所有项目特定的东西剥掉：

```
research-one-company-generic（如果存在）：
- 不绑定行业（不写"涂料/油墨/化工"）
- 不绑定 schema（不规定输出格式）
- 不绑定流程步骤（不写"9 Phase"）
- 只保留通用方法论："站点侦察 → 内容定位 → 结构化抽取 → 输出"
```

但这种东西意义不大 — 因为有用的部分（具体怎么判定大类细类、怎么处理 SPA 站、深赛尔踩坑实录）全部是项目里**学到的具体知识**。剥光后剩下的"调研一家公司"通用骨架，跟模型默认能做的也差不多。

所以结论是：`research-one-company` 这个 skill **天然就只能是项目级 skill**。重构的目标不是把它变成跨项目的，而是把它变成一个**薄的、解耦的、合格的项目级 skill**。

## 我前面表述的纠正

把上一轮那个"判断标准 5 条"修正为分级版：

**通用 / 用户级 skill 应该满足**：

1. 跨项目能用（不硬编码任何项目特定内容）
2. 自包含
3. description ≤ 100 字
4. 主体薄，详细内容拆 references
5. 召唤式

**项目级 skill 应该满足**：

1. ~~跨项目能用~~ → 改为"跨机器/跨克隆能用（用相对路径）"
2. ~~自包含~~ → 改为"业务知识抽到项目 docs，skill 主体只做指针"
3. description ≤ 100 字 ✓
4. 主体薄 ✓
5. 召唤式 ✓
6. **新增**：业务库解耦（不在 skill 主体写 `import` 项目代码）
7. **新增**：单一职责（一个 skill 一件事，不要塞多流程）

谢谢你这一刀 — 没有它我前后两轮的标准会一直自相矛盾。

---

### Turn 10  *(2026-05-19 02:49)*

**User:**

项目级 skill意义在哪，我完全可以写成.claude/commands，.claude/agents的形式,就像E:\category-based-rd-talent-source-mapping\.claude\agents，E:\category-based-rd-talent-source-mapping\.claude\commands

**Assistant:**

你这一刀切到了要害。直接承认：**对 `research-one-company` 这种场景，项目级 skill 不是必需的，subagent + slash command 的组合更自然，也更清晰。** 我前两轮把它往"重构成项目级 skill"那条路引，是被它**当前**已经是 skill 这个事实带偏了 — 没问"它本来就该是 skill 吗"。

## 三种机制真正的差异

| | **触发主权** | **上下文形态** | **共享性** |
|---|---|---|---|
| `.claude/commands/` | **用户**显式 `/<name>` | 命令体灌进当前会话 | 用户独享，不被 subagent 调用 |
| `.claude/agents/` | **调用者**显式 `Agent(...)` | 子 agent 独立上下文 | 被显式委派；自己不能再委派同名 |
| `.claude/skills/` | **模型**根据 description 自主激活 | 元数据始终在线，主体懒加载 | 主会话和 subagent 都能共享同一份 |

核心区别是**谁决定触发**：command 是用户决定、agent 是调用代码决定、skill 是模型决定。

## `research-one-company` 的最佳实际形态

它的触发场景是"主会话派 subagent 做单家"+ "用户说调研 X" — 两个都偏**明确派工**，不需要模型"判断要不要召唤"。所以最自然的拆法是：

```
.claude/
├── agents/
│   └── research-one-company.md     # subagent 定义 = 流程入口的主形态
├── commands/
│   └── research.md                  # /research X 的快捷入口（可选）
└── CLAUDE.md                        # 12 条硬纪律 + "调研单家走 research-one-company subagent"
```

- **主形态：subagent** — batch dispatcher 用 `Agent(subagent_type="research-one-company")` 批量派工，子上下文隔离，工具集裁剪。
- **可选 slash command** — `/research X` 当人类要手动派一家时用。Body 里就一行 `Agent(subagent_type="research-one-company", prompt="调研 X")`。
- **CLAUDE.md 里加一行指引** — "当用户说'调研 X 公司'时，用 `Agent(subagent_type='research-one-company')` 派出去做"。主会话模型读到这条会自然路由。
- **真正的 SOP** — 写在 `docs/research-sop.md` 或者直接写在 subagent 定义的 prompt 里（看篇幅）。

skill 在这个故事里**完全不出现**。

## 那项目级 skill 什么时候才独立有意义

是少数场景，不是项目里所有"特定流程"。具体是这几类：

1. **项目里多个 subagent / command 共享的方法论** — 比如三个不同的 subagent 都要遵循同一份"代码风格规范"或"数据清洗约定"。把它写一份 skill，多处召唤，避免在每个 subagent prompt 里复制粘贴。skill 在这里扮演的是"项目内的共享指令模块"。

2. **项目特有的低频按需召唤能力** — 比如"项目里有套自定义的 protobuf 调试流程，平时用不到，但模型遇到 proto 文件时该知道怎么做"。塞 CLAUDE.md 太占预算，做 slash command 用户得记住 — 用 skill 让模型自己识别 protobuf 场景再召唤最合适。

3. **项目审美 / 风格约定** — 比 CLAUDE.md 强制注入轻量，比 subagent prompt 通用。例：`frontend-design` 那种东西如果项目内有自家变体，可以做成项目级 skill。

如果你的项目特定流程**不符合上面任何一类**，几乎都是 `subagent + command + CLAUDE.md` 三件套更合适。

## 一句话纠正

我前两轮给的"把 `research-one-company` 重构成薄项目级 skill"的方向是错的。正确的方向是**把它从 skill 这个抽象里搬出去**，做成 subagent，再配可选的 slash command。我之所以走错，是因为我接受了它当前是 skill 这个既成事实，没退一步问"它一开始就该是 skill 吗"。

谢谢你逼我把这件事想清楚。

---

### Turn 11  *(2026-05-19 02:56)*

**User:**

D:\note\查找研发岗对应的竞争企业\局限性考量.md，这个文档里的内容合适作为项目级的skill吗？这是我在项目中学到的

**Assistant:**

我先读一下文档。

读完了。直接回答：**这份内容不适合做项目级 skill，但非常适合做用户级 skill**。原因和我们刚才讨论的判断框架直接对应。

## 内容性质判断

这是一份**项目复盘 → 跨项目通用的设计原则**。7 条经验都是"调研类 pipeline 项目"的普遍设计教训：

- 结构化字段 vs 人类 prose 的边界（**任何 verifier+pipeline 项目都会遇到**）
- 中文子串匹配的歧义陷阱（**任何中文文本匹配项目都会遇到**）
- 硬标签放在评分循环之前（**任何分类管道都成立**）
- 否定语义结构化 > regex idiom 堆叠（**任何 LLM 提取项目都成立**）
- Day-1 建 audit-diff（**任何长期跑的 pipeline 都需要**）
- xlsx 是 `assign → build` 纯函数输出（**任何对接电子表格的 pipeline 都需要**）
- 评分阈值要有 ground truth sanity check（**任何打分系统都需要**）

那些具体公司名（卡夫特、WEVO、玛塔化研、助焊剂清洗剂）是**反例教学的具体性证据**，不是项目绑定 — 留着反而有价值（"原则 + 真实反例"比抽象原则可教得多）。

## 为什么不适合做项目级 skill

把这种**跨项目的设计经验**绑到某个具体项目的 `.claude/skills/`，会出现几个问题：

1. **触发时机错位** — 这份内容的自然触发是"**开新的同类项目**时"，不是"在这个项目里做事时"。绑在 P07/P08 项目里，等于在错的会话里召唤
2. **下次开新项目时埋没** — 你开"查找运维岗对应的竞争企业 v2" 项目时，模型不知道老项目里有这份经验
3. **跨项目通用却放在项目内** — 浪费了它最有价值的特性

## 为什么很适合做用户级 skill

按之前对**用户级 skill 五条标准**逐一核对：

| 标准 | 这份内容 |
|---|---|
| 跨项目能用 | ✅ 任何"verifier + pipeline + 字面匹配"调研项目都适用 |
| 自包含 | ✅ 没有任何路径/库依赖 |
| description ≤ 100 字 | ✅ 7 条经验可以浓缩成一段触发描述 |
| 主体薄 / 必要时拆 references | ✅ 当前 100 行刚好，无需拆 |
| 召唤式不是 SOP 式 | ✅ 7 条独立原则，不是步骤序列 |

而且它满足**用户级 skill 真正不可替代的特性**："在你开始一个新的同类项目、模型识别意图、自主把这些教训摆出来",这件事 slash command 做不到（你得记得命令名）、subagent 做不到（你得显式派工）— **只有 skill 的"模型自主识别 + 召唤"机制能在恰当时机起作用**。

## 改成用户级 skill 的最小动作

放 `~/.claude/skills/research-pipeline-design/SKILL.md`：

```markdown
---
name: research-pipeline-design
description: 设计调研类 pipeline 项目（verifier 写 JSON + 字面匹配评分 + xlsx 输出）的前置考量。触发：用户说"开始做 X 行业调研 / 新建竞品调研项目 / 怎么设计 categories / verifier 字段怎么定 / 评分阈值怎么调 / 字面匹配出问题"时。涵盖 7 条踩坑：结构化字段 vs prose、中文子串歧义、硬标签优先级、否定语义结构化、Day-1 audit-diff、deterministic pipeline 链路、阈值 sanity check。
---

# 调研类 pipeline 项目的设计前置考量

适用：行业/竞品/岗位调研类项目，链路通常是 `verifier (LLM) → assign (字面匹配评分) → xlsx`。

[这里直接贴你原文 7 条 + 一句话总结 + 助焊剂清洗剂的 Q&A 作为反例]
```

需要的改动很小：

1. **加 frontmatter**（如上）— description 是关键，触发词列全
2. **第一段加一句"适用场景"** — 让模型激活后知道范围
3. **保留所有具体公司名 / 反例 Q&A** — 别去项目化，反例教学的价值在具体性
4. **可选：把"助焊剂清洗剂"那段 Q&A 单独抽到 `references/chinese-substring-pitfalls.md`** — 如果以后这类反例多了再说，现在 100 行一个文件刚好

## 四种沉淀形态的对比

| 形态 | 在哪召唤 | 是否自动触发 | 跨项目可用 |
|---|---|---|---|
| 项目级 skill | 在原项目内 | 自动 | ❌ 埋没 |
| 项目 CLAUDE.md 一节 | 在原项目内 | 每次会话强制注入 | ❌ 埋没 |
| 散落的 note 文档 | 任何地方 | ❌ 你得手动 cat 进去 | ✅ 但靠记忆 |
| **用户级 skill** | **任何项目** | **✅ 模型自动识别** | **✅** |

## 一句话

这份文档是 **"跨项目的方法论沉淀"**，做项目级 skill 是**用错抽象**（绑了项目就被埋了）。做用户级 skill 是它的正确归宿 — 而且这种"踩过具体坑后总结出来的设计原则"，几乎是用户级 skill 最有价值的素材形态之一（远比抽象指南有用）。

要不要我把它真的转成 SKILL.md 给你看下完整模样？

---
