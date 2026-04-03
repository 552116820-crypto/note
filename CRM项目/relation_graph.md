• 下面按 4 部分给你。先给结论：这是一个“数据库驱动路由/权限 + 多外部系统集成 + 多异步通道”的企业单体 CRM，接手重点不在页面，而在配置、数据、缓存、队列和定时任务链路。config/app.php:13 app/adminApi/route/
  route.php:34

  架构
  后端是 ThinkPHP 6 多应用单体，前端是 Vue 2 + Element UI。VueAdmin 打包后直接输出到后端静态目录 public/adminVueHtml，再由 adminApi/Index/index 作为 SPA 入口渲染。composer.json:22 VueAdmin/package.json:16
  VueAdmin/vue.config.js:31 app/adminApi/controller/Index.php:20

  graph LR
    U[VueAdmin SPA] -->|X-Token /adminApi| A[adminApi]
    U -->|build:prod| P[public/adminVueHtml]
    A --> R[(crm_route + RBAC + AdminToken)]
    A --> S[Service Layer]
    A --> M[(MySQL / MySQL2 / SQLServer)]
    S --> B[Beisen]
    S --> N[NCC]
    S --> W[WeCom]
    S --> O[OSS]
    S --> Q[Qixin / Tianyan / Amap / PLM]
    C[Cron + php think] --> CMD[app/command]
    CMD --> BT[Beanstalkd]
    CMD --> RR[Resque / Redis]
    API[app/api] -->|external systems| A

  - adminApi 的大部分业务路由不是代码静态声明，而是从 crm_route 表加载到文件缓存，再统一套 JWT + TokenCheck。这决定了“菜单、接口、权限”同时受数据库配置控制。app/adminApi/route/route.php:35 app/adminApi/
    middleware/TokenCheck.php:20
  - 前端也不是固定路由树。登录后先取 adminInfo 和 adminMenu，再把后端返回的 path 直接映射到 @/views${item.path} 动态生成前端路由。这是项目最强的耦合点。app/adminApi/controller/system/AdminUser.php:15 VueAdmin/
    src/store/modules/user.js:143 VueAdmin/src/store/modules/permission.js:22
  - 服务层承担了大多数集成职责，BeisenService、NccService、WechatWorkerService、AliyunOssService、AmapService、QixinService、TianyanService 等都在这里。BeisenService.php NccService.php
  - 异步链路是混合的：一部分消息走 Beanstalkd，一部分走 Resque + Redis，大量业务还依赖 php think 定时命令。config/console.php:6 app/service/MessageQueueUtil.php:21 app/util/ResqueQueue.php:11 app/command/
    Resque.php:12

  本地启动与部署清单

  1. 先准备旧版本运行时。后端要求 PHP >=7.1 <7.4，前端是 Vue CLI 4 + Vue 2，更适合 Node 14/16，不建议直接用新环境硬跑。composer.json:22 VueAdmin/package.json:75
  2. 后端 .example.env 只覆盖了最基础的 app/db/lang，远远不够启动；实际还需要 JWT、Redis、北森、NCC、企业微信、短信、邮件、地图、OCR、启信宝、天眼查等环境变量。.example.env app/adminApi/service/
     JwtService.php:16 config/cache.php:7 config/wework.php config/client.php:9
  3. 基础设施至少要有 MySQL、Redis；按业务深度还可能需要 SQL Server/NCC、Beanstalkd、外网访问北森/NCC/企微/OSS 等服务。config/database.php:21 config/queue.php:3 app/service/MessageQueueUtil.php:24
  4. 数据库初始化不能只导表结构，还要带上 crm_route、RBAC、基础配置、部门与员工等核心数据，否则前端菜单和接口会“看起来启动了，但没有业务入口”。app/adminApi/route/route.php:35 app/adminApi/controller/system/
     AdminUser.php:72
  5. 前端构建流程是 VueAdmin 下安装依赖并执行 npm run build:prod，产物会输出到 ../public/adminVueHtml，接口前缀是 /adminApi。VueAdmin/package.json:6 VueAdmin/vue.config.js:32 VueAdmin/.env.production:4
  6. 部署后要刷新两类关键缓存：路由缓存和部门树缓存，否则你改了数据库配置、菜单或组织架构，接口和权限表现会不一致。app/adminApi/controller/system/OperationForCache.php:23
  7. 异步消费者要一起上线。最少要考虑 php think resque start、组织架构同步、客户评级、日报/每小时/每月定时器等命令，不然很多审批、推送、同步、统计都不会落地。config/console.php:6 app/command/Resque.php:16
  8. 线上还要确认系统 crontab，因为项目自带监控模块能读写 crontab，并依赖 shell_exec/exec 获取系统信息和导入任务。app/adminApi/controller/monitor/Monitor.php:45 app/adminApi/controller/monitor/Monitor.php:221
     app/adminApi/controller/monitor/Monitor.php:261
  9. 当前这台机器我没有启动项目。实际检查结果是：php 和 composer 都不在环境里，node 是 v22.22.1、npm 是 10.9.4，这对这个老项目并不理想。

  核心业务模块深挖

  - 客户模块是主数据核心，覆盖客户列表、行业绑定、详情、联系人、地址、转档、交接、创建、付款方式审核等，控制器本身已经是重型业务类。Customer.php:66 Customer.php:536 Customer.php:1538 Customer.php:2054
    Customer.php:2417
  - 订单模块是另一个核心状态机，除了列表、详情、创建、修改、审核，还负责附件、团队成员、取消、推送 NCC、预测批次、安全库存、企知道客户批次等流程。SaleOrder.php:78 SaleOrder.php:957 SaleOrder.php:2425
    SaleOrder.php:3391 SaleOrder.php:4172
  - 组织/人事模块是系统集成中枢。BsEmployeeCtrl 支持员工新增、更新、任职变更、退休、离职、调动；背后再通过 BeisenService、NccService 同步到外部系统。BsEmployeeCtrl.php:54 BsEmployeeCtrl.php:549
    BsEmployeeCtrl.php:909 BsEmployeeCtrl.php:1026 BeisenService.php NccService.php
  - 报表、风控和审批是第三条主线。订单报表直接跨订单、SKU、部门、客户和销售员做组合查询；客户风控依赖 BI 表；导出申请会走自研流程引擎发起审批任务。OrderReportCtrl.php:43 CustomerRisk.php:19 app/adminApi/
    controller/Index.php:35 TaskService.php:62
  - 对外开放 API 也不是边角料。OrganizationInfo 对外提供员工/部门按时间窗口查询，SynOldSrmProduct 接旧 SRM 的产品与价格同步，CustomerRiskAssessmentReport 提供外部触发的风险报告生成。OrganizationInfo.php:23
    SynOldSrmProduct.php:23 CustomerRiskAssessmentReport.php:11
  - 从可维护性看，核心类已经明显“胖控制器/胖服务”了。静态统计里 Customer.php 3773 行，SaleOrder.php 4932 行，BsEmployeeCtrl.php 1484 行，BeisenService.php 1916 行，后续改造应该先拆查询、校验和外部集成。

  安全与风险清单

  - 最高风险是密钥和账号明文入库。仓库里直接包含测试数据库账号、OSS/STS 密钥、企业微信和小程序密钥、客户端 secret 等。config/database.php:111 config/clean.php:6 config/wechat.php:4 config/client.php:9
  - 外部 API 安全模型不统一。ExternalApiBase 支持客户端密钥、限流、可选 IP 白名单，但默认白名单是空；同时另一些接口直接在控制器里硬编码签名密钥，绕开统一网关。ExternalApiBase.php:64 config/client.php:16
    SynOldSrmProduct.php:25 CustomerRiskAssessmentReport.php:13
  - 路由、菜单、权限和缓存都依赖数据库与文件缓存，Git 里无法完整表达真实接口面。部署后如果没刷缓存，表现会和代码不一致。app/adminApi/route/route.php:35 OperationForCache.php:30
  - 技术栈老旧，升级成本高。后端卡在 PHP 7.3 时代，前端还是 Vue 2、Axios 0.18、Vue CLI 4；当前机器上的 Node 22 对这类项目也偏新。composer.json:23 VueAdmin/package.json:20
  - 异步基础设施复杂且有硬编码外部依赖。Beanstalkd 地址直接写死，Resque 又依赖 Redis，定时任务还依赖系统 crontab，排障成本高。MessageQueueUtil.php:24 config/queue.php:3 ResqueQueue.php:45
  - 后台监控模块的权限一旦失守，影响面很大，因为它能操作 crontab、调用 exec/shell_exec、读取日志文件。Monitor.php:45 Monitor.php:229 Monitor.php:277
  - 测试覆盖很弱。我只看到少量 Vue 模板级单测，没有发现后端测试体系；核心风险模块基本靠人工回归和线上经验兜底。VueAdmin/tests/unit/components/Breadcrumb.spec.js
  - 前端还有遗留模板代码，生产环境仍会执行 mockXHR()，虽然当前 mock 的 URL 还是模板默认路径，实际影响有限，但说明清理不彻底。VueAdmin/src/main.js:50
