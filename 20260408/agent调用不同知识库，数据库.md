# Agent 调用不同知识库

## 一、Agent 调用不同知识库的常见做法

### 核心思路：Router/意图识别 → 分发

```
用户问题 → LLM 判断意图 → 选择对应知识库 → 检索 → 生成回答
```

### 具体实现方式

**方式 A：Tool/Function Calling（最主流）**
- 每个知识库注册为一个 tool（工具）
- LLM 根据问题自动决定调哪个 tool

```python
tools = [
    {"name": "search_hr_kb", "description": "查询人事制度相关问题"},
    {"name": "search_tech_kb", "description": "查询技术文档"},
    {"name": "search_finance_kb", "description": "查询财务制度"},
]
# LLM 自动选择调用哪个
```

**方式 B：多 Index + 元数据路由**
- 所有知识库存在同一个向量数据库，用 `namespace`/`collection` 区分
- 查询时通过 metadata filter 选择

```python
results = vectordb.query(
    query_embedding,
    filter={"knowledge_base": "hr_policy"},  # 元数据过滤
    top_k=5
)
```

**方式 C：两阶段检索（Router + RAG）**
- 第一步：LLM 判断该查哪些知识库（可以多选）
- 第二步：并行检索 → 合并结果 → 生成回答

### 关键设计点

| 环节 | 做法 |
|------|------|
| **知识库识别** | Function Calling / 分类器 / embedding 相似度 |
| **多库并查** | 同时检索多个库，用 rerank 合并 |
| **兜底策略** | 所有库都查不到时，走通用回答 |
| **知识库描述** | 给每个库写清晰的 description，直接影响路由准确率 |

---

## 二、LangChain 中 Agent 调用不同知识库的实现

### 核心思路：每个知识库封装为一个 Tool，由 Agent 自动路由

### 1. 创建多个知识库（Retriever）

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# 分别加载不同知识库
hr_vectorstore = FAISS.load_local("hr_index", embeddings)
tech_vectorstore = FAISS.load_local("tech_index", embeddings)
finance_vectorstore = FAISS.load_local("finance_index", embeddings)

hr_retriever = hr_vectorstore.as_retriever()
tech_retriever = tech_vectorstore.as_retriever()
finance_retriever = finance_vectorstore.as_retriever()
```

### 2. 把 Retriever 包装成 Tool

```python
from langchain.tools.retriever import create_retriever_tool

hr_tool = create_retriever_tool(
    hr_retriever,
    name="search_hr_policy",
    description="查询人事制度、考勤、请假、薪资福利相关问题时使用",
)

tech_tool = create_retriever_tool(
    tech_retriever,
    name="search_tech_docs",
    description="查询技术文档、API接口、系统架构相关问题时使用",
)

finance_tool = create_retriever_tool(
    finance_retriever,
    name="search_finance_rules",
    description="查询财务报销、预算、发票相关问题时使用",
)

tools = [hr_tool, tech_tool, finance_tool]
```

### 3. 创建 Agent

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate

llm = ChatOpenAI(model="gpt-4o")

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个企业知识助手，根据用户问题选择合适的知识库查询并回答。"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

### 4. 调用

```python
result = executor.invoke({"input": "公司年假有几天？"})
# Agent 自动选择 search_hr_policy → 检索 → 回答

result = executor.invoke({"input": "订单接口的请求参数是什么？"})
# Agent 自动选择 search_tech_docs → 检索 → 回答
```

### 执行流程

```
用户: "报销流程是什么？"
  ↓
Agent(LLM) 看到 3 个 tool 的 description
  ↓ 判断匹配 "财务报销"
调用 search_finance_rules("报销流程")
  ↓
Retriever 向量检索 → 返回相关文档片段
  ↓
Agent(LLM) 基于检索结果生成最终回答
```

### 关键点

| 要点 | 说明 |
|------|------|
| **路由靠 description** | Tool 的 `description` 写得越精准，路由越准确 |
| **可以多次调用** | Agent 可以在一轮中调多个知识库，合并回答 |
| **verbose=True** | 开启后可以看到 Agent 选了哪个 tool，方便调试 |
| **兜底** | 可以加一个 "通用知识库" tool 作为 fallback |

### 进阶：用 MultiQueryRetriever 提升召回

```python
from langchain.retrievers.multi_query import MultiQueryRetriever

# 用 LLM 生成多个查询变体，提高召回率
multi_retriever = MultiQueryRetriever.from_llm(
    retriever=tech_retriever,
    llm=llm,
)
```

---

## 三、Dify、FastGPT 等平台介绍

### Dify
一个**开源的 LLM 应用开发平台**（低代码/可视化）：
- 拖拽式编排 Agent、RAG、Workflow
- 内置知识库管理、Tool 集成、对话管理
- 适合**快速搭建**企业级 AI 应用，不太需要写代码
- GitHub: `langgenius/dify`

### FastGPT
一个**开源的知识库问答系统**（国内团队开发）：
- 专注于 RAG 知识库场景，可视化编排工作流
- 内置文档导入、向量检索、多知识库路由
- 适合**企业知识库客服**等场景
- GitHub: `labring/FastGPT`

### 对比

| 项目 | 定位 | 是否 Agent 框架 |
|------|------|----------------|
| **LangChain** | 开发框架（代码级） | 是，纯框架，灵活度最高 |
| **Dify** | 应用平台（低代码） | 包含 Agent 能力，但更偏上层平台 |
| **FastGPT** | 知识库问答平台 | 有简单 Agent 能力，核心是 RAG |
| **Coze（扣子）** | 字节的 Agent 平台 | 是，可视化构建 Agent |

简单理解：
- **LangChain** = 乐高积木（自己拼）
- **Dify / FastGPT** = 预制的组装模型（拖拽配置）

---

## 四、自建 Agent 框架

### Agent 的本质就是一个循环

```python
while True:
    # 1. LLM 根据用户问题 + 工具列表，决定下一步
    action = llm.decide(question, tools, history)
    
    # 2. 如果 LLM 说"我可以直接回答了"→ 结束
    if action.type == "final_answer":
        return action.answer
    
    # 3. 否则执行 LLM 选择的工具
    result = execute_tool(action.tool_name, action.tool_args)
    
    # 4. 把工具结果加入历史，继续循环
    history.append(result)
```

### 自建的最小实现（约 100 行）

```python
import json
from openai import OpenAI

client = OpenAI()

# --- 定义工具 ---
def search_kb_hr(query: str) -> str:
    """模拟查询HR知识库"""
    return "年假规定：工作满1年享5天，满10年享10天"

def search_kb_tech(query: str) -> str:
    """模拟查询技术知识库"""
    return "订单接口：POST /api/orders，参数：user_id, product_id"

TOOLS_MAP = {
    "search_kb_hr": search_kb_hr,
    "search_kb_tech": search_kb_tech,
}

TOOLS_SCHEMA = [
    {
        "type": "function",
        "function": {
            "name": "search_kb_hr",
            "description": "查询人事、考勤、薪资相关问题",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "search_kb_tech",
            "description": "查询技术文档、API接口相关问题",
            "parameters": {
                "type": "object",
                "properties": {"query": {"type": "string"}},
                "required": ["query"],
            },
        },
    },
]

# --- Agent 核心循环 ---
def agent_run(user_input: str) -> str:
    messages = [
        {"role": "system", "content": "你是企业知识助手，根据问题选择合适的工具查询后回答。"},
        {"role": "user", "content": user_input},
    ]

    for _ in range(5):  # 最多循环5次，防止死循环
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS_SCHEMA,
        )
        msg = response.choices[0].message

        # LLM 直接回答 → 结束
        if not msg.tool_calls:
            return msg.content

        # 执行 LLM 选择的工具
        messages.append(msg)
        for tool_call in msg.tool_calls:
            fn = TOOLS_MAP[tool_call.function.name]
            args = json.loads(tool_call.function.arguments)
            result = fn(**args)
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result,
            })

    return "抱歉，无法回答该问题"

# --- 使用 ---
print(agent_run("公司年假有几天？"))
# → LLM 自动调 search_kb_hr → 基于结果回答
```

### 自建 vs 用现成框架的建议

| 场景 | 建议 |
|------|------|
| 学习理解 Agent 原理 | **自建**，上面 100 行就够了 |
| 快速上线企业应用 | **Dify / FastGPT**，省时间 |
| 需要深度定制、复杂编排 | **LangChain** 或 **自建** |
| 团队技术能力强，要完全掌控 | **自建**，避免框架黑盒 |

### 自建时的关键建议

1. **先跑通最小循环**（上面的代码），再逐步加功能
2. **Tool description 是灵魂**——路由准不准全靠它
3. **加 memory**——用数据库存对话历史，支持多轮
4. **加 RAG**——把 `search_kb_*` 替换为真实的向量检索
5. **加可观测性**——记录每次 LLM 选了哪个 tool、为什么选，方便调试

核心就是 **LLM + Tool Calling + 循环**，其他都是在这个基础上加功能。

---

## 五、OpenClaw 框架介绍

### 简介
OpenClaw（前身叫 Moltbot / Clawdbot）是一个**开源的个人 AI Agent 框架**，GitHub 上有 163k+ stars，MIT 协议。

不只是一个 Agent 框架，更像是一个**跑在本地的 AI 助手守护进程**，能连接你日常使用的通讯工具（WhatsApp、Telegram、Slack、Signal 等），代你执行操作。

### 五大架构组件

| 组件 | 作用 |
|------|------|
| **Gateway** | 消息网关，对接 50+ 通讯平台（Slack/微信/WhatsApp 等） |
| **Brain** | 核心大脑，用 ReAct 推理循环编排 LLM 调用 |
| **Memory** | 持久化记忆，本地 Markdown 文件存储 |
| **Skills** | 插件式技能，社区已有 5700+ 技能 |
| **Heartbeat** | 心跳调度器，定时任务、监控收件箱 |

### 特点
- **本地优先**——数据和记忆都存在本地磁盘（Markdown 文件）
- **模型无关**——支持配置多个 LLM provider，自动故障切换
- **技能市场（ClawHub）**——社区贡献的可复用 Skill 插件
- **常驻运行**——即使你不在电脑前也能工作（守护进程）

### 和其他框架的对比

| 框架 | 侧重点 |
|------|--------|
| **LangChain** | 代码级开发框架，灵活拼装 |
| **Dify** | 可视化低代码平台 |
| **FastGPT** | 专注知识库 RAG |
| **OpenClaw** | 本地 AI 助手 + 多平台通讯集成 + 技能插件生态 |

### OpenClaw 技术栈

**核心语言：TypeScript / Node.js**

#### 项目结构（pnpm monorepo）

```
openclaw/
├── packages/core       # 核心框架，Provider 接口，基类
├── packages/gateway    # 控制面：WebSocket 服务、会话管理、通道路由
├── packages/agent      # Agent 运行时（RPC 模式、工具流、block 流）
├── packages/cli        # 命令行工具（初始化、网关控制、消息）
├── packages/sdk        # SDK，用于构建自定义 Tool 和插件
└── packages/ui         # WebChat 界面和控制台 Dashboard
```

#### 关键技术选型

| 层面 | 技术 |
|------|------|
| **语言** | TypeScript |
| **运行时** | Node.js（长驻守护进程） |
| **包管理** | pnpm monorepo |
| **通信协议** | WebSocket（默认端口 18789） |
| **部署方式** | 本地 daemon 或 systemd 服务 |
| **LLM 对接** | Anthropic、OpenAI、本地模型等（Provider 抽象层） |
| **记忆存储** | 本地 Markdown 文件 |

本质上是一个 **TypeScript + Node.js 的 monorepo 项目**，没有依赖 LangChain 之类的框架，是自己实现的 Agent 循环和工具调用体系。

---

## 参考链接

- [OpenClaw 官网](https://openclaw.ai/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [What Is OpenClaw? Complete Guide - Milvus Blog](https://milvus.io/blog/openclaw-formerly-clawdbot-moltbot-explained-a-complete-guide-to-the-autonomous-ai-agent.md)
- [OpenClaw Explained - Medium](https://medium.com/@cenrunzhe/openclaw-explained-how-the-hottest-agent-framework-works-and-why-data-teams-should-pay-attention-69b41a033ca6)
- [OpenClaw Architecture, Explained](https://ppaolo.substack.com/p/openclaw-system-architecture-overview)
- [OpenClaw Architecture & Setup Guide 2026](https://vallettasoftware.com/blog/post/openclaw-2026-guide)
- [Gateway Architecture - OpenClaw Docs](https://docs.openclaw.ai/concepts/architecture)
- [Deep Dive into OpenClaw - Medium](https://medium.com/@dingzhanjun/deep-dive-into-openclaw-architecture-code-ecosystem-e6180f34bd07)
