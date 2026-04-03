## sub2api介绍一下

## Sub2API 介绍

**Sub2API** (又名 CRS2) 是一个开源的 **一站式 AI API 中转网关平台**，由 [Wei-Shaw](https://github.com/Wei-Shaw/sub2api) 开发。

### 核心功能

将 AI 订阅服务（Claude Pro、OpenAI Plus、Gemini 等）转换为标准 API 格式，支持**多人共享（拼车）**，高效分摊成本。

### 主要特性

| 功能 | 说明 |
|---|---|
| 多账号管理 | 支持 OAuth 和 API Key 上游认证 |
| API Key 分发 | 为下游用户生成独立 API Key |
| 用量追踪 | Token 级精确计费 |
| 智能调度 | 粘性会话 + 智能账号选择 |
| 并发控制 | 按用户/账号级别限流 |
| Web 管理面板 | 可视化管理后台 |

### 支持的 AI 服务

- **Claude** (Anthropic)
- **OpenAI**
- **Gemini** (Google)
- **Antigravity**

### 技术栈

- **后端**: Go + Gin + Ent ORM
- **前端**: Vue 3 + Vite + TailwindCSS
- **数据库**: PostgreSQL 15+
- **缓存**: Redis 7+

### 部署方式

1. 一键脚本安装（推荐生产环境）
2. Docker Compose
3. 源码编译

### 生态

- **Sub2ApiPay** — 自建支付网关，支持支付宝、微信、Stripe
- **sub2api-mobile** — iOS/Android 管理端
- **PinCC** (pincc.ai) — 基于 Sub2API 的官方托管中转服务

### 历史

Sub2API 是前身项目 **Claude Relay Service (CRS)** 的第二代重构版本，GitHub 约 9.2k Stars。

简单来说：它让你可以把一个 AI 订阅账号通过 API 的方式分享给多人使用，实现成本分摊。

---

## 生产环境和开发环境有什么不同

## 通用区别

| 维度 | 开发环境 (Development) | 生产环境 (Production) |
|---|---|---|
| **目的** | 编写、调试、测试代码 | 面向真实用户提供服务 |
| **稳定性** | 可以随时重启、崩溃无所谓 | 必须高可用、稳定运行 |
| **数据** | 测试数据，可随意清除 | 真实用户数据，不可丢失 |
| **性能** | 不关注性能 | 需要优化、监控 |
| **安全** | 宽松配置，方便调试 | 严格配置，防攻击 |
| **日志** | 详细 debug 日志 | 精简日志 + 告警 |
| **部署方式** | 手动运行/热重载 | 自动化部署、进程守护 |

## Sub2API 具体区别

### 开发环境（源码编译）
- 直接 `go run` 或编译运行后端
- `npm run dev` 启动前端热重载
- 手动配置 PostgreSQL、Redis
- 方便改代码、调试、看日志
- 适合二次开发或贡献代码

### 生产环境

**方式一：一键脚本（推荐）**
- 自动安装依赖、配置数据库、设置 systemd 守护进程
- 服务崩溃自动重启
- 通常搭配 Nginx 反向代理 + SSL 证书

**方式二：Docker Compose**
- 一条命令拉起所有服务（后端、前端、PostgreSQL、Redis）
- 容器隔离，环境一致性好
- 方便迁移和扩展

### 关键差异总结

```
开发环境:  源码 → 手动启动 → 本地访问 → 随时改代码
生产环境:  编译/镜像 → 进程守护 → 域名+HTTPS → 稳定对外服务
```

生产环境额外需要关注的：
- **数据备份**（PostgreSQL 定时备份）
- **HTTPS 证书**（Let's Encrypt 等）
- **反向代理**（Nginx/Caddy）
- **监控告警**（服务挂了能及时发现）
- **资源限制**（防止单用户耗尽资源）

---


---

