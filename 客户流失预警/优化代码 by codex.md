# 优化代码 by codex

## 说明
以下内容为本次会话的用户可见记录整理版，按时间顺序保存，包含用户请求、助手答复、关键修改和验证结果。不包含系统提示词或内部工具原始日志。

## 会话记录

### 1. 生成仓库贡献指南
**用户**

> Generate a file named AGENTS.md that serves as a contributor guide for this repository.
> Your goal is to produce a clear, concise, and well-structured document with descriptive headings and actionable explanations for each section.
> Follow the outline below, but adapt as needed — add sections if relevant, and omit those that do not apply to this project.
>
> Document Requirements
>
> - Title the document "Repository Guidelines".
> - Use Markdown headings (#, ##, etc.) for structure.
> - Keep the document concise. 200-400 words is optimal.
> - Keep explanations short, direct, and specific to this repository.
> - Provide examples where helpful (commands, directory paths, naming patterns).
> - Maintain a professional, instructional tone.
>
> Recommended Sections
>
> Project Structure & Module Organization
>
> - Outline the project structure, including where the source code, tests, and assets are located.
>
> Build, Test, and Development Commands
>
> - List key commands for building, testing, and running locally (e.g., npm test, make build).
> - Briefly explain what each command does.
>
> Coding Style & Naming Conventions
>
> - Specify indentation rules, language-specific style preferences, and naming patterns.
> - Include any formatting or linting tools used.
>
> Testing Guidelines
>
> - Identify testing frameworks and coverage requirements.
> - State test naming conventions and how to run tests.
>
> Commit & Pull Request Guidelines
>
> - Summarize commit message conventions found in the project’s Git history.
> - Outline pull request requirements (descriptions, linked issues, screenshots, etc.).
>
> (Optional) Add other sections if relevant, such as Security & Configuration Tips, Architecture Overview, or Agent-Specific Instructions.

**助手过程**

- 检查仓库结构，确认这是一个 Python 项目，核心目录为 `app/` 和 `tests/`。
- 读取 `pyproject.toml`、`app/main.py`、`app/config.py`、`app/tasks.py`、`tests/test_engine.py`。
- 发现工作区没有 `.git` 元数据，因此无法从本地 Git 历史验证提交规范。
- 运行测试，结果为 `29 passed in 0.17s`。

**助手结果**

创建了 `AGENTS.md`，标题为 `Repository Guidelines`，内容覆盖：

- 项目结构与模块职责
- 构建、测试与开发命令
- 编码风格与命名约定
- 测试指南
- 提交与 PR 规范
- 安全与配置提示

并说明：

- 文档依据仓库实际结构和配置生成
- 本地缺少 `.git` 历史，因此提交规范只能给出通用建议
- 已验证 `pytest -q` 可通过

### 2. 将贡献指南翻译成中文
**用户**

> 翻译成中文

**助手结果**

将 `AGENTS.md` 翻译为中文版本，保留了原有结构以及仓库里的实际命令、模块划分和配置项。并额外说明：

> 目前标题仍是 `Repository Guidelines`，如果也想把标题改成中文，例如“仓库指南”，可以继续修改。

### 3. 询问如何快速阅读并 review AI 生成的项目
**用户**

> 这个项目是AI生成的代码，请问我该怎么快速阅读并review这个项目代码

**助手结果**

给出了一套面向本仓库的快速阅读路径和 review 方法：

1. 先看入口和对外契约：`pyproject.toml`、`app/main.py`、`app/schemas.py`
2. 再看编排层：`app/service.py`
3. 再看数据层：`app/repository.py` 和 `app/db.py`
4. 最后看算法层：`app/engine.py`
5. 读完再回看测试：`tests/test_engine.py`

并给出一个 30 分钟 review 流程：

- 5 分钟跑 `pytest -q`
- 5 分钟跑 `uvicorn app.main:app --reload`
- 10 分钟顺着请求链读 `main.py -> service.py -> repository/db.py -> engine.py`
- 10 分钟只盯边界条件、状态流转、性能、测试缺口

同时指出了优先怀疑的区域：

- `app/main.py` 的全局单例初始化
- `app/service.py` 逐客户取订单的潜在 N+1
- `app/db.py` 的查询放大
- `app/engine.py` 的阈值和业务假设
- `tests/test_engine.py` 缺 API/MySQL/Celery 路径测试

### 4. 正式代码 review
**用户**

> 我要

**助手结果**

按严重程度给出了正式 review 结论。

#### Findings

**High**

1. 月度量级任务的调度时间和算法口径不一致，会在每月 1 日系统性地产生“下降”误报。
   - `app/tasks.py` 把 `volume` 检测固定在每月 1 日 08:00 运行
   - `app/engine.py` 比较的是“本月 1 日到 biz_date”对比“上月整月”
   - 月初时 `current` 往往为 0 或极小值，容易触发 `down`
   - 本地最小复现得到 `change_rate=-1.0`

2. 已 enrich 的预警在重新跑检测时会被静默覆盖。
   - `app/repository.py` 和 `app/db.py` 把 `enriched` 也视为 active warning
   - `app/service.py` 重跑时会刷新旧记录
   - 本地复现结果是 `status` 仍为 `enriched`，但人工填写的 `reason` 和 `sales_message` 被系统文案覆盖

**Medium**

3. 去重逻辑在并发下不安全。
   - `app/service.py` 采用“先查再写”的应用层去重
   - `app/db.py` 没有数据库级唯一约束
   - 并发重试或多个 worker 重叠执行时可能双写

4. MySQL 路径查询放大明显，存在性能风险。
   - `app/service.py` 对每个客户单独拉订单
   - `app/db.py` 的 `customer_count_for_sales_ids()` 先全查再 `len()`
   - `find_active_warning()` 先查 ID 再逐条 `get_warning()`

#### Open Questions

- `enriched` 是否应视为人工确认后的最终内容
- 月度量级检测到底要比“完整月 vs 完整月”，还是“月初到当天 vs 上月同期”
- 是否需要保留预警历史，而不是长期折叠为同一条记录

#### Summary

- 现有 `pytest` 全通过，但没有覆盖月初 volume 调度、enrich 后重跑、并发去重和 MySQL 路径性能这几个高风险场景
- 建议先修前两个高优问题，再补回归测试

### 5. 推进第一轮修复
**用户**

> 帮我继续推进吧

**助手过程**

- 新增 `app/job_dates.py`
- 修改 `app/tasks.py`
- 修改 `app/service.py`
- 新增 `tests/test_job_dates.py`
- 新增 `tests/test_service.py`
- 跑全量测试

**助手修复内容**

1. 修复月初 volume 误报
   - 在 `app/job_dates.py` 中新增 `resolve_run_detection_date()`
   - 对 `volume` 定时任务，在未显式传入 `biz_date` 时，默认使用“上个月最后一天”
   - `app/tasks.py` 中的 `run_detection()` 改为调用这个日期解析逻辑

2. 修复 enrich 内容被覆盖
   - `app/service.py` 中，重复检测时只刷新 `pending_enrich`
   - 如果记录已是 `enriched`，只复用原 `warning_id`
   - 不再覆盖人工填写的 `reason`、`sales_message`、推荐参数和严重级别

3. 新增回归测试
   - `tests/test_job_dates.py` 验证 `volume` 默认取上月月末
   - `tests/test_service.py` 验证 enrich 后重跑不覆盖人工内容

**助手结果**

> 已推进并落地了两个高优修复。
>
> 在 `app/service.py` 里，重复检测现在只会刷新 `pending_enrich` 的预警；如果预警已经被人工 enrich，就只复用原 `warning_id`，不再覆盖人工填写的 `reason`、`sales_message` 和推荐参数。
>
> 针对月度量级误报，新增了 `app/job_dates.py`，并让 `app/tasks.py` 在未显式传入 `biz_date` 时，将 `volume` 任务默认落到“上个月最后一天”。
>
> 补了两组回归测试，验证结果为 `33 passed`。

### 6. 继续推进中优问题
**用户**

> 你继续推进吧

**助手过程**

- 开始处理 MySQL 路径的查询放大
- 开始处理并发下去重脆弱的问题
- 修改了 `app/repository.py`，先给仓储接口增加能力

**用户**

> 继续

### 7. 推进第二轮修复
**助手过程**

- 修改 `app/service.py`
- 修改 `app/db.py`
- 更新 `tests/test_service.py`
- 跑全量测试

**助手修复内容**

1. 批量取单，减少查询放大
   - 在 `app/repository.py` 中新增 `get_orders_for_customers()`
   - `InMemoryRepository` 实现了批量取单
   - `app/service.py` 的 `detect_batch()` 先一次性加载整批客户订单，再按客户分发给检测引擎
   - `app/db.py` 实现了 MySQL 的批量取单逻辑，按 customer_id 分组返回

2. 优化计数与预警读取
   - `app/db.py` 中 `customer_count_for_sales_ids()` 改成 SQL `COUNT(*)`
   - `get_warning()` 改成直接 SQL 查询并反序列化
   - `find_active_warning()` 改为单次查询返回完整字段，不再二次 `get_warning()`
   - `update_warning()` 同步写回 `metadata`

3. 增加并发去重锁
   - 在 `app/repository.py` 中增加 `warning_dedup_lock()` 钩子
   - `app/service.py` 在去重和保存区间使用仓储级锁
   - `app/db.py` 中用 MySQL `GET_LOCK` / `RELEASE_LOCK` 实现细粒度去重锁
   - 锁粒度为“客户 + 异常类型 + 产品/方向”

4. 新增测试
   - `tests/test_service.py` 新增批量取单路径测试
   - 验证 `detect_batch()` 使用批量加载，而不是逐客户调用 `get_orders_for_customer()`

**助手结果**

> 继续推进了两块中优问题。
>
> 在 `app/service.py` 和 `app/repository.py` 里，把批量检测改成先一次性加载整批客户的订单，再按客户分发给检测引擎，不再逐客户调用 `get_orders_for_customer()`。
>
> 在 `app/db.py` 里，现在支持 MySQL 批量取单、SQL 侧 `COUNT(*)` 统计客户数，以及单次查询反序列化 active warning，避免原来那种按客户和按 warning 的查询放大。
>
> 并发去重这块，在 `app/service.py` 的去重/保存区间加了仓储级锁入口，在 `app/db.py` 里用 MySQL `GET_LOCK`/`RELEASE_LOCK` 做了按“客户 + 异常类型 + 产品/方向”的细粒度去重锁。
>
> 补了新的 service 回归测试，确认 `detect_batch()` 走的是批量取单路径。当前本地测试结果是 `34 passed`。
>
> 还没做的是 MySQL 集成级验证。当前测试套件没有起真实 MySQL，所以 `app/db.py` 的锁和批量 SQL 做了代码级收敛，但还没在实际数据库环境里跑过。

### 8. 保存本次会话
**用户**

> 把这次会话的所有内容存入E:\note\客户流失预警\优化代码 by codex.md

**助手结果**

开始将本次会话整理并保存到：

- Windows 路径：`E:\note\客户流失预警\优化代码 by codex.md`
- WSL 路径：`/mnt/e/note/客户流失预警/优化代码 by codex.md`

## 本次会话落地产物

### 文档
- `AGENTS.md`
- `E:\note\客户流失预警\优化代码 by codex.md`

### 新增文件
- `app/job_dates.py`
- `tests/test_job_dates.py`
- `tests/test_service.py`

### 修改文件
- `AGENTS.md`
- `app/tasks.py`
- `app/service.py`
- `app/repository.py`
- `app/db.py`

## 验证结果

- 第一轮修复后：`33 passed`
- 第二轮修复后：`34 passed`

## 后续建议

1. 增加真实 MySQL 集成测试，验证 `GET_LOCK`、批量 SQL 和事务边界。
2. 视业务决定是否需要 `closed` 状态或预警历史表，避免所有同类预警长期折叠为一条记录。
3. 为 FastAPI 层补接口测试，覆盖 `start -> batches -> detect -> enrich -> get` 的完整闭环。
