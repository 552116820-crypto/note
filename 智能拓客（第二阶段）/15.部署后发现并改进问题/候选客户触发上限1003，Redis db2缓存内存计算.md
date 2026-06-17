# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `d130256c-4bc6-4aeb-bb1c-b678c46bd559`
- **范围**: Turn 1-11
- **导出时间**: 2026-06-17 15:04:40

---

### Turn 1  *(2026-06-17 03:48)*

**User:**

/api/v1/leads/score-and-info这个接口返回{"code":1003,"message":"候选客户过多，请缩小搜索半径","data":null}这是什么情况啊？候选客户上限是多少？

**Assistant:**

找到了错误码定义位置和接口入口。让我深入看错误抛出逻辑和上限配置。

现在找抛出点和上限值。

看一下查询实现和 service 调用上下文，确认候选定义和半径处理。

已经定位到完整逻辑，给你讲清楚。

## 候选客户上限：默认 **20000**

配置在 `app/lead/config/settings.py:55`：

```python
max_candidates: int = int(os.getenv("MAX_CANDIDATES", "20000"))
```

可通过环境变量 `MAX_CANDIDATES` 调整，默认 20000。

## 这是什么情况

接口二在**缓存未命中**时，会按你传的搜索半径去 CRM 捞所有 `type=0`（潜在客户）的客户来打分。判定逻辑在 `app/lead/repositories/geo_repo.py:85-86`：

```python
if len(rows) > settings.max_candidates:        # 超过 20000
    raise TooManyCandidates(len(rows), radius_km, settings.max_candidates)
```

链路是这样的：

1. `lead_service._compute_blob` → `geo_repo.fetch_candidates(lat, lng, radius_km)`（`lead_service.py:78`）
2. SQL 用 `LIMIT 20001`（`max_candidates + 1`，多取 1 个当哨兵探测溢出，`geo_repo.py:71`）
3. 半径圆圈内的潜客数 **> 20000** → 抛 `TooManyCandidates`
4. `app/main.py:191-193` 的异常处理器把它转成业务错误 `TooManyCandidatesError`（code 1003），**HTTP 状态是 200**，body 即你看到的 envelope

也就是说：**你这次请求的搜索半径内，潜在客户数量超过了 20000 个**。

## 几个要点

- **判定是严格大于**：恰好 20000 不报，第 20001 个才触发。
- **服务端不会自动缩半径重试**——设计上是宁可报错让用户缩，也不返回误导性的"部分页"（`geo_repo.py:6-7` 注释写明了这个取舍）。所以前端文案"请缩小搜索半径"就是唯一的处理路径。
- **半径参数**：`radius_km` 默认 60km、上限 200km（`RADIUS_MAX_KM`，`settings.py:53-54`）。半径越大、所在区域客户越密，越容易撞上限。

**怎么解决**：
- 业务上：让业务员把搜索半径调小（最直接，符合产品意图）。
- 运维上：若确认是上限设小了，调大环境变量 `MAX_CANDIDATES` 即可（注意这会让单次打分的候选规模变大，影响接口耗时和缓存 blob 体积）。

需要我帮你确认线上 `.env` 里 `MAX_CANDIDATES` 当前实际配的是多少，或者评估调大上限的性能影响吗？

---

### Turn 2  *(2026-06-17 03:52)*

**User:**

MAX_CANDIDATES改成多少合适，我直接在.env上改

---

### Turn 3  *(2026-06-17 03:52)*

**User:**

MAX_CANDIDATES改成多少合适，我直接在服务器的.env上改

**Assistant:**

要给一个能直接写进生产 `.env` 的数字，我得先把"上限调大会拖累什么"这条链路摸清楚——打分循环、tag 批查、Redis blob 体积、SQL 扫描。先把这些读透。

`_compute_blob` 是关键：cache-miss 时要对半径内**全部**候选打分、组装、排序，再把整张列表塞进一个 Redis blob。所以上限直接决定单次首屏的 CPU、那条巨型 `IN(...)` tag 查询、以及单个缓存 value 的体积。再确认几个缓存/内存相关的配置，好把数字钉准。

摸清了整条链路，几个关键事实让"调成多少"这件事比表面复杂——我先把代价讲清楚，再给数。

## 先说结论：20000 已经偏大，别盲目往上调

cache-miss 时（`_compute_blob`），半径内**全部**候选都要现算一遍——打分、组装成 ~20 字段的 dict、全局排序、再把**整张列表**塞进**一个** Redis blob。所以上限 N 直接线性放大四样东西：

| 候选数 N | 单个缓存 blob | 冷算耗时（geo+打分+序列化） | `IN()` tag 查询文本 |
|---|---|---|---|
| **20000（现状）** | ~12–20 MB | ~8–10s（已逼近锁窗口） | ~180 KB |
| 30000 | ~24 MB | ~12–15s | ~270 KB |
| 50000 | ~40 MB | ~20s+ | ~450 KB |
| 100000 | ~60–100 MB | 30s+ | ~900 KB |

（blob 按每候选 ~0.6–1KB 估；冷算按现状线性外推，"geo ~7–10s" 是 `settings.py:71` 注释里写死的真实锚点。）

往上调会撞到三个硬约束：

1. **缓存击穿锁窗口**：singleflight 锁 `CACHE_LOCK_TTL_SECONDS=20`、输家等待 `CACHE_LOCK_WAIT_SECONDS=15`。冷算一旦超过这俩，防击穿就失效、并发惊群冷算。现状 ~8–10s 已不宽裕,50000 必破。
2. **Redis 大 key + 内存**：`CACHE_MAX_KEYS=256`，单个 lead 缓存最坏堆 256 个 blob（现状峰值 ~4GB，50000→~10GB）。db2 与 db0/db1 同实例、`volatile-lru`——大 blob 在 LRU 下还会互相挤占、把刚算完的列表淘汰掉，命中率下降→更多冷算，恶性循环。
3. **首屏延迟**：page≥2 是纯切片不重算，但 page1 的 cache-miss 要等整个 N 算完才返回。

## 但根因多半不在上限，调大也治不到点上

两个代码事实：

- **业务员一次只看一页**（默认 20 条、最多 100）。算 2 万候选纯粹是为了全局排序取 top 几页，而 top 几乎都是"近距离 + 行业匹配"的客户。把上限提到 10 万，**前几页几乎不变**，冷算和 blob 却涨 5 倍——性价比极低。
- **默认距离曲线 60km 外距离分恒为 0**（`scoring.distance_score_default`，`DISTANCE_SCORER=default`）。而 `RADIUS_DEFAULT_KM=60`、`RADIUS_MAX_KM=200`。**如果触发 1003 的查询半径 > 60km，那 60km 以外的候选距离分全是 0、排序沉底、业务员永远翻不到，却实打实把你顶过了 20000**——纯成本、零价值。

所以 1003 的对症解通常是**调小半径**，不是调大上限：`RADIUS_DEFAULT_KM` 60 → 35~40，直接砍候选、消除报错，且不动缓存/锁、零额外内存代价。真要上调上限，**保守到 30000 为止**；再高必须联动调大 `CACHE_LOCK_*` 并核算 db2 内存。

为了给你钉准数值和要一起改的 env，确认两点：

收到：上调到 **30000**，触发场景待查。在给你确切操作前，先确认两件事——`.env` 里这个 key 现在怎么写的，以及"还不确定"时能从哪查触发分布。

关键信息齐了。`build_log_record` 记了 `biz_code`、`result_total`(候选总数)、`cost_ms`(冷算耗时) 和含 `radius_km` 的 `request_body`——这正好能回答你"还不确定"的场景。确认一下日志表的列名，好给你能直接跑的自查 SQL。

都确认了。日志表列：`id, employee_code, endpoint, request_body, biz_code, result_total, cost_ms, created_at`（`created_at` 库默认填充）。下面是可直接执行的方案。

## 1. 怎么改

服务器 `.env` 里把这一行（对应 `.env.example:122`）：

```
MAX_CANDIDATES=20000   →   MAX_CANDIDATES=30000
```

生产 `.env` 里若没有这行就新增一行。**纯运行时配置——不碰代码、不用 PR、不影响 526 测试基线**；代码默认值和 `.env.example` 保持 20000 即可（20000 是通用默认，30000 是这个部署的覆盖）。

> 你工作区里那个 `.env` 我没读也不动（含 DB 密码、API key），你在服务器自己改。

## 2. 怎么生效

`settings.py` 是 `@lru_cache get_settings()` + 启动时 `load_env()` 读一次环境 → **改完必须重启 API 进程才生效**（`docker compose restart api`，或你重启 uvicorn/gunicorn 的方式）。不重启读的还是旧值。churn（Celery）不碰这个接口，不用动。

## 3. 这次不用动别的 env

30000 是"**不联动其它 env 的安全上界**"，所以 `CACHE_LOCK_*` / `CACHE_MAX_KEYS` 都保持不变。记住这是上界——再往上就得回到 50000+ 的联动方案了。

## 4. 改完盯两个指标

- **冷算首屏 `cost_ms`**：别逼近 `15000`ms（=`CACHE_LOCK_WAIT_SECONDS`）。没破就说明 singleflight 防击穿还有效。
- **Redis db2 内存**：`redis-cli -n 2 info memory`。单 blob 会从 ~16MB 升到 ~24MB。

## 5. 红线

若改到 30000 **还**报 1003 → 该区域候选 > 30000 → **别再加上限**，转调小 `RADIUS_DEFAULT_KM`（60→35~40）。因为 60km 外距离分恒为 0，那批候选纯占名额、业务员永远翻不到。

## 6. 闭环你的"还不确定"

如果生产开了 `ENABLE_API_LOGGING=1`，`amer_sales_assist.lead_api_log` 直接能回答根因。三条 SQL：

**(a) 谁、用多大半径触发 1003**（普遍 vs 偶发）：
```sql
SELECT employee_code, request_body, cost_ms, created_at
FROM lead_api_log
WHERE endpoint = '/api/v1/leads/score-and-info' AND biz_code = 1003
ORDER BY id DESC LIMIT 100;
```
`request_body` 里有 `radius_km`/`lat`/`lng`——看是不是集中在某几个人或某个大半径。

**(b) 成功查询的候选规模分布**（看离 20000 多近、30000 够不够）：
```sql
SELECT COUNT(*) AS n, MAX(result_total) AS max_total,
       ROUND(AVG(result_total)) AS avg_total,
       SUM(result_total >= 15000) AS near_cap
FROM lead_api_log
WHERE endpoint = '/api/v1/leads/score-and-info' AND result_total IS NOT NULL;
```
（1003 时 `data=null` → `result_total` 为 NULL，这里只统计成功的；`result_total` 就是当时的候选总数。）

**(c) 大候选的冷算耗时**（验证 30000 不破锁窗口）：
```sql
SELECT result_total, cost_ms, created_at
FROM lead_api_log
WHERE endpoint = '/api/v1/leads/score-and-info' AND result_total >= 15000
ORDER BY result_total DESC LIMIT 50;
```
看接近 20000 那些的 `cost_ms`，线性外推到 30000 确认不逼近 15s。

若 `ENABLE_API_LOGGING=0`（默认关），表是空的——可以先打开观察几天，再据 (b) 的 `max_total` 决定 30000 是否真够。

---

一句话总结：**`.env` 改 `MAX_CANDIDATES=30000` → 重启 API → 盯 `cost_ms` 别过 15s + db2 内存；还报 1003 就改半径别加上限。**

我把这次的调优约束链记到项目记忆里，省得下次再翻一遍代码。

---

### Turn 4  *(2026-06-17 06:06)*

**User:**

触发1002是什么意思

**Assistant:**

1002 是 `InvalidCoordinates`（经纬度不合法），之前读 `errors.py` 时见过。但 1002 实际有两个触发来源，我确认一下校验逻辑再答。

1002 比字面"经纬度不合法"更宽——它其实是 lead 的**参数校验失败统称**。看一下 `main.py` 的异常处理器确认。

查清楚了。结论是：**1002 不只是"经纬度不合法"——它是 lead 模块所有"参数校验失败"的统称**，而且返回的 `message` 是动态的，会告诉你具体哪个字段错了。

## 1002 的真实来源

`errors.py:12` 里那个 `InvalidCoordinates`（message="经纬度不合法"）**其实当前代码里没有任何地方 `raise`**（全仓只有定义、没有抛出点）。坐标校验早就前移到了 Pydantic。线上真正产出 1002 的是 `main.py:180-188` 这个处理器：

```python
@app.exception_handler(RequestValidationError)   # 仅对 /api/v1/leads/* 生效
...
msg = f"参数校验失败: {loc} {err.get('msg', '')}".strip()
return JSONResponse(status_code=200, content=envelope(code=1002, message=msg, data=None))
```

也就是说：**任何请求体校验不过都返回 1002，HTTP 200，`message` 带上出错字段**。所以你看到的 message 不会是"经纬度不合法"，而是类似 `"参数校验失败: lat lat 必须在 [-90, 90]"`。

## 哪些情况会触发（`LeadQueryRequest` / skip 的校验，`schemas/request.py`）

| 字段 | 触发条件 |
|---|---|
| `lat` | 不在 [-90, 90]（`request.py:42-47`） |
| `lng` | 不在 [-180, 180]（`request.py:49-54`） |
| `radius_km` | ≤ 0 或 > 100000（`request.py:56-64`） |
| `employee_code` | 空 / 纯空白（`request.py:71-74`） |
| `page` / `page_size` | < 1（Field `ge=1`） |
| `industry_ids` / `industry_names` | 超过 100 项 |
| `customer_id`（skip 接口） | 非数字（`request.py:99-102`） |
| 任意字段 | **类型错误、缺必填字段、请求体不是合法 JSON**（Pydantic 原生） |

外加一个旁路：**请求体超过 1MB**（`MAX_REQUEST_BODY_BYTES`）也复用 1002，但那条是 **HTTP 413**（`shared/middleware.py:86-88`、`main.py:137-146`，仅 lead 路径）。

## 和 1003 的区别

- **1002 = 输入本身不合法**（前端传的参数有问题），HTTP 200，message 指明字段。
- **1003 = 参数都合法**，只是半径内候选 > `MAX_CANDIDATES` 上限（业务结果问题）。

## 排查钥匙

直接看返回的 **`message`**——它写明了哪个字段、什么原因。如果前端没回显 message，就查日志表：

```sql
SELECT employee_code, request_body, created_at
FROM lead_api_log
WHERE endpoint = '/api/v1/leads/score-and-info' AND biz_code = 1002
ORDER BY id DESC LIMIT 50;
```

`request_body` 里能看到前端到底传了什么（lat/lng/radius_km 等），一眼定位是哪个字段越界或缺失。最常见的就是**前端定位没拿到、传了 lat/lng = 0 或空**，或者经纬度传反（lat 填了经度值 → 超 90 → 1002）。

---

### Turn 5  *(2026-06-17 06:18)*

**User:**

触发1004是什么意思

**Assistant:**

1004 是 `InvalidCustomerId`（customer_id 不合法），http_status 200。确认它的抛出点和校验逻辑——这个是接口四（客户详情）相关的。

---

### Turn 6  *(2026-06-17 06:24)*

**User:**

我已经在.env 改 MAX_CANDIDATES=30000并重启 API，请问怎么看生效了没有？盯 cost_ms 别过 15s + db2 内存是什么意思

**Assistant:**

你已经动手了，那"怎么确认生效"就是最关键的一步——尤其有个 `docker compose restart` 的常见坑可能让你白改。先确认你们 compose 怎么注入 `.env` 和 Redis 配置，再给你能直接敲的命令。

关键信息都拿到了，其中有个**可能让你白改的坑**，先说这个。

## ⚠️ 先确认：你是怎么"重启 API"的？

你们 `api` 服务用的是 `env_file: .env`（`docker-compose.yml:33`）——`.env` 是在**容器创建时**注入成容器环境变量的，应用读的是这份注入值（不是容器内的 .env 文件）。这意味着：

- **如果你用了 `docker compose restart api` → 大概率没生效**。`restart` 只重启容器里的进程，**不会重新读 .env**，注入的还是旧的 20000。
- 正确做法：`docker compose up -d api`（compose 检测到 env 变化会重建容器并重新注入），保险起见加 `--force-recreate`：
  ```
  docker compose up -d --force-recreate api
  ```
- 例外：如果你不是 docker、而是裸 uvicorn 进程（systemd 等），那 restart 进程就够了——因为 `settings.py` 启动时 `load_env()` 直接读 .env 文件。

## 怎么确认生效（docker 场景，三层从外到里）

```bash
# 1) env 层：容器里到底注入了多少
docker compose exec api printenv MAX_CANDIDATES
#    → 30000 = .env 已进容器；还是 20000/空 = restart 没重读，回去 up -d

# 2) 代码层：应用 settings 实际加载的值（最权威）
docker compose exec api python -c "from app.lead.config.settings import settings; print(settings.max_candidates)"
#    → 打印 30000 就对了

# 3) 效果层：之前报 1003 的那个请求再打一次，若该区域候选在 2万~3万之间，现在应能正常返回
```

`/healthz` 不暴露这个值，所以靠上面 1、2 来确认，最干净。

## "盯 cost_ms"是什么意思

`cost_ms` 是接口流量日志（`lead_api_log`）里记的**每次请求服务端耗时（毫秒）**，中间件用 `perf_counter` 量的（收到请求→响应发出）。

为什么盯它、为什么是 15s：cache-miss（第一次查某个 `员工+坐标+半径`）要把半径内**全部候选**现算一遍（打分→排序→写缓存）。候选 2万→3万，这次冷算变慢。**15s = `CACHE_LOCK_WAIT_SECONDS`**：防缓存击穿的锁，同一 key 并发时只让一个去冷算、其余最多等 15s 就放弃等待、自己也算。**一旦冷算 > 15s，大家一起冷算 → 惊群 → DB 压力骤增**。所以那次冷算的 `cost_ms` 必须明显 < 15000。

注意只有 cache-miss 那次 `cost_ms` 大；命中（翻页、300s 内重复查）是纯切片，几十毫秒。所以挑 `result_total` 大的行看峰值（需 `ENABLE_API_LOGGING=1`）：

```sql
SELECT result_total, cost_ms, created_at
FROM lead_api_log
WHERE endpoint='/api/v1/leads/score-and-info' AND result_total >= 15000
ORDER BY cost_ms DESC LIMIT 30;
```

阈值：**< 10000ms 安心，10000~15000 警戒，≥15000 危险**（要么回退上限，要么联动调大 `CACHE_LOCK_TTL/WAIT_SECONDS`）。

没开日志的话，手动冷算计时一次（**新坐标才是冷算**，同参数第二次就命中了；本地 curl 按铁律必须 `--noproxy`）：
```bash
curl --noproxy '*' -s -o /dev/null -w 'cost=%{time_total}s\n' \
  -X POST http://127.0.0.1:8000/api/v1/leads/score-and-info \
  -H 'Content-Type: application/json' \
  -d '{"employee_code":"<某员工>","lng":<经度>,"lat":<纬度>,"radius_km":60}'
```

## "盯 db2 内存"是什么意思

Redis 的 **db2 是 lead 缓存库**（compose 第36行钉死 `REDIS_URL=redis://redis:6379/2`）。接口二把每个查询的整张候选列表存成一个 blob 放这。候选 2万→3万，单 blob 从 ~16MB 涨到 ~24MB。

**这里有个比我上次说的更紧的硬约束**：你们 Redis 整实例只给了 **512MB 上限**（`docker-compose.yml:20`，`--maxmemory 512mb --maxmemory-policy volatile-lru`），而且 **db0(broker)/db1(result)/db2 共用这 512MB**。所以我上次"峰值几个 GB"的说法不适用——真实天花板是 512MB，24MB 的 blob 满打满算只能放二十来个，还要和 Celery 键抢。blob 变大 → db2 能缓存的查询数变少 → 超了就 LRU 淘汰 → 命中率掉 → 更多冷算。

怎么看（redis 在容器里、无密码）：

```bash
docker compose exec redis redis-cli info memory | grep -E "used_memory_human|maxmemory_human"
docker compose exec redis redis-cli -n 2 dbsize                    # db2 缓存了多少个查询
docker compose exec redis redis-cli info stats | grep evicted_keys # 淘汰累计数，持续涨=内存吃紧
docker compose exec redis redis-cli -n 2 --bigkeys                 # 扫出最大的 blob 及字节
```

健康判断：`used_memory` 离 512MB 有余量（比如稳定 < ~350MB）、`evicted_keys` 基本不涨 → 安心。逼近 512MB 且 `evicted_keys` 持续上升 → db2 在抖动、缓存形同虚设，这时要么缩半径（减小 blob），要么把 redis `maxmemory` 调大（这是 compose 改动、要重建 redis，另说）。

---

一句话清单：**`docker compose up -d --force-recreate api`（别用 restart）→ `printenv` + `python -c` 确认是 30000 → 查 `lead_api_log` 大候选的 `cost_ms` 别过 15s → `redis-cli info memory` 看离 512MB 余量。**

> 顺带你刚问的 **1004**（`InvalidCustomerId`，"customer_id 不合法"）：和 1002 里那个 `InvalidCoordinates` 一样，这个类在当前代码里**没有任何地方 raise**——是预留但未启用的码。customer_id 传了非纯数字串时，实际被 schema 的 `_coerce_digit_id` 拦截、返回的是 **1002**（message 形如"参数校验失败: customer_id customer_id 必须为数字"），不是 1004。

我把 db2 那条 512MB 的硬约束补回项目记忆（上次估算写成了几 GB，是错的）。

重试——更新这条 memory，把 db2 内存的真实天花板（512MB）和 docker `restart` 不重读 .env 的坑补正。

记忆已更新好了（之前那次工具调用没成功，刚重试通过）。修正了两处之前不准确的地方：

- **db2 内存天花板**：上次我按 `CACHE_MAX_KEYS=256 × blob` 估成几 GB 是错的——真实硬上限是 Redis 的 `--maxmemory 512mb`（`docker-compose.yml:20`，且 db0/db1/db2 共用这 512MB）。所以 24MB 的 blob 顶多放二十来个，这才是要盯 `evicted_keys` 的原因。
- **重启方式**：补上了 docker `env_file: .env` + `restart` 不重读 .env 的坑，以及 `up -d --force-recreate` 的正解和 `printenv` / `python -c` 验证法。

你现在的下一步就是先确认那个重启坑——如果刚才是 `docker compose restart api`，很可能没生效，跑一下 `docker compose exec api printenv MAX_CANDIDATES` 一看便知。

---

### Turn 7  *(2026-06-17 06:33)*

**User:**

evicted_keys是什么

---

### Turn 8  *(2026-06-17 06:36)*

**User:**

evicted_keys是什么？ 24MB 的 blob 顶多放二十来个，这怎么理解？按 CACHE_MAX_KEYS=256 × blob 估成几 GB和真实硬上限是 Redis的 --maxmemory 512mb，这两个有什么区别与联系

**Assistant:**

这三个问题都是缓存内存的概念，我先确认一个关键事实——`CACHE_MAX_KEYS=256` 这个"256"在 Redis 模式下到底起不起作用，免得我解释又出偏差。读一下缓存实现。

读完缓存实现，发现一个比上次更关键的事实，正好能把你第三个问题（两个上限的区别）讲到根上。逐个回答。

## 1. evicted_keys 是什么

Redis `INFO stats` 里的一个运行统计：**因内存撞到 `maxmemory` 上限、被淘汰策略主动删掉的 key 的累计数量**。

- 机制：Redis 已用内存达到 `maxmemory`（你们 512MB）后又有新写入 → Redis 按 `maxmemory-policy`（你们 `volatile-lru`）挑一个 key 删掉腾地方 → 每删一个 `evicted_keys` +1。
- 它是**只增不减的累计值**（从 Redis 启动算起）。所以别看绝对值，看**增长速率**：一段时间内涨得快 = 内存吃紧、缓存在被反复挤掉；长期不动 = 内存够用。
- `volatile-lru` 只淘汰**带 TTL 的 key**（你们 lead blob 是 `setex` 带 300s TTL、`sf:` 锁也带 TTL;Celery broker 队列键无 TTL → 天然免疫。这正是 compose 选 volatile-lru 而非 allkeys-lru 的原因——别把任务消息淘汰没了)。
- 和我们的关系：blob 从 16MB 涨到 24MB，db2 更容易填满 512MB → 一满，新查询写入就淘汰最久没用的旧 blob → `evicted_keys` 开始涨 → 含义是"刚算好的缓存很快被挤掉，同样的查询下次又得冷算"，命中率掉。所以它持续上涨 = "内存不够、缓存在抖动"的信号。

## 2. "24MB 的 blob 顶多放二十来个"怎么理解

就是个除法：Redis 总内存 512MB，一个 30000 候选的结果 blob ≈ 24MB，512 ÷ 24 ≈ **21**。再多就把内存填满、触发淘汰。

而且实际更少，因为这 512MB 不是 db2 独享：

- **db0（Celery broker）/db1（result）/db2（lead 缓存）共用这一个 512MB 实例**——Celery 那部分先占一块；
- db2 里也不只有接口二的大 blob，还有 `emp:`（员工上下文）、`timeline:`、`recommend:`、接口四 detail 等 key；
- Redis 自身有数据结构和内存碎片开销。

所以"二十来个"是个量级感：**上限调到 30000 后，db2 实际只能同时缓存十几到二十个不同查询**。对比 20000 时单 blob ~16MB、512÷16≈32 个——这次调整把"可同时缓存的查询数"从 ~30 降到 ~20。如果你们活跃的 `员工×位置×半径` 组合数超过这个量，命中率就会下降、冷算变多。这就是要盯 `evicted_keys` 的实际意义。

## 3. 两个上限的区别与联系（这里我上次说错了，现在讲到根上）

它们是**两个不同层、不同单位、作用于不同缓存后端**的上限，根本不是同一个东西：

| | `CACHE_MAX_KEYS=256` | `--maxmemory 512mb` |
|---|---|---|
| 在哪定义 | 应用层（`app/shared/cache.py` + `.env`） | Redis 服务端（`docker-compose.yml:20`） |
| 单位 | **key 数量**（不管每个多大） | **字节**（不管多少个 key） |
| 作用后端 | **只有进程内缓存** `InProcessTTLCache` | **只有 Redis 后端** |
| 怎么淘汰 | 超 256 个 → 应用层 popitem 删最旧（`cache.py:57-58`） | 超 512MB → Redis 内核按 volatile-lru 删 |

**关键（我上次的错根）**：`build_cache` 里，**只要配了 `REDIS_URL` 就走 `RedisTTLCache`，而它的构造函数压根不接收 `max_keys`**（`cache.py:100-108` vs `209-223`）。`RedisTTLCache.set` 就是一句 `setex`，数量完全不限、纯靠 TTL + Redis 的 maxmemory 淘汰。

也就是说——**你们生产用 Redis，`CACHE_MAX_KEYS=256` 这个配置根本不生效**（它只在 Redis 挂掉、降级回进程内缓存时才起作用）。Redis 模式下 db2 能放多少 blob，**完全由 512MB ÷ blob 体积决定，没有"256 个"这回事**。

所以"256 × 24MB ≈ 6GB"错在两层：

1. **张冠李戴**——把"进程内缓存的数量闸(256)"乘到了"Redis 模式才有的大 value(blob)"上，这俩不在同一个后端；
2. 即便真有 256 个 key，**字节上限(512MB)也会远早于数量(256)触发淘汰**——512÷24≈21 就满了，数量永远摸不到 256。

**联系**：两者都叫"缓存容量上限"，但管的是不同后端、而且**同一时刻只有一个在生效**——配了 Redis 就只有 512MB（字节）说了算，`CACHE_MAX_KEYS` 是备用后端的数量闸、平时是死配置。**推论：想给 db2 扩容，只能调 Redis 的 `maxmemory`（compose 改动、要重建 redis），调 `CACHE_MAX_KEYS` 没有任何用。**

我把"256 仅进程内后端生效"这个根因也补进记忆，免得以后又拿它当 Redis 的闸。

---

### Turn 9  *(2026-06-17 06:44)*

**User:**

volatile-lru 和 allkeys-lru这两个是什么？--maxmemory 的 512mb可以调大吗

**Assistant:**

两个问题——第一个是概念，第二个你之前已经被我引到这了，正好闭环。

## 1. volatile-lru vs allkeys-lru

**LRU** = Least Recently Used，内存满时优先淘汰"最久没被访问过"的 key。这俩策略的唯一区别是**淘汰对象的范围**：

| 策略 | 淘汰范围 |
|---|---|
| `volatile-lru` | 只在**设了 TTL（过期时间）的 key** 里挑最久未用的淘汰；**没设 TTL 的 key 永不被淘汰** |
| `allkeys-lru` | 在**所有 key**（不管有没有 TTL）里挑最久未用的淘汰 |

**你们为什么必须用 volatile-lru**（也是铁律#4 把它冻死的原因）：一个 Redis 实例混放了三类数据，关键差异在有没有 TTL——

- **db0 Celery broker**：排队的任务消息，**没有 TTL**（任务要一直待到被消费）；
- **db2 lead 缓存**：`setex` 带 300s TTL；`sf:` 锁也带 TTL。

如果用 `allkeys-lru`：内存一满，Redis 可能挑中 db0 里**没 TTL 的任务消息**淘汰掉 → **业务员的检测任务静默消失、还不报错**。
用 `volatile-lru`：只淘汰带 TTL 的（lead 缓存、锁），broker 任务消息因无 TTL 而免疫 → 内存压力只牺牲"丢了能重算"的缓存，绝不丢任务。所以 compose 注释写"严禁 allkeys-lru"。

（Redis 共 8 种策略，还有 `*-lfu` 按访问频率、`*-random`、`volatile-ttl` 按剩余寿命、`noeviction` 满了直接拒写。你们的场景 volatile-lru 是对的，不用换。）

## 2. 512mb 可以调大吗

**可以，而且不违反铁律#4**——铁律#4 冻结的是"**分库结构 + `volatile-lru` 策略**"，**没有冻结 `maxmemory` 这个数值**。只要不动 policy、不动分库，调大这个数是允许的。

**怎么调**：改 `docker-compose.yml:20` 的 command（不是 `.env`）：
```yaml
command: ["redis-server", "--maxmemory", "1gb", "--maxmemory-policy", "volatile-lru"]
```
然后 `docker compose up -d redis`（command 变了，compose 会重建容器使其生效；`restart` 不行，command 是容器创建时定的）。

**调多大看宿主机物理内存**——Redis 是纯内存进程，maxmemory 不能超过服务器可用内存（要给 api/celery/nginx 容器和系统留余量）。先在服务器上看：
```
free -h
```
- 内存充裕（如 ≥8GB）→ 调到 **1GB** 很安全（db2 能稳定缓存 ~40 个 24MB blob），观察 `evicted_keys` 是否平稳；不够再上 2GB。
- 内存紧张（2~4GB 小机器）→ 别硬加，回到"缩半径减小 blob"那条路更稳。

**⚠️ 重建 redis 会清空数据**（必须低峰期做）：你们 redis **没配持久化卷**（compose 里 redis 无 `volumes`、无 `--save`/`--appendonly`），是纯内存。所以 `up -d redis` 重建（甚至单纯 `restart redis`，因无持久化）都会**清空整个 Redis**：

- **db2 lead 缓存清空** → 重建后一阵子查询都冷算、有性能抖动，但会自愈（缓存重新填充）；
- **db0 broker 清空** → **排队中未执行的 Celery 任务消息会丢**。churn 的周期检测任务由 beat 下个周期自动重调、能补上，但"此刻在队列里等执行"的那批会丢。

操作前可瞄一眼队列深度确认没有大量在途任务：
```
docker compose exec redis redis-cli -n 0 dbsize          # db0 broker 的 key 数
docker compose exec redis redis-cli -n 0 keys 'celery*'  # 看任务队列键（生产 key 少时可用；key 多别用 keys，改 scan）
```

**我的建议**：先 `free -h`。内存够就把 maxmemory 调到 1GB、低峰期重建、之后盯 `evicted_keys` 看是否归零——这比单纯把候选上限顶到 30000 更能根治"缓存被大 blob 挤爆"。内存不够就别加 redis 内存，改走缩半径。

把"maxmemory 可调大、不违反铁律#4、但重建清空 broker"这条扩容路径补进记忆。

记忆已补好。小结这一串排查的落点：

- **生效验证**：先排除 `docker compose restart` 不重读 `.env` 的坑 → `up -d --force-recreate api` → `printenv` / `python -c` 确认是 30000。
- **30000 的代价**：单 blob ~24MB，512MB 的 Redis 实际只能同时缓存 ~20 个查询，盯 `cost_ms`(<15s) 和 `evicted_keys`(别持续涨)。
- **如果缓存被挤爆**：根治有两条——内存够就 `free -h` 后把 redis `maxmemory` 调到 1GB（低峰期重建，会清空 broker 在途任务）；内存不够就缩 `RADIUS_DEFAULT_KM`。两条都比再往上顶候选上限稳。

需要的话，等你 `free -h` 和 `redis-cli info memory` 的实际数字出来，我可以帮你定具体把 maxmemory 设多少、或判断到底该加内存还是缩半径。

---

### Turn 10  *(2026-06-17 06:52)*

**User:**

怎么看宿主机物理内存；free -h 和 redis-cli info memory，这两个怎么查

**Assistant:**

这俩都是在**生产服务器上**敲的命令（不是 WSL 开发机）。逐个说，重点是输出怎么读。

## 1. `free -h` —— 看宿主机物理内存

直接在服务器 shell 跑（`-h` = 人类可读，自动 GB/MB）：

```bash
free -h
```

输出长这样：

```
              total        used        free      shared  buff/cache   available
Mem:           15Gi       4.2Gi       1.1Gi       300Mi       9.7Gi        10Gi
Swap:         2.0Gi          0B       2.0Gi
```

怎么读（**别只看 `free` 那列，会误判**）：

| 列 | 含义 |
|---|---|
| `total` | 物理内存总量（这台机器有多少 RAM） |
| `used` | 真正被进程占用的 |
| `free` | **完全空闲**——这个数通常很小，别慌：Linux 把闲置内存拿去做磁盘缓存了 |
| `buff/cache` | 内核的文件缓存，**需要时可随时回收** |
| **`available`** | ⭐ **最该看的**——预估"现在还能拨给新进程多少内存"（已含可回收的 buff/cache） |

**判断能不能给 redis 加内存，就看 `available`**：比如 available 有 10Gi，给 redis 调到 1~2GB 很安全；如果 available 只剩几百 MB，就别加。

补充：
- `free -h -s 2` 每 2 秒刷新看波动（Ctrl+C 退出）；`free -g` 用整数 GB。
- 万一这台机器没装 `free`（极简系统）：`grep -i memavailable /proc/meminfo`。

**配合看容器各自占多少**（决定"还能匀给 redis 多少"），在 compose 项目目录跑：

```bash
docker stats --no-stream
```
（`--no-stream` = 打一次快照就退出，不加会实时刷新）。会列出 api / celery / redis / nginx 每个容器的 `MEM USAGE / LIMIT`。

## 2. `redis-cli info memory` —— 看 Redis 内存

redis 在容器里、无密码，所以经 compose 进去跑（**要在有 `docker-compose.yml` 的目录下**）：

```bash
docker compose exec redis redis-cli info memory
```

关键字段（输出有几十行，挑这几个看）：

```
used_memory_human:117.74M       ← 当前已用
used_memory_peak_human:210.00M  ← 历史峰值（是否曾顶到上限）
used_memory_rss_human:130.00M   ← 操作系统视角占用（含碎片）
maxmemory_human:512.00M         ← 上限，应是 512M（确认 compose 生效）
maxmemory_policy:volatile-lru   ← 淘汰策略，应是 volatile-lru
mem_fragmentation_ratio:1.10    ← rss/used 碎片率，>1.5 才需留意
```

**判断吃不吃紧：拿 `used_memory_human` 对比 `maxmemory_human`（512M）**——稳定在 350M 以下宽裕，逼近 512M 就紧张。`used_memory_peak_human` 看历史是否撞过顶。

## 一键体检（复制粘贴，把判断要用的都打出来）

```bash
echo "===== 宿主机内存 ====="; free -h
echo "===== 容器占用 =====";   docker stats --no-stream
echo "===== Redis 内存 =====";  docker compose exec redis redis-cli info memory | grep -E "used_memory_human|used_memory_peak_human|maxmemory_human|maxmemory_policy"
echo "===== 淘汰/命中 =====";   docker compose exec redis redis-cli info stats | grep -E "evicted_keys|keyspace_hits|keyspace_misses"
echo "===== 各库 key 数 ====="; docker compose exec redis redis-cli info keyspace
```

输出里你重点对照这三件事：

1. **`free -h` 的 `available`** 够不够大 → 决定能不能给 redis 加内存；
2. **`used_memory_human` vs 512M** + **`evicted_keys` 是否在涨** → 现在缓存吃不吃紧；
3. **`info keyspace` 里 `db2:keys=…`** → db2 当前缓存了多少个查询（30000 上限下这个数会比以前小）。

把这几行输出贴给我，我就能帮你定：到底该把 maxmemory 调到多少，还是其实该缩半径。

---

### Turn 11  *(2026-06-17 06:58)*

**User:**

D:\note\智能拓客（第二阶段）\15.部署后发现并改进问题\服务器的内存使用情况.md，我把结果放在这个文档里了

**Assistant:**

好消息——这组数据把"要不要给 redis 加内存"这个问题直接解决了：**不用调，512MB 绰绰有余，省了重建 redis 的麻烦。**

## 数据解读

**宿主机**（`free -h`）：total **30Gi**、available **24Gi**——内存极其充裕。顺带看出这是台混部机，除了 amer_sales_assist，还跑着一套 Dify（`docker-plugin_daemon/weaviate/sandbox/db_postgres…`）和几个 `wecom_aibot`。但 amer 这套很轻：api 59M + nginx 11M + celery 168M + redis 8.5M ≈ **总共才 250M**。

**amer 的 redis**（`info memory`）：

| 字段 | 值 |
|---|---|
| `used_memory_human` | **1.77M** |
| `used_memory_peak_human` | **8.50M**（历史峰值） |
| `maxmemory_human` | 512.00M |
| `maxmemory_policy` | volatile-lru ✓ |

**当前用 1.77M / 512M，历史峰值才 8.5M——离上限差 60 倍。** redis 内存毫无压力。

## 为什么这么空（关键）

我之前担心的"24MB blob 挤爆 512MB、只能放 20 个"，在你的**真实负载下根本不存在**，原因有三：

1. **触发 1003 的大查询不写缓存**——`TooManyCandidates` 是在 `geo_repo.fetch_candidates` 阶段就抛出的，根本走不到打分和写 blob。所以越是会撑大内存的大查询，越不会留下 blob。
2. **峰值才 8.5M 说明成功查询的候选数都不大**——如果真有 30000 候选的 24M blob 存进去过，peak 至少会到 24M+。
3. 可能 lead 流量本身不高，或缓存 300s TTL 过期了 / 最近重建过还没填回来。

## 结论与建议

- **`maxmemory` 不要动**。没必要，还能避免重建 redis 清空 db0 broker 在途任务的风险。这条路堵死，是好事。
- **30000 这个上限保留即可**，内存这条警报解除。接下来只需盯**两件和内存无关的事**：
  1. **行为验证**——拿原来报 1003 的那个查询再打一次,确认现在能返回了；
  2. **冷算 `cost_ms`**——候选接近 2~3 万时的耗时别破 15s（这是 CPU/DB 耗时，不是内存问题）。
- **附带一个判断**：redis 这么空、peak 才 8.5M，说明实际上**极少有查询接近候选上限**。这给你之前"还不确定触发场景"提供了线索——**1003 大概率是偶发**（个别密集城区或某人传了大半径），不是普遍现象。如果确实偶发,30000 多半够用,甚至当初缩半径就能解决。

> 一个小提醒:`info memory` 里 `mem_fragmentation_ratio:7.64` 看着吓人,**忽略它**——数据量才 1.77M 时,jemalloc 的基础驻留(startup ~948K)让这个比率天然虚高,实际浪费才约 11M,无意义。数据量大了它会自己回落。

把这组实测结论补进记忆——主要是"redis 实际负载极低、maxmemory 无需调"这个结论,免得下次又纠结加内存。

---
