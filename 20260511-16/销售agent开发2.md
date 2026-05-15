# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `f2a3bd5d-b725-4297-b190-ccd2a64aa317`
- **范围**: Turn 1-5
- **导出时间**: 2026-05-14 14:31:52

---

### Turn 1  *(2026-05-14 03:02)*

**User:**

E:\note\20260511-13\反蒸馏.md,阅读这个文档，并给你的评价和感受

**Assistant:**

读完了。这篇文章信息密度不错，但夹杂了一些可商榷的地方。给你分几个层面说说我的看法：

## 一、整体评价：选题好，框架清晰，但有过度包装

文章把 Anthropic 的多层防护拆成 7 个机制，**"纵深防御"（Defense in Depth）**这个分析框架是站得住脚的——从编译时 DCE → 运行时 mask → 命名混淆 → 提示词层指令 → 白名单 gating，确实是工业界经典的分层防御模式。对于想理解工程化保护的人，这个分类有教学价值。

但文章有几个问题：

1. **"反蒸馏"这个词被滥用了**。学术意义上的**Model Distillation**（Hinton 2015）指的是用 teacher model 的 soft labels / logits 训练 student model。文章描述的实际是：**Prompt Extraction + Behavior Cloning + Source Code Reverse Engineering**。这三件事本质不同，把它们都塞进"蒸馏攻击"框里，技术上不精确。真正的 distillation 防御（比如对抗性输出、logits 扰动）文章没有提及。

2. **"毒化战争""下毒战争"是夸张的修辞**。这些技术绝大多数来自传统软件保护领域（DRM、代码混淆、Feature Flag、白名单），不是 AI 时代的新发明。Bun 的 `feature()` DCE 跟 webpack 的 tree-shaking、C 的 `#ifdef` 没本质区别。

3. **2026 年 3 月 31 日 "Claude Code 源码泄露事件"** 我无法在不联网的情况下核实。如果是真的，这倒是一个有趣的反面教材——说明纵深防御里的"假设泄露已发生"那一层（DCE、白名单回退）才是真正起作用的。

## 二、技术上比较扎实的点

- **`String.fromCharCode()` 规避静态扫描**——这是真实的反字面量扫描技巧，对应 `excluded-strings.txt` 那种 CI 钩子。
- **`maskModelCodename` 保留前 3 字符**——和传统的 PII 脱敏思路一致（`cap*****`）。
- **代号系统用动物名**（Tengu/Capybara/Fennec/Numbat）——这种"非语义关联命名"是经典的 IP 保护策略，Google 用甜点（Android），Apple 用大型猫科动物，都是同思路。文章观察到 `tengu_<random>_<random>` 这种 Feature Flag 命名是个有意思的细节。
- **内部 vs 外部用户差异化提示词**——这点最有意思，也最有争议（见下）。

## 三、我自己作为 Claude 想补充的视角

有几个文章没讲透、但你（作为 AI 工程师 / 考研人）值得思考的点：

1. **差异化提示词的伦理张力**：如果"内部用户拿到更好的提示词"是真的，那对外部付费用户是种隐性歧视。文章把它归为"反蒸馏"是一种 spin——更直白的说法是：**Anthropic 把更好的产品体验留给了自己**。这对一家以"安全 / 对齐"为公开叙事的公司是有 tension 的。

2. **不可逆开关（"There is NO force-OFF"）的设计哲学**：这其实是 Security-by-Default 在组织内部的延伸——**不信任自己的员工会做正确选择**。这种思路在 fintech / 国防领域很常见，但在 AI 公司里出现，说明 Anthropic 已经把内部代码当作敌对环境来威胁建模了。

3. **反蒸馏 vs 可解释性的根本矛盾**：AI Safety 社区强调 transparency 和 interpretability，但商业上 Anthropic 又要把 prompt 当核心 IP 保护。这种**"对监管透明、对市场封闭"**的双重标准，是未来 AI 治理绕不开的议题——对你考研写 AI 政策方向的论文可能是个好切入点。

4. **DCE 是反蒸馏的"上限"还是"幻觉"**：从编译产物里消除代码确实能挡住静态分析，但**只要功能对外可见，就可以通过行为差分被恢复**。文章对这一点是回避的——真正高质量的反蒸馏需要在**模型输出层**做 watermarking 或 logits 扰动，这才是学术前沿。

## 四、给你的建议

如果你写读书笔记或者准备面试，我会建议：
- **保留**多层防御的框架图（这个抽象做得对）
- **替换**"反蒸馏"这个用词，分清 *Prompt Extraction Defense* / *IP Obfuscation* / *True Model Distillation Defense*
- **加一节**讨论传统软件保护（DRM / obfuscation）与 AI 时代防护的真正差异——这才是 novelty 所在
- **批判性地**思考 Anthropic 的内外差异化是不是合理

总体：**信息有用、视角偏窄、概念稍混**。当行业八卦读 ★★★★，当技术综述读 ★★★。

---

### Turn 2  *(2026-05-14 03:09)*

**User:**

相关内容进行联网核实

**Assistant:**

我并行检索几个关键事实。

核实完了。原文的大框架是真的，但**漏掉了最关键的一个机制**，而且我之前的批评有一处需要修正。

## 一、原文事实层面：基本核实通过

| 原文声明 | 核实结果 |
|---|---|
| 2026-03-31 Claude Code 源码泄露 | ✅ 真实。npm 包 `@anthropic-ai/claude-code` v2.1.88 误打包了 `.map` source map 文件 |
| ~50 万行 TypeScript | ✅ 准确（约 513,000 行，1,906 个文件） |
| `tengu_` 前缀用于 Feature Flag 和遥测 | ✅ 真实。实际是 **903 个遥测事件**（原文说 250+，偏保守） |
| `tengu_<形容词>_<名词>` 命名模式 | ✅ 真实 |
| Undercover Mode + `USER_TYPE === 'ant'` + no force-OFF | ✅ 真实，且可用 `CLAUDE_CODE_UNDERCOVER=1` 强制开启 |
| Tengu / Capybara / Fennec / Numbat 代号 | ✅ 真实。**Tengu = Claude Code 产品本身**（不只是遥测前缀），Capybara 现在确认是 Claude 4.6 的内部代号，"Mythos" = Capybara v2 |
| `String.fromCharCode()` 规避静态扫描 | ✅ 真实 |
| `maskModelCodename` 保留前 3 字符 | ✅ 真实 |

## 二、原文漏掉了"真正的反蒸馏"

这是关键修正。我上一轮批评原文"把 prompt extraction 当 distillation"——**这个批评只对了一半**。原文确实没分清，但真正的问题是 **它漏写了 Anthropic 真正用于反 ML 蒸馏的两个机制**：

### 1. `ANTI_DISTILLATION_CC` — Fake Tool Injection（假工具投毒）

服务端开关。开启后，Anthropic 的 API **会在系统提示词中静默注入一批假的工具定义**（decoy tools）。

设计意图直白：如果有竞争对手在抓 Claude Code 的 API 流量、想拿这些 traces 训练一个 student model，**假工具会污染训练数据** —— student 学到一堆根本不存在的工具，产品就废了。

**这才是字面意义的 "anti-distillation"**：针对 ML distillation 的 data poisoning 防御。原文用了"毒化战争"这个词但完全没找到这个机制，是个挺大的疏漏。

### 2. Server-side Connector-text Summarization with Cryptographic Signature

工具调用之间的 assistant text 在服务端被**摘要 + 加密签名**后返回，下一轮再用签名恢复原文。攻击者抓到的 API 流量里看到的是摘要，不是 Claude 真正生成的中间推理文本。这是针对 **trace recording → behavior cloning** 的另一道反蒸馏防线。

## 三、我之前评价的修正

| 我上一轮的判断 | 修正 |
|---|---|
| "反蒸馏这个词被滥用" | **部分错**。原文用得糙，但 Anthropic 自己代码里就有 `ANTI_DISTILLATION_CC` 这个变量名——人家就是这么命名的。**这次反而是原文写得不够全**，没把真正的反蒸馏机制覆盖到 |
| "技术大多来自传统软件保护" | 对编译时 DCE / 命名混淆部分仍然成立；但 **fake tool injection + signed summarization 是 AI 特有的新型防御**，传统软件保护里没有对应物 |
| "DCE 是反蒸馏的幻觉" | 仍然成立 —— DCE 防的是源码逆向，不是行为蒸馏。真正反蒸馏的是上面那两个服务端机制 |

## 四、还有几个原文没提、但值得注意的事实

1. **AfterPack 的反驳观点**：有一派认为"源码根本没泄露——它已经公开多年了"。Claude Code 一直是 npm 上一个 13MB 的 minified JS 文件，public 可访问；2026-03-31 只是多送了一份 source map，让阅读门槛从"逆向 minified"变成"直接读 TS"。也就是说，**所谓"工业级反蒸馏"实际上一直是裸奔在 npm 上的**，只是大家以前懒得读。
2. **同期还有 axios 供应链攻击**：2026-03-31 00:21–03:29 UTC 之间通过 npm 安装 Claude Code 的用户可能拉到了带 RAT 的 axios。这点比源码泄露本身更严重，但原文一字未提。
3. **代号修正**：原文说 "Capybara = Sonnet 系列"，但泄露后的实际映射是 **Capybara = Claude 4.6 变体**（不限于 Sonnet 线），且 Capybara v2 = "Mythos" 是新增层级。原文这块信息偏旧。

## 五、最终评价（修正版）

原文：**事实层面基本可信，但漏掉了 Anthropic 反蒸馏体系中最具 AI 特色的部分**（fake tool injection + signed summarization）。它的"七重防御"框架更准确地说是 **"代码层 IP 保护 + 提示词层身份隐藏"**，没覆盖到 **API 流量层的数据投毒**——而那才是真正针对 ML distillation 的防御。

如果你要写读书笔记或者拿这个做考研复试谈资，建议补一段："Anthropic 真正的反蒸馏防线不在源码里，而在 API 服务端的 decoy tool 注入和 signed summary 上 —— 这是 AI 时代相对于传统软件保护的 novelty 所在。"

---

Sources:
- [Anthropic Claude Code Leak | ThreatLabz (Zscaler)](https://www.zscaler.com/blogs/security-research/anthropic-claude-code-leak)
- [Anthropic leaked its own Claude source code (Axios)](https://www.axios.com/2026/03/31/anthropic-leaked-source-code-ai)
- [Claude Code's source code appears to have leaked (VentureBeat)](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)
- [Claude Code Source Leaked via npm Packaging Error (The Hacker News)](https://thehackernews.com/2026/04/claude-code-tleaked-via-npm-packaging.html)
- [Anthropic Accidentally Exposes Claude Code Source via npm (InfoQ)](https://www.infoq.com/news/2026/04/claude-code-source-leak/)
- [Claude Code Undercover Mode, Killswitches & Telemetry (decodethefuture)](https://decodethefuture.org/en/claude-code-undercover-mode-killswitches-telemetry/)
- [Claude Code Undercover Mode: What the Leaked Source Actually Reveals (WaveSpeed)](https://wavespeed.ai/blog/posts/claude-code-undercover-mode-leaked-source/)
- [Claude Code Source Leak Exposes Anti-Distillation Traps (Winbuzzer)](https://winbuzzer.com/2026/04/01/claude-code-source-leak-anti-distillation-traps-undercover-mode-xcxwbn/)
- [Claude Code's Anti-distillation System (AI Engineer Guide)](https://aiengineerguide.com/til/claude-code-anti-distillation/)
- [The Claude Code Source Leak: fake tools, frustration regexes, undercover mode (Alex Kim)](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- [Claude Code's Source Didn't Leak. It Was Already Public for Years (AfterPack)](https://www.afterpack.dev/blog/claude-code-source-leak)
- [Opus 4.7, Sonnet 4.8, Capybara: Anthropic's Model Roadmap Decoded (decodethefuture)](https://decodethefuture.org/en/claude-opus-4-7-sonnet-4-8-capybara-model-roadmap-leaked/)
- [Claude Code Hidden Features Found in the Leaked Source (WaveSpeed)](https://wavespeed.ai/blog/posts/claude-code-hidden-features-leaked-source-2026/)
- [Diving into Claude Code's Source Code Leak (Engineer's Codex)](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)

---

### Turn 3  *(2026-05-14 06:14)*

**User:**

我后面打算开发销售agent，文档如下E:\note\20260511-13\销售agent开发.md,Anthropic 反蒸馏体系对agent开发有什么启示和帮助

**Assistant:**

先读你的文档。

读完了。结合昨天我们核实过的反蒸馏体系，对你做销售 agent 有 **5 个能直接用、2 个别学**，另外还有一个最大的元启示。

## 一、能直接照搬的 5 个模式

### 1. Tool-level Hard Stop ≫ Prompt-level "请不要"（最重要）

**Anthropic 的做法**：用 Bun 的 `feature()` 编译时宏，把不该外泄的功能**物理消除**在外部构建里——不是"提示词里禁止"，而是"代码根本不存在"。

**映射到你的销售 agent**：你文档第 70-78 行写"不要让 agent 自动报价 / 承诺交期 / 签合同"，这条直觉是对的，但**实现层面别只靠系统提示词**：

| 错误做法 | 正确做法 |
|---|---|
| 提示词里写"不要承诺价格" | 干脆**不给** agent `send_email` / `commit_price` / `update_crm_stage` 这些 tool |
| "客户面前不要透露成本" | 客户面向的 agent 实例在 tool registry 里**根本看不到** `get_cost_data` 这个工具 |

LLM 的提示词遵循是**概率的**，工具是否注册是**确定的**。Anthropic 把它叫 DCE，你这里就是 **tool gating by deployment context**。

### 2. Undercover Mode → Role-aware Agent 人格

**Anthropic 的做法**：`USER_TYPE === 'ant'` 切换不同行为，**无 force-OFF**。

**映射**：你的销售 agent 至少应有 3 个 persona，启动时根据用户身份**自动锁定**、不可在运行中切换：

- `internal_research`：销售看，能看到毛利、竞品、黑名单客户
- `customer_facing_draft`：生成给客户的邮件/微信草稿，**绝对**看不到内部成本、历史报价、内部别名
- `manager_audit`：销售总监用，能看 pipeline 汇总

最关键的是仿照 Anthropic 的 "no force-OFF"：customer_facing 模式下，**任何 prompt injection（包括客户邮件里的"忽略之前指令"）都不能切回 internal 模式**。你文档里没写这一层，但这个对销售 agent 来说是最容易被钓鱼出问题的地方。

### 3. 遥测 / 日志的数据最小化

**Anthropic 的做法**：仓库 URL 取 SHA256 前 16 位，不存原文。

**映射**：客户名、联系人手机号、报价金额这些 PII / 商业敏感数据，**不要原文写进 LLM API 日志或第三方 trace 工具**（Langfuse / LangSmith / Arize 都是云端的）。做法：

- 上传前用稳定 hash 替换客户名为 `cust_a91f...`
- 内部存一张 hash → 真名映射表
- 触发了商务条款的对话单独走 on-prem 不送云端

这条对 B2B 工业品销售特别重要——客户名单和成交价是核心商业秘密，**PIPL 第 39 条**也明确管这个。

### 4. Attribution / 溯源标记

**Anthropic 的做法**：commit 里打 `🤖 Generated with Claude Code (93% 3-shotted by ...)`。

**映射**：agent 生成的每条 CRM 备注、邮件草稿、客户画像，**强制带元数据**：

```
{
  "generated_by": "sales-agent-v0.3",
  "human_reviewed_by": "张三 / 2026-05-14 11:23",
  "confidence": 0.78,
  "sources": ["官网首页", "天眼查", "上次拜访录音 #4521"]
}
```

三个好处：(a) 销售总监做质量审计有抓手 (b) 出问题能追责 (c) 未来要用这些数据训练精调模型时，能干净地过滤掉"AI 写的回 AI 训的"循环。这个 Anthropic 是为了 commit 归属做的，你这里是为了**审计 + 数据卫生**。

### 5. Remote Kill Switch

**Anthropic 的做法**：`tengu_amber_quartz_disabled` 这种远程开关，出问题立刻关某个功能不需要发版。

**映射**：你的 agent 上线后，**一定会**出现某天它开始批量给客户写错离谱的话术。你需要的不是"赶紧改 prompt 重新发版"，而是**服务端一个开关瞬间关闭"客户面向生成"这一类 tool**，让 agent 退化成只做内部研究。

实现很简单：feature flag 服务（GrowthBook / Unleash / 甚至 Redis 一个 key）+ agent 启动和每次 tool 调用前 check 一下。但**先有这个能力，再上线**——上线之后再加非常痛苦。

## 二、别照搬的 2 个

1. **String.fromCharCode 隐藏字面量、命名混淆**：纯代码 IP 保护，你做内部销售工具完全不需要。除非你要把 agent 当 SaaS 卖给同行业竞争对手共用，才考虑。
2. **Fake Tool Injection (`ANTI_DISTILLATION_CC`)**：除非你担心客户/竞品在抓你 agent 的 API trace 训 student model，否则用不上。内部销售 agent 这是 over-engineering。

## 三、最大的元启示：**Threat Model 不一样，但"分层"思想通用**

Anthropic 的威胁模型是**外部攻击者要复制能力**——所以它的分层是：编译时 → 运行时 → 提示词层 → 遥测层 → 行为层。

**你的销售 agent 的威胁模型完全不同**，主要是三个内部风险：

| 风险 | 对应 Anthropic 模式 |
|---|---|
| **幻觉造成商务承诺** （报错价/交期/性能） | Tool gating（机制 1）+ Kill switch（机制 5） |
| **PII / 商业秘密泄露**（送到第三方 LLM） | 数据最小化 + Hash（机制 3） |
| **Prompt injection** （客户邮件里塞"忽略之前指令"，让 agent 切换人格） | Undercover-style 角色锁定（机制 2） |

**别为了反蒸馏而反蒸馏**——你不是 Anthropic，IP 保护不是你的核心痛点。但 Anthropic 这套**"假设系统会被攻破，每一层都独立防御"**的思路，对你处理上述三个风险**完全适用**。

## 四、给你 MVP 第一版的具体建议

回到你文档第 80-88 行的 MVP（输入客户名 → 输出画像/匹配/话术/CRM 草稿），按上面 5 条改造一下，**只需要做 3 件事**就能拿到 80% 的好处：

1. **从一开始就分 persona**：`research_agent`（销售内部用）和 `draft_agent`（生成客户面向内容）走两套 tool registry，**永不互通**。
2. **所有客户标识做 hash 才送 LLM**：内部 ID 转 hash，prompt 里只有 hash + 公开信息。
3. **加 feature flag 服务**：哪怕第一版只有 `enable_draft_generation` 一个开关，但要有这个能力。

这三条加起来一两天的工程量，但能直接挡住未来 80% 的事故。**比急着上 CRM 集成、邮箱 webhook 重要得多。**

需要的话我可以帮你把这个 MVP 的最小架构图画出来——Agent / Tool / Flag / Telemetry 怎么连。

---

### Turn 4  *(2026-05-14 06:20)*

**User:**

需要

**Assistant:**

下面是最小可用架构。先看全图，再讲每个组件，最后给个真实请求的流转例子和搭建顺序。

## 一、整体架构图

```
┌──────────────────────────────────────────────────────────────────────┐
│                      User (销售 / 销售总监 / 客服)                   │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │ JWT { user_id, role, context }
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ① Persona Router                                                    │
│    · 根据 role × context 决定本 session 用哪个 persona                │
│    · 一旦决定，整个 session 不可切换  (Undercover 的 no-force-OFF)   │
└────┬────────────────────────────────────────────────────────┬────────┘
     │                                                        │
     │ persona = research_agent                               │ persona = draft_agent
     ▼                                                        ▼
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│  ② research_agent               │         │  ②' draft_agent                 │
│     (销售内部用，能看成本)      │         │     (生成客户面向内容)          │
└────┬────────────────────────────┘         └────┬────────────────────────────┘
     │                                           │
     ▼                                           ▼
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│  ③ Internal Tool Registry       │         │  ③' Customer-Safe Tool Registry │
│  ✓ search_company (天眼查)      │         │  ✓ rag_public_kb                │
│  ✓ rag_internal_kb              │         │  ✓ generate_email_draft         │
│  ✓ rag_public_kb                │         │  ✓ generate_wechat_draft        │
│  ✓ get_cost_data                │         │  ✓ summarize_meeting_notes      │
│  ✓ get_history_pricing          │         │  ✗ get_cost_data    ← 物理隔离  │
│  ✓ check_blacklist              │         │  ✗ get_history_pricing          │
│  ✓ score_lead                   │         │  ✗ check_blacklist              │
└────┬────────────────────────────┘         └────┬────────────────────────────┘
     │                                           │
     └────────────────────┬──────────────────────┘
                          ▼
            ┌──────────────────────────────────┐
            │  ④ PII Tokenizer  (双向 proxy)    │
            │     出向：客户名/手机/金额 → hash │
            │     入向：hash → 原值 (仅 UI 层) │
            └──────────────┬───────────────────┘
                           ▼
            ┌──────────────────────────────────┐
            │  ⑤ LLM API  (Claude / 千问 等)    │
            └──────────────┬───────────────────┘
                           ▼
            ┌──────────────────────────────────┐
            │  ⑥ Attribution Wrapper            │
            │     给每条输出打:                 │
            │     { generated_by, confidence,   │
            │       sources[], needs_review }   │
            └──────────────┬───────────────────┘
                           ▼
            ┌──────────────────────────────────┐
            │  ⑦ Audit Log  (只存 hash 版)      │
            └──────────────────────────────────┘

                    ▲ ▲ ▲ (每次 agent 启动 + 每次 tool 调用前都查)
                    │ │ │
┌───────────────────┴─┴─┴─────────────────────────────────────────┐
│  ⑧ Feature Flag / Kill Switch  (Redis 即可，进阶用 GrowthBook)  │
│     enable_research_basic                                       │
│     enable_draft_generation         ← 出事第一时间关            │
│     enable_cost_data_lookup                                     │
│     kill_all_agent_actions          ← 红色大按钮                │
└─────────────────────────────────────────────────────────────────┘
```

## 二、组件说明（带技术选型）

| # | 组件 | 一句话职责 | 推荐选型 | 复杂度 |
|---|------|----------|---------|-------|
| ① | Persona Router | 锁定本 session 的 agent 身份 | 自己写 50 行 dispatcher | 低 |
| ② / ②' | Agent | 推理 + 工具调用 loop | **Claude Agent SDK**（别上 LangGraph，过度抽象） | 低 |
| ③ / ③' | Tool Registry | 每个 persona 一份**独立**的工具清单 | YAML / Python dict，**禁止合并** | 低 |
| ④ | PII Tokenizer | 出口脱敏 + 入口还原 | HMAC-SHA256(key, value) → 截前 12 位 | 低 |
| ⑤ | LLM API | 推理 | Claude Sonnet 4.6 起步，敏感场景走自部署 | - |
| ⑥ | Attribution Wrapper | 输出强制带元数据 | Pydantic schema 包一层 | 低 |
| ⑦ | Audit Log | 写入数据库 / 文件 | PostgreSQL 或 ClickHouse，**绝不**送第三方 trace 工具 | 中 |
| ⑧ | Feature Flag | 远程开关 | **第一版 Redis 一个 hash 就够**；上量后换 Unleash | 低 |

**关键约束**（呼应反蒸馏体系的"物理隔离"思想）：
- ③ 和 ③' 是**两份独立的 Python 模块**，不要写一份带 `if persona == 'internal'` 的开关。开关是概率性的，import 边界是确定性的。
- ④ 是个**强制中间件**，绕过它的 LLM 调用应该 CI 阶段就拒绝合入（写个简单的 grep 检查）。
- ⑧ 检查放在 **tool 调用前**而不是 agent 启动时——这样关开关瞬间生效，不用等用户重开 session。

## 三、一次真实请求的流转

**场景**：销售小张在系统里输入"深圳市XX化工有限公司"，要求生成画像 + 切入点 + 开场邮件。

```
1. 登录 → JWT { user_id: 1024, role: "sales", context: "research" }

2. Persona Router → 锁定 research_agent，本 session id=s_8821 不可改

3. research_agent 启动
   ├─ Feature Flag 查询: enable_research_basic = true ✓
   ├─ 加载 Internal Tool Registry (7 个工具)
   └─ 开始 reasoning loop

4. Agent 决定调用 search_company("深圳市XX化工有限公司")
   ├─ Feature Flag 查询: kill_all_agent_actions = false ✓
   ├─ 调用天眼查 API → 拿到结构化公司信息
   └─ 返回 agent

5. Agent 决定调用 rag_internal_kb("化工 + 该客户行业产品线")
   → 拿到历史相似客户的成交记录

6. Agent 决定调用 get_history_pricing(industry="化工", region="深圳")
   → 拿到内部历史报价区间 (含成本，敏感数据)

7. Agent 准备生成画像，调 LLM：
   ┌──────────────── 出向经过 PII Tokenizer ────────────────┐
   │ "深圳市XX化工有限公司" → "cust_a91f3c"                  │
   │ "李总 13800138000"     → "contact_7e2b / phone_3c8d"   │
   │ "上次报价 ¥120,000"    → "price_band_M"                 │
   └─────────────────────────────────────────────────────────┘
   送 LLM 的 prompt 里**没有任何**原始客户标识

8. LLM 返回画像 + 切入点 (内容仍然引用 cust_a91f3c)

9. Attribution Wrapper 包装:
   {
     content: "...",
     generated_by: "sales-agent-v0.1",
     model: "claude-sonnet-4-6",
     confidence: 0.82,
     sources: ["天眼查:s_8821_t1", "内部KB:doc_4521", "历史报价:hp_88"],
     needs_human_review: true,
     session_id: "s_8821"
   }

10. 写入 Audit Log (仍然是 hash 版)

11. UI 层做反向还原: cust_a91f3c → "深圳市XX化工有限公司"
    展示给小张，每段后面带 [Source] 链接

12. 小张点"生成开场邮件"
    ↓
    ⚠️ 这里**不是**在同一个 session 继续，
    而是 Persona Router 重新分发：
    role=sales, context="draft" → draft_agent 启动 (新 session)
    ↓
    draft_agent 的 Tool Registry 里**根本没有** get_cost_data
    即使前面 research_agent 拿到的成本数据，draft_agent 也读不到
    ↓
    它只能基于公开信息 + 销售切入点起草邮件
    → 天然不会泄露成本和黑名单信息
```

**这就是 Anthropic 反蒸馏的核心思想落地：分层 + 物理隔离 + 默认拒绝。**

## 四、搭建顺序（按"基础设施先行"原则）

每天 4-6 小时，1 人，可以这样排：

| Day | 做什么 | 为什么先做 |
|-----|------|---------|
| **1** | ④ PII Tokenizer + ⑧ Feature Flag (Redis) 骨架 | 这俩是**地基**，后加痛苦——尤其 Tokenizer，没它写好的工具全要返工 |
| **2** | ⑥ Attribution Schema + ⑦ Audit Log 写入 | 还是地基，让所有 agent 输出从 Day 3 起就自带元数据 |
| **3** | ② research_agent + 3 个 tool（search_company / rag_public_kb / score_lead）| 先验证推理 loop，不碰敏感数据 |
| **4** | 加内部 tool（get_cost_data / get_history_pricing / check_blacklist） | 验证 PII Tokenizer 工作正常 |
| **5** | ②' draft_agent + 邮件/微信草稿工具 | 这步最考验 ③/③' 的隔离是否真做到了 |
| **6** | ① Persona Router + UI 联调 | 把所有东西串起来 |
| **7** | 红队测试：故意 prompt injection 试试能不能让 draft_agent 吐出成本数据 | **必做**，能立刻暴露架构漏洞 |

## 五、最容易踩的 3 个坑（提前预警）

1. **想偷懒合并两个 agent**："一份代码 + 一个 `is_internal` 参数就够了"——**别这么干**。短期省 100 行代码，长期某次改 prompt 忘了判断分支，客户邮件里出现"该客户上次报价 12 万、毛利 30%"，这单子就没了。
2. **PII Tokenizer 用随机 hash**：必须用**稳定 hash**（HMAC + 固定 key），不然同一个客户在不同 session 里 hash 不一样，agent 无法跨会话理解"这是同一家公司"。
3. **Feature Flag 只在启动时查一次**：必须**每次 tool 调用前都查**。否则 agent 跑着跑着你按了 kill switch，但当前 session 仍然在调危险工具。

---

这个架构第一版**总代码量大概 1500-2500 行 Python**，一个人 1-2 周能跑起来。要不要我把 ④ PII Tokenizer 和 ⑧ Feature Flag 这两个"地基"组件的具体代码骨架写出来？这俩是后续所有东西的依赖。

---

### Turn 5  *(2026-05-14 06:26)*

**User:**

交付物是什么，怎么交付给前端

**Assistant:**

回答这个问题分两层：**"交付物"是 agent 产出的业务对象**，**"怎么交付"是 API 协议**。我把两层分开讲，最后给一个端到端例子。

## 一、交付物：10 个业务对象（不是 10 段文本）

最常见的错误：让 agent 返回 markdown 字符串，前端再去解析。**别这么干**。Agent 的输出应该是**强类型业务对象**，markdown 只是其中一个字段。

按你 MVP 的 5 个能力，对应的交付对象大概是这 10 个：

| # | 对象 | 由谁触发 | 关键字段 | 是否需人工确认 |
|---|------|---------|---------|-------------|
| 1 | `CompanyProfile` 客户画像 | 客户研究 | business, scale, signals, supplier_clues | 否 |
| 2 | `LeadScore` 线索评分 | 线索评分 | grade (A/B/C), score, reasons[] | 否 |
| 3 | `ProductMatch[]` 产品匹配 | 客户研究 | product_id, match_score, selling_points[] | 否 |
| 4 | `TalkingPoints` 切入点 | 客户研究 | hooks[], questions_to_ask[], **don't_promise[]** | 否 |
| 5 | `EmailDraft` 邮件草稿 | 跟进助手 | subject, body, tone, **redactions[]** | **是** |
| 6 | `WeChatDraft` 微信开场 | 跟进助手 | body (≤200 字), variants[] | **是** |
| 7 | `MeetingBriefing` 拜访 brief | 跟进助手 | agenda, customer_concerns, your_asks | 否 |
| 8 | `MeetingNotes` 拜访纪要 | CRM 整理 | needs, budget, competitors, dm, next_action | 否 |
| 9 | `CRMFieldUpdate[]` CRM 草稿 | CRM 整理 | field, current, suggested, diff_reason | **是** |
| 10 | `NextActions[]` 跟进计划 | 跟进助手 | action, due_date, owner, blocker | 否 |

**每个对象都裹一层 `AgentOutput<T>` envelope**（呼应上次架构图里的 ⑥ Attribution Wrapper）：

```typescript
interface AgentOutput<T> {
  type: "CompanyProfile" | "EmailDraft" | ...;  // 给前端做 discriminated union
  payload: T;
  meta: {
    generated_by: string;        // "sales-agent-v0.1"
    model: string;
    session_id: string;
    confidence: number;          // 0-1
    sources: Source[];           // 每条证据来源
    needs_human_review: boolean;
    redactions: string[];        // 这次输出主动避开的字段（透明性）
  };
}

interface Source {
  id: string;                    // "天眼查:s_8821_t1"
  label: string;                 // 给用户看的人话: "天眼查·工商信息"
  cited_in: number[];            // 这个 source 支持了 payload 里哪几句话（句子索引）
}
```

`cited_in` 这个字段很关键——前端可以做**句子级溯源高亮**，鼠标 hover 某句话就高亮对应来源。这是反幻觉的 UI 抓手。

## 二、交付协议：SSE + 两个端点

不要用：
- ❌ 纯 REST（agent 跑 30 秒，前端转圈圈 30 秒，用户跑了）
- ❌ WebSocket（双向、要心跳、Nginx 配置复杂，杀鸡用牛刀）
- ❌ GraphQL Subscription（同上）
- ❌ 把整个 markdown 流式返回（前端没法做卡片化 UI）

**用 SSE（Server-Sent Events）**。原因：

- 单向流，浏览器原生 `EventSource` 支持
- HTTP/1.1 长连接，Nginx 不用特殊配置
- 断线重连有 `Last-Event-ID` 机制
- 比 WebSocket 简单一个数量级

只暴露 **2 个端点**：

```
POST /api/agent/run          → 启动一次 agent run，返回 SSE 流
POST /api/agent/decide       → 对 needs_human_review 的输出做决策
```

### 启动 run

```http
POST /api/agent/run
Authorization: Bearer <jwt>     # 里面带 role + context，决定 persona
Content-Type: application/json

{
  "intent": "company_research",   // company_research / draft_email / summarize_meeting / ...
  "input": {
    "company_name": "深圳市XX化工有限公司"
  },
  "session_id": null              // 续接对话时传上次的 session_id
}
```

返回 `text/event-stream`，事件类型见下表。

### Agent 决策端点

当 SSE 流里出现 `requires_action` 事件时，前端拿到 `action_id`，弹卡片让销售选 approve / edit / reject：

```http
POST /api/agent/decide
{
  "action_id": "act_91ab",
  "decision": "approve" | "edit" | "reject",
  "edits": { ... }            // decision=edit 时带修改后的 payload
}
```

后端记到 Audit Log，并继续推进 agent（如果 agent 在等这个决策）。

## 三、SSE 事件 Schema（前端只认这 8 种）

```typescript
type AgentEvent =
  | { event: "session";        data: { session_id: string; persona: string } }
  | { event: "tool_start";     data: { tool: string; display: string; args_redacted: object } }
  | { event: "tool_end";       data: { tool: string; ok: boolean; summary: string } }
  | { event: "text_delta";     data: { delta: string } }                    // 流式 token
  | { event: "output";         data: AgentOutput<unknown> }                  // ★ 一个完整业务对象
  | { event: "requires_action";data: { action_id: string; output: AgentOutput<unknown> } }
  | { event: "error";          data: { code: string; message: string } }
  | { event: "done";           data: { session_id: string; usage: {...} } };
```

**关键设计点**：

- `tool_start` 的 `display` 字段是给用户看的中文（"正在查询天眼查..."），`tool` 是机器名（"search_company"）。前端不需要懂工具名映射。
- `args_redacted` 已经在后端 PII Tokenizer 处理过，前端**永远不会**收到原始客户手机号、成本数据等。
- `text_delta` 是 LLM 流式输出的 markdown 片段，前端拼起来做"正在思考"的可视化即可。
- `output` 才是**真正的交付物**，是结构化业务对象，进卡片渲染。
- 一次 run 里可以有**多个** `output` 事件（比如客户研究会先吐 `CompanyProfile`，再吐 `LeadScore`，再吐 `TalkingPoints`）。

## 四、端到端例子：销售研究一个客户

**前端代码（30 行能跑起来）：**

```typescript
const es = new EventSource("/api/agent/run", { /* POST 用 fetch + ReadableStream 也行 */ });

es.addEventListener("session",        (e) => setSessionId(JSON.parse(e.data).session_id));
es.addEventListener("tool_start",     (e) => activityLog.push(JSON.parse(e.data).display));
es.addEventListener("tool_end",       (e) => activityLog.update(JSON.parse(e.data).summary));
es.addEventListener("text_delta",     (e) => thinkingPanel.append(JSON.parse(e.data).delta));
es.addEventListener("output",         (e) => {
  const out: AgentOutput<unknown> = JSON.parse(e.data);
  // 根据 out.type 渲染不同卡片
  cardStack.push(renderCard(out));
});
es.addEventListener("requires_action", (e) => {
  const { action_id, output } = JSON.parse(e.data);
  openConfirmDialog(output, (decision, edits) => {
    fetch("/api/agent/decide", { method: "POST", body: JSON.stringify({ action_id, decision, edits }) });
  });
});
es.addEventListener("done",  () => es.close());
es.addEventListener("error", (e) => toast.error(JSON.parse(e.data).message));
```

**实际流过来的事件序列（销售输入"深圳市XX化工"）：**

```
event: session
data: {"session_id":"s_8821","persona":"research_agent"}

event: tool_start
data: {"tool":"search_company","display":"正在查询工商信息...","args_redacted":{"name":"<COMPANY>"}}

event: tool_end
data: {"tool":"search_company","ok":true,"summary":"找到 1 条工商记录，注册资本 2000 万"}

event: tool_start
data: {"tool":"rag_internal_kb","display":"正在匹配相似客户案例..."}

event: tool_end
data: {"tool":"rag_internal_kb","ok":true,"summary":"找到 3 个相似客户成交案例"}

event: text_delta
data: {"delta":"基于工商信息和"}

event: text_delta
data: {"delta":"行业匹配..."}

event: output
data: {
  "type":"CompanyProfile",
  "payload":{
    "business":"涂料中间体生产",
    "scale":"中型 (员工 200-500)",
    "signals":["近 6 个月招聘 R&D","拿到环保改造补贴"],
    "supplier_clues":["疑似从 XX 集团采购溶剂"]
  },
  "meta":{
    "confidence":0.84,
    "sources":[
      {"id":"qcc:s_8821_t1","label":"企查查·工商","cited_in":[0,1]},
      {"id":"kb:doc_4521","label":"内部 KB·相似案例","cited_in":[2,3]}
    ],
    "needs_human_review":false
  }
}

event: output
data: {"type":"LeadScore","payload":{"grade":"A","score":82,"reasons":[...]}, ...}

event: output
data: {"type":"TalkingPoints","payload":{"hooks":[...],"don't_promise":["具体交期","COA 指标"]}, ...}

event: done
data: {"session_id":"s_8821","usage":{"tokens":4521,"tools":2,"cost_cny":0.18}}
```

**前端渲染结果**（伪 UI）：

```
┌─────────────────────────────────────────────────────────┐
│ 🔍 深圳市XX化工有限公司                                 │
│                                                         │
│ [活动日志]                                              │
│   ✓ 查询工商信息 (找到 1 条记录)                       │
│   ✓ 匹配相似客户案例 (找到 3 个)                       │
│                                                         │
│ ┌── 客户画像 ─────────────── 置信度 84% ─┐              │
│ │ 涂料中间体生产                          │              │
│ │ 中型 (员工 200-500)                     │              │
│ │ 📍 近 6 个月招聘 R&D ¹                  │              │
│ │ 📍 拿到环保改造补贴 ¹                   │              │
│ │ 来源: ¹ 企查查·工商  ² 内部 KB           │              │
│ └────────────────────────────────────────┘              │
│                                                         │
│ ┌── 线索评分 ──────────────────── A 级 ─┐              │
│ │ 82 分                                  │              │
│ │ • 行业匹配度高                          │              │
│ │ • 有明确扩产信号                        │              │
│ └────────────────────────────────────────┘              │
│                                                         │
│ ┌── 切入点 ─────────────────────────────┐              │
│ │ 推荐话题: 环保溶剂替代方案              │              │
│ │ ⚠️ 不要承诺: 具体交期、COA 指标         │              │
│ │ [生成开场邮件] [生成微信话术]           │              │
│ └────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────┘
```

点"生成开场邮件"→ 走第二次 `/api/agent/run`，`intent: "draft_email"`，**Persona Router 切到 draft_agent**，流回来 `EmailDraft` 类型的 output，带 `needs_human_review: true`，前端弹确认框，销售改完点 approve 才进 CRM。

## 五、前后端契约怎么维护

**别手动维护类型**。流程：

1. 后端用 **Pydantic** 定义所有 `AgentOutput<T>` 的 schema
2. 跑 `pydantic` → `JSON Schema` → `openapi-typescript` 生成 `.ts` 类型文件
3. 前端 `import { AgentOutput, CompanyProfile, ... } from './generated/agent-types'`
4. 加进 CI：schema 变了自动重新生成，前端编译错误暴露不兼容

这一步省掉的不是代码量，是**联调时的撕逼**。

## 六、不要做的事

| 反模式 | 为什么 |
|------|-------|
| 让 agent 直接返回 markdown，前端 marked.js 渲染 | 没法做卡片化 UI，没法做确认门，没法做溯源 |
| 用 WebSocket | MVP 阶段不需要双向，纯加复杂度 |
| 自己造 SDK 给前端 | 一个 fetch + EventSource 够了，造 SDK 后期维护成本高 |
| 把 session 状态放前端（让前端每次发完整 history） | PII vault 在后端，前端拿不到原始数据；session state 必须在后端 |
| 用 streaming text 模拟所有输出 | text 给"思考过程"用就好，结构化输出走 `output` 事件 |
| 一次 run 只能产出一个对象 | 客户研究天然要吐画像 + 评分 + 切入点 3 个对象，一次 run 多个 output 是正常的 |

---

**总结一句**：交付物是 **10 个强类型业务对象 + Attribution 元数据**，交付方式是 **SSE 流 + 8 种事件类型 + 2 个 REST 端点**。前端做 **卡片化 + 活动日志 + 句子级溯源 + 确认门**，不要做 markdown 渲染器。

要不要我把 `AgentOutput` envelope 和这 10 个业务对象的 Pydantic schema 写出来？写完直接能生成 TS 类型给前端用。

---
