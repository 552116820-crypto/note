• 对，这次你的判断更准。

  截至 2026-03-24，Anthropic 官方帮助中心明确写的是支持 Excel on Windows，所以“你之前在桌面 Excel 里能用 Claude”这件事本
  身是成立的，不是我前面说的“桌面端本来就不支持”。当前官方页面在这里：https://support.claude.com/en/articles/12650343-
  claude-in-excel

  结合你补充的“装了天锐绿盾终端后才打不开”，现在根因基本可以收敛为：天锐绿盾 拦了 Excel 里 Claude 加载项使用的 WebView2/
  HTTPS/登录链路。我本机排到的证据也一致：

  - Claude 的 Excel 加载项实际是 Office Web Add-in，入口是 https://pivot.claude.ai，直接在普通浏览器打开出现 Invalid Ad
    d-in Host 是正常的。
  - Windows 进程能访问 pivot.claude.ai 和 appsforoffice.microsoft.com，但访问 claude.ai 会拿到 Cloudflare 挑战页 403。
  - Claude 加载项运行时实际会用到这些域名：pivot.claude.ai、claude.ai、api.anthropic.com、mcp-proxy.anthropic.com、
    platform.claude.com，以及 Office 侧的 appsforoffice.microsoft.com、addins.store.office.com、
    officeclient.microsoft.com、marketplace.microsoft.com、login.microsoftonline.com。

  最直接的处理办法是让 IT 在绿盾里做例外：

  - 进程放行：EXCEL.EXE、msedgewebview2.exe、必要时 OfficeClickToRun.exe
  - 域名放行：*.claude.ai、api.anthropic.com、mcp-proxy.anthropic.com、platform.claude.com、
    appsforoffice.microsoft.com、addins.store.office.com、officeclient.microsoft.com、marketplace.microsoft.com、
    login.microsoftonline.com
  - 规则层面：关闭这些进程或域名上的 HTTPS 解密/证书替换/网页内容审计/浏览器控件拦截/WebView2 注入/剪贴板或输入管控

  你可以直接把这段发给 IT：

  安装天锐绿盾终端后，Excel 内的 Claude Office Add-in 无法启动。
  该加载项是基于 WebView2 的 Office Web Add-in，需由 EXCEL.EXE / msedgewebview2.exe 访问以下域名：
  *.claude.ai
  api.anthropic.com
  mcp-proxy.anthropic.com
  platform.claude.com
  appsforoffice.microsoft.com
  addins.store.office.com
  officeclient.microsoft.com
  marketplace.microsoft.com
  login.microsoftonline.com

  请在绿盾策略中对上述进程/域名关闭 HTTPS 解密、内容审计、浏览器控件拦截、WebView2 注入及相关 DLP 拦截策略，或加入白名
  单。

  如果你愿意，我下一步可以继续帮你做两件事里的一个：

  1. 给你一份更短的“发 IT 工单”版本
  2. 继续在你机器上查绿盾具体装成了哪个服务/驱动，方便你更精确地提白名单请求
