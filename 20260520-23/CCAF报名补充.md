# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `315c78af-d3bf-4b23-bd8d-a46ae43c5f91`
- **范围**: Turn 1-3
- **导出时间**: 2026-05-22 14:32:15

---

### Turn 1  *(2026-05-22 06:02)*

**User:**

个人可以考CCAF吗？就是claude认证架构师证书

**Assistant:**

可以——CCAF（**Claude Certified Architect — Foundations**，Anthropic 的首个官方认证）本身就是发给**个人**的证书，不是企业资质。考过后拿到的是个人 LinkedIn 徽章。但有个现实门槛要说清楚。

## 谁能考

- **证书定位**：面向已经在用 Claude API / Claude Code 做开发的中高级开发者、架构师，官方定级约「301 级」，**不是入门考试**。这点跟你（在职 AI 工程师）很匹配。
- **报名渠道**：走 Anthropic 的 Skilljar 学习平台（`anthropic.skilljar.com`），需要提交 access request。
- **当前限制**：**免费名额**（早期 5000 个）只给 Anthropic 官方合作伙伴公司的员工；个人/非伙伴目前是**付费 $99 一次**，且公开报名仍在逐步开放中，可能需要排队或提交申请。也就是说「能考」，但不一定能立刻无门槛报上名。

> 这一条是几周前的信息，且我刚才查官方页面时服务器在过载，建议你直接去 `anthropic.skilljar.com` 看实时的报名状态。

## 考试形式

- 120 分钟，60 道多选题，场景题为主（6 个业务场景里随机抽 4 个）
- 在线监考，**不能查 Claude、文档或任何外部工具**
- 5 大考点：Agentic 架构与编排（27%）、工具设计与 MCP 集成（18%）、Claude Code 配置与工作流（20%）、提示工程与结构化输出（20%）、上下文管理与可靠性（15%）
- 2 个工作日出成绩

## 备考资源（这些对所有人免费开放）

官方建议先过这几门 Anthropic Academy 课程，**不需要是合作伙伴也能学**：
- Claude 101
- Building with the Claude API
- Introduction to Model Context Protocol
- Claude Code in Action

## 给你的建议

对你「在职 AI 工程师」的身份，这证书含金量和你日常工作重合度都不错，值得拿。但**考研清华 AI 才是你的主线**——CCAF 对考研复试/简历有一点加分但不是决定性的。务实做法：先免费把上面 4 门课刷完（这部分纯收益、还能直接提升工作能力），等公开报名通道更顺畅、或你公司恰好是 Anthropic 伙伴能拿免费名额时再交钱考。别为了 $99 的考试挤占考研复习时间。

Sources:
- [Claude Architect Certification: Requirements, Cost & Prep Strategy — Applied AI Tools](https://appliedai.tools/anthropic/claude-architect-certification/)
- [Anthropic Courses (Skilljar 官方平台)](https://anthropic.skilljar.com/)
- [Claude Certified Architect — Anthropic's First Official Certification (zenvanriel)](https://zenvanriel.com/ai-engineer-blog/claude-certified-architect-anthropic-certification-guide/)

---

### Turn 2  *(2026-05-22 06:06)*

**User:**

怎么提交 access request，报名网站提示我You aren’t signed in with a partner company employee email. Sign in with your partner email to register.

**Assistant:**

你遇到的这个提示说明了一个关键事实：**那个 access-request 页面本身不是面向个人的自助报名入口**——它会校验你登录邮箱的公司域名是否属于「Claude 合作伙伴公司」。用个人邮箱（Gmail/QQ 等）或非伙伴公司的邮箱，根本走不到「提交申请」那一步。

## 为什么卡住

目前 CCAF 考试**完全挂在 Claude Partner Network（合作伙伴网络）下面**，不存在「个人交 $99 直接报名」的公开通道。报名流程实际是：

> 公司加入合作伙伴网络 → Anthropic 按公司邮箱域名开通 Skilljar 权限 → 你用**公司邮箱**登录 → 才能在 access-request 页提交申请并约考

所以「怎么提交 access request」的真正答案是：**先解决邮箱资格**，没有伙伴邮箱时这一步无解。

## 你的三条现实路径

**1. 让你公司加入 Claude Partner Network（最正规、免费）**
- 入口：`anthropic.com/partners`（或搜 "Claude Partner Network" 公告页）
- 条件：组织是「把 Claude 带向市场」的——做 Claude 相关产品、集成、咨询/交付的公司都算。你是在职 AI 工程师，如果公司在用/做 Claude 相关业务，让对接 Anthropic 的同事或主管去申请，**入会免费**，前 5000 名伙伴员工还免考试费。
- 公司过审后，你用公司邮箱登录那个页面，提示就会消失。

**2. 现在就开始免费备考（任何邮箱都行）**
- Anthropic Academy 在 Skilljar 上的 **13 门课程对所有人免费开放**，不需要伙伴身份。
- 重点过：Claude 101、Building Applications with the Claude API、Introduction to MCP、Claude Code 开发课、Agent Skills。
- 这部分纯收益——既是考点覆盖，也直接提升你工作能力，跟考研复习也不冲突。

**3. 等公开 GA**
- Anthropic 已明确说 2026 年内还会推面向 seller / developer / 高级 architect 的认证，伙伴优先、之后逐步扩大。个人自助通道大概率会晚点开。

## 建议

别在「用个人邮箱硬提交 access request」上耗时间——这条路现在是死的。务实做法：

1. **今天**就用现在的邮箱去 Skilljar 把免费课刷起来；
2. **同时**问一句你们公司有没有/愿不愿意加入 Claude Partner Network——这是你能拿到考试资格、还能省 $99 的唯一捷径；
3. 公司这条走不通就先囤课，等公开通道。

对你「主线是考研清华 AI」来说，第 2 步推不动也别纠结，第 1 步的免费课才是稳赚的部分。

Sources:
- [How to Become a Claude Certified Architect in 2026 — LowCode Agency](https://www.lowcode.agency/blog/how-to-become-claude-certified-architect)
- [Claude Certified Architect: who can take the exam and why access is still limited — Data Studios](https://www.datastudios.org/post/claude-certified-architect-who-can-take-the-exam-and-why-access-is-still-limited)
- [Claude Architect Certification: Requirements & Cost — Applied AI Tools](https://appliedai.tools/anthropic/claude-architect-certification/)

---

### Turn 3  *(2026-05-22 06:24)*

**User:**

主线是考研清华 AI,帮我把这个记忆删除了

**Assistant:**

先确认一下再动手——你要删的这条记忆 `user_profile.md` 内容比 MEMORY.md 索引里那行摘要要多，而且**有矛盾点**：

- 它不只有「考研清华 AI」，还存了你的身份（在职 AI 工程师 / AI Agent+RAG）、美术基础、本科背景；
- 文件里目前写的是「**院校不再锁定清华**」「倾斜**中科院自动化所/计算所**」「在 AI/CS 和清华美院之间摇摆」——这跟你刚说的「主线是考研清华 AI」正好冲突。

所以「删掉这个记忆」有几种做法，差别挺大：

已删除：

- `user_profile.md` 文件已删掉
- `MEMORY.md` 索引里对应那一行也已移除

整条「用户身份与目标」记忆（含在职 AI 工程师、美术基础、本科背景、考研路径等全部内容）现在已经没有了。

一个提醒：以后我在新会话里就不会自动知道你「在职 AI 工程师 / 主线考研清华 AI」这些背景了，每次需要时得你重新说。如果哪天想重新存一条干净的、只写「主线 = 考研清华 AI」，跟我说一声就行。

---
