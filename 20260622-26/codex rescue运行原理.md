
# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `b4a50f76-3e4e-4a55-af1a-2b2431d86d60`
- **范围**: Turn 1
- **导出时间**: 2026-06-23 17:11:42

---

### Turn 1  *(2026-06-23 09:04)*

**User:**

codex rescue运行原理是什么？为什么gpt会员过期了还能用？codex rescue用的是什么模型啊

**Assistant:**

いい質問です。3つとも実際の設定を見れば答えが出るので、推測せず手元の codex プラグインと CLI 設定を調べます。

ChatGPT ログイン（OAuth）方式ですね。秘密情報は伏せつつ、認証モードと有効期限、そして rescue の動作原理を読み解きます。

認証は **ChatGPT ログイン（OAuth）** で、ここに「会員過期でも使える」の答えが出ています。最後に実行チェーンを確定するため companion スクリプトを確認します。

全部確認できました。トークンの中身（秘密）は出さず、認証モードと期限のメタ情報だけ読み取っています。既存メモリも見て整理します。

整理がついたので、まず既存メモリに認証方式とモデルの確定事実を追記し、その後に回答します。

全部実機の設定・トークン・プラグインスクリプトで裏取りした結果です。

## 1. codex rescue 运行原理

它本质是一个**两层转发链**，rescue 自己几乎不干活：

```
你(Claude Opus 4.8 主线程)
  └─ codex:codex-rescue 子 agent  ← 跑在 Claude Sonnet 上，唯一职责是
     │                              "把你的需求改写成更紧的 prompt + 转发"，
     │                              不读库、不自己解题
     └─ 一次 Bash 调用:
        node codex-companion.mjs task "<你的任务>"
        └─ codex-companion.mjs(plugin 里的 Node 脚本，负责 job 管理/前后台/resume)
           └─ spawn 本地 Codex CLI(v0.139.0)的 app-server
              └─ 在你的仓库里 full-access 跑(approval=never, sandbox=danger-full-access)
                 真正读文件、改代码、跑命令的是这一层
```

- 后台任务返回 **job ID**，你用 `/codex:result <id>` 取结果 —— 这是 plugin 刻意的 user-in-the-loop 设计（`status`/`result` 标了 `disable-model-invocation`，只有你能调，我调不了）。这点和我之前那条记忆一致。
- 一句话：**Claude 负责编排和转发，GPT(Codex CLI) 负责实际干活。**

## 2. 为什么 GPT 会员过期了还能用

关键：**这里的认证不是 API key，而是 ChatGPT 账号的 OAuth 登录。**

查 `~/.codex/auth.json`：
- `auth_mode = chatgpt`，`OPENAI_API_KEY` 是**空的** → 它不按 token 计费，也不吃你单独充的 API 余额。
- 存了三个 token：`id_token / access_token / refresh_token`。CLI 靠长期有效的 **refresh_token** 定期换新的短期 access_token。你的 `last_refresh = 2026-06-17`，说明 6 天前还成功刷新过。

**所以"会员过期还能用"的机制**：只要那个 refresh_token 没被 OpenAI 吊销，CLI 就一直能换到可用 token，Codex 后端就继续放行 —— 这跟"网页版 ChatGPT 会员到没到期"不是同一个开关。

⚠️ 一点我无法只靠本地数据下定论的：token 里记的 plus 订阅窗口是 `2026-04-21 → 2026-05-21`（看着像一个多月前就到期），但 6/17 仍刷新成功。两种可能：(a) 你的 Plus 其实是按月自动续了，本地这个是旧 id_token 的 stale 快照，实际并没过期；(b) 真过期了但 refresh_token 仍能换到 token。**想要确切答案**，跑一句 `codex login status`（只读认证状态，不消耗模型额度）就知道当前真实计划。

## 3. codex rescue 用的是什么模型

要分清**两个"模型"**：

| 角色 | 模型 | 来源 |
|---|---|---|
| 真正写代码/诊断的 | **GPT-5.5**，reasoning effort **xhigh** | `~/.codex/config.toml` 默认值；rescue 不带 `--model` 就走它 |
| 包一层的转发 subagent | **Claude Sonnet** | `codex-rescue.md` 的 `model: sonnet` |
| 你现在对话的主线程 | **Claude Opus 4.8** | 当前会话 |

可覆盖：`/codex:rescue --model spark` → `gpt-5.3-codex-spark`；其它名（如 `gpt-5.4-mini`）直接透传；`--effort none|minimal|low|medium|high|xhigh` 调推理强度。

---
顺手把"认证方式 + 默认模型 + 三层模型栈"补进了 codex 协作那条记忆，省得以后再扒一遍。

---
