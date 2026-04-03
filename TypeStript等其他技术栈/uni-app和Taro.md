# uni-app是什么

**uni-app** 是一个使用 Vue.js 开发跨平台应用的框架，由国内的 DCloud 公司开发。

## 核心卖点

**写一套代码，发布到多个平台：**

- 微信/支付宝/百度/抖音等小程序
- iOS App
- Android App
- H5 网页
- 快应用

## 技术栈

```
uni-app = Vue.js + 小程序规范 + 跨平台编译
```

你写的是 Vue 语法的代码，uni-app 帮你编译成各平台能运行的代码。

## 类比

| 框架 | 语言 | 目标平台 |
|------|------|---------|
| uni-app | Vue (JS/TS) | 小程序 + App + H5 |
| Flutter | Dart | iOS + Android + Web |
| React Native | React (JS/TS) | iOS + Android |

## 为什么国内很火

1. **小程序生态** — 国内小程序是刚需，uni-app 对各家小程序支持最全
2. **Vue 门槛低** — 国内 Vue 开发者基数大
3. **一套代码多端发布** — 省人力，特别适合中小团队
4. **配套完善** — 有自己的 IDE（HBuilderX）、插件市场、云服务

## 局限

- 跨平台框架的通病：复杂原生功能体验不如原生开发
- 主要在国内生态，国际上几乎没人用
- 深度定制某个平台时，还是得写平台特定代码

## 简单总结

**uni-app = 国内版的跨平台开发框架，主要优势是一套 Vue 代码同时搞定小程序 + App + H5。**

---

# 和Vue 2 + Element UI + echarts相比如何

这两个不是同一层面的东西，不太存在"对比"，更像是**不同场景下的不同选择**：

## 定位完全不同

| | Vue 2 + Element UI + ECharts | uni-app |
|--|------|---------|
| **目标** | PC 端后台管理系统 | 多端应用（小程序/App/H5） |
| **典型产品** | 管理后台、数据大屏、CRM | 微信小程序、移动端 App |
| **运行环境** | 桌面浏览器 | 手机/小程序/移动端浏览器 |
| **UI 组件** | Element UI（为桌面设计） | uView / uni-ui（为移动端设计） |

## 简单判断

- **做 PC 后台管理系统** → Vue + Element UI + ECharts，这是最成熟的方案
- **做小程序或移动端应用** → uni-app

两者**不冲突**，很多公司同时用：后台管理用 Vue + Element UI，面向用户的小程序/App 用 uni-app。

## 如果你会 Vue 2 + Element UI

学 uni-app 会很快，因为：
- 语法都是 Vue
- 模板写法几乎一样（`v-for`、`v-if`、`@click` 等）
- 主要区别是组件不同（`<div>` 换成 `<view>`，`<span>` 换成 `<text>`）和 API 不同（`uni.request()` 代替 `axios`）

**总结：不是谁替代谁的关系，是 PC 端和移动端两个不同战场的工具。**

---

# Taro是什么

**Taro** 是由京东开源的跨平台开发框架，和 uni-app 定位类似，但技术路线不同。

## 核心区别

```
uni-app = Vue 系
Taro    = React 系（也支持 Vue，但主流用 React）
```

## 支持的平台

和 uni-app 基本一样：微信/支付宝/百度/抖音小程序、H5、React Native（App）

## uni-app vs Taro

| | uni-app | Taro |
|--|---------|------|
| **主要语法** | Vue | React |
| **背后公司** | DCloud | 京东 |
| **社区生态** | 国内更大 | 稍小但质量高 |
| **小程序支持** | 更全面 | 主流都支持 |
| **App 方案** | 自有引擎 | React Native |
| **IDE** | HBuilderX | 无专属，用 VS Code |

## 怎么选

很简单：

- **你会 Vue** → uni-app
- **你会 React** → Taro

技术能力上两者差距不大，主要看团队技术栈。

## 对你来说

你目前会 Python，如果之后学的是 Vue 方向，选 uni-app；如果走 React 方向，选 Taro。不需要两个都学，**选一个就够了**。

---

# React是什么

**React** 是由 Meta（Facebook）开发的前端 UI 框架，和 Vue 是同一层面的东西，是目前全球最流行的前端框架。

## Vue vs React

| | Vue | React |
|--|-----|-------|
| **开发者** | 尤雨溪（个人发起） | Meta（Facebook） |
| **国内流行度** | 非常高 | 高 |
| **国际流行度** | 一般 | 最高 |
| **模板写法** | HTML 模板 | JSX（JS 里写 HTML） |
| **学习曲线** | 更平缓 | 稍陡 |

## 写法对比

Vue：
```vue
<template>
  <button @click="count++">{{ count }}</button>
</template>

<script>
export default {
  data() { return { count: 0 } }
}
</script>
```

React：
```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

做的事情一样，风格不同。Vue 更像传统 HTML 分离写法，React 把 HTML 和 JS 混在一起（JSX）。

## React 生态

- **React** — 前端页面（浏览器）
- **React Native** — 用 React 写手机 App
- **Next.js** — React 的全栈框架（前后端一起搞）

## 国内现状

- **Vue 更多** — 中小公司、后台管理系统、小程序（uni-app）
- **React 更多** — 大厂（字节、阿里）、国际化项目

## 对你来说

你只会 Python，两个都没学过的话：
- **想快速上手、做国内项目** → 先学 Vue
- **想进大厂、做国际化项目** → 先学 React

**两者学一个就行，核心思想相通，会了一个另一个很快能上手。**
