• 会重复，而且不应假设我对这个仓库有“跨会话持久记忆”。

  更准确地说：

  - 同一个会话里，我会保留上下文，后续任务不用每次从零读。
  - 换一个新会话窗口后，我通常需要重新建立项目认知。
  - 所以如果你经常开新会话，最有效的做法不是指望我记住，而是给我一套“低成本重新上手”的入口文件。

  对这个项目，如果目标是“最快理解并开始干活”，我会优先读这几类内容。

  第一层：全局手册

  - AGENTS.md
  - CLAUDE.md
  - README.md
  - DOCKER.md

  这层解决“这是什么项目、怎么跑、哪些规则不能踩”。

  第二层：技术入口

  - composer.json
  - VueAdmin/package.json
  - VueAdmin/vue.config.js
  - compose.yaml

  这层解决“后端/前端/容器分别怎么组织”。

  第三层：真正的业务入口

  - app/BaseController.php
  - app/adminApi/controller/AdminApiBase.php
  - app/adminApi/route/
  - VueAdmin/src/utils/request.js
  - VueAdmin/src/router/
  - VueAdmin/src/api/
  - VueAdmin/src/views/

  这层解决“接口怎么进来、权限怎么走、前端怎么调接口、页面和 API 怎么对上”。

  如果是具体任务，我会再按类型补读：

  - 写接口：先看 controller、对应 validator、相关 model/service、路由加载方式、前端 src/api/ 调用点。
  - 优化前端：先看目标页面 src/views/、对应 src/api/、公共组件、request.js、路由和权限逻辑。
  - 查链路问题：再看 app/command/、app/job/、app/listener/、外部服务封装。

  想减少每次新会话重复读取，最有效的是把文档分层，而不是只放一个大手册：

  1. AGENTS.md
     内容放全局规则、目录地图、开发命令、禁改项、验证清单。
  2. docs/backend.md
     只写后端入口、接口约定、路由/权限、命令/队列链路。
  3. docs/frontend.md
     只写前端目录、API 封装、页面组织、构建输出、常见坑。
  4. docs/modules/*.md
     给高频模块单独做索引，比如 customer.md、company.md、finance.md。

  这样我新开会话时，不需要重读整个项目，只需要：

  先读 AGENTS.md，再根据任务类型读 docs/backend.md 或 docs/frontend.md；
  如果任务属于 customer/company/finance，再读对应模块文档；
  然后再进具体代码。

  最关键的一点是：把 /init 做成固定启动词。比如：

  Read AGENTS.md first.
  Then:
  - if task is backend, read docs/backend.md
  - if task is frontend, read docs/frontend.md
  - if task mentions a module, read docs/modules/<module>.md
  Summarize the relevant architecture before editing.

  这样我每个新窗口都会按同一套路快速建模，不会靠临时搜索乱读。

  如果你要，我可以直接继续帮你做这套结构：

  - 重构 AGENTS.md 成真正的全局入口
  - 新增 docs/backend.md
  - 新增 docs/frontend.md
  - 再给你生成一份可复制的 /init 模板
