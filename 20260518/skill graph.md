# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `e3770e24-9fb8-496e-8183-43eeb05020a5`
- **范围**: Turn 1-5
- **导出时间**: 2026-05-19 09:36:14

---

### Turn 1  *(2026-05-19 00:45)*

**User:**

skill graph 你知道是什么？

**Assistant:**

"skill graph" 不是一个有单一标准定义的术语，需要看你说的是哪个语境：

**Agent / LLM harness 语境下**（最可能）：指 skills（能力包）之间的关系图。一个 skill 通过触发词、`Skill` 工具调用、或文档里的引用链接（如 Anthropic Skills 里的 `[[name]]`），指向另一个 skill，整体构成一个有向图。设计 agent 时关心的问题包括：
- 入口点（哪个 skill 是 user-invocable）
- 调度（谁触发谁，是模型自动召唤还是显式 `Skill()`）
- 循环 / 重入风险（A 调 B，B 又能调 A）
- 上下文预算（每跳一次 skill 都要装载 SKILL.md，会膨胀）

**人才 / 学习语境**：skill graph 指技能图谱 / 能力图，把岗位、知识点、技能拆成节点和先修关系，常见于学习路径推荐、HR 建模、招聘画像。

**知识图谱语境**：偶尔有人把 KG 里专门表达 "person→skill→domain" 的子图叫 skill graph。

你想问的是哪个？是 Claude Code / agent harness 里 skills 互相调用形成的图，还是别的？

---

### Turn 2  *(2026-05-19 00:52)*

**User:**

是 Claude Code / agent harness 里 skills 互相调用形成的图

**Assistant:**

在 Claude Code / agentic harness 这个语境下，"skill graph" 不是 Anthropic 的官方术语，但概念上确实存在 — 一组 skill 通过几种不同性质的"边"互相连接成图。把它讲清楚需要先分清节点、边和调度方式。

## 节点

一个 skill = 一个 SKILL.md（含 frontmatter 元数据 + 主体指令）+ 一个 references 文件树。完整能力是惰性展开的，不是单文件。

## 边的四种类型

| 边类型 | 显式 / 隐式 | 谁决定遍历 |
|---|---|---|
| **触发边** — 模型读用户输入后根据 `description` / 触发词匹配 | 隐式 | LLM 运行时概率选择 |
| **显式调用边** — SKILL.md 主体里指示模型 `Skill("other-name")` | 半显式 | SKILL.md 文本指令 + LLM 跟随 |
| **资源引用边** — SKILL.md 用相对路径或 `[[name]]` 引用 references/*.md | 显式 | 模型按需 Read |
| **委派边** — skill 内部触发 `Agent(subagent_type=...)`，子 agent 又有自己的 skill 池 | 显式 | 当前 agent 决定 |

这是与传统软件里"模块依赖图"的根本差别 — skill graph 大部分边是**概率性的**，靠 LLM 读 description 在运行时选边，不是静态声明的。

## 调度：图不是被遍历的，是被"召唤"的

- 启动时，所有 skill 的 `name + description + 触发词`被拼接进系统上下文，构成一份**紧凑索引**（≈ 上下文窗口的 1%）。这是模型能"看到的图"。
- 模型只看到节点元数据，看不到节点内容，更看不到节点之间的引用边 — 直到某个 skill 被激活、它的 SKILL.md 主体被装载，下游引用才暴露出来。
- 所以 skill graph 是**惰性展开**的，不是预先全部装载的 DAG。

这与上下文工程里的"三级渐进披露"一一对应：
1. 元数据（始终在线，廉价）
2. SKILL.md 主体（激活时载入）
3. references 文件（按需 Read）

## 与 fork / 委派的关系

- **内联激活** — skill 在父 agent 上下文里执行，图遍历共享同一个 token 预算。
- **Fork 激活** — skill 触发子 agent，子 agent 继承全部上下文但**只能下一层**（递归 fork 被运行时阻止，防止上下文指数爆炸）。
- **Coordinator 委派** — 子 agent 零上下文继承，只看到显式 prompt 和它自己的 skill 池。这相当于"另开一张子图，从根重新出发"。

## 工程上的关键约束（容易踩的坑）

1. **触发词截断**：description 列表是拼接的，每条有硬字符上限，超出部分被截掉。所以要把最具区分度的触发词写在前面，否则你的 skill 在图里实际"不可见"。
2. **规范路径去重**：同一 skill 出现在多个源目录（内置 / 用户 / 插件）时，运行时按规范路径去重，不会重复出现两次。
3. **没有循环检测**：skill graph 没有静态分析，A 调 B、B 又指示调 A 完全可能。靠 SKILL.md 作者自律 + 上下文预算自然兜底（token 用完就停）。
4. **隔离 vs 共享**：内联激活共享 token，重型 skill（生成大输出）会污染父级。这种 skill 适合改成"在 SKILL.md 里指示模型用 Agent fork 执行"，让它跑在子 agent 的预算里。

## 实际工程价值

理解 skill graph 主要影响这几件事：
- **写 description 的策略**：你在影响图的"触发边权重"
- **决定哪些 skill 内联、哪些 fork**：决定边的"上下文成本"
- **设计 references 文件层次**：决定节点内部的渐进披露 DAG
- **避免链路过深**：每多跳一次激活，就多一份 SKILL.md 装入上下文

如果想再往下挖，可以选这几个方向之一：(a) 怎么写 description 才能在图里被正确"激活"，(b) skill graph 与 multi-agent coordinator 的边界设计，(c) 大型 skill 该怎么拆 references 来控制激活成本。

---

### Turn 3  *(2026-05-19 00:57)*

**User:**

官方没有skill graph的概念吗？为什么给人感觉只要是复杂的skill就得用skill graph似的？

**Assistant:**

你这个质疑是对的，我上一轮的框架把概念推得过远了。

## 先回答：官方没有这个概念

Anthropic 官方文档里**没有** "skill graph" 这个术语。官方反复出现的核心概念其实只有几个：

- **SKILL.md + frontmatter**：skill 的容器格式
- **Progressive disclosure（渐进披露）**：元数据 → 主体 → references 的三级懒加载
- **Triggering / discovery**：基于 description 让模型自主激活
- **Composition / bundling**：skill 之间可以组合，可以打包

"Skill graph" 是社区 / 工程师们做系统分析时**借用的图论比喻**，不是 Anthropic 推荐的设计语言。我上一轮直接用"四种边"那种讲法，把比喻拔高成了主框架 — 这是我的表述问题。

## 为什么会有"复杂 = graph"的错觉

部分是我表述的锅，部分是 agentic harness 圈讨论时的通病。真实情况是：

- **绝大多数 skill 是扁平的单节点**：用户/模型触发一次，skill 跑完，结束。没有出边。
- **它们之间的"关系"绝大多数是隐式的、概率性的** — 模型基于上下文选一个激活 — 这种"关系"在工程上**不需要被建模成图**，因为没有任何代码去遍历它。
- 真正显式的连接只有两种少见场景：(a) 一个 skill 在 SKILL.md 里写"接下来用 Agent 委派到 X"，(b) reference 文件之间互相链接。这两种都是局部的，不构成全局图。

把它叫"图"，几乎只在**做 harness 整体可观测性分析**时有用 —— 例如统计哪个 skill 被引用最多、哪些 description 重叠冲突。**对单个 skill 的设计完全无关**。

## 真正该用的设计抽象

复杂 skill 的复杂性几乎全部应该通过这三种机制管理，跟"graph"无关：

1. **References 文件层次** — 把主 SKILL.md 写薄，把详细方案、踩坑、模式分到 references/*.md，让模型按需读。这是**树**，不是图，而且是单向的。
2. **委派给子 agent** — 当一个 skill 内部要做很重、上下文成本高的工作（深度调研、大型重构），让它指示模型 fork 一个子 agent 去做，结果回收成摘要。
3. **清晰的触发词** — 让 description 在元数据列表里能被正确召唤，比"skill 之间互相调用"重要 10 倍，因为前者决定 skill 能不能被用到。

如果你拿到一个"复杂 skill 需求"，**第一反应不应该是"画 graph"**，而是：

- 这件事能不能用一个 SKILL.md + 几个 references 文件搞定？
- 内部有没有重型子任务，需要让模型 fork 子 agent？
- description 写成什么样，模型会在该用的时候用、不该用的时候不用？

只有当你已经有几十个 skill、想分析整个仓库的健康度时，"graph"视角才有价值。设计单个 skill 时它是个干扰。

我上一轮把"边的四种类型"放在最前面讲，造成了误导。抱歉。

---

### Turn 4  *(2026-05-19 01:25)*

**User:**

references 文件是什么

**Assistant:**

最直观的办法是直接看一个真实例子。你现在这个会话刚加载的 `agentic-harness-patterns-zh` skill 自己就有 references 文件 — 我先把它的目录结构拉出来给你看。

## 结论先行

**references 文件 = SKILL.md 旁边一个 `references/` 目录里的附录 markdown 文件**。没有特殊格式、没有运行时魔法，本质就是普通 .md 文件。它存在的唯一目的是：让 SKILL.md 主体保持薄，把详细内容拆出去**按需读取**。

## 用刚加载的这个 skill 当例子

你看上面的目录结构：

```
agentic-harness-patterns-zh/
├── SKILL.md                    ← 15KB 主文档，激活时被装载
└── references/                 ← 9 个详细模式文件
    ├── memory-persistence-pattern.md       (12KB)
    ├── skill-runtime-pattern.md            (13KB)
    ├── permission-gate-pattern.md          (13KB)
    ├── tool-registry-pattern.md            (11KB)
    ├── agent-orchestration-pattern.md      (12KB)
    ├── context-engineering-pattern.md      (2KB，又指向 context-engineering/ 子目录)
    ├── hook-lifecycle-pattern.md           (13KB)
    ├── bootstrap-sequence-pattern.md       (12KB)
    ├── task-decomposition-pattern.md       (8KB)
    └── context-engineering/                ← 还能再分子目录
```

激活这个 skill 时，**只有 SKILL.md 那 15KB 被注入我的上下文**。references 里的 100KB+ 内容**一字未读**，直到 SKILL.md 主体里出现这种链接：

```markdown
**深入阅读：** [references/memory-persistence-pattern.md](references/memory-persistence-pattern.md)
```

模型看到这个链接，**当且仅当需要时**才调用 `Read` 工具去把那个文件读进上下文。

## 为什么不直接全部写进 SKILL.md

如果把 references 全合并进 SKILL.md，这个 skill 的元数据列表里就多一个 150KB+ 的怪物。Skills 系统启动时所有 skill 的描述都在系统提示里，激活后整个 SKILL.md 装载 — 这个 skill 每次激活就要烧几万 token，哪怕用户只想问关于"记忆"的一小段。

拆成 references 后：

| 阶段 | 装载内容 | token 成本 |
|---|---|---|
| 启动（永远在线） | 只有 frontmatter 元数据（几十字符） | 几乎零 |
| 激活 | SKILL.md 主体（15KB） | 约 4000 token |
| 深入某个话题 | 该话题的一个 reference 文件（10-13KB） | 约 3000-4000 token |

这就是上一轮我提到的 **progressive disclosure（渐进披露）** 的具体实现。

## 关键性质

1. **没有专门的"references 系统"** — 这只是约定俗成的目录名。叫别的也行（虽然 Anthropic 文档里推荐 references/），运行时不强制。
2. **链接是 markdown 相对路径** — 用 `[文字](references/xxx.md)`，模型把它当成"该文件存在、需要时去读"的提示。
3. **模型用 Read 工具读** — 不是自动加载，是 LLM 自己判断需要后发起的工具调用。
4. **目录可以嵌套** — 你看 context-engineering/ 是个子目录，里面还有 select-pattern.md、compress-pattern.md、isolate-pattern.md。这就是上一轮我说的"references 是树不是图"。
5. **没有循环引用风险** — 因为不是 import 关系，只是文档链接，模型读了就读了，不会无限递归。

## 一句话总结

References 就是把 SKILL.md 拆成"主册 + 附录"。主册讲完整框架（什么时候用、有哪些章节），附录讲单点细节。模型读完主册自己决定要不要翻附录。

---

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

---
