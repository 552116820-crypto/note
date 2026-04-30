# git remote 多个远程地址

可以，可行，而且很常见。Git 一个本地仓库可以同时连接 GitHub、GitLab 等多个远程仓库。

常见有两种做法。

## 方式 1：配置多个 remote，最清晰

```bash
git remote add github git@github.com:yourname/project.git
git remote add gitlab git@gitlab.com:yourname/project.git
```

查看：

```bash
git remote -v
```

推送到 GitHub：

```bash
git push github main
```

推送到 GitLab：

```bash
git push gitlab main
```

如果想两个都推：

```bash
git push github main
git push gitlab main
```

这种方式最推荐，因为来源清楚，出问题也好排查。

## 方式 2：一个 remote 配多个 push 地址

比如 `origin` 同时推到 GitHub 和 GitLab：

```bash
git remote add origin git@github.com:yourname/project.git
git remote set-url --add --push origin git@github.com:yourname/project.git
git remote set-url --add --push origin git@gitlab.com:yourname/project.git
```

之后执行：

```bash
git push origin main
```

会同时推到两个地址。

注意：这种方式通常只适合“同时推送”。`git pull origin main` 仍然只会从一个 fetch 地址拉取，不会同时从两个远程拉。

## 推荐用法

更建议使用方式 1：

```text
origin -> 主远程，比如 GitLab
github -> GitHub 镜像
```

例如：

```bash
git remote add origin git@gitlab.com:yourname/project.git
git remote add github git@github.com:yourname/project.git
```

日常从 GitLab 拉：

```bash
git pull origin main
```

需要同步 GitHub 时：

```bash
git push github main
```

如果想让 GitLab 和 GitHub 保持完全同步，也可以考虑在 GitLab/GitHub 平台上配置镜像同步，比本地手动推两个仓库更稳。



• 可行，但要看你的本地历史是不是连续的。

  简单说：Git push 推的是提交历史，不是“只推 v3 的差异”。

  如果你的本地项目历史是：

  v1 commit -> v2 commit -> v3 commit

  而第一个远程仓库现在停在 v1，那么你把当前 v3 推到第一个仓库时，Git 会把中间没推过的 v2 提交也一起推上去。结果是第一个
  仓库直接从 v1 更新到 v3，历史里包含 v2。

  这种情况是正常可行的：

  git push 第一个远程名 当前分支:远程分支

  例如：

  git push origin main

  或者更明确：

  git push origin HEAD:main

  但有几个注意点：

  1. 如果第一个仓库的 main 还是你本地 v3 的祖先，通常可以直接 push。
  2. 如果第一个仓库在 v1 之后别人也提交过内容，你直接 push 可能会被拒绝，提示 non-fast-forward。这时需要先拉取合并或
     rebase。
  3. 如果你本地的 v2/v3 是基于第二个仓库完全不同的历史做出来的，可能不能直接推到第一个仓库的同名分支，除非你确认要覆盖或
     另开分支。
  4. 如果你不确定，建议先推到第一个仓库的新分支，最安全：

  git push origin HEAD:v3

  这样不会直接影响第一个仓库的 main 分支。

  你可以先检查一下：

  git remote -v
  git status
  git log --oneline --graph --decorate --all

  结论：如果 v3 是在 v1/v2 的连续历史上开发的，可以推到第一个仓库；没推过的 v2 内容会一起被推上去。
