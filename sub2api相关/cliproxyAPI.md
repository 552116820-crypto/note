
## cliproxyAPI也介绍一下

## CLIProxyAPI 介绍

**CLIProxyAPI** 是由 [router-for-me](https://github.com/router-for-me/CLIProxyAPI) 组织开发的开源项目，核心思路是：

> 把多个 AI CLI 工具（Claude Code、Gemini CLI、ChatGPT Codex、Qwen Code 等）包装成**统一的标准 API 服务**，通过 OAuth 认证而非付费 API Key 来访问。

### 核心原理

```
AI CLI 工具（OAuth 登录订阅账号）
        ↓
   CLIProxyAPI（中转层）
        ↓
  标准 API 接口（OpenAI/Gemini/Claude 兼容）
        ↓
  任意支持标准 API 的客户端/应用
```

### 支持的上游 CLI 工具 & 模型

| CLI 工具 | 可用模型 | OAuth 端口 |
|---|---|---|
| Claude Code | claude-sonnet-4, claude-opus-4 | 54545 |
| Gemini CLI | gemini-2.5-pro | 8085 |
| ChatGPT Codex | gpt-5-codex, gpt-5 | 1455 |
| Qwen Code | Qwen 系列 | — |
| Antigravity | — | — |
| iFlow | — | — |

### 主要特性

- **多协议兼容** — 一个接口支持 Chat Completions / Response / Gemini / Claude 协议
- **OAuth 认证** — 用订阅账号登录，无需购买 API Key
- **Go 语言编写** — 高性能、轻量
- **多种部署方式** — Homebrew / Docker / AUR / 源码编译 / Zeabur 一键部署

### 生态工具

| 项目 | 说明 |
|---|---|
| **CLIProxyAPIPlus** | 社区扩展版，支持更多第三方 provider |
| **Management Center** | 单文件 Web UI，管理配置、健康检查、实时日志、请求统计、配额追踪 |
| **EasyCLI** | Windows 桌面 GUI（Tauri v2），可视化管理本地/远程实例 |
| **VSCode 扩展** | 快速切换 Claude Code 模型，支持后台自动启停 |

### 与 Sub2API 的对比

| 维度 | Sub2API | CLIProxyAPI |
|---|---|---|
| **上游来源** | OAuth 账号 + API Key | **CLI 工具**（OAuth） |
| **核心思路** | 订阅账号 → API | CLI 工具 → API |
| **技术栈** | Go + Vue | Go |
| **拼车/多用户** | 内建支持，有计费系统 | 侧重单用户/自用 |
| **管理面板** | 内建 | 独立项目 (Management Center) |
| **适用场景** | 多人共享分摊成本 | 个人将 CLI 转 API 使用 |

简单来说：**Sub2API 偏向"合租平台"**，CLIProxyAPI 偏向**"把 CLI 工具变成 API 服务"**，两者侧重点不同但目标类似——让 AI 订阅以标准 API 形式使用。

---

## "把 CLI 工具变成 API 服务"是什么意思？

**CLIProxyAPI** 的核心思路是：

### CLI → API 的代理层

把一个**命令行工具（CLI）**包装成一个**HTTP API 服务**，让原本只能在终端里敲命令使用的工具，变成可以通过网络请求调用的接口。

### 举个例子

假设你有一个 CLI 工具 `ffmpeg`：

```bash
# 原本只能在终端用
ffmpeg -i input.mp4 -vf scale=720:480 output.mp4
```

加一层 CLIProxyAPI 后：

```http
POST /api/convert
{
  "input": "input.mp4",
  "scale": "720:480"
}
```

API 服务在后台帮你拼接并执行对应的 CLI 命令，再把结果返回。

### 为什么要这么做？

| 动机 | 说明 |
|------|------|
| **远程调用** | 不需要在本地安装 CLI，通过网络就能用 |
| **集成方便** | 前端/其他服务可以直接 HTTP 调用，不用 `exec` 子进程 |
| **权限控制** | API 层可以加认证、限流、日志 |
| **统一接口** | 多个 CLI 工具统一成 REST/gRPC 风格的 API |

### 简单架构

```
客户端 → HTTP 请求 → [CLIProxyAPI 服务] → 拼接命令 → 执行 CLI → 返回结果
```

本质就是一个**代理（Proxy）**，把 API 请求翻译成 CLI 命令执行。