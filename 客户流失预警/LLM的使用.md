# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019d8aac-5c4d-75b3-ad5a-ec64483c7975`
- **范围**: Turn 1-2
- **导出时间**: 2026-04-14 15:00:52

---

### Turn 1  *(2026-04-14 06:36)*

**User:**

这个项目要不要用ai大模型预测下单周期，还是只用python算法引擎

**Assistant:**

我先快速看一下当前项目里流失预警引擎和数据结构的形态，再给你一个贴合这个仓库的判断。

我看到了 README 里已经把核心定位写成「基于 EWMA 周期预测」，也就是现在不是空白决策。再确认一下 `engine.py` 里的预测方式和可解释性边界。

建议：**先只用 Python 算法引擎做生产版，不要用通用 AI 大模型直接预测下单周期**。

这个项目当前已经是 EWMA 周期预测路线：[README.md](/mnt/e/churn-warning-engine/README.md) 和 [app/churn/engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):149 里都有明确实现：订单日期去重排序、计算间隔、IQR 去异常、EWMA 加权、置信度评分。这个场景的核心是结构化订单时间序列，不是自然语言推理；Python 规则/统计模型更可解释、稳定、便宜，也更容易向销售解释“为什么预警”。

大模型可以作为辅助，但不建议放在核心预测链路里。原因是时间序列大模型确实有进展，比如 Chronos 这类模型把时间序列 token 化后做概率预测，并在一些 zero-shot 场景表现不错；但也有 2025 年 ACL 论文指出，通用 LLM 做 zero-shot 时间序列预测对噪声敏感，常常不如简单领域模型稳定。也就是说，除非你有足够数据做回测证明它明显提升，否则它会增加成本、延迟、隐私风险和不可解释性。

更稳的路线是：

1. **短期**：保留现在的 Python EWMA 引擎作为 baseline 和生产默认。
2. **中期**：补一个回测评估脚本，按历史订单模拟预测，比较 `MAE / MAPE / 命中率 / 预警 precision-recall`。
3. **数据量足够后**：如果 EWMA 不够，再引入 Python ML 模型，比如 `GradientBoostingRegressor`、LightGBM、XGBoost，预测“下一次下单间隔”或“未来 N 天未下单概率”。scikit-learn 的梯度提升回归也支持 quantile loss，可以做预测区间。
4. **大模型适合放在外围**：生成销售跟进话术、解释预警原因、总结客户备注、结合 CRM 文本记录补充“可能原因”，而不是替代周期预测。

我的结论是：**核心预测用 Python 算法引擎；大模型只做预警解释和销售辅助。** 后续如果要上“AI”，也优先做可回测的时间序列/表格 ML 模型，而不是直接让通用 LLM 猜下单周期。

参考：Chronos 时间序列模型论文：https://arxiv.org/abs/2403.07815；LLM 时间序列稳健性论文：https://arxiv.org/abs/2506.00457；scikit-learn 梯度提升回归文档：https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.GradientBoostingRegressor.html

---

### Turn 2  *(2026-04-14 06:43)*

**User:**

所以我这个项目，客户流失预警功能是用不到大模型的

**Assistant:**

对，**就你这个项目当前的客户流失预警核心功能来说，不需要大模型**。

更准确地说：下单周期预测、关键产品断购、月度采购金额波动这些核心判断，应该继续用现在这种 Python 统计/规则算法。你的实现已经在 [app/churn/engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):149 做了周期预测：订单日期去重、计算间隔、IQR 去异常、EWMA 加权、置信度评分；这类结构化订单数据用大模型反而不如这种算法可解释、稳定、便宜。

但不是“大模型完全没价值”。它更适合放在预警之后做辅助，比如：

- 根据 `feature_summary` 生成销售跟进话术
- 根据客户备注、拜访记录、投诉、回款记录判断可能原因
- 帮运营总结“为什么这个客户可能流失”
- 给人工丰富预警的 [enrich](/mnt/e/churn-warning-engine/app/churn/service.py):241 流程提供草稿

所以建议架构是：**Python 算法引擎负责发现预警，大模型只做解释和沟通辅助**。核心链路别依赖大模型，除非你后面有足够历史数据回测证明它比当前 EWMA/规则引擎明显更准。

---
