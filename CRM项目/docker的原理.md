docker相关文件的作用
docker compose


• 先把几个概念摆正：Dockerfile 是“镜像配方”，镜像是“运行模板”，容器是“镜像启动后的进程实例”，docker compose 是“把多个容器按一个应用来编排”的工具。

  Docker 容器原理
  Docker 容器本质上不是虚拟机。它没有自己完整的内核，而是共享宿主机的 Linux 内核，只是把进程运行环境隔离出来。

  - 隔离靠 namespace：把进程看到的进程树、网络、挂载点、主机名、用户等隔离开。
  - 资源限制靠 cgroups：限制 CPU、内存、IO。
  - 文件系统靠联合挂载：镜像由多层只读 layer 叠起来，容器启动时再加一层可写层。
  - 安全边界还会用 capabilities、seccomp、AppArmor/SELinux 一类机制。
  - 网络通常是虚拟网卡 veth + Linux bridge，容器之间走虚拟网络，端口映射把宿主机端口转发进容器。

  所以容器更像“被隔离和打包的进程”，不是“缩小版虚拟机”。

  如果你在 Windows + Docker Desktop + WSL2 下跑 Linux 容器，真实情况是：

  - Docker Engine 跑在 WSL2/Linux 环境里。
  - 容器仍然是 Linux 进程隔离。
  - 不是直接跑在 Windows 内核上。

  Docker 是怎么实现一次运行的
  以这个项目为例，流程大致是：

  1. docker build 读取 docker/php/Dockerfile:1。
  2. BuildKit 先拉基础镜像元数据，比如 php:7.3-fpm-buster 和 composer:2。
  3. Dockerfile 每执行一条 RUN、COPY，都会生成新的镜像层。
  4. 最后得到一个自定义 PHP 镜像。
  5. docker run 或 docker compose up 再基于镜像创建容器：
      - 加一层可写层
      - 挂载卷
      - 配网络
      - 注入环境变量
      - 启动主进程

  这里主进程如果退出，容器通常也就退出了。容器的生命周期跟 PID 1 绑定。

  docker-compose / docker compose 是什么
  它们是同一个东西的新旧命令形态：

  - docker-compose：旧的独立二进制。
  - docker compose：新的 Docker CLI 插件写法。

  作用是把“一个应用要用到的多个容器”写进一个 YAML 文件里统一管理。比如这个项目的 compose.yaml:1 就定义了 MySQL、Redis、PHP、Nginx、Beanstalkd、Node14。

  Compose 会帮你做这些事：

  - 创建默认网络，服务名可直接互相解析成 DNS 名。
  - 根据 build 构建镜像，或根据 image 拉现成镜像。
  - 创建并启动多个容器。
  - 挂载卷、注入环境变量、映射端口。
  - 管理依赖顺序、健康检查、重启策略。

  这个项目的整体关系
  这个项目的容器拓扑其实很标准：

  浏览器 -> Nginx -> PHP-FPM -> MySQL / Redis / Beanstalkd

  - nginx 对外提供 HTTP。
  - php 容器里跑 PHP-FPM，负责执行业务代码。
  - mysql 存业务数据。
  - redis 做缓存/队列辅助。
  - beanstalkd 是任务队列服务。
  - node14 只是前端构建工具容器，不是主服务。

  compose.yaml:1 是干什么的
  这个文件是“整个多容器应用的总控台”。

  关键段落可以这样理解：

  - mysql 服务 compose.yaml:2
      - 用 mysql:5.7.26 镜像。
      - environment 初始化 root 密码、库名、普通用户。
      - ports 把宿主机 ${MYSQL_PORT:-3307} 映射到容器 3306。
      - volumes 把数据库数据放进命名卷 mysql57_data，避免删容器就丢数据。
      - healthcheck 确保 MySQL 真正可用，不只是进程启动。
  - redis 服务 compose.yaml:25
      - 用 redis:7-alpine。
      - command 开启 AOF 持久化，并设置密码。
      - redis_data 是持久化卷。
      - 同样有健康检查。
  - beanstalkd 服务 compose.yaml:39
      - 不是直接拉现成镜像，而是用 docker/beanstalkd/Dockerfile 构建。
      - 对外暴露 11300。
  - php 服务 compose.yaml:47
      - 用 docker/php/Dockerfile:1 构建自定义 PHP 运行时。
      - working_dir: /var/www/html 表示容器内工作目录。
      - volumes: ./:/var/www/html 把项目源码挂进去，代码改了容器里立即可见。
      - depends_on 让它等 MySQL/Redis/Beanstalkd。
      - extra_hosts 里的 host.docker.internal:host-gateway 很关键，容器内可以通过这个名字访问宿主机。你这个项目的 NCC SQL Server 就是这么接宿主机/外部环境的。
  - nginx 服务 compose.yaml:68
      - 用官方 nginx:1.27-alpine。
      - 绑定宿主机 ${NGINX_PORT:-8080} 到容器 80。
      - 同时挂载项目代码和 Nginx 配置文件。
  - node14 服务 compose.yaml:79
      - 只是给 VueAdmin 跑 npm install / npm run build 的工具容器。
      - profiles: ["tool"] 表示它不是默认常驻服务。
  - volumes compose.yaml:87
      - 声明命名卷 mysql57_data、redis_data。
      - 这类卷由 Docker 管理，适合存数据库和缓存持久化数据。

  docker/php/Dockerfile:1 是干什么的
  这是 PHP 容器镜像的构建配方，作用是“把项目需要的 PHP 运行环境一次性装好”。

  逐段看：

  - FROM php:7.3-fpm-buster docker/php/Dockerfile:1
      - 以官方 PHP 7.3 FPM Debian Buster 镜像为基础。
      - 说明这个项目是老 PHP 环境，不是 PHP 8。
  - ARG DEBIAN_FRONTEND=noninteractive docker/php/Dockerfile:3
      - 安装系统包时避免交互式提示。
  - 大段 RUN apt-get ... docker/php/Dockerfile:5
      - 安装系统依赖，比如 git、curl、mysql client、unzip。
      - 装编译 PHP 扩展要用的开发库，比如 libpng-dev、libjpeg...、libzip-dev。
      - 拉微软源并安装 msodbcsql17，这是为了 SQL Server。
      - 编译并启用 PHP 扩展：bcmath、gd、intl、mbstring、mysqli、pdo_mysql、soap、zip 等。
      - 通过 PECL 安装额外扩展：redis、imagick、sqlsrv、pdo_sqlsrv。
  - COPY --from=composer:2 /usr/bin/composer /usr/local/bin/composer docker/php/Dockerfile:49
      - 这是多阶段构建的用法。
      - 不单独安装 Composer，而是直接从 composer:2 镜像里把二进制拷进来。
      - 这也是你前面报错里为什么会去拉 composer:2 的原因。
  - COPY docker/php/php.ini ... docker/php/Dockerfile:50
      - 把项目自定义 PHP 配置塞进容器。
      - 对应的配置在 docker/php/php.ini:1，比如时区、内存、上传限制、opcache。

  docker/nginx/default.conf:1 是干什么的
  这是 Nginx 的站点配置，决定 HTTP 请求怎么处理。

  - root /var/www/html/public docker/nginx/default.conf:4
      - Web 根目录是项目的 public 目录。
      - 这是典型 PHP 框架入口目录设计，避免把源码直接暴露给浏览器。
  - location / { try_files ... /index.php?$query_string; } docker/nginx/default.conf:9
      - 先找真实文件。
      - 找不到就交给 index.php，也就是框架前控制器模式。
  - location ~ \.php$ docker/nginx/default.conf:13
      - PHP 文件不由 Nginx 自己执行，而是转发给 php:9000。
      - php 就是 Compose 里的服务名，所以这里实际上依赖 Compose 自动建好的容器网络 DNS。
      - 9000 是 PHP-FPM 默认监听端口。
  - 静态资源配置 docker/nginx/default.conf:21
      - css/js/png/... 走缓存，expires 7d。
      - access_log off 减少日志。
      - 这能降低静态资源请求开销。
  - client_max_body_size 100m docker/nginx/default.conf:7
      - 允许 100M 上传。
      - 它和 docker/php/php.ini:5-6 的 post_max_size / upload_max_filesize 是配套的。

  .env.docker 是什么情况
  这个仓库里实际没有 .env.docker，只有模板文件 .env.docker.example:1。官方说明在 DOCKER.md:14-24，意思是：

  - 先 cp .env.docker.example .env
  - 再 cp .compose.env.example .compose.env

  也就是说：

  - 应用运行时配置用 .env
  - Compose 自己的变量替换用 .compose.env

  这两者不能混用，DOCKER.md:78 已经明确写了。

  .env.docker.example:1 字段清单
  这是给 ThinkPHP 应用读的，不是给 Compose 读的。它是分组式 env。

  - APP_ENV
      - 应用环境名。
      - 代码里会影响业务判断，也会参与队列 tube 名拼接，比如 app/service/MessageQueueUtil.php:39。
  - APP_DEBUG
      - 是否调试模式。
      - 数据库配置里也会读它来决定 SQL 监听等行为，见 config/database.php:56。
  - [APP] DEFAULT_TIMEZONE
      - 应用默认时区。
  - [DATABASE]
      - 主 MySQL 连接。
      - DRIVER、TYPE、HOSTNAME、DATABASE、USERNAME、PASSWORD、HOSTPORT、CHARSET、PREFIX
      - 对应主连接 mysql，见 config/database.php:23。
  - [DATABASE2]
      - 第二套 MySQL 连接。
      - 代码里映射到 mysql2，见 config/database.php:74。
  - [DATABASE_ZHENGSHI]
      - 历史兼容的“正式库”连接名。
      - 如果不配，会回退到主库，见 config/database.php:111。
      - 从命名看，这是为了兼容历史命令/正式环境逻辑。
  - [REDIS]
      - REDIS_HOST、REDIS_PORT、REDIS_PASSWORD
      - 缓存和队列配置都读这里，见 config/cache.php:14 和 config/queue.php:4。
  - [BEANSTALKD]
      - HOST、PORT、TIMEOUT
      - 消息队列客户端读取这里，见 app/service/MessageQueueUtil.php:39。
  - [NCC_DATABASE]
      - HOSTNAME、DATABASE、USERNAME、PASSWORD、HOSTPORT
      - 这是 SQL Server 连接，用于 NCC 集成，见 config/database.php:60。
      - 模板里默认 HOSTNAME=host.docker.internal，就是让容器去访问宿主机上的 SQL Server 或宿主机可达的外部服务。

  .compose.env.example:1 字段清单
  这个才是给 docker compose --env-file .compose.env ... 用的。格式必须是单纯的 KEY=VALUE。

  字段都很直接：

  - MYSQL_ROOT_PASSWORD
      - 初始化 MySQL root 密码。
  - MYSQL_DATABASE
      - 初始化创建的数据库名。
  - MYSQL_USER
      - 初始化创建的普通用户。
  - MYSQL_PASSWORD
      - 上面普通用户的密码。
  - MYSQL_PORT
      - 宿主机映射端口，最终是 宿主机:MYSQL_PORT -> 容器:3306。
  - REDIS_PASSWORD
      - Redis 密码。
  - REDIS_PORT
      - Redis 宿主机映射端口。
  - BEANSTALKD_PORT
      - Beanstalkd 宿主机映射端口。
  - NGINX_PORT
      - Web 服务宿主机端口，默认就是你访问的 8080。

  这些变量是被 compose.yaml:6、16、28、45、72 这种 ${VAR:-default} 语法引用的。

  这几个文件之间怎么配合
  关系可以这样记：

  - compose.yaml
      - 定义“有哪些容器、怎么连、开放哪些端口”。
  - docker/php/Dockerfile
      - 定义“PHP 容器长什么样，需要哪些扩展”。
  - docker/nginx/default.conf
      - 定义“HTTP 请求怎么进来、怎么转给 PHP”。
  - .env.docker.example
      - 定义“PHP 应用内部该连哪个 MySQL/Redis/Beanstalkd/NCC”。
  - .compose.env.example
      - 定义“Compose 启动容器时要用哪些外部变量”。

  结合你前面的报错
  你之前那条错误，不是 Dockerfile 语法错，也不是 Compose 写错。它发生在“构建器去 Docker Hub 拉基础镜像元数据”这一步。

  报错点对应的就是：

  - FROM php:7.3-fpm-buster docker/php/Dockerfile:1
  - COPY --from=composer:2 ... docker/php/Dockerfile:49

  BuildKit 要访问 auth.docker.io，但 DNS 10.255.255.254:53 超时了，所以还没开始执行 Dockerfile 里的安装命令就失败了。

  一句话概括那个问题：

  - 不是项目配置逻辑问题。
  - 是 Docker/WSL 当前网络或 DNS 解析不到 Docker Hub。
