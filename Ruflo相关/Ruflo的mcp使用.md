用户：

ruflo mcp怎么大，是怎么塞进模型上下文的

助手：

先分清两个“`大`”：

`Ruflo` 的确大，但有两种不同含义。

- 安装包大：官方 README 写的是默认安装大约 `340MB`，`--omit=optional` 的精简安装大约 `45MB`，因为它把 embeddings / ML 相关依赖也带上了。[Ruflo README](https://raw.githubusercontent.com/ruvnet/ruflo/main/README.md)
- 工具面大：Ruflo 官方 README 现在直接写“`310+ MCP tools`”，再加上 `100+` agents、`137+` skills，以及可选插件会继续加工具。[Ruflo README](https://raw.githubusercontent.com/ruvnet/ruflo/main/README.md)

但这不等于“把 340MB 或 310 多个工具一次性全文塞进模型上下文”。

**它怎么进上下文**
MCP 协议里，client 先通过 `tools/list` 拿到工具定义；工具定义主要是这些元数据：

- `name`
- `description`
- `inputSchema`
- `outputSchema`

也就是“工具说明书”，不是工具实现源码本身。[MCP Tools Spec](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)

Claude Code 对“大 MCP”用的是按需加载，不是全量预灌。官方文档写得很明确：

- 当 MCP 工具描述会吃掉超过 `10%` 上下文窗口时，会自动启用 `Tool Search`
- 启用后，工具先被延迟，不预加载
- Claude 先搜索相关工具
- 只把实际需要的工具加载进上下文
- 这个能力默认开启；关闭后才会预加载全部工具定义  
  [Claude Code MCP Docs](https://code.claude.com/docs/zh-CN/mcp)

所以对 Ruflo 这种超大 MCP，实际机制更像：

```text
Ruflo server
  -> 暴露很多工具定义
Claude Code
  -> 建索引/搜索
  -> 只在需要时把相关工具定义给模型
模型
  -> 选择工具
Claude Code
  -> 调 tools/call
  -> 把返回结果放回对话
```

协议层面 `tools/list` 本身也支持分页，所以超大工具集不必一次返回完。[MCP Pagination Spec](https://modelcontextprotocol.io/specification/2025-11-25/server/utilities/pagination)

**真正占上下文的是哪部分**
真正持续吃上下文的主要是两类：

- 工具定义元数据：名称、描述、schema
- 工具调用结果：也就是 tool output

而 tool output 这块，Claude Code 也有限制：

- 超过 `10,000` tokens 会警告
- 默认上限 `25,000` tokens
- 可用 `MAX_MCP_OUTPUT_TOKENS` 调整  
  [Claude Code MCP Docs](https://code.claude.com/docs/zh-CN/mcp)

另外：

- MCP `resources` 不是默认全塞进来，只有你用 `@server:...` 引用时才会抓取并作为附件加入
- MCP `prompts` 也是在你执行 `/mcp__server__prompt` 时才注入对话  
  [Claude Code MCP Docs](https://code.claude.com/docs/zh-CN/mcp)

**一句话结论**
Ruflo 的“大”，更多是“磁盘包体大 + 工具目录大”；进模型上下文时，Claude Code 主要靠 `Tool Search + 按需加载 + 输出上限` 来控制，不是把整个 Ruflo 一股脑塞进 prompt 里。

如果你愿意，我下一条可以继续给你拆：
`Claude Code 里一个 MCP 工具从注册到被模型调用，完整经过哪些 block / message`。

用户：

愿意

助手：

可以。先给你一句最实用的结论：

**Ruflo 这种“大 MCP”并不是把所有工具源码塞进模型上下文，而是把“工具定义”按需提供给 Claude Code；真正进上下文的主要是工具元数据和实际调用结果。**

**完整链路**
```text
1. 你配置 Ruflo MCP
   claude mcp add ... ruflo -- npx -y ruflo@latest mcp start

2. Claude Code 启动 Ruflo server
   stdio / http 连接建立

3. MCP 握手
   initialize -> initialized
   协商 server 支持 tools / resources / prompts 等能力

4. 工具发现
   Claude Code -> tools/list
   Ruflo -> 返回工具定义(name/description/inputSchema/...)

5. 上下文装载
   小工具集：直接预加载工具定义
   大工具集：启用 Tool Search，只按需把相关工具定义放进上下文

6. 模型决策
   模型基于：
   - 用户问题
   - 当前已加载的工具定义
   - 已引用的资源/提示
   决定要不要调用某个 Ruflo 工具

7. 真正调用
   Claude Code -> tools/call(name, arguments)
   Ruflo 执行

8. 结果回填
   Ruflo -> content / structuredContent / isError
   Claude Code 把结果作为对话内容回填给模型继续推理
```

**哪些部分是官方能确认的**
MCP 协议层面，client 会先 `initialize`，再 `initialized`，之后通过 `tools/list` 发现工具，通过 `tools/call` 调用工具。这是标准流程。  
来源：
- [MCP Lifecycle](https://modelcontextprotocol.io/specification/latest/basic/lifecycle)
- [MCP Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)

`tools/list` 返回的不是实现代码，而是工具定义，核心字段是：
- `name`
- `description`
- `inputSchema`
- 可选 `outputSchema`

来源：
- [MCP Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)

而 Claude Code 官方明确写了：

- 当 MCP 工具描述会占用超过 `10%` 上下文窗口时，会自动启用 `Tool Search`
- 工具会被“延迟”而不是预加载
- Claude 会先搜索相关工具
- 只把真正需要的工具加载进上下文
- 这个能力默认开启
- 文档还明确提到了 `tool_reference` blocks

来源：
- [Claude Code MCP 文档](https://code.claude.com/docs/zh-CN/mcp)

**所以“怎么塞进上下文”的准确说法**
严格说，官方没有公开 Claude Code 给模型拼 prompt 的完整内部格式。  
但从官方文档能确定的是：

- 小 MCP：工具定义可能预先进入上下文
- 大 MCP：先不全塞
- Claude 先通过搜索找到相关工具
- 再把这些工具以某种 `tool_reference` / 工具定义块的形式提供给模型
- 模型选中后，Claude Code 再发起 `tools/call`

这里“某种块的具体序列化格式”属于内部实现细节，官方没完全展开；“按需加载 + tool_reference”是文档明确写了的。

**真正吃 token 的是这几样**
- 工具定义的描述和 schema
- 用户显式 `@` 引用的 MCP resources
- 你执行 `/mcp__server__prompt` 后注入的 MCP prompts
- 工具调用结果

其中：
- `@resource` 只有在引用时才会自动抓取并作为附件加入
- `/mcp__...` prompt 结果会直接注入对话
- MCP tool output 超过 `10,000` tokens 会警告，默认最大 `25,000` tokens

来源：
- [Claude Code MCP 资源](https://code.claude.com/docs/zh-CN/mcp)
- [Claude Code MCP prompts](https://code.claude.com/docs/zh-CN/mcp)
- [Claude Code MCP 输出限制](https://code.claude.com/docs/zh-CN/mcp)

**回到 Ruflo**
Ruflo README 现在直接写了“`310+ MCP tools`”。这意味着它更依赖 Claude Code 的 Tool Search 机制，而不是全量 preload。否则上下文会很快被工具定义吃掉。  
来源：
- [Ruflo README](https://raw.githubusercontent.com/ruvnet/ruflo/main/README.md)

如果你要，我下一条可以继续画一张更细的图：  
`用户一句话 -> Claude Code -> MCPSearch -> Ruflo工具被选中 -> tools/call -> 结果回填 -> 下一轮推理` 的消息级时序图。
