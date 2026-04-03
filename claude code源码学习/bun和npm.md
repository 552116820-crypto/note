## Bun 和 npm 脚本是什么

### Bun

Bun 是一个现代 JavaScript/TypeScript **运行时**，类似 Node.js，但更快。它同时是：
- **运行时**：直接执行 `.ts` 文件，不需要先编译
- **打包器**：把多个源文件打包成一个可执行文件
- **包管理器**：类似 npm，但速度更快

### npm 脚本

npm 脚本是 `package.json` 中 `scripts` 字段定义的命令快捷方式：

```json
{
  "scripts": {
    "build": "bun build src/main.tsx --outdir dist",
    "test": "jest",
    "lint": "eslint src/"
  }
}
```

运行方式：`npm run build`、`npm test` 等。

---

## 它们在这个项目中的作用

### Bun 的三个角色

#### 1. 编译时死代码消除（最核心的用途）

全项目 50+ 个文件导入了 `bun:bundle`，用 `feature()` 函数做**编译时条件分支**：

```typescript
import { feature } from 'bun:bundle'

// Bun 打包时，如果 VOICE_MODE 关闭，整个分支被物理删除
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// 内部员工专用代码
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool
  : null
```

这不是运行时 `if/else`，而是**打包时直接删掉死分支**，最终产物里根本不存在被关闭的代码。关键开关包括：

| 开关 | 控制的功能 |
|------|-----------|
| `KAIROS` | 助手/移动端模式 |
| `COORDINATOR_MODE` | 多 Agent 协调器 |
| `VOICE_MODE` | 语音输入输出 |
| `BRIDGE_MODE` | 远程会话桥接 |
| `AGENT_TRIGGERS` | 定时/触发 Agent |
| `WORKFLOW_SCRIPTS` | 工作流脚本 |
| `REACTIVE_COMPACT` | 响应式上下文压缩 |

#### 2. 编译为原生二进制

Bun 把整个 TypeScript 项目打包成**单个可执行文件**（不需要用户安装 Node.js）：

```typescript
// src/utils/bundledMode.ts
export function isInBundledMode(): boolean {
  return (
    typeof Bun !== 'undefined' &&
    Array.isArray(Bun.embeddedFiles) &&  // Bun 嵌入文件 API
    Bun.embeddedFiles.length > 0
  )
}
```

#### 3. 运行时检测

代码中会检测当前是 Bun 还是 Node.js 运行环境，走不同路径：

```typescript
export function isRunningWithBun(): boolean {
  return process.versions.bun !== undefined
}
```

---

### npm 的角色：分发渠道（正在淘汰）

这个项目通过 npm 注册表分发给用户，包名是 `@anthropic-ai/claude-code`。

**更新机制**（`src/cli/update.ts`）：
```bash
# 查询最新版本
npm view @anthropic-ai/claude-code@latest version

# 全局安装
npm i -g @anthropic-ai/claude-code

# 本地安装（~/.claude/local/）
npm install @anthropic-ai/claude-code@指定版本
```

**四种安装方式共存**：
1. `npm-global` — 全局 npm 安装
2. `npm-local` — 本地 npm 安装到 `~/.claude/local/`
3. `native` — Bun 编译的原生二进制（新的主推方式）
4. `package-manager` — Homebrew、winget 等系统包管理器

**npm 正在被淘汰**（`src/hooks/notifs/useNpmDeprecationNotification.tsx`）：
> "Claude Code has switched from npm to native installer. Run `claude install`..."

npm 用户会看到弃用警告，引导迁移到原生安装。

---

### 本快照的情况

这个仓库是源码快照，**没有 `package.json`**，所以：
- 无法运行 `npm install` / `npm run xxx`
- 无法用 Bun 打包
- `feature()` 调用在源码中保留原样，所有条件分支都可见（反而方便研究）

---

### 一句话总结

**Bun 是这个项目的构建引擎**（打包 + 死代码消除 + 编译为原生二进制），**npm 是分发渠道**（正在被原生安装替代）。两者都不影响源码阅读——`feature()` 当作 `if` 看就行。
