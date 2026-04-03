  skill 的本质

  - 可以把 skill 理解成“外置的能力包”，确实属于把一部分模型上下文放到外部。
  - 但它不只是“给模型读的长文档”。
  - 一个完整 skill 通常包含 4 类东西：
      - 触发条件：什么时候该用它
      - 核心指令：用什么流程做事
      - 可选参考资料：按需加载，不一次全塞给模型
      - 可选工具绑定：脚本、API、函数，不只是文字

  所以它更接近“可检索的程序化能力单元”，不是普通知识库文档。

  它和超长上下文的区别

  - 超长上下文：把很多材料一次性塞给模型，让模型自己在里面找重点。
  - skill：先决定“要不要用这个能力”，再只加载相关说明，必要时再加载子文档或调用工具。
  - 超长上下文主要解决“信息放得下”。
  - skill 主要解决“能力如何被稳定触发和复用”。

  所以你说“把重要状态转移到外部”，这个说法只对一半。

  - skill 更适合放“稳定的 procedural knowledge”，也就是流程、规则、工具使用方法。
  - 不适合放“运行时状态”，比如当前用户缓存、这轮对话变量、任务执行中间结果。
  - 运行时状态应该放 memory/store/db/cache，不该放 skill。

  自己搭 agent 时，skill 怎么用
  你可以按这个架构做：

  1. 建一个 skill registry

  - 每个 skill 一条记录：
      - name
      - description
      - triggers/examples
      - skill_md
      - references
      - tools

  2. 先做 skill routing

  - 用户请求进来后，先判断要不要启用 skill。
  - 路由方式可以是：
      - 规则匹配
      - embedding 检索
      - 小模型分类
      - 主模型先做 planner/router

  3. 只注入最小 skill 内容

  - 不要一上来把整个 skill 包全塞进去。
  - 先注入：
      - skill 名称
      - description
      - SKILL.md 里的核心 workflow
  - 如果任务深入，再按需加载 references/ 某一段。

  4. 把 tools 和 skill 一起挂载

  - 如果 skill 依赖脚本/API，不要只把文档给模型。
  - 要把对应工具也注册进 agent。
  - 模型看到 skill 指令后，才能真的执行对应动作。

  5. 把状态和 skill 分开

  - skill 存“怎么做”
  - memory/store 存“当前做到哪了”
  - knowledge base 存“业务事实/资料”
  - 这是最重要的边界

  一个实用心智模型

  - Skill = 能力模板
  - Memory = 运行时状态
  - Knowledge/RAG = 外部事实
  - Tools = 可执行动作

  四者不要混。

  最常见的错误

  - 把 skill 写成大而全说明书，导致每次都塞很多无关上下文。
  - 把用户会话状态写进 skill 文档。
  - 只有文档，没有工具接入，模型知道怎么做但做不了。
  - 没有路由层，所有请求都加载所有 skills，最后退化成“超长上下文”。

  你如果要落地，推荐最小实现

  - 先做一个 skills/ 目录
  - 每个 skill 一个 skill.json 或 SKILL.md
  - 加一个 router：
      - 输入用户请求
      - 选出 top-1/top-2 skill
  - agent prompt 里只注入选中的 skill 摘要
  - references 按需二次检索
  - tools 按 skill 绑定暴露
