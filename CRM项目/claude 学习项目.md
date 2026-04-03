● 已完成项目学习。以下是项目概况：

  CRM管理后台 2.0

  技术栈

  - 后端: ThinkPHP 6 (PHP 7.1~7.3)
  - 前端: Vue 2 + Element UI + Vuex + Vue Router (基于 vue-admin-template)
  - 数据库: MySQL 5.7 (主库) + SQL Server (NCC对接) + Redis (缓存)
  - 消息队列: Beanstalkd (组织架构异步同步)
  - 容器化: Docker Compose (mysql / redis / beanstalkd / php / nginx / node14)

  项目结构

  - app/adminApi/controller/ — 后台API控制器，按业务模块分目录（customer、company、finance、order、salary 等 30+ 模块）
  - app/model/ — 数据模型层，50+ 子目录涵盖客户、员工、财务、签约、薪资等
  - app/service/ — 服务层，包含 NCC/北森/天眼查/企信/阿里OSS/企业微信 等第三方集成
  - app/command/ — 70+ CLI 命令，处理定时任务、数据同步、异步导出、风控分析等
  - VueAdmin/src/ — 前端 SPA，views 50+ 页面模块，api/ 下 60+ 接口模块

  核心业务

  1. 客户管理 — 客户档案、品牌、黑名单、风控评估、客户公海
  2. 组织架构 — 北森(BeiSen)同步员工/部门/职位，通过 Beanstalkd 异步推送到 NCC 等系统
  3. 财务模块 — 收款、账单、费用、汇率
  4. 签约管理 — 合同签约流程
  5. 薪资绩效 — 薪资计算、业绩目标、OKR
  6. RBAC权限 — 基于角色的菜单/权限控制，JWT认证
  7. 报表/BI — 多维度数据报表
  8. 企业微信集成 — WeWork JS-SDK 对接

  API约定

  - 成功响应 code 20000，失败 50000
  - AdminApiBase 为所有后台控制器基类，自动处理 token 认证、操作日志、权限加载
