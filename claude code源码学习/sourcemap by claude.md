# Source Map 学习笔记 —— 与 Claude 的对话

---

## Q: 怎么从60MB的source map里，扒出来51万行Claude Code源代码

Source map 文件本质就是一个 JSON，关键字段是：

- **`sources`** — 原始文件路径数组
- **`sourcesContent`** — 对应的源代码内容数组

两个数组一一对应，提取就是遍历写文件。

### 方法一：一个简单脚本搞定

```javascript
const fs = require('fs');
const path = require('path');

const map = JSON.parse(fs.readFileSync('cli.mjs.map', 'utf8'));

map.sources.forEach((srcPath, i) => {
  if (!map.sourcesContent[i]) return;
  
  // 清理路径（source map 里经常有 ../node_modules 之类的前缀）
  const clean = srcPath.replace(/^(\.\.\/)*/,'');
  const outPath = path.join('output', clean);
  
  fs.mkdirSync(path.dirname(outPath), { recursive: true });
  fs.writeFileSync(outPath, map.sourcesContent[i]);
});

console.log(`提取了 ${map.sources.length} 个文件`);
```

### 方法二：用现成工具 `shuji`

```bash
npm i -g shuji
shuji cli.mjs.map --output ./extracted
```

### 关键步骤拆解

| 步骤 | 说明 |
|------|------|
| **1. 找到 source map** | Claude Code 装好后，在 `~/.claude/local/` 或 npm 全局目录下找 `.map` 文件 |
| **2. 确认结构** | `jq 'keys'` 看看有没有 `sourcesContent`，没有的话只有 mapping 没法还原 |
| **3. 检查文件数** | `jq '.sources \| length'` 看有多少原始文件 |
| **4. 提取** | 用上面任一方法把 `sourcesContent[i]` 写到 `sources[i]` 对应路径 |

### 为什么能提取 51 万行

Claude Code 是用 TypeScript 写的，打包成单个 `cli.mjs`（bundle），但发布时**附带了完整 source map**，里面 `sourcesContent` 字段包含了所有原始 `.ts` 源文件的完整内容——不是反编译，是原始代码。

60MB 的 map 文件里大部分体积是 `mappings` 字段（位置映射编码）和 `sourcesContent`（原始代码）。51 万行就是把所有 `sourcesContent` 条目加起来的总行数。

---

## Q: 我还是没能理解为什么有了完整 source map，就能映射出原始代码，能否举个例子

核心一句话：**sourcesContent 里存的就是原文件的完整拷贝，不是"映射回去"的，是"打包时塞进去"的。**

### 举个完整例子

假设你有两个 TypeScript 源文件：

**src/utils.ts**
```typescript
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

**src/index.ts**
```typescript
import { greet } from './utils';
console.log(greet("World"));
```

#### 打包后生成的 bundle.js

```javascript
function e(n){return`Hello, ${n}!`}console.log(e("World"));
//# sourceMappingURL=bundle.js.map
```

一行压缩完毕，人类看不懂原始结构了。

#### 打包同时生成的 bundle.js.map

```json
{
  "version": 3,
  "sources": [
    "src/utils.ts",
    "src/index.ts"
  ],
  "sourcesContent": [
    "export function greet(name: string): string {\n  return `Hello, ${name}!`;\n}\n",
    "import { greet } from './utils';\nconsole.log(greet(\"World\"));\n"
  ],
  "mappings": "AAAO,SAASA,EAAMC,GAClB,MAAO,UAAQA,KAGjB,CCFAC,QAAQC,IAAIH,EAAM,SAAS",
  "names": ["greet", "name", "console", "log"]
}
```

### 关键理解

| 字段 | 作用 | 内容 |
|------|------|------|
| `sources` | 原始文件**路径**列表 | `["src/utils.ts", "src/index.ts"]` |
| `sourcesContent` | 原始文件**完整源码** | 逐字逐句的原文，一个字都没变 |
| `mappings` | 位置对应关系 | 压缩后的第几列 ↔ 原始文件第几行第几列 |
| `names` | 被重命名的标识符原名 | `greet` 被压缩成 `e`，这里记着原名 |

### 两层信息，用途不同

```
                    sourcesContent          mappings
                    ──────────────          ────────
存的是什么？         原始源码全文              位置坐标对应表
用来干什么？         DevTools 里显示源码       断点/报错定位到原始行号
没有它会怎样？       只能看到压缩后的代码       有源码但点不准行号
提取源码靠谁？       ✅ 靠它                  ❌ 不靠它
```

所以提取源码根本不需要 `mappings`，只要：

```
sourcesContent[0]  →  写入  →  sources[0] 路径
sourcesContent[1]  →  写入  →  sources[1] 路径
...
```

### 为什么打包器会塞原文进去

打包工具（esbuild/webpack/rollup）生成 source map 时有两种模式：

- **有 `sourcesContent`**：map 文件自包含，浏览器/DevTools 打开就能显示原始代码，不需要原始文件在磁盘上。**Claude Code 用的就是这种。**
- **没有 `sourcesContent`**：map 里只有路径和 mappings，调试器需要自己去磁盘按 `sources` 路径找原始文件。

选第一种纯粹是为了调试方便——但副作用就是，任何人拿到 `.map` 文件就等于拿到了全部源码原文。

---

## Q: 也就是说60MB的source map里有sourcesContent，也就是五十多万行的源码

对，就这么直接。

那个 60MB 的 `.map` 文件里，体积大头就两块：

- **`sourcesContent`** — 所有原始 `.ts` 源文件的完整内容，占了大部分体积
- **`mappings`** — 一长串 Base64 VLQ 编码的位置映射字符串

提取源码只需要读 `sourcesContent`，`mappings` 完全用不到。

你可以验证一下：

```bash
# 看有多少个原始文件
node -e "const m=JSON.parse(require('fs').readFileSync('cli.mjs.map','utf8')); console.log(m.sources.length)"

# 看 sourcesContent 总行数
node -e "const m=JSON.parse(require('fs').readFileSync('cli.mjs.map','utf8')); console.log(m.sourcesContent.reduce((a,c)=>a+(c?c.split('\n').length:0),0))"

# 看 sourcesContent 占总文件体积的比例
node -e "const m=JSON.parse(require('fs').readFileSync('cli.mjs.map','utf8')); const sc=JSON.stringify(m.sourcesContent).length; const total=require('fs').statSync('cli.mjs.map').size; console.log((sc/total*100).toFixed(1)+'%')"
```

本质上就是打包时 esbuild 把每个源文件 `fs.readFileSync` 了一遍，原样塞进了 JSON 数组里。拿出来就是还原，没有任何反编译或猜测的过程。

---

## Q: 60MB大小的文件怎么能存五十多万行的代码，这些代码不怎么消耗存储空间吗

算笔账就清楚了。

### 51万行代码有多大

一行 TypeScript 平均大约 **35-45 个字符**（含缩进空格），取 40：

```
510,000 行 × 40 字节/行 ≈ 20MB
```

纯源码也就 **20MB 左右**，并不大。作为参考，一张手机照片就 5-10MB。

### 60MB 怎么分配的

```
60MB source map
├── sourcesContent  ≈ 25-30MB  （源码 + JSON 转义开销）
├── mappings        ≈ 25-30MB  （位置映射编码，最占空间的往往是它）
├── sources         ≈ 几百KB    （文件路径列表）
└── names           ≈ 几百KB    （标识符名称列表）
```

`sourcesContent` 在 JSON 里比原文还要大一些，因为：
- 换行符 → `\n`（1字节变2字节）
- 双引号 → `\"`（1字节变2字节）
- 反斜杠 → `\\`

所以 20MB 原始源码在 JSON 里膨胀到约 25-30MB。

### 直觉校准

代码就是纯文本，非常轻量：

| 参照物 | 大小 |
|--------|------|
| Linux 内核（2800万行） | ~1.1GB |
| VS Code 源码（~50万行） | ~60MB |
| 一本小说（20万字） | ~0.6MB |
| 一张 4K 照片 | ~10MB |
| **51万行 TS 代码** | **~20MB** |

51万行听起来多，但纯文本存储极其紧凑——每个字符就1个字节，一行代码还不如一个像素占的空间大。

---

## Q: 为什么需要source map

因为你发布出去的代码和你写的代码已经不是同一个东西了。

### 从源码到产物，经历了三层变形

拿 Claude Code 举例，原始代码是这样的：

```typescript
// src/utils/token.ts（第42行）
export function validateToken(token: string): boolean {
  const decoded = decodeJWT(token);
  if (decoded.exp < Date.now() / 1000) {
    throw new TokenExpiredError("Session expired");
  }
  return true;
}
```

发布后变成了这样：

```javascript
// cli.mjs（第1行，第83712个字符附近）
...,function(e){let t=w(e);if(t.exp<Date.now()/1e3)throw new P("Session expired");return!0},...
```

中间经历了：

| 变形 | 做了什么 | 为什么 |
|------|---------|--------|
| **编译** | TypeScript → JavaScript | 运行时不认 TS，类型标注全部擦除 |
| **打包** | 几百个文件 → 一个 `cli.mjs` | 减少文件数，解决模块依赖 |
| **压缩** | 变量重命名、删空格换行、简化语法 | 减小体积，加快加载 |

### 没有 source map 会怎样

线上报错你看到的是：

```
Error: Session expired
    at e (cli.mjs:1:83712)
```

- `e` 是什么函数？不知道
- `cli.mjs:1:83712` 对应原始哪个文件哪一行？不知道
- 想打断点调试？对着压缩成一行的代码打

**有了 source map：**

```
Error: Session expired
    at validateToken (src/utils/token.ts:44)
```

浏览器/DevTools 通过 `mappings` 把 `cli.mjs:1:83712` 翻译回 `token.ts:44`，通过 `sourcesContent` 直接展示原始 TypeScript 代码，你可以像调试源码一样打断点、单步执行。

### 一句话总结

Source map 就是**产物到源码的翻译词典**——让调试器在你面前展示你写的代码，而不是机器吃的代码。

---

## Q: 是否多此一举，github上都是发布的源码啊

Claude Code 没有开源。GitHub 上没有它的源码仓库。

你能拿到的只有 npm 发布的包，里面是打包压缩后的 `cli.mjs`。恰好附带了 source map，所以才能提取出源码。

### 不同项目情况不同

| 项目 | 源码公开？ | 需要 source map 调试？ |
|------|-----------|----------------------|
| VS Code | 开源，GitHub 有完整源码 | 有源码可以直接本地构建调试 |
| Claude Code | **不开源**，只发 npm 包 | source map 是唯一看到原始代码的途径 |
| 一般网站前端 | 源码不公开 | 生产环境通常**不放** source map，只内部用 |

### 那开源项目带 source map 是不是多此一举？

也不是。即使源码公开，source map 解决的是**运行时调试**的问题：

- 你 `npm install` 一个开源库，`node_modules` 里是编译后的产物
- 报错堆栈指向的是产物里的行号
- 有 source map，调试器自动映射回原始行号

你不会为了调试一个依赖库就去 clone 它的仓库、本地构建。source map 让你**装了就能调试**。

### Claude Code 为什么带 source map 发布

多数闭源产品在生产环境会**故意去掉** source map（防止泄露源码）。Claude Code 保留了它，可能是为了方便排查用户侧的错误堆栈——但副作用就是任何人都能提取出完整源码。

---

## Q: claude code 的npm包是怎样的，npm install是什么个原理

### Claude Code npm 包实际长这样

```
@anthropic-ai/claude-code/
├── package.json        ← 包的元信息（1.4KB）
├── cli.js              ← 全部源码打包成的单个文件（13MB, ~17000行压缩代码）
├── sdk-tools.d.ts      ← TypeScript 类型声明（115KB）
├── LICENSE.md
├── README.md
├── vendor/
│   ├── ripgrep/        ← 内置的 rg 搜索工具二进制
│   └── audio-capture/  ← 音频捕获工具
└── node_modules/       ← 可选依赖（sharp 图片处理，按平台装）
```

核心就一个 `cli.js`——13MB，几百个 TypeScript 源文件全压缩在这一个文件里。

### npm install 的原理

```
你执行: npm install -g @anthropic-ai/claude-code

        ┌──────────────────────────────────────────┐
  ①     │  npm 向 registry.npmjs.org 请求包的元信息    │
        │  得到 tarball 下载地址:                      │
        │  ...claude-code-2.1.91.tgz                 │
        └──────────────┬───────────────────────────────┘
                       ▼
        ┌──────────────────────────────────────────┐
  ②     │  下载 .tgz 压缩包                          │
        │  （就是 tar.gz，里面只有 package.json         │
        │   里 "files" 字段声明的那些文件）              │
        └──────────────┬───────────────────────────────┘
                       ▼
        ┌──────────────────────────────────────────┐
  ③     │  解压到 node_modules/@anthropic-ai/claude-code/ │
        │  安装 optionalDependencies（sharp 等）      │
        └──────────────┬───────────────────────────────┘
                       ▼
        ┌──────────────────────────────────────────┐
  ④     │  处理 "bin" 字段:                           │
        │  "bin": { "claude": "cli.js" }             │
        │                                            │
        │  创建一个符号链接/脚本:                       │
        │  ~/.nvm/.../bin/claude → ../lib/node_modules/ │
        │      @anthropic-ai/claude-code/cli.js       │
        └──────────────────────────────────────────┘

之后你在终端输入 claude，实际执行的就是:
  node cli.js
```

### package.json 里的关键字段

```json
{
  "bin": { "claude": "cli.js" },     // ← 注册 claude 命令
  "files": ["cli.js", "sdk-tools.d.ts", "vendor/..."],  // ← 发布到 npm 的文件白名单
  "optionalDependencies": {
    "@img/sharp-linux-x64": "..."    // ← 按平台可选装，装不上也不报错
  }
}
```

`files` 字段决定了 tgz 里包含什么。当前版本（2.1.91）**没有** `.map` 文件在 files 列表里——说明现在的版本已经不再发布 source map 了。之前能提取 51 万行源码，是因为早期版本把 `.map` 文件也一起发布了。

---

## Q: .map文件有什么，cli.js是运行的文件，可以通过map将运行文件映射回源码，是这样理解吗

对，就是这样。

```
开发时的几百个源文件          打包工具(esbuild)           发布的产物
                              
src/index.ts        ──┐                          ┌──  cli.js      (运行用)
src/auth/login.ts   ──┤      打包 + 压缩          │
src/tools/read.ts   ──┼──────────────────────────►├──  cli.js.map  (映射用)
src/ui/render.ts    ──┤                          │
...几百个文件        ──┘                          └──
```

- **cli.js** — Node.js 实际运行的文件，所有源码压缩成一坨
- **cli.js.map** — 记录着 cli.js 里的每个位置对应原始哪个文件的哪一行，同时内嵌了所有原始文件的完整内容

两者的关系就一句话：**cli.js 是给机器跑的，.map 是给人调试的，.map 能映射回（也包含着）全部原始源码。**

---

## Q: 那你之前和我说的sourcesContent是什么，按你说的意思我只需要cli.js和cli.js.map就能还原源码了啊

对，`.map` 文件里有两种东西能帮你还原：

### 方式一：靠 `sourcesContent`（直接还原）

```json
{
  "sources": ["src/auth/login.ts", "src/tools/read.ts", ...],
  "sourcesContent": ["原始源码全文...", "原始源码全文...", ...],
  "mappings": "AAAA,SAAS..."
}
```

`sourcesContent` 里存的就是源码原文，直接取出来写入文件就行。**根本不需要 cli.js 参与。**

### 方式二：靠 `mappings`（映射还原）

`mappings` 记录的是位置对应关系：

```
cli.js 第1行第83712列  ←→  src/auth/login.ts 第44行第3列
```

这个需要 cli.js 配合——你得从 cli.js 里把那个位置的代码拿出来，通过映射关系放回原始文件的对应位置。这种方式**只能还原代码内容，还原不了注释和格式**，因为压缩时已经丢掉了。

### 所以

```
有 sourcesContent 的 .map  →  只需要 .map 一个文件，100% 还原
没有 sourcesContent 的 .map →  需要 .map + cli.js 配合，且还原不完整
```

Claude Code 早期发布的 `.map` 里带了 `sourcesContent`，所以一个文件就能提取出全部 51 万行原始源码，cli.js 都不用看。

---

## Q: .map 文件长什么样

就是一个 JSON 文件。拿一个最小的真实例子看：

假设你有两个源文件：

**src/greet.ts**
```typescript
// 打招呼工具
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

**src/index.ts**
```typescript
import { greet } from './greet';
const result = greet("World");
console.log(result);
```

打包压缩后生成 **bundle.js**：

```javascript
function e(o){return`Hello, ${o}!`}let t=e("World");console.log(t);
//# sourceMappingURL=bundle.js.map
```

同时生成的 **bundle.js.map** 长这样：

```json
{
  "version": 3,

  "sources": [
    "src/greet.ts",
    "src/index.ts"
  ],

  "sourcesContent": [
    "// 打招呼工具\nexport function greet(name: string): string {\n  return `Hello, ${name}!`;\n}\n",
    "import { greet } from './greet';\nconst result = greet(\"World\");\nconsole.log(result);\n"
  ],

  "names": [
    "greet",
    "name",
    "result",
    "console",
    "log"
  ],

  "mappings": "AACO,SAASA,EAAMC,GACpB,MAAO,UAAQA,KAGjB,CCJA,OAAS,QAAQD,MAAW,YACxBE,EAASF,EAAM,SACnBG,QAAQC,IAAIF"
}
```

五个字段，各管各的：

| 字段 | 内容 | 作用 |
|------|------|------|
| `version` | `3` | source map 规范版本，目前都是 3 |
| `sources` | 原始文件路径列表 | 告诉你有哪些源文件 |
| `sourcesContent` | 每个源文件的**完整原文** | 提取源码就靠它 |
| `names` | 被压缩重命名的标识符原名 | `e` 原来叫 `greet`，`o` 原来叫 `name` |
| `mappings` | Base64 VLQ 编码的位置映射 | 压缩后的位置 ↔ 原始位置的对应表 |

Claude Code 的 `.map` 和这个结构一模一样，只不过 `sources` 数组有几百个文件路径，`sourcesContent` 数组有几百份完整源码，`mappings` 字符串几千万个字符长——撑到了 60MB。

---

## Q: Base64 VLQ 编码的位置映射，这个怎么用，原理是什么

### 先搞清楚 mappings 要编码什么

mappings 要回答的问题是：**压缩后文件的每个位置，对应原始文件的哪个位置。**

拿刚才的例子：

```javascript
// bundle.js (压缩后，就一行)
function e(o){return`Hello, ${o}!`}let t=e("World");console.log(t);
```

mappings 要记录的就是一堆这样的对应关系：

```
bundle.js 第1行第0列的 "function"  → src/greet.ts 第2行第0列
bundle.js 第1行第9列的 "e"         → src/greet.ts 第2行第16列（原名 greet）
bundle.js 第1行第11列的 "o"        → src/greet.ts 第2行第22列（原名 name）
...
```

每条映射有 4~5 个数字：

```
[ 生成文件的列, 源文件索引, 原始行, 原始列, 名称索引(可选) ]
```

### 为什么需要特殊编码

如果直接存 JSON 数组：

```json
[[0,0,1,0],[9,0,1,16,0],[11,0,1,22,1],...]
```

几万个映射点，JSON 会非常大。所以 source map 用了两个压缩技巧：

#### 技巧一：差值编码（delta encoding）

不存绝对值，存和前一个的差：

```
绝对值:  列0, 列9, 列11, 列14, 列35, 列38
差值:    0,   9,   2,    3,    21,   3     ← 数字变小了
```

数字越小，编码越短。

#### 技巧二：Base64 VLQ

把这些小整数用尽可能少的字符表示。

### VLQ 编码原理

VLQ = Variable-Length Quantity，变长编码。核心思想：**小数字用 1 个字符，大数字用多个字符。**

每个 Base64 字符代表 6 个 bit：

```
  ┌─────────────────────────────┐
  │  1 个 Base64 字符 = 6 bit     │
  │                               │
  │  [ C ][ D ][ D ][ D ][ D ][ S ] │
  │    ▲                         ▲  │
  │    │                         │  │
  │  续接位                   符号位(仅第一个字符) │
  │  1=后面还有               0=正数, 1=负数      │
  │  0=这是最后一个                               │
  └─────────────────────────────┘
```

### 手动编码一个数字

把 **-21** 编码成 Base64 VLQ：

```
第一步：取绝对值，确定符号
  21, 负数 → 符号位 = 1

第二步：转二进制
  21 = 10101

第三步：拆成每组 4 bit（第一组留一位给符号位，只放4bit；后续组放5bit）
  第一组(4bit): 0101   + 符号位1 → 01011
  第二组(5bit): 1      → 00001

第四步：加续接位（不是最后一组 → 1，是最后一组 → 0）
  第一组: 1 01011  → 101011 = 43 → Base64字符: r
  第二组: 0 00001  → 000001 =  1 → Base64字符: B

结果：-21 编码为 "rB"
```

Base64 字母表：

```
值:  0  1  2  3  4  ... 25 26 27 ... 51 52 ... 61 62 63
字符: A  B  C  D  E  ... Z  a  b  ... z  0  ...  9  +  /
```

### 几个常见数字的编码

```
 0  →  "A"        (1个字符)
 1  →  "C"        (1个字符)
-1  →  "D"        (1个字符)
 5  →  "K"        (1个字符)
15  →  "e"        (1个字符)
16  →  "gB"       (2个字符)    ← 超过15就需要2个字符
100 →  "oG"       (2个字符)
-21 →  "rB"       (2个字符)
```

小数字（差值编码后大部分都是）只需要 1 个字符，这就是为什么它压缩率高。

### 完整 mappings 的结构

```
"mappings": "AACO,SAASA,EAAMC,GACpB,..."
             ───── ─────  ─────  ──────
             段1    段2    段3    段4
             ↑
             用逗号分隔同一行内的各个映射段

             分号 ; 表示换行（生成文件的下一行）
```

解码 `AACO`：

```
A = 000000 → 续接0, 值00, 符号0 → 生成列 +0
A = 000000 → 源文件索引 +0
C = 000010 → 原始行 +1
O = 001110 → 原始列 +7
```

意思是：生成文件当前行第 0 列 → 第 0 个源文件的第 1 行第 7 列。

### 总结

```
原始映射数据（几万组坐标）
    │
    ▼  差值编码：存差值而非绝对值，数字变小
    │
    ▼  VLQ：小数字用1个字符，大数字用多个
    │
    ▼  Base64：用可打印ASCII字符表示
    │
结果："AACO,SAASA,EAAMC,GACpB,..."
```

一层层压缩下来，几万个映射点只需要一个紧凑的字符串，而不是巨大的 JSON 数组。



❯ 我要怎么才能下载泄露的map文件

● 早期版本的 npm 包里带了 .map 文件，npm 上的历史版本是可以下载的。

  # 下载一个早期版本的 tgz 包（source map 大约在 0.2.x 时期还存在）
  npm pack @anthropic-ai/claude-code@0.2.57

  # 解压看看里面有没有 .map
  tar tzf anthropic-ai-claude-code-0.2.57.tgz | grep '.map'

  如果那个版本没有，可以试着找找哪些版本有：

  # 查看所有发布过的版本
  npm view @anthropic-ai/claude-code versions --json

  然后挑早期版本逐个试。具体是哪个版本去掉了 .map，需要你自己二分查找一下。找到有 .map 的版本后：

  # 解压
  tar xzf anthropic-ai-claude-code-*.tgz
  # .map 文件就在 package/ 目录下
  ls -lh package/*.map
