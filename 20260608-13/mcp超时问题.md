● 修复完成。总结一下这次 /doctor 三个问题的处理：

  根因（一个原因，三个症状）

  tavily、chrome-devtools、playwright 都配成了 npx -y <包>@latest——@latest 标签强制 npx 每次启动都先连 npm registry 解析版本，而这个请求走 127.0.0.1:7890 的 Clash
  代理。会话启动那一刻代理没就绪（或节点慢），三个就一起卡满 30 秒超时。对照组完美印证：firecrawl 用裸包名（命中本地缓存、零网络）正常，jina 是远程 HTTP（不经 npm）正常。

  排除了两个候选嫌疑：不是 WSL MTU 巨帧问题（网卡现在是 1500/1400，之前装的 systemd 服务在正常工作），也不是包坏了（三个包的缓存版本恰好都等于最新版，npx chrome-devtools-mcp@latest --version 实测只要 1.6
  秒）。

  已做的修改

  按你确认的方案，把三处 @latest 去掉，对齐 firecrawl 的写法：

  - ~/.claude.json 用户级：tavily-mcp@latest → tavily-mcp，chrome-devtools-mcp@latest → chrome-devtools-mcp
  - 项目级：@playwright/mcp@latest → @playwright/mcp
  - 改前备份在 ~/.claude.json.bak-20260610-mcpfix，改后 JSON 已校验合法
  - 诊断结论和操作细节已存入记忆

  你需要做的

  重启 Claude Code 会话（当前会话用的还是内存里的旧配置），然后跑一次 /doctor 复查。一个小提醒：因为是在会话运行中改的配置文件，退出时有小概率被旧副本覆写——如果复查发现又超时了，跟我说一声，重新套用一遍即可。

  以后想升级某个 MCP 包，手动跑一次 npx -y <包名>@latest 刷新缓存就行，配置不用再动。