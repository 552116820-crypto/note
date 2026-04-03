# Symlink 与 Skill 安装机制

## 安装 Skill 的命令

```bash
npx skills add github:keli-wen/agentic-harness-patterns-skill --yes
```

使用 `--yes` 标志可以跳过交互式选择，自动安装所有 skill。

安装结果：成功安装了 2 个 skill：
- **agentic-harness-patterns** — 英文版
- **agentic-harness-patterns-zh** — 中文版

---

## Skill 安装位置

| 路径 | 内容 |
|---|---|
| `/mnt/c/Users/55211/.agents/skills/` | skill 的实际文件（当前工作目录下） |
| `~/.claude/skills/` | Claude Code 全局 skill 目录（存放 symlink） |

安装的 `agentic-harness-patterns-zh` 的文件结构：

```
/mnt/c/Users/55211/.agents/skills/agentic-harness-patterns-zh/
├── SKILL.md              # 主文件
└── references/           # 参考文档
    ├── agent-orchestration-pattern.md
    ├── bootstrap-sequence-pattern.md
    ├── context-engineering/
    │   ├── compress-pattern.md
    │   ├── isolate-pattern.md
    │   └── select-pattern.md
    ├── context-engineering-pattern.md
    ├── hook-lifecycle-pattern.md
    ├── memory-persistence-pattern.md
    ├── permission-gate-pattern.md
    ├── skill-runtime-pattern.md
    ├── task-decomposition-pattern.md
    └── tool-registry-pattern.md
```

---

## Symlink 链接关系

通过 `find ~/.claude -type l` 可以看到实际的 symlink：

```
/mnt/c/Users/55211/.claude/skills/agentic-harness-patterns
    → ../../.agents/skills/agentic-harness-patterns

/mnt/c/Users/55211/.claude/skills/agentic-harness-patterns-zh
    → ../../.agents/skills/agentic-harness-patterns-zh

/mnt/c/Users/55211/.claude/skills/skill-vetter
    → ../../.agents/skills/skill-vetter
```

翻译成绝对路径：

```
/mnt/c/Users/55211/.claude/skills/agentic-harness-patterns-zh  (symlink，快捷方式)
        ↓ 指向
/mnt/c/Users/55211/.agents/skills/agentic-harness-patterns-zh  (实际文件)
```

---

## Symlink 工作原理

**Symlink（symbolic link，符号链接）本质上是一个特殊的小文件，里面只存了一个路径字符串。**

```
普通文件:  文件 → 直接存储内容
Symlink:  文件 → 存储一个路径 → 操作系统自动跳转到那个路径读取内容
```

### Claude Code 如何通过 symlink 加载 skill

1. Claude Code 启动时去 `~/.claude/skills/` 目录下扫描所有 skill
2. 发现 `agentic-harness-patterns-zh` 这个条目
3. 操作系统发现它是 symlink，**自动、透明地**跳转到 `../../.agents/skills/agentic-harness-patterns-zh`
4. 读取到那里的 `SKILL.md` 和 `references/`

对 Claude Code 来说，它**感知不到**这是 symlink 还是真实目录——操作系统在文件系统层面就完成了跳转。

---

## 为什么这样设计

```
.agents/skills/          ← 实际文件（项目级，可以被多个工具共用）
                           安装器说的 "Codex, Amp, Cursor +7 more" 都读这里

~/.claude/skills/        ← symlink（Claude Code 专用入口）
                           只是一个指针，不复制文件
```

好处：
- **一份文件，多处引用** — 不浪费磁盘，不产生副本不一致的问题
- **更新只需改一处** — 更新 `.agents/skills/` 里的文件，所有通过 symlink 引用它的工具都自动看到新内容
- **卸载干净** — 删掉 `.agents/skills/` 里的目录，symlink 自动失效（变成"断链"）

---

## Symlink 文件本身的位置和大小

Symlink 文件地址：

```
/mnt/c/Users/55211/.claude/skills/agentic-harness-patterns-zh
```

验证方式：

```bash
$ file /mnt/c/Users/55211/.claude/skills/agentic-harness-patterns-zh
symbolic link to ../../.agents/skills/agentic-harness-patterns-zh

$ ls -la /mnt/c/Users/55211/.claude/skills/
l rwxrwxrwx  agentic-harness-patterns    -> ../../.agents/skills/agentic-harness-patterns
l rwxrwxrwx  agentic-harness-patterns-zh -> ../../.agents/skills/agentic-harness-patterns-zh
l rwxrwxrwx  skill-vetter               -> ../../.agents/skills/skill-vetter
```

注意 `ls -la` 输出中的关键信息：

```
l rwxrwxrwx  agentic-harness-patterns-zh -> ../../.agents/skills/agentic-harness-patterns-zh
↑                                         ↑
文件类型是 l (link)                        箭头指向实际目标
```

- **大小**：只有 48 字节（存储路径字符串的长度）
- **内容**：就一个字符串 `../../.agents/skills/agentic-harness-patterns-zh`

它不是目录，不是副本，就是一个 48 字节的"指路牌"。



● 对，这就是 3 个 symlink。每一行都是一个：

  l rwxrwxrwx  agentic-harness-patterns -> ../../.agents/skills/agentic-harness-patterns
  │                                     │
  │                                     └── 指向的实际目标
  └── 开头的 "l" 表示这是一个 link（符号链接），不是普通文件(-)或目录(d)
