• 你现在这台 Windows 机器，最稳的用法是：

  1. 在 WSL2 里跑 OpenClaw（官方推荐，兼容性最好）。
  2. 先用 dashboard 聊起来（不用先配 WhatsApp/Telegram）。
  3. 再按需接渠道、开审批和自动任务。

  最短可用流程（10 分钟）

  4. 在 PowerShell（管理员）安装 WSL2：

  wsl --install -d Ubuntu-24.04

  5. 首次进 Ubuntu 后启用 systemd（用于守护进程）：
  sudo tee /etc/wsl.conf >/dev/null <<'EOF'
  [boot]
  systemd=true
  EOF
  exit

  6. 回到 PowerShell 重启 WSL：

  wsl --shutdown
  wsl

  7. 在 WSL 里安装 OpenClaw：

  curl -fsSL https://openclaw.ai/install.sh | bash

  8. 跑新手引导并安装后台服务：

  openclaw onboard --install-daemon

  9. 验证服务并打开 Web 控制台：

  openclaw gateway status
  openclaw dashboard

  10. 直接在浏览器聊天（最快路径）：

  - 本机地址通常是 http://127.0.0.1:18789/

  如果你要接 WhatsApp/Telegram（可选）

  1. 登录渠道（以 WhatsApp 为例）：

  openclaw channels login

  2. 如果私信被配对拦截，审批配对码：
  openclaw pairing list whatsapp
  openclaw pairing approve whatsapp <code>
 
  3. 健康检查：
 
  openclaw status
  openclaw health

  你已经下载了源码，开发模式这样用
 
  在 WSL 里进入你下载的代码目录（Windows 的 E:\... 在 WSL 下是 /mnt/e/...）：
 
  cd /mnt/e/github_example/openclaw
  pnpm install
  pnpm ui:build
  pnpm build
  pnpm openclaw onboard --install-daemon
  pnpm openclaw dashboard

  如果想前台调试 Gateway：

  node openclaw.mjs gateway --port 18789 --verbose
 
  给你的建议
 
  - 日常使用：安装版 + onboard --install-daemon。
  - 二次开发：你这个源码目录 + pnpm 工作流。
  - 安全上先保守：先只在 dashboard 本地用，确认稳定后再开外部渠道。
 
  参考文档（你本地仓库里）：
 
  - getting-started.md
  - wizard.md
  - windows.md
 
  1. 你要我按“日常使用”还是“源码开发”给你出一套一键命令清单（直接可复制）？
