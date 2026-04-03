• 阅读原则
  不要一上来扫 app/model 和全部控制器，这个项目太大。先抓骨架，再抓登录权限，再抓 3 到 4 条核心业务链，最后才补异步、集
  成和细枝末节。

  计划清单

  1. 先建立项目全景。先读 README.md、composer.json、VueAdmin/package.json、config/app.php。目标是先确认技术栈、前后端关
     系、默认应用和部署形态。
  2. 再看入口和路由。读 public/index.php、app/adminApi/controller/Index.php、route/app.php、app/adminApi/route/
     route.php。这里要搞清楚一点：adminApi 的业务路由大多是数据库动态加载，不是纯代码静态路由。
  3. 单独梳理登录、权限和菜单。重点读 app/adminApi/controller/system/Login.php、app/adminApi/service/JwtService.php、
     app/adminApi/middleware/TokenCheck.php、app/adminApi/controller/AdminApiBase.php、app/adminApi/controller/system/
     AdminUser.php。读完后你应该能画出“登录 -> token -> 权限 -> 菜单 -> 前端页面”的链路。
  4. 衔接前端。读 VueAdmin/src/main.js、VueAdmin/src/utils/request.js、VueAdmin/src/store/modules/user.js、VueAdmin/src/
     store/modules/permission.js、VueAdmin/src/router/index.js。目标不是记住页面，而是理解前端如何依赖后端菜单数据动态拼
     路由。
  5. 先挑 3 条核心业务线读，不要贪多。推荐顺序是客户、订单、组织架构。
     app/adminApi/controller/customer/Customer.php
     app/service/CustomerService.php
     app/adminApi/controller/order/SaleOrder.php
     app/service/SaleOrderService.php
     app/adminApi/controller/company/BsEmployeeCtrl.php
     app/service/BeisenService.php
     app/service/NccService.php
     这一步只回答 3 个问题：谁发起，状态怎么流转，落哪些表，是否调用外部系统。
  6. 然后补报表、风控和审批。读 app/adminApi/controller/report/OrderReportCtrl.php、app/adminApi/controller/riskCenter/
     CustomerRisk.php、app/service/process/Service/TaskService.php。你会看到这个项目不只是 CRUD，还有审批流和 BI/风控衍
     生表。
     把全项目收敛成一张你的个人地图。

  建议产出
  第一天结束时，你应该能写出一页“系统骨架图”和一页“登录权限链路图”。

  第三天结束时，你应该能说清客户、订单、组织架构这三条链路分别怎么流转。

  一周内，你应该能整理出一份“启动依赖清单 + 队列/定时任务清单 + 风险点清单”。

  不建议的读法

  - 不要先从 app/model 全量扫。
  - 不要先试图读懂全部 139 个 adminApi controller。
  - 不要先碰前端页面细节。
  - 不要忽略数据库配置、缓存刷新和队列命令，这个项目很多真实行为不在单个控制器里。



• 7天阅读任务单

  按“先骨架，后链路，再模块，最后异步和风险”的顺序读。每天的目标不是读完，而是产出你自己的项目地图。

  Day 1
  目标：建立项目全景，搞清前后端关系、默认入口、部署形态、技术栈版本。
  重点文件：README.md composer.json VueAdmin/package.json config/app.php public/index.php app/adminApi/controller/
  Index.php VueAdmin/vue.config.js .example.env
  当天产出：一页“系统概览笔记”，至少写清 后端框架、前端框架、构建产物放哪、默认入口、主要外部依赖。

  Day 2
  目标：吃透登录、鉴权、动态路由、菜单加载。
  重点文件：app/adminApi/route/route.php app/adminApi/controller/system/Login.php app/adminApi/service/JwtService.php
  app/adminApi/middleware/TokenCheck.php app/adminApi/controller/AdminApiBase.php app/adminApi/controller/system/
  AdminUser.php VueAdmin/src/utils/request.js VueAdmin/src/store/modules/user.js VueAdmin/src/store/modules/
  permission.js
  当天产出：一张“登录到页面展示”的链路图，必须写出 登录接口、JWT、AdminToken、权限校验、菜单来源、前端动态路由拼装。

  Day 3
  目标：深读客户主数据链路，只抓核心，不追全部细节。
  重点文件：app/adminApi/controller/customer/Customer.php app/service/CustomerService.php app/model/customer/
  CustomerModel.php app/model/salesman/SalesmanCustomerModel.php app/model/customer/CustomerContactModel.php app/model/
  enterprise/EnterpriseBaseinfoModel.php
  当天产出：一页“客户模块地图”，写清 客户列表怎么查、客户和业务员关系怎么存、客户联系人/地址/企业信息怎么挂、转档/交接/
  审核在哪些方法里。

  Day 4
  目标：深读订单链路，搞清创建、审核、附件、推送外部系统。
  重点文件：app/adminApi/controller/order/SaleOrder.php app/service/SaleOrderService.php app/model/sale/
  SaleOrderModel.php app/model/sale/SaleOrderDetailModel.php app/model/sale/SaleOrderAuditRecord.php app/service/
  PushNccService.php
  当天产出：一张“订单状态机草图”，至少列出 创建、修改、审核、取消、推送NCC、附件处理 这些节点，以及对应的方法名。

  Day 5
  目标：深读组织架构和人事同步链路，搞清 CRM、北森、NCC 的职责边界。
  重点文件：app/adminApi/controller/company/BsEmployeeCtrl.php app/adminApi/controller/company/BsDepartmentsCtrl.php
  app/service/BeisenService.php app/service/NccService.php app/service/MessageQueueUtil.php app/api/controller/
  OrganizationInfo.php
  当天产出：一页“组织架构同步说明”，写清 谁是主数据源、哪些操作要推北森/NCC、消息队列怎么介入、外部系统怎么回拉组织和员
  工信息。

  Day 6
  目标：读异步、定时任务、报表、风控、审批，不求全懂，先建立分类。
  重点文件：config/console.php app/util/ResqueQueue.php app/command/Resque.php app/adminApi/controller/report/
  OrderReportCtrl.php app/adminApi/controller/riskCenter/CustomerRisk.php app/service/process/Service/TaskService.php
  app/adminApi/controller/monitor/Monitor.php
  当天产出：一张“后台任务清单”，分成 定时任务、Beanstalkd 消费者、Resque worker、报表查询、风控数据源、审批流程 六栏。

  Day 7
  目标：收口，补运行依赖、缓存、风险点、对外接口，形成自己的接手文档。
  重点文件：config/database.php config/cache.php config/queue.php app/adminApi/controller/system/OperationForCache.php
  app/api/controller/ExternalApiBase.php app/api/controller/SynOldSrmProduct.php app/api/controller/
  CustomerRiskAssessmentReport.php
  当天产出：一份你自己的 项目接手摘要，至少包含 系统骨架、3条核心业务链、运行依赖、缓存点、异步任务、外部系统、高风险区
  域。

  每天固定动作

  - 阅读前先写 3 个问题，读完必须回答。
  - 每天只允许深挖 1 到 2 个大文件，其他文件只做定位。
  - 每天最后整理一页笔记，不超过 1 屏，避免越记越散。
  - 第二天开始前先回看前一天的图和结论，再继续读。

  每天的完成标准

  - Day 1 完成后，你能说清这是“ThinkPHP 多应用 + Vue 后台 + 动态菜单路由”的项目。
  - Day 2 完成后，你能独立解释一次登录到页面展示的完整链路。
  - Day 3 到 Day 5 完成后，你能说清客户、订单、组织架构三条主链路。
  - Day 6 完成后，你知道哪些业务依赖队列、定时器和审批。
  - Day 7 完成后，你手里有一份能给新同事看的接手文档。
