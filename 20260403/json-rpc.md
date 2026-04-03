# 问：介绍一下通信协议JSON-RPC 2.0

# JSON-RPC 2.0

JSON-RPC 2.0 是一种轻量级的远程过程调用（RPC）协议，使用 JSON 作为数据编码格式。

## 核心概念

- **无状态**：每个请求独立，服务端不需要维护会话状态
- **传输无关**：可运行在 HTTP、WebSocket、TCP、stdio 等任何传输层之上
- **双向通信**：支持请求/响应和单向通知两种模式

## 消息类型

### 1. 请求（Request）

```json
{
  "jsonrpc": "2.0",
  "method": "subtract",
  "params": [42, 23],
  "id": 1
}
```

四个字段：
- `jsonrpc` — 固定为 `"2.0"`
- `method` — 要调用的方法名
- `params` — 参数，可以是数组（按位置）或对象（按名称），可选
- `id` — 请求标识符，用于匹配响应

### 2. 响应（Response）

**成功：**
```json
{
  "jsonrpc": "2.0",
  "result": 19,
  "id": 1
}
```

**失败：**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": null
  },
  "id": 1
}
```

`result` 和 `error` 互斥，只会出现其中一个。

### 3. 通知（Notification）

没有 `id` 字段的请求就是通知，服务端**不返回响应**：

```json
{
  "jsonrpc": "2.0",
  "method": "log",
  "params": {"level": "info", "msg": "started"}
}
```

### 4. 批量调用（Batch）

将多个请求放入数组中一次发送：

```json
[
  {"jsonrpc": "2.0", "method": "add", "params": [1, 2], "id": 1},
  {"jsonrpc": "2.0", "method": "add", "params": [3, 4], "id": 2}
]
```

服务端返回一个响应数组（通知不产生响应）。

## 预定义错误码

| 错误码 | 含义 |
|--------|------|
| `-32700` | Parse error — JSON 解析失败 |
| `-32600` | Invalid Request — 不是合法的 JSON-RPC 请求 |
| `-32601` | Method not found — 方法不存在 |
| `-32602` | Invalid params — 参数无效 |
| `-32603` | Internal error — 服务端内部错误 |
| `-32000` ~ `-32099` | 服务端自定义错误（保留范围） |

## 典型应用场景

- **MCP（Model Context Protocol）**：Claude 等 AI 工具与外部工具的通信协议底层就是 JSON-RPC 2.0
- **LSP（Language Server Protocol）**：VS Code 等编辑器与语言服务器的通信
- **区块链**：Ethereum、Bitcoin 的节点 API
- **微服务间通信**：作为 REST 的轻量替代

## 与 REST 的对比

| | JSON-RPC 2.0 | REST |
|--|--|--|
| 风格 | 面向动作（调用方法） | 面向资源（CRUD） |
| URL | 通常单一端点 | 每个资源一个 URL |
| 批量 | 原生支持 | 需额外设计 |
| 复杂度 | 极简 | 较重（状态码、动词、内容协商） |

JSON-RPC 2.0 的规范非常短（只有几页），适合需要简单、高效 RPC 通信的场景。

---

# 问：详细介绍JSON-RPC 2.0在MCP和LSP这两个场景中的应用

# JSON-RPC 2.0 在 MCP 和 LSP 中的应用

## 一、LSP（Language Server Protocol）

LSP 是微软在 2016 年提出的协议，用于**编辑器/IDE 与语言服务器之间的通信**，解决了 M×N 问题（M 个编辑器 × N 种语言）。

### 架构

```
┌──────────────┐    JSON-RPC 2.0     ┌──────────────────┐
│   编辑器/IDE  │ ◄════════════════► │   语言服务器       │
│  (VS Code等)  │   stdio / TCP      │  (gopls, pylsp等) │
└──────────────┘                     └──────────────────┘
     Client                              Server
```

### 传输方式

LSP 在 JSON-RPC 消息外面包了一层 HTTP 风格的头部：

```
Content-Length: 73\r\n
\r\n
{"jsonrpc":"2.0","method":"textDocument/didOpen","params":{...}}
```

底层传输通常是 **stdio**（编辑器启动语言服务器子进程，通过 stdin/stdout 通信）或 TCP。

### 三种消息模式的实际运用

**1. 请求/响应 — 编辑器主动查询**

用户触发"转到定义"时，编辑器发送请求：

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "textDocument/definition",
  "params": {
    "textDocument": { "uri": "file:///src/main.ts" },
    "position": { "line": 10, "character": 15 }
  }
}
```

语言服务器返回定义位置：

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "uri": "file:///src/utils.ts",
    "range": {
      "start": { "line": 5, "character": 0 },
      "end": { "line": 5, "character": 20 }
    }
  }
}
```

类似的请求还有：
- `textDocument/completion` — 自动补全
- `textDocument/hover` — 悬停提示
- `textDocument/references` — 查找引用
- `textDocument/rename` — 重命名符号

**2. 通知 — 无需响应的单向消息**

用户编辑文件时，编辑器持续发送通知（无 `id`），服务器不回复：

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/didChange",
  "params": {
    "textDocument": { "uri": "file:///src/main.ts", "version": 12 },
    "contentChanges": [
      { "text": "const x = 1;" }
    ]
  }
}
```

其他通知：
- `textDocument/didOpen` — 文件打开
- `textDocument/didSave` — 文件保存
- `textDocument/didClose` — 文件关闭

**3. 服务器 → 客户端的请求和通知**

语言服务器分析完代码后，**主动推送**诊断信息（通知）：

```json
{
  "jsonrpc": "2.0",
  "method": "textDocument/publishDiagnostics",
  "params": {
    "uri": "file:///src/main.ts",
    "diagnostics": [
      {
        "range": { "start": {"line": 3, "character": 0}, "end": {"line": 3, "character": 5} },
        "severity": 1,
        "message": "Type 'string' is not assignable to type 'number'."
      }
    ]
  }
}
```

服务器也可以向客户端发请求（有 `id`），比如 `window/showMessageRequest` 弹对话框让用户选择。

### 生命周期

```
Client                              Server
  │                                    │
  │── initialize (Request) ──────────►│   能力协商
  │◄── initialize (Response) ─────────│
  │── initialized (Notification) ────►│   确认就绪
  │                                    │
  │   ......正常工作阶段......           │
  │                                    │
  │── shutdown (Request) ────────────►│   准备关闭
  │◄── shutdown (Response) ───────────│
  │── exit (Notification) ───────────►│   进程退出
```

关键点：`initialize` 阶段双方交换 **capabilities**，声明各自支持哪些功能，实现渐进式兼容。

---

## 二、MCP（Model Context Protocol）

MCP 是 Anthropic 在 2024 年提出的协议，用于 **AI 应用（Host/Client）与外部工具/数据源（Server）之间的通信**。

### 架构

```
┌───────────────┐                         ┌───────────────┐
│   Host        │                         │  MCP Server   │
│  (Claude等)    │    JSON-RPC 2.0        │  (工具提供方)   │
│  ┌──────────┐ │ ◄══════════════════════►│               │
│  │ MCP Client│ │   stdio / SSE / HTTP    │  Tools        │
│  └──────────┘ │                         │  Resources    │
└───────────────┘                         │  Prompts      │
                                          └───────────────┘
```

- **Host**：AI 应用（如 Claude Desktop、Claude Code）
- **Client**：Host 内部为每个 Server 维护的 1:1 连接
- **Server**：暴露工具、资源、提示模板的外部进程

### 传输方式

MCP 支持多种传输：

| 传输 | 场景 | 说明 |
|------|------|------|
| **stdio** | 本地进程 | Host 启动 Server 子进程，通过 stdin/stdout 通信 |
| **Streamable HTTP** | 远程服务 | 单一 HTTP 端点，支持 SSE 流式推送 |

与 LSP 不同，MCP 的 stdio 模式**不需要** `Content-Length` 头部，直接按行发送 JSON-RPC 消息（每行一个完整 JSON 对象）。

### 三大原语（Primitives）

MCP Server 向 Client 暴露三种能力：

**1. Tools（工具）— 模型可调用的函数**

Client 发现可用工具：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

Server 返回工具定义（使用 JSON Schema 描述参数）：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "query_database",
        "description": "Execute a read-only SQL query",
        "inputSchema": {
          "type": "object",
          "properties": {
            "sql": { "type": "string", "description": "The SQL query" }
          },
          "required": ["sql"]
        }
      }
    ]
  }
}
```

AI 决定调用工具时，Client 发送：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "query_database",
    "arguments": { "sql": "SELECT count(*) FROM users" }
  }
}
```

Server 执行并返回结果：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      { "type": "text", "text": "Count: 1523" }
    ]
  }
}
```

**2. Resources（资源）— 上下文数据**

类似 REST 的 GET，让 AI 读取外部数据：

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/read",
  "params": {
    "uri": "postgres://localhost/mydb/schema"
  }
}
```

**3. Prompts（提示模板）— 预定义的交互模板**

```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "prompts/get",
  "params": {
    "name": "explain_code",
    "arguments": { "language": "python" }
  }
}
```

### 生命周期

```
Client                               Server
  │                                     │
  │── initialize (Request) ───────────►│  协议版本 + 能力协商
  │◄── initialize (Response) ──────────│
  │── notifications/initialized ──────►│  确认就绪
  │                                     │
  │── tools/list ─────────────────────►│  发现工具
  │── tools/call ─────────────────────►│  调用工具
  │── resources/read ─────────────────►│  读取资源
  │                                     │
  │◄── notifications/tools/list_changed│  Server 通知工具变化
  │                                     │
  │── ping ───────────────────────────►│  心跳检测
  │◄── ping response ─────────────────│
```

### 通知的典型用途

Server 端工具列表发生变化时，主动推送通知：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

Client 收到后重新调用 `tools/list` 获取最新工具列表。

---

## 三、两者的对比

| | LSP | MCP |
|--|--|--|
| **发起方** | 微软 (2016) | Anthropic (2024) |
| **目的** | 编辑器 ↔ 语言服务器 | AI 应用 ↔ 外部工具/数据 |
| **JSON-RPC 传输** | stdio + `Content-Length` 头 | stdio（按行）/ Streamable HTTP |
| **能力发现** | `initialize` 中交换 capabilities | `tools/list`, `resources/list`, `prompts/list` |
| **消息方向** | 双向（都可发请求和通知） | 双向（都可发请求和通知） |
| **典型请求量** | 高频（每次击键都可能触发） | 相对低频（AI 决策时触发） |
| **设计哲学** | 解决 M×N 问题（编辑器×语言） | 解决 AI 与外部世界的连接问题 |

## 四、为什么都选择 JSON-RPC 2.0

1. **极简** — 规范只有几页，实现成本低
2. **双向通信** — 两端都可以发请求和通知，适合长连接场景
3. **传输无关** — stdio、HTTP、WebSocket 都能用，不绑定特定传输
4. **请求/通知区分** — 通过 `id` 的有无天然区分需要回复和不需要回复的消息
5. **批量调用** — 可一次发送多个请求，减少往返开销

两个协议都从 JSON-RPC 2.0 的简洁性中获益，在此基础上定义了各自的方法名、参数结构和生命周期管理。

---

# 问：json-rpc就是传输信息的格式

对，本质上就是这样。JSON-RPC 2.0 只定义了**消息的 JSON 结构**——怎么表示"调用什么方法、传什么参数、返回什么结果、报什么错"。

它**不管**的事情：

- 消息怎么传输（HTTP？stdio？WebSocket？随你）
- 消息怎么序列化到字节流（要不要加 `Content-Length` 头？按行分割？由上层协议自己决定）
- 业务语义（方法名叫什么、参数结构是什么、调用顺序是什么）

所以 LSP 和 MCP 做的事情是在 JSON-RPC 2.0 这个**消息格式**之上，各自定义了：

```
JSON-RPC 2.0        ← 消息长什么样
   +
传输约定             ← 消息怎么送达（stdio 加头 / 按行分割 / HTTP）
   +
方法与参数定义        ← textDocument/definition / tools/call 等
   +
生命周期管理          ← initialize → 工作 → shutdown
   +
能力协商             ← 双方声明自己支持什么
```

JSON-RPC 2.0 只负责最底下那一层。
