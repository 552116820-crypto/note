• 加上你刚确认的两个约束（SSE 流式输出 + 向量库用 Qdrant/Milvus），这条 20 节点工作流的迁移工期一般会比“非流式 + pgvector”多一点，主要多在“流式编排/连接管理”和“向量库接
  入与运维”上。

  工期（1 名熟练工程师，800 人×每人 60k 文档，一人一文档）

  - POC（跑通：异步同步 + 过滤检索 + SSE）：约 7–10 个工作日（1.5–2 周）
  - MVP（可给业务用：稳定、可追踪、可回归）：约 3–4 周（15–20 个工作日）
  - 生产可用（可运维：监控、重试、灰度、权限隔离更完整）：约 6–8 周（30–40 个工作日）

  > 你们规模不大（总 chunk 量级大概率在 6–12 万），工期的决定因素不是“量”，而是：异步同步的状态一致性 + SSE 流式工程化 + 文档更新/重嵌入策略。

  ———

  ## 你应该怎么做（推荐落地路径）

  1. 把 Dify 的 doc_ready 改成“按 employee_code 的文档就绪状态”

  - Dify 里是会话变量；迁移后建议变成表/状态：employee_code -> {ready, last_synced_at, version_hash, last_job_status}
  - 否则换个会话就会重复同步/重复 embedding，成本和稳定性都会出问题

  2. 拆成两条链：在线问答链 + 离线/异步同步链

  - 异步同步链（worker）：CRM -> normalize -> chunk -> embed -> upsert -> mark_ready
  - 在线问答链（API）：
      - 若该员工文档已 ready：直接 rewrite -> retrieve(filter employee_code) -> answer（SSE 输出）
      - 若未 ready：立刻触发同步任务并在 SSE 里返回清晰状态（见下面“事件设计”）

  3. 向量库设计（Qdrant/Milvus 通用思路）

  - 单 collection（或 partition）存所有 chunk
  - 每条向量 payload/fields 至少包含：employee_code、doc_id、chunk_id、section、updated_at、（可选）version_hash
  - 检索必须强制：filter employee_code == 输入 employee_code（这是你现在 Dify 的 metadata 过滤）
  4. 更新策略（对应“存在更新，不存在新建”）

  - 最简单可靠：每次同步时 按 employee_code 先删旧 chunk 再写新 chunk（或用 upsert 覆盖同 chunk_id）
  - 为了省 embedding：先算 version_hash（例如 CRM 关键字段拼接后 hash）；hash 不变就跳过重建

  5. SSE 事件设计（建议你们第一版就标准化）

  - 不建议只“吐 token”；建议同时吐“状态事件”，方便前端/调用方处理：
      - event: status：sync_started / sync_in_progress / sync_done / sync_failed
      - event: token：模型流式输出片段
      - event: citations：命中的 chunk 来源（最后一次性发也行）
      - event: done：最终结构化结果 + 统计信息（耗时、命中数、版本）

  ———

  ## Qdrant vs Milvus（选型建议，直接影响复杂度）

  - Qdrant：接入简单、payload 过滤直观、运维相对轻；你这个规模优先推荐它
  - Milvus：更偏“重型检索基础设施”，能力强但组件多、运维复杂度更高；适合你们已有 k8s/向量平台经验时选

  如果你们没有专门运维同学，Qdrant 往往能把工期压缩 3–5 天。

  ———

  ## 你需要做的准备（给了这些，基本就能开工）

  - CRM
      - 接口文档 + 3 个真实样例（字段全/字段缺/超长）
      - 同步频率目标：多久刷新一次？（1 天/1 周/手动触发）
  - 文档结构
      - 60k 文档最好按业务模块分段（例如：客户概览/商机/跟进记录/目标/风险点），否则 chunking 容易“语义打散”
  - 输出与引用
  如果你们没有专门运维同学，Qdrant 往往能把工期压缩 3–5 天。

  ———
 
  ## 你需要做的准备（给了这些，基本就能开工）
 
  - CRM
      - 接口文档 + 3 个真实样例（字段全/字段缺/超长）
      - 同步频率目标：多久刷新一次？（1 天/1 周/手动触发）
  - 文档结构
      - 60k 文档最好按业务模块分段（例如：客户概览/商机/跟进记录/目标/风险点），否则 chunking 容易“语义打散”
  - 输出与引用
      - 给 2–3 个“期望输出样例”（包括引用要怎么展示：字段名、时间、来源链接/ID）
  - SSE 交互期望
      - “未 ready 时”要不要直接返回“初始化中请稍后再问”，还是要显示进度并等待同步完成后继续回答（体验更好但工程复杂度更高）
 
  ———
 
  ## 需要学习/补齐的知识点（按优先级）

  - FastAPI SSE：StreamingResponse、连接断开处理、心跳、反压（客户端慢读怎么办）
  - LangGraph 流式：把图执行过程/LLM token 流转成 SSE 事件（不仅是最终结果）
  - 异步任务队列：Celery/RQ/Arq 任选其一（别只靠 FastAPI BackgroundTasks，可靠性不够）
  - 向量库实战（Qdrant/Milvus）：upsert、按字段过滤、删除策略、索引/分区、批量写入
  - RAG 工程：chunk 策略、top_k、（可选）rerank、引用生成与回归评测

  ———
 
  你再确认 2 个产品决策，我就能把“2–8 周”进一步细成可执行排期（按周拆里程碑）：
 
  1. 未 ready 时：要不要“等同步完成后同一次 SSE 里继续回答”，还是先返回“初始化中 + job_id”？
  2. Qdrant/Milvus 你们更倾向哪一个（以及是否已有 k8s/运维支持）？
