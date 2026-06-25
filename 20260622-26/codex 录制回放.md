# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `187ff21b-ed44-4ca1-bcbf-9d9b17b39d77`
- **范围**: Turn 1-4
- **导出时间**: 2026-06-25 16:39:18

---

### Turn 1  *(2026-06-25 07:56)*

**User:**

codex最近推出的新功能，克隆功能是什么来的

**Assistant:**

已经定位到了。我先深入确认一下细节再回答——中文媒体说的"Codex 克隆功能"对应的应该是官方的 **Record & Replay（录制回放）**。让我抓两个源核实。

你说的"克隆功能"就是 OpenAI 在 **6 月 18 日**给 Codex（macOS 桌面 App）推出的 **Record & Replay（录制回放）**。中文媒体（6/19 那批稿子）把它叫"克隆功能"，是因为它能把你的一套操作流程**原样复制成一个可复用的技能**——不是官方名字，官方叫 Record & Replay。

## 它到底干什么

一句话:**演示一遍,它就学会,以后自动重放**(show, don't tell)。

1. 你在 Mac 上点"录制技能",然后**真人操作一遍**某个重复流程(报销、传视频、建 Issue、下载周报……)
2. Codex 全程"看屏幕",记录你的点击、输入和窗口内容
3. 停止录制后,它的模型分析这段录制,自动生成一个 **`SKILL.md`** ——人类可读、可编辑的技能文档,里面写清:何时用、需要哪些输入变量、每一步怎么做、怎么验证成功
4. 下次你换一组参数调用这个技能,它就用 **Computer Use(控桌面)+ 浏览器操作 + 插件**(Slack/Gmail/Notion/Salesforce 等)把整个流程自动跑完,不用再录

## 和传统 RPA 录制有什么不一样

- 产物是**自然语言的 `SKILL.md`,不是锁死的坐标宏**——所以换变量、界面小改动它能自适应
- 能**打通没有 API 的应用**(本地表格 → 网页门户 → 提交报销 → Slack 确认 一条龙)
- 文件可打开、可改、可分享给团队;还能"内外循环"自进化:你每次手动改它的结果,它会回头优化技能逻辑

## 使用门槛(对你比较关键)

- **只支持 macOS App**,单录最长 30 分钟,需要开 Computer Use + 屏幕录制/辅助功能权限
- 套餐:Plus / Pro / Business / Enterprise / Edu
- **EEA(欧盟)、英国、瑞士暂不开放**
- ⚠️ 你现在是 **Windows + WSL** 环境、平时走的是 Codex CLI / 插件——这个功能是桌面 App 专属,**你这套环境暂时用不上**,得在 Mac 上才能体验

---

顺带澄清一下,"Codex + 克隆"在中文圈还有两个**不是这个**的说法,别混了:
- **repo 克隆**:Codex Cloud 接到任务会把你的仓库 clone 进隔离沙箱跑——这是老行为,不是新功能
- **镜像克隆**:有人用 Codex 去"复刻整个网站"(抓页面+还原 JS/CSS),那是网友玩法,不是官方功能

如果你指的其实是后两者之一,跟我说一声,我再展开。

---

### Turn 2  *(2026-06-25 08:20)*

**User:**

Record & Replay（录制回放），所以这个功能强大吗？、

**Assistant:**

好问题——"录一遍就自动重放"听起来很猛,但它到底强不强,关键看底层 Computer Use 跑长流程的可靠性。我先抓点冷静的评测/质疑视角,别只复述发布会口径。

一句话结论:**它"强"在把自动化的门槛和可维护性打下来了,不"强"在长流程的执行可靠性**——这两件事要分开看,否则容易被发布会带跑。

## 最该记住的一个数字:复合失败

这是判断它强不强的命门。Temporal 的测算被多家引用:**单步可靠性 85% 的 Agent,跑一个 10 步流程,端到端成功率只有约 20%**(0.85¹⁰)。而 OpenAI 自己的文档也写了,Record & Replay "**在步骤稳定、成功判据清晰时效果最好**"——这话翻译过来就是:流程一长、一带分支,它就开始掉链子。

底层引擎 Computer Use 在 2026 年的真实水位(Stanford AI Index / 各 benchmark):
- OSWorld(跨系统电脑任务)约 **66%**,即**每 3 次仍失败 1 次**
- 实验室分数和真实部署之间还有约 **37% 的落差**
- 业界共识:"**适合受约束、有人盯着的工作流**",不是 set-and-forget 的 AI 员工

## 它真正强的地方(架构层面,不是噱头)

1. **录制产物是自然语言的 `SKILL.md`,不是坐标宏**——这是和传统 RPA(UiPath 那套)的本质区别。UI 一改,RPA 录的坐标就废了;它存的是"你想干什么",回放时让模型对着当前屏幕重新理解,所以**换变量、界面小改能自适应**。这把 2009 年就被指出的"按演示编程太脆"问题往前推了一大步。
2. **打通没有 API 的应用**——Zapier/Make 只能连有 API 的系统,这块 GUI-only 的空白正是它的主场。
3. **可读、可审计、可分享、可自进化**——技能文件能打开逐行改,团队复用,你每次手动修正它还会回头优化逻辑。

## 它会拉胯 / 你要警惕的地方

1. **长的、动态的、带条件分支的流程别交给它**——复合失败那条数学摆在那;录一遍也很难覆盖所有 edge case。
2. **不可逆 / 涉钱 / 对客户的动作,必须留人工 checkpoint**——它是拿着你的凭据自主操作屏幕,等于一个"特权身份",得当特权账号那样治理、逐行审 SKILL.md。
3. **稳定性要靠"周"为单位验证**——业界建议从低风险内部流程起步,短技能跑稳几周再上复杂的。

## 对你具体的结论

- 形态对的活儿(报销、填单、建 Issue、批量传内容这类**高重复、步骤固定、GUI-only、容忍盯梢**的)——是真省事,值得用。
- 期待它无人值守跑长链路复杂业务——**目前不行,会翻车**。
- ⚠️ 而且别忘了上一条说的:**它只在 macOS App 上,你 Windows + WSL 现在压根用不了**。对你来说短期更多是"了解趋势",不是马上能上手的工具。

要我说,它是个**方向正确、authoring 体验确实领先**的功能,但天花板被 Computer Use 的可靠性卡着——**惊艳的是范式,不是当下的执行力**。你那句"RECEIPTS BEAT VIBES"放这儿正合适。

---

### Turn 3  *(2026-06-25 08:24)*

**User:**

computer use 是什么？claude code是不是也有computer use？

**Assistant:**

好——这俩问题其实有意思,因为 **"Computer Use" 这个词最早就是 Anthropic(Claude)2024 年提出来的**,OpenAI 那套是跟进。我先把 Anthropic 这边的权威细节核实一下再答,避免凭记忆。

核对完了。有意思的是——**"Computer Use"这个词是 Anthropic 2024 年 10 月先造的**(随升级版 Claude 3.5 Sonnet 一起发),OpenAI 的 Operator / Codex 那套 Computer Use 是后来跟进的。下面分两块答你。

## 一、Computer Use 是什么

它是 **Anthropic API 里的一个工具**,让模型像人一样操作图形界面:**给它截图 → 它输出鼠标点击 / 键盘输入 / 滚动等动作 → 你的程序在环境里执行 → 再截图回传**,如此循环。本质就是"模型看屏幕、动鼠标键盘"。

- **怎么跑(agent loop)**:截图 → 模型决定下一步动作 → 你执行 → 新截图 → 重复,直到任务完成。
- **它是 client-side(客户端执行)工具**:Anthropic 只负责"看图、决定动作",**真正的桌面环境要你自己提供**——通常是一个 Docker 容器 + 虚拟显示器的沙箱。Anthropic 不托管这个桌面、也不留数据。
- **状态**:一直是 **beta**(测试版工具),在第一方 API、Bedrock、Vertex、Foundry 上都是 beta。
- **支持的模型**:2024-10 随 Claude 3.5 Sonnet 首发(工具类型 `computer_20241022`),3.7 / 4 系列升级为 `computer_20250124` 并提高精度;现在的 Opus 4.x / Sonnet 4.x 一路都支持。

跟你前面问的串起来:**Codex 的 Record & Replay 就是跑在 OpenAI 版 Computer Use 之上的**;Anthropic 这边的对应物,就是这个 API 工具。

## 二、Claude Code(CLI)有没有 Computer Use

**直接回答:Claude Code 这个 CLI 产品本身没有"桌面版 Computer Use",但有"浏览器版"的等价能力。** 要把三层分清:

1. **Computer Use 是"模型 / API"层的能力,不是 Claude Code 自带的功能。** Claude Code 是个**终端 + 代码** agent,原生工具是 Bash、读写文件、grep/glob、WebFetch/WebSearch——它**不截你的桌面、不去点任意 App**。

2. **但它能开浏览器干活**,这是最接近 computer use 的部分:
   - `/chrome` 命令把 Claude Code 接到一个 Chrome 上;
   - 加上 **chrome-devtools / playwright 这两个 MCP**(你自己调研时就在用),能导航、点击、填表单、截图、看 console。
   - 不过这是**浏览器范围**,而且主要走 DevTools 协议 / 可访问性树 + 截图,跟 API 里那个"通用桌面 Computer Use 工具"不是一回事。

3. **底层模型(Opus 等)本身是支持 Computer Use 的**——所以你要真想要"桌面级"操控,可以用 **Claude Agent SDK** 自己搭一个 computer-use agent(配沙箱桌面);但 Claude Code 这个 CLI 不是为这个设计的。

---

一句话收口:**Computer Use ≈ "模型看屏幕点鼠标"的底层能力,Anthropic 和 OpenAI 都有;Claude Code 没把这个桌面能力做进 CLI,只给了浏览器自动化(`/chrome` + playwright/chrome-devtools MCP)。** 而 Codex 的 Record & Replay,是在 OpenAI 的桌面 Computer Use 之上又包了一层"录制→生成技能"。

想要我帮你拿这两个 MCP(playwright / chrome-devtools)实际跑一个浏览器自动化的小例子看看效果吗?这个你现在的 WSL 环境是能直接用的。

---

### Turn 4  *(2026-06-25 08:31)*

**User:**

我该怎么用Anthropic的Computer Use

**Assistant:**

要给你能直接上手的步骤,我先把 Anthropic 官方 Computer Use 文档抓下来核对当前的工具版本、beta 头、动作集和官方 Docker 快速启动——这些细节我不想凭记忆给你。按你定的"单页用 Jina"。

Jina 在那个 `.md` 路径上 404 了(文档站迁移过)。换条路:Tavily 搜当前规范 URL + 关键参数,同时直接读官方 quickstart 仓库。

拿到官方最新参数了。给你两条路:**最快是跑官方 Docker demo**(正好适配你 WSL+Docker),想嵌进自己代码就用 API。

## 前提:你需要一个 Anthropic API key

Computer Use 是 **API 付费功能**,要的是 console.claude.com 上的 **API key + 余额**,**不是** Claude Code 的订阅/OAuth。这是最容易卡住的一步——先确认你有个能调 API 的 key。

## 路线一(推荐先玩):官方 Docker quickstart

它打包好了一切——一个容器里的 Linux 桌面(X11+VNC)+ agent loop + 工具实现 + 网页 UI。一条命令:

```bash
export ANTHROPIC_API_KEY=你的key
docker run \
    -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
    -v $HOME/.anthropic:/home/computeruse/.anthropic \
    -p 5900:5900 -p 8501:8501 -p 6080:6080 -p 8080:8080 \
    -it ghcr.io/anthropics/anthropic-quickstarts:computer-use-demo-latest
```

然后浏览器开 **http://localhost:8080**(左边聊天、右边看 Claude 操作那个虚拟桌面)。给它发"帮我在桌面存一张猫的图片"之类,就能看它截图→点鼠标→打字地干活。镜像默认用 **Claude Opus 4.8**。

> 容器里是个**隔离的虚拟桌面**,不是碰你真实的 Windows 桌面——这正是推荐的安全做法。

## 路线二:自己写代码调 API

核心就是把 `computer` 工具塞进 `client.beta.messages.create`,加上 beta 头,然后自己跑 agent loop:

```python
import anthropic
client = anthropic.Anthropic()

response = client.beta.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    tools=[
        {"type": "computer_20251124", "name": "computer",
         "display_width_px": 1024, "display_height_px": 768, "display_number": 1},
        {"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"},
        {"type": "bash_20250124", "name": "bash"},
    ],
    messages=[{"role": "user", "content": "Save a picture of a cat to my desktop."}],
    betas=["computer-use-2025-11-24"],   # 只有 computer 工具需要这个 beta 头
)
```

**当前版本**(2026):工具类型 `computer_20251124` + beta 头 `computer-use-2025-11-24`(上一代是 `computer_20250124` / `computer-use-2025-01-24`)。

**agent loop 你得自己转**:
1. 调 API → Claude 返回 `stop_reason: "tool_use"`,带一个动作(截图/移动/点击/输入/按键/滚动/拖拽…)
2. 你的程序**在一个真实桌面环境里执行**这个动作,把**新截图**作为 `tool_result` 发回
3. 回到第 1 步,直到 Claude 完成(`stop_reason: "end_turn"`)

关键点:**它是 client-side 工具**——桌面环境要你自己提供(虚拟机/容器+虚拟显示器),所有截图和操作都在**你这边**,Anthropic 只实时处理图片不留存。

## 几个注意

- **分辨率用 XGA 1024×768**。别超过 XGA/WXGA,高分辨率靠 API 自动缩放会掉精度、变慢——要更高分辨率就自己缩放图+换算坐标。
- **还在 beta**,精度有限;别拿它做需要绝对精确或碰敏感凭据的事,关键动作留人工确认。
- **prompt injection 风险**:网页/图片里的文字可能劫持 Claude。隔离沙箱 + 域名白名单 + 不给登录信息。

## 对你 WSL 环境

你有 Docker,这条 Docker 路**能直接跑**;唯一要留意的是镜像从 ghcr.io 拉(走 Clash 可能慢,几个 GB),`localhost:8080` 在 Windows 浏览器里直接能开。

要的话我现在就帮你把镜像拉起来跑一遍——你把能调 API 的 `ANTHROPIC_API_KEY` 准备好(或告诉我用 `! export ...` 自己设),我来发 `docker run` 并盯着启动日志。

---
