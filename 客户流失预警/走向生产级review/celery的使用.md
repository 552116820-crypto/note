  改完后重建并重启：

  docker compose up -d --build celery

  或整个项目一起重启：

  docker compose up -d --build
  如果你想现在立刻补跑一次 2026-04-27 的检测，可以手动派发：

  docker compose exec celery celery -A app.celery_app call app.churn.tasks.run_detection --args='["frequency","2026-04-27"]'

  产品异常也补跑：

  docker compose exec celery celery -A app.celery_app call app.churn.tasks.run_detection --args='["product","2026-04-27"]'

# Codex CLI 对话记录

- **来源**: Codex CLI
- **会话**: `019dcc48-c3b1-7ed1-9798-bb19f32400ac`
- **范围**: Turn 1-14
- **导出时间**: 2026-04-27 15:01:48

---

### Turn 1  *(2026-04-27 00:14)*

**User:**

怎么在服务器上看celery的运行日志

**Assistant:**

我先看一下仓库里有没有 Celery 启动或部署配置，这样回答能贴近你们服务器上的实际运行方式。

仓库里有 `docker-compose.yml`，Celery 服务名就是 `celery`，启动命令里也直接把日志打到标准输出。服务器如果是按这个 compose 部署，看 Docker 日志就是主路径。

你这个仓库默认是 Docker Compose 部署，Celery 服务名是 `celery`，所以服务器上最常用这样看：

```bash
cd /path/to/churn-warning-engine
docker compose ps
docker compose logs -f --tail=200 celery
```

看最近 2 小时并带时间戳：

```bash
docker compose logs -f -t --since=2h celery
```

查错误：

```bash
docker compose logs --tail=1000 celery | grep -Ei "error|exception|traceback|failed|retry"
```

如果服务器用的是老版本 Compose，命令换成：

```bash
docker-compose logs -f --tail=200 celery
```

如果不确定容器名：

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Command}}' | grep -i celery
docker logs -f --tail=200 <容器名>
```

这个项目的 `docker-compose.yml` 里 Celery 是 `worker --beat` 合在同一个服务里，所以这些日志会同时包含定时调度 beat 和任务执行 worker 的日志。

另外也可以通过接口看最近检测运行记录：

```bash
curl -H "X-API-Key: $API_KEY" \
  "http://127.0.0.1:8080/api/detection-runs?limit=20"
```

如果服务器没配置 `API_KEY`，就去掉 `-H "X-API-Key: ..."`。

---

### Turn 2  *(2026-04-27 00:23)*

**User:**

怎么看有没有报错，或者正常运行

**Assistant:**

我再看一下项目里 Celery 的定时任务时间，方便你区分“没到执行时间所以没日志”和“真的没跑”。

可以按这几个层次看。

**1. 看容器是否活着**

```bash
cd /path/to/churn-warning-engine
docker compose ps celery
```

正常大概是 `Up` / `running`，如果看到 `Restarting`、`Exited` 就不正常。

也可以看重启次数：

```bash
docker inspect --format='Status={{.State.Status}} RestartCount={{.RestartCount}} Error={{.State.Error}}' \
  $(docker compose ps -q celery)
```

**2. 看 Celery 日志有没有错误**

```bash
docker compose logs --tail=500 celery
```

只筛错误关键词：

```bash
docker compose logs --tail=1000 celery | grep -Ei "error|exception|traceback|critical|failed|retry|refused|timeout"
```

实时盯日志：

```bash
docker compose logs -f -t --tail=200 celery
```

正常启动日志里通常会看到类似：

```text
celery@... ready
beat: Starting...
```

任务正常执行时通常会看到：

```text
Task app.churn.tasks.run_detection[...] received
Task app.churn.tasks.run_detection[...] succeeded
```

如果有 `Traceback`、`CRITICAL`、`OperationalError`、`Connection refused`、`Authentication failed`，基本就是需要排查的问题。

**3. 直接问 Celery worker 是否响应**

```bash
docker compose exec celery celery -A app.celery_app inspect ping
```

正常会返回类似：

```text
celery@xxxx: OK
    pong
```

再看当前是否有任务在跑：

```bash
docker compose exec celery celery -A app.celery_app inspect active
docker compose exec celery celery -A app.celery_app inspect scheduled
docker compose exec celery celery -A app.celery_app inspect reserved
```

**4. 看业务运行记录**

这个项目有检测运行历史接口：

```bash
curl -H "X-API-Key: $API_KEY" \
  "http://127.0.0.1:8080/api/detection-runs?limit=20"
```

正常关注 `status`：

```text
completed  正常完成
partial    部分批次失败
failed     任务失败
```

如果没配置 `API_KEY`，去掉请求头即可。

**注意时间**

当前定时任务不是一直执行：

- 每周一 08:00 跑 `frequency`
- 每周一 08:10 跑 `product`
- 每月 1 日 08:00 跑 `volume`
- 时区是 `Asia/Shanghai`

所以日志里暂时没有新任务，不一定异常；先看 `docker compose ps celery`、`inspect ping` 和最近日志有没有报错。

---

### Turn 3  *(2026-04-27 00:27)*

**User:**

为什么数据库中存在biz_date=2026-04-26且status=pending_enrich的行，2026-04-27celery跑了一次啊

**Assistant:**

我看一下 Celery 任务里 `biz_date` 是怎么计算、`pending_enrich` 是在哪一步写入的；这个问题通常是“任务执行日期”和“业务日期”不是同一天。

已经看到关键点：2026-04-27 跑的是“执行日期”，但 `frequency/product` 没传业务日期时会在 service 里按当天推导；如果配置了 `DEMO_REFERENCE_DATE` 或当前业务日期被显式传入，就可能落到 2026-04-26。我继续确认 `pending_enrich` 的状态来源和是否会被 AI 自动改掉。

核心原因：`biz_date` 不是 Celery 执行日期，而是“业务检测日期”。`status=pending_enrich` 也不是任务没跑完，而是“预警已生成，等待人工丰富”。

按这份代码：

- 2026-04-27 是周一，Celery 定时会跑：
  - 08:00 `frequency`
  - 08:10 `product`
- `frequency/product` 没显式传 `biz_date` 时，最终用：
  - `DEMO_REFERENCE_DATE`，如果 `.env` 配了
  - 否则用 worker 进程里的 `date.today()`
- 新生成的预警默认就是 `pending_enrich`
- Celery 跑完只会把 `detection_runs.status` 置为 `completed/partial/failed`，不会自动把 `warning_records.status` 从 `pending_enrich` 改成 `enriched`

先查这几个点：

```sql
SELECT run_id, job_id, job_type, biz_date, started_at, finished_at,
       status, total_customers, total_anomalies, failed_batches
FROM detection_runs
ORDER BY started_at DESC
LIMIT 20;
```

如果 2026-04-27 那次 run 的 `biz_date` 也是 `2026-04-26`，优先看服务器 `.env`：

```bash
docker compose exec celery printenv | grep DEMO_REFERENCE_DATE
```

如果输出：

```text
DEMO_REFERENCE_DATE=2026-04-26
```

那就是它导致的。

再看这些 `pending_enrich` 属于什么类型：

```sql
SELECT anomaly_type, biz_date, status, COUNT(*)
FROM warning_records
WHERE biz_date = '2026-04-26'
  AND status = 'pending_enrich'
GROUP BY anomaly_type, biz_date, status;
```

如果 4/27 的同类型任务确实跑了，而且 `biz_date` 应该是 2026-04-27，但这些 4/26 还留着，常见原因是：

1. 4/27 那次 run 是 `partial` 或 `failed`，部分 batch 没处理到。
2. 这些客户当天没有出现在 CRM 返回的销售/客户列表里，所以自动 resolve 逻辑没覆盖到。
3. 它们属于另一个 `anomaly_type`，当天没跑对应类型。
4. worker 实际使用了 `DEMO_REFERENCE_DATE=2026-04-26`，所以 4/27 执行但业务日期仍是 4/26。

判断是否“正常运行”，以 `detection_runs` 为准：`status=completed` 且 `failed_batches=[]` 基本说明检测跑完；`warning_records.status=pending_enrich` 本身是正常业务状态，不代表 Celery 报错。

---

### Turn 4  *(2026-04-27 03:49)*

**User:**

服务器用的时间是哪个时区的

**Assistant:**

我看下 Dockerfile 和 compose 里有没有显式设置 `TZ`，因为 Celery 调度时区和容器系统时区可能不是同一个概念。

这个项目里有两套时间概念：

1. **Celery 定时调度时区**：代码固定是 `Asia/Shanghai`

```python
# app/celery_app.py
"timezone": "Asia/Shanghai"
```

所以周一 08:00、08:10 是按北京时间触发。

2. **任务里算 `biz_date` 用的系统时区**：`date.today()` / `datetime.now()` 用的是 Celery 容器自己的本地时区。

你现在看到 `2026-04-27` 跑了一次，但写出 `biz_date=2026-04-26`，很像是：Celery 按北京时间 2026-04-27 08:00 触发，但容器本地时区还是美国时间或其他慢于北京时间的时区，所以 `date.today()` 得到的是 `2026-04-26`。

在服务器上直接查：

```bash
date
timedatectl
docker compose exec celery date
docker compose exec celery python - <<'PY'
from datetime import date, datetime
import os, time
print("TZ env:", os.getenv("TZ"))
print("time.tzname:", time.tzname)
print("datetime.now():", datetime.now())
print("date.today():", date.today())
PY
```

再查 Celery 配置：

```bash
docker compose exec celery python - <<'PY'
from app.celery_app import celery_app
print(celery_app.conf.timezone)
PY
```

如果输出里：

```text
celery_app.conf.timezone = Asia/Shanghai
date.today() = 2026-04-26
```

那就能解释这个现象。

当前仓库的 `Dockerfile` 和 `docker-compose.yml` 没有设置 `TZ`，所以容器时区取决于镜像/环境。建议在 `.env` 加：

```env
TZ=Asia/Shanghai
```

然后重建或重启容器：

```bash
docker compose up -d --build
```

不过更稳的代码修复是把 `date.today()` 改成显式北京时间日期，例如 `datetime.now(ZoneInfo("Asia/Shanghai")).date()`，避免容器时区影响业务日期。

---

### Turn 5  *(2026-04-27 03:56)*

**User:**

docker compose exec celery python - <<'PY'
  from app.celery_app import celery_app
  print(celery_app.conf.timezone)
  PY这要怎么复制到服务器中

**Assistant:**

最简单：不要复制多行，直接复制这一行到服务器执行：

```bash
docker compose exec celery python -c "from app.celery_app import celery_app; print(celery_app.conf.timezone)"
```

完整流程一般是：

```bash
ssh root@你的服务器IP
cd /path/to/churn-warning-engine
docker compose exec celery python -c "from app.celery_app import celery_app; print(celery_app.conf.timezone)"
```

如果你想复制多行版本，要注意最后的 `PY` 前面不能有空格：

```bash
docker compose exec -T celery python - <<'PY'
from app.celery_app import celery_app
print(celery_app.conf.timezone)
PY
```

你前面写的这个：

```bash
  from app.celery_app import celery_app
  PY
```

因为前面有空格，`PY` 可能不会被 shell 识别为结束标记，所以建议直接用第一条单行命令。

---

### Turn 6  *(2026-04-27 03:57)*

**User:**

怎么换行啊

**Assistant:**

在服务器终端里可以直接粘贴多行，按回车换行就行。关键是最后一行 `PY` 必须顶格，前面不要有空格。

你可以这样输入：

```bash
docker compose exec -T celery python - <<'PY'
from app.celery_app import celery_app
print(celery_app.conf.timezone)
PY
```

实际操作：

1. 先粘贴第一行，然后回车：

```bash
docker compose exec -T celery python - <<'PY'
```

2. 终端会等你继续输入，继续粘贴：

```python
from app.celery_app import celery_app
print(celery_app.conf.timezone)
```

3. 再单独输入最后一行并回车：

```bash
PY
```

如果你不想处理换行，直接用单行版：

```bash
docker compose exec celery python -c "from app.celery_app import celery_app; print(celery_app.conf.timezone)"
```

---

### Turn 7  *(2026-04-27 03:59)*

**User:**

from app.celery_app import celery_app
  print(celery_app.conf.timezone)这两行代码也是要一行一行粘贴的吗

**Assistant:**

是的，多行版里这两行就一行一行粘贴：

```python
from app.celery_app import celery_app
print(celery_app.conf.timezone)
```

注意第二行前面不要有空格。你写的：

```python
  print(celery_app.conf.timezone)
```

前面有两个空格，可能会报 `IndentationError`。应该顶格：

```python
print(celery_app.conf.timezone)
```

完整输入顺序是：

```bash
docker compose exec -T celery python - <<'PY'
from app.celery_app import celery_app
print(celery_app.conf.timezone)
PY
```

或者直接复制单行，最省事：

```bash
docker compose exec celery python -c "from app.celery_app import celery_app; print(celery_app.conf.timezone)"
```

---

### Turn 8  *(2026-04-27 04:00)*

**User:**

怎么改时区啊，[root@iZ7xvdwme1yos06mt96oq6Z churn-warning-engine]# docker compose exec -T celery python - <<'PY'
> from app.celery_app import celery_app
> print(celery_app.conf.timezone)
> PY
Asia/Shanghai ，我现在是在上海

**Assistant:**

你查到的是 **Celery 调度时区**，已经是上海：

```text
Asia/Shanghai
```

所以 Celery 的 08:00 / 08:10 定时本身不用改。现在要确认的是 **容器系统时间** 和 Python 的 `date.today()` 是不是上海日期。

先执行：

```bash
date
docker compose exec celery date
docker compose exec celery python -c "from datetime import date, datetime; import os, time; print('TZ=', os.getenv('TZ')); print('tzname=', time.tzname); print('now=', datetime.now()); print('today=', date.today())"
```

如果容器里不是上海时间，就这样改。

编辑 `.env`：

```bash
vi .env
```

加一行：

```env
TZ=Asia/Shanghai
```

然后重建/重启容器：

```bash
docker compose up -d --build
```

再验证：

```bash
docker compose exec celery date
docker compose exec celery python -c "from datetime import date, datetime; print(datetime.now()); print(date.today())"
```

如果服务器宿主机时区也不是上海，可以顺手改宿主机：

```bash
timedatectl set-timezone Asia/Shanghai
timedatectl
date
```

重点：你现在 Celery 配置已经是上海；要解决 `biz_date` 可能偏一天的问题，主要看 `docker compose exec celery date` 和 Python 的 `date.today()`。

---

### Turn 9  *(2026-04-27 06:05)*

**User:**

[root@iZ7xvdwme1yos06mt96oq6Z churn-warning-engine]# date
Mon Apr 27 02:03:31 PM CST 2026
[root@iZ7xvdwme1yos06mt96oq6Z churn-warning-engine]# docker compose exec celery date
Mon Apr 27 06:03:49 UTC 2026
[root@iZ7xvdwme1yos06mt96oq6Z churn-warning-engine]# docker compose exec celery python -c "from datetime import date, datetime; import os, time; print('TZ=', os.getenv('TZ')); print('tzname=', time.tzname); print('now=', datetime.now()); print('today=',date.today())"
TZ= None
tzname= ('UTC', 'UTC')
now= 2026-04-27 06:04:15.825499
today= 2026-04-27，所以现在服务器用的什么时间啊

**Assistant:**

现在是：

- **宿主机服务器时间**：上海时间 / CST / UTC+8  
  你看到的是：

```text
Mon Apr 27 02:03:31 PM CST 2026
```

也就是 **2026-04-27 14:03:31 上海时间**。

- **Celery 容器系统时间**：UTC 时间  
  你看到的是：

```text
Mon Apr 27 06:03:49 UTC 2026
```

这个和上海时间正好差 8 小时，等价于 **2026-04-27 14:03:49 上海时间**。

- **Celery 调度时区**：Asia/Shanghai  
  你前面查到：

```text
Asia/Shanghai
```

所以结论是：**服务器宿主机用上海时间，Celery 容器内部用 UTC，但 Celery 定时调度按上海时间执行。**

不过你这个输出里 `date.today()` 也是：

```text
today=2026-04-27
```

所以如果 Celery 是 2026-04-27 08:00 上海时间跑的，容器 UTC 是 2026-04-27 00:00，`biz_date` 正常也应该是 `2026-04-27`，不是 `2026-04-26`。

因此 `biz_date=2026-04-26` 更可能不是时区导致，优先查：

```bash
docker compose exec celery printenv | grep DEMO_REFERENCE_DATE
```

以及数据库里 4/27 那次运行到底写的业务日期：

```sql
SELECT run_id, job_type, biz_date, started_at, finished_at, status, total_anomalies, failed_batches
FROM detection_runs
ORDER BY started_at DESC
LIMIT 20;
```

---

### Turn 10  *(2026-04-27 06:08)*

**User:**

我希望celery是2026-04-27 07:00 上海时间跑，怎么弄

**Assistant:**

Celery 调度时区已经是 `Asia/Shanghai`，所以只要改 `app/celery_app.py` 里的 crontab 时间即可。

当前是：

```python
"weekly-frequency-check": {
    "schedule": crontab(day_of_week="monday", hour=8, minute=0),
},
"weekly-product-check": {
    "schedule": crontab(day_of_week="monday", hour=8, minute=10),
},
```

如果你希望每周一 **上海时间 07:00** 跑 `frequency`，改成：

```python
"schedule": crontab(day_of_week="monday", hour=7, minute=0),
```

如果 `product` 也要提前，建议设成 07:10：

```python
"schedule": crontab(day_of_week="monday", hour=7, minute=10),
```

改完后重建并重启：

```bash
docker compose up -d --build celery
```

或整个项目一起重启：

```bash
docker compose up -d --build
```

然后确认配置：

```bash
docker compose logs -f --tail=100 celery
```

注意：现在已经是 **2026-04-27 14:00 多上海时间**，所以 **2026-04-27 07:00 已经过去了**。改完后会从下一个符合条件的时间开始自动跑，也就是下周一 07:00。

如果你想现在立刻补跑一次 2026-04-27 的检测，可以手动派发：

```bash
docker compose exec celery celery -A app.celery_app call app.churn.tasks.run_detection --args='["frequency","2026-04-27"]'
```

产品异常也补跑：

```bash
docker compose exec celery celery -A app.celery_app call app.churn.tasks.run_detection --args='["product","2026-04-27"]'
```

---

### Turn 11  *(2026-04-27 06:14)*

**User:**

如果按现在的时间，如果8点frequency到8点10分还没完成，8点10分的product怎么办

**Assistant:**

按现在配置，`frequency` 和 `product` 是两个独立的 Celery 任务。**8:00 的 `frequency` 没跑完，8:10 的 `product` 仍然会被 beat 派发到队列里。**

实际会怎样取决于 worker 并发：

```yaml
--concurrency=${CELERY_CONCURRENCY:-4}
--prefetch-multiplier=${CELERY_PREFETCH_MULTIPLIER:-1}
```

默认 `CELERY_CONCURRENCY=4`，所以：

- 如果 8:10 时还有空闲 worker 进程，`product` 会马上开始跑。
- 如果 4 个 worker 都被 `frequency` 或其他任务占满，`product` 会先排队，等有 worker 空出来再跑。
- Celery 不会自动等 `frequency` 完成后再派发 `product`。
- 两个任务可能并发访问 CRM、MySQL、外部配置同步、预警写入和外部导出接口。

如果你希望避免并发，比较简单的做法是把 `product` 时间拉开，比如 09:00：

```python
"weekly-product-check": {
    "task": "app.churn.tasks.run_detection",
    "schedule": crontab(day_of_week="monday", hour=9, minute=0),
    "args": ("product",),
},
```

或者把 Celery 并发设成 1：

```env
CELERY_CONCURRENCY=1
```

这样 8:10 的 `product` 仍会入队，但会等 `frequency` 跑完才执行。代价是所有 Celery 任务都串行，AI 分析任务也会被拖慢。

---

### Turn 12  *(2026-04-27 06:16)*

**User:**

今天传给crm库的数据有很多都是employee_code为空，你帮我排查一下原因

**Assistant:**

我会用 `fastapi-python` 技能来按代码链路排查：先找 `employee_code` 在导出 payload 里从哪里来，再看今天 Celery 跑出的数据可能在哪一步变成空值。

我先定位 `employee_code` 的生成位置和 CRM 导出格式，再沿着 warning 生成时的销售字段来源往上查。

导出字段已经确认：`employee_code` 直接等于 `warning.sales_id`。所以问题不是导出层改名，而是生成 `warning_records` 时 `sales_id` 已经为空，下一步查 CRM 数据源如何给 Customer 填销售编码。

代码链路显示 `employee_code` 没有单独计算，它就是预警表里的 `sales_id`。我现在重点看两类来源：一是 CRM API 返回的客户/订单归属，二是 MySQL CRM 源表里的 `sales_id`。

我查了代码，`employee_code` 的来源很直接：

- 导出给 CRM 时：`employee_code = warning.sales_id`，在 [app/churn/warning_export.py](/mnt/e/churn-warning-engine/app/churn/warning_export.py:247)
- 生成预警时：`warning.sales_id = customer.sales_id`，在 [app/churn/anomaly_strategies/support.py](/mnt/e/churn-warning-engine/app/churn/anomaly_strategies/support.py:483)
- `customer.sales_id` 来自 CRM 数据源：
  - API 模式：CRM 接口返回的 `sales_id`，在 [app/shared/crm_api.py](/mnt/e/churn-warning-engine/app/shared/crm_api.py:351)
  - MySQL 模式：`crm_customers.sales_id`，在 [app/shared/crm_mysql.py](/mnt/e/churn-warning-engine/app/shared/crm_mysql.py:78)

所以结论是：**不是导出字段映射错了，而是生成预警时 `sales_id` 已经为空或只有空格。导出层当前没有拦截，会照样把空 `employee_code` 推给 CRM。**

你先在服务器跑这两个排查命令。

看当前用的 CRM 数据源：

```bash
docker compose exec celery sh -lc 'echo REPOSITORY_TYPE=$REPOSITORY_TYPE; echo CHURN_REPOSITORY_TYPE=$CHURN_REPOSITORY_TYPE; echo CRM_CUSTOMER_API_URL_SET=${CRM_CUSTOMER_API_URL:+yes}; echo CRM_API_SALES_IDS_SET=${CRM_API_SALES_IDS:+yes}'
```

查今天 `warning_records` 里是不是已经有空 `sales_id`：

```bash
docker compose exec -T celery python - <<'PY'
from datetime import datetime
from zoneinfo import ZoneInfo
from sqlalchemy import create_engine, text
import os

biz = datetime.now(ZoneInfo("Asia/Shanghai")).date().isoformat()
engine = create_engine(os.environ["DATABASE_URL"])

with engine.connect() as c:
    print("biz_date =", biz)

    rows = c.execute(text("""
        SELECT anomaly_type,
               COUNT(*) AS total,
               SUM(CASE WHEN sales_id IS NULL OR TRIM(sales_id) = '' THEN 1 ELSE 0 END) AS empty_sales_id
        FROM warning_records
        WHERE biz_date = :biz
        GROUP BY anomaly_type
    """), {"biz": biz}).mappings().all()
    print("summary:")
    for r in rows:
        print(dict(r))

    rows = c.execute(text("""
        SELECT warning_id, anomaly_type, customer_id, customer_name, sales_id, sales_name
        FROM warning_records
        WHERE biz_date = :biz
          AND (sales_id IS NULL OR TRIM(sales_id) = '')
        LIMIT 20
    """), {"biz": biz}).mappings().all()
    print("samples:")
    for r in rows:
        print(dict(r))
PY
```

如果这里 `empty_sales_id` 大于 0，就说明问题已经在本系统的 `warning_records.sales_id` 里了。

下一步判断来源：

- 如果 `REPOSITORY_TYPE=mysql`：查 `crm_customers.sales_id` 是否为空。
- 如果 `REPOSITORY_TYPE=api`：查 CRM 订单/客户接口返回的 `sales_id` 是否为空，尤其是没有配置 `CRM_CUSTOMER_API_URL` 时，系统会更多依赖订单接口里的 `sales_id`。

代码层建议后续加一个保护：导出 CRM 前发现 `employee_code` 为空就跳过并打日志，或者直接让任务失败暴露数据问题，避免继续把脏数据写到 CRM。

---

### Turn 13  *(2026-04-27 06:25)*

**User:**

summary:
{'anomaly_type': 'frequency', 'total': 2231, 'empty_sales_id': Decimal('0')}
{'anomaly_type': 'product', 'total': 1860, 'empty_sales_id': Decimal('0')}

**Assistant:**

这个结果说明：**本系统今天 `warning_records` 里没有空 `sales_id`**。按代码映射，发给 CRM 的 `employee_code` 应该也不是空。

所以现在优先怀疑两种情况：

1. 你查的 CRM 库数据不是今天这批 `biz_date=2026-04-27` 导出的。
2. CRM 接口收到 `employee_code` 后，在它们那边入库时没有保存，或字段映射/校验失败。

再查一下最新 Celery 实际跑的是哪个 `biz_date`，以及那批 `warning_ids` 的 `sales_id`：

```bash
docker compose exec -T celery python - <<'PY'
from sqlalchemy import create_engine, text
import json, os

engine = create_engine(os.environ["DATABASE_URL"])

with engine.connect() as c:
    runs = c.execute(text("""
        SELECT run_id, job_type, biz_date, status, total_anomalies,
               warning_ids, started_at, finished_at
        FROM detection_runs
        ORDER BY started_at DESC
        LIMIT 5
    """)).mappings().all()

    for run in runs:
        print("\nRUN", run["job_type"], run["biz_date"], run["status"], run["total_anomalies"])
        ids = json.loads(run["warning_ids"] or "[]")
        if not ids:
            continue
        rows = c.execute(text("""
            SELECT COUNT(*) total,
                   SUM(CASE WHEN sales_id IS NULL OR TRIM(sales_id) = '' THEN 1 ELSE 0 END) empty_sales_id
            FROM warning_records
            WHERE warning_id IN :ids
        """).bindparams(ids=tuple(ids))).mappings().first()
        print(dict(rows))
PY
```

如果这里也是 `empty_sales_id = 0`，那就能确定：**Celery 本次导出源数据没有空 employee_code**。

再随机看几条本系统会导出的值：

```bash
docker compose exec -T celery python - <<'PY'
from datetime import datetime
from zoneinfo import ZoneInfo
from sqlalchemy import create_engine, text
import os

biz = datetime.now(ZoneInfo("Asia/Shanghai")).date().isoformat()
engine = create_engine(os.environ["DATABASE_URL"])

with engine.connect() as c:
    rows = c.execute(text("""
        SELECT warning_id, anomaly_type, customer_id, customer_name,
               sales_id AS employee_code, sales_name AS employee_name
        FROM warning_records
        WHERE biz_date = :biz
        LIMIT 20
    """), {"biz": biz}).mappings().all()
    for r in rows:
        print(dict(r))
PY
```

如果这些都有 `employee_code`，那下一步就要让 CRM 那边查 `saveCustomerWarn` 接口的入参日志，确认他们收到的 JSON 里 `data[].employee_code` 是否为空。当前代码不会把非空 `sales_id` 改成空。

---

### Turn 14  *(2026-04-27 06:46)*

**User:**

可以帮我降低saveCustomerWarn的推送并发数，和降低batch_size吗？我认为是并发推送的结果太多，crm库没有保存下来导致的

**Assistant:**

我会按 `fastapi-python` 的配置路径来改：先确认当前默认值和 `.env` 里是否已经设置了推送批量/并发，然后把生产可控参数降下来，不碰敏感配置。

当前 `.env` 里还没有显式设置这两个参数，代码默认是 `batch_size=50`、`concurrency=20`，这个并发确实偏高。为了尽快止血，我准备把运行配置设成 `batch_size=10`、`concurrency=1`，先验证 CRM 是否能稳定落库。

我会只改运行参数，不改数据库和接口地址：在 `.env` 增加 `CHURN_WARNING_EXPORT_BATCH_SIZE=10` 和 `CHURN_WARNING_EXPORT_CONCURRENCY=1`，并把示例配置同步改成这个保守值。

配置已经写入 `.env`，不过 Celery 进程不会自动读取变更；需要重启 celery 容器后才生效。我先验证文件里只出现这两个运行参数。

要让参数生效需要重启 Celery。我先检查一下容器和当前 active 任务，避免在任务执行中间直接打断。

已把推送参数降下来了：

- [.env](/mnt/e/churn-warning-engine/.env:19)：`CHURN_WARNING_EXPORT_BATCH_SIZE=10`
- [.env](/mnt/e/churn-warning-engine/.env:20)：`CHURN_WARNING_EXPORT_CONCURRENCY=1`
- [.env.example](/mnt/e/churn-warning-engine/.env.example:37) 也同步成同样配置

含义是：每次最多推 10 条，并且串行推送，不再 20 并发打 CRM。

我这边没法直接重启容器，因为当前环境没有 `docker` 命令。你在服务器上执行：

```bash
cd /path/to/churn-warning-engine
docker compose up -d --force-recreate celery
```

如果你也会通过 API 手动跑 `detect-batch` 并触发 `saveCustomerWarn`，也重建 API：

```bash
docker compose up -d --force-recreate api celery
```

验证是否生效：

```bash
docker compose exec celery printenv | grep CHURN_WARNING_EXPORT
```

应该看到：

```text
CHURN_WARNING_EXPORT_BATCH_SIZE=10
CHURN_WARNING_EXPORT_CONCURRENCY=1
```

---

• beat 是 Celery 的定时调度器。它负责按 app/celery_app.py:19 里的 beat_schedule 到点“派发任务”到 broker，比如 Redis；真
  正执行任务的是 worker。

  我刚查了当前环境里的进程：

  pgrep -af "celery|beat"

  没有任何匹配输出，所以在这个当前机器/当前环境里，celery beat 没有启动。

  如果你只启动了：

  uvicorn app.main:app --reload

  那只是 API，不会跑定时任务。

  要启动定时派发，需要跑：

  celery -A app.celery_app worker --loglevel=info --beat

  这个命令会同时启动 worker 和内嵌 beat。生产环境通常会把 worker 和 beat 拆成两个独立进程。