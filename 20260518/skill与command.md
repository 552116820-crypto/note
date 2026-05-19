# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `fbecfecf-f899-4a77-a897-8c0bdec8dc73`
- **范围**: Turn 12
- **导出时间**: 2026-05-18 18:07:19

---

### Turn 1  *(2026-05-18 10:00)*

**User:**

"自动化操作流"的skill有哪些好的？"E:\category-based-rd-talent-source-mapping\.claude\commands\run-position.md"这个command是不是可以改成自动化操作流的skill，但从设计哲学上commands设计是给项目特化，而skills设计给跨项目能力

**Assistant:**

先看完了 `run-position.md`，直接答。

## 好的"自动化操作流"skill 长什么样

判断标准只有三条：**一个动词**、**跨项目**、**自包含**（不依赖具体项目结构）。本会话能看到的内置 skill 里：

| 算好的                          | 原因                                       |
| ---------------------------- | ---------------------------------------- |
| `simplify`                   | 一个动作（重审 + 修），任何代码库都适用                    |
| `init`                       | 在任意 repo 生成 CLAUDE.md                    |
| `review` / `security-review` | PR 审查逻辑普适，不依赖具体业务                        |
| `fewer-permission-prompts`   | 扫历史 + 改 settings.json，每个用户都需要            |
| `update-config`              | 改 settings.json，schema 是 Claude Code 通用的 |
| `loop` / `schedule`          | 调度原语，本质就是给"任意 prompt"加时间维度               |

它们的共性：**输入是任意上下文，输出是行为变更**，没有项目专属知识。

反面例子（如果谁把这种东西写成 skill 就是滥用）：
- "跑 ACME 公司的部署流水线" — 太项目特化
- "做我每天的代码评审" — 个人 workflow，不是普遍能力
- "审 Foo 类的命名规则" — 团队 convention

## 你的 `/run-position` 该不该转 skill

**不该。** 你的直觉完全对，而且这文件本身就是反例样本。它"项目特化"特化到什么程度，列几条硬证据：

1. **硬编码路径骨架**：`config/categories.yaml`、`positions/P0X_*/`、`work/scratch/P0X/`、`docs/workflow.md`、`docs/conventions.md`、`docs/mcp-tools.md` — 换个项目这些路径全失效
2. **硬编码子 agent**：`exhibition-finder`、`exhibition-verifier`、`cross-batch-merger` — 这些 subagent_type 是这个项目的 `.claude/agents/` 里定义的
3. **硬编码 pipeline 模块**：`pipeline.scrape_exhibition` / `merge_batches` / `apply_decisions` / `assign` / `build_xlsx` / `url_check` — 全是项目 Python 包
4. **硬编码 MCP 选型**：jina / firecrawl 的优先级是你这个项目的爬取策略
5. **硬编码业务语义**：P01-P07、Group A/B/C/N、main_categories、reject_reason_tag、merge_suspects.tsv — 这是品类×公司图谱这个项目的领域语言，搬到别的项目一个词都不对

把它转成 skill 反而**有害**：
- 必须把所有项目知识打包进 skill 体内，膨胀
- 装到别的项目自动出现在 skill 列表里，触发了反而坑
- skill 设计意图是"任何会话都可能被激活"，你这个只在这个 repo 才有意义

`commands/` 就是给这种东西准备的——**项目内的、由用户显式 `/foo args` 触发的、绑定具体目录结构的 orchestrator**。它躺在 `.claude/commands/` 里跟着 repo 一起版本化，新人 clone 下来就能跑，这正是它的归属。

## 真要从中抽 skill，能抽什么

如果你跨项目反复写类似的多步 orchestrator（A 项目跑数据 pipeline，B 项目跑模型评测，结构都是"分步 + 用户闸门 + 后台 agent 扇出 + 重试"），那可以抽**一个抽象层**，但它**不是**操作型 skill，而是**参考型 skill**——叫类似"多阶段 orchestrator 设计模式"，内容是：

- 闸门：什么时候停下等用户、确认信息怎么打印
- 扇出：背景 agent 并发上限、batch 切分、retry 模式
- 幂等：靠产物文件存在性 skip、`--redo stepN` 怎么实现
- 错误升级：哪类错该中断、哪类该警告继续
- dry-run：outline 模式怎么设计

这个 skill 触发时给 Claude（或你）灌一份 checklist，用来**写下一个 `run-position` 式命令**。**生成器**而不是**执行器**。

但说实话——你已经写出一个相当成熟的 run-position.md 了，自己再写一份"怎么写 run-position"的 skill 收益边际很小，除非你确认未来还会写 ≥3 个类似复杂度的命令。先观察重复模式有没有真的出现，出现了再抽。**过早抽象比项目特化更坑**。

---
