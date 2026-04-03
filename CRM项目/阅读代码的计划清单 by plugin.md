**阅读目标**
按“先跑通骨架，再读核心链路，最后下钻业务模块”的顺序最省力。这个项目不适合一上来按目录硬啃，因为很多真实路由、菜单和权限都在数据库里，不全在代码里。

**第 1 阶段：先建立全局地图**
- [ ] 看根目录结构，先分清三块：后端应用、前端管理端、公共静态资源。
- [ ] 重点认识这几个目录：
  - [app](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app)
  - [VueAdmin](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin)
  - [config](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config)
  - [public](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/public)
- [ ] 明确这是 `ThinkPHP6 + Vue2` 的同仓项目，不是纯前后端完全拆仓。
- [ ] 记住核心子应用：
  - `app/adminApi`：Vue 后台接口层
  - `app/admin`：旧后台
  - `app/api`：给外部系统的接口层

**第 2 阶段：先读启动入口**
- [ ] 先读后端入口：
  - [public/index.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/public/index.php)
  - [config/app.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/app.php)
  - [route/app.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/route/app.php)
- [ ] 再读应用基础设施：
  - [app/BaseController.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/BaseController.php)
  - [app/AppService.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/AppService.php)
  - [app/provider.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/provider.php)
- [ ] 前端入口按这个顺序看：
  - [VueAdmin/package.json](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/package.json)
  - [VueAdmin/vue.config.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/vue.config.js)
  - [VueAdmin/src/main.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/main.js)

**第 3 阶段：抓主链路，不要先看业务细节**
- [ ] 先搞懂后台首页是怎么出来的：
  - [app/adminApi/controller/Index.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/Index.php)
- [ ] 再看 `adminApi` 的真实路由组织方式：
  - [app/adminApi/route/route.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/route/route.php)
- [ ] 重点理解“动态路由来自数据库”这件事。
- [ ] 搞懂登录、JWT、落库 token、权限判断：
  - [app/adminApi/controller/AdminApiBase.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/AdminApiBase.php)
  - [app/adminApi/controller/system/Login.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/Login.php)
  - [app/adminApi/controller/system/AdminUser.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/AdminUser.php)
  - [app/adminApi/middleware/TokenCheck.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/middleware/TokenCheck.php)
  - [app/adminApi/service/JwtService.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/service/JwtService.php)

**第 4 阶段：前端只读权限和请求骨架**
- [ ] 先不要急着看所有页面，先看前端怎么拿 token、菜单、权限：
  - [VueAdmin/src/utils/request.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/utils/request.js)
  - [VueAdmin/src/permission.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/permission.js)
  - [VueAdmin/src/store/modules/user.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/store/modules/user.js)
  - [VueAdmin/src/store/modules/permission.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/store/modules/permission.js)
  - [VueAdmin/src/api/user.js](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/api/user.js)
- [ ] 搞懂一个关键事实：前端页面路径是按数据库菜单 `path` 动态拼出来的。

**第 5 阶段：补数据库认知**
- [ ] 先只盯住这些表，别一次看太多：
  - `crm_route`
  - `crm_admin`
  - `crm_admin_token`
  - `crm_rbac_permission`
  - `crm_rbac_roles`
  - `crm_rbac_role_has_permissions`
- [ ] 目标不是读字段大全，而是回答这 4 个问题：
  - 路由怎么存
  - 菜单怎么存
  - 按钮权限怎么存
  - 登录态怎么存

**第 6 阶段：按业务域阅读，不要按文件夹穷举**
推荐顺序：
- [ ] `system`：系统、权限、路由、消息
- [ ] `company / bs`：组织、人事、北森同步
- [ ] `customer`：客户主流程
- [ ] `order / rec / sale_outbound_order`：订单、收款、出库
- [ ] `riskCenter / riskControl`：风控
- [ ] `report / salary / finance`：报表、薪酬、财务

对应先看控制器，再回头看 service 和 model：
- [app/adminApi/controller/system](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system)
- [app/adminApi/controller/company](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/company)
- [app/adminApi/controller/customer](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/customer)
- [app/adminApi/controller/order](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/order)

**第 7 阶段：专门读异步、定时任务、外部集成**
- [ ] 看命令注册：
  - [config/console.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/console.php)
- [ ] 看队列和消息：
  - [config/queue.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/queue.php)
  - [app/service/MessageQueueUtil.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/MessageQueueUtil.php)
  - [app/command](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command)
- [ ] 看事件机制：
  - [app/event.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/event.php)
- [ ] 看外部系统接入：
  - [config/database.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/database.php)
  - [config/wework.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/wework.php)
  - [config/client.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/client.php)
  - [config/bs.php](/e:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/bs.php)

**第 8 阶段：阅读输出物**
每读完一轮，强制产出这 5 份笔记：
- [ ] 项目架构图：入口、应用层、服务层、数据层、外部系统
- [ ] 登录与权限链路图
- [ ] 菜单/路由/页面映射表
- [ ] 核心业务域清单
- [ ] 高风险点清单

**建议时间安排**
- 第 1 天：第 1-3 阶段
- 第 2 天：第 4-5 阶段
- 第 3-5 天：第 6 阶段
- 第 6-7 天：第 7-8 阶段

**阅读时最容易踩的坑**
- [ ] 不要以为代码里的路由就是全部路由，很多在 `crm_route`
- [ ] 不要一上来读 `app/service` 全量文件，会很散
- [ ] 不要把 `admin` 和 `adminApi` 混成一套
- [ ] 不要忽略数据库表和配置，它们是这个项目的“半个代码库”

如果你愿意，我下一步可以直接按这个计划，给你做一份“第 1 天阅读任务单”，细到先看哪几个文件、每个文件要回答什么问题。



**使用方式**
每天按这个节奏走最稳：先读 2 小时代码，再花 30 分钟画图/记笔记，最后用 30 分钟回顾“今天我真正搞懂了什么”。不要追求读得多，优先把主链路读通。

**第 1 天**
- 目标：先建立“这是什么项目”的全局印象，不碰细节。
- 先看根目录和依赖定义：[README.md](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/README.md) [composer.json](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/composer.json) [VueAdmin/package.json](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/package.json)
- 再看后端启动入口：[public/index.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/public/index.php) [config/app.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/app.php) [config/route.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/route.php) [route/app.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/route/app.php)
- 看应用基础类：[app/BaseController.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/BaseController.php) [app/AppService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/AppService.php) [app/provider.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/provider.php)
- 认识三套应用分工：[app/admin](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/admin) [app/adminApi](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi) [app/api](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/api)
- 今天必须回答 3 个问题：默认启动的是哪个应用、Vue 后台是怎么挂进来的、`admin` 和 `adminApi` 有什么区别。
- 当日产出：画一张“仓库结构图”，最多 1 页。

**第 2 天**
- 目标：把登录、鉴权、权限、菜单主链路读通。
- 先看 `adminApi` 路由入口：[app/adminApi/route/route.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/route/route.php)
- 再看控制器基础类和登录逻辑：[app/adminApi/controller/AdminApiBase.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/AdminApiBase.php) [app/adminApi/controller/system/Login.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/Login.php)
- 再看 token 中间件和 JWT：[app/adminApi/middleware/TokenCheck.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/middleware/TokenCheck.php) [app/adminApi/service/JwtService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/service/JwtService.php)
- 最后看管理员信息和菜单接口：[app/adminApi/controller/system/AdminUser.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/AdminUser.php)
- 今天必须回答 4 个问题：JWT 里放了什么、数据库 token 为什么还要单独存、菜单数据从哪来、按钮权限是怎么校验的。
- 当日产出：画一张“登录到菜单加载链路图”。

**第 3 天**
- 目标：搞懂前端如何消费后端的权限和菜单。
- 先看前端入口和构建输出：[VueAdmin/src/main.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/main.js) [VueAdmin/vue.config.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/vue.config.js)
- 再看请求封装和路由守卫：[VueAdmin/src/utils/request.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/utils/request.js) [VueAdmin/src/permission.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/permission.js)
- 再看用户与权限 store：[VueAdmin/src/store/modules/user.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/store/modules/user.js) [VueAdmin/src/store/modules/permission.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/store/modules/permission.js)
- 配套看用户接口定义：[VueAdmin/src/api/user.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/api/user.js)
- 快速扫一遍静态路由：[VueAdmin/src/router/index.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/router/index.js)
- 今天必须回答 4 个问题：前端 token 放哪、什么页面是静态路由、什么页面是动态路由、数据库 `path` 为什么能直接映射到前端页面。
- 当日产出：整理一份“前端菜单加载 8 步流程”。

**第 4 天**
- 目标：先啃组织、人事、北森这条基础业务线，因为它是很多模块的前置依赖。
- 先看配置和同步背景：[config/bs.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/bs.php) [app/adminApi/controller/Index.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/Index.php)
- 重点看人员和部门控制器：[app/adminApi/controller/company/BsEmployeeCtrl.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/company/BsEmployeeCtrl.php)
- 再看组织同步核心服务：[app/service/BeisenService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/BeisenService.php) [app/service/BeisenPullService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/BeisenPullService.php)
- 配套看组织相关模型目录：[app/model/bs](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/model/bs) [app/model/corp](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/model/corp)
- 如果有余力，再看组织同步命令：[app/command/AsyncOrganizationalStructureCommand.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command/AsyncOrganizationalStructureCommand.php) [app/command/PullOrganizationalStructureCommand.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command/PullOrganizationalStructureCommand.php)
- 今天必须回答 4 个问题：员工主数据来自哪里、CRM 如何保存组织树、北森/NCC/企业微信在组织上是什么关系、为什么组织同步要走队列。
- 当日产出：画一张“组织与员工主数据流转图”。

**第 5 天**
- 目标：进入 CRM 主业务，优先读客户、订单、收款三条线。
- 先读客户控制器目录：[app/adminApi/controller/customer](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/customer)
- 配套看客户服务：[app/service/CustomerService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/CustomerService.php)
- 再读订单和收款控制器目录：[app/adminApi/controller/order](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/order) [app/adminApi/controller/rec](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/rec) [app/adminApi/controller/sale_outbound_order](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/sale_outbound_order)
- 配套看订单服务：[app/service/SaleOrderService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/SaleOrderService.php) [app/service/SaleRecService.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/SaleRecService.php)
- 再扫前端对应接口文件，建立前后端映射：[VueAdmin/src/api/customer.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/api/customer.js) [VueAdmin/src/api/sale_order.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/api/sale_order.js) [VueAdmin/src/api/rec.js](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/VueAdmin/src/api/rec.js)
- 今天必须回答 5 个问题：客户的核心实体有哪些、订单主流程怎么走、收款和订单怎么关联、哪些地方开始出现风控/审批、前端哪个页面对应哪个接口。
- 当日产出：整理一份“客户到订单到收款”的主流程清单。

**第 6 天**
- 目标：补齐异步、队列、定时任务、审批流程这几个“运行机制”。
- 先看命令注册：[config/console.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/console.php)
- 再看消息队列和 Redis 配置：[config/queue.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/queue.php) [app/service/MessageQueueUtil.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/MessageQueueUtil.php)
- 重点挑 4 个命令看：[app/command/AsyncDataExportCommand.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command/AsyncDataExportCommand.php) [app/command/CustomerEvaluateCommand.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command/CustomerEvaluateCommand.php) [app/command/RecAssignAndBalanceCommand.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command/RecAssignAndBalanceCommand.php) [app/command/EveryDayAt1Timer.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/command/EveryDayAt1Timer.php)
- 再看审批流程子系统：[app/service/process](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/service/process)
- 配套看事件系统：[app/event.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/event.php) [app/listener](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/listener)
- 今天必须回答 4 个问题：哪些功能是同步请求，哪些功能是异步队列，审批是独立子系统还是业务附属能力，定时任务是怎么落地到服务器的。
- 当日产出：做一张“同步/异步/定时”运行机制图。

**第 7 天**
- 目标：收口阅读成果，建立你的个人项目认知框架。
- 补读外部系统接入层：[app/api/controller/ExternalApiBase.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/api/controller/ExternalApiBase.php) [config/client.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/client.php) [config/wework.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/wework.php) [config/database.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/config/database.php)
- 再读监控和运维耦合点：[app/adminApi/controller/monitor/Monitor.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/monitor/Monitor.php)
- 回头看一遍你这周最常碰到的表和模型目录：[app/model](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/model)
- 把全局结论整理成 4 份固定文档：
  - 架构图
  - 登录与权限图
  - 核心业务域地图
  - 风险与技术债清单
- 今天必须回答 5 个问题：这个系统最核心的 3 个模块是什么、最难维护的 3 个模块是什么、最依赖数据库配置的地方是什么、最依赖外部系统的地方是什么、如果你明天要改需求会先避开哪些区域。
- 当日产出：写一份 1 到 2 页的“项目总览摘要”。

**每天都要做的固定动作**
- 读完一个核心文件后，写一句话总结它的职责。
- 遇到不懂的表名、模型名、接口名，立刻建术语表。
- 每天至少画一张图，不要只看文字。
- 每天结束前，把“今天读过但没读懂的点”单独列出来。

**这 7 天结束后的达标标准**
- 你能口头讲清登录和菜单权限链路。
- 你知道前端页面和后端接口是如何对上的。
- 你能说出客户、订单、收款、组织同步各自的主入口文件。
- 你能判断一个新需求大概应该改 `controller`、`service`、`model`、前端 `api` 还是数据库配置表。

如果你愿意，我可以继续把这份任务单再细化成“每天上午看什么、下午看什么”的打卡版。