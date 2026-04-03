截至 2026 年 3 月 30 日，Anthropic 官方文档里 Claude Opus 4.6 的 API 模型 ID 是 `claude-opus-4-6`。你要走 Anthropic 原生接口，直接用 `Messages API`：

```bash
export ANTHROPIC_API_KEY="你的_api_key"

curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-opus-4-6",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": "用中文解释一下 TCP 三次握手"
      }
    ]
  }'
```

Python 这样写：

```python
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

resp = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "用中文解释一下 TCP 三次握手"}],
)

print(resp.content[0].text)
```

Node.js / TypeScript 这样写：

```ts
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

const resp = await client.messages.create({
  model: "claude-opus-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "用中文解释一下 TCP 三次握手" }],
});

console.log(resp.content);
```

如果你要用 Opus 4.6 官方推荐的思考模式，可以额外加：

```json
"thinking": { "type": "adaptive" },
"output_config": { "effort": "medium" }
```

补充两点：
- `thinking: {type: "enabled", budget_tokens: ...}` 在 Opus 4.6 上已经被官方标记为弃用，推荐改成 `adaptive`。
- 如果你要开 1M 上下文窗口，需要加请求头 `anthropic-beta: context-1m-2025-08-07`；官方说明这是 beta，通常要求 usage tier 4 或自定义限额。标准上下文是 200K。
- 如果你做网页前端，不要把 `ANTHROPIC_API_KEY` 直接放浏览器里，应该由你自己的后端转发请求。

官方文档：
- 快速开始: https://platform.claude.com/docs/zh-CN/get-started
- Claude 4.6 新特性: https://platform.claude.com/docs/zh-CN/about-claude/models/whats-new-claude-4-6
- 1M 上下文窗口: https://platform.claude.com/docs/zh-CN/build-with-claude/context-windows
- TypeScript SDK: https://platform.claude.com/docs/en/api/sdks/typescript

如果你告诉我你现在用的是 `Python / Node.js / Java / Go / curl / FastAPI / Next.js`，我可以直接给你一份能跑的最小项目示例。
