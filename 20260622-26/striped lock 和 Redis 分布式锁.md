# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `44937c95-5202-4d08-a723-fe9c7cb7fe2d`
- **范围**: Turn 1-3
- **导出时间**: 2026-06-09 17:51:27

---

### Turn 1  *(2026-06-09 09:06)*

**User:**

striped lock是什么

**Assistant:**

**Striped lock（分段锁 / 条带锁）** 是一种并发控制手法，介于"一把全局大锁"和"每个 key 一把锁"之间的折中。

## 核心思路

维护一个**固定大小 N 的锁数组**，对任意 key 通过哈希映射到某一把锁：

```
lock = locks[hash(key) % N]
with lock:
    # 只锁这一条带，不影响映射到其它条带的 key
    ...
```

## 解决什么问题

| 方案 | 并发度 | 锁数量 | 问题 |
|------|--------|--------|------|
| 全局一把锁 | 差 | 1 | 不相关的 key 互相阻塞 |
| 每 key 一把锁 | 最好 | 无上界 | key 多时内存爆炸、锁本身要管理/回收 |
| **Striped lock** | 好（可调） | 固定 N | 哈希冲突导致"假争用" |

它用**有界的锁数量**换来**接近 per-key 的并发度**：映射到不同条带的 key 完全互不阻塞，N 越大冲突越少、内存越多 —— N 是个调优旋钮。

唯一代价是**假争用（false contention）**：两个逻辑无关的 key 恰好哈希到同一条带时，仍会互等。

## 经典实现

- **Java `ConcurrentHashMap`**（JDK 8 之前）：基于 Segment 的分段锁，就是这个思想。
- **Guava `Striped<Lock>` / `Striped<ReadWriteLock>`**：把它做成了显式 API。

## 在你这个项目（Python）里的典型用途

Python 有 GIL，所以它不是为了 CPU 并行，而是用来**串行化对同一实体的操作、同时放行不同实体**。比如：

```python
import threading

_STRIPES = 64
_locks = [threading.Lock() for _ in range(_STRIPES)]

def lock_for(lead_id: str) -> threading.Lock:
    return _locks[hash(lead_id) % _STRIPES]

# 同一个 lead 的并发写互斥，不同 lead 并行
with lock_for(lead_id):
    update_lead(lead_id)
```

这种模式很适合"按 lead_id / user_id 加锁但不想为每个 id 建锁"的场景 —— 比如你内存里那套进程内缓存回退、或合并后多任务竞写同一条记录时。asyncio 下把 `threading.Lock` 换成 `asyncio.Lock` 同理。

如果你是在某段代码里看到了 striped lock 想确认它具体怎么用的，告诉我文件，我帮你看。

No response requested.

---

### Turn 2  *(2026-06-09 09:35)*

**User:**

store.py里有striped lock

**Assistant:**

确认了 —— `app/cache/store.py:69` 就是我上面讲的那套，用在 `InProcessTTLCache` 里，但它的用途是**singleflight（防缓存击穿 / 请求合并）**，不是普通互斥。

## 它在这里具体干什么

```python
_LOCK_STRIPES = 2048
self._stripes = tuple(threading.Lock() for _ in range(_LOCK_STRIPES))

def _key_lock(self, key):
    return self._stripes[hash(key) % _LOCK_STRIPES]   # store.py:71
```

用在 `get_or_compute`（`store.py:94`）的**双重检查**模式里：

```python
cached = self.get(key)          # 1. 无锁快路径
if cached is not None: return cached
with self._key_lock(key):       # 2. 锁住该 key 所属条带
    cached = self.get(key)      # 3. 再查一次（可能别的线程已算完）
    if cached is not None: return cached
    value = compute()           # 4. 同一 key 只有一个线程在冷算
    self.set(key, value)
```

效果：**同一个 key 并发请求只触发一次 `compute()`**，其余线程等它算完直接吃缓存；不同 key（落在不同条带）完全并行。

## 为什么用条带池，而不是 per-key 锁字典

`store.py:64-68` 的注释把动机写得很清楚 —— 它替换掉了原来的"每 key 一把锁的字典"，原因是那个字典有**慢性内存泄漏**：锁在三条路径下不会被回收 ——

1. TTL 过期
2. `compute()` 抛异常
3. 结果不可缓存（见 `get_or_compute_cacheable`，`store.py:106`）

条带池锁数恒为 2048、与唯一 key 数解耦，所以 `set()` 做 LRU 淘汰时不用回收锁（`store.py:92` 那句注释就是这个意思），泄漏问题根除。代价就是我前面说的**假争用**，2048 槽下概率极低，且不破坏 singleflight 语义。

## 注意：两套锁，职责分明

- `self._lock`（`store.py:63`）：全局数据锁，只护 `OrderedDict` 的读写/淘汰，持有时间极短。
- `self._stripes`：只护 compute 阶段。

加锁顺序始终是 stripe → data（持条带锁时才会短暂取数据锁，反过来不会），所以不存在死锁。

另外文件下半部（`store.py:140` 起）还有一套基于 Redis `SET NX EX` + token + Lua 释放的版本 —— 那是**跨进程的分布式锁**，是同一个 singleflight 思想在多实例下的对应实现，跟进程内这套条带锁是两层。合并 churn-engine 后如果多实例共用缓存，真正生效的是 Redis 那层。

---

### Turn 3  *(2026-06-09 09:48)*

**User:**

Redis 那层分布式锁怎么实现的

**Assistant:**

是单实例 Redis 锁的经典写法：**`SET NX EX` + 唯一 token + Lua 比对删除**，外面包一层跨进程 singleflight。拆开看三块：

## 1. 拿锁 `_acquire`（`store.py:158`）

```python
self._r.set(lock_key, token, nx=True, ex=self._lock_ttl)
```

一条命令同时拿到互斥和防死锁：
- `nx=True` —— key 不存在才设，保证只有一个进程能拿到（互斥）。
- `ex=lock_ttl` —— 锁带 TTL 自动过期，**持锁者崩了也不会把全集群卡死**。
- `token = uuid4().hex`（`store.py:180`）—— 每次尝试一个唯一令牌，标识"这把锁是我的"。
- 返回三态：`True` 拿到 / `False` 别人持有 / `None` Redis 挂了 → 调用方退化为直接算。

## 2. 放锁 `_release` + Lua（`store.py:166` / `:123`）

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end
```

**这段 Lua 是整个锁正确性的核心**，防的是这个竞态：

```
A 拿锁，但 compute() 跑得比 lock_ttl 久 → 锁自动过期
B 拿到同一把锁（已空），开始算
A 算完调 release —— 如果无脑 DEL，就把 B 的锁删了！
→ C 又能拿锁 → 同一 key 两个进程同时冷算，singleflight 失效
```

`get == token` 的检查让 A 只会删自己的锁，过期后被 B 重拿的锁删不掉。而且**检查和删除必须原子** —— 若在 Python 里拆成 GET 再 DEL，两步之间锁可能过期被人重拿，照样误删；Lua 在 Redis 单线程里原子执行，杜绝这个缝。

## 3. singleflight 主循环（`store.py:172`）

```
get(key) 命中 → 直接返回                    （快路径）
loop:
  acquire:
    None → run()                            （Redis 挂 → 自算）
    True → 双检 get(key)，没有就 run()，finally 释放
    False（别人在算）:
        get(key) 命中 → 返回
        超过 deadline → run()               （等太久 → 自算兜底）
        否则 sleep(poll) 再试               （轮询等结果落缓存）
```

设计哲学是 **best-effort、绝不无限阻塞**：通往"直接算"的兜底有三条 —— Redis 异常、等待超时、锁不可用。它把**活性看得比"严格只算一次"更重**：失败时退化成"可能多算几次"（对缓存只是浪费，不是错误），而不是把请求挂死。配合 `socket_timeout=2`（`store.py:139`），Redis 再慢也拖不垮请求。

## 几个要知道的边界

- **这不是 Redlock**，是单实例锁。Redis 主从切换时若 SET 还没复制到从库，新主上锁会"丢"，理论上能出现两个持锁者。但对缓存 singleflight 来说双算只是浪费，所以代码主动接受了这个弱保证，没上 Redlock 那套复杂度 —— 这个取舍是对的。
- **等待靠轮询**（poll 间隔，非 pub/sub 通知），实现简单、延迟略高，但有 `lock_wait` 上限兜底。
- **持锁者没写缓存时**（如降级结果不缓存），等待者读不到，释放后会各自串行 run —— docstring（`store.py:175`）明确写了这是已知行为，不是 bug。

## 和 churn 合并那件事的关联

你 memory 里记的"共用 Redis 真正的坑是淘汰策略"在这层有个**额外、独立的**影响：这些 `sf:` 锁 key 也住在同一实例里，如果共享实例配了 `allkeys-lru`/`allkeys-random` 之类淘汰策略，Redis 可能**在 TTL 到期前就把 `sf:` 锁 key 淘汰掉** —— 等于锁提前消失，另一个进程立刻能拿锁，互斥被悄悄削弱。这和"缓存值需要 noeviction"是两件事：缓存值被淘汰只是回退到重算（可接受），而锁被淘汰会破坏 singleflight 语义。所以合并后给这实例上 `noeviction`（或锁/缓存分实例）时，记得这条也是理由之一。

---
