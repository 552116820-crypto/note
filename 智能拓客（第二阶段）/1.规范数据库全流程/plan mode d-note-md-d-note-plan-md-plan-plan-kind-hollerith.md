# 规范数据库与表名（churn_db → amer_sales_assist）— plan 评估与改进

## Context / 背景

你自己建的 `churn_db` 当初只考虑了客户流失预警；现在业务长出了"智能拓客"，两块属于同一个"AI 销售助手"领域、同一拨人维护。决定**复用这个库并趁早规范化**，为后续功能开发和团队接管做准备：

- 库名 `churn_db` → `amer_sales_assist`（中性、可扩展，装 churn + lead + 以后的 AI 分析全家桶）
- churn 5 张表加 `churn_` 模块前缀；`churn_ai_analyses` 已规范，仅移库不改名
- 新建两张拓客表 `lead_skip` / `lead_api_log`（**本期只建空表占位，写入链路延后**）
- 表/列写中文注释，沉淀命名规范文档

**拓扑（两台不同实例，别混）**：
- A 机 `47.114.77.122:3306` → `amer_crm_v2`（CRM 业务库，拓客只读，**本次完全不动**）
- B 机 `8.138.104.14:23356` → `churn_db`（churn 数据 ~11k 行，**本次重命名 + 新增 lead 表**）

**涉及两个项目**：
- `E:\churn-warning-engine`（FastAPI+SQLAlchemy+Celery，Docker 部署在 B 机侧）— 改动几乎全在这里
- `E:\smart_lead_generation`（当前目录，FastAPI+裸 PyMySQL，**非 git 仓库**）— 它的 `LOG_DB_*` 是死配置（`load_dotenv` 把 .env 读进了环境变量，但 `Settings` 模型没有字段去取它 → 没有任何代码使用它），本次只改配置串/脚本/文档

---

## 一、对你原 7 步 plan 的评估（你问的"是否合理 / 是否全面"）

骨架是对的，但有 **3 个会导致线上事故的硬伤** + 几处遗漏/顺序问题：

| 你的步骤 | 评估 |
|---|---|
| 1 备份 churn_db | ✅ 对，必须先备份 |
| 2 建新库迁表删旧库 | ⚠️ 原子性要补：**删旧库不能急**——留着空 `churn_db` 当回滚后路，验证通过后次日再删。MySQL 没有 `RENAME DATABASE`，靠**跨库 `RENAME TABLE`** 一句话实现"移库+改名"（元数据操作、秒级） |
| 3 改本地 churn 代码 | ❌ **最大硬伤**：聊天记录说"表名只在 5 个 `__tablename__`、改 5 行就完事"——**经核实是错的**。`mysql_repo.py` 里还有 **~12 处裸 SQL 字符串硬编码表名**：`FROM warning_jobs`(L923)、`FROM detection_runs`(L1613/1678)，以及传给 `_read_text_column()` 的 `"warning_records"`/`"warning_jobs"`/`"detection_runs"` 字面量。只改 `__tablename__`，这些 MySQL 读路径会**运行时报"表不存在"**。漏改清单还含 `.env.example`、`db.py:22` 兜底串、测试断言 |
| 4 查 view/触发器/事件 | ⚠️ **顺序错了**：这是"动手前的体检"，要放在**重命名之前**，不是之后。改完才查没意义 |
| 5 新建拓客表 | ✅ 对；要明确：DDL 进 churn-engine 的 `017` 迁移（它现在是 schema owner），别散落；表/列带注释 |
| 6 服务器 churn 改动 | ⚠️ 不完整：要和第 2 步**在同一个停机窗口**里做完（新代码查新表名、旧库还是旧表名 → 两边不同步就报错）。含：停 api+celery → 跑 SQL → git pull → 手改服务器 .env 库名 → `up -d --build` → 验证 |
| 7 测试 | ⚠️ 太泛：必须专门测**裸 SQL 读路径**（get_job / get_detection_run / list_detection_runs / 大文本分块读），因为 ORM happy path 即使漏改也不报错，最易漏的恰是这些 |

**原 plan 完全没提、但必须补的**：
- **拓客项目侧也有引用**：`inspect_churn_db.py:84,87` 硬编码 `table_schema='churn_db'`、`.env`/`.env.example` 的 `LOG_DB_NAME=churn_db`、README 文案
- **索引名**：`RENAME TABLE` 不改索引名（已与你确认：一并规范）
- **回滚预案**：没有
- **仓库外影响面**：BI/Navicat/Grafana/cron mysqldump/监控 SQL 直连 `churn_db.*` 或旧表名，git 管不到
- **命名规范文档**：目标是"为后续开发/交接做准备"，但规范现在只活在聊天记录里——要落成 repo 内文档

### Codex 对抗式复核结论（已并入下方 plan）
- **确认正确**：裸 SQL 表名（L923/1613/1678 的 `FROM` + `_read_text_column` 字面量）、跨库 `RENAME TABLE` + `RENAME INDEX` 语法、"不要改的 Python 标识符"清单、库内无 FK/自增/生成列/触发器/事件、`churn_ai_analyses` 仅移库不改名。
- **已修正**：① ORM 索引行去掉 `201`（那是不改名表 `churn_ai_analyses` 的索引）；② 回滚 SQL 的通配符写法**不可执行** → 改成显式表对；③ 切换补全必须的 `docker compose up -d --build`（`COPY . .` 把代码打进镜像、`env_file` 仅启动时读，光 pull+改 .env 不生效），另给"窗口外先 build"压缩停机的可选优化；④ `inspect_churn_db.py` 改**全文** churn_db（不止 :84/:87）；⑤ `DROP` 前加 `churn_db` 空表校验；⑥ 补**非 root 授权**检查。
- **保留判断**：000-015 前向式不重写（已在迁移策略里写明新装顺序与 migration-guide 追加要求）。

---

## 二、我的 plan（改进版）

**范围（按你的选择）**：只规范命名（**不接**拓客写入链路）+ 索引名**一并**规范。
**执行分工**（原则：仓库内/本地文件改动**我**来；一切**写生产库 + 重部署**由**你**执行，沿用"生产数据我不擅自动"；只读体检/验证我可在你授权下代跑并回报）：

| 阶段 | 我（Claude）做 | 你做 |
|---|---|---|
| **A** 仓库内编辑（churn-engine） | 改全部代码/测试/文档/迁移：`mysql_repo.py` 那 ~20 处、`db.py`、`.env.example`、测试断言、`docs/`、新建 `016`/`017`/`naming-conventions.md`；本地跑 pytest 确认绿 | review diff；`git push` 到 GitLab |
| **A2** 拓客项目编辑（lead-gen） | 改 `.env`/`.env.example`/`inspect_churn_db.py`/`README`（本地文件，非 git 仓库） | review（非 git，无需 push） |
| **B** 窗口前体检（连生产 DB，只读） | 给出全部体检 SQL；**经你授权**可在本地直接跑只读 `SHOW`/`SELECT` 并回报 | 默认由你跑这些只读查询；按结果定要不要补 view/触发器/授权处理 |
| **C** 停机窗口切换 | 给出可直接粘贴的：重命名 SQL、`.env` 改法、docker 命令清单 | 执行：停容器 → 备份 → 跑改库 SQL → `git pull`+改服务器 `.env` → `up -d --build` → 留空 `churn_db` |
| **D** 验证 | 给出验证 SQL + curl 清单；**API/DB 可达且你授权**时可代跑只读验证并回报 | 在生产确认三条裸 SQL 路径 + 写路径 + 日志无 `doesn't exist` |
| **E** 回滚（仅失败时） | 给出反向 `RENAME` / dump 还原的精确命令 | 真要回滚时由你执行（生产写操作） |
| **F** 仓库外审计 | 给出 grep 命令 + 排查清单 | 在你各台机器/BI/cron/Grafana 自查并修 |
| 次日 **DROP** | 给出 DROP 前的空表校验 SQL | 确认无碍后执行 `DROP DATABASE churn_db` |

**逐阶段闸门（串行，不可跳）**：每阶段末尾有 **✅ 闸门**，列"退出条件 + 双方确认"。**退出条件全满足、且你我都确认各自任务完成，才进入下一阶段**；任一条不满足就停在本阶段修复，不带未决项前进。其中 **B→C 是硬闸门**（C 脚本要靠 B 实测结果定稿）、**D 不绿不退出窗口**、**E 是失败分支不算前进**。每过一个闸门，我会停下来跟你逐条对一遍"我的 ✓ / 你的 ✓"再继续。

### 阶段 A — 仓库内编辑（churn-engine；可提前做、CI 绿、合并，先不部署）

**`app/churn/mysql_repo.py`（核心，同一文件多处）**：
- `61/76/138/163/177` `__tablename__` → 加 `churn_` 前缀（`199`=`churn_ai_analyses` **不动**）
- `923` `FROM warning_jobs`、`1613/1678` `FROM detection_runs` 裸 SQL → 新名
- `562/575/718/726/734/742/750` 传入的 `"warning_records"` → `"churn_warning_records"`
- `931/939` `"warning_jobs"`、`1621/1629` `"detection_runs"` → 新名
- `79/85/94/102/111/143/180` ORM `Index(...)`/`UniqueConstraint(...)` **名字符串** → 加前缀（与下面索引 RENAME 同步）。⚠️ **不含 `201`**——那是不改名的 `churn_ai_analyses` 的 `ix_churn_ai_analysis_cache`（已核实 L198-202），保持原样

**其它文件**：
- `app/shared/db.py:22` 兜底默认 `/churn_warning` → `/amer_sales_assist`（注意这是第三个不一致的名字）
- `.env.example:8` `/churn_db?` → `/amer_sales_assist?`（`.env` 被 gitignore：本地另改，服务器那份在窗口里手改）
- `tests/churn/test_mysql_repo.py:211,970-971,1111-1112` SQL 字面量断言 → 新名
- `docs/` 四处（technical-design / migration-guide / runbook / api-guide）表名/库名/示例 SQL → 新名（runbook 里有可直接跑的 `FROM warning_records` 查询）
- **新增** `migrations/mysql/churn/016_rename_churn_tables.sql`：in-place `RENAME TABLE` + `RENAME INDEX`（供新装/留痕）
- **新增** `migrations/mysql/churn/017_create_lead_tables.sql`：`lead_skip`/`lead_api_log` 带列注释 DDL（采用聊天记录 Turn 23 那版 DDL）
- **迁移策略（前向式，刻意不重写 000-015）**：保留 000-015 生产历史不动；新装顺序 `000→…→015`（建旧名）`→016`（改新名+索引）`→017`（建 lead 表），最终落到新名。**必须把 016/017 追加进 `migration-guide.md` 的执行清单**，否则新装会停在旧名
- **新增** `docs/naming-conventions.md`：`{模块}_{实体}` 约定、`churn_`/`lead_` 前缀、`ix_{表}_{列}`/`uq_` 索引规范、每表每列带 `COMMENT`、"churn-engine 拥有 amer_sales_assist schema，所有 DDL 走编号迁移"

**绝不要改**（裸眼 grep 的假阳性，是 Python 标识符不是 SQL，改了会破坏代码）：`memory_repo.py`/`repository.py`/`routes.py`/`service.py` 里的 `list_detection_runs`/`self.detection_runs`/`list_warning_configs` 等方法/属性名。逐文件校验：`grep -nE '\b(FROM|JOIN|INTO|UPDATE|TABLE)\s+(warning_|detection_runs)' <file>`，无命中即纯标识符。

**✅ 闸门 A**（关代码工作流）：退出条件 = churn-engine 全部编辑完成 + 本地 `pytest` 全绿 + `grep -rnE 'churn_db|warning_records|warning_jobs|detection_runs|warning_configs|warning_global_configs' app/ tests/` 仅剩应保留项（`churn_ai_analyses`、Python 标识符、016/017 内的新旧名映射）。确认 = 我交 diff + pytest 结果 → 你 review 通过 → 你 `git push`、**CI 绿**。这条线闭合是进入 C 的前置之一。

### 阶段 A2 — 拓客项目编辑（smart_lead_generation；死配置 housekeeping，低风险）
- `.env` / `.env.example`：`LOG_DB_NAME=churn_db` 值 + 上方注释行一并改（`.env:12/17`、`.env.example:13/18`）→ `amer_sales_assist`
- `scripts/inspect_churn_db.py`：**全文** churn_db 都改——功能性 `table_schema='churn_db'`（:84/:87）+ 文案（docstring :2、label :82/:85）。校验 `grep -n churn_db scripts/inspect_churn_db.py`，否则它去查不存在的 schema、返回空
- `README.md:4` "日志库 churn_db" → 新名
- 业务库 `amer_crm_v2`（A 机）配置**保持不动**——明确写出来，防止误改

**✅ 闸门 A2**：退出条件 = lead-gen 四处改完 + `grep -n churn_db scripts/inspect_churn_db.py` 仅余应留文案。确认 = 我交改动 → 你 review（非 git，无需 push）。（A2 与 A 相互独立，可并行）

### 阶段 B — 窗口前体检（B 机，无停机，提前几天做）
这是你原 plan 的第 4 步，**提前到重命名之前**。连 B 机查：
- VIEW：`SELECT TABLE_NAME FROM information_schema.VIEWS WHERE TABLE_SCHEMA='churn_db';`
- TRIGGER：`SHOW TRIGGERS IN churn_db;`（⚠️ 有触发器时**跨库 `RENAME TABLE` 会直接失败**，必须先 DROP、迁完在新库重建）
- EVENT：`SELECT EVENT_NAME FROM information_schema.EVENTS WHERE EVENT_SCHEMA='churn_db';`
- 外键：`information_schema.KEY_COLUMN_USAGE ... REFERENCED_TABLE_SCHEMA='churn_db'`
- **真实索引名 + COLLATE + 行数**：`SHOW INDEX FROM churn_db.warning_records;`（5 张表都查，用来补全 `RENAME INDEX` 的确切名字）+ `SELECT DEFAULT_COLLATION_NAME FROM information_schema.SCHEMATA WHERE SCHEMA_NAME='churn_db';`
- **非 root 授权**：`SELECT user,host,db FROM mysql.db WHERE Db='churn_db';`——churn 与拓客都连 root（全局权限、不受影响），但若有 BI/只读用户被 `GRANT ... ON churn_db.*`，改名后会**静默失效**，需补 `GRANT ... ON amer_sales_assist.*`

有 view/触发器/事件/FK 引用旧表 → 在窗口脚本里补对应处理。repo 里没定义这些，但库里可能有人手搓过。

**✅ 闸门 B（硬闸门）**：退出条件 = 体检 SQL 全跑完，并明确——有无 view/触发器/事件/FK 引用旧表、**确切索引名**、**COLLATE**、各表**行数基线**（记下 `warning_records` ~11k 供 D 比对）、非 root 授权情况。确认 = 你（或授权我只读）跑完 → 结果回贴 → 我据此把窗口脚本的 `RENAME INDEX`/`COLLATE`/触发器处理补成**最终可执行版** → 双方确认脚本定稿。**脚本没据实测定稿，绝不进 C。**

**🔵 闸门 B 实测结果（2026-06-05，只读）**：**无** view/触发器/事件/外键；**无**非 root 授权（`mysql.db` 空）；各表 collation 均 `utf8mb4_0900_ai_ci`（新库按此建）；行数基线 `warning_records=14092`、`warning_jobs=25`、`warning_configs=6`、`warning_global_configs=3`、`detection_runs=45`、`churn_ai_analyses=0`；**15 个待重命名索引名与 016/cutover 脚本逐一吻合，无需改**。定稿一次性脚本：`/tmp/cutover_churn_db_to_amer_sales_assist.sql`（跨库 RENAME + 索引 + lead 表，collation 已填 `utf8mb4_0900_ai_ci`）。

### 阶段 C — 停机窗口切换（你选了"可以先停服务"）

**进入前置**：闸门 A（代码已 push、CI 绿）+ 闸门 B（脚本据实测定稿）均已通过，方可进窗口。

服务器侧三件事一个都不能少：`git pull` + 改 `.env` + **`docker compose up -d --build`**。`--build` 是原来"光 pull+改 .env"漏掉的关键——`Dockerfile:9` 是 `COPY . .`（代码打进镜像）、compose 里 `api`/`celery` 都 `build: .` 且没挂源码卷、`.env` 经 `env_file` 仅容器创建时读，所以不重建就还是旧代码连旧库。按序：

1. **停写库容器**（先停 celery beat，它会定时写检测）：`docker compose stop celery api` → `ps` 确认 Exited
2. **备份**：`mysqldump -h 8.138.104.14 -P 23356 -uroot -p --single-transaction --routines --triggers --events --databases churn_db > churn_db_<ts>.sql`，确认文件非空、tail 有 `Dump completed`
3. **改库**（本地 mysql 客户端连那台远程库即可，不必上服务器）：跑一次性重命名 SQL（见下）。**非原子提醒**：`RENAME TABLE`（6 张一句话）原子，但其后 `RENAME INDEX`、`CREATE lead 表`是独立语句——某条失败时数据已安全移好，修正后**重跑那一条**即可（可重入），不影响已迁数据
4. **服务器** `git pull`（新代码 + 016/017）+ 手改服务器 `.env` 第 7 行 `/churn_db?` → `/amer_sales_assist?`
5. **`docker compose up -d --build`** 重建（这步才让新代码 + 新 .env 生效；`create_all` 只建缺失表，churn 表已存在 → no-op）
6. **验证**（阶段 D）。绿了退出窗口
7. **空的 churn_db 先留着**当回滚后路；次日确认 `SHOW TABLES IN churn_db;` **返回空** 且新库 6 张表行数都对，再 `DROP DATABASE churn_db`

> 可选优化：想把停机从"build 的几分钟"压到几秒，就把 `git pull && docker compose build`（只 build 不 up）提到第 1 步**之前**做——旧容器仍用旧镜像连旧库、一切一致；窗口内第 5 步去掉 `--build`、只 `docker compose up -d`。

**一次性重命名 SQL（B 机，root）**：
```sql
CREATE DATABASE IF NOT EXISTS amer_sales_assist CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;  -- 实测: churn_db 各表均为此 collation

-- 跨库 RENAME = 一句话同时"移库 + 改名"，元数据操作、秒级、原子
RENAME TABLE
  churn_db.warning_records        TO amer_sales_assist.churn_warning_records,
  churn_db.warning_configs        TO amer_sales_assist.churn_warning_configs,
  churn_db.warning_global_configs TO amer_sales_assist.churn_warning_global_configs,
  churn_db.warning_jobs           TO amer_sales_assist.churn_warning_jobs,
  churn_db.detection_runs         TO amer_sales_assist.churn_detection_runs,
  churn_db.churn_ai_analyses      TO amer_sales_assist.churn_ai_analyses;  -- 移库不改名

-- 索引/唯一键改前缀（RENAME TABLE 不会自动改），确切名字以阶段B SHOW INDEX 实测为准：
ALTER TABLE amer_sales_assist.churn_warning_records
  RENAME INDEX ix_warning_records_customer_id TO ix_churn_warning_records_customer_id,
  /* …其余 ix_warning_records_* 按实测逐条补… */
  RENAME INDEX uq_warning_active_dedup TO uq_churn_warning_active_dedup;
ALTER TABLE amer_sales_assist.churn_warning_configs
  RENAME INDEX uq_warning_configs_customer_anomaly TO uq_churn_warning_configs_customer_anomaly;
ALTER TABLE amer_sales_assist.churn_detection_runs
  RENAME INDEX ix_detection_runs_job_id TO ix_churn_detection_runs_job_id /* …按实测补… */;

-- 建拓客表（017 内容，带列注释）：
CREATE TABLE amer_sales_assist.lead_skip ( /* …Turn 23 的 DDL，列带 COMMENT… */ );
CREATE TABLE amer_sales_assist.lead_api_log ( /* …Turn 23 的 DDL，列带 COMMENT… */ );

-- DROP DATABASE churn_db;  ← 本窗口先别跑，留作回滚后路
```

**✅ 闸门 C**：退出条件 = 备份文件完整（非空、有 `Dump completed`）+ 改库 SQL 成功（新库 6 张 churn 表 + 2 张 lead 表 + 索引已改名）+ 服务器 `.env` 已改 + `up -d --build` 后容器 Up + `/healthz` 通过。确认 = 你逐步执行、关键节点回报（备份 ✓ / 改库 ✓ / 重建+健康 ✓）。健康没过就停在 C 修，不做 D 的功能验证。

### 阶段 D — 验证（专打裸 SQL 路径，别只测 ORM happy path）
- **schema**：`SHOW TABLES` 应见 6 张 `churn_*` + `churn_ai_analyses` + `lead_skip` + `lead_api_log`；`SELECT COUNT(*) FROM churn_warning_records` 对上 **14092**（闸门 B 实测基线）；`SHOW INDEX` 见 `ix_churn_*`
- **健康**：`curl /healthz`
- **detection_runs 裸路径**：调 list-detection-runs 接口（500 = 漏改 detection_runs 字面量）
- **warning_jobs + 大文本裸路径**：拉一条预警**详情**（触发 `get_job`@923 + 5 处 `_read_text_column` 分块读 `churn_warning_records` 的 MEDIUMTEXT），确认大字段完整返回
- **预警列表**（ORM 路径）作基线对照
- **写路径**：触发一次检测 → `SELECT run_id,status,started_at FROM churn_detection_runs ORDER BY started_at DESC LIMIT 3` 见新行
- **日志**：`docker compose logs --since 5m api celery | grep -iE "doesn't exist|unknown table"` 应为空

**✅ 闸门 D**：退出条件 = 三条裸 SQL 路径（detection-run 列表 / 预警详情大文本 / 触发检测写入）全 200 且有真实数据 + 新库行数与 B 基线一致 + 日志无 `doesn't exist`。确认 = 验证全绿、双方确认 → **才退出停机窗口**。任一红 → 转阶段 E 回滚或窗口内修复，**不前进**。

### 阶段 E — 回滚
- 重命名后验证失败（代码漏改）→ **反向 RENAME**（数据没动过，秒级）。⚠️ `RENAME TABLE` **不支持通配符**，必须写全显式表对：
  ```sql
  RENAME TABLE
    amer_sales_assist.churn_warning_records        TO churn_db.warning_records,
    amer_sales_assist.churn_warning_configs        TO churn_db.warning_configs,
    amer_sales_assist.churn_warning_global_configs TO churn_db.warning_global_configs,
    amer_sales_assist.churn_warning_jobs           TO churn_db.warning_jobs,
    amer_sales_assist.churn_detection_runs         TO churn_db.detection_runs,
    amer_sales_assist.churn_ai_analyses            TO churn_db.churn_ai_analyses;
  ```
  之后 `git checkout 旧sha` + 还原服务器 .env 的 `/churn_db?` + `docker compose up -d`。前提：空 churn_db 没删。（索引名仍是 `ix_churn_*`，功能无碍——没有代码按名字引用索引；要彻底干净就走下面 dump 还原。）
- 数据异常/部分失败 → 从 dump 还原：`DROP DATABASE amer_sales_assist; mysql < churn_db_<ts>.sql`，代码/.env 同步回滚。
- 铁律：**DB 与 代码+.env 必须锁步回滚**。
- 阶段 E 是**失败分支**、不是前进步骤：回滚后回到对应阶段重做，不带病进 F。

### 阶段 F — 仓库外影响面（你自己排查，git 管不到）
每台可能放脚本的机器跑：
```bash
grep -rniE 'churn_db|\b(warning_records|warning_jobs|detection_runs|warning_configs|warning_global_configs)\b' \
  ~/ /etc/cron* /opt /srv 2>/dev/null | grep -v churn-warning-engine
```
重点：BI/Navicat 保存的查询、Grafana SQL 面板、cron 里的 `mysqldump churn_db`、监控告警 SQL、别的服务 `.env` 里的 churn_db DSN。

**✅ 闸门 F + 收尾**：退出条件 = 仓库外 grep/BI/cron/Grafana 排查完成、残留项已改或确认无 + 生产观察 ≥1 天无异常 + `SHOW TABLES IN churn_db;` 为空 + 新库行数稳定。确认 = 你确认外部无残留 + 观察期无异常 → 执行 `DROP DATABASE churn_db`，迁移收尾。

---

## 验证（end-to-end 小结）
关键判据：6 张改名表 + 2 张 lead 表就位、`churn_warning_records` 行数对上 **14092**（闸门 B 实测）；**三条裸 SQL 路径**（detection-run 列表 / 预警详情大文本 / 触发检测写入）均返回 200 且有真实数据；容器日志无 `doesn't exist`。本地侧另跑 churn-engine 的 pytest（断言已改新名）应全绿。

---

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21)：ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4)：chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent Explore：已用于两个仓库的引用全扫（blast radius）；执行期若冒出新引用可再扫
- Agent Plan：已用于设计迁移/切换 runbook
- Skill code-review：改完后审一遍多文件 rename diff，专查漏改的裸 SQL 表名字面量
- Skill verify：本地起 churn-engine 跑 pytest + 冒烟读路径，确认改名没破坏

**考虑过但不用**：
- Skill simplify：与 code-review 二选一——本次重点是"有没有漏改/改错"（正确性）不是简化，选 code-review
- Skill run：与 verify 二选一——要"确认改动有没有破坏"用 verify 更贴，run 偏"看效果演示"
- MCP chrome-devtools / firecrawl / jina / tavily：全是浏览器/网页/检索类，本任务是本地 DB+代码改名，均不涉及
