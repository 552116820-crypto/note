


• 可以。下面给你一份“OpenClaw 配成 Claude Cowork 工作流”的最小可执行清单（基于 2026-03-05 官方文档）。

  目标
  做到这 5 件事：本地文件读写、命令执行可审批、浏览器自动化、周期任务、多代理并行。

  1. 安装并跑起来（Windows）

  iwr -useb https://openclaw.ai/install.ps1 | iex
  openclaw onboard --install-daemon
  openclaw gateway status
  openclaw dashboard

  2. 绑定 Claude 模型（Anthropic）

  # 方案A：API Key
  $env:ANTHROPIC_API_KEY="sk-ant-..."
  openclaw onboard --anthropic-api-key "$env:ANTHROPIC_API_KEY"

  # 方案B：Claude setup-token（订阅体系）
  openclaw models auth setup-token --provider anthropic
  3. 写一份 Cowork 风格最小配置
     配置文件路径：~/.openclaw/openclaw.json（Windows 通常是 C:\Users\<你>\.openclaw\openclaw.json）

  {
    "gateway": {
      "mode": "local",
      "bind": "loopback",
      "auth": { "mode": "token", "token": "replace-with-long-random-token" }
    },
    "session": { "dmScope": "per-channel-peer" },
    "agents": {
      "defaults": {
        "model": { "primary": "anthropic/claude-opus-4-6" },
        "sandbox": {
          "mode": "non-main",
          "scope": "session",
          "workspaceAccess": "rw",
          "browser": { "allowHostControl": true }
        }
      }
    },
    "tools": {
      "profile": "coding",
      "allow": ["browser", "cron"],
      "fs": { "workspaceOnly": true },
      "exec": { "security": "allowlist", "ask": "on-miss" },
      "elevated": { "enabled": false }
    },
    "channels": {
      "whatsapp": {
        "dmPolicy": "pairing",
        "groups": { "*": { "requireMention": true } }
      }
    }
  }
  这份配置的含义是：
 
  - 有 Cowork 核心能力（文件/命令/浏览器/定时）。
  - 但默认比较“保守”，避免一上来就给无限执行权限。
 
  4. 把“可执行命令审批”打开（关键）
     先看当前审批策略：
 
  openclaw approvals get
 
  按二进制绝对路径加 allowlist（示例）：
 
  openclaw approvals allowlist add --agent main "C:\Program Files\Git\cmd\git.exe"
  openclaw approvals allowlist add --agent main "C:\Python312\python.exe"
 
  说明：
 
  - exec approvals 是 OpenClaw 里最接近 Cowork “关键动作要确认”的机制。
  - 建议默认 security=allowlist + ask=on-miss，再逐步放开。
 
  5. 开启“长期任务/自动跑”
 
  openclaw hooks enable session-memory
  openclaw hooks enable command-logger
  openclaw hooks check
 
  加一个定时任务（示例）：
 
  openclaw cron add `
    --name "Morning brief" `
    --cron "0 9 * * *" `
    --tz "Asia/Shanghai" `
    --session isolated `
    --message "汇总今天待办和日程，给出3个优先事项。" `
    --announce
  再验证：
 
  openclaw cron list
  openclaw cron runs --id <jobId>
 
  6. 多代理并行（可选，接近 Cowork sub-agents）
 
  openclaw agents add planner
  openclaw agents add worker
  openclaw agents list --bindings
 
  后续你可以把不同 agent 绑定到不同渠道/账号，形成“计划者 + 执行者”分工。
 
  你需要知道的差异（和 Cowork 不完全一样）
 
  7. Cowork 是官方 Desktop 一体体验；OpenClaw 是网关化、自托管，需要你自己调策略。
  8. Cowork 的“删除文件审批”等是产品内建；OpenClaw 需要你通过 tools 策略 + exec approvals 来实现。
  9. Cowork 定时任务依赖桌面端状态；OpenClaw 的 cron 在 Gateway 内运行，更适合常驻服务。
  10. Cowork 的官方 connectors 生态和 OpenClaw 插件体系不是同一套，需要按你场景单独接。
