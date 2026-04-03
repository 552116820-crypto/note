# Obsidian 笔记上传到 GitHub

## 方式一：Git 命令行（最直接）

```bash
# 1. 进入 vault 目录
cd E:\note

# 2. 初始化 git 仓库
git init

# 3. 关联远程仓库（先在 GitHub 上创建一个空仓库）
git remote add origin https://github.com/你的用户名/仓库名.git

# 4. 添加所有文件并提交
git add .
git commit -m "Initial commit"

# 5. 推送
git push --set-upstream origin master
```

之后每次更新：
```bash
git add .
git commit -m "更新笔记"
git push
```

## 方式二：Obsidian Git 插件（推荐，自动同步）

1. Obsidian 中打开 **设置 → 第三方插件 → 关闭安全模式**
2. 搜索安装 **Obsidian Git** 插件
3. 先用命令行完成上面的初始化和远程仓库配置
4. 插件设置中可以配置**自动备份间隔**（如每 10 分钟自动 commit + push）

## 遇到的问题及解决

### 1. dubious ownership 错误

```
fatal: detected dubious ownership in repository at 'E:/note'
```

**解决：** 把目录标记为安全：
```bash
git config --global --add safe.directory E:/note
```

### 2. Author identity unknown 错误

```
Author identity unknown
```

**解决：** 设置用户名和邮箱：
```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱@example.com"
```

### 3. LF will be replaced by CRLF 警告

这只是警告，不是错误。Git 自动转换 Linux/Windows 换行符，可以忽略。

### 4. 弹出编辑器（没加 -m 参数）

commit 时没加 `-m` 会弹出编辑器。**以后每次 commit 记得加 `-m "说明"`**：
```bash
git commit -m "你的提交说明"
```

### 5. remote origin already exists 错误

需要先删除再重新添加：
```bash
git remote remove origin
git remote add origin https://github.com/你的用户名/仓库名.git
```

### 6. src refspec master does not match any 错误

说明之前的 commit 没成功，重新执行：
```bash
git add .
git commit -m "Initial commit"
git push --set-upstream origin master
```

## 建议的 .gitignore

```
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.trash/
```

## 注意事项

- 如果笔记包含隐私内容，记得创建 **Private** 仓库
- 图片等附件会增大仓库体积，考虑是否需要同步
- `git push --set-upstream origin master` 只需执行一次，之后直接 `git push` 即可
