• 已处理。现在 Windows PowerShell 里可以直接运行：

  claude-export --ls
  claude-export output.md -s e9c33078 --overwrite
  claude-export output.md -p E:\churn-warning-engine -s e9c33078 --overwrite

  我做了这些改动：

  - 新增 Windows 命令包装器：AppData/Local/Programs/Python/Python310/Scripts/claude-export.cmd:1，这个目录已经在你的 Windows 用户 PATH 里。
  - 更新 /home/huangch001002/.claude/scripts/claude-export.py:34，让路径处理兼容 Windows 和 WSL。
  - 修正 Windows 下按当前目录匹配 Claude 项目目录的问题：/home/huangch001002/.claude/scripts/claude-export.py:130。
  - 给 --ls 增加别名支持；原来的 -ls 也还保留：/home/huangch001002/.claude/scripts/claude-export.py:465。
  - JSONL 读取改成 UTF-8，避免 Windows 默认编码影响。

  验证过：

  Get-Command claude-export
  claude-export --ls
  claude-export C:\Users\55211\test.md -s e9c33078 --overwrite
  claude-export C:\Users\55211\test.md -p E:\churn-warning-engine -s e9c33078 --overwrite

  都能跑通。测试生成的临时 markdown 我已经删掉了。
