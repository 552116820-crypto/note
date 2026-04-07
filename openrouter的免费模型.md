# OpenRouter 免费模型笔记

> 记录时间：2026-04-07

## Gemma (Google DeepMind)

- **Gemma 3**（2025年3月发布）：1B/4B/12B/27B，多模态，128K 上下文，140+ 语言
- **Gemma 4**（2026年4月2日发布）：最新版本，Apache 2.0 开源

## Qwen 千问 (阿里巴巴)

- 2025年2月品牌统一为"千问"（Qwen）
- **Qwen 3.5**：总参数 3970 亿，推理激活 1700 亿（MoE），性能对标 GPT-5.2，API 定价仅 Gemini 3 Pro 的 1/18
- **Qwen 3.6 Plus**（2026年4月）：代码能力全面增强，多模态识别与推理效率提升

## OpenRouter 免费模型列表（2026年4月）

共 29 个免费模型，无需信用卡，注册即可用。

### 主要免费模型

| 模型 | 上下文 | 特点 |
|------|--------|------|
| Qwen3.6 Plus | 1M | 视觉 + 工具调用 |
| Qwen3.6 Plus Preview | 1M | CoT 推理 |
| Qwen3 Coder 480B | - | 当前最强免费编程模型 |
| DeepSeek R1 | - | 推理能力强 |
| Llama 3.3 70B | - | Meta 开源 |
| Gemma 3 | - | Google 开源 |
| NVIDIA Nemotron 3 Super 120B | 262K | 工具调用 |
| Google Lyria 3 Pro Preview | 1M | 视觉 |
| Mistral 系列 | - | 法国开源 |

### 免费限制

- 约 20 请求/分钟，200 请求/天
- 提示词数据可能被用于模型改进
- 免费状态可能在预览期结束后变化

### 使用方式

OpenRouter 兼容 OpenAI API 格式：

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="YOUR_API_KEY",
)

# 指定模型
response = client.chat.completions.create(
    model="qwen/qwen3.6-plus:free",
    messages=[{"role": "user", "content": "Hello"}],
)

# 或使用自动路由（自动选择可用的免费模型）
response = client.chat.completions.create(
    model="openrouter/free",
    messages=[{"role": "user", "content": "Hello"}],
)
```

### 常用模型 ID

- `qwen/qwen3.6-plus:free`
- `qwen/qwen3.6-plus-preview:free`
- `openrouter/free`（自动路由）

### 429 限流问题

免费模型所有用户共享上游配额，高峰期容易触发 429 Rate Limit Error。解决方案：
1. 等几分钟重试
2. 代码中加重试逻辑（指数退避）
3. 到 https://openrouter.ai/settings/integrations 绑定自己的 API Key 获取独立限额
4. 使用付费模型（去掉 `:free` 后缀）

## 参考链接

- [OpenRouter 免费模型合集](https://openrouter.ai/collections/free-models)
- [OpenRouter 免费模型路由](https://openrouter.ai/openrouter/free)
- [Qwen3.6 Plus (free)](https://openrouter.ai/qwen/qwen3.6-plus:free)
- [Gemma 3 - Google DeepMind](https://deepmind.google/models/gemma/gemma-3/)


