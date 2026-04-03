• 不是 OAuth 账号问题，是这台 WSL 环境的 DNS 解析坏了，claude 登录时解析 platform.claude.com 失败才报 getaddrinfo
  EAI_AGAIN。

  我这边实测到的情况是：

  - getent hosts platform.claude.com 解析失败
  - 去掉代理后，curl https://platform.claude.com 卡在 Resolving
  - 指定 DNS 1.1.1.1 时能解析出 IP
  - 你当前 WSL 里的 /etc/resolv.conf 只有 nameserver 10.255.255.254
  - 代理 127.0.0.1:7890 是通的，所以站点本身没问题

  先试一次把代理变量补全再登录：

  export HTTP_PROXY=http://127.0.0.1:7890
  export HTTPS_PROXY=http://127.0.0.1:7890
  export ALL_PROXY=http://127.0.0.1:7890
  claude auth login

  如果还报同样错误，直接修 WSL DNS。执行：

  sudo tee /etc/wsl.conf >/dev/null <<'EOF'
  [network]
  generateResolvConf = false
  EOF

  sudo rm -f /etc/resolv.conf
  sudo tee /etc/resolv.conf >/dev/null <<'EOF'
  nameserver 1.1.1.1
  nameserver 223.5.5.5
  EOF

  然后在 Windows PowerShell 里执行：

  wsl --shutdown

  重新打开 WSL 后验证：

  getent hosts platform.claude.com
  claude auth login

  补充一点：你现在的 claude 是旧的 npm 安装路径。登录恢复后，建议再跑一次：

  claude install

  这样会切到官方原生安装方式，后续稳定性通常更好。

  如果你要，我可以下一步直接帮你把 WSL 的 DNS 配掉。


• 我清理一下临时文件，不动现有配置。之后就只剩一次可选的 WSL 重启，用来让这个 DNS 配置在以后每次启动都持续生效。

• 已经处理完了。

  我更新了 /etc/wsl.conf 和 /etc/resolv.conf，把 WSL 的 DNS 固定到 1.1.1.1 和 223.5.5.5，并禁用了自动生成 resolv.conf。
  现在 platform.claude.com 在当前会话里已经能正常解析，Node 的 dns.lookup 也恢复了。

  claude auth status 现在返回的是已登录状态，所以之前那次 OAuth error: getaddrinfo EAI_AGAIN platform.claude.com 已经不
  再是 DNS 层面的阻塞。另一个结论是：不走代理时直连仍然超时，所以你这边后续继续用 Claude 时不要清掉代理环境变量；如果有
  工具只认 ALL_PROXY，补上这条即可：

  export ALL_PROXY=http://127.0.0.1:7890

  为了让这个 DNS 配置在以后每次启动 WSL 时都持续生效，再执行一次这个命令即可：

  wsl --shutdown

  然后重新打开 WSL。若你愿意，我下一步可以继续帮你把代理变量也固化到 shell 配置里。
