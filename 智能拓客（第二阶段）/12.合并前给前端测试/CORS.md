### Turn 3  *(2026-06-11 01:39)*

**User:**

CORS是什么

**Assistant:**

CORS（Cross-Origin Resource Sharing，跨域资源共享）是浏览器的一套安全机制，用来控制「一个网页能不能调用另一个来源的接口」。

## 背景：浏览器的同源策略

浏览器默认有个规矩叫**同源策略**：网页里的 JS 只能随意请求和自己「同源」的地址。所谓同源 = **协议 + 域名 + 端口**三者完全相同：

- 前端页面跑在 `http://localhost:5173`（Vite dev server）
- 你的后端在 `http://localhost:8000`

域名一样，但**端口不同 → 不同源**。前端 JS 调你的接口就属于「跨域请求」，浏览器默认会把响应拦下来，前端代码根本拿不到数据，控制台报 CORS 错误。

这个限制存在的原因是安全：如果没有它，你登录着网银的同时打开一个恶意网页，那个网页的 JS 就能带着你的 cookie 去请求网银接口并读到返回的数据。

## CORS 怎么解开这个限制

同源策略是浏览器单方面执行的，而 CORS 就是服务端的「授权声明」——服务端在响应头里明确说「我允许某某来源调我」，浏览器看到了就放行。流程是：

1. 浏览器发请求时自动带上 `Origin: http://localhost:5173`（你伪造不了）；
2. 服务端响应里带 `Access-Control-Allow-Origin: *`（或具体来源）；
3. 浏览器对比：在允许名单里 → 把响应交给前端 JS；不在 → 拦截。

另外，像你们这种 `POST` + `Content-Type: application/json` 的请求，浏览器还会先发一个**预检请求**（OPTIONS 方法）探路：「我要用 POST、带 json 头，行不行？」服务端要正确回应预检，真正的 POST 才会发出去。昨天我验证的就是这个，服务端现在的回应：

```
access-control-allow-origin: *
access-control-allow-methods: DELETE, GET, HEAD, OPTIONS, PATCH, POST, PUT
access-control-allow-headers: content-type
```

我加的那个 FastAPI `CORSMiddleware` 干的就是这两件事：响应预检 + 给所有响应加上这些头。

## 两个容易混淆的点

**CORS 只管浏览器。** curl、Postman、后端服务之间互调完全不受影响——所以之前没配 CORS 时我用 curl 测接口照样通，但前端页面一调就会挂。反过来说它也不是接口的访问控制：别人用脚本依然能直接调你的接口，CORS 防的是「恶意网页借用户的浏览器作恶」，不是防爬虫。

**为什么生产环境不用开。** 上线后按 README 的部署方式，前端页面和 API 都由同一个 nginx 对外提供（API 走 `proxy_pass` 反代）——对浏览器来说两者同源，根本不会触发跨域。所以我把开关默认设为关，只在本地联调（前端 dev server 和后端端口不同）时用 `CORS_ORIGINS='*'` 打开。
