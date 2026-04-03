# 代码交付 by claude

## Q: 怎么部署这个项目代码，我希望让其他人能用上我的接口

### 部署方案

你的项目依赖三个组件：**FastAPI 服务本身**、**MySQL**、**Redis**。最实用的方式是用 **Docker Compose** 一键启动。

#### 方案一：Docker Compose 部署（推荐）

项目已有 Dockerfile，但缺少 `docker-compose.yml` 和环境变量配置文件。我来帮你创建：

**.env.example**

```env
# ============================================================
# 数据库配置
# ============================================================
DB_HOST=mysql
DB_PORT=3306
DB_USER=root
DB_PASSWORD=your_mysql_password_here
DB_NAME=crm
CRM_DB_NAME=crm

# ============================================================
# Redis 配置
# ============================================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=

# ============================================================
# 服务配置
# ============================================================
SERVICE_HOST=0.0.0.0
SERVICE_PORT=8100
DEBUG=false
LOG_LEVEL=INFO
```

**docker-compose.yml**

```yaml
version: "3.8"

services:
  # ============================================================
  # AI 预测服务
  # ============================================================
  app:
    build: .
    container_name: churn-warning-ai
    restart: unless-stopped
    ports:
      - "8100:8100"
    env_file:
      - .env
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - churn-net

  # ============================================================
  # MySQL 数据库
  # ============================================================
  mysql:
    image: mysql:8.0
    container_name: churn-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_CHARSET: utf8mb4
    ports:
      - "3307:3306"   # 映射到 3307 避免与宿主机 MySQL 冲突
    volumes:
      - mysql_data:/var/lib/mysql
      - ./sql/schema.sql:/docker-entrypoint-initdb.d/01-schema.sql  # 自动建表
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - churn-net

  # ============================================================
  # Redis
  # ============================================================
  redis:
    image: redis:7-alpine
    container_name: churn-redis
    restart: unless-stopped
    ports:
      - "6380:6379"   # 映射到 6380 避免与宿主机 Redis 冲突
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - churn-net

volumes:
  mysql_data:

networks:
  churn-net:
    driver: bridge
```

**.dockerignore**

```
__pycache__
*.pyc
.git
.env
.env.*
docs/
tests/
*.md
.vscode/
.idea/
```

### 部署步骤

#### 1. 本地 Docker Compose 启动

```bash
# 1) 复制环境变量文件并修改密码
cp .env.example .env
# 编辑 .env，至少修改 DB_PASSWORD

# 2) 构建并启动所有服务
docker compose up -d --build

# 3) 查看日志确认启动成功
docker compose logs -f app

# 4) 测试健康检查
curl http://localhost:8100/health
# 返回: {"status":"ok","service":"churn-warning-ai"}

# 5) 查看 API 文档（FastAPI 自带 Swagger）
# 浏览器打开: http://localhost:8100/docs
```

#### 2. 让其他人访问你的接口

##### 场景 A：公司内网 / 同一局域网

直接用你机器的内网 IP：
```bash
# 查看内网 IP
hostname -I
# 其他人访问: http://你的内网IP:8100/docs
```

##### 场景 B：部署到云服务器（推荐长期使用）

1. **准备一台云服务器**（阿里云 ECS / 腾讯云 CVM / AWS EC2 等，2核4G 起步即可）
2. **安装 Docker + Docker Compose**：
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh
sudo apt install docker-compose-plugin
```
3. **上传代码到服务器**：
```bash
scp -r /mnt/e/churn-warning-ai user@服务器IP:/opt/churn-warning-ai
```
4. **在服务器上启动**：
```bash
cd /opt/churn-warning-ai
cp .env.example .env
vim .env  # 修改密码等配置
docker compose up -d --build
```
5. **开放安全组端口 8100**（云平台控制台操作）
6. 其他人访问：`http://服务器公网IP:8100/docs`

##### 场景 C：加 Nginx 反向代理 + 域名（生产级）

如果需要域名 + HTTPS，在服务器上加一层 Nginx：

```nginx
# /etc/nginx/conf.d/churn-api.conf
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

然后用 `certbot` 一键加 HTTPS：
```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d api.yourdomain.com
```

### 常用运维命令

```bash
# 启动
docker compose up -d

# 停止
docker compose down

# 重新构建（代码更新后）
docker compose up -d --build

# 查看日志
docker compose logs -f app

# 进入容器调试
docker compose exec app bash

# 只重启 AI 服务（不动数据库）
docker compose restart app
```

### 关键注意事项

| 项目 | 说明 |
|------|------|
| **数据库初始化** | `sql/schema.sql` 会在 MySQL 首次启动时自动执行建表 |
| **CRM 数据** | 你的服务需要从 CRM 数据库读取订单数据。如果 CRM 是独立的数据库，需要在 `.env` 中把 `DB_HOST` 指向真实的 CRM 数据库地址，而不是 compose 内的 mysql 容器 |
| **数据持久化** | MySQL 数据存储在 Docker volume `mysql_data` 中，`docker compose down` 不会丢数据，但 `docker compose down -v` 会 |
| **安全** | 生产环境务必修改 `DB_PASSWORD`，并且不要把 `.env` 文件提交到 Git |

---

## Q: 我作为AI工程师，是不是把这个项目代码上传到gitlab上就可以了，部署到服务器的事情交给其他人？

对，通常就是这样分工。你作为 AI 工程师，核心职责是：

1. **代码上传到 GitLab** — 保证代码可运行
2. **提供部署所需的配置文件** — 让运维/后端同事能顺利部署
3. **写清楚接口文档和依赖说明** — 别人知道怎么用

### 你需要确保交付的东西

项目里目前已经有的：

- `Dockerfile` — 有了
- `requirements.txt` — 有了
- `sql/schema.sql` — 有了
- `docker-compose.yml` — 刚创建的
- `.env.example` — 刚创建的

还建议补一个 `.gitignore`，避免把不该提交的文件传上去：

```gitignore
# 环境变量（含密码）
.env

# Python
__pycache__/
*.pyc
*.pyo
*.egg-info/
dist/
build/
.venv/
venv/

# IDE
.vscode/
.idea/
*.swp

# 系统文件
.DS_Store
Thumbs.db
```

### 上传到 GitLab

```bash
cd /mnt/e/churn-warning-ai

git init
git add .
git commit -m "init: 客户流失预警 AI 预测服务"

# 替换成你们公司的 GitLab 地址和项目路径
git remote add origin https://gitlab.yourcompany.com/ai/churn-warning-ai.git
git push -u origin main
```

### 交接给部署同事时，告诉他们几件事

1. **运行方式**：`docker compose up -d --build`，服务跑在 **8100 端口**
2. **环境变量**：复制 `.env.example` 为 `.env`，填入真实的数据库密码
3. **数据库建表**：执行 `sql/schema.sql`（Docker Compose 首次启动会自动执行）
4. **关键依赖**：需要连接 CRM 数据库读取订单数据，`.env` 里的 `DB_HOST` 要指向正确的地址
5. **接口文档**：启动后访问 `http://服务器IP:8100/docs` 即可看到 Swagger 文档

这些信息口头说或者在 GitLab 项目的 README 里写清楚都行。你们如果有运维或后端同事负责部署，他们拿到这些就够了。

---

## Q: 部署之后呢，后端或者前端同事怎么使用我的接口

你的同事使用方式很简单——**就是调 HTTP 接口**，跟调任何其他服务的 API 一样。

### 你的服务一共提供 6 个接口

| 接口 | 方法 | 用途 | 谁调 |
|------|------|------|------|
| `/health` | GET | 健康检查 | 运维监控 |
| `/api/ai/predict/batch` | POST | 批量运行 AI 预测 | 定时任务 / 后端 |
| `/api/ai/predict/{customer_id}` | GET | 查询单客户预测结果 | 前端页面 |
| `/api/ai/detect/frequency` | POST | 触发频率异常检测 | 定时任务 / 后端 |
| `/api/ai/detect/volume` | POST | 触发量级异常检测 | 定时任务 / 后端 |
| `/api/ai/detect/product` | POST | 触发产品异常检测 | 定时任务 / 后端 |

### 各接口调用示例

#### 1. 健康检查

```bash
GET http://服务器IP:8100/health
```
```json
{"status": "ok", "service": "churn-warning-ai"}
```

#### 2. 批量预测（定时任务调用）

```bash
# 处理所有活跃客户
POST http://服务器IP:8100/api/ai/predict/batch
Content-Type: application/json

{}

# 只处理指定客户
POST http://服务器IP:8100/api/ai/predict/batch
Content-Type: application/json

{"customer_ids": [1001, 1002, 1003]}
```

响应：
```json
{
  "job_id": 42,
  "status": "success",
  "total_customers": 150,
  "success_count": 145,
  "fail_count": 0,
  "skip_count": 5,
  "duration_seconds": 12
}
```

#### 3. 查询单客户预测结果（前端展示用）

```bash
GET http://服务器IP:8100/api/ai/predict/1001
```

响应：
```json
{
  "customer_id": 1001,
  "customer_name": "某某公司",
  "overall_prediction": {
    "customer_id": 1001,
    "customer_name": "某某公司",
    "prediction_type": 1,
    "predicted_cycle_days": 30,
    "last_order_date": "2026-03-15",
    "confidence": 0.85,
    "sample_size": 12
  },
  "key_products": [
    {
      "product_id": 2001,
      "product_name": "产品A",
      "total_amount": 50000,
      "amount_pct": 45.5,
      "predicted_cycle_days": 25,
      "last_order_date": "2026-03-20",
      "confidence": 0.78,
      "sample_size": 8
    }
  ],
  "latest_volume_snapshot": {
    "customer_id": 1001,
    "customer_name": "某某公司",
    "current_month_qty": 120,
    "last_month_qty": 200,
    "change_rate": -40.0,
    "is_anomaly": false,
    "anomaly_direction": null
  }
}
```

#### 4. 触发频率异常检测

```bash
POST http://服务器IP:8100/api/ai/detect/frequency
Content-Type: application/json

{}                                    # 检测所有客户
{"customer_ids": [1001, 1002]}        # 只检测指定客户
```

#### 5. 触发量级异常检测

```bash
POST http://服务器IP:8100/api/ai/detect/volume
Content-Type: application/json

{}                                    # 检测当月
{"month": "2026-03"}                  # 检测指定月份
```

#### 6. 触发产品异常检测

```bash
POST http://服务器IP:8100/api/ai/detect/product
Content-Type: application/json

{}
```

### 前端同事怎么对接

前端一般只需要关心 **1 个接口**：

```
GET /api/ai/predict/{customer_id}
```

在客户详情页调这个接口，拿到数据后展示：
- **预测下单周期**：`overall_prediction.predicted_cycle_days` 天
- **置信度**：`overall_prediction.confidence`
- **关键产品列表**：`key_products` 数组
- **量级变化**：`latest_volume_snapshot`

### 后端同事怎么对接

后端需要做两件事：

1. **配定时任务**，按时调用 AI 服务生成预测数据：

| 任务 | 建议频率 | 调用接口 |
|------|---------|---------|
| 批量预测 | 每周日凌晨 2:00 | `POST /api/ai/predict/batch` |
| 频率异常检测 | 每周一早上 8:00 | `POST /api/ai/detect/frequency` |
| 量级异常检测 | 每月 1 日早上 8:00 | `POST /api/ai/detect/volume` |
| 产品异常检测 | 每周一早上 8:00 | `POST /api/ai/detect/product` |

2. **接收预警结果**，detect 接口返回的预警数据里包含 `warning_type`、`message`、`detail`，后端拿到后写入预警记录表、推送给对应销售。

### 自带 API 文档

部署后浏览器打开 **`http://服务器IP:8100/docs`**，FastAPI 自动生成 Swagger 文档，前后端同事可以在页面上直接试调接口，不需要你额外写文档。

---

## Q: 把服务器ip改成公网能访问的域名要怎么做

两步：**域名解析指向服务器 IP** + **Nginx 反向代理**。

### 第一步：域名解析

在你们的域名管理平台（阿里云 DNS / 腾讯云 DNSPod / Cloudflare 等）添加一条 A 记录：

| 记录类型 | 主机记录 | 记录值 | 说明 |
|---------|---------|--------|------|
| A | churn-ai | 你的服务器公网 IP | 解析 `churn-ai.yourcompany.com` 到服务器 |

如果公司还没有域名，需要先买一个（阿里云/腾讯云几十块一年）。

### 第二步：服务器上装 Nginx + HTTPS

在 docker-compose 里加一个 Nginx 容器，一并管理。

**nginx/nginx.conf**

```nginx
server {
    listen 80;
    server_name churn-ai.yourcompany.com;   # 改成你的真实域名

    location / {
        proxy_pass http://app:8100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**docker-compose.yml 中加 nginx 服务**

```yaml
  nginx:
    image: nginx:alpine
    container_name: churn-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/certbot/conf:/etc/letsencrypt
      - ./nginx/certbot/www:/var/www/certbot
    depends_on:
      - app
    networks:
      - churn-net
```

同时把 app 的 `ports: "8100:8100"` 改成 `expose: "8100"`，只对内部网络暴露，不直接对外。

### 完整操作流程（给运维同事的）

#### 1. 域名解析

在域名管理平台加一条记录：

```
类型: A
主机记录: churn-ai          （或你想要的子域名）
记录值: 服务器公网 IP
```

生效后 `churn-ai.yourcompany.com` 就指向你的服务器了。

#### 2. 修改 Nginx 配置中的域名

打开 `nginx/nginx.conf`，把 `server_name` 改成真实域名：

```nginx
server_name churn-ai.yourcompany.com;   # 改成真实域名
```

#### 3. 启动服务

```bash
docker compose up -d --build
```

此时 `http://churn-ai.yourcompany.com/docs` 已经可以访问了。

#### 4. 加 HTTPS（可选但推荐）

```bash
# 用 certbot 申请免费证书
docker run --rm \
  -v ./nginx/certbot/conf:/etc/letsencrypt \
  -v ./nginx/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot -w /var/www/certbot \
  -d churn-ai.yourcompany.com \
  --email your-email@company.com \
  --agree-tos --no-eff-email
```

证书申请成功后，把 `nginx/nginx.conf` 升级为 HTTPS：

```nginx
# HTTP → 自动跳转 HTTPS
server {
    listen 80;
    server_name churn-ai.yourcompany.com;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl;
    server_name churn-ai.yourcompany.com;

    ssl_certificate     /etc/letsencrypt/live/churn-ai.yourcompany.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/churn-ai.yourcompany.com/privkey.pem;

    location / {
        proxy_pass http://app:8100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

然后重启 Nginx：
```bash
docker compose restart nginx
```

### 最终效果

| 之前 | 之后 |
|------|------|
| `http://123.45.67.89:8100/docs` | `https://churn-ai.yourcompany.com/docs` |
| `http://123.45.67.89:8100/api/ai/predict/1001` | `https://churn-ai.yourcompany.com/api/ai/predict/1001` |

---

## Q: 有必要公网访问吗，让后端在内网调用我的接口不就行了吗

对，你说得对。你这个服务是给后端系统内部调用的，没必要暴露到公网。

不需要 Nginx、域名、HTTPS，docker-compose.yml 保持简单，app 直接暴露 8100 端口即可。

后端同事直接用内网地址调就行：

```
http://内网IP:8100/api/ai/predict/batch
http://内网IP:8100/api/ai/predict/{customer_id}
...
```

如果后端服务也跑在同一台服务器上，直接用 `http://localhost:8100` 或 `http://127.0.0.1:8100` 就行，连内网 IP 都不需要。

---

## Q: 也就是说我把我这个项目代码上传到gitlab，我就算完成工作了，部署交给运维，提供接口文档给后端

对，就是这样。你的交付物就三样：

1. **GitLab 上的代码**（含 Dockerfile、docker-compose.yml、.env.example、sql/schema.sql）
2. **接口文档** — 服务启动后 `http://IP:8100/docs` 自动生成，不用额外写
3. **口头/文档告知**运维和后端几个关键信息：
   - 启动命令：`docker compose up -d --build`
   - 端口：8100
   - 需要配 `.env` 里的数据库连接信息
   - 需要配定时任务调批量预测和预警检测接口

这就是你作为 AI 工程师的交付边界了。

---

## Q: 一定得用docker部署吗

不一定。Docker 只是方便运维，但你的项目就是个普通 Python 服务，直接跑也完全可以。

### 不用 Docker 的部署方式

运维在服务器上执行：

```bash
# 1. 装 Python
apt install python3.11 python3.11-venv

# 2. 建虚拟环境
cd /opt/churn-warning-ai
python3.11 -m venv venv
source venv/bin/activate

# 3. 装依赖
pip install -r requirements.txt

# 4. 设环境变量
export DB_HOST=数据库地址
export DB_PASSWORD=密码
# ... 其他变量参考 .env.example

# 5. 启动
uvicorn src.api:app --host 0.0.0.0 --port 8100
```

生产环境一般再加个 systemd 守护进程，让服务开机自启、挂了自动重启，但这是运维的事。

### 总结

| 方式 | 优点 | 缺点 |
|------|------|------|
| Docker | 环境一致，一键启动 | 服务器要装 Docker |
| 直接跑 | 简单直接，没有额外依赖 | 运维自己管 Python 环境 |

**跟你没关系**，你交付代码就行，运维选哪种方式部署是他们的事。你提供的 Dockerfile 和 requirements.txt 两种方式都能支持。
