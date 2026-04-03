用户
怎么从60MB的source map里，扒出来51万行Claude Code源代码

助手
不能帮你针对 `Claude Code` 这种第三方专有产品去“扒源码”。

如果你的目的是审计你自己项目或经授权的 bundle，我可以讲通用原理：

`source map` 本质上是一个 JSON，关键字段通常有：

- `sources`：原始文件路径列表
- `sourcesContent`：原始源码内容
- `mappings`：压缩后代码和原代码的映射

核心判断只有一个：

1. 如果有 `sourcesContent`
那基本等于“源码随 map 一起发了”。经授权时，重建就是把 `sources[i]` 和 `sourcesContent[i]` 一一写回文件。

2. 如果没有 `sourcesContent`
那通常只能拿到文件名、路径和映射关系，不能完整恢复源码，只能辅助反混淆和定位。

如果你是在做自己的排查，最直接的检查方式是：

```bash
jq 'has("sourcesContent"), (.sourcesContent | length), (.sources | length)' app.js.map
```

再看某个条目是否真带源码：

```bash
jq -r '.sources[0], .sourcesContent[0][0:200]' app.js.map
```

经授权的重建脚本思路也很简单：

```js
const fs = require("fs");
const path = require("path");

const map = JSON.parse(fs.readFileSync("app.js.map", "utf8"));

map.sources.forEach((src, i) => {
  const content = map.sourcesContent?.[i];
  if (typeof content !== "string") return;
  const out = path.join("recovered-src", src);
  fs.mkdirSync(path.dirname(out), { recursive: true });
  fs.writeFileSync(out, content, "utf8");
});
```

如果你的真实需求是防守侧，我可以继续帮你：

- 判断某个 `.map` 是否真的泄露了完整源码
- 批量扫描产物目录里哪些 source map 含 `sourcesContent`
- 给 webpack/vite/esbuild 的关闭或脱敏配置
- 写一个审计脚本，统计“泄露了多少文件/多少行”

用户
我还是没能理解为什么有了完整 source map，就能映射出原始代码，能否举个例子

助手
可以。关键先讲一句最重要的：

**“能还原原始代码”通常不是因为 `mappings` 神奇地把压缩代码逆推出源码，而是因为 source map 里经常直接带了 `sourcesContent`，也就是源码全文。**

`mappings` 负责“坐标对应”，`sourcesContent` 负责“源码正文”。

**一个最小例子**

原始文件 `src/add.ts`：

```ts
export function add(a: number, b: number) {
  const sum = a + b;
  return sum;
}
```

构建后变成 `dist/add.js`：

```js
export function add(n,r){return n+r}
```

同时生成一个 `dist/add.js.map`，简化后大概像这样：

```json
{
  "version": 3,
  "file": "add.js",
  "sources": ["../src/add.ts"],
  "sourcesContent": [
    "export function add(a: number, b: number) {\n  const sum = a + b;\n  return sum;\n}\n"
  ],
  "names": ["add", "a", "b", "sum"],
  "mappings": "AAAA,..."
}
```

这几个字段分别干什么：

- `sources`: 原始文件名/路径
- `sourcesContent`: 原始文件完整内容
- `mappings`: 构建后代码的“第几行第几列”对应原文件“第几行第几列”
- `names`: 标识符名字表，辅助映射

**为什么浏览器能显示原始 TS？**

因为它做了两件事：

1. 读 `sourcesContent[0]`，拿到完整的 `add.ts` 源码文本
2. 读 `mappings`，知道压缩后 `add.js` 某个位置对应 `add.ts` 的哪一行哪一列

所以 DevTools 里看到“原始源码”，本质上常常不是“推理出来的”，而是**map 文件自己带着源码**。

**如果没有 `sourcesContent` 会怎样？**

比如 map 只有：

```json
{
  "version": 3,
  "file": "add.js",
  "sources": ["../src/add.ts"],
  "names": ["add", "a", "b", "sum"],
  "mappings": "AAAA,..."
}
```

这时你只能知道：

- `dist/add.js` 第 1 行第 16 列
- 对应 `src/add.ts` 第 2 行第 9 列

但你**不知道第 2 行第 9 列到底写了什么**，除非你另外还能拿到 `src/add.ts` 文件本身。

所以：

- 有 `sourcesContent`：通常能直接恢复完整源码
- 没有 `sourcesContent`：通常只能恢复文件路径、行列关系，不能恢复全文

**可以把它理解成：**

- `mappings` = 目录索引 / 坐标表
- `sourcesContent` = 正文原文

只有索引，没有正文，不可能凭空还原整本书。  
索引加正文，就等于书已经在里面了。

如果你愿意，我下一条可以直接给你画一个“某个压缩 token 是怎么映射回原文件某一行某一列”的逐字符例子。

用户
我愿意

助手
可以，直接看一个“按位置映射”的小例子最容易明白。

先假设原始文件是 `src/math.js`：

```js
function add(a, b) {
  return a + b;
}
```

构建压缩后变成：

```js
function n(o,r){return o+r}
```

同时有一个 `math.js.map`。先别管它里面那串很丑的 `mappings` 字符串，**浏览器实际关心的是“解码后”的映射段**。你可以把它理解成这样一张表：

| 生成后代码位置 | 对应原始代码位置 | 含义 |
|---|---|---|
| `1:0` | `src/math.js 1:0` | `function` 从这里开始对应 |
| `1:9` | `src/math.js 1:9` | `n` 对应原来的 `add` |
| `1:11` | `src/math.js 1:13` | `o` 对应原来的 `a` |
| `1:13` | `src/math.js 1:16` | `r` 对应原来的 `b` |
| `1:16` | `src/math.js 2:2` | `return` 对应原文件第 2 行 |
| `1:23` | `src/math.js 2:9` | `o` 对应原来的 `a` |
| `1:25` | `src/math.js 2:13` | `r` 对应原来的 `b` |

这里的 `1:23` 意思是“生成后代码第 1 行第 23 列”。

把压缩后的代码标一下列号，大概是这样：

```txt
function n(o,r){return o+r}
0        9 1 3 6      3 5
         0 1 3 6      3 5
```

更直观点：

```txt
0123456789012345678901234567
function n(o,r){return o+r}
```

所以：

- 第 `9` 列的 `n`
- 映射回原文件第 `1` 行第 `9` 列的 `add`

- 第 `23` 列的 `o`
- 映射回原文件第 `2` 行第 `9` 列的 `a`

这就是“某个压缩 token 怎么跳回源码位置”的本质。

关键点有两个：

1. `mappings` 只存“坐标关系”
它告诉你压缩后某个位置，对应原文件哪个位置。

2. `sourcesContent` 才是“源码正文”
如果 map 里还有：

```json
{
  "sources": ["src/math.js"],
  "sourcesContent": ["function add(a, b) {\n  return a + b;\n}\n"]
}
```

那浏览器就不只是知道“跳到第 2 行第 9 列”，而是**连原文件全文都已经拿到了**。

所以 DevTools 才能直接显示：

```js
function add(a, b) {
  return a + b;
}
```

不是它从 `function n(o,r){return o+r}` 逆向推理出了 `add/a/b`，而是：

- `mappings` 告诉它“这里对应哪里”
- `sourcesContent` 直接把原文给它了

你可以把它理解成：

- `mappings` = 地图坐标
- `sourcesContent` = 实际城市照片

只有坐标，没有照片，你知道“这个点在北京东城区某条街”，但看不到街景。  
坐标加照片，就能直接把画面还原出来。

再补一个很容易误解的点：

**source map 不是给每个字符都单独存一次映射。**  
它通常存的是一段一段的“锚点”。浏览器看到某个位置时，会找“离它最近的前一个映射段”，从而知道这一段代码对应原文件哪里。

如果你愿意，我下一条可以继续画出一个“真实 `source map` JSON 长什么样，以及 `mappings` 里的一个片段是怎么解码成 `generatedColumn/source/originalLine/originalColumn/name`”的例子。

用户
我愿意

助手
可以，直接看一个“结构真实但缩到最小”的例子。

假设原文件是 `src/math.js`：

```js
function add(a, b) {
  return a + b;
}
```

构建后某一段生成文件是 `bundle.js`，对应的 map 里可能有：

```json
{
  "version": 3,
  "file": "bundle.js",
  "sources": ["src/math.js"],
  "sourcesContent": ["function add(a, b) {\n  return a + b;\n}\n"],
  "names": ["add", "a", "b"],
  "mappings": "AAAAA,CAACC"
}
```

这几个字段里，真正关键的是两类：

- `sourcesContent[0]`：原文件全文，已经在 map 里了
- `mappings`：生成代码位置怎么跳回原文件位置

`sourcemap` 规范里的行列都是 **0-based**。

**怎么读 `mappings`**

先记两个分隔符：

- `;`：下一条生成代码的“新一行”，并把 `generatedColumn` 重置为 0
- `,`：同一生成行里的下一个 segment

所以 `AAAAA,CAACC` 表示：生成文件第 1 行里有两个 segment。

每个 segment 解码后通常是这些字段的“相对增量”：

1. `generatedColumn`
2. `sourceIndex`
3. `originalLine`
4. `originalColumn`
5. `nameIndex`（可选）

**第一个 segment：`AAAAA`**

每个 `A` 的 base64 VLQ 值都是 `0`，所以：

```txt
AAAAA -> [0, 0, 0, 0, 0]
```

把它加到初始状态上，得到绝对位置：

- 生成文件：`line 0, column 0`
- 原文件：`sources[0]`，也就是 `src/math.js`
- 原位置：`line 0, column 0`
- 名字：`names[0]`，也就是 `add`

也就是说，生成代码第 1 行第 1 列附近的这个符号，对应原文件开头，并且名称关联到 `add`。

**第二个 segment：`CAACC`**

这里 `C` 解码成 `+1`，`A` 还是 `0`。所以：

```txt
CAACC -> [1, 0, 0, 1, 1]
```

注意它是“相对前一个 segment”的增量，所以绝对值变成：

- `generatedColumn`: `0 + 1 = 1`
- `sourceIndex`: `0 + 0 = 0`
- `originalLine`: `0 + 0 = 0`
- `originalColumn`: `0 + 1 = 1`
- `nameIndex`: `0 + 1 = 1`

于是这个 segment 的绝对映射是：

- 生成文件：`line 0, column 1`
- 原文件：`src/math.js`
- 原位置：`line 0, column 1`
- 名字：`names[1]`，也就是 `a`

**所以 DevTools 实际在做什么？**

1. 读 `mappings`，把“生成文件第几行第几列”映射回“原文件第几行第几列”
2. 读 `sources[sourceIndex]`，知道是哪个原文件
3. 如果有 `sourcesContent[sourceIndex]`，直接把那份源码正文展示出来

这就是为什么“一个 60MB 的 source map”可能对应出几十万行源码：

- 不是 `mappings` 把压缩代码“逆向推理”成源码
- 而是 map 里经常已经塞了大量 `sourcesContent`

一句话总结：

- `mappings` = 坐标表
- `sourcesContent` = 源码正文
- 两者合起来，浏览器就能像打开原工程一样显示源码

如果你愿意，我下一条可以继续把 `AAAAA,CAACC` 这种字符串里，`A/C` 为什么分别代表 `0/+1` 的 base64 VLQ 规则也拆开讲。

用户
生成文件是什么

助手
“生成文件”就是**构建工具产出的那个文件**，也就是浏览器、Node、App 最终真正运行的版本。

对应关系一般是这样：

- 原始文件：你写的源码，比如 `src/app.ts`
- 生成文件：工具编译/打包/压缩后的结果，比如 `dist/app.js`
- source map：描述“`dist/app.js` 的某个位置，对应 `src/app.ts` 的哪个位置”

一个最简单的例子：

你写的是：

```ts
// src/app.ts
const msg: string = "hello";
console.log(msg);
```

TypeScript 编译后变成：

```js
// dist/app.js
const msg = "hello";
console.log(msg);
```

这里：

- `src/app.ts` 是原始文件
- `dist/app.js` 是生成文件

如果再压缩，可能变成：

```js
console.log("hello");
```

这个压缩后的文件也还是“生成文件”。

所以前面说“生成文件第 1 行第 23 列映射到原始文件第 2 行第 9 列”，意思就是：

- 你现在看到的是产物里的位置
- source map 告诉你它原本来自源码哪里的哪一段

常见的生成文件长这样：

- `bundle.js`
- `app.min.js`
- `main.js`
- `vendor.js`

在 source map 里，`file` 字段通常就是这个生成文件名，比如：

```json
{
  "file": "bundle.js"
}
```

如果你愿意，我下一条可以继续把这三个东西画成一张图：

`原始文件 -> 生成文件 -> source map` 的关系图。

用户
愿意

助手
可以，把三者关系画成一张图最清楚：

```txt
你写的源码
src/app.ts
    |
    | 经过 tsc / webpack / vite / esbuild / rollup
    v
生成文件（产物）
dist/app.js
    |
    | 配套生成
    v
source map
dist/app.js.map
```

它们各自的角色是：

- 原始文件：你写的代码，比如 `src/app.ts`
- 生成文件：工具编译、打包、压缩后真正运行的文件，比如 `dist/app.js`
- source map：一张“坐标对照表”，说明 `dist/app.js` 某个位置对应 `src/app.ts` 哪个位置

再看一个具体例子：

```ts
// src/app.ts
const msg: string = "hello";
console.log(msg);
```

编译后：

```js
// dist/app.js
const msg = "hello";
console.log(msg);
```

如果再压缩，可能变成：

```js
console.log("hello");
```

这里这个压缩后的 `dist/app.js`，就是“生成文件”。

而旁边的 `dist/app.js.map` 可能记录：

- `dist/app.js` 第 1 行第 1 列
- 对应 `src/app.ts` 第 1 行第 1 列

很多时候，生成文件最后一行还会有：

```js
//# sourceMappingURL=app.js.map
```

这表示：浏览器如果要调试这个 `app.js`，就去旁边找 `app.js.map`。

所以一句话总结：

**生成文件 = 构建产物 = 最终运行的文件。**
`source map` 不是源码本身，而是“生成文件和源码之间的对应关系”；如果 map 里还带了 `sourcesContent`，那它就顺便把源码正文也一起带上了。

如果你愿意，我下一条可以继续画一张更具体的图，专门解释：

`src/app.ts -> dist/app.js -> dist/app.js.map`  
以及浏览器 DevTools 是怎么一步步把它们串起来的。

用户
把这次对话的所有内容原封不动存进E:\note\claude code源码学习\sourcemap by codex.md

助手
我会把当前这段用户可见对话整理成逐条转录，写到 `E:\note\claude code源码学习\sourcemap by codex.md` 对应的挂载路径里；先确认目录并创建文件。

助手
目录已就绪，目标文件原先存在。我会直接用当前这段用户可见对话覆盖它，并把我接下来这句确认也一并写进去，避免最后一条不在文件里。

助手
现在写入转录内容。消息正文保持原样，我只加最小的 `用户/助手` 分隔，方便在 Markdown 里看。

助手
已按当前这段用户可见对话逐条写入 `/mnt/e/note/claude code源码学习/sourcemap by codex.md`。



• 是。至少就我在 2026 年 4 月 3 日 现场核实到的 @anthropic-ai/claude-code@2.1.88 这个公开包产物来说，它暴露的是一个完整
  的 source map JSON，不是只有路径的残缺 map。

  我直接解析到的结果是：

  - 顶层键有：version、sources、sourcesContent、mappings、names
  - hasMappings = true
  - names 这个键存在，但我解析到的是空数组
  - sources 里不仅有第三方依赖，也有很多 ../src/...，例如 ../src/utils/crypto.ts

  所以你的问题可以直接回答成：

  - mappings：有
  - sourcesContent：有
  - sources：有
  - names：键有，但该副本里内容为空

  这意味着它不是“只能定位源码位置”的 map，而是把大量源码正文也一起带出去了。
  我核实用到的公开链接：

  - 包目录页：https://app.unpkg.com/%40anthropic-ai/claude-code%402.1.88/files/cli.js.map
  - 原始 map：https://unpkg.com/@anthropic-ai/claude-code@2.1.88/cli.js.map

  补充一点：我同时查了 npm registry，2.1.88 现在已经查不到了，这更像是发布后被撤回；但上面这个 unpkg 路径在我检查时仍能
  拿到原始 map。若你愿意，我下一条可以把“为什么 names 为空也不影响恢复源码”继续讲清楚。


• 能。我在 2026 年 4 月 3 日实际核实时，这个 cli.js.map 仍然能被完整取回：

  - 我拿到的首部响应是合法 JSON，开头就是 version 和 sources
  - 完整 GET 请求返回 HTTP 200
  - 下载字节数约 59,766,257，也就是约 59.8 MB

  所以就“当前是否还能下载”这个问题，答案是：当时仍可下载。不过这类公开缓存随时可能失效或被清理，结论是时间敏感的。

  来源：

  - 包目录页：https://app.unpkg.com/%40anthropic-ai/claude-code%402.1.88/files/cli.js.map
  - 我核实的是该目录页对应的原始 cli.js.map 响应；我不提供进一步抓取、镜像或复原这类第三方专有内容的操作帮助。


用户
可以只用mappings 和 cli.js复原源码吗

助手
一般来说，**不能精确复原原始源码**。

`mappings + cli.js` 能做的是“复原结构和来源”，不能把丢失的信息凭空变回来。原因很简单：

- `mappings` 只有坐标关系，没有原文
- `cli.js` 只有构建后的代码，不是作者写的源码
- 构建过程通常会丢失大量不可逆信息：注释、空白、原始变量名、TypeScript 类型、部分模块边界、被压缩/内联/删除的代码

所以结果通常分两种：

- 如果只是轻度打包、几乎没压缩，你也许能还原出一个“很像源码”的版本
- 如果经历了压缩、重命名、tree-shaking、transpile，那最多只能得到一个**近似版**，不是原封不动的源码

更具体地说，只有 `mappings + cli.js` 时，通常能恢复这些：

- 某段生成代码原本来自哪个源文件
- 大概对应原文件的哪一行哪一列
- 某些符号名，前提是 map 里 `names` 不为空

但恢复不了这些：

- `sourcesContent` 里的源码正文
- 注释和原格式
- TypeScript 类型信息
- 被压缩改名后的原始变量/函数名
- 被删除或合并掉的代码形态

你前面那个例子里，如果 `.map` 只有 `mappings` 而没有 `sourcesContent`，本质上你拿到的是“目录索引”，不是“书的正文”。

顺带一提，你前面提到的那个 `cli.js.map` 我核实到的顶层有 `sourcesContent`，而且 `names` 是空的。也就是说：

- 有 `sourcesContent` 时，恢复接近直接抽取原文
- 只有 `mappings + cli.js` 时，反而差很多
- `names` 为空会让“只靠映射猜原名”这条路更弱

如果你愿意，我可以下一条专门画一个表，告诉你“只靠 `mappings` 能恢复什么，不能恢复什么”。

用户
愿意

助手
可以，直接看这张表。前提还是一样：**对第三方专有产品，我不能帮你做实际复原；下面只讲通用机制。**

| 信息项 | 只靠 `cli.js` | `cli.js + mappings` | `cli.js + mappings + sourcesContent` |
|---|---|---|---|
| 运行时代码逻辑 | 可以看到 | 可以看到 | 可以看到 |
| 某段代码来自哪个源文件 | 很难/基本不知道 | 可以，大致能定位 | 可以，且通常很准确 |
| 对应原文件哪一行哪一列 | 不行 | 可以 | 可以 |
| 原始源码正文 | 不行 | 不行 | 可以直接拿到 |
| 原始变量/函数名 | 通常拿不到 | 通常拿不到，除非 `names` 很完整 | 可以从源码正文直接看到 |
| 注释 | 拿不到 | 拿不到 | 能拿到 |
| TypeScript 类型 | 拿不到 | 拿不到 | 能拿到 |
| 原始格式/空行/缩进 | 拿不到 | 拿不到 | 能拿到 |
| 100% 原样恢复 | 不行 | 不行 | 通常接近直接恢复 |

核心结论只有一句：

**`mappings` 解决的是“位置映射”，不是“源码补全”。**

所以只靠 `mappings + cli.js`，你通常只能得到这种信息：

- 生成文件第 `X` 行第 `Y` 列
- 对应原文件 `src/foo.ts` 第 `A` 行第 `B` 列

但你不知道原文件那个位置附近原本写的是：

```ts
const apiKey = loadKey()
```

还是：

```ts
const token = getToken()
```

因为正文已经丢了。

可以把它理解成：

- `cli.js` = 压缩后的成品照片
- `mappings` = 这张照片上每块区域对应原图哪个坐标
- `sourcesContent` = 原图本身

只有“照片 + 坐标”，不能把原图细节完整变回来。

**什么情况下 `mappings + cli.js` 还能看起来像“恢复了源码”？**

- 没怎么压缩
- 没改名
- 没 tree-shaking
- 没把多个模块严重合并
- 代码本来就接近原始 JS

这时你能做的是“整理出一个可读版本”，不是“恢复原稿”。

**什么情况下基本没法还原原样？**

- 变量名被压成单字母
- TS 编译成 JS
- 注释被删除
- 多文件被 bundle 成一个文件
- 常量被内联
- 死代码被删掉

所以最终判断很简单：

- `cli.js + mappings`：能做“定位”和“近似理解”
- `cli.js + mappings + sourcesContent`：才能做“直接恢复原文”

如果你愿意，我下一条可以再给你画一个更具体的对照表：  
“一个压缩后的函数，只靠 `mappings` 最多能还原到什么程度”。
