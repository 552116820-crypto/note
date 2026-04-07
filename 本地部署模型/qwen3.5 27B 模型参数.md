# Qwen3.5 27B 模型资料与参数显存关系

---

## Qwen3.5 27B 关键信息

### 发布时间
2026 年 2 月 16 日首发 Qwen3.5 系列，2 月 25 日发布 27B 版本。

### 模型架构
- **密集模型（Dense）**，270 亿参数全部激活，非 MoE
- 混合注意力架构：3:1 比例交替使用 **Gated DeltaNet 线性注意力层**和标准全局注意力层
- 原生视觉-语言模型（多模态）
- 上下文窗口：最高 **262K ~ 1M tokens**

### 性能评测
- 准确率从上代的 65.5% 提升至 **72.4%**，排名从第 51 位跃升至**第 8 位**
- 工具调用、搜索、编程等 Agent 评测中**超过 GPT-5 mini**
- 视觉理解能力超过 Qwen3-VL 旗舰和 Claude Sonnet 4.5
- 知乎上的评价是"Qwen3.5 系列，无脑选 27B"

### 本地部署
- BF16 全精度单卡 H20 可跑（~54GB 显存）
- FP8 量化性能与原模型**几乎一致**，是推荐的部署方式
- 4-bit 量化后仍显著强于 Qwen3.5 9B
- vLLM 部署命令示例：
  ```bash
  vllm serve Qwen/Qwen3.5-27B-FP8 --port 8000 --max-model-len 262144 --reasoning-parser qwen3
  ```

### 注意事项
- 有用户反馈 vLLM 上 27B 的并发性能不太理想，可能需要调优参数
- thinking mode 开启/关闭对性能影响较大
- 相比 35B-A3B（MoE 版），27B 推理速度更慢但精度更高

### 参考来源
- [Qwen3.5 系列大模型，无脑选 Qwen3.5-27B - 知乎](https://zhuanlan.zhihu.com/p/2011948376619499548)
- [阿里Qwen3.5-27B实测 - 知乎](https://zhuanlan.zhihu.com/p/2010679440431145266)
- [Qwen3.5本地部署终极指南 - 知乎](https://zhuanlan.zhihu.com/p/2010371388985319968)
- [Qwen3.5开源矩阵震撼发布！选型指南 - CSDN](https://blog.csdn.net/2401_85375151/article/details/158661902)
- [如何评价Qwen3.5的性能表现 - 知乎](https://www.zhihu.com/question/2006785281806861062)
- [1M Tokens/s: Qwen 3.5 on B200 GPUs with vLLM](https://medium.com/google-cloud/1-million-tokens-per-second-qwen-3-5-27b-on-gke-with-b200-gpus-161da5c1b592)
- [Qwen3.5 Usage Guide - vLLM Recipes](https://docs.vllm.ai/projects/recipes/en/latest/Qwen/Qwen3.5.html)
- [vLLM 部署 Qwen3.5 满血&量化版 - 知乎](https://zhuanlan.zhihu.com/p/2014273757963894922)
- [Qwen/Qwen3.5-27B - Hugging Face](https://huggingface.co/Qwen/Qwen3.5-27B)
- [QwenLM/Qwen3.5 - GitHub](https://github.com/QwenLM/Qwen3.5)

---

## 模型参数与显存的关系

### 核心公式

**显存 = 参数量 × 每个参数的字节数**

| 精度 | 每个参数占用 | 27B 模型权重显存 |
|---|---|---|
| FP32（32位） | 4 字节 | 27 × 4 = **108 GB** |
| BF16/FP16（16位） | 2 字节 | 27 × 2 = **54 GB** |
| FP8（8位） | 1 字节 | 27 × 1 = **27 GB** |
| INT4（4位） | 0.5 字节 | 27 × 0.5 = **13.5 GB** |

速算口诀：**1B 参数 BF16 ≈ 2GB 显存**。

### 实际运行时显存构成

模型权重不是全部，实际运行时显存还要装：

```
总显存 = 模型权重 + KV Cache + 激活值 + 框架开销
         ~~~~~~~~   ~~~~~~~~   ~~~~~   ~~~~~~~~
         上面算的    最大头      较小     1-2GB
```

**KV Cache** 是推理时的主要额外开销，和上下文长度、并发数成正比：

```
KV Cache ≈ 2 × 层数 × 隐藏维度 × 上下文长度 × 精度字节数 × 并发数
```

### Qwen3.5-27B（BF16，单请求）显存估算

| 上下文长度 | 模型权重 | KV Cache | 总计约 |
|---|---|---|---|
| 4K tokens | 54 GB | ~2 GB | ~57 GB |
| 32K tokens | 54 GB | ~16 GB | ~71 GB |
| 128K tokens | 54 GB | ~64 GB | ~120 GB（爆显存） |

单卡 H20（96GB）跑 27B BF16：
- 短对话（4-8K）：绰绰有余
- 中等上下文（32K）：可以
- 长上下文（128K+）：需要用 FP8 量化（权重降到 27GB），把显存留给 KV Cache

### 显存速算表

| 显存 | BF16 能跑的最大参数量 | 推荐模型 |
|---|---|---|
| 8 GB | ~3B | Qwen3.5-4B（INT4） |
| 16 GB | ~7B | Qwen3.5-9B（INT4） |
| 24 GB | ~12B | Qwen3.5-9B（BF16） |
| 48 GB | ~24B | Qwen3.5-27B（INT4） |
| 96 GB | ~48B | Qwen3.5-27B（BF16） |

一句话总结：**参数量 × 2 = BF16 显存（GB），剩下的显存留给 KV Cache 跑上下文**。
