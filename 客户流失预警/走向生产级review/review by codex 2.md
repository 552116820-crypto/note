# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019d89b3-7b9b-7313-951d-5e8d5ca14138`
- **范围**: Turn 1-8
- **导出时间**: 2026-04-14 14:19:06

---

### Turn 1  *(2026-04-14 01:58)*

**User:**

请你从业务逻辑，项目代码，工程落地等方面，综合评价这个项目是否达到生产级

**Assistant:**

我会先快速读项目结构、关键分层和测试覆盖，再跑一遍测试或至少做可执行性检查。评价会按生产级关心的风险来组织，不只看代码是否能跑。

仓库已经明显做过一次从单体到“共享基础设施 + 功能切片”的改造，当前工作树也有大量未提交变更。我会基于当前文件状态评价，不把这些未提交变更回滚或当作历史基线。

README 和启动逻辑显示 churn 是主功能、scoring 还只是脚手架。接下来我会重点看 churn 的算法、去重/持久化、CRM 数据接入和运维边界，因为这些决定“能否生产运行”。

代码层面有一些生产化动作：API Key、请求 ID、MySQL 仓储、去重键、Celery 定时任务和迁移文档都存在。但 scoring 明确未实现，且 churn 还需要看测试和数据库迁移能否支撑并发与恢复。

测试结果是 `98 passed, 1 skipped`，说明当前单元/接口层回归是健康的。现在我在补最后一层判断：生产交付不只看测试绿，还要看数据校准、迁移、运行治理和未实现模块是否会进入用户可见范围。

**结论**

当前项目还不能算“完整生产级”。如果只看 `churn` 流失预警主功能，它已经接近“可灰度的生产候选”：分层清晰、算法闭环存在、MySQL 持久化和去重机制也有雏形。但如果按整个仓库和工程交付标准评估，还不建议直接全量上线。

**主要依据**

优点很明确：`churn` 已实现三类检测，包含 EWMA 周期预测、IQR 去噪、置信度、严重度、频率/量级/关键产品预警，见 [engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):149、[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):248、[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):353、[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py):473。Service 层也承担了任务、分批、批次检测、去重、人工丰富、关闭预警等编排，见 [service.py](/mnt/e/churn-warning-engine/app/churn/service.py):85 和 [service.py](/mnt/e/churn-warning-engine/app/churn/service.py):116。MySQL 侧有 `dedup_key + active_slot` 唯一约束和 `GET_LOCK` 分布式锁，见 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):44 和 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):244。生产配置校验也覆盖了仓储、数据库和 API Key，见 [main.py](/mnt/e/churn-warning-engine/app/main.py):42。

但业务逻辑还缺“生产校准”。当前阈值主要是配置默认值和规则推导，缺少历史回测、误报/漏报统计、不同客户分层阈值、季节性/节假日处理、低金额基数保护。README 也明确说 API 模式只从回溯窗口内有订单的客户发现检测对象，长期未下单客户会漏掉，见 [README.md](/mnt/e/churn-warning-engine/README.md):112 和 [README.md](/mnt/e/churn-warning-engine/README.md):114。这对“流失预警”是核心风险。

代码层面，`scoring` 还是脚手架，不是可用功能：打分引擎直接 `NotImplementedError`，路由没有端点，任务函数 `pass`，见 [engine.py](/mnt/e/churn-warning-engine/app/scoring/engine.py):15、[routes.py](/mnt/e/churn-warning-engine/app/scoring/routes.py):12、[tasks.py](/mnt/e/churn-warning-engine/app/scoring/tasks.py):13。如果对外宣称“客户智能平台”，这部分不能算生产级。

工程落地上，迁移和运维还偏手工。MySQL 仓储构造时会 `create_all`，见 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):84，这不适合作为正式迁移机制；迁移 SQL 也明确是手工迁移且不是全幂等，见 [001_warning_records_active_dedup.sql](/mnt/e/churn-warning-engine/migrations/mysql/churn/001_warning_records_active_dedup.sql):1。批次处理状态只在内存对象上标记，MySQL 保存 job 时只持久化 `batches`，不会持久化 `processed_batches`，见 [service.py](/mnt/e/churn-warning-engine/app/churn/service.py):215 和 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):116。

安全和运行治理也不足：API Key 有，但默认 CORS 是 `*` 且允许 credentials，见 [main.py](/mnt/e/churn-warning-engine/app/main.py):91；`/healthz` 只返回 `ok`，没有数据库、CRM、Celery 可用性检查，见 [main.py](/mnt/e/churn-warning-engine/app/main.py):100。仓库里也没看到 Dockerfile、CI 配置或 Alembic 迁移入口。当前 git 工作树还有大量旧文件删除和新结构未跟踪，发布快照本身还没整理干净。

**测试情况**

我跑了 `pytest -q`，结果是 `98 passed, 1 skipped`。这是加分项，尤其是 churn 的边界、去重、API、配置都有覆盖。但 MySQL 相关测试主要用 SQLite 内存库，不是真 MySQL 集成测试，见 [test_mysql_repo.py](/mnt/e/churn-warning-engine/tests/churn/test_mysql_repo.py):15；真实 CRM 端到端脚本在仓库根目录，当前 pytest 配置只跑 `tests/`，见 [pyproject.toml](/mnt/e/churn-warning-engine/pyproject.toml):36。

**建议评级**

按 `churn` 单功能：生产候选，建议先灰度，评分约 `7/10`。  
按整个项目：未达到生产级，评分约 `5.5/10`。

上线前最关键的三件事是：补业务回测与阈值校准，补正式迁移/部署/监控链路，明确屏蔽或完成 `scoring` 未实现模块。

---

### Turn 2  *(2026-04-14 02:10)*

**User:**

scoring不用管；我为什么要用到MySQL啊，我数据从api接口拿，运行日志存进项目目录中不就行了吗？

**Assistant:**

可以不用 MySQL，前提是你接受对应的运行约束。这里的核心点是：**API 只解决订单数据来源，不解决预警系统自己的状态存储**。

当前代码里其实已经把两件事分开了：

- `REPOSITORY_TYPE=api`：订单/客户数据从 CRM API 拿。
- `CHURN_REPOSITORY_TYPE=mysql|memory`：预警任务、预警记录、人工处理状态、去重状态存在哪里。

为什么我不建议只“把运行日志存进项目目录”：

1. 日志不是业务状态  
   预警需要能查询、更新、关闭、人工丰富、去重。比如 `pending_enrich -> enriched -> resolved`，这些是业务数据，不是日志。代码里对应的是 [service.py](/mnt/e/churn-warning-engine/app/churn/service.py):241 和 [service.py](/mnt/e/churn-warning-engine/app/churn/service.py):269。

2. 去重需要可靠约束  
   同一个客户同一类活跃预警不能重复生成。MySQL 版本用了唯一约束和 `GET_LOCK`，见 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):55 和 [mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):244。日志文件很难保证并发任务、重跑、失败重试时不重复。

3. 项目目录不适合放生产状态  
   部署、升级、容器重建、git clean、权限变更、日志轮转都可能影响项目目录。生产数据一般应放数据库或受管持久化目录，并有备份和恢复策略。

4. Celery/多进程会放大问题  
   如果 API 服务、Celery worker、beat 分开运行，单纯写项目日志无法可靠共享“哪些预警已经存在、哪些已处理、哪个 batch 跑过”。

所以更准确的结论是：**你不一定需要 MySQL，但你需要一个可靠的 Churn 状态存储。**

如果你的运行形态是单机、单进程、内部 MVP，可以这样做：

```bash
REPOSITORY_TYPE=api
CHURN_REPOSITORY_TYPE=memory
```

这能跑，但重启后任务和预警状态会丢，人工处理状态也会丢，不建议叫生产级。

如果你不想引入 MySQL，我更建议做一个 `SQLiteChurnRepository`，把数据库文件放在项目外的持久化目录，比如 `/var/lib/churn-warning/churn.db`，用 SQLite 的唯一索引保存 `dedup_key + active_slot`，用事务保证去重。这样比“日志文件”轻很多，也更符合当前 `ChurnRepository` 抽象。MySQL 只是当前项目已经实现的生产持久化方案，不是唯一选择。

---

### Turn 3  *(2026-04-14 02:16)*

**User:**

灰度要怎么做

**Assistant:**

可以按“先影子运行，再小范围人工使用，再扩大”的方式做。针对这个项目，建议这样灰度：

**1. 先定灰度范围**

选 1-2 个销售 ID，先不要全量跑。你现在已经支持这个入口：

```bash
REPOSITORY_TYPE=api
CRM_API_SALES_IDS=003748,002527
API_KEY=your-key
uvicorn app.main:app --reload
```

如果你暂时不用 MySQL，也不要把 `APP_ENV` 设成 `production`，否则当前代码会强制要求 `CHURN_REPOSITORY_TYPE=mysql`。可以先用 `APP_ENV=staging` 或不设。

**2. 第一阶段：影子运行**

只跑检测，不让销售实际触达客户。每天或每周固定跑：

1. `POST /api/warning-jobs/start`
2. `GET /api/warning-jobs/{job_id}/batches`
3. 对每个 batch 调 `POST /api/warning-jobs/{job_id}/detect-batch`
4. 用 `GET /api/warnings/pending-enrich` 和 `GET /api/warnings/{warning_id}/feature-summary` 查看结果

这个阶段看几个指标：

- API 拉数是否稳定：分页、超时、返回量是否异常
- 每个销售/客户的预警数量是否离谱
- `frequency/product/volume` 哪类误报最多
- 是否有明显漏报：业务知道已流失，但系统没报
- 重复检测是否重复生成同类预警
- 结果解释销售是否看得懂

**3. 第二阶段：小范围人工确认**

让一个销售主管或运营同学每天看 `pending_enrich`，只做内部确认，不自动发客户话术。确认后手工调用：

```http
POST /api/warnings/{warning_id}/enrich
```

如果确认无效或已处理：

```http
POST /api/warnings/{warning_id}/resolve
```

这一阶段重点看“可用率”，比如 100 条预警里有多少条被认为值得跟进。低于一个阈值，比如低于 30%，就先调参数，不扩大范围。

**4. 第三阶段：小范围真实使用**

只放开给那 1-2 个销售 ID。要求所有客户触达仍由人工确认，系统只提供预警和建议话术。跑 1-2 个完整周期：

- `frequency/product`：至少 1-2 周
- `volume`：至少 1 个自然月末

这时建议记录一个简单台账：`warning_id、客户、异常类型、销售判断、是否触达、触达结果、是否误报`。

**5. 第四阶段：扩大范围**

当误报率、漏报反馈、API 稳定性都可接受后，再从 2 个销售扩大到 5 个、10 个、全量。每次扩大只改：

```bash
CRM_API_SALES_IDS=003748,002527,xxxx,yyyy
```

**关键提醒**

如果只是影子运行，日志存在项目目录勉强可以。但一旦进入第二阶段，涉及“人工确认、已解决状态、重复预警去重”，就需要可靠状态存储。你不一定要 MySQL，但至少要 SQLite/文件型仓储加文件锁，不能只靠运行日志。日志适合追踪，不能当业务数据库。

---

### Turn 4  *(2026-04-14 02:21)*

**User:**

把这个项目我需要解决的问题罗列出来，让我走向生产级的道路

**Assistant:**

下面按“走向生产级必须解决的问题”列清单，暂时不管 `scoring`。

**P0：不上线前必须解决**

1. **决定并实现可靠状态存储**
   - API 只提供订单数据；预警任务、预警记录、人工处理状态、去重状态仍需要存储。
   - 如果不想用 MySQL，建议实现 `SQLiteChurnRepository`，不要用“项目目录里的运行日志”当业务状态。
   - 需要覆盖：`warning_jobs`、`warning_records`、`processed_batches`、状态流转、去重键、唯一约束、备份恢复。
   - 相关位置：[repository.py](/mnt/e/churn-warning-engine/app/churn/repository.py)、[memory_repo.py](/mnt/e/churn-warning-engine/app/churn/memory_repo.py)、[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py)。

2. **补“从 API 拿数据”的覆盖边界**
   - 当前 API 模式只能发现回溯窗口内有订单的客户，长期没下单的客户可能直接漏掉。
   - 这是流失预警的核心风险：最需要预警的人可能因为窗口内没有订单而不参与检测。
   - 要么接入客户主数据/销售客户清单接口，要么明确业务范围只覆盖“窗口内有过订单的客户”。

3. **做历史回测和阈值校准**
   - 现在的阈值如 `FREQUENCY_MULTIPLIER=1.5`、`VOLUME_DROP_THRESHOLD=0.35`、`PRODUCT_MULTIPLIER=2.0` 是规则默认值，不等于生产可用阈值。
   - 至少要用历史数据回测：命中多少真实流失客户、误报多少、不同销售/品类/客户等级是否要分层阈值。
   - 相关位置：[engine.py](/mnt/e/churn-warning-engine/app/churn/engine.py)、[config.py](/mnt/e/churn-warning-engine/app/churn/config.py)。

4. **把任务执行做成可恢复**
   - 当前批次处理依赖去重避免重复，但 `processed_batches` 在 MySQL 里没有真正持久化；Celery 里 batch 失败后只是记录并跳过。
   - 生产级需要：任务状态、批次状态、失败原因、重试次数、可恢复执行、避免同一 `job_type + biz_date` 并发重复跑。
   - 相关位置：[service.py](/mnt/e/churn-warning-engine/app/churn/service.py):215、[tasks.py](/mnt/e/churn-warning-engine/app/churn/tasks.py):56。

5. **正式化迁移机制**
   - 当前 MySQL 仓储初始化会 `create_all`，迁移 SQL 也是手工脚本。生产环境不建议让应用启动时自动改表。
   - 如果用 SQLite，也一样要有初始化/迁移脚本、schema version、回滚策略。
   - 相关位置：[mysql_repo.py](/mnt/e/churn-warning-engine/app/churn/mysql_repo.py):84、[001_warning_records_active_dedup.sql](/mnt/e/churn-warning-engine/migrations/mysql/churn/001_warning_records_active_dedup.sql)。

**P1：灰度前建议解决**

6. **增加健康检查和运行观测**
   - `/healthz` 现在只返回 `ok`，需要至少能检查 CRM API、状态存储、Celery/定时任务是否正常。
   - 如果日志写文件，要配置日志目录、权限、轮转、保留天数、错误告警。
   - 相关位置：[main.py](/mnt/e/churn-warning-engine/app/main.py):100、[logging_config.py](/mnt/e/churn-warning-engine/app/shared/logging_config.py)。

7. **收紧安全配置**
   - `API_KEY` 已有，但生产还应收紧 `CORS_ORIGINS`，避免默认 `*`。
   - API Key 也只是最低限度认证；如果给多角色使用，后续要做权限区分，例如运营只能处理预警，销售只能看自己客户。
   - 相关位置：[auth.py](/mnt/e/churn-warning-engine/app/shared/auth.py)、[main.py](/mnt/e/churn-warning-engine/app/main.py):91。

8. **补真实集成测试**
   - 当前 `pytest` 是绿的：`98 passed, 1 skipped`，这是好基础。
   - 但 MySQL/CRM 真实链路测试还不够，根目录的 `test_crm_api.py` 不在 pytest 默认 `tests/` 范围内。
   - 上线前至少补：CRM API 分页/超时/异常返回、真实状态库并发去重、Celery 重试恢复。

9. **部署形态固定下来**
   - 现在没看到 Dockerfile、CI、systemd/compose/K8s 配置。
   - 需要明确：API 服务怎么启动、Celery worker/beat 是否分离、环境变量如何管理、数据目录放哪里、日志放哪里、怎么重启和回滚。

**P2：上线后持续优化**

10. **预警闭环指标**
   - 记录每条预警是否有效、是否触达、客户反馈、是否成交/恢复采购。
   - 用这些数据调阈值，而不是长期依赖固定规则。

11. **业务规则补强**
   - 增加客户等级、品类季节性、春节/长假、项目型客户和周期型客户区分。
   - 给小金额客户设置最低金额门槛，避免小额波动产生大量噪音。

12. **API 性能和缓存策略**
   - 当前 API 仓储会把窗口内订单缓存到进程内存。数据量大时要评估内存、拉取耗时、分页失败、CRM 限流。
   - 必要时做本地快照表或按销售 ID 分片缓存。

**最短生产路径**

如果你坚持“不用 MySQL”，我建议路径是：

1. 实现 `SQLiteChurnRepository`，状态文件放项目外，比如 `/var/lib/churn-warning/churn.db`。
2. 保持 `REPOSITORY_TYPE=api`，新增 `CHURN_REPOSITORY_TYPE=sqlite`。
3. 补 SQLite 唯一索引和事务，保证同客户同类型活跃预警不重复。
4. 用 1-2 个 `CRM_API_SALES_IDS` 灰度影子运行一周。
5. 回测和人工标注误报/漏报，再扩大销售范围。

---

### Turn 5  *(2026-04-14 02:23)*

**User:**

还是用mysql吧，mysql也只用要存运行日志，这样可以吧

**Assistant:**

可以，但要把概念说准：**MySQL 不建议只存“运行日志”，而是存“预警运行状态和预警业务结果”**。CRM 原始订单数据仍然可以全部从 API 拿，不必落 MySQL。

推荐边界：

```bash
REPOSITORY_TYPE=api
CHURN_REPOSITORY_TYPE=mysql
DATABASE_URL=mysql+pymysql://user:pass@host/db
API_KEY=your-key
```

这样 MySQL 存这些最小必要数据：

- `warning_jobs`：某次检测任务、业务日期、批次划分
- `warning_records`：生成的预警、状态、特征摘要、话术、去重键
- 后续建议补一张 `warning_batches`：批次状态、失败原因、重试次数
- 可选 `warning_events`：人工丰富、关闭、重跑等审计流水

而这些不必存 MySQL：

- CRM 订单明细原始数据，继续从 API 拉
- 普通运行日志、错误栈、请求日志，建议写 stdout/文件，由日志系统或文件轮转处理

关键原因是：系统要支持 `pending_enrich` 查询、人工 enrich、resolve、重跑去重。只存“日志文本”无法可靠支撑这些业务操作。只要你把 MySQL 当作“预警状态库”，而不是“订单数据仓库”，这个方案是合理的，也是当前项目已有设计最接近生产级的路径。

---

### Turn 6  *(2026-04-14 02:47)*

**User:**

这个功能开发好了，怎么给到团队里的其他人

**Assistant:**

建议不要用压缩包传项目目录，按“代码交付 + 环境交付 + 使用交付”走。

**给开发同事**

1. 整理当前改动，确认不提交缓存/本地文件：

```bash
git status --short
pytest -q
```

2. 建一个功能分支并提交：

```bash
git checkout -b feature/churn-warning-production-ready
git add app tests docs README.md CHANGELOG.md pyproject.toml migrations/mysql
git commit -m "Add churn warning API and MySQL state persistence"
git push origin feature/churn-warning-production-ready
```

3. 发 PR/MR，让团队 review。PR 里写清楚：
- 订单数据仍来自 CRM API：`REPOSITORY_TYPE=api`
- 预警状态存在 MySQL：`CHURN_REPOSITORY_TYPE=mysql`
- 新增/依赖环境变量：`DATABASE_URL`、`API_KEY`、`CRM_API_SALES_IDS`、`CRM_API_PAGE_SIZE`
- 迁移脚本位置：`migrations/mysql/churn/001_warning_records_active_dedup.sql`
- 测试结果：`98 passed, 1 skipped`

**给测试/业务同事**

提供一个灰度环境，不要让他们本地跑源码。部署时用类似配置：

```bash
APP_ENV=staging
REPOSITORY_TYPE=api
CHURN_REPOSITORY_TYPE=mysql
DATABASE_URL=mysql+pymysql://user:pass@host/db
API_KEY=your-key
CRM_API_SALES_IDS=003748,002527
```

给他们这几个入口就够了：
- `POST /api/warning-jobs/start`
- `GET /api/warning-jobs/{job_id}/batches`
- `POST /api/warning-jobs/{job_id}/detect-batch`
- `GET /api/warnings/pending-enrich`
- `GET /api/warnings/{warning_id}/feature-summary`
- `POST /api/warnings/{warning_id}/enrich`
- `POST /api/warnings/{warning_id}/resolve`

配套给 [api-guide.md](/mnt/e/churn-warning-engine/docs/api-guide.md)，再补一份 Postman/Apifox 集合会更好。

**给运维/部署同事**

交付一个最小上线说明：

```bash
pip install -e .[all]
mysql -u user -p database_name < migrations/mysql/churn/001_warning_records_active_dedup.sql
APP_ENV=production REPOSITORY_TYPE=api CHURN_REPOSITORY_TYPE=mysql DATABASE_URL=... API_KEY=... uvicorn app.main:app --host 0.0.0.0 --port 8000
celery -A app.celery_app worker --loglevel=info --beat
```

同时说明：MySQL 只存预警任务和预警记录，不存 CRM 原始订单；普通运行日志走 stdout 或日志文件轮转。

**你现在需要注意**

当前工作树之前显示有大量未提交改动和新文件。正式给团队前，先把历史删除、新目录迁移、文档、迁移脚本整理成一个清晰 PR，避免同事拿到的是“能跑但不可审”的工作区快照。

---

### Turn 7  *(2026-04-14 03:00)*

**User:**

整理成一个清晰 PR,这个PR要怎么做

**Assistant:**

我先看一下当前分支和工作区状态，这样给你的 PR 拆分步骤会贴合现在这个仓库，而不是泛泛说流程。

你现在这个工作区适合做成一个 **“churn 功能模块化 + API 数据源 + MySQL 预警状态持久化”** 的 PR。不要在 `main` 上直接提交；现在是在 `main`，而且有一大批删除、移动、新增文件。

我建议这样做：

**1. 先开分支**

```bash
git switch -c feature/churn-warning-api-mysql-state
```

这不会丢当前改动，只是把当前工作区挂到新分支上。

**2. 先确认 PR 范围**

当前改动包含：

- 旧的 `app/*.py` 单体模块删除
- 新增 `app/churn/`
- 新增 `app/shared/`
- 新增 `app/celery_app.py`
- 新增 README、docs、CHANGELOG
- 新增 `migrations/mysql/churn/`
- 新增 `app/scoring/` 和 `tests/scoring/`
- 新增根目录 `test_crm_api.py`

如果你说 `scoring` 不管，PR 里有两种选择：

- 推荐做法：保留 `app/scoring/`，但在 PR 描述里明确它只是 scaffold，不属于本次生产上线范围。因为当前 `app/main.py` 已经 include 了 scoring router，不提交它会导致导入失败。
- 更干净但要改代码：把 `app/main.py` 里 scoring router 移除，同时不提交 `app/scoring/` 和 `tests/scoring/`。这会让 PR 只聚焦 churn。

按当前状态，**建议先保留 scoring scaffold，但 PR 标题和描述不要把它当功能卖出去**。

**3. 不建议提交根目录 `test_crm_api.py`**

这个文件是手动真实 CRM API 验证脚本，不在 `tests/` 默认 pytest 范围内，而且写死了真实接口和销售 ID。建议先不放进 PR，除非你明确要团队一起维护这个脚本。

可以先 stage 主要改动：

```bash
git add -A
git restore --staged test_crm_api.py
```

然后检查：

```bash
git diff --cached --name-status
git diff --cached --stat
```

如果确认 `test_crm_api.py` 没被 staged，就提交：

```bash
pytest -q
git commit -m "Modularize churn warning engine with API source and MySQL state"
```

**4. push 并开 PR**

```bash
git push origin feature/churn-warning-api-mysql-state
```

然后在 Git 平台开 PR，目标分支选 `main`。

**PR 标题建议**

```text
Modularize churn warning engine with API source and MySQL state
```

**PR 描述可以这样写**

```markdown
## Summary

- Reorganize churn warning into vertical feature modules under `app/churn/`
- Move shared CRM/auth/db/logging infrastructure into `app/shared/`
- Add CRM API repository as the default order data source
- Add MySQL-backed churn state repository for jobs, warnings, statuses, and deduplication
- Add Celery entrypoint and scheduled churn detection task
- Add startup production config checks for repository type, database URL, and API key
- Add docs for API usage, environment variables, and MySQL migration

`app/scoring/` is scaffold only and is not part of this production rollout.

## Runtime Model

- CRM order/customer data source: `REPOSITORY_TYPE=api`
- Churn warning state store: `CHURN_REPOSITORY_TYPE=mysql`
- CRM raw orders are not persisted by this service
- MySQL stores warning jobs, warning records, manual enrichment/resolution state, and deduplication state

## Required Environment Variables

- `REPOSITORY_TYPE=api`
- `CHURN_REPOSITORY_TYPE=mysql`
- `DATABASE_URL=mysql+pymysql://...`
- `API_KEY=...`
- Optional: `CRM_API_SALES_IDS=...`
- Optional: `CRM_API_PAGE_SIZE=1000`

## Migration

Apply:

```bash
mysql -u user -p database_name < migrations/mysql/churn/001_warning_records_active_dedup.sql
```

## Verification

```bash
pytest -q
```

Result:

```text
98 passed, 1 skipped
```

## Notes

- API mode only controls CRM order data loading.
- Churn warning state still requires durable persistence in production.
- Current production recommendation is API data source + MySQL churn state repository.
```

**5. 开 PR 前再看一眼工作区**

```bash
git status --short
```

如果只剩下 `?? test_crm_api.py`，说明你把手动脚本留在本地了，这是可以的。你也可以把它移到 `scripts/` 后再单独开一个小 PR，但不要混进这个生产功能 PR。

---

### Turn 8  *(2026-04-14 03:14)*

**User:**

开 PR是什么啊？没理解，PR全称Pull Ruquests，是说把在主分支拉去你修改的代码吗

**Assistant:**

对，PR 全称是 **Pull Request**。你可以理解成：

**“我在一个分支上改好了代码，现在请求把这些改动合并回主分支。”**

不是“从主分支拉取代码”，而是“请求主分支接收你的代码”。

一个典型流程是：

```text
main 主分支
  |
  |  开一个新分支
  v
feature/churn-warning-api-mysql-state
  |
  |  你在这个分支上改代码、测试、提交
  v
提交 commit
  |
  |  在 GitHub/GitLab/Gitee 上创建 Pull Request
  v
团队 review
  |
  |  没问题后 merge
  v
main 主分支获得你的改动
```

为什么不直接改 `main`？

因为 PR 提供几个团队协作能力：

- 让别人看到你改了哪些文件
- 让别人 review 代码
- 自动跑测试
- 讨论风险点，比如环境变量、数据库迁移、API 变化
- 最后由负责人点 “Merge” 合并进主分支

在你这个项目里，PR 的意思就是：

> 你现在把 `churn` 模块化、API 数据源、MySQL 预警状态存储这些改动放到一个新分支里，然后发起一个请求，让团队确认后合并到 `main`。

命令层面大概是：

```bash
git switch -c feature/churn-warning-api-mysql-state
git add -A
git commit -m "Modularize churn warning engine with API source and MySQL state"
git push origin feature/churn-warning-api-mysql-state
```

然后去 GitHub/GitLab/Gitee 页面上点 **Create Pull Request** 或 **New Merge Request**。

不同平台叫法略有差异：

- GitHub / Gitee 通常叫 Pull Request，简称 PR
- GitLab 通常叫 Merge Request，简称 MR

本质都是“请求把我的分支合并进主分支”。

---
