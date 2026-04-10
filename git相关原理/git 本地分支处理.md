# Git 本地分支处理

日期：2026-04-10

项目路径：`E:\churn-warning-engine`

WSL 路径：`/mnt/e/churn-warning-engine`

---

## 1. 当时的问题

用户问：

> 我现在这个项目还在fix分支吗？我在github把fix分支删除了啊

核心结论：

> GitHub 上删除远程分支，不会自动删除本地分支，也不会自动把本地当前分支切回 `main`。

所以即使 GitHub 上已经删掉 `fix`，本地仍然可能还停留在 `fix` 分支。

---

## 2. 本地检查结果

执行：

```bash
git branch --show-current
git status --branch --short
```

当时输出显示：

```text
fix
## fix...origin/fix
 M app/db.py
```

含义：

- 当前本地分支是 `fix`
- 本地 `fix` 仍然在跟踪 `origin/fix`
- 工作区还有一个未提交修改：`app/db.py`

执行：

```bash
git branch -vv
```

当时输出：

```text
* fix  81bd08f [origin/fix] Fix 2
  main 7042d89 [origin/main: behind 2] Fix warning dedup migration cleanup
```

含义：

- `* fix`：当前分支是 `fix`
- `fix` 指向提交 `81bd08f`
- 本地 `main` 落后于 `origin/main`

执行：

```bash
git log --oneline --decorate --graph --all -n 12
```

当时看到：

```text
*   144e207 (origin/main, origin/HEAD) Merge pull request #1 from 552116820-crypto/fix
|\
| * 81bd08f (HEAD -> fix, origin/fix) Fix 2
|/
* 7042d89 (main) Fix warning dedup migration cleanup
* 143e0c1 first commit
```

含义：

- `fix` 分支的提交 `81bd08f` 已经通过 PR 合并进了 `origin/main`
- 本地 `fix` 还停在 `81bd08f`
- 远程 `origin/main` 已经在 merge commit：`144e207`
- 本地 `main` 仍在旧提交 `7042d89`

---

## 3. 为什么 GitHub 删除分支后本地还在 fix

Git 分支分三类：

```text
本地分支：fix
远程跟踪引用：origin/fix
GitHub 上真实远程分支：fix
```

GitHub 删除的是：

```text
GitHub 上真实远程分支 fix
```

但它不会自动删除：

```text
本地分支 fix
```

也不会自动清理：

```text
origin/fix
```

本地要知道远程分支被删了，需要执行：

```bash
git fetch --prune origin
```

`--prune` 的作用是清理本地已经失效的远程跟踪引用。

---

## 4. 为什么 fetch prune 失败

当时执行：

```bash
git fetch --prune origin
```

失败信息：

```text
fatal: could not read Username for 'https://github.com': No such device or address
```

含义：

- 当前终端无法读取 GitHub HTTPS 凭据
- 所以没法访问 GitHub 远程仓库
- 因此也没法确认 GitHub 上 `fix` 是否真的已经删除
- 也没法自动清理本地的 `origin/fix`

这不是代码问题，是 Git 认证问题。

---

## 5. 当时还有一个未提交改动

执行：

```bash
git diff -- app/db.py
```

看到：

```diff
 class MySQLRepository(Repository):
     """
     MySQL 仓储 — 生产环境用

+
     客户/订单数据从 CRM 表读取 (只读)
     预警任务/记录写入自建表
     """
```

含义：

`app/db.py` 只多了一个空行。

如果不需要这个空行，可以丢弃：

```bash
git restore app/db.py
```

如果需要保留，可以先 stash：

```bash
git stash push -m "temp db change"
```

---

## 6. 怎么安全回到 main

如果 `app/db.py` 的空行不需要，推荐：

```bash
git restore app/db.py
git checkout main
git merge --ff-only origin/main
```

解释：

```bash
git restore app/db.py
```

丢弃 `app/db.py` 的未提交改动。

```bash
git checkout main
```

切换到本地 `main` 分支。

```bash
git merge --ff-only origin/main
```

把本地 `main` 快进到 `origin/main`。

`--ff-only` 的意思是：只允许快进合并。如果不能快进，就直接失败，不自动创建 merge commit。

---

## 7. 怎么删除本地 fix 分支

切回 `main` 后，可以删除本地 `fix`：

```bash
git branch -d fix
```

注意：

不能在 `fix` 分支上删除 `fix` 自己。

如果当前还在 `fix`，执行：

```bash
git branch -d fix
```

会失败，因为 Git 不允许删除当前所在分支。

正确顺序：

```bash
git checkout main
git branch -d fix
```

如果 `fix` 已经合并进 `main`，`-d` 会成功。

如果 Git 认为还没有合并，会提示不能删除。这时不要急着用 `-D`，先确认提交是否真的已经进入 `main`。

强制删除命令是：

```bash
git branch -D fix
```

但只有在确认这个分支没有需要保留的提交时才用。

---

## 8. 推荐的完整清理流程

如果不保留 `app/db.py` 空行：

```bash
git restore app/db.py
git checkout main
git merge --ff-only origin/main
git branch -d fix
```

如果要先保留 `app/db.py` 空行：

```bash
git stash push -m "temp db change"
git checkout main
git merge --ff-only origin/main
git branch -d fix
```

之后如果要恢复 stash：

```bash
git stash pop
```

---

## 9. 用户后来输入的命令

用户随后输入：

```bash
git branch -d fix
```

但当时仍然在 `fix` 分支上。

如果直接执行，这个命令通常会失败，原因是：

> 不能删除当前正在使用的分支。

正确做法仍然是：

```bash
git checkout main
git branch -d fix
```

如果当前工作区有未提交改动导致无法切分支，需要先处理这些改动：

```bash
git restore app/db.py
```

或：

```bash
git stash push -m "temp db change"
```

---

## 10. 一句话总结

GitHub 删除远程 `fix` 分支，只影响 GitHub 上的分支。

本地仍然可能有：

```text
fix
origin/fix
```

要回到正常状态，通常做：

```bash
git checkout main
git merge --ff-only origin/main
git branch -d fix
git fetch --prune origin
```

其中 `git fetch --prune origin` 需要 GitHub 凭据可用。
