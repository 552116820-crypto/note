# Figma MCP 工具能给到模型什么能力

MCP（Model Context Protocol）是让 AI 模型直接连接外部工具的协议。Figma MCP 让模型**不再依赖截图**，而是能直接读取 Figma 设计文件的结构化数据。

## 核心能力

| 能力 | 说明 |
|------|------|
| **读取设计文件** | 获取文件中所有页面、画框、图层的结构树 |
| **获取元素属性** | 精确拿到每个元素的尺寸、颜色、字体、间距、圆角等数值 |
| **获取组件信息** | 识别哪些是组件、组件的变体（Variants）、属性 |
| **获取样式/Token** | 读取设计系统中定义的颜色、文字、效果等样式 |
| **导出图片** | 将指定节点导出为 PNG/SVG 等格式 |
| **读取布局信息** | Auto Layout 的方向、间距、对齐方式等 |

## 截图 vs MCP 的区别

```
截图方式：
  Figma 设计 → 截图(PNG) → AI 看图猜测 → 生成代码
  问题：颜色不精确、间距靠猜、字体可能认错

MCP 方式：
  Figma 设计 → API 结构化数据(JSON) → AI 精确读取 → 生成代码
  优势：颜色值精确、间距精确、字体/组件信息完整
```

## 实际给模型的数据示例

模型通过 MCP 拿到的是类似这样的结构化信息：

```json
{
  "name": "登录按钮",
  "type": "FRAME",
  "width": 320,
  "height": 48,
  "cornerRadius": 8,
  "fills": [{ "type": "SOLID", "color": "#4F46E5" }],
  "children": [{
    "type": "TEXT",
    "content": "登 录",
    "fontSize": 16,
    "fontFamily": "Inter",
    "fontWeight": 600,
    "color": "#FFFFFF"
  }],
  "layoutMode": "HORIZONTAL",
  "primaryAxisAlignItems": "CENTER",
  "counterAxisAlignItems": "CENTER",
  "paddingLeft": 24,
  "paddingRight": 24
}
```

有了这些数据，模型生成的代码**精度远高于看截图**。

## 怎么用

目前主流的 Figma MCP 方案：

1. **Figma 官方 Dev Mode MCP** — Figma 官方提供的 MCP server
2. **社区方案** — 如 `figma-mcp-server`，通过 Figma REST API 实现

配置方式大致是在 Claude Code 的 MCP 设置中添加：

```json
{
  "mcpServers": {
    "figma": {
      "command": "npx",
      "args": ["-y", "figma-mcp-server"],
      "env": {
        "FIGMA_ACCESS_TOKEN": "你的Figma API Token"
      }
    }
  }
}
```

配置完成后，你只需要把 Figma 文件链接发给 Claude，模型就能直接读取设计数据并生成精准的前端代码，不再需要手动截图。

---

**总结**：MCP 把 Figma 从"给 AI 看图" 升级为 "给 AI 读数据"，代码还原精度有质的提升。
