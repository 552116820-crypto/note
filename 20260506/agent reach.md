• 我查到的更可能是 Agent Reach，不是 “Agency Reach”。

  它是一个给 AI Agent 用的 联网/检索 Skill + CLI 脚手架。核心作用是让 Claude Code、OpenClaw、Cursor、Windsurf 这类
  coding agent 能通过命令行去读和搜互联网内容，比如网页、Twitter/X、Reddit、YouTube 字幕、GitHub、B站、小红书、微信公众
  号、微博、RSS 等。

  它不是一个大模型，也不是浏览器，而是把一堆上游工具装好并写进 SKILL.md，让 Agent 知道该用什么命令。例如：

  - 读网页：通过 Jina Reader
  - 搜 GitHub：gh search repos
  - 读 GitHub 仓库：gh repo view
  - 提取 YouTube 字幕：yt-dlp

  比较实用的场景：

  - “帮我总结这个 YouTube 视频”
  - “去 Reddit/Twitter 搜这个产品的评价”
  - “看看这个 GitHub repo 是干什么的”
  - “搜一下小红书/微博上的口碑”
  - “监控 RSS 或网页更新”
  - “做跨平台资料收集和市场研究”

  需要注意的是，它会让 Agent 执行 shell 命令、安装依赖，并可能保存平台 cookie/token 到本地配置。官方 README 也提醒：
  Twitter、小红书这类 cookie 登录平台有封号和凭据风险，建议用专用小号，不要用主账号。安装前最好先用 safe/dry-run 模式看
  它会做什么。

  来源：

  - GitHub 项目：Panniantong/Agent-Reach (https://github.com/Panniantong/Agent-Reach)
  - ClawHub 技能页：Agent Reach (https://clawhub.ai/panniantong/agent-reach)
  - Playbooks 说明：agent-reach skill (https://playbooks.com/skills/openclaw/skills/agent-reach)


› safe/dry-run 模式 这是什么


• safe 和 dry-run 是安装/运维工具里常见的两个保护模式：

  dry-run：只预演，不真正执行
  它会告诉你“如果真的运行，我会安装哪些包、改哪些文件、写哪些配置”，但不会实际改系统。适合先审查风险。

  agent-reach install --dry-run

  safe：安全安装模式
  它通常不会自动安装系统级依赖或大幅修改环境，只列出需要你手动确认/安装的东西。适合生产机、公司电脑、多人共用机器，或者
  你不想让 Agent 自己乱装东西。

  agent-reach install --safe

  也可以组合：

  agent-reach install --safe --dry-run

  对 Agent Reach 来说，这两个模式主要是为了避免 Agent 自动执行 pip install、装 Node/gh/mcporter/twitter-cli 等依赖、修改
  skill 目录或 MCP 配置时超出你的预期。先 dry-run 看清楚，再决定是否真正安装，会更稳妥。
