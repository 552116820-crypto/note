• 结论：按你描述的目标（“从知识库召回内容 → 大模型总结回复”，并且要嵌在企业微信工作台的 CRM 侧边栏 iframe 里），更推荐 Dify 作为对话/RAG 应用层；n8n 更适合做“数据同步/编
  排”，而不是当聊天 WebApp 来嵌入。 (docs.dify.ai (https://docs.dify.ai/en/use-dify/publish/webapp/embedding-in-websites?utm_source=openai))

  为什么更偏向 Dify

  - 直接给前端一个可嵌入的 URL：Dify 发布应用后自带 WebApp，并官方支持用 iframe 嵌入到你们现有页面里（也有 Chat widget/JS 控制等方式）。 (docs.dify.ai (https://
    docs.dify.ai/en/use-dify/publish/webapp/embedding-in-websites?utm_source=openai))
  - RAG 是原生能力：有“Knowledge Retrieval/知识检索”能力，把召回的文档块和元数据输出给下游 LLM 用来生成总结，并且能做引用/归因。 (docs.dify.ai (https://docs.dify.ai/en/ 
    use-dify/nodes/knowledge-retrieval?utm_source=openai))
  - 检索策略可调：知识库支持向量/全文/混合检索与 rerank 等配置，能更容易把“召回准确率”调上来。 (docs.dify.ai (https://docs.dify.ai/en/guides/knowledge-base/create-      
    knowledge-and-upload-documents/setting-indexing-methods?utm_source=openai))
  - 对话体验开箱即用：WebApp 支持“自动开场白（Conversation Opener）”以及“语音转文字输入（Speech-to-Text）”（用户点麦克风即可）。 (docs.dify.ai (https://docs.dify.ai/    
    versions/3-0-x/en/user-guide/application-publishing/launch-your-webapp-quickly/conversation-application?utm_source=openai))

  n8n 为什么不优先（在你这个“iframe + Web 聊天页”场景）

  - n8n 当然能做 RAG（HTTP 拉 CRM、向量库检索、拼上下文、调大模型），但如果你走“Webhook 直接返回 HTML 当聊天页”，从 n8n 1.103.0 起 HTML 会被自动包进 sandbox iframe：    
    localStorage/top window 访问失败、认证 header 不可用、相对 URL 容易失效；再叠加你们 CRM 侧边栏本身也是 iframe，整体会更脆。 (docs.n8n.io (https://docs.n8n.io/       
    integrations/builtin/core-nodes/n8n-nodes-base.respondtowebhook/?utm_source=openai))
  - 也就是说：n8n 更像“后端编排/自动化平台”，要做出 Dify 那种可嵌入、可维护的聊天 WebApp，需要你额外写不少前端与鉴权适配。

  最稳的组合方案（很多团队会这么做）

  - Dify 负责：对话 UI + 知识检索 + 总结生成 + iframe 嵌入。 (docs.dify.ai (https://docs.dify.ai/en/use-dify/publish/webapp/embedding-in-websites?utm_source=openai))    
  - n8n 负责：从 CRM 接口增量拉 JSON → 清洗成文档 → 写入 Dify 知识库（用 Dify 的知识库 API 以“文本建文档/更新文档”方式维护）。 (docs.dify.ai (https://docs.dify.ai/      
    versions/3-0-x/en/user-guide/knowledge-base/knowledge-and-documents-maintenance/maintain-dataset-via-api?utm_source=openai))
  - 如果你更想“实时查 CRM 而不是入库”，Dify 工作流也支持 HTTP Request 节点去调用外部 API（包括 CRM）。 (docs.dify.ai (https://docs.dify.ai/en/guides/workflow/node/http- 
    request?utm_source=openai))

  你只想做“总结性回复”，还是需要按当前 CRM 记录（例如 account_id/deal_id）做强关联（侧边栏随选中客户变化而变化）？这会决定更适合“预先入库检索”还是“实时查 CRM + 总结”。  
