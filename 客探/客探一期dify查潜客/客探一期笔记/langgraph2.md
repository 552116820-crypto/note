  Prep Checklist（你现在就该准备的材料，会直接决定排期准确度）

  - CRM：
      - 接口文档 + 3 份真实样例（含边界情况：字段缺失/为空/超长）
      - 更新频率：多久算“过期需要重同步”（比如 1 天/1 周）
  - 文档与输出：
      - 你希望向量库里存的“员工文档”长什么样（按模块？按字段？）
      - 2–3 个“期望输出样例”（包含引用怎么展示）
  - 运行要求：
      - 是否需要鉴权（JWT/SSO）第一版就上
      - 是否必须流式（SSE）第一版就上

  ———

  What To Learn（团队需要补的知识，按优先级）

  - FastAPI：Pydantic 模型、依赖注入、异常处理、SSE 流式
  - 异步任务：别只用 FastAPI BackgroundTasks（不可靠），建议了解 Celery/RQ/Arq 这类 worker 模式
  - LangGraph：state 设计、conditional edge、checkpoint（让状态可恢复）
  - RAG：chunk 策略、embedding、过滤检索、引用(citations)与评测方法
  - 向量库（按你的选型）：pgvector 的索引/查询写法，或 Qdrant/Milvus 的过滤与 upsert

  ———

  如果你确认两点，我可以把工期再压到更准并给出“第一版交付范围”建议：

  - 第一版要不要 SSE 流式输出？
  - 你们倾向向量库用 pgvector（Postgres） 还是 Qdrant/Milvus（独立服务）？
