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
