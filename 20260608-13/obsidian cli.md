# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `abef43f3-6be5-414d-af61-d4d5e924611b`
- **范围**: Turn 1-5
- **导出时间**: 2026-06-10 17:57:19

---

### Turn 1  *(2026-06-10 08:58)*

**User:**

obsidian cli 你知道吗？它能做什么

**Assistant:**

我先快速搜一下确认最新情况（Obsidian 官方 CLI 是最近才出的，我想核实一下细节再回答）。

知道，而且这正好和你这个 vault 相关。Obsidian 在 **1.12 版（2026 年 2 月）推出了官方 CLI**：随桌面端自带，在 Settings → General → Command line interface 里开启并点 "Register CLI" 即可使用。它和老的社区版 Yakitrak/obsidian-cli 不是一回事——官方版功能全得多。

**它是什么**：一个"遥控器"，通过 IPC 控制正在运行的 Obsidian 应用（没开会自动拉起），不是独立的 headless 工具。好处是所有操作都走 Obsidian 内部 API——比如用它移动文件会自动更新所有 wikilink，改属性会立刻反映到索引里，比直接改 vault 里的 .md 文件安全。

**能做什么**（100+ 个命令）：

- **笔记 CRUD**：读取、创建、追加/前插内容、移动、删除文件
- **搜索**：全文搜索，支持 JSON 输出，方便接脚本
- **日记和任务**：打开/创建今日日记、添加任务、勾选任务
- **元数据**：管理 frontmatter 属性、标签（含跨全库重命名标签）、别名、书签
- **链接**：查反链、找未解析的链接
- **模板**：对文件应用模板
- **应用管理**：装卸/重载插件、切主题、控制 Obsidian Sync（可在服务器上无 GUI 跑同步、做远程备份）
- **开发者命令**：截图、打开 devtools、执行 JavaScript

**对你最实际的用途**：让 AI 工具（比如我）通过 CLI 安全地读写你的 vault——搜索笔记、建日记、整理标签，而不用直接碰文件。官方还出了配套的 [Claude Code skill（kepano/obsidian-skills）](https://github.com/kepano/obsidian-skills/blob/main/skills/obsidian-cli/SKILL.md)。

一个针对你环境的提醒：你在 WSL2 里，而 Obsidian 大概率装在 Windows 侧。注册 CLI 后得到的是 Windows 的可执行文件，在 WSL 里要通过 Windows 互操作调用（类似 `obsidian.exe`），路径转换上可能要折腾一下。如果你想在这个 vault 里用起来，我可以帮你配置并测试。

Sources: [Obsidian CLI 官网](https://obsidian.md/cli) · [官方帮助文档](https://help.obsidian.md/cli) · [DEV: Obsidian's Official CLI Is Here](https://dev.to/shimo4228/obsidians-official-cli-is-here-no-more-hacking-your-vault-from-the-back-door-3123) · [完整命令指南](https://frankanaya.com/obsidian-cli/) · [Windows 安装避坑记录](https://zenn.dev/sora_biz/articles/obsidian-cli-setup-guide?locale=en)

---

### Turn 2  *(2026-06-10 09:10)*

**User:**

vault 是什么

**Assistant:**

Vault（仓库/笔记库）是 Obsidian 对"一个笔记库"的叫法。本质上它就是**你电脑上的一个普通文件夹**：Obsidian 打开哪个文件夹，那个文件夹就是一个 vault，里面所有的 `.md` 文件就是你的笔记。

它有两个特点：

- **就是本地文件夹**，没有任何封装或数据库。你可以用资源管理器、git、或者任何编辑器直接操作里面的文件——这也是为什么我现在能直接读写你的笔记。
- 文件夹里有个隐藏的 **`.obsidian` 子目录**，存这个库的配置：主题、插件、快捷键、工作区布局等。每个 vault 的配置是独立的。

具体到你这里：`/mnt/d/note`（也就是 Windows 的 `D:\note`）就是一个 vault——里面有 `.obsidian` 文件夹和你按周组织的笔记（`20260608-13/`、`客探/` 这些）。你平时在 Obsidian 里看到的整个侧边栏文件树，就是这个文件夹的内容。

所以上一条里说的"CLI 读写你的 vault"，意思就是通过命令行操作 `D:\note` 这个笔记库——但走 Obsidian 的 API 而不是直接改文件，链接、索引这些都会自动维护好。

---

### Turn 3  *(2026-06-10 09:19)*

**User:**

帮我配置并测试好obsidian cli，尝试兼容环境吧，我的claude code装在wsl，obsidian装在windows，我希望能让claude code把obsidian cli 用起来；给这个文档装一个项目级skill，即官方配套的 Claude Code skill（kepano/obsidian-skills）
  (https://github.com/kepano/obsidian-skills/blob/main/skills/obsidian-cli/SKILL.md)。

**Assistant:**

好，这个任务分两块：① 打通 WSL → Windows 的 Obsidian CLI 调用并测试；② 把官方 skill 装到这个 vault 的 `.claude/skills/` 里。我先看官方文档的安装细节，同时摸一下你 Windows 侧的 Obsidian 安装情况。

Windows 用户名是 `55211`。继续取文档、查 Obsidian 安装。

命令很全。做一个完整的写入往返测试：创建 → 追加 → 读取 → 删除。

全部配置完成，并且 skill 已经被当前会话识别加载了（可用 skill 列表里已出现 `obsidian-cli`），不用重启会话。

## 配置结果

**好消息是你之前已经在 Obsidian 里点过"Register CLI"**——Windows 用户 PATH 里已有 `D:\Program Files\Obsidian`，应用配置里 `cli: true`。剩下的跨环境工作我做了两件事：

1. **WSL 垫片**：`~/.local/bin/obsidian` → 转发到 Windows 侧的 `Obsidian.com`。这样在 WSL 里直接敲 `obsidian` 就能用，skill 也无需任何改动（它假设 PATH 上有 `obsidian` 命令）。新开 shell 验证过依然有效。
2. **项目级 skill**：官方仓库的 SKILL.md 原样装到 `.claude/skills/obsidian-cli/SKILL.md`。

## 测试情况（全部通过）

- `vaults` / `version`：识别到 vault `note`，版本 1.12.7 ✓
- 读中文路径文件（`20260608-13/待完成.md`）✓
- 完整写入往返：创建 → 追加中文内容 → 读回 → 删除（进回收站）✓
- `search`、`tags`、`backlinks` 类元数据命令 ✓

## 三个坑（已写进新建的 `CLAUDE.md`，以后每个会话自动知晓）

- **中文搜索要包一层内引号**做短语搜索：`query='"工作日志"'`。不带内引号的多字中文词会返回空。
- **出错时退出码也是 0**，判断成败要看输出里有没有 `Error:`，不能信 `$?`。
- 需要 Windows 侧 Obsidian 在运行（没开会自动拉起，等几秒）。

一个小观察：`daily:read` 返回空——你的库似乎没启用每日笔记（daily notes）插件，如果以后想用 `daily:append` 记日志，需要先在 Obsidian 里开它。

新增的三个文件（`CLAUDE.md`、`.claude/skills/obsidian-cli/SKILL.md`、WSL 侧垫片）里前两个在 vault 内，会随你的下一次 vault backup 自动提交。从现在起你可以直接说"搜一下笔记里关于 X 的内容"或"把这条加到某某笔记"，我会走 CLI 操作。

---

### Turn 4  *(2026-06-10 09:33)*

**User:**

怎么开始用

**Assistant:**

两种用法，最简单的是第一种：

## 1. 直接对我说（推荐）

什么都不用记，用自然语言提需求就行，我会自动调 CLI。比如：

- "搜一下笔记里所有提到 nginx 反代的内容"
- "把'联调企微接口'加到 `20260608-13/待完成.md` 里"
- "今天的工作日志里追加一条：修复了 score-and-info 的筛选 bug"
- "客探目录下有哪些笔记？帮我列个清单"
- "把 `张雪峰（舆论）.md` 移到归档文件夹，别弄断链接"

最后这类（移动/重命名）是 CLI 最有价值的场景——走 Obsidian API，所有引用它的 wikilink 会自动更新。

## 2. 自己在终端敲

WSL 终端里直接用（当前和新开的 shell 都已生效）：

```bash
obsidian help                                  # 全部命令
obsidian vault=note search query='"工作日志"'   # 搜索（中文记得包内引号）
obsidian vault=note read file="待完成"          # 按名字读笔记，不用写路径
obsidian vault=note append file="待完成" content="12.新任务"
obsidian vault=note tags sort=count counts     # 看标签使用统计
```

Windows 侧（PowerShell/cmd）也能用，**新开一个终端**就行（PATH 注册需要新会话才生效），语法相同。

## 上手建议

先从一个真实小事开始试，比如下一条要记的工作日志直接让我写：你说内容，我追加到 `每天的工作日志.md`，你再去 Obsidian 里确认格式合不合意。确认没问题后，搜索、整理、批量改标签这些再逐步交给我。

另外如果你想用 `daily:append` 这种每日笔记命令，需要先在 Obsidian 设置里启用"日记（Daily notes）"核心插件——目前你的库没开。

---

### Turn 5  *(2026-06-10 09:46)*

**User:**

归档文件夹是什么？daily:append 记日志，又是什么？引用它的 wikilink 会自动更新，这是什么意思？

**Assistant:**

三个都解释一下：

## 归档文件夹

那只是我举例随口说的名字，**你的库里并没有这个文件夹**。它指的是一种常见的整理习惯：建一个叫 `归档/` 或 `archive/` 的文件夹，把已经完结、不再活跃的笔记挪进去，让主目录保持清爽。比如你的 `智能拓客（第二阶段）` 项目如果彻底结束了，整个文件夹可以挪进归档。要不要建、叫什么名字，完全由你定。

## daily:append 和每日笔记

「每日笔记（Daily notes）」是 Obsidian 内置的一个功能（核心插件，需手动开启）：每天自动生成一篇以日期命名的笔记，比如 `2026-06-10.md`，作为当天的流水账/日志。

`obsidian daily:append content="..."` 就是"往**今天**这篇日记末尾加一行"，不存在会自动创建。适合随手记：开个终端敲一行，不用打开 Obsidian 找文件。

你现在的习惯是手动维护 `每天的工作日志.md`（每周文件夹里一个大文件），这是两种不同的组织方式。**不开每日笔记也完全没问题**——你让我"往工作日志里加一条"，我用 `append file="每天的工作日志"` 一样能办到，只是命令不同而已。

## wikilink 自动更新

Wikilink 是 Obsidian 里笔记互相引用的语法：在笔记里写 `[[每天的工作日志]]`，就生成一个点击可跳转的链接。

问题在于：**链接里存的是文件名**。假设你的 `待完成.md` 里写了 `[[张雪峰（舆论）]]`，现在把这个文件改名或挪走：

- **直接在文件系统里挪**（资源管理器拖拽、终端 `mv`）：Obsidian 不知道发生了什么，`待完成.md` 里那个链接还指着旧名字，变成断链（点击会创建一个同名空笔记，很坑）。
- **通过 Obsidian 或 CLI 挪**（`obsidian move` / `obsidian rename`）：走 Obsidian 内部 API，它知道"这个文件改名了"，会**自动把全库所有引用它的链接改成新名字**，一个断链都不会有。

这就是之前说"操作笔记优先走 CLI 而不是直接改文件"的原因——尤其是移动和重命名。纯粹改文件内容（追加一行字）倒是两种方式都安全。

---
