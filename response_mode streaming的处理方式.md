# response_mode streaming 的处理方式

## Dify API 请求示例

```json
{
  "inputs": {},
  "query": "整理拜访总结",
  "response_mode": "streaming",
  "user": "001708"
}
```

---

## 1. streaming 模式返回的是 SSE（Server-Sent Events）

当 `response_mode` 设为 `"streaming"` 时，接口返回的是 SSE 格式的流式数据。

每条数据以 `data:` 开头，格式如下：

```
data: {"event": "message", "task_id": "xxx", "answer": "你", ...}
data: {"event": "message", "task_id": "xxx", "answer": "好", ...}
data: {"event": "message_end", "task_id": "xxx", "metadata": {...}}
```

### 主要事件类型

| event | 说明 |
|-------|------|
| `message` | 逐步返回的文本片段（`answer` 字段） |
| `message_end` | 流结束，包含 `metadata`（token 用量等） |
| `error` | 出错信息 |

### Python 处理示例

```python
import requests

response = requests.post(url, json={
    "inputs": {},
    "query": "整理拜访总结",
    "response_mode": "streaming",
    "user": "001708"
}, headers={"Authorization": "Bearer YOUR_API_KEY"}, stream=True)

for line in response.iter_lines():
    if line:
        line = line.decode("utf-8")
        if line.startswith("data: "):
            import json
            data = json.loads(line[6:])
            if data["event"] == "message":
                print(data["answer"], end="", flush=True)
            elif data["event"] == "message_end":
                print("\n--- 完成 ---")
```

### JavaScript (fetch) 处理示例

```javascript
const response = await fetch(url, {
    method: "POST",
    headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer YOUR_API_KEY"
    },
    body: JSON.stringify({
        inputs: {},
        query: "整理拜访总结",
        response_mode: "streaming",
        user: "001708"
    })
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    const text = decoder.decode(value);
    for (const line of text.split("\n")) {
        if (line.startsWith("data: ")) {
            const data = JSON.parse(line.slice(6));
            if (data.event === "message") {
                process.stdout.write(data.answer);
            }
        }
    }
}
```

---

## 2. Python 代码逐行解释

```python
for line in response.iter_lines():
```
逐行迭代响应，每收到一行就处理，不用等全部返回。

```python
    if line:
```
跳过空行（SSE 中事件之间用空行分隔）。

```python
        line = line.decode("utf-8")
```
把字节转成字符串。

```python
        if line.startswith("data: "):
```
只处理 `data:` 开头的行（SSE 数据行）。

```python
            data = json.loads(line[6:])
```
去掉前6个字符 `"data: "`，把剩余部分解析成 JSON 字典。

```python
            if data["event"] == "message":
                print(data["answer"], end="", flush=True)
```
如果是文本片段事件，打印 `answer` 内容（`end=""` 不换行，`flush=True` 立即输出）。

```python
            elif data["event"] == "message_end":
                print("\n--- 完成 ---")
```
如果是结束事件，打印完成提示。

### 实际效果

假设 Dify 返回：
```
data: {"event": "message", "answer": "拜访"}
data: {"event": "message", "answer": "总结"}
data: {"event": "message", "answer": "如下：..."}
data: {"event": "message_end", "metadata": {...}}
```

终端输出：
```
拜访总结如下：...
--- 完成 ---
```

就像打字机一样，字一个个蹦出来，而不是等全部生成完才显示。

---

## 3. blocking 模式返回的是普通 JSON

`blocking` 模式等全部生成完毕后一次性返回完整 JSON。

### blocking vs streaming 对比

| | blocking | streaming |
|---|---|---|
| **返回格式** | 一次性返回完整 JSON | SSE 逐条推送 |
| **等待时间** | 需等生成完才能拿到结果 | 立即开始接收片段 |
| **适用场景** | 后台任务、不需要实时展示 | 聊天界面、实时展示 |

### blocking 返回示例

```json
{
  "event": "message",
  "task_id": "xxx",
  "answer": "拜访总结如下：...(完整内容)",
  "metadata": {
    "usage": {
      "prompt_tokens": 100,
      "completion_tokens": 200,
      "total_tokens": 300
    }
  }
}
```

### blocking 处理方式

```python
response = requests.post(url, json={
    "inputs": {},
    "query": "整理拜访总结",
    "response_mode": "blocking",
    "user": "001708"
}, headers={"Authorization": "Bearer YOUR_API_KEY"})

result = response.json()
print(result["answer"])  # 直接拿完整结果
```

简单说：**blocking = 一个 JSON，streaming = 一串 SSE 事件**。

---

## 4. SSE 的本质

SSE（Server-Sent Events）本质就是一个**长连接的 HTTP 响应**，服务端不断往里写文本，客户端持续读取。

- 就是普通的 HTTP 请求，但服务端**不关闭连接**，持续往响应体里写数据
- Content-Type 为 `text/event-stream`
- 数据是**纯文本**，用固定格式分隔

### 协议格式

```
data: 第一条消息\n\n
data: 第二条消息\n\n
data: 第三条消息\n\n
```

每条消息以 `data: ` 开头，以**两个换行** `\n\n` 结束。

### 跟普通 HTTP 响应的区别

```
普通 HTTP：  请求 → 等待 → 一次性收到完整响应 → 连接关闭
SSE：       请求 → 持续收到一条条数据 → ... → 服务端主动关闭
```

### SSE vs WebSocket

| | SSE | WebSocket |
|---|---|---|
| **方向** | 单向（服务端→客户端） | 双向 |
| **协议** | 普通 HTTP | 独立协议（ws://） |
| **复杂度** | 非常简单 | 较复杂 |
| **适合场景** | AI 流式输出、通知推送 | 聊天室、实时游戏 |

简单说，SSE 就是**服务端一直往一个没关的 HTTP 连接里写文本**，没有什么特殊协议，浏览器和 HTTP 客户端天然支持。

---

## 5. WebSocket 是什么

WebSocket 是一个**全双工通信协议**，让客户端和服务端可以随时互相发消息。

### 原理

```
1. 客户端发一个 HTTP 请求，带 Upgrade: websocket 头
2. 服务端同意，返回 101 Switching Protocols
3. 连接"升级"成 WebSocket，不再是 HTTP
4. 双方随时互发消息，直到一方关闭
```

### HTTP vs SSE vs WebSocket

```
HTTP：      客户端问 → 服务端答 → 结束（一问一答）
SSE：       客户端问 → 服务端持续说 → 结束（单向推送）
WebSocket： 握手后 → 双方随时说话（双向对等）
```

### 握手过程

客户端发：
```
GET /chat HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZQ==
```

服务端回：
```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

之后就不再是 HTTP 了，变成 WebSocket 自己的**二进制帧协议**。

### 使用示例

**JavaScript（浏览器原生支持）：**
```javascript
const ws = new WebSocket("ws://example.com/chat");

// 收消息
ws.onmessage = (event) => {
    console.log("收到：", event.data);
};

// 发消息
ws.send("你好");
```

**Python：**
```python
import websockets, asyncio

async def chat():
    async with websockets.connect("ws://example.com/chat") as ws:
        await ws.send("你好")       # 发
        reply = await ws.recv()     # 收
        print(reply)

asyncio.run(chat())
```

### 适用场景

| 场景 | 为什么用 WebSocket |
|---|---|
| 聊天室 | 双方都要实时发消息 |
| 在线游戏 | 需要低延迟双向通信 |
| 协同编辑 | 多人同时编辑要实时同步 |
| 实时行情 | 高频推送，HTTP 开销太大 |

简单说，WebSocket 就是**HTTP 握个手后升级成一条双向管道**，双方随时收发，不用反复建连接。

---

## 6. WebSocket 的本质

WebSocket 的本质就是一个**基于 TCP 的全双工通信协议**，借 HTTP 握手建立连接，之后完全脱离 HTTP，用自己的二进制帧格式收发数据。

### 本质拆解

#### 1) 建立连接：借 HTTP 的壳

WebSocket 不能直接建连接，它利用 HTTP 完成一次"升级"握手：

```
客户端 --HTTP请求--> 服务端
       "我想升级成 WebSocket"
服务端 --101响应--> 客户端
       "同意，升级成功"
```

握手完成后，**这条 TCP 连接就不再走 HTTP 协议了**。

#### 2) 传输数据：二进制帧

握手后，数据以**帧（frame）** 为单位传输，不再是 HTTP 的文本格式：

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |            (16/64)            |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+-------------------------------+
|    Masking-key (0 or 4 bytes) |          Payload Data         |
+-------------------------------+-------------------------------+
```

关键字段：
- **opcode**：标识数据类型（文本 `0x1`、二进制 `0x2`、关闭 `0x8`、ping `0x9`、pong `0xA`）
- **MASK**：客户端发的数据必须掩码，服务端发的不需要
- **Payload**：实际数据

#### 3) 为什么不直接用裸 TCP

| 问题 | WebSocket 的解决 |
|---|---|
| 防火墙/代理拦截非 HTTP 流量 | 借 HTTP 握手，走 80/443 端口，穿透性好 |
| 裸 TCP 没有消息边界 | 帧格式自带长度，天然分包 |
| 浏览器不能直接用 TCP | WebSocket 是浏览器原生支持的 API |

### 三者本质对比

```
HTTP：      一条 TCP 连接，一问一答，答完可以关
SSE：       一条 TCP 连接，HTTP 响应不关，服务端持续写纯文本
WebSocket： 一条 TCP 连接，HTTP 握手后升级，双向收发二进制帧
```

**一句话总结：WebSocket = HTTP 握手 + TCP 长连接 + 自定义二进制帧协议。**
