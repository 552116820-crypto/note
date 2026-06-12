# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `664d77bf-00f5-4ad9-ad29-4c9c6f4f3a73`
- **范围**: Turn 1-3
- **导出时间**: 2026-06-11 18:02:48

---

### Turn 1  *(2026-06-11 09:39)*

**User:**

computer-use

---

### Turn 2  *(2026-06-11 09:40)*

**User:**

computer-use 怎么用

**Assistant:**

**Computer use** 是 Anthropic 的 beta 能力：Claude 看截图、发出鼠标/键盘指令来操作桌面。关键设计是 **Anthropic 只定义工具 schema，动作由你的代码在你的环境里执行**（client-side tool）。用它有两条路径：

---

## 路径 A：API 自己搭（CCA-F 考点密集区）

**版本与 header**（成对记）：

| Tool type | Beta header | 支持模型 |
|---|---|---|
| `computer_20251124`（新） | `computer-use-2025-11-24` | Opus 4.8/4.7/4.6、Sonnet 4.6、Opus 4.5 |
| `computer_20250124`（旧，未废弃） | `computer-use-2025-01-24` | Sonnet 4.5、Haiku 4.5、Opus 4.1 及更早 |

⚠️ Fable 5 **没有**出现在 computer use 官方支持列表里。

**动作空间**：`screenshot` / `left_click` / `type` / `key` / `scroll` / `left_click_drag` / `double_click` / `hold_key` / `wait` 等；`computer_20251124` 新增 `zoom`（看某区域的全分辨率细节，需在工具定义里加 `enable_zoom: true`）。

**你要自己实现的 agent loop**：收到 `tool_use` → 提取 action → 在 VM/容器里执行 → 截图/结果包成 `tool_result` 返回 → 循环直到 `stop_reason != "tool_use"`。

```python
client.beta.messages.create(
    model="claude-opus-4-8",
    tools=[{
        "type": "computer_20251124",
        "name": "computer",
        "display_width_px": 1024,
        "display_height_px": 768,
    }],
    betas=["computer-use-2025-11-24"],
    messages=[...],
)
```

**最快上手 = 官方 quickstart**（含虚拟 X11 桌面 + Firefox + agent loop + Web UI），WSL2 里装了 Docker 就能跑：

```bash
export ANTHROPIC_API_KEY=<key>
docker run \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -v $HOME/.anthropic:/home/computeruse/.anthropic \
  -p 5900:5900 -p 8501:8501 -p 6080:6080 -p 8080:8080 \
  -it ghcr.io/anthropics/anthropic-quickstarts:computer-use-demo-latest
```

打开 `http://localhost:8080` 即可对话+看 Claude 操作桌面。

---

## 路径 B：Claude Code 桌面版内置（research preview）

桌面 app（macOS/Windows）已内置 computer use，**需 Pro/Max 订阅**：Settings → General → 打开 **Computer use** 开关（macOS 还要授予辅助功能+录屏权限）。

- 工具优先级：Connectors → Bash → Chrome 集成 → computer use 兜底
- 按 app 分级且不可改：浏览器/交易软件**只读**，终端/IDE **只能点击**，其余全控制
- **CLI、云会话、Linux 都没有这功能** —— 你现在 WSL2 终端里的这个会话就用不了；要体验得开 Windows 桌面版 app

## 你这个会话里的现状

我这里没有 computer use 工具。要自动化操作的话，最接近的是浏览器层：playwright MCP（操作）/ chrome-devtools MCP（诊断），桌面级别做不到。

---

## 安全与成本要点（考试可能碰）

- **必须沙箱**：专用 VM/容器、最小权限；别给凭据；高风险操作（交易、同意条款）要人工确认
- **Prompt injection 防御**：模型经过抗注入训练 + 自动分类器扫描截图，可疑时会停下来要确认
- 截图按正常 image input token 计费，无额外费用；推荐 1024×768（XGA）分辨率
- 文档：[computer-use-tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use-tool) ｜ [quickstart repo](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)

你是想跑 quickstart demo 玩一下，还是为 CCA-F 备考整理知识点？前者我可以现在帮你检查 Docker 环境直接跑起来。

---

### Turn 3  *(2026-06-11 09:50)*

**User:**

https://dify.amer.cn/tools?category=mcp,我不知道为什么我明明配置了mcp，但在搭建工作流的添加节点的时候，看不见这个mcp

**Assistant:**

先说结论：在 Dify 里「Tools 页面有这张 MCP 卡片」≠「工作流里能用」，中间还有几道关卡。按命中概率从高到低排查：

## 1. 点开那张 MCP 服务器卡片，看它到底拉到工具没有

这是最常见的原因：服务器加上了，但**工具列表是空的或授权没完成**。卡片上应该显示「N 个工具」；点进去如果是 0 个、或显示「未授权/未配置」，工作流里自然什么都看不到。

- 点卡片 → **「更新工具」**（Update Tools）强制重新拉取
- 有 OAuth 的服务器要走完 **重新授权**

注意一个大坑：拉工具列表是 **Dify 服务端**发起的连接，不是你浏览器。如果 dify.amer.cn 是 docker 部署，而 MCP server 填的是 `127.0.0.1`/`localhost`，那指向的是容器自己，必然拉不到 —— 要改成内网 IP 或 `host.docker.internal`。

## 2. 在工作流编辑器里找对位置

MCP 工具**不在「节点」列表里**，在画布上点 `+` 后：

```
添加节点弹窗 → 切到「工具」tab → 子分类里选「MCP」
```

Agent 节点则是在节点配置面板的「工具列表」里添加，同样进那个弹窗选 MCP 分组。很多人翻遍「节点」tab 找不到就以为没生效。

## 3. 强刷编辑器页面

如果工作流编辑器是在你配置 MCP **之前**就打开的，工具面板用的是旧缓存 —— `F5` 刷新再看。

## 4. 检查 Dify 版本 ≥ 1.6.0

内置 MCP（Tools 页的 MCP 分类 + 工作流独立 MCP 节点）是 **v1.6.0（2025 年 7 月）** 才有的。头像菜单 → 「关于」可以看版本。如果你们公司这套是老版本，那个页面上的 MCP 可能是 `MCP SSE` 插件 —— 插件方案**不会**以独立工具节点出现，只能在 Agent 节点里配合「Agent 策略(支持 MCP 工具)」插件使用。

## 5. 确认同一工作空间

MCP 服务器是 workspace 级配置，配 MCP 和建工作流必须在同一个空间。

---

如果这台 Dify 是你自己部署的，还可以直接看日志定位拉取失败原因（SSRF 代理拦截内网地址是 docker 部署的常见坑）：

```bash
docker logs -f docker-api-1 2>&1 | grep -i mcp
```

你先告诉我两件事，我帮你锁定原因：**① 点开卡片后显示几个工具、什么状态？② Dify 版本号是多少？** 或者如果方便，我也可以起个浏览器窗口，你登录一下 dify.amer.cn 我直接上去看。

---
