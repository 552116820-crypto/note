# 原生开发，WXML + WXSS + JavaScript/TypeScript，这个是什么

这是**微信小程序的原生开发技术栈**。

## 逐个解释

| 技术 | 对应 Web 技术 | 作用 |
|------|-------------|------|
| **WXML** | HTML | 页面结构（微信自定义的标记语言） |
| **WXSS** | CSS | 页面样式（微信自定义的样式语言） |
| **JS/TS** | JS/TS | 页面逻辑（这个没变） |

本质就是微信把 HTML/CSS **换了个名字，加了点限制和扩展**。

## 举个例子

Web 写法：
```html
<div class="box">
  <span>{{ message }}</span>
</div>
```

微信小程序原生写法：
```xml
<view class="box">
  <text>{{ message }}</text>
</view>
```

几乎一样，就是标签名不同（`div` → `view`，`span` → `text`）。

## "原生"是相对于谁说的

| 方式 | 说明 |
|------|------|
| **原生开发** | 直接用微信提供的 WXML/WXSS/JS，只能跑微信小程序 |
| **uni-app / Taro** | 用 Vue/React 写，框架帮你编译成 WXML/WXSS/JS |

就像你可以直接写汇编（原生），也可以写 Python 让解释器帮你转（框架）。

## 优缺点

- **优点**：性能最好、和微信官方文档完全对应、没有框架中间层的坑
- **缺点**：只能跑在微信，想同时出支付宝/抖音小程序就得重写

## 总结

**WXML + WXSS + JS/TS = 微信小程序的"方言版" HTML + CSS + JS，是最直接但只能用于微信的开发方式。** 用 uni-app/Taro 则可以避免被微信绑定。
