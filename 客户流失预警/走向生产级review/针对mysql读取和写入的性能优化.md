
---

### Turn 10  *(2026-04-22 07:15)*

**User:**

感觉没必要把 discovery 彻底干掉，反正只会对全量销售做流失预警检测

**Assistant:**

这个场景下，没必要强行干掉 `discovery`。

更准确的说法是：不必消灭 `discovery`，只要把“发现销售列表”和“判断当前客户归属”这两件事解耦就够了。对你现在这种“只跑全量销售流失预警”的模式，`discovery` 反而有合理性，因为它会一次性把整窗订单拉下来并预热缓存，后面的批次检测直接复用；如果改成完全依赖 `CRM_API_SALES_IDS` + 按销售逐个拉订单，未必更快，很多时候还会变成更多 HTTP 请求。

所以现在比较合适的策略是：
- `discovery` 保留，继续负责全量销售发现和全窗订单预热
- `CRM_CUSTOMER_API_URL` 保留，当前客户归属直接走 `getSalesmanCustomer`
- `CRM_API_SALES_IDS` 不作为常态配置，只保留给灰度、排障、临时重跑

这也是我刚才已经改回去的原因：`.env` 里不再写死 `CRM_API_SALES_IDS`，但 `CRM_CUSTOMER_API_URL` 已经配好了。接下来如果继续优化，重点就不该放在“删掉 discovery”，而该放在：
- 客户接口并发拉取
- 订单接口 discovery 的稳定性和分页吞吐
- MySQL 持久化层

如果你同意这个方向，我下一步就按“保留 discovery，只优化全量跑的吞吐”继续改。

---

### Turn 11  *(2026-04-22 07:17)*

**User:**

按“保留 discovery，只优化全量跑的吞吐”继续改

**Assistant:**

我先把 discovery 路径和当前全量跑的瓶颈再对一遍，重点看三处：全窗订单 discovery、客户名册拉取、批次里的订单/客户缓存命中率。然后我直接改吞吐，不碰“是否保留 discovery”的业务策略。

我把可动的点收敛成两项：一是把 discovery 的分页拉数改成并行，二是让 discovery 完成后把“发现到的销售列表”真正落进缓存，并在有客户接口时一次性预热客户名册。这样不改业务流程，但全量跑的冷启动吞吐会明显好很多。

我开始改 `crm_api` 了。会把 discovery 页并发、发现销售缓存修正、客户名册预热这三块一起落下，并补两组回归测试卡住关键行为。

我把回归测试也补了：一个盯住 discovery 分页并发，一个盯住“discovery + 客户接口”时销售列表和客户名册都会预热。接下来我先跑 `crm_api` 单测，确保路径没拧坏。

我把并发参数也挂到工厂了，顺手补一个工厂层测试，保证环境变量能真正传进 `CrmApiRepository`，避免只改了 README 没进代码路径。

测试在跑。我先盯一下有没有 factory / crm_api 这两块的新回归问题，尤其是 discovery + 客户接口这条组合路径。

这轮我按“保留 discovery，只优化全量跑吞吐”做了两类改动，核心都在 [app/shared/crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:115)。

第一块是 discovery 冷启动加速。订单接口分页现在支持并行拉取，仍然保留原来的顺序校验和“分页未推进”保护，[app/shared/crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:684)。同时修正了一个组合路径问题：`discovery + CRM_CUSTOMER_API_URL` 时，发现到的 `sales_id` 现在会真正写入缓存，不会再因为客户接口开启而丢失销售列表，[app/shared/crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:953)。在此基础上，如果配置了客户接口，discovery 完成后会一次性预热这些销售的客户名册，避免后续每个 batch 再分段补拉，[app/shared/crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:959)。

第二块是把并发度做成可调参数。工厂现在支持从环境变量读取 `CRM_API_MAX_PARALLEL_SALES_FETCHES` 和 `CRM_API_MAX_PARALLEL_PAGE_FETCHES`，[app/churn/factory.py](/mnt/e/churn-warning-engine/app/churn/factory.py:58)。示例和文档也补了，[.env.example](/mnt/e/churn-warning-engine/.env.example:15) [README.md](/mnt/e/churn-warning-engine/README.md:166)。

我补了回归测试，覆盖了 discovery 并行分页、`discovery + customer_api` 预热、以及工厂读取新并发参数，[tests/shared/test_crm_api.py](/mnt/e/churn-warning-engine/tests/shared/test_crm_api.py:204) [tests/shared/test_crm_api.py](/mnt/e/churn-warning-engine/tests/shared/test_crm_api.py:505) [tests/churn/test_api_and_factory.py](/mnt/e/churn-warning-engine/tests/churn/test_api_and_factory.py:415)。已跑通 `pytest -q tests/shared tests/churn`，结果是 `221 passed`。

还没重新跑真实 CRM + MySQL 的全量烟测，所以这轮我能确认的是代码路径和回归都正确，新的真实耗时数字还没有。如果你要，我下一步直接用当前配置跑一轮真实全量，对比优化前后的总耗时和各阶段耗时。

---

### Turn 12  *(2026-04-22 07:23)*

**User:**

"applied_pause_windows": ["春节制造业停工", "春节"]，这里文案只用["春节制造业停工"]就好

**Assistant:**

我先定位 `applied_pause_windows` 的生成逻辑，确认是业务日历去重策略还是预警组装层的问题，然后直接改成只保留更具体的停工窗口名称。

我看到生成点了，`adjustment_between()` 现在是按天把所有命中的窗口名都塞进 `applied_windows`，所以春节法定假 + 制造业停工会同时出现。我再看下业务日历测试，然后直接把“更具体窗口覆盖更通用窗口”的规则补进去。

我准备在业务日历层直接做“同一天如果已有行业/自定义停工窗口，就不再把法定节假日窗口记进 `applied_windows`”。这样输出会自然变成只保留 `春节制造业停工`，而不是在展示层硬编码特例。

我会连同预警层一起补一个断言，保证 `feature_summary["applied_pause_windows"]` 也只剩一个 `春节制造业停工`，这样以后不会在别的地方又把法定节假日名字拼回去。

我直接在业务日历层收口，不在展示层打补丁。逻辑会很小：某天只要命中了行业/自定义停工，就不再把同天的法定节假日窗口加入 `applied_windows`。

改完了。我先跑业务日历和预警引擎相关测试，重点验证 `applied_pause_windows` 现在只剩 `春节制造业停工`，且不影响天数计算。

这块局部回归已经过了。我再跑一遍全量 `shared + churn`，确认业务日历改动没有把别的链路带坏。

已经改了。现在同一天如果已经命中了行业停工或自定义停工窗口，就不再把同天的法定节假日窗口记进 `applied_pause_windows`，所以春节这类重叠场景会只保留更具体的 `春节制造业停工`，[app/shared/business_calendar.py](/mnt/e/churn-warning-engine/app/shared/business_calendar.py:132)。

我同时补了回归断言，业务日历层和预警 `feature_summary` 层现在都要求输出只有这一项，[tests/shared/test_business_calendar.py](/mnt/e/churn-warning-engine/tests/shared/test_business_calendar.py:24) [tests/churn/test_engine.py](/mnt/e/churn-warning-engine/tests/churn/test_engine.py:270) [tests/churn/test_engine.py](/mnt/e/churn-warning-engine/tests/churn/test_engine.py:826)。已跑通 `pytest -q tests/shared tests/churn`，结果 `221 passed`。

---

### Turn 13  *(2026-04-22 07:32)*

**User:**

用当前配置跑一轮真实全量，对比优化前后的总耗时和各阶段耗时

**Assistant:**

我先不动代码，直接按当前 `.env` 跑真实 CRM + MySQL 全量。为了让结果能对上这次优化点，我会把每个类型拆成 `start_job`、`list_batches`、`detect_batches` 三段计时，再和上次优化前的总耗时做对比。

我先把“优化前”的可对照数据补齐一下：从库里取老 `detection_run` 的实际运行时长。这样待会儿新跑完后，至少 `detect_batches` 这一阶段能做前后直接对比，不只是对总耗时。

真实全量开始跑了。我会按批次吐进度，先盯 `frequency`，跑完一类就给你一个阶段汇总。

`frequency` 已经起跑，当前在 `start_job` 冷启动阶段。这个阶段会走 discovery 全窗订单加载，也是这次吞吐优化最关键的部分。

还在 `frequency.start_job`。暂时没有批次输出，说明这会儿还在做 discovery 拉数/客户名册预热，还没进入批次检测循环。

冷启动阶段比单批检测更重一些，符合预期。等第一条 `batch_done` 出来后，就能知道这次 `start_job` 的真实耗时。

已经超过 1 分钟，还没进批次循环，说明 `frequency` 这次的 discovery 路径仍然是分钟级，但要看它比上次压了多少。再等第一批出来就能判断。

`frequency.start_job` 还没结束，当前至少说明两点：一是 discovery 全量预热仍然是最大的单段耗时；二是如果这段压不下来，整体收益就会被吃掉。我继续盯第一批出数。

现在还停留在第一类的冷启动，没有报错也没有批次输出，像是持续在拉 CRM。只要不报异常，我先继续等；这段如果明显超过上次频率任务的总耗时，就说明 discovery 仍然是主瓶颈。

第一批已经出来了，但有新情况：`frequency` 的首批在 MySQL 查询阶段超时断连了，报的是 `Lost connection to MySQL server during query (timed out)`。我先继续让它把整轮跑完，看看这是个首批偶发还是系统性问题。

---

### Turn 14  *(2026-04-22 07:42)*

**User:**

继续啊

**Assistant:**

我继续接着盯刚才那轮，不重跑。先检查那个长任务还在不在，如果还活着就直接把后续批次结果接回来。

我先查一下刚才那轮在库里的状态：看 `frequency` 这次的 `job/run` 是不是还留着，以及已经跑到哪一批。这样可以决定是续跑还是整轮重来。

刚才那轮不能直接拿来对比了，原因有两个：一是你中途打断了，本次总耗时会被人为拉长；二是 `frequency` 首批碰到了一次 MySQL 查询超时。  
我现在做两步清理后重跑一轮可恢复的基准：先把那条残留 `run` 标成失败，再启动一个支持断点续跑、批次失败自动重试 1 次的计时脚本。

新基准开始重跑了，这次会把每个类型的结果实时写到 `/tmp/churn_real_full_compare_resume.json`，就算中途再断也能续。先还是从 `frequency` 开始。

新一轮 `frequency` 已经开始。这次脚本带断点续写和单次重试，所以就算某一批再抖一次，也不会整轮数据全丢。

还在 `frequency.start_job`，暂时没有进入批次循环。这个阶段如果能比上一轮更早吐出第一批，就说明 discovery/名册预热确实压下来了。

`frequency` 还没出第一批，说明冷启动依然是分钟级。等首批出来以后，我就能把 `start_job` 和单批耗时一起给你。

目前还是在第一类的预热段，没有报错。只要这段结束后批次能够稳定跑完，这轮对比就还有价值。

已经过了两分钟，`start_job` 还是当前最大的单段。后面如果 `detect_batches` 能大幅缩短，总时长仍然可能优于旧版；否则 discovery 还得继续压。

还在冷启动，暂时没见新的异常。这个阶段结束越晚，说明“保留 discovery”的成本仍然很高，我后面会把它单独拆出来给你看。

第一批又复现了同样的 MySQL 查询超时，不过这次脚本已经自动进入重试。若重试也失败，就基本可以判定：当前配置下不是偶发，而是系统性超时。

重试这次也拖得比较久。如果它在 60 秒左右再断一次，就说明问题点基本锁定在 `detect_batch` 里的某个 MySQL 长查询，而不是网络偶发抖动。

重试也失败了，首批两次都在约 60 秒超时，已经不是偶发了。我先继续看下第二批，确认是“个别批次”还是“整轮系统性失败”。

第二批也开始同样的超时重试了，基本可以判断不是单个批次的问题。我去看下 `detect_batch` 里最可能卡 60 秒的点，优先怀疑 MySQL 锁等待或某个大 `IN` 查询。

我已经把可疑点缩到 `detect_batch` 的 MySQL 段了。锁等待只有 10 秒，不像元凶；更像是某个数据库查询本身稳定跑过 60 秒，撞上了连接 `read_timeout`。  
我再等第二批的重试结果，如果同样失败，我就先停压测，改做首批分段定位。

问题已经定位到了，不是网络偶发，也不是锁。当前版本在 [app/churn/mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:1125) 的 `list_active_warnings_for_customers(...)` 上触发了稳定超时。  
也就是说，我刚加的“批量预取活动预警”这条查询在远端 MySQL 上退化了，导致全量真实跑根本过不去。先把这个回归修掉，再跑对比才有意义。

我准备直接修这个回归，再重跑基准。改法会偏保守：`list_active_warnings_for_customers()` 去掉没必要的排序，缩小批量 `IN` 块大小，并且只查真正需要的轻量字段，避免远端 MySQL 在首批就卡死。

我先做个最小修复，不动业务逻辑：去掉这条批量预取的 `ORDER BY`，并把大 `IN (...)` 切小。修完先跑仓储测试，再回到真实数据首批验证。

修补已经落了。先跑仓储回归，确认我没有把活动预警引用查询的语义改坏；过了就直接拿真实 `frequency` 首批复测。

我先不直接整轮重跑，先拿真实 `frequency` 首批做回归验证。只要首批能过，说明这个 MySQL 回归点修掉了，再跑全量才有意义。

真实首批回归已经开始了，还是同样的 `frequency + batch_size=100`。这次只盯 `start_job` 和第一批 `detect_batch`，能过的话我就直接全量重跑。

首批回归还在 `start_job` 冷启动阶段，暂时没有异常。关键还是看后面第一批会不会再稳定卡在 60 秒超时。

首批回归已经过了冷启动，`start_job = 126.83s`，比刚才那两次明显短。现在进入真正的第一批 `detect_batch`，这是验证修复是否生效的关键。

我继续把这轮真实全量 benchmark 跑完。先处理刚才暴露出来的真实瓶颈：`detect_batch` 前置的活动预警预取查询在远端 MySQL 超时，我先把这条路径改成更便宜的存在性查询，再回头重跑真实数据对比。

现在已经定位到真实问题了，不是 CRM，而是 MySQL 这条“批量预取活动预警”的 SQL 仍然太重。我先把 `detect_batch` 改成只查“哪些客户存在活动预警”，不再在批次前置阶段把 warning 记录整行拉出来。

我要先把接口层补齐，不然 service 改了以后几套仓储和测试桩都会断。接下来会同时改 `repository`、`memory_repo`、`mysql_repo` 和对应测试，然后先跑单测，再回到真实链路。

我要动代码了。这一版只改一件事：把批次前置的“活动预警预取”从明细拉取改成存在性查询，尽量不碰检测算法和落库逻辑，先把 benchmark 的硬阻塞打掉。

先跑最小回归，确认这次接口改动和 SQL 断言没把仓储层打坏。过了以后我会先跑一个真实 `frequency` 首批次 smoke，看这个查询是不是已经不超时。

单测失败是我用例写错了，不是实现问题。当前库本来就禁止同一客户同类型同时存在两条活动预警，所以这个场景不合法；我把用例改成合法数据后继续。

单测顺手暴露了一个老问题：`save_warnings()` 批量插入时，`active_slot=None` 会被 DB 的默认值吃掉，`resolved` 记录也会被写成活动态。这个我一并修掉，不然后面的存在性查询语义不稳。

单测已经过了。现在先跑一个真实 `frequency` 首批次 smoke，确认这条 MySQL 查询链路已经不再在第一个 batch 上超时；过了我就直接启动完整对比跑数。

真实 smoke 已经启动了。`start_job` 本身会先做 discovery 和批次准备，所以前几分钟可能没有输出；我先盯着这条链路，确认首批次能不能完整走通。

当前还停在 `start_job` 阶段，说明 discovery 仍然是冷启动的大头，但这次目标先看它能不能顺利进入 `detect_batch`。我继续等到首批次开始打印为止。

首批次已经走通了：`start_job` 约 `121.5s`，`detect_batch` 约 `78.2s`，之前卡死的那条 MySQL 预取查询已经不是硬阻塞。下一步我直接跑三类真实全量，并把优化前的总耗时拉出来做对比。

我用“每个类型独立初始化一套 service”的方式跑，保证和上次每类任务的总耗时更可比。脚本会记录 `start_job`、`list_batches`、`detect_batches` 三段时间，并逐批打印进度。

全量 benchmark 已经起跑，先从 `frequency` 开始。等 `start_job` 结束后就会开始逐批打印，这样我能边跑边确认吞吐有没有稳定下来。

`frequency.start_job` 还在跑，冷启动依旧接近 2 分钟，这和刚才的首批次 smoke 基本一致。当前重点是看后面的每个 batch 能否稳定落在 1 分钟多而不是再出现 MySQL 查询超时。

第一批已经稳定落下来了，`72.6s`，比刚才单批 smoke 还快一点。现在继续盯后续批次，确认不是偶发值。

`frequency` 后半段明显更快了，说明当前瓶颈主要还是“每批客户规模”，不是又卡回固定 SQL 超时。接下来我继续把这一类跑完，再看 `product` 和 `volume` 的整体曲线。

`frequency` 已经跑完了，当前总耗时 `482.1s`，比优化前 `1646s` 已经下来了很多。现在切到 `product`，这一类能反映 MySQL 持久化层优化是不是普适，而不是只对 `frequency` 生效。

`product` 这轮没跑完，但不是又回到先前那种批次前置全局阻塞，而是在首批次里一个 `warning_id IN (2 ids)` 的小查询上掉了连接，更像远端 MySQL 的瞬时连接问题。`volume` 我先继续跑完，随后会单独补跑 `product` 并加重试，避免把偶发网络抖动误判成性能回退。

`volume` 目前是稳定的，前两批在 `100s` 左右，比 `frequency` 重一些，但还没有出现超时。看起来剩下需要单独补救的是 `product` 那个偶发掉线点。

我先不把 `product` 的失败当成性能回退。它更像连接层抖动，而且这轮脚本没加重试。先看下仓储里现有的 MySQL 断线重试点，再决定是只补跑，还是顺手把 `get_warnings()` 也补上同级别的重试。

这一步我会补一个很小的韧性改动：`get_warnings()` 遇到 MySQL `2013/2006` 这类断线时自动重试一次并 `dispose` 连接池。这样不改业务语义，但能把这种长任务里的偶发掉线吸收掉。

我把 `get_warnings()` 的 MySQL 断线重试也补上了，单测已过。现在单独补跑 `product`，并带一层批次重试，把这一类的数据补齐到对比报告里。

补跑的 `product` 已经进入批处理阶段了，还没报错，说明刚才那次掉线更像连接问题而不是稳定复现的 SQL 性能问题。我继续等到首批次和中段批次出来，看它的真实吞吐。

`product` 现在是稳定复现同一个点了，不再像偶发断线。问题已经缩到很小：加载 2 个活动预警明细时，`get_warnings()` 这条带 `char_length(...)` 的查询在远端 MySQL 上超时。我要先看产品预警去重到底需要哪些字段，争取把这里从“全量 warning 明细”再压成“只拿匹配必需字段”。

我在脱离业务流程的情况下直接复现这两个 `warning_id` 的 `get_warnings()`。如果这里也挂住，就说明问题根本不在 `detect_batch`，而在这条明细读取实现本身。

我在做列级定位。`EXPLAIN` 看起来正常，都是主键命中两行，所以问题不像是执行计划，而更像“取哪些列会触发非常慢的行读取”。我先把慢列缩出来，再决定是改 `get_warnings()` 还是改调用路径。

已经定位了：不是哪一列单独慢，而是“标量列 + `CHAR_LENGTH(text)`”混在同一条 `SELECT` 里会稳定卡死远端 MySQL。我要把 `get_warnings()` 拆成两条查询再合并，这个改动很小，但应该正好打掉 `product` 的复现点。

我先不急着重跑整批 `product`，先做一个最小验证：直接读那两个会卡住的 `warning_id`。这一步如果几秒内过了，就说明根因已经被这次拆查询修掉了。

最小复现已经过了，那两个 `warning_id` 现在 `0.1s` 就能读出来。我重新补跑 `product` 全量，把报告补完整。

`product` 已经恢复正常了，首批次 `73.5s` 成功落库。也就是说，这次拆查询确实打掉了刚才那个稳定复现点。

我再做一次更细的验证：同样的 3 个 `warning_id`，如果一条条按主键读能不能秒回。要是单条主键稳定，那最稳妥的方案就是把 `get_warnings()` 改成按 `warning_id` 逐条拉，而不是 `IN (...)` 批量拉。

这里的模式已经很明确了：同样的数据，`WHERE warning_id = :id` 单条主键读取是毫秒级，`WHERE warning_id IN (...)` 批量读取会稳定超时。我要把 MySQL 的 `get_warnings()` 改成逐个 `warning_id` 读取，先换稳定性。

现在这 3 个会卡死的 `warning_id` 也能在 `0.26s` 内读出来了。我再补跑最后一次 `product` 全量，目标是把完整对比报告补齐，不再改逻辑。

`product` 现在前两批已经连续过了，耗时和 `frequency` 同一个量级，说明这次修正不只是“能读出来”，而是把整批持久化链路也稳定住了。

现在三类真实全量都跑齐了。我再补一轮回归测试，把这次针对 MySQL 读取路径的改动和前面的 CRM/日历改动一起回扫一遍，然后给你最终对比结论。

真实全量 benchmark 已经跑完了。当前配置下，三类都用真实 CRM + MySQL 跑通，`batch_size=100`，`failed_batches=0`。完整明细在 [churn_real_full_compare_after.json](/tmp/churn_real_full_compare_after.json)。

优化前只保留了总耗时，没有历史阶段埋点；所以“前后对比”里，`before` 是上次真实跑数的总耗时，`after` 同时给出本轮 `start_job / list_batches / detect_batches / total`。

**结果**
- `frequency`：`1646s -> 482.12s`，减少 `1163.88s`，降幅 `70.71%`
- `frequency` 当前阶段：`start_job 118.79s`，`list_batches 0.15s`，`detect_batches 363.18s`，`total 482.12s`
- `product`：`1190s -> 458.86s`，减少 `731.14s`，降幅 `61.44%`
- `product` 当前阶段：`start_job 128.22s`，`list_batches 0.11s`，`detect_batches 330.53s`，`total 458.86s`
- `volume`：`1511s -> 588.87s`，减少 `922.13s`，降幅 `61.03%`
- `volume` 当前阶段：`start_job 119.35s`，`list_batches 0.09s`，`detect_batches 469.43s`，`total 588.87s`
- 三类合计：`4347s -> 1529.85s`，总计减少 `2817.15s`，降幅 `64.81%`

**这轮新增修正**
- `detect_batch` 先查“哪些客户存在活动预警”，避免前置阶段批量拉活动预警明细，[service.py](/mnt/e/churn-warning-engine/app/churn/service.py:668)
- MySQL `get_warnings()` 拆成标量列查询和文本长度查询，并在 MySQL 下按 `warning_id` 逐条主键读取，绕开远端库对 `IN (...)` 批量明细读取的异常慢路径，[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:561) [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py:968)
- 补了对应回归测试，[test_mysql_repo.py](/mnt/e/churn-warning-engine/tests/churn/test_mysql_repo.py:188) [test_mysql_repo.py](/mnt/e/churn-warning-engine/tests/churn/test_mysql_repo.py:1370)

`pytest -q tests/shared tests/churn` 已跑过，`224 passed`。

现在最大的剩余瓶颈已经很清楚了：`list_batches` 基本可以忽略，固定成本主要是 `start_job` 的 discovery 冷启动，运行时成本主要集中在前 2-3 个大批次的 `detect_batches`。如果你要继续压，我下一步就直接盯这两段。

---

### Turn 15  *(2026-04-22 09:02)*

**User:**

就目前而言，性能已经很强了吧，没必要再优化了吧

---

### Turn 16  *(2026-04-22 09:08)*

**User:**

就目前而言，性能已经很强了吧，没必要再优化了吧

**Assistant:**

对当前这套业务形态来说，已经够用了，不建议继续做性能优化。

依据很直接：真实 CRM + MySQL 下，三类全量都能稳定跑通，单类总耗时已经降到 `7.6` 到 `9.8` 分钟，三类串行合计约 `25.5` 分钟；相比优化前总耗时 `4347s`，现在是 `1529.85s`，整体下降了 `64.81%`。这已经从“明显偏慢”进入“工程上可接受”的区间了。

剩下的热点也很清楚，但不算必须处理：
- `start_job` 的 discovery 冷启动仍然要 `120s+`
- 每类前 2 到 3 个大 batch 还是最慢，`70s` 到 `110s` 左右

如果你的运行模式是“定时全量跑”或“人工偶发触发”，这两点都能接受。只有在下面几种情况下，才值得继续压性能：
- 你要求单类全量必须压到 `5` 分钟以内
- 需要多任务并发跑，而不是串行跑
- 白天频繁人工重跑，希望接近实时返回
- 远端 MySQL 连接质量不稳定，重新出现超时

更合适的下一步不是继续提速，而是做稳定性和观测：
- 记录 `start_job`、前 3 个 batch、总耗时
- 监控 MySQL `2013/2006` 断连次数
- 保留一套真实数据冒烟基线，后续改动拿它回归

结论就是：目前性能已经达标，继续优化的收益不高，优先级可以降下来了。

---
