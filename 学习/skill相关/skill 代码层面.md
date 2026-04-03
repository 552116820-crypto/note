结合[agent_runtime\test.py](E:\KeTan_Dify\Test_SearchCustomer\agent_runtime\test.py)食用

• 整体流程

  - SkillRegistry.discover() 先扫描 skills/*，只加载带 SKILL.md 的目录。它从 SKILL.md 的 frontmatter 取 name/description，再把所有 ## 标题拆成 section；同时再读同目录的 skill.json，拿到 triggers、
    entry_sections、references、tools，组装成 SkillDefinition。skill_registry.py:52 models.py:15 SKILL.md skill.json:1
  - SkillRouter.route() 对每个 skill 打分后选最高分。规则很直接：显式写 $skill_name 加 100，每个 trigger 子串命中加 6，query 和 name + description + triggers 的 token overlap 再加 1..5；最终如果最高分低于
    2.0，就认为没命中任何 skill。router.py:22
  - 路由成功后，demo.main() 会创建一个 RuntimeState，把 task_id、user_request、active_skill、cwd、target_files、findings、memory_summary 写进去，然后落盘到 .agent_runtime/state/<task_id>.json。demo.py:61
    runtime_store.py:9
  - 最后 PromptAssembler.assemble() 把内容拼成可以直接喂给模型的三段 prompt：system 是固定总规则，developer 注入 active skill、skill description、选中的 skill section、runtime state 和可选 reference，user 就是    原始请求。prompt_assembler.py:8

  你提到的三种路由

  - 显式 $skill：本质是强制命中。代码直接检查 query 里是否包含 "$" + skill.name。router.py:42
  - 触发短语：来自 skill.json 的 triggers，做的是小写后的子串匹配，不是 embedding 检索，也不是复杂语义路由。skill.json:3
  - token overlap：_tokenize() 只识别 [a-z0-9_-]+，所以它对英文词有效，对纯中文基本无效。你这个仓库里之所以还能靠 overlap 命中，是因为 query 里带了 Dify、CRM、DSL 这类 ASCII token。router.py:8

  实际跑出来是什么样

  我实际跑了：

  python3 -m agent_runtime "帮我分析这个 Dify CRM 客户查询工作流的 DSL，并告诉我该改哪些节点" --target-file Test_SearchCustomer.yml

  结果命中了 maintain-dify-crm-customer-workflows，分数是 3.0，原因只有 token overlap: crm, dify, dsl。运行时状态被保存到 task-59594386.json，里面现在保存的是一次任务快照；最终输出的 prompt 已经包含标准的
  system/developer/user 三段，可直接传给模型。

  当前边界

  - 现在这套东西只是“发现 skill -> 规则路由 -> 拼 prompt -> 落 state”的最小骨架，还没有真实模型调用和工具编排。demo.py:61
  - FileRuntimeStore.load() 虽然已经写了，但 demo.main() 目前没有真正用它恢复旧状态；--task-id 现在更像是指定 state 文件名，而不是续跑会话。runtime_store.py:14 demo.py:83
  - tool_history 字段也还没被填充，所以 .agent_runtime/state/ 现在保存的是轻量运行时上下文，不是完整 agent memory。models.py:33

