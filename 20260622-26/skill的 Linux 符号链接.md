# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `fad42daf-0255-4590-a45c-428478840dde`
- **范围**: Turn 1-3
- **导出时间**: 2026-06-24 16:16:40

---

### Turn 1  *(2026-06-24 08:04)*

**User:**

"C:\Users\55211\.claude\skills"为什么打不开

**Assistant:**

Good — that control test is the key. `cmd.exe` even fails to `cd` into `C:\Windows` here, so all those "语法不正确" errors were just a WSL→cmd.exe interop artifact, not real. PowerShell is the reliable witness. One last check to lock down the reparse-tag and the broken link:

先把结论说清楚：**`skills` 文件夹本身没坏，Windows 完全能打开**——我用 PowerShell 验证了 `Get-ChildItem` / `Test-Path` / `Set-Location` 都正常。真正"打不开"的是它里面那两个技能条目。

（说明：刚才那堆 `cd /d ... 语法不正确` 是 WSL 里启动 cmd.exe 的串台 bug，连 `cd /d C:\Windows` 都报同样的错，所以那些报错不算数，PowerShell 才是可信的。）

## 里面到底是什么

`skills\` 下面不是真文件夹，是两个**在 WSL 里用 `ln -s` 建的 Linux 符号链接**，真身在 `C:\Users\55211\.agents\skills\`：

| 条目 | 指向 | 状态 |
|---|---|---|
| `agentic-harness-patterns-zh` | `..\..\.agents\skills\agentic-harness-patterns-zh` | 目标**存在**，PowerShell 能跟进去 |
| `skill-vetter` | `..\..\.agents\skills\skill-vetter` | 目标**已不存在 = 断链** |

## 为什么"打不开"——两个独立原因

1. **WSL 符号链接，资源管理器（双击）不认。** 这两个条目的 reparse 标签是 WSL 专属的 LX_SYMLINK，命令行 / PowerShell / WSL 能解析（我实测 PowerShell 能 `cd` 进 `agentic-harness-patterns-zh`），但 **Windows 图形资源管理器不解析这种链接**，所以你在 Explorer 里双击点进去就会失败、看着像打不开。

2. **`skill-vetter` 是死链。** 它指向的目标在 `.agents\skills\` 里已经没了（那目录现在只剩 `agentic-harness-patterns-zh`），所以这个不管 Windows 还是 WSL 都打不开——目标本身没了。这也解释了为什么本次会话里只加载到了 `agentic-harness-patterns-zh`，没有 `skill-vetter`。

## 怎么办（看你想干嘛）

- **只是想看/编辑技能内容** → 别走链接，直接开真身目录 `C:\Users\55211\.agents\skills\`，那是普通文件夹，Explorer 正常打开。
- **想让链接在资源管理器里也能点开** → 把 WSL 符号链接换成 Windows 原生符号链接（管理员 PowerShell `New-Item -ItemType SymbolicLink`，或 `mklink /D`，且用绝对路径）。
- **清理那个死链 `skill-vetter`** → 直接删掉无效链接，或把对应技能补回 `.agents\skills\skill-vetter`。

---

你具体是**在资源管理器里双击打不开**，还是**某个技能在 Claude Code 里没生效**？告诉我一句，我可以直接帮你：删掉 `skill-vetter` 死链，或把这两个链接转成 Windows 能认的原生符号链接。

---

### Turn 2  *(2026-06-24 08:12)*

**User:**

删掉 skill-vetter 死链

**Assistant:**

已删除。`skill-vetter` 死链清掉了，`skills\` 目录现在只剩有效的 `agentic-harness-patterns-zh`。

删之前加了道保险——只有确认它确实是「符号链接 + 目标不存在」才动手（`rm` 删的是链接本身，不会碰目标，何况目标早没了），所以不会误伤别的东西。

剩下的 `agentic-harness-patterns-zh` 是好的，要是哪天也想让它在资源管理器里能双击点开，再叫我把它转成 Windows 原生符号链接就行。

---

### Turn 3  *(2026-06-24 08:14)*

**User:**

为什么会用 Linux 符号链接

**Assistant:**

直接原因一句话：**这些链接是 WSL 里的进程建的，而 WSL 里任何对 `/mnt/c` 调用 `symlink()` 的程序，建出来的都是 Linux 风格符号链接**——这是 WSL 实现 `ln -s` 的固定行为，它会写成一个带 `LX_SYMLINK` 标签的 NTFS reparse point，而不是 Windows 原生符号链接。

展开几层：

**谁建的** — 几乎可以肯定是 Claude Code 的技能管理。它的结构是分两层的：
- `.agents\skills\` = 技能**真身仓库**（内容放这）
- `.claude\skills\` = Claude Code 读取的**"已启用"视图**，里面放的是指回仓库的链接

Claude Code 这个 node 进程在 WSL/Linux 下跑，它去建链接，建出来的自然就是 Linux 链接。

**为什么用符号链接、而不是直接复制** — 省一份拷贝、一个源多处共享、启用/停用一个技能只需要加/删链接，不用搬文件。这跟 nvm、pnpm 那套"用链接拼出一个视图"是一个思路。

**为什么偏偏是 Linux 链接、不是 Windows 原生** — 三点叠加：
1. 创建者在 WSL 里，它根本不会去走 Windows 的 `CreateSymbolicLink`；
2. 链接目标是相对 POSIX 路径 `../../.agents/skills/...`（正斜杠、`../`），这是 Linux 写法；
3. Windows 原生符号链接默认要**管理员权限或开发者模式**才能建，而在 WSL 里建链接不需要任何 Windows 特权——对工具来说 WSL 路径更省事。

**所以这其实不是配置错误。** 对在 WSL 里运行的 Claude Code 而言，这些链接完全正常、能正确解析；只有当你从 Windows 图形资源管理器跨过 WSL/Windows 边界去双击它们时，才因为 Explorer 不认 LX 链接而显得"打不开"。换句话说，是"用错了打开方式"，不是"链接坏了"(`skill-vetter` 那个除外，那是真断链，已经删了)。

---
