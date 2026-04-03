# Git 状态字母含义

## `git status --short` / `git diff` 中的状态字母

| 字母 | 含义 |
|------|------|
| `M` | **Modified** — 已修改 |
| `A` | **Added** — 新添加到暂存区 |
| `D` | **Deleted** — 已删除 |
| `R` | **Renamed** — 已重命名 |
| `C` | **Copied** — 已复制 |
| `U` | **Unmerged** — 有冲突，未合并 |
| `?` | **Untracked** — 未跟踪（新文件，未 `git add`） |
| `!` | **Ignored** — 被 `.gitignore` 忽略 |

## `git status --short` 的两列含义

输出格式是 `XY filename`，其中：
- **X**（左列）= 暂存区状态（staged）
- **Y**（右列）= 工作区状态（unstaged）

示例：
```
M  file.txt   # 已暂存的修改
 M file.txt   # 未暂存的修改
MM file.txt   # 暂存了一部分修改，工作区还有未暂存的修改
A  file.txt   # 新文件已暂存
?? file.txt   # 未跟踪的文件
UU file.txt   # 合并冲突（双方都修改了）
```

## 合并冲突时的 `U` 组合

| 标记 | 含义 |
|------|------|
| `DD` | 双方都删除了 |
| `AU` | 我方添加，对方未合并 |
| `UD` | 我方修改，对方删除 |
| `UA` | 对方添加，我方未合并 |
| `DU` | 我方删除，对方修改 |
| `AA` | 双方都添加了（内容不同） |
| `UU` | 双方都修改了 |
