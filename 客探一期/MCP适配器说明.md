# MCP适配器实现说明（通用，不局限 Dify）

可以，用 FastMCP 做 MCP 适配器是很合适的，而且不只适用于 Dify，后续接 LangGraph 或自建框架都通用。

## 先把概念捋清

1. MCP 是“给 LLM 用的协议层”，不是替代你后端 API。  
2. 你的后端 API（REST/gRPC/GraphQL）仍然是业务实现层和数据真源。  
3. MCP 适配器是中间层：把后端能力包装成 MCP 的 `tools/resources/prompts`，让 AI 客户端能发现和调用。  
4. 典型调用链是：`initialize -> tools/list -> tools/call -> 你的后端API`。  

## MCP 和后端 API 的关系（工程上怎么理解）

1. 后端 API 面向“前端/服务间调用”，MCP 面向“模型工具调用”。  
2. MCP Tool 可以一对一映射 API，也可以聚合多个 API（例如查客户 + 风险 + 联系人）。  
3. MCP 里要定义清晰输入 schema 和稳定输出结构，这样模型调用更稳定。  
4. 认证密钥应只保留在 MCP 服务器侧，不要让模型客户端直接拿业务 API 密钥。  

## 怎么写一个通用 MCP 适配器（FastMCP 版）

1. 先定义工具边界：按业务动作拆成 5-20 个稳定 Tool，不要把整个系统做成一个“大一统 tool”。  
2. 每个 Tool 入参做强约束（类型、必填、枚举、默认值）。  
3. Tool 内部调用你的 service/API 层，不直接写杂乱脚本逻辑。  
4. 统一错误输出（例如 `ok/error_code/error_message`），避免模型误判。  
5. 做超时、重试、限流、审计日志。  
6. 生产优先用 Streamable HTTP；本地开发可先用 stdio。  

```python
# server.py
import os
import httpx
from fastmcp import FastMCP

mcp = FastMCP("biz-gateway")
BASE_URL = os.environ["BACKEND_BASE_URL"]
TOKEN = os.environ["BACKEND_TOKEN"]

async def call_backend(path: str, params: dict) -> dict:
    async with httpx.AsyncClient(timeout=15) as c:
        r = await c.get(
            f"{BASE_URL}{path}",
            params=params,
            headers={"Authorization": f"Bearer {TOKEN}"}
        )
        r.raise_for_status()
        return r.json()

@mcp.tool
async def search_customer(keyword: str, limit: int = 10) -> dict:
    if not keyword.strip():
        return {"ok": False, "error_code": "BAD_INPUT", "items": []}

    try:
        data = await call_backend("/customers/search", {"q": keyword, "limit": limit})
        return {
            "ok": True,
            "total": data.get("total", 0),
            "items": data.get("items", [])
        }
    except httpx.HTTPError as e:
        return {
            "ok": False,
            "error_code": "BACKEND_REQUEST_FAILED",
            "error_message": str(e),
            "items": []
        }

if __name__ == "__main__":
    # 生产建议放在受控网络和鉴权之后
    mcp.run(transport="http", host="127.0.0.1", port=8000)
```

运行示例：

```bash
fastmcp run server.py:mcp --transport http --port 8000
```

## LangGraph 场景怎么接

1. 作为“消费者”：LangGraph 通过 `langchain-mcp-adapters` 连接多个 MCP Server，把 MCP tools 转成 LangChain/LangGraph 可用工具。  
2. 作为“被封装能力”：你也可以在 MCP Tool 内部直接调用 `graph.invoke/ainvoke`，把 LangGraph 工作流暴露成一个或多个 Tool。  
3. 建议不要把整个图状态原样返回给模型，只返回必要字段（answer、status、trace_id）。  

## 实现时最容易踩的坑

1. 把 MCP 当成 API 网关替代品，导致职责混乱。  
2. Tool 粒度过粗或过细，模型难以稳定规划调用。  
3. 输出结构不稳定（今天字符串，明天对象）。  
4. Streamable HTTP 上忽略鉴权、Origin 校验、CORS。  
5. 会话状态设计不清（你需要无状态还是跨调用状态）。  

如果你愿意，我可以下一步直接给你一份“可落地模板仓库结构”（`server.py + schemas.py + service.py + auth.py + tests`），按你后端接口直接替换就能跑。

## 参考资料

- MCP 架构总览（官方）: https://modelcontextprotocol.io/docs/learn/architecture  
- MCP 传输层规范（2025-06-18）: https://modelcontextprotocol.io/specification/2025-06-18/basic/transports  
- MCP Tools 规范（`tools/list` / `tools/call`）: https://modelcontextprotocol.io/specification/2025-06-18/server/tools  
- FastMCP Quickstart: https://gofastmcp.com/getting-started/quickstart  
- FastMCP PyPI: https://pypi.org/project/fastmcp/  
- LangChain/LangGraph 使用 MCP: https://docs.langchain.com/oss/python/langchain/mcp  
- 官方 MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk
