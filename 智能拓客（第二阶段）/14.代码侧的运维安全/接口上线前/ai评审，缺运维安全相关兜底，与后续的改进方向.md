# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `3a675520-5665-4904-9c43-6f97f5693a63`
- **范围**: Turn 1-4
- **导出时间**: 2026-06-15 15:33:00

---

### Turn 1  *(2026-06-15 00:10)*

**User:**

请你和codex从多个维度评价我这个项目

---

### Turn 2  *(2026-06-15 00:10)*

**User:**

请你和codex从多个维度评价我这个项目，还有哪些可以改进的地方

**Assistant:**

我来系统地评估这个项目。我会同时启动 Codex 做独立评审，自己也并行深入分析代码库，最后把两边的视角综合给你。

让我先并行启动 Codex 独立评审 + 自己的代码库勘探。

Codex 在后台独立评审中。我继续自己的独立勘探——读设计文档、工具链配置、CI、装配点，并跑客观检查。

已有清晰的事实底座。几个信号已经很明显:`.github` 为空(**无 CI**)、无依赖锁文件、mypy 非全严格、pre-commit 仅"可选"。现在深入读核心代码形成独立判断。

Codex 任务已在其运行时异步跑起来(`task-mqegknpk-7t64lh`),我继续独立分析,稍后取回它的报告对照。

几个初步信号已经浮现:`main.py:156-162` 的 CORS 默认 `*` + `allow_credentials=True`;`db_mysql` 是 per-request 连接无连接池;`celery_app.py` 无 `task_time_limit`/`acks_late`。继续验证横切关注点 + 读核心业务文件。

可观测性确认是空白(只有 `perf_counter` 打日志,无 metrics/tracing);`service.py` 的批次隔离做得好,但我注意到一个幂等性问题:任务重试时 `run_detection` 重新生成 run_id 跑全部批次,warning 行靠 dedup 不会重复,但 **CRM 推送会重复**。现在验证 lead 侧(用户输入面)的 SQL 拼接、SSRF/超时,以及密钥是否进了 git。

安全面整体不错:`.env` 未进 git、`.gitignore` 完善(含生产 dump)、SQL 全参数化、CRM/LLM 客户端超时可配。两个值得记的点:`.env.example` 里写了**真实生产 DB IP**(`47.114.77.122`/`8.138.104.14`),CRM 重试**无退避**。现在确认连接池策略、测试 mock 密度,并查 Codex 进度。

测试质量其实是**强项**:用内存 fake(`InMemoryCrmRepository`)而非 mock,churn 跑全链路检测、lead 用 TestClient 走真实 HTTP+中间件,conftest 刻意不 autouse。连接池确认**完全没有**(pool.py 连 `connect` 都不直接调,全走 shared 的 per-request)——这是头号可扩展性项。

我自己的独立评审已成型。现在去取回 Codex 的报告做对照。

SendMessage 不可用。让我看看 Codex CLI 的任务管理子命令,直接取回那个异步任务的结果。

Codex 的异步任务由 codex-rescue 的运行时托管,标准 CLI 取不到。我派一个 codex-rescue 去取回那个任务的完整报告(明确不许新建任务)。

Codex 评审仍在运行(11 个维度 + 跑 mypy/ruff,耗时正常)。companion 要求先过 `/codex:status` 才能 resume。让我调用它查进度。

我无法直接调用 `/codex:status`(被禁止 model 调用),只有你能运行它。Codex 的评审还在跑。我不让你干等——先在写最终报告前补齐最后几块(lead 的 AI 核心逻辑、Dockerfile、迁移),然后把我的完整独立评审交付,Codex 的部分稍后并入。

最后一块拼图:核查 CRM 推送的幂等性(决定我那个"重试重复推送"的结论该多强),读 `warning_export.py`。

核查完成,结论清晰。`warning_export.py` 的 CRM 推送**不带幂等键**,且我发现一处疑似字段互换 bug(`sku_coding ← product_name`、`sku_name ← product_id`,见 `warning_export.py:219-224`)。

我的独立评审已完整成型。Codex 那份还在跑且取回被卡在只有你能运行的 `/codex:status`,所以先把**我(Claude)的评审**完整交付,Codex 的随后并入。

---

# AMER Sales Assist —— Claude 独立多维评审

## 总体印象

这是一个**代码工艺达到资深水准、但运维成熟度还差一档**的模块化单体。架构治理(铁律)、测试设计(fake 而非 mock)、文档密度、Docker、LLM 降级链都属于"高于一般生产项目"的水平。可改进的面**主要不在代码本身,而在工程化/可观测/运维这层**——也就是"把这套好代码安全地长期运行下去"所需的脚手架。

成熟度定位:**代码 ≈ 4.5/5,运维 ≈ 3/5**。

## 维度评分总表

| 维度 | 分 | 一句话 |
|---|---|---|
| 架构与模块化 | 4.5 | 铁律治理优秀;遗留:跨模块迁移编号、import 期单例 |
| 代码质量 | 4.5 | 干净一致;少量 380–630 行大文件,1 处疑似字段互换 |
| 测试 | 4.5 | fake 驱动、全确定性、走真实 HTTP;但**无 CI 强制** |
| 类型安全 | 4.0 | mypy 全绿,但只 `check_untyped_defs`,未 `disallow_untyped_defs` |
| 错误处理与韧性 | 3.5 | 降级链极好;但**重试幂等性 + Celery 无超时**是真缺口 |
| 安全 | 3.5 | 基本功扎实(参数化/密钥不入库);CORS 默认、生产校验偏薄 |
| **可观测性** | **2.5** | 只有日志,**无指标/追踪/告警**——定时任务系统的最大风险 |
| 性能与扩展 | 3.0 | **无连接池**(per-request 连),当前流量够、上量即瓶颈 |
| 配置管理 | 4.0 | `env_*` 解析 + `.env.example` 同步好;生产必填校验偏薄 |
| 文档 | 4.5 | 同体量项目里罕见的好;轻微陈旧(494 vs 507) |
| **依赖/构建/CI** | **2.5** | Docker 优秀,但**无 CI、无依赖锁、容器无资源上限** |

---

## 重点问题(按主题,带证据)

### 1. 工程化:有质量门,却没人把门 ⚠️ 最高 ROI
- `.github/` **为空**——ruff / mypy / 494+ 测试这套投入巨大的质量基线,**没有任何 CI 在 push/PR 时强制**。`.pre-commit-config.yaml` 还是"可选启用"。一次忘跑就能把 main 带绿成红。
- **无依赖锁**:`pyproject.toml:10-22` 全是范围 pin(`>=0.116,<0.117`),无 `uv.lock`/`requirements.txt`。Docker 每次 `pip install .[mysql,celery]` 重新解析,**构建不可复现**,某天上游发版就可能漂移。

### 2. 可观测性:近乎空白
- 全仓 grep 无 `prometheus/opentelemetry/statsd/metric`,只有中间件里 `perf_counter` 打日志(`app/shared/middleware.py:32`)。
- churn 是**定时任务驱动**的业务系统:一个周一早 7 点的检测任务静默失败,**没有指标、没有告警**,只能靠人去翻日志。这是当前架构与可观测性之间最大的不匹配。

### 3. 韧性与幂等:降级很好,重试会重复
- `app/churn/tasks.py:51-59` 任何异常都 `self.retry`(max 3);重试时 `run_detection` 换新 `run_id` **重跑全部批次**(`service.py:196-265`)。
- warning 行靠 dedup 不会重复入库 ✓,**但** `detect_sales_batch` 把 `saved_warnings` 推 CRM(`service.py:332`),而 `saved_warnings` **含已匹配的存量预警**(`warning_persister.py:289-303`);CRM 导出 `saveCustomerWarn` **不带幂等键**(`warning_export.py:128-135`)。⇒ 重试/每轮检测都会**对同一批预警重复 POST**,是否产生重复告警取决于 CRM 端 upsert 语义——而代码没有约束也没记录这个假设。
- `app/celery_app.py:20-44` **无 `task_time_limit`/`soft_time_limit`/`acks_late`**:卡死的检测任务会一直占着 worker。
- **重试退避不一致**:`warning_export.py:301` 有指数退避 ✓,但 `lead/clients/crm.py:156-168` 重试**无退避**(立即重发)。
- **疑似字段互换 bug**(`warning_export.py:219-224`):`sku_coding ← product_name`、`sku_name ← product_id`,看起来正好反了(coding 应取 id)。请核对 CRM 字段语义。

### 4. 安全:基本功好,边角要收口
- ✅ `.env` 未进 git、`.gitignore` 含生产 dump、SQL 全参数化、密钥不落库、LLM 提示词显式声明"时间轴不可信、不执行其中指令"(`recommendation_service.py:46`)+ Pydantic `extra="forbid"`——LLM 安全卫生做得好。
- ⚠️ `app/main.py:156-162` CORS 默认 `*` + `allow_credentials=True` + 全通配方法/头;且生产配置校验(`main.py:82-109`)**只查 3 个 key**,不校验 CORS、不校验密钥存在——错配的生产也能起来。
- ⚠️ `.env.example:91,109` 写了**真实生产 DB IP**(`47.114.77.122`/`8.138.104.14`),已入库,泄露基础设施拓扑。

### 5. 性能与扩展:无连接池
- `app/shared/db_mysql.py:168-177` 每次查询 `connect_with_retry` 现连现关,lead 的 `db/pool.py` 也只是转调它(grep 无任何 pool)。churn 批量低频无所谓;但 lead 是 HTTP 面,**每个 API 请求都新建跨主机 MySQL 连接**(其一还是跨地域 47.x↔8.x),连接建立延迟叠加在每个请求上。当前流量够用,上量即首个瓶颈。

### 6. 运维/部署
- `docker-compose.yml` 有 `restart:always` + healthcheck ✓,但 **api/celery 容器无 mem/cpu 上限**——内存泄漏或峰值可拖垮宿主。
- 迁移 `migrations/mysql/churn/*.sql` **仅向前、无回滚、疑似手工执行**(无 alembic/版本表);lead 表建在 `churn/017`(架构文档已认领为历史债)。
- 文档轻微陈旧:`CLAUDE.md`/Makefile 标基线 **494**,而近期 review 记录是 **507**。

---

## 亮点(值得保持)

模块边界与"铁律"治理、fake-over-mock 的确定性测试、多阶段非 root Dockerfile + graceful shutdown、LLM"重试带反馈→确定性模板"的降级链、CRM 客户端对 `1==True` 这类坑的显式防御——这些都是经验沉淀,别在后续重构里弄丢。

---

## 如果只做 5 件事(按 ROI 排序)

| # | 事项 | 工作量 | 为什么 |
|---|---|---|---|
| 1 | **加 CI**:GitHub Actions 跑 `ruff + mypy + pytest`,PR 必过 | ~半天 | 质量基线已全有,只差强制;投入最低、收益最高 |
| 2 | **可观测最小集**:Celery 任务成功/失败/耗时指标 + 检测 run 失败告警(`prometheus-client` + `/metrics`,或推到现有监控) | 1–2 天 | 定时业务系统不能只靠翻日志 |
| 3 | **幂等收口**:`run_detection` 重试不重推 CRM(检查点或幂等键)+ Celery 补 `task_time_limit`/`acks_late` + 修 `sku_coding/sku_name` 疑似互换 | 1–2 天 | 直接关系到给业务员的告警质量 |
| 4 | **可复现 + 兜底**:加依赖锁(uv/pip-tools constraints)+ compose 容器资源上限 | ~半天 | 构建可复现 + 宿主不被拖垮 |
| 5 | **安全收口**:生产配置校验扩展(CORS 非 `*`、关键密钥存在)+ `.env.example` 去真实 IP + CRM 重试加退避 | ~半天 | 低成本堵住错配与信息泄露 |

第 2、3 项里的 Celery 超时可合并一起做。

---

## Codex 那份评审怎么并进来

Codex 的独立评审(`task-mqegknpk-7t64lh`,同样 11 维)**仍在运行**,取回被卡在一个**只有你能运行**的命令上(我这边 `/codex:status` 被禁止 model 调用)。两个办法,你选一个:

1. **你运行 `/codex:status`** 看它跑完没,完了跟我说一句"取回评审报告",我用 `--resume-last` 拉回全文,再和上面这份逐条对照(谁发现了对方没发现的、评分分歧在哪)。
2. 让我**定时自动重试取回**(任务一结束 resume-last 通常就能拉到),我设个 ~20 分钟的回看。

你想走哪条?或者要我先就上面某一项(比如 CI 配置、或那个 sku 字段 bug)直接动手?

---

### Turn 3  *(2026-06-15 00:24)*

**User:**

 /codex:status

**Assistant:**

你运行了 `/codex:status`,companion 现在应该解锁了。我派 codex-rescue 用 `--resume-last` 把完整报告拉回来(明确不新建任务)。

Codex 的报告回来了。在综合两份评审前,我先**亲自核验 Codex 那几个我漏掉的高价值发现**(无鉴权、token 入库、JSON 日志 bug),避免转述错误。

三个都已核验,结论清晰(并有校准):

- **JSON 日志 bug — 确认且严重**:`logging_config.py:20` 生产日志是手拼字符串 `"message":"%(message)s"`,而全仓大量 `logger.warning("... %s", exc)` 打外部错误文本——异常消息一旦含引号/换行就产出**非法 JSON**,偏偏在出事时最需要日志时坏掉。Codex 抓得准。
- **lead 全平台无鉴权 — 确认,且是"刻意钉死的契约"**:`test_security_surface.py:36` 明写"无鉴权",docstring 说"employee_code 只是业务员 ID 不是凭证"。这是我漏掉的最大一条,严重性取决于暴露面。
- **token 入 api_log — 确认但需降级**:`test_api_log_privacy.py:71` 自己说"这些接口 schema 本不含敏感字段,风险低"——是 denylist 加固项(将来 schema 演进才会真泄露),非当前活跃泄露。Codex 略微高估。

下面是两份评审的综合。

---

# Claude × Codex 联合评审综合

## 一句话

两份**独立**评审在结构性问题上高度重合——重合的就是**最高置信、必修**的。真正的价值在不重合处:**Codex 钻得更深(代码级安全与缺陷),我铺得更广(工程化/运维脚手架),两边各有一个标志性盲区**——Codex 整份**没提"无 CI"**,我**漏了"无鉴权"**。

---

## 一、强共识(两边独立命中 = 闭眼修)

| 问题                                     | 证据                                           |
| -------------------------------------- | -------------------------------------------- |
| MySQL 无连接池,per-request 现连现关            | `shared/db_mysql.py:168`                     |
| Celery 重试不幂等(run_id 每次随机、重跑全批、重推 CRM)  | `churn/tasks.py:51` · `service.py:196,332`   |
| CORS 默认 `*` + `allow_credentials=True` | `main.py:156`                                |
| mypy 未开 `disallow_untyped_defs`,不算严格   | `pyproject.toml:74`                          |
| 无 metrics/追踪                           | 全仓 grep 无 prometheus/otel                    |
| 依赖只范围 pin、无锁                           | `pyproject.toml:10`                          |
| 生产配置校验偏薄(只查 3 个 churn key)             | `main.py:92`                                 |
| 测试基线文档陈旧(494 vs 实际 507)                | `CLAUDE.md:10` · `README.md:43`              |
| 大函数多职责 / import 期全局单例                  | `service.py:188` · `lead/clients/crm.py:231` |

---

## 二、Codex 抓到、我漏了的(✓ 已逐条核验)

| 问题 | 我的校准 |
|---|---|
| **lead 全平台无鉴权**(`test_security_surface.py:36`) | ✓ 真实且刻意。**全场最高危**——但严重性取决于暴露面(可能是 nginx 后内网专用)。先定威胁模型 |
| **生产 JSON 日志被引号/换行破坏**(`logging_config.py:20`) | ✓ 真 bug,出事时坏。比"无 metrics"更该先修 |
| token/敏感字段留在 api_log(`api_log.py:37` 只剥 `score_context`) | ✓ 但**当前低危**(schema 不含敏感字段),属加固项 |
| 配置 import 期 `int/float(os.getenv)` 非法值即崩(`settings.py:22`) | ✓ 合理,应 fail-fast 化 |
| N+1 / 大 IN:geo 单次拉 2 万候选(`geo_repo.py:67`)、churn 逐 warning 查(`mysql_repo.py:167`) | ✓ 比我只说"无连接池"更具体 |
| 批量持久化路径未加 customer lock(`warning_persister.py:83` vs 单客户 `:147` 有锁) | ✓ 并发盲区,补我幂等结论的另一面 |
| response 模型定义了却没挂 `response_model`,OpenAPI 契约空转(`response.py` · `leads.py:18`) | ✓ 类型安全好补 |
| README 仍说 shared 含 auth/CRM,与架构文档冲突;`sqlalchemy_engine.py` 注释说在 shared 实则在 churn | ✓ 陈旧比我说的(494/507)更深 |
| Docker base 未 pin digest、uvicorn `--forwarded-allow-ips *` 信任过宽 | ✓ 供应链/信任边界 |

## 三、我抓到、Codex 漏了的

| 问题                                                                        | 证据                                                  |
| ------------------------------------------------------------------------- | --------------------------------------------------- |
| **无 CI**——`.github/` 为空,ruff/mypy/507 测试**无人在 PR 强制**(Codex 整份未提)         | `.github/` 空                                        |
| **疑似字段互换 bug**:`sku_coding ← product_name`、`sku_name ← product_id`(看着正好反) | `warning_export.py:219-224`                         |
| CRM 导出**无幂等键** + 重试退避不一致(导出有退避、lead CRM 客户端无)                             | `warning_export.py:128` · `lead/clients/crm.py:156` |
| Celery 无 `task_time_limit`/`acks_late`,卡死任务占 worker                       | `celery_app.py:20`                                  |
| 容器无 mem/cpu 上限                                                            | `docker-compose.yml`                                |
| 迁移仅向前、无回滚、疑似手工执行                                                          | `migrations/mysql/churn/`                           |
| `.env.example` 写了真实生产 DB IP                                               | `.env.example:91,109`                               |

---

## 四、评分分歧与校准

| 维度 | Claude | Codex | 校准后 & 理由 |
|---|---|---|---|
| 安全 | 3.5 | **2.0** | **取 2–2.5,Codex 更准**——我漏了无鉴权 |
| 代码质量 | 4.5 | 3.0 | ~3.5–4,Codex 对多职责函数更严,合理 |
| 文档 | 4.5 | 3.0 | ~3.5,Codex 找到的陈旧更深,我高估了 |
| 可观测性 | 2.5 | 3.0 | ~2.5,Codex 给了 request_id 分但又揪出日志 bug,我偏严更对 |
| 依赖/构建/CI | **2.5** | 3.0 | **取 2.5,我更准**——Codex 没评 CI 缺失才给到 3 |
| 其余(架构/测试/类型/韧性/性能/配置) | — | — | 两边一致(差 ≤0.5),无实质分歧 |

校准后画像:**代码工艺 ~4,运维/安全成熟度 ~2.5–3**。

---

## 五、合并后的优先级(取代两份各自的 top-5)

**P0 —— 先堵风险**
1. **安全收口(先定威胁模型)**:确认 lead 是否可达信任网络之外;若是 → 网关/API-Key 鉴权 + 生产 CORS 白名单 + api_log 改 allowlist 脱敏。
2. **幂等与并发**:Celery 幂等键(`job_type+biz_date`)+ 批量路径加锁或 DB upsert + 导出带幂等键/去重 + **修 sku 字段互换**。

**P1 —— 补成熟度**
3. **CI**(最便宜、Codex 漏的):GitHub Actions 跑 `ruff+mypy+pytest` 做 PR 门禁。
4. **可观测**:先**修 JSON 日志**(换真 formatter,如 `python-json-logger`)+ 任务/外部调用 metrics + 检测失败告警。
5. **配置 fail-fast**:迁 pydantic-settings + 生产校验 lead DB/密钥 + 同步 `.env.example` 默认值。

**P2 —— 收尾**
6. lead 输出挂 `response_model`、拆大函数、依赖锁 + 镜像 digest pin + 容器资源上限、迁移工具化、文档一致性修订 + `.env.example` 去真实 IP。

---

两份评审都同意:**底子是好的,要做的是"把好代码安全地长期运行"的加固**,不是返工。

我可以直接动手的几个低风险切入点(按性价比):**① 修 sku 字段互换**(几行、纯 bug)、**② 修生产 JSON 日志**(独立、影响排障)、**③ 搭 CI 骨架**(半天、收益最大)。要我从哪个开始?或者先把这份综合落成一份 `docs/review-2026-06-15.md` 存档?

No response requested.

No response requested.

---

### Turn 4  *(2026-06-15 07:14)*

**User:**

② 修生产 JSON 日志(独立、影响排障)、③ 搭CI 骨架(半天、收益最大)，分析一下这两个点的价值

**Assistant:**

两个点的价值性质完全不同:**② 是修一个具体潜伏 bug(线性价值),③ 是给所有后续工作上保险(复利价值)**。逐个拆。

---

## ② 修生产 JSON 日志

### 问题本质
`shared/logging_config.py:20` 的生产格式是**手拼字符串**,不是真正的 JSON 序列化:
```python
fmt = '{"time":"%(asctime)s",...,"message":"%(message)s"}'
```
`%(message)s` 直接插值,**不做 JSON 转义**。两类内容会击穿它:
1. **消息含引号/换行/反斜杠** —— 异常文本天然带这些。如 `crm.py:167` `logger.warning("CRM %s ... failed: %s", ..., exc)`、`recommendation_service.py:177` 打 `ValidationError`(多行 + 引号)。
2. **`logger.exception` 的 traceback** —— 这是更狠的:`logging.Formatter` 把 traceback 追加在格式串**之后、另起几行**。所以一次 `main.py:226` 的兜底异常实际产出:
```
{"time":"...","message":"unhandled error on POST /api/v1/leads/..."}
Traceback (most recent call last):
  File "...", line ...
```
traceback 那几行**完全在 JSON 对象之外**,是裸行。

### 失败场景
- 这个格式**只在 `APP_ENV=production` 生效**——也就是说,JSON 日志是**专门为机器解析而存在的**,却恰好在"被机器解析"这件事上是坏的。
- 失败时机最坏:健康的 INFO 行能解析,**出错的行(异常/CRM 失败/DB 抖动/LLM 校验失败)解析不了**。日志管道平时好好的,一出事就崩——你最需要它的那一刻它最不可靠。
- 对 churn 尤其致命:churn **没有 metrics**,`logger.exception` 几乎是它唯一的故障信号(`service.py:220/229`、`tasks.py:54`),而这个唯一信号正是被打碎成裸行的那类。

### 影响面 / 修复成本 / 风险
- **影响面**:生产事故排障质量;若已接 ELK/Loki/CloudWatch/Datadog 的 JSON 解析,错误行会进 parse-failure 桶或被丢,基于 `level:ERROR` 的告警会**漏计**。条件:取决于是否真的在做 JSON 摄取——但既然特地按 production 切了 JSON,意图就是摄取。
- **成本**:小。两条路——引 `python-json-logger`(一个依赖),或自写 ~15 行 `Formatter` 子类用 `json.dumps` 收 `record.getMessage()` + `formatException(exc_info)` + extras。
- **风险**:极低。隔离在一个已集中化的文件,dev 格式不动,改动让日志**更**合规不会更差。顺手可补:把 `request_id` 也放进格式(现在有 `RequestIDMiddleware` 却没进日志,无法按请求关联)。
- **可测**:打一条含引号/换行的日志,断言输出能 `json.loads` 成功——确定性、契合本项目测试风格。

### 价值定性
**真 bug、低成本、低风险、零依赖**。唯一的"条件"是日志管道是否真在解析 JSON;但这功能本就是为此存在的。**单点 ROI 很高,但价值是线性的——修好这一个,不防下一个。**

---

## ③ 搭 CI 骨架

### 问题本质
`.github/` 为空。项目重金投入的三道质量门——`ruff`、`mypy`(全 app/ 须全绿)、`pytest`(507,**全确定性、无外部依赖**)——**没有任何自动化在 PR/push 上强制**。`.pre-commit-config.yaml` 还是"可选启用",靠每个开发者自觉本地装。

### 为什么对**这个**项目价值特别高
1. **测试是天生为 CI 准备的**。"全确定性、无外部依赖"——不需要 DB/Redis/网络。这恰是大多数项目搞 CI 最难的部分,而这里**已经解决了**。所以 CI 几乎纯收益:`pip install .[all]` + 三条命令,边际成本异常低。
2. **铁律目前靠人肉守**。CLAUDE.md 的硬规则——`grep -rn "from app.lead\|from app.churn" app/shared/` **须为空**、churn/lead 互不 import、mypy 全绿、基线测试数——写着"违反即回滚",但**没有任何东西自动执行**。那条 grep 本身就是一行 CI step。**CI 把"铁律"从文档变成强制。**
3. **铁律 #2 自己就要求可 bisect**。#2 明说 churn 改动要能独立回滚/bisect,因为真实集成路径只有 fake 覆盖。**而 bisect 只有在 main 始终是绿的时候才可信**——main 能变红,bisect 就是噪声。CI 直接支撑项目自己定的风险策略。
4. **单人项目最吃这个**。git 显示单作者、36 commits。mypy/ruff/测试的投入,**价值上限取决于"每次都记得跑"的纪律**;CI 把纪律依赖去掉,是留给"累了的、切了上下文的、或要带第二个人上手的"未来的你的安全网。

### 影响面 / 修复成本 / 风险
- **影响面**:**无条件**。每一次后续改动、绿色基线的可信度、铁律执行、可 bisect 性,全覆盖。
- **成本**:低。一个 workflow(~30–40 行):matrix 跑 py3.10–3.12(pyproject 是 `>=3.10,<3.13`)→ `pip install .[all]` → `ruff format --check` / `ruff check` / `mypy` / `pytest -q` + 铁律 grep 守卫。含调绿 + 开分支保护,约半天。
- **风险**:近零(纯增量,不碰 app 代码)。唯一"风险"是 CI 可能**暴露干净环境下并非真绿**——Codex 在沙箱就撞到 `mypy 1.20.2 内部错误` 和 ruff/pytest 缓存目录问题。但这不是风险,是 CI **在干它该干的事**:告诉你"507 全绿"是只在作者的 WSL 成立,还是在干净机器也成立。诚实地讲成本:**半天是"假设套件在干净环境真绿";要预留点修复可复现性的时间。**

### 价值定性
**全 review 里 ROI 最高**。它不直接修任何 bug,但它**institutionalize(制度化)其余一切**——sku 修复、JSON 日志修复、幂等改造,落地后都被 CI 永久守住不回归。**它是复利/杠杆型价值。**

---

## 对比与排序

| | ② JSON 日志 | ③ CI 骨架 |
|---|---|---|
| 价值类型 | 线性(修一个潜伏 bug) | **复利(放大所有后续工作)** |
| 成本 | 小、**有界** | 半天、**可能要清可复现性** |
| 风险 | 极低 | 近零 |
| 价值是否有条件 | 是(取决于日志管道) | **否(无条件)** |
| 独立性 | 完全独立 | 是其它所有改动的底座 |

**建议先 ③ 再 ②**,理由是排序本身有价值:
- 先立 CI,再修 JSON 日志(并补它的测试)——**CI 当场验证这个修复、并永久护住它**;反过来先修 ②,下一次改动仍能悄悄打破它。
- CI 还会告诉你 ② 那条新测试在干净环境**是否真过**。
- 让 JSON 日志修复**作为第一个走新 CI 的 PR**,顺带把流水线端到端跑通验证一遍。

唯一会让我倒过来(先 ②)的情况:你想要一个**今天就拿到、时间完全可控**的快赢,而不想为 CI 调绿冒"半天变一天"的不确定性。

要我现在动手吗?可以**先搭 CI 骨架**(顺便用它照出干净环境的真实绿/红状态),或者你要的话**先修 JSON 日志**拿个有界快赢。

---
