**实机验证结果**
我已经把这个项目跑过一轮真实验证，不只是静态看代码。

- 已连通测试库 `amer_crm_v2`，数据库版本是 `MySQL 5.7.38-log`，库里约 `851` 张表。
- 已确认队列端口 `47.114.77.122:11300` 可达。
- 我用临时便携版 `PHP 7.3.33` 在本机启动了项目，并成功提供 HTTP 服务。
- `GET /adminApi/` 能正常返回 Vue 管理端入口 HTML，对应入口实现见 [Index.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/Index.php#L24)。
- 未登录访问 `GET /adminApi/AdminUser/adminInfo` 返回 `50008`，说明鉴权中间件生效，对应 [TokenCheck.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/middleware/TokenCheck.php#L22)。
- 登录失败链路已验证：`POST /adminApi/Login/accountLogin` 对不存在账号返回业务错误 JSON，对应 [Login.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/Login.php#L18)。
- 我写入了一条临时测试登录态后，受保护接口能正常访问：
  - `GET /adminApi/AdminUser/adminInfo` file:///E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/AdminUser.php#L15返回 `20000`，拿到了角色、权限、管理员信息，对应 [AdminUser.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/AdminUser.php#L15)。
  - `GET /adminApi/AdminUser/adminMenu` 返回系统菜单数据，对应 [AdminUser.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/AdminUser.php#L72)。
  - `GET /adminApi/Route/index?request_mode=all&page=1&limit=2` 返回成功，`total=984`，说明动态路由表和查询接口正常，对应 [Route.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/system/Route.php#L15)。
  - `GET /adminApi/BsEmployee/list?page=1&limit=2` 返回成功，`total=7055`，说明真实业务数据查询正常，对应 [BsEmployeeCtrl.php](E:/crm_system_dev/crm_manage_system_2.0_full-dev-3a3022f1d0335a18abd1929f58ab8b1952124575/app/adminApi/controller/company/BsEmployeeCtrl.php#L44)。
  - `POST /adminApi/Login/logout` 成功后，再访问 `adminInfo` 返回 `50012 账号已被登出`，说明登出失效链路正常。

**补充结论**
这说明项目当前至少有这几层是通的：数据库连接、ThinkPHP 启动、`adminApi` 路由、JWT/Token 鉴权、菜单权限接口、系统路由配置接口、北森员工业务查询接口。

这次验证没有保留临时痕迹：我已删除临时 `.env`、停止本地 PHP 服务、清理插入到 `crm_admin_token` 的测试 token。唯一没动的是系统全局环境；临时 PHP 和 Node 验证工具留在 `%TEMP%`，不在项目目录里。

**还没做的**
- 没用真实账号密码做正式登录，只验证了失败链路和临时登录态链路。
- 没跑 `php think` 命令型任务，因为这台机器原生没有 PHP 运行时，我是用便携 PHP 只做了 Web 链路验证。
- 没验证短信、邮件、企业微信、Beanstalk 消费者实际业务执行。

下一步最值得继续验的是三条高频链路：客户、订单、风控。