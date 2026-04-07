
## 背景

硬件：单卡 NVIDIA H20（96GB HBM3），模型：Qwen3.5 27B，推理框架：vLLM。

H20 核心优势是 96GB 显存和 ~4 TB/s 显存带宽，适合大模型推理。27B 模型 BF16 全精度约需 ~54GB 显存，单卡轻松运行，无需量化。

---

## 什么是推理框架

"推理"指模型运行生成文本的过程（inference），不是指 AI 的思考/推理能力。

**推理框架**是用来加载模型、处理请求、生成回复的运行软件。类比：

- 模型权重 = 发动机
- 推理框架 = 汽车底盘和变速箱

vLLM 负责：
- 把模型加载到 GPU 显存
- 接收 prompt
- 高效调度 GPU 计算，生成 token
- 管理显存（PagedAttention 等优化）
- 支持并发请求

同类框架还有 SGLang、llama.cpp、TensorRT-LLM 等。

---

## 为什么需要推理框架

开源模型可以直接用 HuggingFace Transformers 裸跑，几行 Python 就行：

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen3.5-27B")
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen3.5-27B")

inputs = tokenizer("你好", return_tensors="pt").to("cuda")
output = model.generate(**inputs)
print(tokenizer.decode(output[0]))
```

但裸跑**慢**且**费显存**。vLLM 的优化让同样的硬件跑得更快、塞得更多、等得更短。

|  | 裸跑 (transformers) | vLLM |
|---|---|---|
| 生成速度 | 慢 | 快 2-4 倍 |
| 显存利用 | 浪费多 | PagedAttention 省显存 |
| 并发请求 | 一个一个排队 | 可以同时处理多个 |
| 长上下文 | 容易 OOM | 显存管理更智能 |

一个人用、不在乎速度，裸跑完全可以。追求速度或多人用，vLLM 就有意义了。

---

## vLLM 核心优化详解

### 1. PagedAttention（最核心的创新）

**问题：** LLM 生成时需要存储 KV Cache（每个 token 的注意力键值对），传统方式预分配一整块连续显存。

```
传统方式（HuggingFace Transformers）：
┌──────────────────────────────────┐
│ 请求A KV Cache（预分配2048长度）   │  ← 实际只用了 500，剩下 1548 浪费
├──────────────────────────────────┤
│ 请求B KV Cache（预分配2048长度）   │  ← 实际只用了 300，剩下 1748 浪费
├──────────────────────────────────┤
│         空闲显存                   │  ← 明明还有很多显存，但因为碎片化放不下新请求
└──────────────────────────────────┘
显存浪费率：60%~80%
```

**vLLM 的做法：** 借鉴操作系统的虚拟内存分页机制，把 KV Cache 拆成固定大小的 block（默认 16 tokens/block），按需分配，不连续也没关系。

```
vLLM PagedAttention：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ A0 │ A1 │ B0 │ A2 │ B1 │ C0 │ C1 │ A3 │  ← 不同请求的block交错存放
└────┴────┴────┴────┴────┴────┴────┴────┘
显存浪费率：<4%（只有最后一个block有内部碎片）
```

**数值：**
- 显存浪费：**60-80% → <4%**
- 同等显存下可服务的并发请求数：**提升 2-4 倍**

---

### 2. Continuous Batching（连续批处理）

**问题：** 传统 static batching 必须等一批请求全部生成完毕才能开始下一批。

```
Static Batching：
时间 →→→→→→→→→→→→→→→→→→
请求A: [=========生成完毕====]
请求B: [===短回复==]........... ← 生成完了还得等 A
请求C:                          [====等前一批结束才能开始====]
                    ↑ 这段时间 GPU 在空转
```

**vLLM 的做法：** 某个请求生成完就立刻踢出，新请求立刻插入，GPU 永远不闲着。

```
Continuous Batching：
时间 →→→→→→→→→→→→→→→→→→
请求A: [=========生成完毕====]
请求B: [===短回复==]
请求C:             [=立刻插入=====]  ← B 结束瞬间 C 就进来了
请求D:                        [==]  ← A 结束瞬间 D 进来
```

**数值：**
- 吞吐量对比 static batching：**提升 2-8 倍**（请求长度差异越大，优势越明显）

---

### 3. Prefix Caching（前缀缓存）

多个请求如果有相同的 system prompt，KV Cache 可以共享复用，不用重复计算。

```
请求1: [system prompt] + 用户问题A
请求2: [system prompt] + 用户问题B
                ↑ 这部分 KV Cache 只算一次，两个请求共享
```

**数值：**
- 相同前缀部分的计算和显存：**节省接近 100%**
- 对于 system prompt 长（比如几千 token）的场景，首 token 延迟（TTFT）降低 **50-80%**

---

### 4. 整体性能对比（vLLM 论文数据）

| 对比对象 | vLLM 吞吐提升 |
|---|---|
| HuggingFace Transformers | **最高 24 倍** |
| HuggingFace Text Generation Inference (TGI) | **2.2 - 3.5 倍** |

单卡 H20 + Qwen3.5 27B 场景估算：

| 指标 | 裸跑 transformers | vLLM |
|---|---|---|
| 生成速度（tokens/s，单请求） | ~30-40 | ~50-70 |
| 最大并发请求 | 1 | 10-20+ |
| 首 token 延迟（TTFT） | ~1-3s | ~0.3-1s |
| 显存中可缓存的 context 长度 | 短（浪费多） | 长（利用率 >96%） |

> 注意：以上单卡数值是基于已知 benchmark 的估算，实际取决于 prompt 长度、生成长度、并发数等。
