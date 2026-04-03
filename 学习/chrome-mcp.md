● chrome-devtools-mcp 是一个基于 MCP（Model Context Protocol） 协议的服务器，它将 Chrome 浏览器的 DevTools 功能暴露给 AI 代理（如 Claude Code）使用。

  核心功能：
  - 通过 Chrome DevTools Protocol (CDP) 连接并控制 Chrome 浏览器
  - 让 AI 能够执行浏览器操作：导航网页、点击元素、截图、读取 DOM、查看控制台日志、监控网络请求等
  - 实现 AI 驱动的浏览器自动化和网页调试

  典型用途：
  - AI 辅助前端调试（查看页面渲染、检查元素、排查 CSS/JS 问题）
  - 自动化 E2E 测试
  - 网页截图和视觉验证
  - 抓取页面内容辅助开发

  工作原理：
  Claude Code  ──MCP协议──>  chrome-devtools-mcp server  ──CDP──>  Chrome 浏览器

  它本质上是一个桥梁，把 Chrome DevTools 的能力包装成 MCP 工具，让 AI 可以像开发者使用 F12 一样操控浏览器。