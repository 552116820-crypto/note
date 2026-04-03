• 已完成。后端聚合和导出在 app/adminApi/controller/report/CustomerOrderDailyReportCtrl.php:10，按日期统计有效订单量、去重客户数、订单金额，并支持 __export=1 导出；核心 SQL 口径和日期校验也写在同文件里。前端报
  表页在 VueAdmin/src/views/report/customer_order_daily.vue:1，包含日期区间、部门、业务员筛选，汇总卡片、表格渲染和 Excel 导出；接口接入和直达路由在 VueAdmin/src/api/report_table.js:173 和 VueAdmin/src/router/
  index.js:306。

  Docker 验证也跑完了。测试种子和脚本在 docker/test/init_customer_order_daily_report_test.sql:1 与 docker/test/e2e_customer_order_daily_report.sh:1，执行结果是 PASS=21 FAIL=0。另外用 PHP 容器跑了语法检查，
  CustomerOrderDailyReportCtrl.php 无语法错误；再用临时 node:14 容器执行了 npm run build:prod，生产构建成功，产物已刷新到 public/adminVueHtml/。后端路由注册 SQL 单独放在 docker/sql/
  customer_order_daily_report_seed.sql。

  一个必要说明：当前 Docker 测试库里没有 crm_rbac_permissions_vue / 角色权限表，所以我没有补侧边栏菜单权限种子；页面现在可直接通过 /adminVueHtml/#/report/customer_order_daily 访问。如果你还要把它挂进正式菜单，
  我下一步可以继续补权限菜单 SQL。


 404 已修复。

  根因不是控制器缺失，而是 adminApi 这里有一层数据库驱动的动态路由缓存。当前运行环境按 .env 连接的库里没有同步到 CustomerOrderDailyReport/list，所以请求在进鉴权前就被路由层直接打成了 RouteNotFoundException。我
  已经在 app/adminApi/route/route.php:35 把这条报表接口补成了静态注册，和现有 SalesRankingReport/list 同级，避免再依赖 crm_route 是否已同步。

  报表功能本身已经就位: 后端聚合和导出在 app/adminApi/controller/report/CustomerOrderDailyReportCtrl.php:10，前端页面在 VueAdmin/src/views/report/customer_order_daily.vue:1，接口封装在 VueAdmin/src/api/
  report_table.js:173，页面路由在 VueAdmin/src/router/index.js:306。页面入口是 /adminVueHtml/#/report/customer_order_daily。

  验证结果:

  - 现在访问 /adminApi/CustomerOrderDailyReport/list 已不再返回 404，而是正常进入鉴权，返回 50008 token不能为空。
  - Docker 端到端脚本 docker/test/e2e_customer_order_daily_report.sh:1 已重新跑通，结果是 PASS=21 FAIL=0。

  如果你还要让这个页面出现在左侧菜单，而不是仅通过直链访问，还需要在正式库补一条 crm_rbac_permissions_vue 菜单/权限配置。




 9528 这边我已经重启过前端开发容器，并确认当前运行中的 bundle 里已经有这条路由:
  VueAdmin/src/router/index.js:306

  同时 9528 代理出去的后端接口也已经不是 404 了，/adminApi/CustomerOrderDailyReport/list 现在返回的是鉴权提示 50008 token不能为空，说明前后端链路都在。

  你现在按这个顺序试:

  1. 打开 http://localhost:9528/adminVueHtml/#/login
  2. 先登录
  3. 登录后再手动打开 http://localhost:9528/adminVueHtml/#/report/customer_order_daily

  这个项目登录后不会按 redirect 回原页面，而是固定跳到 /dashboard，代码在:
  VueAdmin/src/views/login/index.vue:524
  VueAdmin/src/permission.js:52

  如果你还是看到 404，先对 localhost:9528 做一次 Ctrl+F5 强刷。因为我这边当前 9528 返回的 bundle 已经明确包含:

  - path: '/report/customer_order_daily'
  - title: '客户订单日报表'
  - url: '/CustomerOrderDailyReport/list'


• 不是 mock。现在这个报表读的是实际订单表 crm_sale_order 的 tax_amount 字段，并直接做汇总，代码在 app/adminApi/controller/report/CustomerOrderDailyReportCtrl.php:45 和 app/adminApi/controller/report/
  CustomerOrderDailyReportCtrl.php:81。系统其他地方也把这个字段明确当成“订单金额(含税)”来用，见 app/adminApi/controller/order/SaleOrder.php:387。

  但它不一定能直接理解成“人民币元”。订单本身有币种字段 nc_currency_code，订单详情页也单独展示“订单币别”，见 app/adminApi/controller/order/SaleOrder.php:389 和 VueAdmin/src/views/sale_order/
  sale_order_details.vue:56。当前日报表没有做币种过滤，也没有折算成人民币，只是直接 SUM(o.tax_amount)。所以：

  - 如果你当前库里的订单全是 CNY，那可以按人民币元理解。
  - 如果库里混有美元、欧元等非 CNY 订单，那现在这个“金额”就是混合币种汇总，不是严格的人民币总额。

  另外，之前我跑 Docker E2E 用的测试环境已经恢复回原 .env 了，见 docker/test/e2e_customer_order_daily_report.sh:24。所以你现在登录后看到的，是当前环境库里的真实数据，不是我插进去的测试种子。

  如果你要这个报表严格按人民币口径，我可以继续改成这三种之一：

  1. 只统计 nc_currency_code = 'CNY'
  2. 增加“币种”列并按币种分组
  3. 按汇率统一折算成人民币后再汇总
