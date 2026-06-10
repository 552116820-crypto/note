# 智能拓客系统 · 全接口多维度测试（最终版）

## Context

用户要"测试所有接口，从多个方面测"，确认走最全面方案：离线(TestClient+mock) + 连真库 live 冒烟 + 承接压测 + 真实 DeepSeek LLM，并补写新测试补齐空白维度。本版已吸收 Codex 独立审查的 4 个致命风险与 7 处测试盲区。

项目是 FastAPI 智能拓客系统，6 个端点（5 业务 + `/healthz`），统一 envelope `{code,message,data}`。现有 22 测试文件、约 271 个离线单测（`TestClient`+Fake 类 mock），基础扎实，空白集中在：envelope 契约无单一守护者、`/healthz` 零测试、缓存键污染/降级路径/passport 时区/中间件隐私等失败模式缺回归。

**关键代码结论（已与 Codex 交叉核验）**：错误码 **1001(EmployeeNotFound/404) 与 1004(InvalidCustomerId) 当前不可达**——`app/schemas/request.py:9` 的 `_coerce_digit_id` 对非数字 customer_id 先抛 1002；未知员工在 `app/repositories/employee_repo.py:64-68` 软返回空 context 而非抛异常；`app/services/lead_service.py`、各 repo 中唯一的业务 raise 只有候选过多(`geo_repo.py:79`)。`settings.py:97` 有 `employee_not_found_404` 配置项但无代码引用。**测试只验证现状（非数字 cid→1002、未知员工→软返回），不为这两个码强造可达路径**，在契约测试里加注释说明即可。

## 测试范围：6 接口 × 7 维度

|接口|功能正确|参数边界|错误码/降级|缓存|并发/性能|安全/隐私|envelope/data 契约|
|---|---|---|---|---|---|---|---|
|`/healthz`|🟡补|—|🟡补(db挂)|—|—|🟡补|🟡补(不套)|
|`score-and-info`|✅+🟡补不变量|✅|✅+🟡补skip降级|✅+🟡补键污染|连库压测|🟡补|🟡补矩阵|
|`skip`|✅|✅|✅(1505)|—|连库幂等|🟡补|🟡补矩阵|
|`industries`|✅|—|🟡补HTTP三态|🟡|连库|🟡补|🟡补矩阵|
|`timeline`|✅|✅|✅+🟡补键污染|✅+🟡补|连库压测|🟡补passport|🟡补矩阵|
|`recommend-reason`|✅|✅|✅+🟡补不缓存|✅+🟡补键污染|连库+真LLM|🟡补|🟡补data字段|
|中间件(api_log/request_size)|✅|—|✅|—|连库|🟡补脱敏+流式|—|

✅=现有覆盖 · 🟡补=本次补写 · 连库=连真库脚本 · —=不适用

## 执行计划

### 阶段 0 · 环境前置 + 防误连硬闸门（连库/压测前必做）

1. **改 MTU**：`ip -br addr | grep 198.18` 找隧道网卡 → `sudo ip link set dev <网卡> mtu 1400`（WSL 重启失效，网卡名会变）。`load_test.py` 的 `tunnel_dev()/mtu_hint()`(`load_test.py:55-70`) 可自动探测打印命令。
    
2. **防误连生产库闸门（吸收 Codex F3）**：执行任何连库脚本前，先打印 `.env` 实际生效的 `DB_HOST/DB_NAME/LOG_DB_HOST`（仅 host/db、**不打印密码**），人工确认指向**可写的开发/测试库**而非生产。注意 `settings.py:4-5` env 变量会覆盖 `.env`，且 `.env.example` 直指真实业务库 IP——务必核对真实生效值，不要假设。
    
3. **/healthz 闸口**：确认 `db=true` 且 `industries_source=db`（非 snapshot，否则压测数据失真）；`load_test.py:204` 的 `preflight()` 自动卡这两条。
    

### 阶段 1 · 离线层（执行现有 + 补写空白）

- 基线：`python3 -m pytest tests/ -q`（期望 ~271 全绿）。
    
- 新增 `tests/conftest.py`，**仅抽 opt-in fixture**（`client`/`client_no_raise`/`valid_score`/`Recorder`/`isolate_caches`/路径常量），**绝不用 autouse 全局替换缓存**——否则 singleflight/TTL 测试失真（Codex 重点提示）。现有文件渐进收敛，迁移后须复跑确认旧测试仍绿。
    
- 补写下表测试文件，全部纯离线、复用现有 Fake 类。
    

### 阶段 2 · 连库 live 冒烟（含少量自清写入，非纯只读 —— 吸收 Codex F1）

- `python3 scripts/http_smoke.py`：真实 uvicorn(8077)，校验 envelope + happy 不变量(求和=total_score、按分降序、len≤page_size) + page2 缓存命中 + `/skip` 幂等沉底。**会写 1 行 lead_skip + 若干 lead_api_log**，脚本自清。
    
- **清理安全前提（吸收 Codex F2）**：脚本按 `id > pre_max` 删 `lead_api_log`，若测试库**同时有其他真实流量会误删**。执行前确认该库此刻无并发写入；若不能保证，先做下方"可选加固"的精确清理改造再跑。
    
- `python3 scripts/probe_crm.py <有CRM历史的customer_id>`：CRM 三接口契约 + passport(北京时区 md5)验证。
    
- 可选：`integration_check.py`(resolver→geo→query_leads 端到端)、`phase0_probe.py`(schema/索引只读探针；注意它硬编码 host，运行时核对实际连接 host)。
    

### 阶段 3 · 承接压测 + 真实 LLM（吸收 Codex F4：默认不花钱）

- **先省钱全跑**：`python3 scripts/load_test.py --no-llm`（9 阶段：冷热路径/singleflight 击穿/冷路径 distinct 键/timeline 冷热/skip 并发幂等/日志开关吞吐差与丢弃率/混合负载；跳过 recommend）。
    
- **再单独真调 LLM（需显式确认成本）**：`python3 scripts/load_test.py --customer <有CRM历史cid>`（默认 `--recommend-n 5` 控成本）。**判据：`reason_source=="llm"` 至少出现一次**才算 LLM 通路打通；若全是 template，说明 key 未配或样例客户无时间轴，换客户重试。
    
- 各阶段量化 n/conc/ok/err/QPS/p50/p95/p99；与 `docs/接口文档-对接前端.md:472-480` 的已有基线对照判定（执行时读取该基线）。
    

### 阶段 4 · 汇总测试报告（产出 TEST_REPORT.md）

含：环境与运行日期(passport 按天，须标注；目标 DB host/db 非密钥) / 离线结果(通过·失败·跳过·覆盖率) / **envelope 契约矩阵结论** / 连库冒烟逐段 PASS·FAIL / 压测分阶段阈值判定(hot/cold p95、singleflight 冷算=1、api_log 丢弃=0) / `reason_source` 分布 + LLM token 消耗 / CRM 契约结论 / **1001·1004 不可达明确结论** / 风险与清理核对(lead_skip·lead_api_log 测试行已自清) / CI 建议(离线层适配，连库层标 `@pytest.mark.live` 默认 skip)。

## 新增测试文件清单（已按 Codex 建议归并）

|新文件|覆盖空白|复用|
|---|---|---|
|`tests/conftest.py`|统一 opt-in fixture/路径常量(无 autouse)|`Recorder`(test_api_boundary.py:38)、`isolate_caches`(test_api_errors.py:109)|
|`tests/test_envelope_contract.py`|**P0** 全端点×全错误码 envelope 形态+status↔code 矩阵 + 各端点 data 字段契约(recommend 的 reason_source/degraded) + 1001/1004 不可达注释（**合并原 domain_error_codes，单一真相源 app/main.py:74-111**）|`_raiser` + `DBUnavailable`/`LogDBUnavailable`|
|`tests/test_healthz.py`|**P0** happy + db 挂形态 + 锁死"不套 envelope"（独立保留）|monkeypatch `app.main.ping`/`industry_resolver`|
|`tests/test_cache_key_isolation.py`|**P1** score(归一 employee+5位坐标+半径+版本)/timeline(customer+date)/recommend(customer+employee+date+fingerprint+model_ver) 键构造与跨键不污染 + skip-set 读失败降级为空集(lead_service.py:121-126)|`InProcessTTLCache`、monkeypatch repo 抛异常|
|`tests/test_crm_time_boundary.py`|**P1** passport 跨时区：北京 00:00–08:00 vs UTC 同 today 贯穿 cache/window/passport(crm.py:54-70、timeline_service.py:202-209)|注入固定 tz/clock|
|`tests/test_api_log_privacy.py`|**P1** api_log 敏感字段不落库：password/token/authorization + 现有 score_context 裁剪(api_log.py:35)|`test_api_log_helpers.py` 现有范式|
|`tests/test_security_surface.py`|**P2**（收窄）无鉴权契约 + 注入入参(SQL 式 employee_code/customer_id)被拒或安全透传 + 异常 content-type 不裸崩|conftest、注入串参数化|
|`tests/test_score_invariants_http.py`|**P1** HTTP happy 不变量：求和/按分降序+距离 tie-break/page_size cap/list 长度|桩 `lead_service.query_leads` 返多项 blob|
|`tests/test_industries_fallback_http.py`|**P1** HTTP 下 source 三态 db/snapshot/none(industry_resolver.py:43)|monkeypatch `industry_resolver._source`|
|增补 `tests/test_request_body_limit.py`|**P2** request_size 流式累计超限分支(request_size.py:62-68)，非只 Content-Length 快路径|现有文件追加|

最高 ROI：`test_envelope_contract.py` 一处定义 `_assert_envelope`（断言 `set(json)=={code,message,data}`、`code∈合法集`、`status==CODE_TO_STATUS.get(code,200)`、内部错误不泄露异常细节），全错误路径共用。

## 可选 · 测试基础设施加固（提升安全性与可判定性）

按需采纳，改的是 `scripts/`（非接口本身）：

- **精确清理（Codex F2，目标库有并发流量时必做）**：把 `http_smoke.py:188`、`load_test.py:366,395` 的 `DELETE ... WHERE id>pre_max` 改为按本次 run 的 employee marker / 唯一 run_id 精确删，避免误删并发真实日志。
    
- **非零退出码（Codex I2）**：`http_smoke.py`/`integration_check.py` 现仅 print PASS/FAIL，加 `sys.exit(非零)` 或结果 JSON，便于 CI 门禁。
    
- **分阶段阈值（Codex I3）**：`load_test.py` 总判定现仅看 `err==0`，补 hot/cold p95、singleflight 冷算=1、api_log dropped=0 的阶段阈值（基线取 docs）。
    
- **LLM 闸门**：真调 LLM 增加 `ALLOW_LIVE_LLM=1` + 检测到 `CI` 环境变量则强制 `--no-llm`。
    

## 验证步骤

cd /mnt/e/smart_lead_generation  
​  
# 离线：基线 + 新增（无需库/Redis/网络）  
python3 -m pytest tests/ -q  
python3 -m pytest tests/test_envelope_contract.py tests/test_healthz.py \  
  tests/test_cache_key_isolation.py tests/test_crm_time_boundary.py \  
  tests/test_api_log_privacy.py tests/test_security_surface.py \  
  tests/test_score_invariants_http.py tests/test_industries_fallback_http.py -v  
​  
# 连库前置：改 MTU + 核对目标库（非生产）  
ip -br addr | grep 198.18  
sudo ip link set dev <上一步网卡> mtu 1400  
python3 -c "from app.config.settings import settings as s; print('DB:',s.db_host,s.db_name,'| LOG:',s.log_db_host)"  # 人工确认非生产  
​  
# 连库 live 冒烟（含自清写入）  
python3 scripts/http_smoke.py  
python3 scripts/probe_crm.py <有CRM历史的customer_id>  
​  
# 压测：先不花钱，再单独真调 LLM  
python3 scripts/load_test.py --no-llm  
python3 scripts/load_test.py --customer <有CRM历史cid>   # 真调 DeepSeek，看 reason_source 是否出现 llm

判定标准：离线全绿；契约矩阵全端点全错误码绿；`http_smoke.py` 每段 PASS；`load_test.py --no-llm` 总错误=0 且 singleflight 冷算=1；真调 LLM 阶段 `reason_source` 至少一次 `llm`；测试写入行确认自清。

## 风险点（执行时遵守）

- **防误连生产**：连库前强制打印并核对真实生效的 DB host/db（env 覆盖 .env）；确认指向可写测试库。
    
- **写数据 + 清理**：仅 skip/日志阶段写库，按 pre-id 自清；目标库有并发流量时先做精确清理加固，否则会误删他人日志行。崩溃残留会打印手删 SQL。
    
- **LLM 花钱**：默认 `--no-llm`；真调为独立显式步骤，CI/无人值守禁用。
    
- **MTU 黑洞**：WSL 重启 MTU 回 9000，连库前必重设；preflight 会 abort 防数据失真。
    
- **端口固定**：http_smoke(8077)/load_test(8078) 不可重入，勿并发跑。
    

## 会用到的能力

**扫描范围**（本次会话可见）：

- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
    
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily
    

**会用**：

- Agent Explore：摸清接口/测试/依赖三方面（探索阶段，已完成）
    
- Agent Plan：设计 6×7 测试维度矩阵与空白清单（已完成）
    
- Skill codex:rescue：对本 plan 做独立审查，确认 1001/1004 不可达、抓出 4 个执行风险与 7 处盲区（已完成，已吸收进本版）
    
- Skill code-review：补写完新测试后审查测试代码质量（可选增强）
    

**考虑过但不用**：

- Skill verify / run：本可真实启动服务观察，但项目自带 http_smoke.py / load_test.py 已线程内起真实 uvicorn 打真实后端，带 preflight/MTU 探测/自清理，更贴本项目——用脚本不用 skill。
    
- Skill simplify：与 code-review 功能重叠（只清理不查 bug），被覆盖。
    
- Skill security-review：面向"审查代码改动安全性"，本次安全维度是黑盒测接口鉴权/注入/隐私，用 test_security_surface.py + test_api_log_privacy.py 更直接。
    
- Skill claude-api：recommend-reason 走 DeepSeek(OpenAI 兼容)，不涉及 Anthropic，无需查 Claude API 参考。
    
- MCP chrome-devtools / firecrawl / jina / tavily：均为浏览器/web 抓取，与后端 API 测试无关。