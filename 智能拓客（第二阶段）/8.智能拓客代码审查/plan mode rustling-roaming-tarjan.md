# 智能拓客 代码审查报告（业务逻辑 + 性能）

## Context

用户要求审查 `smart_lead_generation`（智能拓客）项目，关注**业务逻辑是否正确**与**性能优化是否到位**。
这是一个 FastAPI + PyMySQL（无 ORM）+ 可选 Redis 的同步服务，核心是"按地理位置取候选客户 →
四维打分排序 → 缓存 → 分页"，外加客户行为时间轴（调 CRM 三接口）和大模型推荐理由（DeepSeek，可降级模板）。

本报告基于**完整通读核心文件**（不只是片段），已对三个探索 agent 的初步结论逐条核实、去伪存真。

---

## 总体评价

代码质量**明显高于一般业务项目**：容错（全程降级、必返 envelope）、防缓存击穿（singleflight）、
防注入（行业 id 强转 int、IN 占位符）、防提示词注入（时间轴标记不可信 + `_san` 压平分隔符）、
CRM 状态判断避开 `1==True`/`0==False` 陷阱、"伪成功"检测、`COUNT(DISTINCT vbillcode)` 去重多行单据——
这些都做得很到位。下面是**真问题**与**需业务确认的设计点**，多数是打磨级，少数值得动手。

---

## 一、业务逻辑发现

### B1〔中〕距离评分写死 60km，但半径上限是 200km —— 需业务确认
- `scoring.py:85-91` `distance_score_default(d_km, radius_km)` **忽略 `radius_km`**，硬编码 `≤20→40`、
  `20–60 线性`、`≥60→0`。而 `settings.radius_max_km=200`、`radius_default_km=60`（`settings.py:51-52`）。
- 后果：当调用方显式传 `radius_km > 60`（如 100/200），60–200km 的候选**距离分恒为 0**，该波段内
  排序只剩 行业+规模+标签（距离这 40 分变成常量、失去区分度），且候选集可能逼近 `MAX_CANDIDATES=20000`
  触发 `TooManyCandidates`，同时把首包拉到 7–10s（见 P5）。
- 注意：**默认半径 60 时曲线恰好在边界归零，是自洽的**；问题只在前端/调用方允许选 >60km 时出现。
- 已有 `distance_score_normalized`（`scoring.py:94-103`）能把衰减拉伸到实际半径，但**不是默认**。
- 结论：这不是 bug，是设计取舍。需确认：UI 是否允许 >60km？若 60km 就是业务意图的最大触达，
  建议下调 `radius_max_km`；若要支持更大半径，建议默认切 `normalized` 或明确文档化"超 60km 距离中性"。

### B2〔低〕时间轴排序时间戳跨格式时区不一致
- `timeline_service.py:58-81` `_to_sort_ts`：纯数字时间戳是绝对值（UTC）；带时区的 ISO 串正确；但
  **不带时区的 naive 串**（`fromisoformat` 无 tz、或 `strptime` 那几种格式）走 `.timestamp()` 会按
  **服务器本地时区（生产是 UTC）**解释。
- 若 CRM 三个源返回的时间格式不一致（例如订单是 epoch 毫秒、拜访是 `"2024-01-01 10:00:00"` 北京 naive），
  naive 的会比绝对值偏 8h → 相近时间的事件可能**错排**。仅取最近 5 条，影响有界，但确有可能把该上榜的挤掉。
- 作者注释"排序用，绝对时区无所谓"——这个判断**仅当三源同格式才成立**，探针只验了字段名、没保证格式统一。
- 建议：naive 解析统一按 `crm_passport_tz`（Asia/Shanghai）赋时区再取 epoch；或用探针确认三源格式一致。

### B3〔低/统一性〕"无行业"哨兵在三处处理不一致
- `lead_service.py:100` 展示值用 `int(ind_id) if ind_id not in (None, 0) else "-"`，**没兜 `"0"` 字符串**；
  而 `scoring.py:67` 和 `industry_resolver.py:100` 都兜了 `(None, 0, "0")`。
- 若该列偶发返回字符串 `"0"`（int 列，实际概率极低），展示会存成整数 `0` 而非 `"-"`，与打分/名称口径不一。
- 几乎无实际影响，但是三处口径不统一的隐患，建议抽一个 `_is_empty_industry(x)` 共用。

### 已核实的"非问题"（探索 agent 过度解读，特此澄清）
- **"行业过滤类型不匹配导致错误排除"**——不成立。`idset = {int(x)}`（`lead_service.py:145`）与存储的
  `amer_industry_id`（int）类型一致能匹配；`"-"` 永不命中是**正确**行为。过滤逻辑无误。
- **"skip 集合读失败静默当空是 bug"**——这是**有意降级**（`lead_service.py:121-125`，且文档化）。
  唯一代价：LOG_DB 抖动期间被跳过的客户会短暂重现（UX 取舍，非 bug）。

---

## 二、性能发现

### P1〔中〕进程内缓存 `_key_locks` 缓慢内存泄漏（影响默认后端！）
- `store.py`：`_key_lock(key)`（62-67）为每个 key 建一把锁存进 `_key_locks`；**只在 `set()` 的 LRU 淘汰
  时**才 `pop`（88 行）。但 **TTL 过期走的是 `get()`（76-77），只 `pop` 了 `_data`、没动 `_key_locks`**；
  `compute()` 抛异常时该 key 也从不 `set`，锁同样残留。
- `lead:` key 是 (业务员, lat, lng, radius) 维度、长期看无界。常态是 TTL 过期而非 LRU 淘汰 → `_data` 维持
  在上限、`_key_locks` 却只增不减。跑几天就是实打实的（慢）泄漏，且默认后端就是它。
- 修复（低成本）：`get()` 命中过期时一并 `pop` `_key_locks`；或周期性把 `_key_locks` 裁剪到 `_data` 现存 key；
  或改用固定大小的 striped lock 数组替代 per-key 锁。

### P2〔中〕时间轴串行调 3 次 CRM
- `timeline_service.py:190-193`：orders→visits→services **顺序**调，各 `crm_http_timeout=5s` + 1 重试。
  冷路径延迟 ≈ 三者之和，最坏可达 ~30s，直接影响 `/timeline` 与 `/recommend-reason` 首包。
- 同步 handler 无法 `asyncio.gather`。建议用一个小 `ThreadPoolExecutor` 把三调用并发（保持同步代码、改动小），
  冷路径降到 ≈ max(三者)。（彻底方案是改 async + httpx.AsyncClient，但改动大。）

### P3〔低-中〕带行业过滤时每次翻页都重扫全量（≤20k）
- `lead_service.py:169-174`：过滤生效时，**每个请求（含第 2、3 页…）都对整份缓存 items 跑两个
  推导式**（最多 2 万次），抵消了"page≥2 纯切片"的设计。无过滤路径仍是纯切片、没问题。
- ~2 万次迭代约几毫秒，不致命。可选：按 (cache_key, idset) 把过滤结果也缓存一个 TTL；或接受现状。

### P4〔中/视规模〕无 DB 连接池，冷请求跨两台主机开 ~5 条新连接
- `pool.py`/`log_pool.py`：每次查询**新建 pymysql 连接** + 重试（`sleep(min(2*attempt,5))` 退避）。
  一次冷 `_compute_blob` = 员工 ncc 计数(1)+按 ncc 查客户(1)+geo(1)+tags(1)+skip 集合(1，**在另一台主机**) ≈ 5 次握手，
  跨主机链路（已知 MTU 脆弱）下握手开销叠加。
- 缓存把热路径摊到 0 次 DB，作者也声明"当前流量够用"。当前规模可接受，但这是下一个瓶颈；
  并发上来后建议上连接池（如 DBUtils PooledDB）。另注：DB 抖动时 3 次重试退避最多给请求加 ~9s 才报 DBUnavailable。

### P5〔可接受/需文档化〕地理查询 2×LEFT JOIN + Haversine + filesort，整批取回（≤20k）
- `geo_repo.py:27-53`：首包 7–10s 的主因。bounding box 走 (lat,lng) 索引，但 `HAVING`/`ORDER BY` 落在计算列
  `distance_km` 上 → 对框内全部行算距离 + filesort，再整批取回内存。这是"全量打分、绝不静默截断"的固有代价
  （`TooManyCandidates` 兜底）。主要调节杆是半径（与 B1 联动）：若 `radius_max` 仍是 200，城区查询会常态逼近 2 万上限。

### P6〔低/边界〕Redis singleflight 在"结果不可缓存"时退化
- `store.py:168-209` + `get_or_compute_cacheable`：CRM 降级时时间轴/理由结果不缓存，同客户并发请求
  无法享受 singleflight，等待者轮询到 `cache_lock_wait_seconds=15s` 后各自计算，**实际串行**地各跑 3 次 CRM，
  CRM 抖动期延迟被放大。代码 docstring（171 行）已自承。可接受，合并时需知悉。

---

## 三、与 churn-warning-engine 合并的交叉影响（结合记忆）

记忆记录：拓客将与 churn 合并，真正的坑是 **Redis 淘汰策略冲突**（churn 的 Celery 可能要 `noeviction`，
本缓存适合 `allkeys-lru`）。本次审查补充两条代码层事实，供合并决策：

- **本服务对 Redis 完全是"缓存 + 短 TTL 的 SETNX 锁"**：能容忍淘汰（miss 就重算；`set` 失败已 try/except 吞掉
  降级为不缓存）。所以从拓客这侧看，`allkeys-lru` 最佳，`noeviction` 也安全（OOM 时 set 抛错被吞）。
- **关键发现：`RedisTTLCache` 不强制任何 key 上限**（`store.py:148-150` 只 `setex`），`cache_max_keys=256`
  **仅对进程内后端生效**（221 行）。即 Redis 后端下 blob 数量只由 `TTL × 到达率` 约束。单个 blob 可能很大
  （2 万条 items）。**若被迫与 churn 共用一个 `noeviction` 实例**，繁忙的拓客可能把实例顶到 OOM。
  → 这是"应给缓存单独实例 / 或保持 allkeys-lru"的有力论据（与记忆里"分库只隔离 keyspace、解决不了淘汰冲突"一致）。

---

## 实施方案（已选：修掉已确认的真问题，不动 B1）

落地顺序与具体改法。每项都小而局部、配套测试，不改对外 envelope 契约。

### 1. P1 — 修进程内缓存锁泄漏（`app/cache/store.py`）
根因：`_key_locks` 每次 miss 都新增锁对象，但**只有 `set()` 的 LRU 淘汰**才清理；TTL 过期（走 `get()`）
与 `compute()` 抛异常（key 从不进 `_data`）两条路径都不清 → 无界增长。
改法：`self._key_locks` 由普通 dict 换成 `weakref.WeakValueDictionary`，锁用可弱引用的小包装：
```python
class _RefLock:
    __slots__ = ("lock", "__weakref__")
    def __init__(self): self.lock = threading.Lock()
```
`_key_lock()` 返回 `_RefLock`，调用处改为 `rl = self._key_lock(key); with rl.lock:`。计算进行中，调用方
栈上持有强引用 → 字典条目存活、并发线程拿到**同一把**锁（singleflight 语义不变）；待所有持有/等待者
释放后条目自动回收 → TTL 过期与异常两条泄漏路径都自愈。同时删掉 `set()` 里现在多余的
`self._key_locks.pop(evicted, None)`。
> 备选：固定大小 striped-lock 数组（更简单，但不同 key 偶发假争用、会让无关冷算互等）。选 WeakValueDictionary
> 以保持精确 per-key 语义、零行为回归。
测试：大量唯一 key 过 TTL 后断言 `len(cache._key_locks)` 不随之线性增长；`compute()` 抛异常后该 key 锁不残留。

### 2. P2 — 时间轴三 CRM 调用改并发（`app/services/timeline_service.py`）
`_compute_timeline` 内 orders/visits/services 改用 `ThreadPoolExecutor(max_workers=3)` 并发提交；
`passport`/`today` 仍在并发**前**算一次共用（不破坏跨零点一致性）。把 `_safe_fetch` 拆成"提交 future"与
"收取 future"两步——`sources[name]` 一律在主线程 `f.result()` 之后写入（无 dict 竞态）；`httpx.Client` 线程安全。
冷路径墙钟从 ≈三者之和 → ≈max(三者)，直接降 `/timeline` 与 `/recommend-reason` 首包。
测试：mock 三接口各 sleep ~0.5s，断言 `_compute_timeline` 墙钟 < 1s；单源抛错仍 `degraded=true` 且其余正常返回。

### 3. B2 — 时间轴 naive 时间统一赋时区（`app/services/timeline_service.py`）
`_to_sort_ts` 中 `fromisoformat`/`strptime` 得到的 **naive** datetime，统一 `.replace(tzinfo=_SORT_TZ)` 再
`.timestamp()`；`_SORT_TZ = ZoneInfo(settings.crm_passport_tz)` 提为模块级常量。带时区的 ISO 串与纯数字
epoch 维持原样。消除"某源带时区、某源 naive"导致的 8h 错排。
测试：构造 epoch 与北京 naive 串表示**同一时刻**，断言两者 `_to_sort_ts` 相等（误差 < 1s）。

### 4. B3 — 统一"无行业"哨兵（`app/services/scoring.py` + 三处调用）
在叶子模块 `scoring.py` 加 `def is_empty_industry(v) -> bool: return v in (None, 0, "0")`，替换四处口径不一的判断：
`scoring.score_industry`(67)、`industry_resolver.id_to_name`(100)、`lead_service.py:100`、`employee_repo.py:75`
（后者由 `(None, 0)` 收紧到也含 `"0"`，更一致；无 import 环：scoring 仅依赖 stdlib）。
测试：`is_empty_industry` 单测；补 `lead_service` 装配在 `ind_id="0"` 时 `amer_industry_id` 落 `"-"`。

### 5. P3 — 行业过滤结果做进程内小缓存（`app/services/lead_service.py`）
新增一个**仅进程内**的 `InProcessTTLCache`（不接 Redis，避免加重已发现的 Redis 大值/无上限问题），
键 = `f"{cache_key}|{sorted(idset)}"`，值 = `(filtered_items, unseen_total)`，TTL 同 blob、容量小（如 64）。
`query_leads` 过滤前先查该 memo，命中直接用、未命中算一次再存。覆盖"同一筛选连续翻页"的常见场景，
去掉每页对 ≤2 万条的重扫。
> 这是本轮里收益最低的一项、且完全本地隔离。如倾向保持简单，可只做 1–4、跳过本项——不影响其余修复。
测试：同 (cache_key, idset) 翻第 2 页时断言过滤只执行一次（打桩计数）。

### 本轮范围外（已记录、暂不动）
- **B1 半径/距离**：需业务先定方向（下调 `radius_max_km` / 默认换 `normalized` / 文档化），属设计决策。
- **P4 连接池、P5 geo 查询**：当前规模可接受，列"观察项"，QPS 上升再动。
- **Redis 无 key 上限 + 与 churn 合并的淘汰策略**：属合并期部署/改造决策，留给合并任务统一处理。

---

## 验证方式

- **跑库前先修 MTU**（记忆/README）：`sudo ip link set dev <198.18.x 网卡> mtu 1400`；WSL 重启会回 9000
  导致大查询黑洞，网卡名不稳定（eth0/1/2 乱跳），`load_test.py` 已自动探测。
- 单元测试：`pytest`（scoring/cache/crm/recommendation 等 18 个测试文件，纯函数与缓存逻辑可离线验）。
- 集成：`scripts/integration_check.py`（resolver→geo→employee→score→paginate 全链）。
- 压测：`scripts/load_test.py`（冷/热/stampede/timeline/recommend/skip/混合九阶段，含 singleflight 计数证明）。
- P1 验证：构造大量唯一 key + TTL 过期，断言 `_key_locks` 不随过期增长。
- P2 验证：mock CRM 三接口各 sleep 1s，断言 `_compute_timeline` 墙钟 ≈1s 而非 3s。

---

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design,
  codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help,
  verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent Explore：已用于并行摸清架构/性能/业务逻辑（本报告基础）。
- Agent Plan：进入修复阶段时设计具体改法（P1/P2 等）。
- Skill code-review / simplify：修复产生 diff 后复核（针对改动，不是对存量做 diff）。
- Skill verify：改完后跑应用验证行为（受 DB 隧道/MTU 约束）。

**考虑过但不用**：
- codex:rescue：用于把疑难调试甩给 Codex；本次问题已定位清楚，不需要。
- security-review：本次聚焦业务逻辑+性能，非安全审计（虽顺带记录了若干安全做得好的点）。
- MCP chrome-devtools / firecrawl / jina / tavily、deep-research：均为 Web/浏览器/联网研究，与后端存量代码审查无关。
