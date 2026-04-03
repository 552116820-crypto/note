• 原因找到了，登录本身是成功的，这不是鉴权问题，是前端国际化配置有 bug。

  根因在 VueAdmin/src/lang/index.js:17：

  - 项目实际只启用了 zh / vi / idn
  - 但 getLanguage() 之前在匹配不到时默认返回 en
  - 一旦浏览器语言或旧 cookie 是 en，ElementUI 找不到对应中文词条，就会把 key 直接显示出来，比如 el.pagination.pageClassifier

  我已经改了：

  - 不再默认回退到 en
  - 只接受项目支持的语言
  - 旧的无效语言值自动回退到 zh

  前端 dev 服务已经重新编译好了。你现在直接：

  1. 在页面按 Ctrl+Shift+R 强刷
  2. 如果还显示英文 key，就删掉浏览器里这个 cookie：language
  3. 再打开 http://localhost:9528/adminVueHtml/

  正常情况下，那个位置现在应该显示成“页”，不会再显示 el.pagination.pageClassifier。
