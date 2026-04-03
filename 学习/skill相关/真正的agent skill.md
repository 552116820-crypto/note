• 我直接把它拆成两层：一层是你这个仓库里已经实现的“prompt skill”，另一层是更完整的“agent skill”。这样你能一眼看出现在做到哪、还差哪。

• 可以。先给你一个最实用的判断：

  你这个仓库现在实现的是 Prompt Skill，不是完整的 Agent Skill。

  两层架构

  用户请求
    |
    v
  [路由器]
    按 $skill / trigger / token overlap 选 skill
    |
    v
  [Prompt Skill 层]
    读取 SKILL.md / skill.json
    读取 runtime state
    选择 section / reference
    组装 system / developer / user prompt
    |
    v
  [大模型]
    基于注入的上下文做推理和回答

  上面这层，就是你仓库当前已经落地的部分。

  再往下扩一层，才是完整 agent：

  用户请求
    |
    v
  [路由器]
    |
    v
  [Agent Skill 层]
    约束执行步骤
    决定调用哪些工具
    读写长期/短期状态
    校验中间结果
    必要时重试/回滚/分支处理
    |
    v
  [Prompt Assembler]
    只负责把当前必要上下文喂给模型
    |
    v
  [大模型]

  两者的本质区别

  Prompt Skill：

  - 重点是“给模型喂什么上下文”。
  - skill 本身不执行动作，主要负责提示词组织。
  - 目标是减少上下文稀释、提高稳定性、让模型少靠临场猜测。

  Agent Skill：

  - 重点是“系统如何做事”。
  - skill 不只是文本说明，还会驱动工具、状态机、验证逻辑。
  - 模型只是其中一个推理部件，不是全部。

  你这个仓库目前属于哪一层

  从实现看，当前链路是：

  - SkillRegistry 发现 skill
  - SkillRouter 选 skill
  - PromptAssembler 把 skill 内容和 runtime state 拼成 prompt
  - FileRuntimeStore 把状态写到 .agent_runtime/state/

  对应代码：

  - router.py:22
  - skill_registry.py:52
  - prompt_assembler.py:8
  - runtime_store.py:9

  所以它现在更准确的定位是：

  “可路由的结构化提示词系统，外加轻量运行时状态持久化。”

  它解决了什么问题

  它确实在帮模型对抗一部分“上下文压缩/稀释”问题，具体体现在：

  - 不是把所有说明都塞给模型，而是只塞当前 skill 的相关 section。
  - 把运行时状态显式写进 prompt，而不是全靠模型自己记。
  - 把 skill 说明外置到文件，不必每次都在对话里重复展开。
  - 让 prompt 更稳定、可审计、可复用。

  所以你那句话可以改成更精确的版本：

  目前的 skill 主要还是一种结构化 prompt 注入机制，用外部文件和运行时状态来补足模型上下文，从而减少关键信息被稀释或遗忘的概率。

  但它还没做到的部分

  还不能算完整 agent skill，因为现在还缺这些核心能力：

  - 自动恢复旧 state 并续跑任务
  - 基于 skill 决定工具调用计划
  - 工具执行后的状态更新
  - 中间结果校验与失败重试
  - 多步工作流编排

  比如 load() 已经有了，但主流程还没真正接上续跑逻辑：

  - runtime_store.py:14
  - demo.py:61

  一句话区分

  - Prompt Skill：告诉模型“该怎么想”。
  - Agent Skill：规定系统“该怎么做”。
