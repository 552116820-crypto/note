# Karpathy 会话记录

保存时间：2026-04-10  
目标路径：`E:\note\20260409\Karpathy.md`

说明：本文记录本次会话中用户与助手的可见内容，不包含系统/开发者隐式指令和工具调用过程。

## 用户

你知道kapathy吗

## 助手

你说的应该是 **Andrej Karpathy** 吧，常被中文圈叫“卡帕西”。

他是很有名的 AI 研究员和工程师，主要和深度学习、计算机视觉、大模型教育内容有关。比较广为人知的经历包括：

- 斯坦福博士，导师是 Fei-Fei Li
- 曾在 OpenAI 工作
- 曾任 Tesla AI 负责人，参与 Autopilot / 自动驾驶相关工作
- 做过很多高质量 AI 教学内容，比如 `CS231n`、`Neural Networks: Zero to Hero`
- 在大模型圈很出名，尤其擅长把复杂概念讲得很清楚

如果你是看到 “kapathy” 这个名字，多半是把 **Karpathy** 拼错了。

## 用户

Karpathy 知识库

## 助手

你说的应该是 **Karpathy 最近提出的 LLM Wiki / LLM Knowledge Base 模式**。

核心不是“把资料塞进向量库再 RAG”，而是：

> 把原始资料放进 `raw/`，让 LLM 逐步“编译”成一个可读、可链接、可维护的 Markdown Wiki。

**一句话版：**  
RAG 是“每次提问时临时找资料”；Karpathy 知识库是“先让模型把资料整理成长期可积累的 wiki，以后直接基于 wiki 推理”。

**基本结构：**

```text
knowledge/
├── raw/              # 原始资料：论文、文章、网页、笔记、PDF 转 Markdown
│   ├── articles/
│   ├── papers/
│   ├── notes/
│   └── transcripts/
├── wiki/             # LLM 生成和维护的知识库
│   ├── index.md      # 总索引
│   ├── log.md        # 操作日志
│   ├── concepts/
│   ├── people/
│   ├── projects/
│   └── summaries/
└── AGENTS.md         # 给 AI agent 的维护规则
```

**工作流：**

1. **Ingest / 摄入**  
   你把文章、论文、会议记录、网页剪藏放进 `raw/`。LLM 读取后，生成摘要页、概念页、人物页，并更新 `wiki/index.md`。

2. **Compile / 编译**  
   重点不是简单摘要，而是把新资料合并进已有知识结构：补充旧页面、建立反向链接、标注矛盾、更新结论。

3. **Query / 查询**  
   之后提问时，LLM 优先读 `wiki/index.md` 和相关 wiki 页面，而不是每次重新从原始资料里临时拼答案。

4. **Lint / 体检**  
   定期让 LLM 检查知识库：孤立页面、过期结论、互相矛盾的说法、缺少来源的断言、需要补充研究的问题。

**它和 RAG 的区别：**

| 维度 | 传统 RAG | Karpathy LLM Wiki |
|---|---|---|
| 核心动作 | 查询时检索片段 | 摄入时编译知识 |
| 存储形式 | chunk + embedding | Markdown wiki |
| 可读性 | 向量库不直观 | 人可以直接读、改、diff |
| 知识积累 | 每次问题重新拼 | wiki 会持续变厚 |
| 风险 | 检索漏召回 | LLM 写错后污染 wiki |

**适合场景：**

- 个人研究：论文、博客、视频字幕、GitHub repo
- 项目记忆：架构决策、会议记录、需求变更
- 学习系统：课程笔记、概念图谱、错题、复习卡
- 商业分析：竞品资料、客户访谈、市场报告
- 自我知识库：日记、目标、健康、心理、长期计划

**最重要的实践规则：**

- `raw/` 是事实来源，尽量只读，不让模型改。
- `wiki/` 是模型维护层，人主要审阅，不频繁手改。
- 每个 wiki 页面都要带来源。
- 给页面加元数据，比如 `confidence`、`last_updated`、`sources`。
- 用 Git 管理整个知识库，方便看模型改了什么。
- 不要把模型生成内容当作事实，关键结论必须能追溯到 `raw/`。

一个可用的页面模板大概这样：

```markdown
---
title: "Transformer Attention"
type: concept
confidence: 0.82
last_updated: 2026-04-10
sources:
  - raw/papers/attention-is-all-you-need.md
  - raw/notes/karpathy-lecture.md
---

# Transformer Attention

## Summary

## Key Claims

## Evidence

## Related Pages

## Open Questions
```

资料来源：  
- Karpathy 官方 Gist：[LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)  
- VentureBeat 报道：[Karpathy shares LLM Knowledge Base architecture](https://venturebeat.com/data/karpathy-shares-llm-knowledge-base-architecture-that-bypasses-rag-with-an/)  
- DeepakNess 摘要：[LLM Knowledge Bases post by Andrej Karpathy](https://deepakness.com/raw/llm-knowledge-bases/)  

我也可以直接给你整理一版中文的 `AGENTS.md`，用来让 Codex/Claude/Cursor 按 Karpathy 这个模式维护你的本地知识库。

## 用户

把这次会话的所有内容存进E:\note\20260409\Karpathy.md
