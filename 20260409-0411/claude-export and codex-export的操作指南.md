# claude-export & codex-export 操作指南

将 Claude Code 和 Codex CLI 的对话内容导出为 Markdown 文件的终端工具。

## 前置条件

在终端中执行一次以加载命令：

```bash
source ~/.bashrc
```

或重新打开终端即可。

## 两个命令

| 命令 | 数据来源 |
|------|----------|
| `claude-export` | Claude Code 的对话记录 |
| `codex-export` | Codex CLI 的对话记录 |

两者参数完全一致，只是数据来源不同。Session ID 不能混用：claude-export 只能用 Claude 的 ID，codex-export 只能用 Codex 的 ID。

## 查看历史会话

```bash
# 列出 Claude Code 所有项目的会话（跨项目）
claude-export -ls

# 列出 Claude Code 指定项目的会话
claude-export -ls -p E:/churn-warning-engine

# 列出 Codex CLI 所有会话
codex-export -ls
```

输出示例：

```
/mnt/e/churn-warning-engine (4 个会话)
    1. [2026-04-10 07:01] (40 轮) 再次审查我这个项目代码
       ID: 4c840b9c-c472-42c6-a5b9-1fa3d19eeb42
    2. [2026-04-10 02:17] (73 轮) 再次审查这个项目代码
       ID: 693d152c-d9bc-4e3a-a579-b95111826806
```

从输出中获取 Session ID 用于后续导出。

## 导出对话

### 基本用法

```bash
# 导出当前项目最新会话的全部内容
claude-export E:/note/output.md

# 导出指定会话
claude-export E:/note/output.md -s 4c840b9c

# Codex 同理
codex-export E:/note/output.md -s 019d764e
```

Session ID 支持部分匹配，不需要输入完整 UUID，前几位即可。

### 指定轮次范围 `-t`

"轮次"指一轮用户提问 + 助手回答。

```bash
# 第3到第7轮
claude-export E:/note/output.md -s 4c840b9c -t 3-7

# 第5轮到结尾
claude-export E:/note/output.md -s 4c840b9c -t 5-

# 最后3轮
claude-export E:/note/output.md -s 4c840b9c -t -3

# 仅第5轮
claude-export E:/note/output.md -s 4c840b9c -t 5
```

### 指定项目 `-p`（仅 Claude Code）

不加 `-p` 时，`-s` 会自动在所有 Claude Code 项目中搜索。加 `-p` 则限定在指定项目内查找。

```bash
claude-export E:/note/output.md -p E:/churn-warning-engine -s 4c840b9c
```

## 文件写入规则

| 场景 | 行为 |
|------|------|
| 目标文件不存在 | 新建并写入 |
| 目标文件已有内容 | 自动追加到末尾 |
| 加 `--overwrite` | 清空后覆盖写入 |

多次导出到同一文件时，内容会依次追加，不会丢失之前的内容。

## 包含工具调用详情

默认只导出用户提问和助手回复文本。加 `--tools` 可同时导出工具调用记录（执行了什么命令、读了哪些文件等）。

```bash
claude-export E:/note/output.md -s 4c840b9c --tools
```

## 路径写法

在 WSL 终端中使用 Windows 路径时，用正斜杠：

```bash
# 正确 - 用正斜杠
claude-export E:/note/output.md

# 正确 - 用单引号包裹反斜杠
claude-export 'E:\note\output.md'

# 错误 - 不加引号的反斜杠会被 bash 吞掉
claude-export E:\note\output.md
```

Linux 原生路径正常使用：

```bash
claude-export /mnt/e/note/output.md
claude-export ~/notes/output.md
```

## 参数速查

| 参数 | 说明 |
|------|------|
| `-ls` | 列出会话列表 |
| `-s <id>` | 指定 Session ID（支持部分匹配） |
| `-t <range>` | 指定轮次范围（如 `3-7`、`-3`、`5-`） |
| `-p <path>` | 指定 Claude Code 项目目录 |
| `--tools` | 包含工具调用详情 |
| `--overwrite` | 覆盖已有文件（默认追加） |

## 脚本位置

- Python 主脚本：`~/.claude/scripts/claude-export.py`
- Bash wrapper：`~/.claude/scripts/chat-export`
- 别名定义：`~/.bashrc`
