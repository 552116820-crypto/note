# Dify 可行性追问（续）

## 问：不能从CRM接口返回的订单信息中，用代码脚本挑选出超一个月没下单的客户吗？这样Dify就不会有并发问题

### 方案架构

```
HTTP 请求(调CRM接口，拿某销售的全部客户订单) → 一个 Code 节点批量处理 → 输出预警列表
```

没有迭代节点，没有并发问题。一次 HTTP 拿数据，一次 Code 算完，干净利落。

### 关键决策：要哪种精度？

#### 方案A：简单规则（超1个月没下单）

```python
# 一个 Code 节点就能搞定，不需要 NumPy
from datetime import datetime, timedelta

threshold = 30  # 天
today = datetime.now().date()
warnings = []

for customer in customers:
    days_since = (today - customer["last_order_date"]).days
    if days_since > threshold:
        warnings.append(customer)

return warnings
```

- Dify 默认 Code 节点就能跑，**零配置**
- 但所有客户用同一个阈值（30天），不区分下单频率高低的客户
- 月均下单的客户和季度下单的客户吃同一个标准，**误报会很多**

#### 方案B：保留 EWMA 算法，但在一个 Code 节点里批量跑

```python
# 需要自定义沙箱装 NumPy，或用纯 Python 重写
import numpy as np

warnings = []
for cid, orders in group_by_customer(all_orders):
    intervals = calc_intervals(orders)
    intervals = iqr_filter(intervals)
    predicted_cycle = ewma(intervals)
    days_since = (today - orders[-1]["order_date"]).days
    if days_since >= predicted_cycle * multiplier:
        warnings.append(...)

return warnings
```

- 每个客户有自己的预测周期，精度高
- 数据量不大（1个销售 ~100客户 ~2000条订单），一个 Code 节点几秒就算完
- 需要装 NumPy 或用纯 Python 实现算法

#### 两种方案对比

| 维度         | 方案A：固定阈值       | 方案B：EWMA预测       |
| ---------- | -------------- | ---------------- |
| 实现难度       | 极低，默认就能跑       | 中等，需配沙箱或重写算法     |
| 预警精度       | 低，一刀切          | 高，千人千面           |
| Dify 可行性   | 完全可行           | 可行               |
| NumPy 依赖   | 不需要            | 需要（或纯 Python 替代） |

### 结论

不管选 A 还是 B，**"一次拿数据 + 一个 Code 节点算完"这个架构是对的**，确实绕开了迭代和并发问题。

关键决策点是：能接受固定阈值的误报率吗？如果能，方案A在 Dify 里几乎零门槛就能落地。

---

## 问：公司有1000名销售，怎么触发客户预警的工作流？

### 方案一：外部脚本循环调 Dify API

```python
# 某个定时任务脚本
import requests, time

salesperson_ids = get_all_salesperson_ids()  # 1000个

for sid in salesperson_ids:
    requests.post("https://dify/v1/workflows/run", json={
        "inputs": {"salesperson_id": sid}
    })
    time.sleep(2)  # 控制节奏，避免打爆
```

- 1000个 × 2秒间隔 = **跑完要 33 分钟**
- 需要自己写并发控制、失败重试、进度追踪
- 问题：**这个脚本本身就是一个 batch 编排器**

### 方案二：Dify 内部迭代 1000 个销售

一次触发，Iteration 节点遍历 1000 个销售 ID：

```
开始 → HTTP 拿销售列表 → Iteration(1000次) → [HTTP拿订单 → Code计算 → 条件判断] → 汇总推送
```

- 1000次 × 2~3秒 = **30~50 分钟，大概率超时**
- 中间任何一个失败，后面全部跳过
- **不可行**

### 方案三：CRM 接口一次返回全部数据

如果 CRM 能提供一个**批量接口**，一次返回全部客户的最近下单日期：

```
开始 → HTTP 拿全部客户最近下单记录 → 一个 Code 节点算完 → 推送
```

- 只有 1 次 HTTP + 1 次 Code，**几秒到十几秒搞定**
- 没有迭代、没有并发问题
- 数据量：1000销售 × 100客户 = 10万行，Code 节点内存扛得住
- **前提是 CRM 能给这个批量接口**

### 对比

| 方案   | 需要额外写什么           |
| ---- | ----------------- |
| 方案一  | 一个 Python 定时脚本做调度 |
| 方案二  | 不可行               |
| 方案三  | 一个 CRM 批量查询接口     |

### 核心矛盾

**方案一**：写完那个调度脚本，它和现有的 `batch_predict.py` 做的事几乎一样——遍历、调用、错误处理、日志。只不过计算逻辑搬进了 Dify。多了一层网络调用，多了一个要维护的系统。

**方案三**：如果 CRM 能批量返回数据，一个 Code 节点确实能算完。但这个 Code 节点本质上就是 `cycle_predictor.py` + `warning_engine.py` 塞进了一个 Dify 节点——逻辑一样，只是换了个壳跑。

每次解决一个 Dify 的限制，都要在 Dify 外面补一块东西。补到最后：

```
外部定时触发 + 外部批量接口 + Dify Code 节点里塞完整算法 = 用 Dify 当了个代码运行器
```

这和直接跑 Python 服务相比，**多了 Dify 这一层，但没有多出任何价值**。

除非让 LLM 参与进来——比如算完后接一个 LLM 节点生成个性化跟进话术，那 Dify 才有存在的意义。推荐的混合架构：**Python 服务做算法计算 + Dify 做 LLM 增值（个性化话术、销售问答助手等）**。
