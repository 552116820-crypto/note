❯ 为什么项目这套harness给到codex跑，效果会差很多

● 主要是这套 harness 把 Claude Code 的能力当成了"地基"在用，迁到 Codex 会一层一层塌：

  1. subagent + 背景任务并行模型
  "每家公司一个 subagent、N 个并行、靠 task-notification 收回 JSON" 这套是 Claude Code 的 Agent + TaskCreate + run_in_background 三件套深度耦合的产物。Codex CLI 没有等价的并行子代理 +
  通知回收机制，要么退化到串行（吞吐崩 10×），要么把 N 路 transcript 都灌进主上下文（信息洪流 + 漂移）。

  2. 多模态 Read 当 OCR 用
  项目大量靠 Read 直接读 PNG/JPG（武藏图标对位、东洋色材 portfolio-item、实创 cnscc 扫描 PDF 渲染图、道顿粉末详情页）。Claude 是原生 vision、Read 直接喂图给模型；Codex 走 GPT 链路，CLI 对图片的传递路径和
  prompt shape 不一样，型号 OCR 命中率会差一个量级。

  3. 严格指令跟随 + trace_token 校验
  SKILL.md Phase 0→9、JSON schema、skill_trace 这种"必须严格逐 phase 跑、返回特定 token"的契约式提示，Claude（尤其 Opus）守得住，GPT 系倾向于"语义完成"——跳 phase、提前总结、缩列表。trace_token 不匹配 →
  主会话按规则 retry/BLOCKED，体感上就是"Codex 跑出来错很多次还得返工"。

  4. 长 HTML 穷举 vs. 总结倾向
  拆型号表、分页审计、详情页全表 SKU 抽取，本质是"枯燥的穷举"。Claude 在这类任务上比 GPT 稳得多（GPT 会自作主张合并相似行 / 用"…等" / 跳过看起来重复的）。在 702 SKU 嘉宝莉、869 行 cnscc
  这种规模上，差距会被放大。

  5. memory / skill / hooks 这套上下文不会自动加载
  auto-memory（踩坑 hook）、.claude/skills/ 的 frontmatter 解析、CLAUDE.md 的自动注入，都是 Claude Code harness 提供的。Codex
  没这一层，必须手动复制粘贴或自己写加载器，少塞一个文件就缺一段关键经验（比如"详情页是图片要 OCR 兜底"这类反直觉知识）。

  6. MCP / 工具命名假设
  chrome-devtools MCP、browser-use MCP、WebFetch 在 Codex 那边即便 MCP 配上了，prompt 里写的 WebFetch/Read 字面也对不上，规则索引表里的"工具退路"全得改名。

  根因一句话：这套 harness 不是 LLM-agnostic 的"调研流程"，它是"基于 Claude Code 工具矩阵 + Claude 指令跟随特性 + Claude 多模态原生 Read"组装出的特定工作流。换底座必须重写 5
  条主会话职责的实现层，不只是换模型。
