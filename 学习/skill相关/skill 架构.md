• 对你现在这个场景，我建议把自建 agent 的 skill 设计成 4 层，而不是单纯一份长文档。

  推荐架构

  - Skill Registry：存每个 skill 的元数据、核心说明、references、tools 绑定。
  - Skill Router：判断这次请求要不要启用 skill，启用哪个 skill。
  - Prompt Assembler：只把选中的 skill 核心片段注入模型，references 按需二次加载。
  - Runtime：管理 memory、state、tool calls、RAG，不把这些混进 skill。

  边界

  - skill 存“怎么做”
  - memory/state 存“现在做到哪了”
  - knowledge 存“外部事实和业务资料”
  - tools 存“能执行什么动作”

  对你这个 CRM/Dify 维护场景，skill 应该是“维护 Dify CRM 工作流的方法包”，不是“当前 workflow 的运行时状态快照”。

  建议目录

  agent/
    skills/
      maintain-dify-crm-customer-workflows/
        skill.json
        SKILL.md
        references/
          dify-workflow-patterns.md
          crm-dataset-contract.md
        tools/
          manifest.json
    runtime/
      router.py
      prompt_assembler.py
      executor.py
      memory_store.py

  一个可实现的 skill.json

  {
    "name": "maintain-dify-crm-customer-workflows",
    "description": "Analyze and modify Dify CRM customer-query workflows with HTTP, dataset sync, retrieval, validation, and LLM answer chains.",
    "triggers": [
      "分析 Dify DSL",
      "修改 CRM 客户查询工作流",
      "排查 indexing_status",
      "调整 metadata 检索",
      "update Dify workflow YAML"
    ],
    "entry_sections": [
      "Overview",
      "Quick Start",
      "Stable Contracts",
      "Debugging Order"
    ],
    "references": [
      {
        "id": "workflow-patterns",
        "path": "references/dify-workflow-patterns.md",
        "when": "需要看通用节点模式和联动修改点"
      },
      {
        "id": "crm-contract",
        "path": "references/crm-dataset-contract.md",
        "when": "需要核对 CRM 字段、dataset metadata、query contract"
      }
    ],
    "tools": [
      "read_file",
      "search_code",
      "run_tests"
    ]
  }

  请求处理链路

  1. 用户请求进来。
  2. Router 用规则或 embedding 选 top-1/top-2 skill。
  3. Prompt Assembler 只注入：
      - system prompt
      - 选中 skill 的 description
      - SKILL.md 的核心 section
      - 当前任务相关 state 摘要
  4. 模型先产出计划或直接执行。
  5. 如果模型需要更细节，再按 section/reference id 加载补充材料。
  6. 工具执行结果写入 runtime state，不回写 skill。

  最关键的 Prompt 组装
  不要整份 SKILL.md 全塞进去。先塞最小集。

  System:
  You are an engineering agent.

  Active skill:
  maintain-dify-crm-customer-workflows

  Skill summary:
  <description>

  Skill instructions:
  <Overview>
  <Quick Start>
  <Stable Contracts>

  Current task:
  <user request>

  Current state:
  <repo path, files, previous findings>

  只有在模型判断“需要更细的排障规则”时，再加载：

  - Debugging Order
  - Common Changes
  - 某个 reference 文件

  Router 怎么做
  先别上复杂规划器，最小可用方案就够了：

  - 第一层：显式触发
      - 用户输入 $maintain-dify-crm-customer-workflows
  - 第二层：规则匹配
      - 命中 dify|workflow|dsl|yaml|crm|dataset|retrieval|indexing_status
  - 第三层：embedding 检索
      - 用用户请求去检索 skill description + triggers
  - 第四层：阈值判断
      - 分数低就不用 skill，避免误触发

  运行时状态建议
  用一个独立对象，不要写进 skill：

  {
    "task_id": "t_123",
    "active_skill": "maintain-dify-crm-customer-workflows",
    "repo_context": {
      "cwd": "/mnt/e/KeTan_Dify/Test_SearchCustomer",
      "target_files": ["Test_SearchCustomer.yml"]
    },
    "findings": [
      "workflow uses doc_ready as cache gate"
    ],
    "tool_history": [],
    "memory_summary": "User wants to modify retrieval behavior."
  }

  你这个场景的 skill 内容该怎么拆

  - SKILL.md
      - 主流程、联动修改原则、排障顺序
  - references/dify-workflow-patterns.md
      - 节点模式、常见分支、loop/indexing 结构
  - references/crm-dataset-contract.md
      - employee_code、metadata key、document naming、query contracts
  - tools
      - 文件读取
      - YAML 搜索
      - repo grep
      - 可选测试/校验命令

  最常见的落地错误

  - 把 skill 当成长期 memory
  - 每次都注入完整 skill 包
  - 有 skill 文档但没有配套 tools
  - state、RAG、skill 混成一个知识库
  - router 不做阈值，导致误触发很多 skill

  对你当前项目的直接建议
  先实现一个最小版本：

  - 1 个 skill registry
  - 1 个 router
  - 1 个 prompt assembler
  - 1 个 runtime state store
  - 先只支持 top-1 skill
  - 先只支持 SKILL.md 核心 section + references 按需加载
