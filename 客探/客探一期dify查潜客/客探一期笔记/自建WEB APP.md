• 要做到“0 点击自动开始对话”，核心思路是：别用现成的 /chat/<token> WebApp 页面，而是你自己做一个前端页面，页面加载后立刻调用 Dify 的 Service API 发送第一条消
  息（比如“开始”），并把 sales_id 作为 inputs 传进去。

  下面给你一条“最省事、可直接跑起来”的 Next.js 方案（推荐），以及关键点解释。

  1) Dify 端准备

  - 你的 Chatflow 里要有一个输入变量，key 必须叫：sales_id（不是显示名），后续节点里用 {{sales_id}}。
  - 到应用的 API 页面创建/复制 Service API Key（这就是你请求 POST /v1/chat-messages 用的 Bearer key）。

  2) 为什么需要“自建 WebApp + 后端代理”

  - 不能把 Dify 的 API Key 暴露在浏览器里（否则谁打开页面都能扒走 key）。
  - 所以前端只请求你自己的 /api/chat，由你的后端（Next.js API Route）去带着 Dify Key 请求 Dify。

  3) 从零搭一个最小可用的自建 WebApp（Next.js）

  创建项目：

  mkdir dify-autostart-webapp
  cd dify-autostart-webapp
  npx create-next-app@latest . --ts --app --eslint

  配置环境变量：新建 .env.local

  DIFY_BASE_URL=https://dify.amer.cn
  DIFY_API_KEY=你的_Dify_Service_API_Key

  新增一个后端代理接口：app/api/chat/route.ts

  import { gunzipSync } from 'node:zlib'

  function decodeMaybeGzipBase64(input: string) {
    try {
      const normalized = input.replace(/-/g, '+').replace(/_/g, '/')
      const padded = normalized + '='.repeat((4 - (normalized.length % 4)) % 4)
      const raw = Buffer.from(padded, 'base64')
      return gunzipSync(raw).toString('utf8')
    } catch {
      return input
    }
  }

  export async function POST(req: Request) {
    const { query, sales_id, conversation_id, user } = await req.json()

    if (!process.env.DIFY_BASE_URL || !process.env.DIFY_API_KEY) {
      return Response.json({ error: 'Missing DIFY env' }, { status: 500 })
    }

    const sid = decodeMaybeGzipBase64(String(sales_id ?? '')).trim()
    if (!sid) return Response.json({ error: 'Missing sales_id' }, { status: 400 })

    const resp = await fetch(`${process.env.DIFY_BASE_URL}/v1/chat-messages`, {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.DIFY_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        inputs: { sales_id: sid },
        query: String(query ?? '开始'),
        response_mode: 'blocking',
        user: String(user ?? 'anonymous'),
        conversation_id: conversation_id ?? undefined,
      }),
    })

    const data = await resp.json().catch(() => ({}))
    return Response.json(data, { status: resp.status })
  }

  前端页面加载即自动发第一条消息：app/page.tsx

  'use client'

  import { useEffect, useRef, useState } from 'react'

  function getOrCreateUserId() {
    const key = 'dify_user_id'
    let v = localStorage.getItem(key)
    if (!v) {
      v = (crypto?.randomUUID?.() ?? `u_${Date.now()}_${Math.random()}`).toString()
      localStorage.setItem(key, v)
    }
    return v
  }

  export default function Page() {
    const [answer, setAnswer] = useState<string>('')
    const [conversationId, setConversationId] = useState<string | null>(null)
    const started = useRef(false)

    useEffect(() => {
      if (started.current) return
      started.current = true

      const params = new URLSearchParams(window.location.search)
      const salesId = params.get('sales_id') // 支持你原来那种 H4sIA...（后端会解码）
      if (!salesId) {
        setAnswer('URL 缺少 sales_id，比如：?sales_id=000002')
        return
      }

      fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          query: '开始',
          sales_id: salesId,
          user: getOrCreateUserId(),
        }),
      })
        .then((r) => r.json())
        .then((data) => {
          setAnswer(data.answer ?? JSON.stringify(data))
          setConversationId(data.conversation_id ?? null)
        })
        .catch((e) => setAnswer(String(e)))
    }, [])

    return (
      <main style={{ maxWidth: 720, margin: '40px auto', padding: 16, fontFamily: 'system-ui' }}>
        <h1>Dify Auto Start</h1>
        <div style={{ opacity: 0.7, marginBottom: 12 }}>conversation_id: {conversationId ?? '-'}</div>
        <pre style={{ whiteSpace: 'pre-wrap', padding: 12, background: '#f5f5f5', borderRadius: 8 }}>
          {answer || 'Loading...'}
        </pre>
      </main>
    )
  }

  启动：

  npm run dev

  然后访问：

  - http://localhost:3000/?sales_id=000002
  - 或者兼容你现在那种：http://localhost:3000/?sales_id=H4sIAAAAAAAEADMwAAIjAMpkSNcGAAAA（后端会解码成 000002）

  4) 你接下来大概率还需要的两点

  - 继续对话：把你后续用户输入再 POST /api/chat，并带上上一次返回的 conversation_id，这样就是同一段会话。
  - 流式输出：把 response_mode 换成 streaming，前端用 SSE 读流（实现会长一点，我可以按你现在的 UI 形态给你补齐）。

  你希望自建 WebApp 的 UI 是“只要自动开场 + 简单展示答案”就行，还是要做成完整聊天框（消息列表、输入框、流式输出、加载状态）？我可以按你要的样式把最小工程补
  全。



• 不是每一轮都要把 sales_id 当成 inputs 传进去。
  1) 后续每一轮要带什么参数？
  - 你确实需要每一轮都 POST /api/chat（你自己的后端接口），因为每一轮用户发言都要调用一次 Dify 的 POST /v1/chat-messages。
  - 但 sales_id 这种“App 定义的 inputs 变量”，只在开新会话（不传 conversation_id）那一次传就够了。
  - 一旦你在后续请求里传了 conversation_id，Dify 官方说明：新的 inputs 会被忽略，只有 query 会被处理。(docs.dify.ai (https://docs.dify.ai/en/guides/
    application-publishing/developing-with-apis?utm_source=openai))

  所以推荐的请求形态是：

  - 第 1 轮（创建新会话）：
  POST /v1/chat-messages
  {
    "inputs": { "sales_id": "000002" },
    "query": "开始",
    "response_mode": "streaming",
    "user": "u_abc"
  }

  - 第 N 轮（继续同一会话）：

  POST /v1/chat-messages
  {
    "conversation_id": "上一轮返回的conversation_id",
    "query": "继续问问题……",
    "response_mode": "streaming",
    "user": "u_abc"


  - conversation_id：用来续聊；第一轮不传，让 Dify 生成并在响应里返回。(docs.dify.ai (https://docs.dify.ai/en/guides/application-publishing/developing-with-
  - user：必须一直保持同一个值（同一个终端用户），否则 Dify 会把它当成不同用户的会话空间。(docs.dify.ai (https://docs.dify.ai/api-reference?

  2) 那 sales_id 怎么在后续轮次还能用？

  - 你不能靠“每轮重传 inputs”（因为会被忽略）。(docs.dify.ai (https://docs.dify.ai/en/guides/application-publishing/developing-with-apis?utm_source=openai))
  - 正确做法是：第一轮把 sales_id 传进 inputs，并在 Chatflow 里（更稳）用 Variable Assigner 把它写入“会话变量（Conversation Variables）”，这样跨多轮都能读
    到。会话变量就是为 Chatflow 多轮持久化设计的。(docs.dify.ai (https://docs.dify.ai/versions/3-0-x/en/user-guide/workflow/variables?utm_source=openai))

  （如果你不需要中途修改 sales_id，其实只传首轮 inputs 通常也够用；但用会话变量更“确定”。）

  3) 自建 WebApp 是怎么通过 Dify Service API 实现“对话”的？
  自建 WebApp 本质上就是你自己实现一个“聊天 UI + 会话管理”，把每轮对话转成 Dify API 调用：

  - 前端（浏览器）负责：
      - 从 URL 读 sales_id
      - 显示消息列表、输入框
      - 保存 conversation_id（内存/LocalStorage 都行）
      - 每次用户发送消息，就调用你自己的后端 /api/chat
  - 你自己的后端（必须有，别把 Key 放前端）负责：
      - 持有 Dify API_KEY（只放服务端）
      - 转发请求到 Dify：POST /v1/chat-messages（流式 SSE 或 blocking 都可）(docs.dify.ai (https://docs.dify.ai/api-reference?utm_source=openai))
      - 把 Dify 返回的 conversation_id、answer（以及 streaming 的 chunk）回传给前端

  常用配套接口（做完整聊天体验会用到）：

  - 继续对话：POST /v1/chat-messages（带 conversation_id）(docs.dify.ai (https://docs.dify.ai/api-reference?utm_source=openai))
  - 拉历史消息：GET /v1/messages?conversation_id=...&user=... (docs.dify.ai (https://docs.dify.ai/api-reference/conversations/get-conversation-history-
    messages))
  - 停止生成：POST /v1/chat-messages/{task_id}/stop（user 必须一致）(docs.dify.ai (https://docs.dify.ai/api-reference/chat/stop-chat-message-generation?
    utm_source=openai))
  - 列出会话：GET /v1/conversations?user=... (docs.dify.ai (https://docs.dify.ai/api-reference/conversations/get-conversations?utm_source=openai))

  4) 你现在这个 sales_id=H4sIA... 还要不要解码？

  - Dify 现成 WebApp 会把 inputs 做 gzip+base64 这类编码塞进 URL，你自建 WebApp 调 Service API 时一般建议传“真实值”（比如 000002），不建议继续沿用压缩串。
  - 如果你的来源 URL 已经是 H4sIA... 这种，那就在你后端 /api/chat 里先解码成明文再传给 Dify（你之前那串解出来是 000002）。

  如果你告诉我你打算用 streaming 还是 blocking，我可以给你一份“前端完整聊天组件 + 后端 SSE 透传”的最小实现（包含自动首轮发送、conversation_id 持久化、Stop 按
  钮）。
