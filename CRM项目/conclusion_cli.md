• 结论

  这是一个典型的“强业务耦合企业内部单体 CRM”项目，不是通用模板项目。它的核心形态是：

  - 后端：ThinkPHP 6 多应用单体，PHP 版本被限制在 <7.4，依赖很多企业集成与队列组件，composer.json:22
  - 前端：Vue 2 + Element UI 的后台 SPA，构建产物直接输出到后端 public/adminVueHtml 目录，由后端渲染入口页，VueAdmin/package.json:16 VueAdmin/vue.config.js:31 app/adminApi/controller/Index.php:20
  - 权限和菜单：高度动态化，接口路由、左侧菜单、按钮权限都和数据库配置强绑定，不是靠纯代码静态定义，app/adminApi/route/route.php:34 app/adminApi/controller/AdminApiBase.php:99 app/adminApi/controller/system/
    AdminUser.php:72

  从规模看，它已经不是“小 CRM”了。我本地静态统计到：

  - app/adminApi/controller 139 个 PHP 文件
  - app/service 127 个 PHP 文件
  - app/model 374 个 PHP 文件
  - app/command 73 个命令脚本
  - VueAdmin/src/views 387 个前端页面文件

  架构

  后端默认应用还是老的 admin，说明项目保留了传统服务端渲染后台；同时又有 adminApi 这套新接口层给 Vue 后台使用，config/app.php:13 app/admin/middleware.php:2 app/adminApi/middleware.php:2。也就是说这是“旧后台 +
  新后台并存”的演进式代码库。

  登录链路是：账号/手机/邮箱/企业微信登录 -> 生成 adminToken 落库 -> 再包装成 JWT 给前端，app/adminApi/controller/system/Login.php:17 app/adminApi/service/JwtService.php:16。接口侧统一从 X-Token 里验 JWT，并再
  回查 AdminTokenModel 和权限，app/adminApi/middleware/TokenCheck.php:20。前端则把 token 放 cookie，每次请求自动加 X-Token，VueAdmin/src/utils/request.js:18。

  最关键的耦合点是“菜单即路由即页面路径”。后端把菜单树放进登录态和 adminMenu 返回；前端再把返回的 path 直接拼成 @/views${item.path} 动态加载页面，app/adminApi/controller/system/AdminUser.php:72 VueAdmin/src/
  store/modules/permission.js:22。这意味着数据库里的路由配置、前端 views 文件路径、按钮权限节点三者必须严格对齐，否则页面直接打不开。
	
  - company：组织、人事、团队、销售角色、队列日志
  - customer：客户、标签保护、黑名单、维护、数据审核
  - product：产品、样品单、服务申请、投诉、回收桶
  - order / sale_outbound_order / rec：订单、收款、出库、核销
  - report / budget / salary / riskCenter：报表、预算、薪酬、风控
  - system / logcenter / monitor / process：系统管理、日志中心、监控、审批流程

  另外它内置了一个自研审批引擎，不只是普通 CRUD。导出申请等功能会发起审批任务，app/adminApi/controller/Index.php:27 app/service/process/Service/TaskService.php:56。

  依赖与风险

  这个项目的运行依赖明显偏重：

  - MySQL 主库 + 第二 MySQL + SQL Server/NCC 库，config/database.php:21
  - 定时任务大量依赖 php think ... 命令和系统 crontab，监控模块甚至能读/写 crontab 并执行系统命令，app/adminApi/controller/monitor/Monitor.php:45 app/adminApi/controller/monitor/Monitor.php:221 app/adminApi/
    controller/monitor/Monitor.php:277

  最大的问题不是代码风格，而是“配置安全性和运维耦合”。

  - 仓库里直接有测试数据库账号密码，config/database.php:111
  - 直接提交了 OSS/STS、WMS/TMS、企业微信、小程序等密钥，config/clean.php:6 config/wechat.php:4 config/client.php:9
  - 前端技术栈明显老旧：Vue 2.6、axios 0.18、vue-cli 4.4，VueAdmin/package.json:16
  - main.js 里生产环境仍会启用 mock 代码，虽然当前 mock URL 看起来还是模板默认路由，实际影响不大，但这是明显遗留物，VueAdmin/src/main.js:50
  - 我只发现了少量 Vue 模板级单测，没有发现项目级 PHP 测试体系；这类项目改动风险主要靠经验和线上验证兜底，Breadcrumb.spec.js

  接手建议

  如果你是要继续维护这个项目，我建议先做这 4 件事：

  - 先把“运行基线”摸清：数据库、Redis、Beanstalkd、邮件、短信、企业微信、OSS 哪些是必须的，哪些能 。
  - 先导出并审视 crm_route、权限表、角色表。这个项目的真实接口面不在代码里全量静态可见，而在数据库。
  - 先清理配置安全问题，把仓库内硬编码密钥全部迁到环境变量。
  - 先对高频模块建立最小回归清单：登录、菜单加载、客户维护、订单、收款、报表、队列、定时任务。
