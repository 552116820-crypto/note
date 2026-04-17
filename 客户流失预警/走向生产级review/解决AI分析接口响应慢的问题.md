# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019d9a60-42db-7cf1-a4ea-b5ac0c270019`
- **范围**: Turn 12-14
- **导出时间**: 2026-04-17 17:10:15

---

### Turn 1  *(2026-04-17 08:44)*

**User:**

http://127.0.0.1:8000/api/warnings/ai-analysis这个接口响应时间很长，为什么啊？会不会很影响

**Assistant:**

我先按代码路径拆一下这个接口的耗时来源，重点看它是在等 CRM、跑本地检测，还是等 DeepSeek。

是的，`/api/warnings/ai-analysis` 响应时间长是正常现象，但现在这个接口如果直接给前端同步调用，**确实会影响用户体验和并发能力**。

它慢主要有三段原因。

**1. 它不是单纯查表接口**

`POST /api/warnings/ai-analysis` 不是直接从 `churn_ai_analyses` 取结果。它会先重新构造本次分析输入上下文，代码在 [`service.py`](/mnt/e/churn-warning-engine/app/churn/service.py:473)：

1. 校验 AI 分析器是否存在。
2. 准备 CRM 数据窗口。
3. 找客户。
4. 拉客户订单。
5. 拉最近三次拜访记录。
6. 实时跑该客户的预警检测。
7. 组装 `visit_records`、`warning_summaries`、`order_summary`。
8. 计算 `input_hash`。
9. 查缓存。
10. 缓存没命中才调用 DeepSeek。
11. 保存到 `churn_ai_analyses`。

所以即使最后命中缓存，当前代码也要先拿 CRM 数据并计算 `input_hash`，不能一进来就直接查缓存。

**2. 当前 `REPOSITORY_TYPE=api` 时，CRM 数据加载可能很重**

CRM API 仓储在 [`crm_api.py`](/mnt/e/churn-warning-engine/app/shared/crm_api.py:597) 有 `_ensure_loaded()`。

如果没有配置 `CRM_API_SALES_IDS`，它会进入 discovery mode，从 CRM API 拉一个时间窗口内的数据来发现销售和客户。这个窗口默认跟 `ANALYSIS_WINDOW_DAYS=365` 有关。

也就是说，本地第一次请求可能不是“查一个客户”，而是触发一次比较大的 CRM 数据加载。后续同一进程内有缓存，会快一些，但冷启动第一次会慢。

**3. DeepSeek 调用本身是同步等待**

DeepSeek 调用在 [`ai_analysis.py`](/mnt/e/churn-warning-engine/app/churn/ai_analysis.py:41)，当前是同步 HTTP 请求：

```python
response = client.post(...)
```

默认配置：

- `DEEPSEEK_MODEL=deepseek-reasoner`
- `DEEPSEEK_TIMEOUT_SECONDS=60`
- `DEEPSEEK_MAX_TOKENS=4096`

`deepseek-reasoner` 本身就可能比普通对话模型慢，尤其输入上下文里有订单摘要、拜访记录、预警摘要时。第一次生成等十几秒甚至更久都不奇怪。

**会不会很影响？**

会，分场景：

- 单人本地测试：影响不大，就是等得久。
- 前端页面同步点击后等待：用户会明显感觉慢，需要 loading。
- 多人同时点 AI 分析：会占用 FastAPI 的工作线程、MySQL 连接、CRM API 请求、DeepSeek 配额，可能拖慢服务。
- 如果大量 `force_refresh=true`：每次都打 DeepSeek，影响最大。
- 如果都是缓存命中：会好很多，但当前仍要先构造上下文，所以不是纯查缓存那么快。

`include_context=false` 只是不返回上下文字段，**不会减少构造上下文和 DeepSeek 调用耗时**。它主要减少响应体大小和数据暴露面。

**现在比较合理的使用方式**

生产上不要把这个接口当成普通实时列表接口频繁调用。更推荐：

1. 检测任务生成预警后，由 Celery 后台自动生成 AI 分析。
   代码里已经有 [`generate_ai_analysis_for_customer`](/mnt/e/churn-warning-engine/app/churn/tasks.py:106) 和检测后派发逻辑。

2. 前端打开客户详情时，优先读已有缓存结果。
   如果缓存不存在，再提示“正在生成 AI 分析”。

3. 前端不要默认传 `force_refresh=true`。
   只有用户点“重新分析”时才传。

4. 本地测试时如果只想验证 DeepSeek，不要走真实 CRM API，可以用 memory 模式。

5. 如果要优化实时体验，下一步应该把接口改成异步模式：
   - `POST /api/warnings/ai-analysis` 只创建/派发任务，快速返回 `202 + analysis_id/job_id`
   - 前端轮询 `GET /api/warnings/ai-analysis/{analysis_id}` 或按客户查最新分析
   - Celery 后台生成后写入 `churn_ai_analyses`

这样前端不会卡在 DeepSeek 和 CRM API 上。

**短期可做的优化**

- 配置 `CRM_API_SALES_IDS`，避免 discovery mode 拉太大范围。
- 本地测试缩短 `ANALYSIS_WINDOW_DAYS`。
- 不要频繁 `force_refresh=true`。
- 给 AI 分析链路加分段耗时日志，明确是 CRM 慢、检测慢、缓存慢，还是 DeepSeek 慢。
- 把 AI 分析从同步接口改成后台任务 + 查询结果接口。

我建议下一步做两件事：先加耗时日志定位瓶颈，然后把 AI 分析改成异步任务模式。这样本地和服务器都会稳很多。

---

### Turn 2  *(2026-04-17 08:48)*

**User:**

为什么不是从数据库中拿AI分析的结果呢？

**Assistant:**

因为当前接口的语义是“**为这个客户在当前输入上下文下生成或读取 AI 分析**”，不是“**按客户直接读取历史 AI 分析结果**”。

现在的缓存判断依赖 `input_hash`，而 `input_hash` 是根据这次输入上下文算出来的。上下文包括：

- 客户信息
- 最近三次拜访记录
- 实时预警摘要
- 近期订单摘要
- `biz_date`
- guardrails

所以代码必须先构造上下文，才能知道这次输入的 `input_hash` 是什么，然后才能去 `churn_ai_analyses` 查有没有完全一致的缓存。对应逻辑在 [`service.py`](/mnt/e/churn-warning-engine/app/churn/service.py:503) 到 [`service.py`](/mnt/e/churn-warning-engine/app/churn/service.py:533)。

这样设计的优点是：**不会拿旧数据的 AI 结论冒充新数据的结论**。比如客户昨天新增了一次拜访记录、订单变化了、业务日期变了，`input_hash` 会变，系统就不会误用旧分析。

但缺点也很明显：**即使命中缓存，也要先拉 CRM/订单/拜访记录、跑预警、算 hash**，所以响应时间仍然可能长。

你说的“为什么不是从数据库中拿 AI 分析的结果”，这个需求是合理的，但它应该是另一个读取接口，例如：

```text
GET /api/warnings/ai-analysis/latest?sales_id=000455&customer_id=115849
```

这个接口可以直接查 `churn_ai_analyses` 最新一条记录，立即返回，不重新拉 CRM、不调用 DeepSeek、不算实时上下文。它适合前端页面默认展示“最近一次 AI 分析结果”。

然后现在的：

```text
POST /api/warnings/ai-analysis
```

保留为“生成/刷新分析”的接口。

我建议改成这个模式：

- `GET /api/warnings/ai-analysis/latest`：快速读库，给页面默认展示
- `POST /api/warnings/ai-analysis`：需要生成或刷新时才调用，可能慢
- `force_refresh=true`：用户明确点“重新分析”时才用
- Celery 检测任务后自动预生成 AI 分析，前端大多数时候只读库

这样逻辑更清晰，性能也会好很多。

---

### Turn 3  *(2026-04-17 08:50)*

**User:**

改成你说的这个模式

**Assistant:**

我会按“读库展示”和“生成/刷新”拆开：新增一个快速 `GET /api/warnings/ai-analysis/latest`，只查 `churn_ai_analyses` 最新记录，不拉 CRM、不调用 DeepSeek；现有 `POST /api/warnings/ai-analysis` 保留生成/刷新语义。先补 repository 能力，再接 service、route 和测试。

我会先实现最小稳定版本：`latest` 只读 `churn_ai_analyses` 最新一条，不重新算 `input_hash`，所以不会访问 CRM/DeepSeek。默认仍不返回上下文，`include_context=true` 时复用已有响应组装逻辑。

仓储层已新增按客户查最新分析。接下来接 service 和 route，让前端能通过 GET 快速取库中已有结果。

路由已经加好。现在补测试：验证 latest 读的是缓存表最新记录，不触发 Fake analyzer；也验证 `include_context=true` 才带上下文。

测试覆盖已补。接下来更新 README 和 API guide，明确 GET 读库、POST 生成/刷新。

代码和文档都已更新。现在跑聚焦测试，先确认新 GET 接口和 MySQL latest 查询行为。

AI 相关测试通过。接下来跑完整测试套件，确认新增抽象方法没有漏实现，路由也没有影响现有接口。

完整测试通过。最后我做一次局部差异和行号检查，方便你后面接前端调用。

已改成这个模式：**GET 读库展示，POST 生成/刷新**。

新增接口：

```http
GET /api/warnings/ai-analysis/latest?sales_id=000455&customer_id=115849
```

这个接口只查 `churn_ai_analyses` 里该客户最新一条记录，不拉 CRM、不调用 DeepSeek、不重新跑预警上下文。没有缓存时返回 `404`。

需要上下文时：

```http
GET /api/warnings/ai-analysis/latest?sales_id=000455&customer_id=115849&include_context=true
```

现有接口保留：

```http
POST /api/warnings/ai-analysis
```

它仍然负责生成/刷新分析，会构造上下文、算 `input_hash`、必要时调用 DeepSeek。

改动位置：

- 新增 repository 方法：[`repository.py`](/mnt/e/churn-warning-engine/app/churn/repository.py:87)
- 内存实现：[`memory_repo.py`](/mnt/e/churn-warning-engine/app/churn/memory_repo.py:167)
- MySQL 实现：[`mysql_repo.py`](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:784)
- Service 快速读取方法：[`service.py`](/mnt/e/churn-warning-engine/app/churn/service.py:562)
- 新 GET 路由：[`routes.py`](/mnt/e/churn-warning-engine/app/churn/routes.py:244)
- 测试覆盖：[`test_api_and_factory.py`](/mnt/e/churn-warning-engine/tests/churn/test_api_and_factory.py:590)、[`test_mysql_repo.py`](/mnt/e/churn-warning-engine/tests/churn/test_mysql_repo.py:239)
- 文档已同步：[`README.md`](/mnt/e/churn-warning-engine/README.md)、[`docs/api-guide.md`](/mnt/e/churn-warning-engine/docs/api-guide.md)

前端推荐调用顺序：

1. 页面打开先调 `GET /api/warnings/ai-analysis/latest`
2. 如果返回 `200`，直接展示历史 AI 分析
3. 如果返回 `404`，显示“暂无分析”或触发生成
4. 用户点“生成/重新分析”时再调 `POST /api/warnings/ai-analysis`

验证已跑：

```bash
pytest -q
```

结果：`140 passed`，只有 4 个 sqlite date adapter 的既有 warning。

---
