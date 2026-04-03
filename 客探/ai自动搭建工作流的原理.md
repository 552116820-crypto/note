• 这次我用的是**“API 级编辑工作流图”**，不是纯手工点点点，也不是直接导入整份 DSL。
  这次实际做法
 
  1. 用你已登录的会话拿到 Dify 控制台权限（cookie + csrf）。
  2. 读取当前草稿：

  - GET /console/api/apps/{appId}/workflows/draft

  3. 在返回的 graph JSON 里直接增改节点和连线：     

  - 新增 trigger-schedule / code / http-request 节点
  - 设置 error_strategy: fail-branch
  - 补 value_selector（变量引用）
  - 增加/重连 edges

  4. 保存草稿：

  - POST /console/api/apps/{appId}/workflows/draft  
  - 带 hash 做并发保护（乐观锁）
  5. 发布：
 
  - 在 UI 里点“发布更新”（本质是调用 publish 接口）

  6. 校验：

  - 查触发器状态 GET /console/api/apps/{appId}/triggers
  - 触发运行和公开页实跑验证
 
  你看到的“节点已经出现、连线已生效”，本质是我直接改了草稿里的图结构并发布。
 
  ———

  和“模拟人类点击搭建”对比

  1. 模拟点击搭建（UI 自动化）
  - 优点：完全贴近人工操作
  - 缺点：慢、容易受页面加载/DOM变化影响、稳定性差一些
 
  2. 直接改草稿图（我这次用的）

  - 优点：快、可重复、结构化可校验、批量改动稳定
  - 缺点：需要懂 Dify 图结构和接口字段
 
  3. 直接导入 DSL（YAML）
 
  - 优点：适合“从零新建”或整套迁移
  - 缺点：对已有应用做增量改造时，容易整图覆盖；和线上当前草稿合并处理更复杂
 
  ———
 
  底层原理（Dify 工作流）
  Dify 编排页底层就是一个 DAG/流程图数据结构，核心字段：
  1. nodes[]
 
  - 每个节点有 id、data.type（如 http-request/code/llm/trigger-schedule）
  - 节点配置都在 data 里（URL、代码、提示词、重试、异常策略等）

  2. edges[]

  - 定义节点连接关系
  - sourceHandle 决定分支类型，例如：
      - source：正常输出
      - fail-branch：异常分支
 
  3. value_selector

  - 变量引用机制，格式类似 [上游node_id, output_key]
  - 相当于“把上游输出映射到当前节点输入参数”
 
  4. error_strategy
  - 例如 fail-branch，节点报错时走异常链，不直接让整个流程崩掉
 
  5. publish

 
  - 发布后才是线上运行版本
  - trigger（定时）也需要在发布版本上启用

 
  ———
 
 
  你问的关键点一句话总结
  这次是：UI 用于验证/发布，核心搭建是直接改 Dify 工作流草稿 JSON（API 方式），不是导入整份 DSL。
