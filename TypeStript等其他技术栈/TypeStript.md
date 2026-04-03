# TypeScript 学习笔记

## 学习路径（已会 Python 的情况下）

### 1. 先学 JavaScript 基础（1-2 天）

TypeScript 是 JavaScript 的超集，所以需要先了解 JS：
- 变量：`let`, `const`（类似 Python 的变量，但有块作用域）
- 函数：`function` / 箭头函数 `=>`（类似 Python 的 `lambda`，但更强大）
- 对象和数组（类似 Python 的 `dict` 和 `list`）
- `async/await`（和 Python 的 asyncio 几乎一样）
- 解构赋值、展开运算符

### 2. 学 TypeScript 的类型系统（核心）

这是 TS 相比 JS 的核心区别，类似 Python 的 type hints 但更强大：

```typescript
// Python: def greet(name: str) -> str:
function greet(name: string): string {
  return `Hello, ${name}`;
}

// Python: items: list[int] = [1, 2, 3]
const items: number[] = [1, 2, 3];

// Python: class User(TypedDict): ...
interface User {
  name: string;
  age: number;
  email?: string;  // 可选字段
}
```

### 3. 推荐学习资源

- **官方文档**：typescriptlang.org/docs — 有个很好的 "TS for New Programmers" 章节
- **实践平台**：在 TypeScript Playground（typescriptlang.org/play）在线写代码，即时看效果
- **书**：《TypeScript 编程》（O'Reilly）
- **视频**：Matt Pocock 的 Total TypeScript 系列（YouTube 免费部分就够入门）

### 4. Python vs TypeScript 速查

| Python | TypeScript |
|--------|-----------|
| `def` | `function` / `=>` |
| `dict` | `object` / `Record` |
| `list` | `Array` / `[]` |
| `None` | `null` / `undefined` |
| `f"hello {name}"` | `` `hello ${name}` `` |
| `pip install` | `npm install` |
| `typing.Optional[str]` | `string \| null` |

### 5. 实践建议

1. 装好环境：`npm install -g typescript`，用 `tsc` 编译
2. 用 VS Code 写 TS，自动补全和类型提示非常强
3. 从小项目开始：把你熟悉的 Python 脚本用 TS 重写一遍
4. 直接上手一个小的 Node.js/Next.js 项目练手

有 Python 基础，最大的优势是编程思维已经有了。TypeScript 的语法 1-2 周就能上手，类型系统多用多练就会熟。

---

## 超集（superset）是什么意思

**超集（superset）** 的意思是：所有合法的 JavaScript 代码，都是合法的 TypeScript 代码，但反过来不行。

类比：
- 所有正方形都是长方形，但不是所有长方形都是正方形 — 长方形是正方形的超集

具体来说：

```javascript
// 这段 JS 代码
let name = "hello";
console.log(name);
```

直接放到 `.ts` 文件里，**完全合法**。

但 TypeScript **额外增加了**类型系统：

```typescript
// 这段 TS 代码
let name: string = "hello";  // ": string" 是 TS 独有的
console.log(name);
```

这段代码放到 `.js` 文件里会报错，因为 JS 不认识 `: string`。

所以关系就是：

```
TypeScript = JavaScript + 类型系统（及少量额外特性）
```

这也意味着学 TS 不需要"从零开始"，JS 的知识全部通用，只需要额外学类型部分就行。
