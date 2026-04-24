# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019daf87-01fb-7ba3-9202-96066f75b8a1`
- **范围**: Turn 1-2
- **导出时间**: 2026-04-22 08:30:02

---

### Turn 1  *(2026-04-21 10:13)*

**User:**

评价一下这个项目代码的性能，避免高复杂度，导致真实大量数据运行缓慢，还要保证后期维护不太困难

**Assistant:**

我先按性能评审的方式读一遍关键路径，重点看 `churn` 模块的数据流、复杂度热点、仓储层是否会在真实数据量下退化，然后再给出针对“性能 + 可维护性”的具体结论。

我已经定位到主链路：`service.detect_batch()` 驱动批量检测，核心复杂度在 `engine.py` 的频次/体量/产品异常计算，以及 `mysql_repo.py` 的查询和反序列化。下一步我直接看这些热点的实现细节和查询形态。

服务层看下来，批处理入口本身还算克制，真正决定大数据表现的是两类地方：一类是仓储层是否能批量查全、避免 N+1；另一类是 `engine.py` 里每个客户订单集合会不会被重复全表扫描。现在我继续把这两部分摊开看。

环境里没有 `python` 命令，我改用 `python3` 做静态复杂度扫描，主要是把“代码大”“逻辑复杂”“查询可能退化”这三类风险分开，不然结论会太泛。

我已经看到两个很明确的热点了：`volume` 检测里有按客户订单数平方增长的匹配逻辑，`API/DB` 侧则有明显的“逐条/逐批次多次往返”模式。再补一层配置和批量参数后，我就能给出更完整的结论和分级建议。

我差不多可以给结论了。最后我把几个关键热点的行号收一下，这样结论能直接落到代码位置，不会变成泛泛而谈。

**Findings**
1. 高风险：`volume` 检测里的全额退款匹配是平方级复杂度。[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py:268) 对每个退款单都遍历全部正向订单构造 `candidates`，而 [detect_volume_anomaly]( /mnt/e/churn-warning-engine/app/churn/engine.py:690) 每个客户都会调用一次。客户订单一多，月末批跑会明显放大。建议按 `signature` 预分组，再按日期维护可匹配队列，把复杂度降到接近 `O(n log n)` 或 `O(n)`。

2. 高风险：批处理落库路径是逐条锁、逐条查、逐条写，并且产品类去重在 miss 时会退化成“扫该客户全部活跃产品预警 + 解析 JSON”。[service.py](/mnt/e/churn-warning-engine/app/churn/service.py:776) 和 [service.py](/mnt/e/churn-warning-engine/app/churn/service.py:620) 都是逐 warning 处理；[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:1424) 在精确 `dedup_key` miss 后会拉整批 `product_rows` 再逐条比对。真实 warning 数上来后，这会比检测算法本身更慢。建议先批量预取 active warnings，内存里完成 dedup/resolve 计划，再 bulk insert/update。

3. 中风险：CRM API 模式按 `sales_id` 串行拉数，吞吐线性依赖销售人数。[crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:635) 和 [crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:655) 都是 `for sid in ...: _fetch_and_merge(sid)`；[service.py](/mnt/e/churn-warning-engine/app/churn/service.py:689) 的 `list_batches()` 还会逐 batch 调 `customer_count_for_sales_ids`。如果接的是远程 CRM，首轮跑数和查批次都会慢。建议支持多 `sales_id` 合并请求或受控并发抓取，并把 batch customer count 做成一次性统计。

4. 中风险：列表接口已经做了轻量列选择，但分页仍是 `OFFSET` + 多次往返。[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:844) 一次列表至少 3 到 4 次查询；[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:1624) 的 pending 列表分块后仍然是 `OFFSET`。页码很深时 MySQL 还是会越来越慢。建议热路径改成 keyset pagination（`biz_date`, `warning_id`）。

5. 结构性风险：核心逻辑集中在几个超大文件里，后续优化成本偏高。[service.py](/mnt/e/churn-warning-engine/app/churn/service.py:701) 的 `detect_batch()`、[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py:690) 的 `detect_volume_anomaly()`、[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:1424) 的 `_find_active_warning_once()` 都已经很长。现在还能读，但再加规则后很容易把性能优化和业务改动绑死。建议按 anomaly 拆分 `frequency|volume|product` 策略模块，再把持久化去重流程单独抽成 writer/persister。

**Assessment**
这个项目不是“性能意识很弱”的代码库。正面的点有三处：批检测已经走批量订单加载而不是 N+1，[service.py](/mnt/e/churn-warning-engine/app/churn/service.py:725) 且有测试兜底 [test_service.py](/mnt/e/churn-warning-engine/tests/churn/test_service.py:199)；配置也支持 batch preload [service.py](/mnt/e/churn-warning-engine/app/churn/service.py:432), [test_service.py](/mnt/e/churn-warning-engine/tests/churn/test_service.py:800)；MySQL 列表查询和活跃预警查询已经刻意避开大 JSON 列，并补了索引 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:69), [test_mysql_repo.py](/mnt/e/churn-warning-engine/tests/churn/test_mysql_repo.py:185), [test_mysql_repo.py](/mnt/e/churn-warning-engine/tests/churn/test_mysql_repo.py:652)。

我的判断是：小到中等数据量，这个项目能跑；真实大量数据下，瓶颈会先出现在 `volume` 退款匹配、逐条去重落库、以及 CRM API 串行抓取这三处，而不是 FastAPI/Celery 框架本身。

**Next Steps**
1. 先把 `volume` 的退款匹配改成按签名分桶 + 日期游标，去掉平方级算法。
2. 把 `detect_batch()` 改成“先批量读取 active warnings，再生成 insert/update/resolve 计划，最后批量落库”。
3. 给 CRM API 仓储加 sales 批量抓取或受控并发，并去掉 `list_batches()` 的逐批次计数查询。
4. 把 warning 列表改成 keyset 分页。
5. 增加性能回归测试，至少覆盖“单客户 1000 单”“单批 1 万预警”“深分页”三组数据。

没做真实压测，这次结论基于静态代码审查。若你要，我下一步可以直接给你出一版“低复杂度重构方案”，或者先落地第 1 个热点。

---

### Turn 2  *(2026-04-22 00:25)*

**User:**

先把 volume 的退款匹配改成按签名分桶 + 日期游标，去掉平方级算法。

**Assistant:**

我先收拢 `volume` 相关测试和当前退款匹配语义，再把 `_match_fully_refunded_orders()` 改成按签名分桶 + 游标的线性/近线性实现，最后跑对应测试确认行为没变。

我要开始改 `engine.py` 了，改动只会落在退款匹配这段：把“每个退款单扫描全部正向单”的实现换成“按签名分桶 + 游标推进 + 可用栈回收”的等价逻辑，并补一条覆盖多笔同签名退款的测试。

代码已经改完，逻辑上等价但不再为每个退款单全表扫正向单。我先跑 `engine` 里的退款/volume 相关用例，再补一轮更窄的服务层回归，确认没有把现有 dedup 或汇总结果带偏。

退款匹配相关用例和 service 层 volume 回归都过了。为了避免这次重构只在局部成立，我再跑一轮完整的 `tests/churn/test_engine.py`，把频率、产品、去重这些相邻逻辑一起扫掉。

核心 `engine` 用例已经全过。为了把这次改动闭环到批处理层，我再补跑完整的 `tests/churn/test_service.py`，这样 volume 逻辑经过 service 编排、仓储去重后也能一起验证。

已把 `volume` 的退款匹配改成按签名分桶 + 日期游标 + 可用栈回收，不再对每个退款单扫描全部正向订单。核心改动在 [engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py:271)：先把正向订单按 `_OrderMatchSignature` 分桶，再随退款时间顺序推进游标，把“已到当前退款时间点之前”的候选订单压栈，匹配时直接 `pop()` 最新未占用订单。这样匹配阶段从原来的近似 `O(P * R)` 降到了按签名分桶后的近线性复杂度，原来的“匹配最近一笔更早的同签名原单”语义保持不变。

我还补了一条回归测试 [test_engine.py](/mnt/e/churn-warning-engine/tests/churn/test_engine.py:388)，覆盖“多笔同签名订单 + 多笔退款时，应优先匹配最新未匹配原单”的场景。

验证跑过：
```bash
pytest -q tests/churn/test_engine.py
pytest -q tests/churn/test_service.py
```
都通过了。未跑全仓测试。

---
