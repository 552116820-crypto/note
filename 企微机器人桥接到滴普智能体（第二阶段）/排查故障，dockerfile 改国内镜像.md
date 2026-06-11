# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `be03f72c-47fd-4404-a8fc-a92aec41fa6c`
- **范围**: Turn 1-2
- **导出时间**: 2026-06-11 14:59:24

---

### Turn 1  *(2026-06-11 06:19)*

**User:**

 => [wecom_deepexi_aibot 3/8] RUN apt-get update     && apt-get install -y --no-install-recommends ca-certifica  260.9s
 => => # Hit:1 http://deb.debian.org/debian trixie InRelease
 => => # Get:2 http://deb.debian.org/debian trixie-updates InRelease [47.3 kB]
 => => # Get:3 http://deb.debian.org/debian-security trixie-security InRelease [43.4 kB]
 => => # Get:4 http://deb.debian.org/debian trixie/main amd64 Packages [9671 kB]

**Assistant:**

That `apt-get` step is crawling because it's pulling from `deb.debian.org`, which is painfully slow from inside China. Let me look at the Dockerfile and switch it to a domestic mirror.

The Dockerfile itself is fine — the problem is the network path. Let me quickly verify one suspicion about the base image:

**这不是卡死，也不是报错 —— 是 `apt-get update` 在从境外官方源 `deb.debian.org` 拉取软件包索引，国内访问这个源速度极慢。** 你看到的 `Get:4 ... trixie/main amd64 Packages [9671 kB]` 就是它正在以龟速下载那个 9.6MB 的索引文件，260 秒还没下完。放着不管最终大概率能跑完，但要再等很多分钟；而且就算熬过这一步，下一步 `pip install`（第 4/8 步）连的是 pypi.org，同样会慢成这样。

我已经把 Dockerfile 改成走国内镜像了，两处改动：

1. **apt 换源**：在 `apt-get update` 之前把 `deb.debian.org` 替换成 `mirrors.aliyun.com`（trixie 的源配置在 `/etc/apt/sources.list.d/debian.sources`，sed 一行搞定）。
2. **pip 换源**：在 ENV 里加了 `PIP_INDEX_URL=https://mirrors.aliyun.com/pypi/simple/`，下一步装 requirements 时直接走阿里云 PyPI 镜像。

**现在你需要：Ctrl+C 终止当前这个 build，然后重新跑一遍 `docker compose build`（或你原来的 build 命令）。** 改动会使这一层的缓存失效并用新源重建，原来几分钟的步骤应该几秒就过。

顺带一提：`python:3.12-slim` 基础镜像里其实很可能已经自带 `ca-certificates`，这步安装基本是空操作；我本想拉镜像验证后直接删掉这一步，但拉取本身也被同样的网络问题卡住了，所以保留了这步（换源后它只花几秒，无伤大雅）。如果你之后想精简，可以进容器跑 `dpkg -s ca-certificates` 确认后把整个 apt 步骤删掉。

---

### Turn 2  *(2026-06-11 06:29)*

**User:**

企微机器人返回滴普未返回可展示内容。这是什么情况

**Assistant:**

机器人能回话说明回调链路是通的，问题出在调用滴普接口后没解析出内容。我先找到这条兜底文案的触发条件，再看容器日志里上游实际返回了什么。

情况查清楚了，结论如下。

## 这句话意味着什么

"滴普未返回可展示内容。"是写死在 `message_handler.py:923` 的兜底文案，触发条件非常具体：**滴普流式接口连接成功、返回 HTTP 200、SSE 流里也确实有数据，但把整条流解析完后没提取出任何文字、也没有文件，且没识别出错误码**。

所以可以先排除两类怀疑：
- **不是企微侧的问题** —— 回调、WebSocket 回复链路都是通的，机器人才能把这句话发出来；
- **也不是连不上滴普** —— 连接失败/超时/网关 5xx 会走另外几条文案（"滴普接口连续 3 次返回临时错误"、"AI 服务暂时繁忙"或 `REPLY_ERROR_PREFIX`）。机器人实实在在拿到了一条 200 的流，只是里面"看起来"什么都没有。

## 为什么"看起来"什么都没有

解析代码有两个盲点，会把真实的上游错误悄悄吞成这句兜底：

1. **字符串错误码被忽略**（`agent_chat.py:854`）：`_raise_for_error_code` 只在 `code` 是**整数**且非 0 时报错。国内网关常见的 200 + `{"code": "Axxx", "message": "签名错误/未授权/agent不存在"}` 这种字符串 code 会被直接放过，而 `message` 字段又不在内容提取的键列表里 → 整条流"空" → 兜底文案。真实报错信息就这样被吞了。
2. **workflow 输出位置解析不到**（`agent_chat.py:826`）：内容提取只看顶层和 `data` 下的 `answer/delta/content/text/output`。如果滴普这个 agent 走的是 workflow 型事件、最终答案放在 `data.outputs.answer` 里，代码认得 `workflow_finished` 这个结束事件（`:852`）却不解析 `outputs` → 同样空。

可能的真实原因按概率排：**带字符串错误码的业务错误**（agent 未发布/无权限/配额、`DEEPEXI_COMMAND`/`DEEPEXI_CHAT_MODEL` 配置与平台不匹配等）＞ **答案在解析不到的字段里** ＞ agent 真的返回了空。

## 怎么一锤定音

日志帮不上忙 —— 代码里没有任何地方会把 SSE 原始事件打进日志。我从本机重放签名请求也试过了，直接超时被丢（knowledge.amer.cn 有 IP 白名单，只加了服务器出口 8.138.104.14，且本机流量还挂着 7890 代理）。**所以必须在跑机器人的那台机器上**执行下面这条命令，它在容器内用和机器人完全相同的配置发一次请求，把原始 SSE 流打印出来：

```bash
docker exec -i wecom_deepexi_aibot python - <<'PY'
import os, json, time, hmac, hashlib, requests

env = os.environ
base = env["DEEPEXI_BASE_URL"].rstrip("/")
app_id = env["DEEPEXI_APP_ID"]
ts = str(int(time.time() * 1000))
sig = hmac.new(env["DEEPEXI_SECRET_KEY"].encode(), (app_id + ts).encode(), hashlib.sha256).hexdigest()

headers = {"Content-Type": "application/json", "app_id": app_id,
           "signature": sig, "timestamp": ts, "app_user": "diag"}
if env.get("DEEPEXI_PID", "").strip():
    headers["pid"] = env["DEEPEXI_PID"].strip()

payload = {"agent_id": env["DEEPEXI_AGENT_ID"], "query": "你好",
           "inputs": json.loads(env.get("DEEPEXI_FIXED_INPUTS_JSON") or "{}"),
           "response_mode": "streaming"}
if env.get("DEEPEXI_COMMAND", "").strip():
    payload["command"] = env["DEEPEXI_COMMAND"].strip()
if env.get("DEEPEXI_COMMAND_PARAM_JSON", "").strip():
    payload["command_param"] = json.loads(env["DEEPEXI_COMMAND_PARAM_JSON"])
if env.get("DEEPEXI_CHAT_MODEL", "").strip():
    payload["chat_model"] = env["DEEPEXI_CHAT_MODEL"].strip()
if env.get("DEEPEXI_ENABLE_DEEPRESEARCH", "").strip():
    payload["enable_deepresearch"] = env["DEEPEXI_ENABLE_DEEPRESEARCH"].strip().lower() in ("1", "true", "yes", "on")

r = requests.post(base + "/chat/chat-messages", json=payload,
                  headers=headers, stream=True, timeout=120)
print("HTTP", r.status_code, "|", r.headers.get("Content-Type"))
for i, line in enumerate(r.iter_lines(decode_unicode=True)):
    if line:
        print(line[:600])
    if i >= 80:
        print("...(后续截断)")
        break
PY
```

把输出贴给我（注意里面不会含密钥），一眼就能看出是哪种情况。不管结果如何，那两个吞错误的盲点都值得修（字符串 code 也报错、解析 `data.outputs`、把原始事件打进 debug 日志），你说一声我就直接改。

---
