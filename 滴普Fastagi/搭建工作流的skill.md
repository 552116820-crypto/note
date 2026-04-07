# 搭建 Dify 工作流的 Skill

## 用户提问

搭建dify工作流的skill，有这种skill吗，你能直接在网页console帮我搭建工作流吗，还是给我dsl文件

## Claude 回答

目前没有专门的 Dify 工作流 skill。

但我有两种方式可以帮你：

1. **直接在网页操作** — 我有 Playwright 和 Chrome DevTools 浏览器自动化工具，可以帮你在 Dify 网页界面上点击、拖拽、填写来搭建工作流。前提是你本地能打开 Dify 的页面。

2. **生成 DSL 文件** — Dify 支持 YAML 格式的 DSL 导入，我可以根据你的需求直接生成 `.yml` 文件，你在 Dify 里导入即可。这种方式更快、更可控。

**推荐方案 2**（DSL 文件），原因：
- 网页操作依赖页面加载、元素定位，容易出错
- DSL 文件可版本控制、可复用、可批量修改

你想搭建什么样的工作流？告诉我需求，我来生成 DSL。

---

## 用户提问

你应该可以用控制台API 级编辑工作流图啊

## Claude 回答

你说得对，Dify 有 REST API 可以直接操作工作流。我可以通过 API 调用来创建/编辑工作流，不需要在网页上点来点去。

不过我需要先确认几个信息：

1. **Dify 地址** — 你的 Dify 是本地部署的（如 `http://localhost/v1`）还是云端的？
2. **API Key** — 需要你提供 Dify 的 API Key（通常在 Dify 后台设置里获取）
3. **你想搭建什么工作流** — 描述一下具体需求，比如输入输出是什么、中间需要哪些节点（LLM、知识库检索、代码执行、条件分支等）

有了这些我就可以直接通过 API 帮你创建工作流。

---

## 用户提问

既然没有搭建工作流的skill，那我可以自己开发一个专门用于搭建滴普智能体的skill吗

## Claude 回答

完全可以。下面是完整方案：

### 1. 开发 Skill

在 `~/.claude/skills/dify-workflows/` 下创建 `SKILL.md`：

```markdown
---
name: dify-workflows
description: 通过 Dify API 创建、部署和管理 AI 智能体工作流。当用户需要搭建 Dify 工作流时自动触发。
argument-hint: [action] [workflow-name]
allowed-tools: Bash Read Write Grep WebFetch
---

# Dify 智能体工作流构建器

你是 Dify 工作流设计专家，通过 REST API 帮用户搭建工作流。

## API 配置
- Base URL: 环境变量 $DIFY_API_BASE
- API Key: 环境变量 $DIFY_API_KEY

## 支持的操作
- create: 创建工作流
- deploy: 发布工作流  
- list: 列出现有工作流
- update: 更新工作流
- test: 测试工作流

（这里填入具体的 Dify API 调用逻辑、节点模板、DSL 结构等）
```

你还可以在同目录下放模板和脚本：

```
~/.claude/skills/dify-workflows/
├── SKILL.md              # 主指令
├── templates/             # 工作流模板
│   ├── chatbot.json
│   └── multi-agent.json
└── scripts/
    └── deploy.sh          # 部署脚本
```

### 2. 共享给其他部门

有三种方式，按推荐程度排列：

#### 方式一：打包成 Plugin（推荐）

```
dify-workflows-plugin/
├── .claude-plugin/
│   └── plugin.json        # 插件元信息
├── skills/
│   └── dify-workflows/
│       └── SKILL.md
└── README.md
```

`plugin.json`：
```json
{
  "name": "dify-workflows",
  "description": "Dify 智能体工作流构建工具",
  "version": "1.0.0",
  "author": { "name": "你的团队" }
}
```

把这个目录推到公司内部 Git 仓库，其他人在项目的 `.claude/settings.json` 里添加：

```json
{
  "plugins": ["https://git.yourcompany.com/dify-workflows-plugin"]
}
```

#### 方式二：放到共享项目的 `.claude/skills/` 目录

如果大家用同一个代码仓库，直接把 skill 放在项目的 `.claude/skills/dify-workflows/` 下，所有协作者自动可用。

#### 方式三：`--add-dir` 指向共享目录

把 skill 放在网络共享盘或同步文件夹（如 OneDrive/NAS），其他人启动时：

```bash
claude --add-dir /path/to/shared-skills
```

**对于销售等非技术部门**，Plugin 方式最友好 — 他们只需一次配置，之后直接输入 `/dify-workflows 创建一个客户跟进的智能体` 就能用，不需要懂 API 细节。

---

## 用户提问

这样可以把开发好的skill共享到销售和其他部门使用

## 用户提问

打包成 Plugin是什么意思

## Claude 回答

简单说，**Plugin 就是一个带特定目录结构的文件夹**，让 Claude Code 能识别并加载里面的 skill。

### 对比理解

| | 普通 Skill | Plugin |
|---|---|---|
| **本质** | 一个 `SKILL.md` 文件 | 一个包含 skill 的**文件夹包** |
| **位置** | 放在自己电脑 `~/.claude/skills/` | 放在 Git 仓库或共享目录 |
| **谁能用** | 只有你自己 | 任何人引用就能用 |
| **安装方式** | 手动创建 | 别人在配置里加一行地址 |

### Plugin 其实就多了一个文件

普通 skill：
```
dify-workflows/
└── SKILL.md
```

Plugin 只是在外面**套了一层壳**：
```
dify-workflows-plugin/
├── .claude-plugin/
│   └── plugin.json      ← 就多了这个，声明"我是一个插件"
└── skills/
    └── dify-workflows/
        └── SKILL.md      ← 内容和普通 skill 一模一样
```

`plugin.json` 内容很简单，就是名字和版本：
```json
{
  "name": "dify-workflows",
  "description": "Dify 智能体工作流构建工具",
  "version": "1.0.0"
}
```

### 别人怎么用

1. 你把这个文件夹推到公司 Git 仓库
2. 同事在自己的项目配置里加一行引用地址
3. 他们的 Claude Code 就自动加载了你写的 skill

**类比理解**：
- Skill = 你写在自己笔记本上的操作手册
- Plugin = 你把手册印成册子放到公司共享盘，谁需要谁拿

就这么简单，没有编译、没有打包、没有发布流程，本质上就是一个**约定了目录结构的共享文件夹**。
