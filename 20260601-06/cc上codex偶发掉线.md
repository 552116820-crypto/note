# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `e0e7f64b-c12c-472e-bd30-74237c64068b`
- **范围**: Turn 1
- **导出时间**: 2026-06-06 15:52:20

---

### Turn 1  *(2026-06-06 07:38)*

**User:**

为什么claude code 主要调用codex，会偶发掉线

**Assistant:**

简短回答前我先弄清两件事：你的 codex 接入到底是哪种形态（`codex exec` 子进程 / companion 守护进程 / MCP），以及你这台 WSL2 的网络层有没有老毛病在捣乱（你之前记过 eth2 巨帧 MTU 把大响应挂死的坑）。我先做几个只读诊断。

很有料。已经看清你的 codex 接入链路了，而且**发现一个大概率元凶**：你这台机器所有公网流量在走一个 TUN 代理。先把几个关键点坐实再下结论。

观察到的链路（自上而下）：
- Claude Code → `codex-companion.mjs task --wait`（PID 17475/17495，**此刻正有一个 review 任务在跑**）
- → `app-server-broker.mjs`（unix socket `/tmp/cxc-q4eiSC/broker.sock`）
- → `codex app-server`（node 壳）→ `codex app-server`（Rust musl 二进制）→ OpenAI 后端

可疑点：默认路由走 eth1(192.168.40.1)，但 `1.0.0.0/8 … 128.0.0.0/1` 这一整套把**几乎全部公网 IP** 都甩给了 `eth0 / 198.18.0.2`——`198.18.0.0/15` 是典型的 Clash/sing-box TUN 假 IP 段。再确认三件事：

两个 smoking gun 已坐实，而且其中一个**正是你自己记过的老坑**。在下结论前，我想从 codex 自己的日志里捞一条真实的掉线签名（你给过我"声称前先验证"的规矩），`logs_2.sqlite` 在用就用 python 只读打开。注意：**此刻你有一个 codex review 任务正在跑**（smart_lead_generation 的 plan review，PID 17475），所以我只读、绝不动网卡。

`logs` 表的列名不在我的白名单里，没扫到。直接看它的真实结构再捞最近的错误行。

真实日志把结论从"推测"变成"实锤"了。再精确一步：看 502/503/504 到底是不是网关状态码、以及有没有 ERROR/WARN 级别的体，免得我把字节数里的"502"误当错误。

先纠正我自己一步，免得误导你：我按你"声称前先验证"的规矩去 codex 自己的日志（`logs_2.sqlite`，当前会话）里捞掉线签名，结果——

- 之前那批 `502/503/504 ×276` 是**假阳性**：它们落在 WebSocket 帧的十六进制 payload 和 `length:` 字段里，不是网关状态码。撤回。
- 真正的 ERROR/WARN 只有 `codex_core_skills::loader` 的技能加载告警，**与网络无关**。
- proxy 上下文反而是健康的：`proxy> Request completed ... status=200 OK`、`tunneling HTTPS over proxy`、`handshake complete`。

也就是说**当前这个会话是好的**，日志窗口里没有抓到掉线——这正是"偶发"的特征：故障不在我能看到的这段。所以下面是**按架构+你这台机器的已知毛病**推断，不是"我看到它断了"。

## 真正的"为什么"：codex 不是本地算力，是一条长连接的瘦客户端

日志实锤的链路：

```
Claude Code → codex-companion.mjs (task --wait)
  → app-server-broker.mjs (unix sock)
  → codex app-server (Rust)
  → 【tokio-tungstenite WebSocket】→ https://chatgpt.com/backend-api/codex
            ↑ 全程走 https_proxy=http://127.0.0.1:7890 (你的 Clash)
```

关键点：**它和后端是一条持久 WebSocket**（流式吐 `response.output_text.delta`），不是一发一收的短 HTTP。一条长连接意味着——中间任何一跳抖一下，整个 turn 当场断 = 你看到的"掉线"。普通本地 CLI 不会这样，正因为 codex 每个字都要穿过代理到远端。

## 按可能性排序的元凶

1. **代理节点抖动（头号）** — 你的出口是 `Codex → Clash:7890 → 选中的节点 → chatgpt.com`。最脆的是"Clash→节点→OpenAI"这段。节点切换／限速／TCP reset 的瞬间，那条 WS 就死，当前 turn 掉线。日志里它现在 200 OK，所以掉线发生在**节点质量变差的时刻**，不在这个窗口。

2. **`--wait` 240s 超时——像掉线但其实没断** — companion 默认 `DEFAULT_STATUS_WAIT_TIMEOUT_MS = 240000`。长任务（比如**你此刻正在跑的那个 plan review**，PID 17475）一旦超过 240s，Claude 这边按超时报错，可 codex app-server 其实还在跑。这跟你记过的"plugin 是 user-in-the-loop"对得上：长任务应走 `--background` + `/codex:result <jobid>` 取，而不是 `--wait` 干等。

3. **双重代理 + mirrored 模式叠 buff** — 你同时开着 env 代理(7890) **和** TUN（eth0 `198.18.0.1/16`，全公网 IP 走 `198.18.0.2`），外加 `.wslconfig` 里 `networkingMode=mirrored` + `autoProxy` + `dnsTunneling` + `firewall` 全开。路径分裂、谁走谁不走不确定，是公认的偶发性来源。能砍一层就砍（TUN 和 env 代理留一个）。

4. **eth0 MTU=9000（你的老坑，但这次大概率不背锅）** — 承载公网的 eth0 是巨帧 9000，正是你记过的 `wsl_mtu_jumbo` 那个把大响应挂死的毛病（当时是 eth2，mirrored 改了网卡名）。**但**：codex 日志显示它走 `127.0.0.1:7890` 的 HTTP CONNECT 隧道，即先打到回环再由 Windows 侧 Clash 出网，**绕开了 eth0**——所以它多半不是 codex 掉线的主因。该修还是要修（别的走 TUN 的东西会中招），只是别指望它治好 codex。

## 怎么坐实 + 怎么修

**抓真签名**——下次它一掉，立刻跑这条（趁日志还新）：

```bash
python3 - <<'PY'
import sqlite3
c=sqlite3.connect("file:/home/huangch001002/.codex/logs_2.sqlite?mode=ro&immutable=1",uri=True)
for r in c.execute("SELECT ts,level,substr(feedback_log_body,1,200) FROM logs WHERE level IN('ERROR','WARN') OR target LIKE '%proxy%' OR feedback_log_body LIKE '%reset%' OR feedback_log_body LIKE '%closed%' OR feedback_log_body LIKE '%tungstenite%Error%' ORDER BY id DESC LIMIT 40"):
    print(r)
PY
```

看到 `reset by peer`／`Connection closed`／`tungstenite ... Protocol` → 实锤是代理那一跳断的。

**低风险动作（建议你来按，我不擅自动）**：
- 长任务改 `--background` + `/codex:result` —— 直接消掉 240s 这类"假掉线"。
- 代理砍到单层：要么关 Clash 的 TUN 只留 7890，要么反之。
- `sudo ip link set dev eth0 mtu 1500` —— **但现在别动**，你有 review 任务正跑在这条网卡的环境里，改 MTU 可能反而把它弄断；等它结束再说，且重启 WSL 会复原。

---

要我帮你做哪个？我可以：① 等当前 review 任务跑完后帮你压 MTU；② 把"codex 链路架构 + `--wait` 240s + 走 7890 隧道"这几条非显然的基建知识更新进 `codex_collaboration`/`wsl_mtu_jumbo` 记忆，省得下次再从头查。不过最有诊断价值的还是**下次掉线当场跑上面那条 grep**——那才是把头号嫌疑从"推断"变"实锤"的唯一办法。

---
