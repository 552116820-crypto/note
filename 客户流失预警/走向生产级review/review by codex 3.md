
### Turn 7  *(2026-04-14 02:54)*

**User:**

部署不用docker吗？我的接口需要上线吗？我的代码是推到git，然后项目负责人上传到服务器吗？我的代码需要和crm代码放在一个目录下吗

**Assistant:**

不用必须 Docker，但需要有**固定、可重复的部署方式**。如果团队服务器已经有 Python/venv/systemd 规范，可以不用 Docker；如果没有，我建议用 Docker 或至少用 systemd + venv 固化下来。

你的情况建议这样理解：

1. **接口要不要上线**
   - 如果前端、CRM 页面、运营工具、项目负责人需要调用 `/api/warning-jobs/*`、`/api/warnings/*`，那 FastAPI 接口就要上线。
   - 如果只是后台定时生成预警，没人手动调用 API，也可以只部署在内网，不对公网开放。
   - Celery 定时任务如果要自动跑，也必须部署并运行。

2. **代码怎么给团队**
   - 你把代码推到 Git。
   - 项目负责人或 CI/CD 从 Git 的某个分支/tag 部署到服务器。
   - 不建议“复制项目目录上传服务器”作为长期方式，容易漏文件、漏依赖、版本不可追踪。
   - 更标准流程是：你开 PR -> review -> merge -> 打 tag 或 release 分支 -> 负责人部署。

3. **要不要和 CRM 代码放一个目录**
   - 不需要，也不建议。
   - 这个项目应该作为独立服务，比如：
     - `/opt/churn-warning-engine`
     - CRM 代码仍在它自己的目录
   - 两边通过 HTTP API 通信：你的服务从 CRM API 拉订单数据，不需要嵌进 CRM 项目目录。
   - 如果 CRM 页面要调用你的预警接口，可以通过内网地址或 Nginx 反代访问。

4. **推荐部署形态**

最小可行生产/灰度部署：

```bash
APP_ENV=production
REPOSITORY_TYPE=api
CHURN_REPOSITORY_TYPE=mysql
DATABASE_URL=mysql+pymysql://user:pass@host/db
API_KEY=your-api-key
CRM_API_SALES_IDS=003748,002527
```

服务器上跑两个进程：

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
celery -A app.celery_app worker --loglevel=info --beat
```

更正式一点，用 systemd 管理这两个进程，日志走 journald 或 `/var/log/churn-warning-engine/`，不要长期写在项目源码目录里。

5. **Docker 的判断**

不用 Docker 也可以上线，但如果团队没有统一 Python 环境管理，我建议 Docker。Docker 的价值是：依赖版本、启动命令、环境隔离都固定，换服务器也更容易。

如果短期灰度，`venv + systemd + Git pull` 就够了。  
如果准备长期维护、多环境部署，建议补 Dockerfile、docker-compose 或 CI/CD。

---

### Turn 8  *(2026-04-14 03:04)*

**User:**

运维要做什么吗

**Assistant:**

要。即使代码是你写的，运维/部署负责人至少要负责“把它稳定跑起来、能重启、能查日志、能告警、能回滚”。

他们需要做这些事：

1. **准备服务器环境**
   - Python 3.10-3.12
   - Redis，如果要跑 Celery 定时任务
   - MySQL 数据库和账号
   - 能访问 CRM API 的网络权限
   - 开放内部访问端口，比如 `8000`，或通过 Nginx 反代

2. **创建 MySQL 库和账号**
   - 给这个服务单独建库和账号，不要共用 root。
   - 配置 `DATABASE_URL`。
   - 执行迁移脚本：
     ```bash
     mysql -u user -p database_name < migrations/mysql/churn/001_warning_records_active_dedup.sql
     ```

3. **配置环境变量**
   至少要有：

   ```bash
   APP_ENV=production
   REPOSITORY_TYPE=api
   CHURN_REPOSITORY_TYPE=mysql
   DATABASE_URL=mysql+pymysql://user:pass@host/db
   API_KEY=your-api-key
   CRM_API_SALES_IDS=003748,002527
   CELERY_BROKER_URL=redis://localhost:6379/0
   CELERY_RESULT_BACKEND=redis://localhost:6379/1
   CORS_ORIGINS=https://your-crm-domain.example
   ```

4. **部署并管理进程**
   需要两个进程：

   ```bash
   uvicorn app.main:app --host 0.0.0.0 --port 8000
   celery -A app.celery_app worker --loglevel=info --beat
   ```

   生产上建议用 `systemd`、Supervisor、Docker Compose 或 K8s 管理，不要手动开 shell 跑。

5. **配置日志**
   - API 日志、Celery 日志要能查。
   - 配置日志轮转，避免磁盘打满。
   - 不建议把长期日志写进项目源码目录，建议走 journald 或 `/var/log/churn-warning-engine/`。

6. **配置反向代理和访问控制**
   - 如果给 CRM 前端或内部工具调用，通常用 Nginx 转发：
     - `/churn-api/` -> `127.0.0.1:8000`
   - 外部访问要走 HTTPS。
   - 不要把接口裸露到公网，除非有明确鉴权和网络策略。

7. **健康检查和监控**
   - 定期访问 `/healthz`。
   - 监控 API 进程、Celery 进程、Redis、MySQL。
   - 监控失败日志，比如 CRM API 超时、Celery batch 失败、MySQL 连接失败。
   - 后续最好补一个更完整的 `/readyz`，检查 MySQL 和 CRM API。

8. **备份和回滚**
   - MySQL 里存的是预警状态和人工处理结果，要有备份。
   - 部署前记录 Git commit/tag。
   - 回滚时知道回到哪个版本、环境变量有没有变化、迁移是否可逆。

你这边需要给运维的交付物是：Git 分支/tag、启动命令、环境变量表、MySQL 迁移脚本、端口说明、接口说明、灰度销售 ID 列表。

---
