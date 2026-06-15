# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `9f49c4f7-69ad-44db-bb13-a9224eaf37e5`
- **范围**: Turn 1-8
- **导出时间**: 2026-06-11 09:03:37

---

### Turn 1  *(2026-06-10 08:38)*

**User:**

帮我测试一下这个项目的所有接口，从多个方面测

**Assistant:**

我来先了解一下这个项目的接口结构和现有测试基础，然后制定一个多维度的测试计划。让我先并行启动几个 Explore 代理来摸清项目情况。

探索已经非常充分了。我对项目有了清晰的全貌：FastAPI 项目，5 个业务接口 + 1 个健康检查，已有 271 个离线单测（TestClient + mock），连真库需要修 WSL MTU 和阿里云白名单。

现在我读几个关键文件确认测试方案的复用点，同时启动一个 Plan agent 设计多维度测试矩阵。

我已经掌握了完整图景。综合三份探索和 Plan agent 的测试设计，关键事实清晰了：

- **6 个端点**（5 个业务 + `/healthz`），已有 **271 个离线单测**（TestClient + Fake 类 mock），但发现几处高价值空白：统一 envelope 契约没有单一守护者、`/healthz` 零测试、错误码 **1001/1004 疑似悬空**（定义了但可能不可达）、安全面（无鉴权/注入/content-type）不成体系。
- **连真库**需要先修 WSL MTU + 阿里云白名单 + `.env` 凭证，且会写 `lead_skip`（自清）、调 DeepSeek（花钱）。

在写最终方案前，有两个关键点需要你定夺——它们直接决定测试范围和是否产生真实副作用。

两个都选了最全面的方案：**全套测试（离线 + 连库 + 压测 + 真实 LLM）** 且**执行 + 补写补齐空白**。

在写最终方案前，我再确认几个执行细节——压测脚本的参数和阶段、以及那两个疑似悬空错误码（1001/1004）的真实触发路径，这直接影响测试怎么写。

完整图景已经清晰，几个关键执行细节也确认了。特别是 Plan agent 标记的两个"悬空错误码"——我读了 `request.py:9` 的 `_coerce_digit_id` 和 `leads.py:30` 的路由逻辑，确认 **1004 在当前代码路径下不可达**（非数字 customer_id 在 schema 层就抛 1002，路由里 `int()` 拿到的已是纯数字串），**1001 也是软路径**（未知员工返回空 context 而非 404）。这两个是防御性定义，测试应当验证现状而非强造可达。

现在写最终方案到 plan 文件。

方案已写入 plan 文件。现在请求批准。

用户希望让 codex 审查这份 plan。我用 codex:rescue 把方案交给它做一轮独立审查，重点找遗漏的测试维度、不可行的步骤和风险点，然后据此给出最终 plan。

用户要求让 codex 审查 plan。按 codex:rescue 的流程，我先检查是否有可恢复的 Codex 线程。

没有可恢复的线程，直接路由到 Codex 做这轮只读审查。

court
<invoke name="Agent">
<parameter name="description">Codex 审查测试 plan</parameter>
<parameter name="subagent_type">codex:codex-rescue</parameter>
<parameter name="prompt">审查一份测试方案 plan（只读审查，不要修改项目任何文件）。

## 背景
项目：/mnt/e/smart_lead_generation —— 一个 FastAPI 智能拓客/lead generation 系统。
我（Claude）刚为"测试所有接口、从多个方面测"写了一份多维度测试方案，已落盘到：
/home/huangch001002/.claude/plans/mossy-whistling-ocean.md

用户已确认走最全面的方案：离线(TestClient+mock) + 连真库只读冒烟 + 承接压测 + 真实 DeepSeek LLM 调用，并且要补写新测试补齐空白维度。

## 项目关键事实（供你快速定位，可自行 Read 验证）
- 6 个端点：GET /healthz（app/main.py:68，故意不套 envelope）、POST /api/v1/leads/score-and-info（app/routers/leads.py:20）、POST /api/v1/leads/skip（leads.py:28，写 lead_skip）、GET /api/v1/leads/industries（leads.py:36，db 失败有 CSV 快照兜底）、POST /api/v1/leads/timeline（app/routers/customer_detail.py:18，调 CRM 三接口）、POST /api/v1/leads/recommend-reason（customer_detail.py:25，调 CRM + DeepSeek，失败降级模板）。
- 统一 envelope {code,message,data}；错误码与 http status 映射散落在 app/main.py:74-111；错误码定义在 app/errors.py。
- 现有 22 个测试文件、约 271 个离线单测（python3 -m pytest tests/），全部可离线跑，外部依赖用 FakeCRM/FakeLLM/FakeRedis + monkeypatch。
- 连库脚本在 scripts/：http_smoke.py（8077）、load_test.py（8078，9 阶段压测，--no-llm 可跳过 LLM）、probe_crm.py、integration_check.py、phase0_probe.py。
- 连库前必须修 WSL 隧道 MTU：`sudo ip link set dev <198.18.x网卡> mtu 1400`（WSL 重启失效，网卡名 eth0/1/2 会变）。
- 我在 plan 里下了一个关键判断：错误码 1001(EmployeeNotFound/404) 和 1004(InvalidCustomerId) 是"悬空/不可达"——因为 app/schemas/request.py:9 的 _coerce_digit_id 对非数字 customer_id 先抛 1002，leads.py:30 的 int() 永远拿到纯数字串；未知员工是软返回空 context。请你重点核验这个判断对不对（去读 app/services/lead_service.py、app/repositories/ 下相关文件，确认是否真的没有任何路径会抛 EmployeeNotFound / InvalidCustomerId）。

## 请审查并回答
1. **致命问题**：plan 里有没有事实错误、不可行步骤、或会破坏数据/花冤枉钱的风险（尤其连库写数据、真调 LLM、MTU、生产库误连）？
2. **1001/1004 悬空判断是否准确**：核验代码，给出确定结论（可达则给出触发路径；不可达则确认）。
3. **测试维度有无遗漏**：6 接口 × 7 维度的矩阵里，有没有重要的接口行为/失败模式/边界没被现有测试或我计划补写的测试覆盖？特别关注：缓存键污染、并发竞态、CRM passport 时区、降级标志(degraded/reason_source/source)、接口日志中间件(api_log)、请求体大小限制(request_size)。
4. **新增测试文件的设计是否合理**：conftest.py 渐进迁移、envelope 契约矩阵、healthz、security_surface、score_invariants 这几个文件，组织方式有没有更好的；有没有多余或应合并的。
5. **执行顺序与判定标准**：离线→连库冒烟→压测+LLM 的顺序、各阶段 PASS/FAIL 判据是否合理。

请给出一份"修订建议清单"（按严重程度排序：致命/重要/优化），我会据此产出最终 plan。请务必先 Read 那份 plan 文件和上述关键源码再下结论，不要凭空臆测。这是只读审查任务，不要改任何文件。</parameter>
</invoke>

Codex 给出了一份很扎实的审查。我先核对它的关键发现，再据此重写最终 plan：

- **确认我对 1001/1004 的判断正确**（不可达），还补充了 `settings.py:97` 有个 `employee_not_found_404` 配置项但根本没被代码使用——更坐实了"防御性悬空码"。
- **抓到 4 个我低估的执行风险**：阶段 2 名为"只读"实则写库（名实不符）；按 `id > max` 删日志会误删测试期间的并发真实流量；无防误连生产库的硬闸门（`.env.example` 直指真实业务库 IP，且 env 变量会覆盖 .env）；默认 `load_test.py` 就真调 LLM 花钱。
- **补了几处真实测试盲区**：缓存键构造/跨键污染、skip-set 读取失败降级、CRM passport 跨时区(北京 00:00–08:00)、api_log 敏感字段脱敏、request_size 流式超限、recommend data 字段契约。
- **测试文件组织优化**：conftest 只抽 opt-in fixture(不要 autouse 替换缓存，否则 singleflight/TTL 测试失真)、envelope 与 domain-error 矩阵合并、security_surface 收窄并拆出 api_log_privacy。

这些我全部吸收。现在重写最终 plan。

最终 plan 已写好，Codex 的审查意见全部吸收：确认了 1001/1004 不可达的结论、加了防误连生产库的硬闸门、把"只读冒烟"正名为"含自清写入的 live 冒烟"、压测默认 `--no-llm`、并把缓存键污染/passport 时区/api_log 脱敏/流式超限等盲区补成了独立测试文件。现在请求批准。

计划已批准。我先建立任务清单，同时跑离线基线测试确认现有 271 个测试的现状。先加载任务工具并启动基线。

基线健康——**271 个测试全过，3.8 秒**。现在建立任务清单，并读取补写测试需要的关键源码实现（缓存键构造、passport 时区、api_log 脱敏、request_size 流式、skip-set 降级）。

实现细节清楚了。关键点记录：缓存键三个构造函数都做 strip 归一化（`store.py:23-49`）；passport 用 `crm_today()` 固定北京时区（`crm.py:54-70`）；api_log 仅剥 `score_context`（`api_log.py:36`）；request_size 流式分支在 `limited_receive` 累计（`request_size.py:62-68`）；skip-set 读失败降级为空集（`lead_service.py:121-126`）。

现在读现有测试作为编写范式，避免重复并复用 Fake 类。先更新任务状态。

范式齐全了。关键复用点：`_raiser` 桩异常、`_isolate_caches` 隔离 detail_cache、`_OkCRM/_BoomCRM`、api_log 端到端用 `monkeypatch _schedule_write` 捕获 record、`TestClient(raise_server_exceptions=False)` 测兜底 500。我也确认了 `build_reason_cache_key` 的 fingerprint 维度已被 `test_recommend_cache_fingerprint` 覆盖、api_log 纯函数已被 helpers 覆盖——新文件避开重复，聚焦端到端与未覆盖维度。

先写 conftest.py（仅 opt-in fixture，无 autouse）和最高价值的 envelope 契约矩阵，立即跑一遍验证不破坏现有测试。

现在写最高价值的 envelope 契约矩阵——一处定义 `_assert_envelope`，覆盖全端点全错误码的 `{code,message,data}` 形态与 status↔code 对齐，并固化 1001/1004 经 handler 的映射（同时注明业务路径不可达）。

先验证这两个文件——跑 envelope 契约测试，并确认新 conftest 没有破坏现有 271 个测试。

API Error: Unable to connect to API (ECONNRESET)

No response requested.

---

### Turn 2  *(2026-06-11 00:07)*

**User:**

继续吧

**Assistant:**

Please run /login · API Error: 401 Invalid authentication credentials

No response requested.

---

### Turn 3  *(2026-06-11 00:09)*

**User:**

继续

**Assistant:**

Envelope 契约 14 个全过，conftest 也没破坏现有测试。继续批量补写其余测试文件。先写确定性较高的四个：healthz、缓存键隔离、passport 时区边界、industries 三态降级。

先跑这批四个文件验证。

18 个全过。现在写涉及中间件和 content-type 的两个文件——api_log 隐私端到端、安全面。

跑这两个验证（content-type 和注入透传是潜在坑点）。

14 个全过。最后两个文件需要确认评分函数的精确契约——读 `scoring.py`，用于 `_compute_blob` 的真实计算不变量测试。

评分契约清楚：`score_candidate` 返回 `(total, detail)` 且 `total == sum(detail.values())`（`scoring.py:135`），距离曲线 0-20km=40、20-60线性、>60=0。现在写最后两个文件——评分不变量（真实 `_compute_blob` + mock repo）和流式超限增补。

增补 `test_request_body_limit.py` 的流式超限分支——直接在 ASGI 层驱动中间件，确定性触发 `limited_receive` 累计分支（无 Content-Length，避开快路径）。

所有文件写完。跑最后两个文件，然后全量回归确认无破坏。

7 个全过。现在跑全量回归。

**323 passed**——271 基线 + 52 新增，零破坏。离线层全部完成。

现在进入连库阶段。先做只读安全核对：确认生效配置指向哪个库、隧道网卡 MTU 现状（不连库、不写数据)。

关键发现——隧道网卡 eth1 的 **MTU 已经是 1400**（你之前修过），无需再动；Redis 配在本地 6379，DeepSeek key 已配。目标库是真实业务库（该项目本就连这一个库，`http_smoke` 的写入是一行自清的 `lead_skip`）。

MTU 这关已过，先做连通性探测（业务库/日志库/Redis），确认网络可达再跑冒烟。连库需要真实出站网络到阿里云。

业务库、日志库都通（MTU+白名单 OK）；Redis 没起，按设计回退进程内缓存——单进程冒烟/压测够用，Redis 跨进程路径已有离线 `test_redis_singleflight` 覆盖。

跑 `http_smoke` 前，先按 Codex F2 的提醒只读探一下日志库两表现状，确认没有活跃并发写入（否则按 id 自清会误删他人日志）。

日志库状态清晰：`lead_skip` 空、`lead_api_log` 最近写入是**昨天 16:28**——今天无活跃并发写入。Codex F2 的误删风险在此环境不成立，`http_smoke` 按 id 自清是安全的。

跑 HTTP 冒烟（真实 envelope + 评分不变量 + skip 幂等沉底 + lead_api_log 落行，全部自清）。

HTTP 冒烟**全段 PASS**：healthz `db=true`/70 行业、page1 冷算 10.2s（top score=100 求和正确）、page2 缓存命中 3.9ms、校验 1002、未知员工软返回 code=0（再次印证 1001 不可达）、`/skip` 幂等+沉底+计数全对、`lead_api_log` 落 9 行字段完整——**所有写入已自清**（lead_skip 测试行删除、9 条日志删除）。

现在验证 CRM 契约/passport（用 page1 的 top 客户 73308）。

CRM 契约**全部验证通过**：passport `d36c11...` 被 CRM 接受（state=200），三接口数据结构符合代码预期——visits 返回 3 条拜访记录（按 customer_id 分组的双层数组，客户"唐山海泰新能科技"），orders/services 正确返回空。客户 73308 有 CRM 历史，正好用于稍后真调 LLM。

连库冒烟阶段完成。现在跑承接压测——先 `--no-llm` 全 9 阶段不花钱。

承接压测**全 9 阶段 PASS，总错误 0**。关键指标：热路径 QPS 686（p99 34ms 缓存命中）、**singleflight 击穿 30 并发新键只冷算 1 次**、冷路径 ~6s(真打 DB 地理查询，合 README 基线)、timeline 热 QPS 1048 / degraded=0(CRM 三源健康)、**skip 10 并发幂等 DB 仅 1 行**、**api_log 落库 200/200 丢弃 0**(日志开销 ~11% QPS)。全部写入自清。

现在跑真调 LLM 的精简版（客户 73308 有拜访记录，能真走 DeepSeek），其余阶段调小避免重复长跑——判据是 `reason_source` 出现 `llm`。

真调 LLM 压测**全 PASS**——recommend 阶段 **reason_source: llm=3/3**，LLM 通路确认打通。注意到一个真实现象：DeepSeek 有一次返回**截断的非 JSON**（`{"recommend_reason": "...潜在订单`），attempt 1 失败后**重试成功**，最终仍走 llm——服务层的重试/降级兜底有效。

所有阶段 0 错、写入自清。压测阶段完成。现在汇总测试报告。

报告已生成。最后做一次清理核对——确认连库脚本的写入确实无残留（Codex F2 关注点）。

清理核对发现**残留 663 行** `lead_api_log`（2309 vs 跑前 1646）——脚本自清不完整。这是个真实问题，先诊断残留来源（确认是本轮测试产生、非真实流量），再安全清理。

时间全部落在本轮压测窗口（08:22–08:24）、endpoint 全是 4 个业务接口——**确认 663 行全是本轮测试数据，非真实流量**。根因是 `ENABLE_API_LOGGING=1` 下 load_test 前置阶段全程记日志，而脚本自清只覆盖日志开关阶段自身的行。已确认无并发真实流量，安全清理。

court
<invoke name="Bash">
<parameter name="command">python3 -c "
from app.config.settings import settings
print('enable_api_logging =', settings.enable_api_logging, '(True 即残留根因: load_test 前置阶段全程记日志，自清未覆盖)')
from app.db import log_pool
# 安全前提已确认：id>2385 全部为本轮测试 4 接口、08:22-08:24、无并发真实流量(最近真实写入为前一日)
n = log_pool.execute('DELETE FROM lead_api_log WHERE id > 2385')
print('已删除残留:', n, '行')
after = log_pool.fetch_all('SELECT COUNT(*) AS c FROM lead_api_log')[0]['c']
print('清理后 lead_api_log:', after, '(期望 1646)')
print('清理核对:', 'PASS 无残留' if after == 1646 else 'CHECK 仍有差异')
" 2>&1</parameter>
<parameter name="description">清理残留日志并验证</parameter>
<parameter name="dangerouslyDisableSandbox">true</parameter>
</invoke>

彻底清净：删除 663 行残留，`lead_api_log` 恢复到 **1646 = 跑前值，无残留**。根因确认（`ENABLE_API_LOGGING=1` 让 load_test 前置阶段全程记日志，脚本自清只覆盖日志开关阶段）。更新报告把这个发现记录进去。

court
<invoke name="Edit">
<parameter name="file_path">/mnt/e/smart_lead_generation/TEST_REPORT.md</parameter>
<parameter name="old_string">## 7. 风险与清理核对

- ✅ **写入全部自清**：lead_skip（http_smoke + load_test 各阶段）、lead_api_log（按 run 前 max_id 差量删）。跑前 lead_skip=0 行，跑后仍=0；日志表无本轮残留。
- ✅ **无并发误删风险**：核验日志库 lead_api_log 最近写入为前一日 16:28，本轮无活跃并发实例，按 id 自清安全（Codex F2 风险在此环境不成立）。
- ✅ **目标库已核对**：连库前打印生效 host/db 确认。
- ✅ **LLM 成本可控**：仅真调 3 次（精简量级）。
- ⚠️ Redis 未启动 → 进程内缓存（功能不受影响）。</parameter>
<parameter name="new_string">## 7. 风险与清理核对

- ⚠️→✅ **lead_api_log 自清不完整，已手动清净**：因 `.env` 的 `ENABLE_API_LOGGING=1`，load_test 前置各阶段(热/冷/timeline/skip)全程异步记日志，而脚本自清(phase_api_log_onoff)只覆盖"日志开关"阶段自身的行 → 残留 663 行。已诊断确认**全为本轮测试**(4 个业务接口、08:22–08:24、无并发真实流量)，按 `id>2385` 安全清除。**最终 lead_api_log = 1646 行 = 跑前值，无残留**。
- ✅ **lead_skip 自清完整**：跑前 0 行、跑后 0 行。
- ✅ **无并发误删风险**：核验最近真实写入为前一日 16:28，本轮无活跃并发实例，按 id 清理安全（Codex F2 担心的并发误删在此环境不成立）。
- ✅ **目标库已核对**：连库前打印生效 host/db 确认。
- ✅ **LLM 成本可控**：仅真调 3 次。
- ⚠️ Redis 未启动 → 进程内缓存（功能不受影响）。</parameter>
</invoke>

---

### Turn 4  *(2026-06-11 00:32)*

**User:**

你在干嘛，继续啊

**Assistant:**

抱歉，工具调用格式错了。继续更新报告。

court
<invoke name="Edit">
<parameter name="file_path">/mnt/e/smart_lead_generation/TEST_REPORT.md</parameter>
<parameter name="old_string">## 7. 风险与清理核对

- ✅ **写入全部自清**：lead_skip（http_smoke + load_test 各阶段）、lead_api_log（按 run 前 max_id 差量删）。跑前 lead_skip=0 行，跑后仍=0；日志表无本轮残留。
- ✅ **无并发误删风险**：核验日志库 lead_api_log 最近写入为前一日 16:28，本轮无活跃并发实例，按 id 自清安全（Codex F2 风险在此环境不成立）。
- ✅ **目标库已核对**：连库前打印生效 host/db 确认。
- ✅ **LLM 成本可控**：仅真调 3 次（精简量级）。
- ⚠️ Redis 未启动 → 进程内缓存（功能不受影响）。</parameter>
<parameter name="new_string">## 7. 风险与清理核对

- ⚠️→✅ **lead_api_log 自清不完整，已手动清净**：因 `.env` 的 `ENABLE_API_LOGGING=1`，load_test 前置各阶段(热/冷/timeline/skip)全程异步记日志，而脚本自清(phase_api_log_onoff)只覆盖"日志开关"阶段自身的行 → 残留 663 行。已诊断确认**全为本轮测试**(4 个业务接口、08:22–08:24、无并发真实流量)，按 `id>2385` 安全清除。**最终 lead_api_log = 1646 行 = 跑前值，无残留**。
- ✅ **lead_skip 自清完整**：跑前 0 行、跑后 0 行。
- ✅ **无并发误删风险**：核验最近真实写入为前一日 16:28，本轮无活跃并发实例，按 id 清理安全（Codex F2 担心的并发误删在此环境不成立）。
- ✅ **目标库已核对**：连库前打印生效 host/db 确认。
- ✅ **LLM 成本可控**：仅真调 3 次。
- ⚠️ Redis 未启动 → 进程内缓存（功能不受影响）。</parameter>
</invoke>

---

### Turn 5  *(2026-06-11 00:33)*

**User:**

继续啊

**Assistant:**

第 7 节更新好了。现在在第 8 节补上 load_test 自清缺陷这条发现。

court
<invoke name="Edit">
<parameter name="file_path">/mnt/e/smart_lead_generation/TEST_REPORT.md</parameter>
<parameter name="old_string">5. **[运维] Redis**：生产多副本部署需启用 Redis 以获得跨进程 singleflight；本轮单进程下进程内缓存已验证防击穿（冷算=1）。</parameter>
<parameter name="new_string">5. **[运维] Redis**：生产多副本部署需启用 Redis 以获得跨进程 singleflight；本轮单进程下进程内缓存已验证防击穿（冷算=1）。
6. **[脚本缺陷] load_test 在 `ENABLE_API_LOGGING=1` 下自清不完整**：脚本自清假设日志默认关、只清理"日志开关"阶段的行；当 `.env` 全局开启日志时，前置各阶段产生的日志不被清理（本轮残留 663 行，已手动清净）。建议脚本启动时记一个全局 `pre_max`、结束时统一 `DELETE id>全局pre_max`，或用 Codex F2 的 run-id 标记法精确清理。</parameter>
</invoke>

---

### Turn 6  *(2026-06-11 00:41)*

**User:**

你怎么回事啊，继续汇总测试报告 TEST_REPORT.md

**Assistant:**

报告全部更新完毕。标记最后一个任务完成。

全部完成。报告在 `TEST_REPORT.md`。

## 测试总结

**6 个接口 × 7 个维度，全部通过，零错误。**

### 离线层
- **323 个用例全绿**（271 基线 + **52 个新增**，1.3s 无依赖）
- 补齐了 9 个空白维度文件：envelope 契约矩阵、/healthz、缓存键隔离、passport 时区边界、api_log 隐私、安全面、评分不变量、industries 三态、请求体流式超限

### 连库实测
- **http_smoke**：healthz / 评分(求和+降序) / 缓存命中 3.9ms / skip 幂等沉底 / 日志落行 —— 全 PASS
- **probe_crm**：passport 被 CRM 接受，三接口契约正确（73308 抓到 3 条拜访记录）
- **压测 9 阶段**：总错误 0，singleflight 冷算=1，热路径 QPS 686、timeline 热 QPS 1048、skip 幂等 DB 仅 1 行、日志丢弃 0
- **真调 DeepSeek**：reason_source `llm=3/3`，LLM 通路打通

### 关键发现
1. **1001/1004 确认是不可达的防御性错误码**（与 Codex 交叉核验，实测佐证）
2. **DeepSeek 偶发返回截断 JSON**（疑似 max_tokens=400 边界），服务层重试兜底成功
3. **load_test 自清缺陷**：`ENABLE_API_LOGGING=1` 时前置阶段日志不被清理，本轮**残留 663 行已手动清净**

### 数据安全
连库前核对了目标库、确认无并发流量。最终核对：**lead_skip=0、lead_api_log=1646，均等于跑前值，零残留。**

报告里还有 CI 接入和脚本加固建议（标 `@pytest.mark.live` + `ALLOW_LIVE_*` 闸门、脚本退出码、分阶段 p95 阈值）。需要我把这些加固实际落地到代码吗？

---

### Turn 7  *(2026-06-11 00:47)*

**User:**

 CI 接入和脚本加固建议（标 @pytest.mark.live + ALLOW_LIVE_* 闸门、脚本退出码、分阶段 p95阈值）。做这个建议的价值大不大，帮我评估一下

**Assistant:**

先说结论：**这组加固整体价值"中等偏低"，而且高度依赖一个前提——你到底有没有 CI/自动化流水线。** 没有这个前提，四项里有三项是在打磨已经被"现状"解决掉的问题。我逐条拆给你看。

## 关键事实：现在的隔离其实已经够了

`http_smoke.py` / `load_test.py` / `probe_crm.py` 都在 `scripts/` 下，文件名也不是 `test_*.py`。**pytest 根本不会收集它们** —— `python3 -m pytest tests/` 永远只跑那 323 个离线用例，连库写库的脚本绝无可能被误触发。也就是说，"防止离线套件误连库写数据"这个最核心的安全目标，**当前的 scripts/ vs tests/ 物理隔离已经实现了**。

## 逐项评估

| 建议 | 真实价值 | 说明 |
|---|---|---|
| `@pytest.mark.live` 标记 | **低** | 只有把脚本"改写成 pytest 用例"才需要它。但脚本现在故意独立于 pytest，隔离已达成。为了打标记反而要先把脚本塞进 tests/，是先制造问题再解决问题。 |
| `ALLOW_LIVE_*` 环境闸门 | **中** | 唯一有点独立价值的一项——防止手滑 `python3 scripts/load_test.py` 直接打生产+烧 LLM。但脚本已有 preflight(healthz/MTU/source 校验) 兜底，且跑脚本的人是在"刻意"跑。属于锦上添花。 |
| 脚本非零退出码 | **价值=0 或必需，无中间态** | 没有 CI 时人是肉眼读 PASS/FAIL 的，退出码毫无意义；有 CI 时它是刚需。它的价值完全由"有没有 CI"决定，自身不产生价值。 |
| load_test 分阶段 p95 阈值 | **低，且有反作用** | 冷路径 p95=6.3s 是"真实 DB + 隧道 + 网络"的合成值，波动大。在共享真库上钉死阈值会频繁误报。数字已经打印出来了，人眼比对回归更靠谱。 |

## 这组建议的"天花板"在 CI，而 CI 现在不一定该上

四项里三项（标记、退出码、阈值）本质都是**为 CI 服务的零件**，自己不发光。所以真正的问题是：**该不该现在上 CI？**

而现在上 CI 有两个现实障碍：
- 这个目录**不是 git 仓库**（`Is a git repository: false`），CI 无从挂载；
- 按你的项目记忆，拓客系统**即将和 churn-warning-engine 合并**。在合并、目录结构重组之前搭 CI，大概率是白搭工，合并后还得重来。

所以我的判断：**这组加固现在做，投入产出比不划算。等仓库化 + 合并尘埃落定后，一次性做"CI 接入 + 退出码 + ALLOW_LIVE 闸门"才是顺势而为**（那时三项一起做、互相配套，价值才兑现）。p95 阈值我建议永远别做硬门禁，留作人工观测。

## 如果现在一定要花精力，我会优先修这个

比起上面四项，报告里那条 **load_test 自清缺陷（残留 663 行）才是真 bug**：`ENABLE_API_LOGGING=1` 时前置阶段的日志根本没被清理，我这次是手动兜的。它会**真实污染你的日志库**，且每次跑都复发。修法很小——脚本启动记一个全局 `pre_max`、结束时统一 `DELETE id>全局pre_max`。这个 10 行改动的实际收益，比那四项加起来都大。

**我的建议**：四项加固搁置到合并后随 CI 一起做；现在只值得花 5 分钟修 load_test 的全局清理。要我把这个清理修了吗？（顺带 DeepSeek 截断那条要不要把 `max_tokens` 从 400 提到比如 800，也是一行的事）

---

### Turn 8  *(2026-06-11 00:53)*

**User:**

修 load_test 的全局清理；把 max_tokens 从 400 提到 2000；CI是什么

**Assistant:**

先把两处代码改了，CI 我放最后解释。先读两个文件确认 `max_tokens` 的权威位置和 load_test 收尾块的确切代码。

清楚了。`max_tokens` 的生效值来自 `.env:42`（覆盖了 settings 默认），所以三处都要改：`.env`(运行值)、`settings.py:86`(代码默认)、`.env.example:61`(文档)。`.env` 含密钥，用定向 `sed` 改单行避免读出密钥。

先改 max_tokens 三处 + load_test 加全局清理基线：

验证 grep 用了小写没匹配到大写键（sed 本身没问题）。重新用 `-i` 确认，同时补上 load_test 的收尾全局清理块：

`settings` 仍读到 **400** —— 说明环境里有一个**真实环境变量** `DEEPSEEK_MAX_TOKENS=400`，它的优先级高于 `.env`（`settings.py:4-5` 注明 env > .env）。改 `.env` 没用。得先找到这个 env 变量从哪来。

整套 DeepSeek/DB 配置是作为**真实环境变量**注入当前 shell 的（注意 `DEEPSEEK_MODEL=deepseek-v4-flash` 也只在 env 里、连 .env 都没有），但不在标准 shell rc 文件里。我再找一下注入源——查 `/etc/environment`、`profile.d`、以及项目里可能被 source 的 env 脚本：

找到了根因——`~/.bashrc:127-133` 在启动时用 `set -a; source .env` 把 `.env` 的每一行**导出成真实环境变量**。Claude Code 会话启动时 source 的是**旧 .env（400）**，这个值被"焊进"了当前会话，覆盖了我刚改的磁盘 .env。验证一下：屏蔽这个继承变量后，settings 应读到 2000。

确认无误：屏蔽继承变量后 settings 读到 **2000**（证明 .env 改对了）；当前会话因启动时 source 的是旧 .env，仍是 400。我改了 settings.py，跑一遍离线套件确认没碰坏东西：

两处改动完成，323 测试仍全绿。

## 改动汇总

**1. load_test 全局清理（修了 663 行泄漏的 bug）**
- 开跑前记录全局 `pre_max`，收尾在 `finally` 里 `th.join` 等日志 drain 完，再 `DELETE id>全局pre_max` 兜全量
- 语法校验通过；下次跑 load_test 不论 `ENABLE_API_LOGGING` 开没开，日志都会清干净

**2. max_tokens 400 → 2000**（改了三处：`.env`、`.env.example`、`settings.py` 默认值）
- 屏蔽干扰变量验证 settings 确实读到 2000 ✓

**⚠️ 一个要你知道的坑**：你的 `~/.bashrc:127-133` 在启动时 `set -a; source .env`，把 `.env` 每行**导出成真实环境变量**。当前 Claude Code 会话启动时 source 的是**旧 .env（400）**，这个值被焊进了会话、优先级高于磁盘上的新 .env——所以**此刻**直接跑 load_test 仍会用 400。要生效，二选一：
- 开个新终端 / 重启会话（重新 source 新 .env）→ 自动变 2000
- 或当前会话手动 `export DEEPSEEK_MAX_TOKENS=2000`

（顺带：这也意味着你以后改 `.env` 任何项，都得重开会话才生效。生产/docker 直接读 .env 的话没这问题。）

---

## CI 是什么

**CI = Continuous Integration（持续集成）**，本质是一句话：**每次有人改了代码（push 或提 PR），一台服务器自动把你的测试跑一遍，红了就拦住、绿了才放行。**

放到你这个项目就具体成这样：

```
你 push 代码 → GitHub 自动拉起一个干净容器 → pip install + python3 -m pytest tests/
  → 323 个用例 1.3s 跑完 → 全绿 ✅ 合并；挂了 ❌ 在 PR 上标红、附失败日志
```

实现就是仓库里放一个 `.github/workflows/test.yml`（GitHub Actions），十几行 yaml，定义"在什么事件下、跑什么命令"。你不用手动跑，它在云端自动跑。

**它解决的真实问题**：你现在每次改完得**记得**手动 `pytest`。人会忘、会偷懒、会"就改一行应该没事"。CI 把这件事变成**强制且自动**——任何回归在合并前就被挡下，尤其多人协作或你自己隔几周回来改时，价值最大。

**为什么这跟那几项加固绑在一起**：
- 「脚本非零退出码」——CI 靠**退出码**判断红绿（exit 0=过，非0=挂）。没有 CI，退出码没人看；有了 CI，它才有意义。
- 「`@pytest.mark.live` + `ALLOW_LIVE_*`」——CI 跑在云端**连不到你的阿里云库**，必须能跳过连库用例只跑离线那 323 个。
- 所以这三项是「为 CI 服务的零件」，CI 不存在时它们悬空。

**为什么我建议你现在先别上**：CI 要挂在 git 仓库上，而你这目录现在 `不是 git 仓库`；加上拓客马上要和 churn-engine 合并、目录会重组。现在搭，合并后大概率推倒重来。**等仓库化 + 合并落定，再一次性上 CI（连带退出码、live 闸门）**才顺。那 323 个离线测试 1.3s、零依赖，是天生的 CI 料——到时候接上即用。

需要的话，等你这边 git 仓库就绪，我可以直接给你写好那个 `test.yml`。

---
