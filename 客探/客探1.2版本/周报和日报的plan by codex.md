  # Dify 周报/月报工作流改造方案

  ## Summary

  目标是把现有工作流改造成一个“单次运行只生成一个业务员报告”的报表工作流，支持：

  - 周报：默认统计“上一个完整自然周”，按周一到周日
  - 月报：默认统计“上一个自然月”
  - 时间基准：手动触发，未传日期时默认取运行当天
  - 时区：统一按 Asia/Shanghai

  核心设计是把“时间计算”前置到一个代码节点里，不让 LLM 自己猜日期；知识检索也不靠语义猜时间，而是靠结构化 metadata 精
  准过滤。

  ## Key Changes

  ### 1. 时间处理放到代码节点，不放到提示词里

  工作流输入定义为：

  - report_type: weekly | monthly
  - salesperson_id: 业务员唯一标识
  - base_date: 可选，格式 YYYY-MM-DD；为空时取 sys.timestamp

  代码节点负责输出这些标准变量：

  - effective_date
  - today_weekday
  - period_start
  - period_end
  - period_label
  - period_key
  - timezone

  计算规则固定为：

  - 时区先统一转成 Asia/Shanghai
  - today_weekday 用 ISO 周制，周一=1，周日=7
  - 周报：
      - 先算“本周周一”
      - 再回退 7 天得到“上一个完整自然周”的周一
      - period_end = period_start + 6 天
  - 月报：
      - 先取“本月 1 号”
      - period_end = 本月 1 号 - 1 天
      - period_start = period_end 所在月的 1 号

  建议 period_key 统一为：

  - 周报：YYYY-Www
  - 月报：YYYY-MM

  ### 2. 知识库结构需要调整，不建议“一个业务员一个文档 + 多个时间块”

  你原计划是“两个知识库，每个业务员对应一个文档，文档里存一周/一月的文档块”。这在 Dify 里不够稳，因为知识检索的 metadata 过滤是按“文档”限制，不是按 chunk 做时间过滤。

  建议改成：

  - 保留两个知识库：
      - sales-weekly-kb
      - sales-monthly-kb
  - 但文档粒度改为“一个业务员 + 一个统计周期 = 一个文档”

  也就是：

  - 周报库：业务员A + 2026-W12 是一个文档
  - 月报库：业务员A + 2026-02 是一个文档

  每个文档挂这些 metadata：

  - salesperson_id
  - salesperson_name
  - report_type
  - period_key
  - period_start
  - period_end
  - 可选：team、region

  这样知识检索节点就能直接按：

  - salesperson_id = 输入业务员
  - period_key = 代码节点计算结果
    做精确过滤，不需要靠相似度猜“这是不是上周数据”。

  ### 3. 工作流节点建议

  建议工作流结构：

  1. Start
      - 输入 report_type、salesperson_id、base_date
  2. Code: Resolve Period
      - 统一时区
      - 计算 period_start/end/key
      - 输出给后续节点
  3. If / Branch
      - report_type == weekly 走周报知识库
      - report_type == monthly 走月报知识库
  4. Knowledge Retrieval
      - 周报分支检索 sales-weekly-kb
      - 月报分支检索 sales-monthly-kb
      - metadata filter 固定使用：
          - salesperson_id
          - period_key
  5. LLM
      - 输入：
          - 检索结果
          - 业务员信息
          - period_label
      - 输出标准化报表
      - 提示词里明确要求：
          - 只基于检索内容生成
          - 不补写不存在的数据
          - 标题必须带区间，例如 2026-03-16 至 2026-03-22 周报
  6. 可选 Template / Answer
      - 输出 Markdown 或企业微信/钉钉可直接发送的格式

  ### 4. 提示词职责收窄

  提示词只负责“组织表达”，不负责“判断日期”。

  提示词中只引用代码节点产出的变量，例如：

  - period_label
  - period_start
  - period_end

  不要写这类开放式指令：

  - “请判断今天是周几并生成上周周报”
  - “请根据当前时间决定取哪段数据”

  这类逻辑必须在代码节点里做掉，否则会出现周界、月界、时区错判。

  ## Public Interfaces / Types

  建议固定这几个输入输出接口，后续扩展最省事。

  输入：

  - report_type: weekly | monthly
  - salesperson_id: string
  - base_date?: string(YYYY-MM-DD)

  代码节点输出：

  - effective_date: string
  - today_weekday: number
  - period_start: string
  - period_end: string
  - period_label: string
  - period_key: string
  - timezone: Asia/Shanghai

  知识库文档 metadata：

  - salesperson_id: string
  - salesperson_name: string
  - report_type: weekly | monthly
  - period_key: string
  - period_start: string
  - period_end: string

  ## Test Plan

  必须覆盖这些场景：

  1. 周报日期边界

  - base_date=2026-03-24，应返回 2026-03-16 到 2026-03-22 之外？先校正：按自然周周一到周日，上一个完整周应为 2026-03-
    16 到 2026-03-22 仅当本周周一是 2026-03-23。实现时要以代码节点实算为准。
  - base_date 落在周一、周日时都不能错位

  2. 月报日期边界

  - base_date=2026-03-24，应返回 2026-02-01 到 2026-02-28
  - 跨年测试：base_date=2026-01-05，应返回 2025-12-01 到 2025-12-31

  3. 时区测试
  - 不允许因服务器时区不同导致 period_key 偏移

  4. 检索准确性

  - 不同业务员同一周期文档不能串档
  - metadata 缺失时应返回空结果或明确报错，不允许退化成全库语义检索

  5. 空数据处理

  - 某周期无文档时，输出“该周期暂无可用数据”，不要编造内容

  ## Assumptions

  - 当前方案按你已确认的规则设计：
      - 周报：上一个完整自然周
      - 月报：上一个自然月
      - 手动触发
      - 默认今天为基准
      - 时区为 Asia/Shanghai
      - 单次只生成一个业务员的报告
  - 为了让 Dify 的 metadata 过滤真正可用，默认采用“一个业务员 + 一个周期 = 一个文档”，而不是“一个业务员一个大文档”
  - 时间感知实现依赖 Dify 的调度/系统时间变量、模板时间格式化能力、知识检索 metadata 过滤能力，参考：
      - https://docs.dify.ai/zh/use-dify/nodes/trigger/schedule-trigger
      - https://docs.dify.ai/en/use-dify/nodes/knowledge-retrieval
      - https://docs.dify.ai/en/guides/knowledge-base/metadata
