# 项目说明

这个仓库是一个 Obsidian vault（笔记库），vault 名为 `note`，对应 Windows 路径 `D:\note`。

## Obsidian CLI（WSL → Windows 跨环境）

本机 Claude Code 跑在 WSL，Obsidian 装在 Windows（`D:\Program Files\Obsidian`）。
`obsidian` 命令是 WSL 端的垫片脚本（`~/.local/bin/obsidian`），转发到 Windows 的
`Obsidian.com`，通过 IPC 控制正在运行的 Obsidian 应用。

操作 vault 里的笔记（搜索、创建、移动、改属性等）优先用 obsidian CLI 而不是直接改文件：
CLI 走 Obsidian 内部 API，移动/重命名会自动更新 wikilink。用法见 skill
`.claude/skills/obsidian-cli/SKILL.md`，完整命令列表用 `obsidian help` 查看。

### 本环境的注意事项

- 目标 vault 用 `vault=note` 指定（如 `obsidian vault=note read file="笔记名"`）。
- **中文搜索要在查询词外再包一层引号**（短语搜索）：
  `obsidian vault=note search query='"工作日志"'`。
  不带内层引号的多字中文词可能返回空结果。
- **出错时退出码仍为 0**，不能用 `$?` 判断成败，要检查输出里有没有 `Error:`。
- 需要 Windows 侧 Obsidian 在运行；没运行时第一条命令会自动拉起它（要等几秒）。
- 中文文件名、中文内容的读写都正常，无需转码。
- `delete` 默认进回收站（`.trash`），加 `permanent` 才永久删除——一般不要加。
