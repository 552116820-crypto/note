# Git 和 Bugbot 会话记录

日期：2026-04-10

项目路径：`E:\churn-warning-engine`

WSL 路径：`/mnt/e/churn-warning-engine`

目标笔记：`E:\note\客户流失预警\git 和 Bugbot.md`

---

## 1. 本次代码审查请求

用户要求：

> 再次审查这个项目代码

审查范围：

- `app/engine.py`
- `app/service.py`
- `app/repository.py`
- `app/db.py`
- `app/main.py`
- `app/schemas.py`
- `app/config.py`
- `app/tasks.py`
- `app/warning_keys.py`
- `migrations/mysql/001_warning_records_active_dedup.sql`
- `tests/`
- `.gitignore`
- `explain.md`

运行过的验证命令：

```bash
pytest -q
```

结果：

```text
56 passed
```

---

## 2. 第一次 review 发现的问题

### 2.1 MySQL 迁移脚本对历史活跃预警不安全

位置：

```text
migrations/mysql/001_warning_records_active_dedup.sql
```

问题：

旧迁移脚本先给所有历史数据加：

```sql
dedup_key = ''
active_slot = '1'
```

然后创建唯一约束：

```sql
UNIQUE (customer_id, anomaly_type, dedup_key, active_slot)
```

如果历史表中同一个客户、同一种异常类型下存在多条活跃预警，例如不同关键产品的 `product` 预警，它们都会变成同一个唯一键：

```text
customer_id + product + '' + '1'
```

这样迁移会失败。

建议：

- 建唯一约束前先回填真实 `dedup_key`
- `frequency` 使用 `frequency`
- `volume` 使用 `volume:<direction>`
- `product` 使用 `product:<product_id>`
- 对历史重复活跃记录做合并或关闭

### 2.2 `.gitignore` 没有正确忽略 Python 缓存

原问题：

`.gitignore` 里规则没有覆盖真实目录，`__pycache__` 仍然被 Git 跟踪。

现象：

运行 `pytest` 后，很多 `.pyc` 文件出现在 modified 状态。

应该忽略：

```gitignore
.pytest_cache/
__pycache__/
*.py[cod]
```

### 2.3 `explain.md` 有整文件换行和尾随空白 churn

现象：

```bash
git diff --check
```

报大量 trailing whitespace。

`git diff --numstat explain.md` 显示：

```text
1015  1015  explain.md
```

说明主要是 CRLF / LF 和尾随空白导致的整文件 diff。

---

## 3. 已执行的修复

用户要求：

> 帮我把这些问题解决了

### 3.1 修复 MySQL 迁移脚本

修改文件：

```text
migrations/mysql/001_warning_records_active_dedup.sql
```

新增逻辑：

1. 创建临时表 `warning_records_dedup_backfill`
2. 从历史 `feature_summary` 里解析：
   - `product_id`
   - `direction`
3. 回填 `dedup_key`
4. 对超过 120 字符的 key 做 SHA1 缩短，和 Python 逻辑保持一致
5. 设置 `active_slot`
6. 合并历史重复活跃预警
7. 最后再创建索引和唯一约束

回填规则：

```text
frequency -> frequency
product   -> product:<product_id>
volume    -> volume:<direction>
```

合并历史重复活跃记录的规则：

- 优先保留 `enriched`
- 如果状态优先级相同，保留 `biz_date` 最新的
- 如果日期也相同，保留 `warning_id` 更大的
- 其它重复活跃记录标记为 `resolved`，并把 `active_slot` 设为 `NULL`

注意：

没有实际连接 MySQL 执行迁移，只修正了迁移 SQL 本身。

### 3.2 修复 `.gitignore`

修改文件：

```text
.gitignore
```

最终内容：

```gitignore
/.claude
/.cursor
.pytest_cache/
__pycache__/
*.py[cod]
```

含义：

- `.pytest_cache/`：忽略 pytest 缓存目录
- `__pycache__/`：忽略任意层级 Python 缓存目录
- `*.py[cod]`：忽略 `.pyc`、`.pyo`、`.pyd`

### 3.3 移除已跟踪的 pycache 缓存

执行命令：

```bash
git rm --cached -r app/__pycache__ tests/__pycache__
```

说明：

`--cached` 只从 Git 索引移除文件，不删除本地工作区文件。

结果：

```bash
git ls-files '*__pycache__*'
```

无输出，说明 Git 不再跟踪这些缓存文件。

并且：

```bash
git status --ignored --short app/__pycache__ tests/__pycache__ .pytest_cache
```

会看到：

```text
!! .pytest_cache/
!! app/__pycache__/
!! tests/__pycache__/
```

说明忽略规则已经生效。

### 3.4 规范化 `explain.md`

执行命令：

```bash
perl -0pi -e 's/\r\n/\n/g; s/[ \t]+(?=\n)//g' explain.md
```

作用：

- 把 CRLF 换成 LF
- 移除行尾空白

验证：

```bash
git diff --check
git diff --cached --check
```

均通过。

---

## 4. 修复后的验证

运行：

```bash
pytest -q
```

结果：

```text
56 passed in 0.39s
```

运行：

```bash
git diff --check && git diff --cached --check
```

结果：

无输出，表示通过。

运行：

```bash
git ls-files '*__pycache__*'
```

结果：

无输出，说明缓存文件已经不再被 Git 跟踪。

---

## 5. Staged changes 和 Changes 的区别

用户提问：

> staged changes 和 Changes 有什么不同

解释：

**Staged changes** 是已经放进暂存区的改动。

这些改动会进入下一次：

```bash
git commit
```

**Changes** 通常指工作区里还没暂存的改动。

这些改动不会进入下一次 commit，除非先执行：

```bash
git add
```

流程：

```text
工作区文件
  ↓ git add
Staged changes 暂存区
  ↓ git commit
提交历史
```

常用命令：

```bash
git add .gitignore
```

把文件加入 staged。

```bash
git restore --staged .gitignore
```

把文件从 staged 中移出，但保留工作区修改。

```bash
git diff
```

查看未 staged 的改动。

```bash
git diff --cached
```

查看已经 staged、准备提交的改动。

---

## 6. Cursor Bugbot 是什么

用户提问：

> cursor 的 Bugbot 有什么用

结论：

Cursor 的 Bugbot 可以理解为：

> 面向 GitHub Pull Request 的 AI 代码审查机器人。

主要用途：

1. 自动审查 GitHub PR
2. 在 PR 里对具体代码行评论
3. 找潜在 bug、逻辑漏洞、安全问题、代码质量问题
4. 帮助发现人眼容易漏掉的边界问题
5. 可以根据项目规则定制审查
6. 可以把问题带回 Cursor 或 Background Agent 修复

适合检查的问题：

- MySQL 迁移脚本是否会在历史数据上失败
- dedup key 是否会误合并预警
- API response model 和实际返回字段是否一致
- resolved / pending / enriched 状态流转是否有竞态
- `.gitignore` 是否没有忽略生成文件

它不能替代：

- `pytest`
- CI
- 类型检查
- 真实数据库迁移演练
- 人对业务语义的判断

---

## 7. Bugbot 是不是 Agent Review

用户提问：

> cursor的bugbot是不是agent review

回答：

可以这么理解。

Bugbot 是 Cursor 提供的一个专门做 PR 代码审查的 AI review agent。

但是它和 Cursor 里的普通 Agent 不完全一样。

区别：

| 项目 | Bugbot | Cursor Agent |
|---|---|---|
| 主要用途 | 审查 PR，找 bug | 在编辑器里读代码、改代码、跑命令 |
| 触发位置 | GitHub Pull Request | Cursor IDE / Chat |
| 工作方式 | 看 PR diff，自动评论风险 | 按用户指令执行任务 |
| 输出 | PR review comments | 代码修改、解释、命令结果 |
| 是否主动改代码 | 通常不直接改分支 | 可以直接改本地文件 |
| 最适合 | 第二个 reviewer | 实现、修复、重构、解释代码 |

总结：

```text
Bugbot = 面向 GitHub PR 的自动 AI reviewer
Cursor Agent = 面向当前工作区的 AI 编程助手
```

---

## 8. 怎么使用 Cursor Bugbot

用户提问：

> 怎么用

使用流程：

### 8.1 开启 Bugbot

去 Cursor 的 Bugbot 页面：

```text
https://cursor.com/en-US/bugbot
```

然后登录 Cursor，连接 GitHub。

### 8.2 授权 GitHub 仓库

通常需要安装或授权 Cursor 的 GitHub App，然后选择：

- GitHub organization
- repository
- 是否开启 PR 自动 review

团队仓库可能需要组织管理员授权。

### 8.3 创建 PR

正常开发、提交、push、开 PR：

```bash
git add .
git commit -m "Fix warning dedup migration"
git push origin your-branch
```

然后在 GitHub 上创建 Pull Request。

### 8.4 查看 Bugbot 评论

Bugbot 会像 reviewer 一样在 PR 里评论：

- 哪一行可能有 bug
- 为什么有问题
- 什么情况下会出错
- 建议怎么修

### 8.5 用 Cursor 修复

可以把 Bugbot 评论复制给 Cursor：

```text
根据这个 Bugbot review 修复问题，并补充对应测试：
<粘贴评论>
```

修完后：

```bash
pytest -q
git add -A
git commit -m "Address Bugbot feedback"
git push
```

### 8.6 推荐给本项目的 Bugbot 规则

可以放在 Bugbot / Code Review 设置里，或写入项目规则文件：

```text
Review this Python FastAPI project with focus on:
- Warning deduplication correctness across pending_enrich, enriched, and resolved states.
- MySQL migration safety for existing historical warning_records data.
- Consistency between app.models.WarningRecord and app.schemas.WarningResponse.
- Avoid committing generated files such as __pycache__, *.pyc, and .pytest_cache.
- Volume anomaly jobs must only run on complete month-end biz_date values.
- Changes to app/engine.py, app/service.py, app/db.py, and migrations/ must include focused tests or a clear reason why tests are not possible.
```

---

## 9. GitHub PR 流程是什么

用户提问：

> GitHub PR 流程是什么

PR 是 Pull Request。

含义：

> 我在一个分支上改好了代码，请求把这些改动合并进另一个分支。

典型流程：

### 9.1 从主分支拉最新代码

```bash
git checkout main
git pull origin main
```

如果主分支叫 `master`，把 `main` 换成 `master`。

### 9.2 创建功能分支

```bash
git checkout -b fix-warning-dedup-migration
```

### 9.3 修改代码

例如本次修改：

```text
.gitignore
migrations/mysql/001_warning_records_active_dedup.sql
explain.md
app/__pycache__ 删除记录
tests/__pycache__ 删除记录
```

### 9.4 本地验证

```bash
pytest -q
git diff --check
git status --short
```

### 9.5 暂存改动

```bash
git add .gitignore migrations/mysql/001_warning_records_active_dedup.sql explain.md
git add -u app/__pycache__ tests/__pycache__
```

或者：

```bash
git add -A
```

### 9.6 提交 commit

```bash
git commit -m "Fix warning dedup migration cleanup"
```

### 9.7 推送分支

```bash
git push origin fix-warning-dedup-migration
```

### 9.8 在 GitHub 创建 PR

GitHub 页面一般会出现：

```text
Compare & pull request
```

PR 描述可以写：

```markdown
## Summary
- Backfill warning dedup keys before adding MySQL active dedup constraint
- Ignore Python cache files and remove tracked pycache artifacts
- Normalize explain.md whitespace

## Tests
- pytest -q
- git diff --check
```

### 9.9 等待 CI / Review / Bugbot

CI 会跑测试。

Bugbot 会审查 PR。

团队成员也可以 review。

### 9.10 合并 PR

review 和 CI 都通过后，点击：

```text
Merge pull request
```

常见合并方式：

- Create a merge commit
- Squash and merge
- Rebase and merge

---

## 10. PR 到底是什么意思

用户提问：

> PR到底是什么意思，不是pull requests吗

回答：

是的，PR 就是 Pull Request 的缩写。

更准确的理解：

> 请求把某个分支的代码合并进另一个分支。

例子：

```text
main                         主分支
fix-warning-dedup-migration  修复分支
```

创建 PR：

```text
from: fix-warning-dedup-migration
into: main
```

意思是：

> 请把 `fix-warning-dedup-migration` 这个分支里的改动合并到 `main`。

为什么叫 Pull Request：

站在目标仓库或目标分支的角度，是请求维护者：

> 请从我的分支把代码 pull 进来。

所以叫 Pull Request，不叫 Push Request。

---

## 11. 当前怎么用 Git 把改动提交上去

用户提问：

> 我现在怎么用git把改动提交上去，commit的时间太长了吧，这好像不太正常

检查结果：

```bash
git status --short
```

当前改动已经 staged。

`git status --short` 每行第一列是 `M` 或 `D`，第二列为空，表示这些改动已经在暂存区，会进入下一次 commit。

检查 hooks：

```bash
ls -la .git/hooks
find .git/hooks -maxdepth 1 -type f -perm -111 -printf '%f\n'
git config --get core.hooksPath
```

结果：

- `.git/hooks` 里只有 `.sample` 文件
- 没有实际启用的 Git hook

测试 commit dry-run：

```bash
time git commit --dry-run --short
```

结果约：

```text
real 0m0.099s
```

结论：

commit 本身不应该很慢。

如果执行：

```bash
git commit
```

没有加 `-m`，Git 会打开提交信息编辑器。当前 `GIT_EDITOR` 是：

```text
editor
```

在终端里可能看起来像卡住。

推荐直接用：

```bash
git commit -m "Fix warning dedup migration cleanup"
```

然后推送：

```bash
git push origin main
```

更推荐分支 + PR：

```bash
git checkout -b fix-warning-dedup-migration
git commit -m "Fix warning dedup migration cleanup"
git push origin fix-warning-dedup-migration
```

然后去 GitHub 创建 PR。

如果只想提交修复缓存和迁移脚本，不想带上已有的 `app/*.py` 改动，可以重新整理暂存区：

```bash
git restore --staged .
git add .gitignore migrations/mysql/001_warning_records_active_dedup.sql explain.md
git add -u app/__pycache__ tests/__pycache__
git commit -m "Fix migration and ignore Python cache files"
git push origin main
```

如果在编辑器里卡住：

Vim：

```text
:wq
```

Nano：

```text
Ctrl+O
Enter
Ctrl+X
```

或者按 `Ctrl+C` 退出，重新用：

```bash
git commit -m "Fix warning dedup migration cleanup"
```

---

## 12. 当前工作区状态要点

当前分支：

```text
main...origin/main
```

远程仓库：

```text
origin https://github.com/552116820-crypto/churn-warning-engine.git
```

当前 staged 文件包括：

- `.gitignore`
- `app/db.py`
- `app/engine.py`
- `app/main.py`
- `app/repository.py`
- `app/schemas.py`
- `app/service.py`
- `app/warning_keys.py`
- `explain.md`
- `migrations/mysql/001_warning_records_active_dedup.sql`
- `app/__pycache__/...` 删除记录
- `tests/__pycache__/...` 删除记录

提醒：

如果不想把已有的 `app/*.py` 代码改动一起提交，需要先执行：

```bash
git restore --staged .
```

再只 `git add` 想提交的文件。

---

## 13. 本次建议的最终提交方式

如果要提交全部 staged 改动：

```bash
pytest -q
git diff --check
git diff --cached --check
git commit -m "Fix warning dedup migration cleanup"
git push origin main
```

如果要走 PR 流程：

```bash
git checkout -b fix-warning-dedup-migration
git commit -m "Fix warning dedup migration cleanup"
git push origin fix-warning-dedup-migration
```

然后在 GitHub 上创建 Pull Request。
