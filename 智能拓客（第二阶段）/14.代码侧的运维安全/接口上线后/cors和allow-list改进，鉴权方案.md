# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `ad21a304-dd68-475d-b54e-72312ccb8362`
- **范围**: Turn 1-9
- **导出时间**: 2026-06-23 10:15:35

---
### Turn 2  *(2026-06-23 00:22)*

**User:**

给出cors修改方案和理由，
CORS_ORIGINS / ENABLE_API_LOGGING在服务器上的实际值如下：
[# CORS_ORIGINS=http://localhost:3000,https://your-frontend.com]
ENABLE_API_LOGGING=1；
鉴权方面，我是把接口给前端的，用在企业微信的小程序中，会先让用户登录企微，才能使用智能拓客功能，这种情况需要鉴权吗？

**Assistant:**

三件事一起回答。先说你给的服务器值里一个**要紧的发现**:

> `# CORS_ORIGINS=http://localhost:3000,https://your-frontend.com` —— 这行是 **`#` 注释掉的**,等于服务器上 `CORS_ORIGINS` **没设置**。结合 `main.py` 的 `os.getenv("CORS_ORIGINS", "*")`,**生产此刻正以 `allow_origins=["*"] + allow_credentials=True` 在跑**——就是那个"反射任意来源 + 带凭证"的最宽松配置。下面的修改正是要消掉它。

---

## 一、CORS 修改方案

**改 `app/main.py:156-162`**,把"无条件挂、默认 `*`"换成"留空不挂、非空只认白名单":

```python
# 改之前
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "*").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 改之后
# CORS：仅在显式配置 CORS_ORIGINS 时才挂，且只认白名单、绝不回退到 *。
# 生产经 nginx 同源反代 / 企微小程序 wx.request（无浏览器 Origin）本就不需要 CORS；
# 留空=不挂，对同源与非浏览器客户端零影响。仅前端跨域联调 / H5 webview 才配域名。
_cors_origins = [o.strip() for o in lead_settings.cors_origins.split(",") if o.strip()]
if _cors_origins:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=_cors_origins,
        # * 与 credentials 不可共存（会退化成"反射任意来源+带凭证"）；显式列域名才开凭证
        allow_credentials="*" not in _cors_origins,
        allow_methods=["*"],
        allow_headers=["*"],
    )
```

同步改 `.env.example:46`:把"留空默认 *"改成"**留空=不挂 CORS(同源/小程序无需);仅跨域联调或 H5 webview 才填域名**"。

**服务器侧**:`CORS_ORIGINS` 保持注释/不设即可——改完代码后,不设=不挂中间件,`*`+credentials 的面直接消失。

### 四条理由
1. **消掉当前真实危险面**:服务器现在就是 `*`+credentials 在跑,改后默认不挂,这个面没了。
2. **落实代码自己的意图、消孤儿配置**:`settings.py:114-116` 早写了"留空=不挂 CORS",但 `main.py` 从没用过 `lead_settings.cors_origins`,反而自己读 env、默认还反成 `*`。改后 main.py 真正接上 settings,行为和注释一致。
3. **防 `*` 退化**:`allow_credentials="*" not in _cors_origins` 保证哪怕有人手滑写 `CORS_ORIGINS=*`,也自动关掉凭证,不会再反射任意来源。
4. **对企微小程序零影响**:原生 `wx.request` 不走浏览器 CORS,同源 nginx 反代也不触发预检——留空不挂正合适。

> ⚠️ 重申定位:CORS 是"挡别的网站借用户浏览器跨域读你的响应",**不是鉴权**。它挡不住 `curl` 直接刷接口枚举 PII。所以这步是"补漏",真正上锁靠下面第三部分。

---

## 二、`ENABLE_API_LOGGING=1` 的含义(顺带)

确认日志**在生产开着**——意味着业务员的**实时 GPS(lng/lat)正持续落进 `lead_api_log`**(异地的 amer_sales_assist 库)。这不是对外暴露,但是员工位置数据在库里堆积。所以之前说的"api_log 改 allowlist + 决定 GPS 去留"现在是**真问题不是假设**,建议和 CORS 一起做掉(都是小改、有测试缝)。

---

## 三、鉴权:需要吗?——**需要。而且你手里正好有最好的身份源,只是后端还没接**

直接回答你的纠结点:**"用户先登录企微才能用"这件事,本身保护不了你的接口。** 关键在于区分两件容易混的事:

- **客户端登录态**:用户在企微里登录、打开小程序——这只在**前端**把 UI 门关上。
- **服务端身份校验**:你的 `/api/v1/leads/*` 有没有**验证**这个请求确实来自一个已登录的企微用户。

**现在是前者有、后者没有。** 后果很具体:
- 接口公网可达,`employee_code` 只是 body 里一个**可任意伪造**的字段(连你自己的数据归属都在信这个假字段)。
- 任何人拿 `curl` 直接打接口,**完全绕过小程序**,枚举 `customer_id` 就能拉走客户 PII。"必须先登录企微"对 `curl` 毫无约束。

所以:**是的,需要后端鉴权。** 但好消息是——**企微就是现成的、可信的身份源,你不用自建登录系统,只要让后端去"验"一下这个企微身份。**

### 推荐做法:企微登录态服务端校验(小程序场景的标准姿势)
1. 小程序调 `wx.qy.login()` 拿临时 `code`,发给后端。
2. 后端用 `code` 调企微 `jscode2session`(小程序登录态校验)换出**验证过的 `userid`**(access_token 经 `gettoken` 拿、缓存 ~7200s,Redis 现成可放)。
3. `userid` ↔ `employee_code` 映射后,后端签发**短时效 token**(JWT 或不透明串)给小程序。
4. 之后每次 lead 请求带 `Authorization: Bearer <token>`;后端校验 token,**`employee_code` 从 token 里取**,不再信 body 那个。

这样一次性堵住:无 token 的 `curl` 被拒;`employee_code` 无法伪造;限流/审计能落到真人头上。

### 为什么这是该走的路、而不是过度设计
- 你**已经强制企微登录**,身份是可信且现成的——服务端验它是一段标准集成(企微提供 API),远比从零搭登录轻。
- 不做的话,"登录企微"对 API 层纯粹是装饰,公网可绕过。
- 之前提的"共享 API-Key 塞前端"在小程序里比 H5 稍好(包不是明文网页),但仍可被解包/抓包提取,且**不给每用户身份**——所以仍是次选,能上企微校验就别用它。

**主要成本**:企微 `access_token` 的获取与缓存 + `userid↔employee_code` 映射 + 一处 token 签发/校验的中间件(可只作用于 `LEAD_PREFIX`,和现有 `_LeadScopedBodyLimit` 一个套路)。约一个小迭代。

---

## 建议落地顺序
1. **现在就做**(我可以直接改、单独 commit 带测试):CORS 那段 + api_log allowlist(顺带定 GPS 去留)。
2. **下一个迭代**:企微登录态服务端校验 + Bearer token——这才是真正给客户 PII 上锁的那步。

要我现在先把**第 1 步的 CORS + allowlist** 落地吗?第 2 步如果你点头,我可以先出一份接口/中间件/配置项的具体实现方案再动手。

---

### Turn 3  *(2026-06-23 00:37)*

**User:**

1.cors有什么作用？我按你的方案改了之后，lead接口前端还能正常调用吗？
2."api_log 改 allowlist + 决定 GPS 去留"，这个问题的修改方案是什么？
3.鉴权方面，我也拿不到employee_code啊

**Assistant:**

三个问题逐个答。第 2 个要给具体代码,我先看一眼现有测试,免得方案漏掉"要同步改的测试"。

全部查清了,逐个答。

## 1. CORS 有什么作用?改完前端还能调吗?

### CORS 到底管什么
CORS 是**浏览器**的机制,不是服务器的门禁。它只在这个场景生效:**一个网页(源 A,比如 `evil.com`)用 JS(`fetch`/`XHR`)去请求另一个源 B(你的 API)**。浏览器会拿 B 返回的 `Access-Control-Allow-Origin` 头判断"A 能不能读 B 的响应",不允许就**把响应拦在 JS 之外**。

三个要点:
- **它保护的是"你的用户",不是"你的服务器"**:作用是阻止*别的恶意网站*借用户的浏览器偷读你的接口数据。
- **请求照样到服务器、照样执行**——CORS 只决定"浏览器要不要把结果交给那段 JS"。所以它**挡不住 `curl`/脚本**(它们根本不看 CORS),这也是为什么我反复说 CORS ≠ 鉴权。
- **服务端永远不会因为 CORS 而拒绝请求**。最坏情况只是"跨域网页的 JS 读不到响应",绝不会 500/403。

### 改完你的前端会不会受影响?——**不会**(关键看调用方式)
你是**企业微信小程序**。原生小程序用 `wx.request` 发请求:

- **不带浏览器 Origin、不走 CORS、没有预检**。小程序的网络安全是另一套——你在企微管理后台配的「request 合法域名」白名单(在腾讯那边),跟我们这个 CORS 头**完全无关**。
- 所以**把 CORS 中间件改成"留空不挂"后,`wx.request` 调你的接口照常成功,零影响**。

唯一需要留意的例外:如果小程序里嵌了 **`web-view` 加载 H5 页面**,且那个 H5 是**跨域**用 `fetch` 调 API。这种才吃 CORS。但你生产是「nginx 同源反代」,H5 大概率和 API 同源 → 同样不需要 CORS。

**零风险落地建议**:如果你不确定有没有 H5、嫌"留空"心里没底,就别留空——直接把 `CORS_ORIGINS` 设成你前端的真实域名(原生小程序的话其实设不设都行)。**只要不是 `*`,前端都能正常调**,而且比现在的 `*`+credentials 安全得多。

---

## 2. api_log 改 allowlist + GPS 去留:具体方案

### 现状的问题(已确认 `ENABLE_API_LOGGING=1`,所以是真问题)
日志在生产开着,当前是 **denylist**——只剥 `score_context`,**其余请求体原样落库**。意味着业务员**实时 GPS(lng/lat)正持续写进 `lead_api_log`**,而且明天谁加个敏感字段会**静默落库**(fail-open)。

### 改法:`app/lead/middleware/api_log.py`
把"剥黑名单"换成"只留白名单"(fail-closed),**GPS 默认不记**:

```python
# 改之前
_DROP_KEYS = ("score_context",)

# 改之后：只把"非敏感、对分析有用"的 key 写进 request_body；其余（含未来新增/意外混入）一律丢。
# fail-closed：以后加新字段默认不落库，要显式加进白名单。
# 刻意排除：score_context（庞大冗余）、lng/lat（业务员实时 GPS=位置 PII，分析用不上）。
_BODY_ALLOW_KEYS = frozenset({
    "employee_code", "customer_id",
    "radius_km", "page", "page_size",
    "industry_ids", "industry_names",
})
```

```python
# trim_request_body 主体改成白名单过滤：
def trim_request_body(raw: bytes, max_bytes: int | None = None) -> str | None:
    if max_bytes is None:
        max_bytes = settings.api_log_body_max_bytes
    if not raw:
        return None
    try:
        parsed = json.loads(raw)
    except Exception:  # noqa: BLE001
        return None
    if not isinstance(parsed, dict):   # 非 JSON 对象一律不记（fail-closed）
        return None
    try:
        slim = {k: v for k, v in parsed.items() if k in _BODY_ALLOW_KEYS}
        text = json.dumps(slim, ensure_ascii=False)
        if len(text.encode("utf-8")) > max_bytes:
            return json.dumps({"_truncated": True, "_bytes": len(raw)}, ensure_ascii=False)
        return text
    except Exception:  # noqa: BLE001
        return None
```

### GPS 去留:两个选项
- **默认(推荐)= 丢掉**:`lng/lat` 不在白名单里 → 不落库。日志的用途是"调用次数/效果/耗时",精确 GPS 用不上,丢了最干净。
- **想留"大致区域"做覆盖分析 = 降精度**:把 `"lng","lat"` 加进白名单,并在 `slim` 后降到 2 位小数(~1km,变成"哪片区域"而非"精确点位"):
  ```python
  for k in ("lng", "lat"):
      if isinstance(slim.get(k), (int, float)) and not isinstance(slim.get(k), bool):
          slim[k] = round(slim[k], 2)
  ```

我建议先按**默认丢掉**走,要分析再升级到降精度。

### 要同步改的测试(有现成的 tripwire)
- `tests/lead/test_api_log_privacy.py::test_non_drop_keys_currently_retained` —— 这条**专门固化旧行为**(断言混入的 `token` 被保留),改 allowlist 后它会失败,这正是它设计的目的。**重写**成断言相反:
  ```python
  def test_non_allowlisted_keys_dropped(client, monkeypatch):
      """allowlist：请求体里非白名单字段（含意外混入的 token/password）一律不落库（fail-closed）。"""
      recorded = _capture(monkeypatch)
      monkeypatch.setattr("app.lead.repositories.lead_skip_repo.upsert_skip", lambda emp, cid: None)
      client.post(SKIP, json={"employee_code": "000068", "customer_id": "198", "token": "secret-xyz"})
      logged = json.loads(recorded[0]["request_body"])
      assert "token" not in logged          # 非白名单字段被丢
      assert logged["customer_id"] == "198"  # 白名单字段保留
  ```
  顺手更新文件 docstring 第 ③ 条。可选:在 `test_employee_code_truncated_to_16_in_log` 里加一句 `assert "lng" not in logged` 把 GPS 决策也锁进测试。
- 其余测试(`test_score_context_stripped...`、`test_api_log_helpers.py` 的几条)断言的都是白名单内字段,**照常通过**。

---

## 3. "我也拿不到 employee_code 啊" —— 这恰恰是关键,但你**不需要** employee_code 就能堵住洞

你这句点破了一个我得讲清的区分。鉴权其实是**两件不同的事**,被你这个问题分开了:

| | 要解决什么 | 需要什么 | 紧急度 |
|---|---|---|---|
| **A. 访问控制** | 挡住"公网 `curl` 匿名刷接口枚举 PII" | 只需证明"调用者是个真·已登录企微用户" | 🔴 紧急(就是那个洞) |
| **B. 参数完整性** | 挡住"已登录用户冒用*别人的* employee_code 查询" | 需要 `userid → employee_code` 映射 | 🟡 可选(防内部越权) |

你纠结的"拿不到 employee_code",卡的是 **B**。但**洞是 A,而 A 根本不需要 employee_code**:

- 企业微信小程序登录(`wx.qy.login` → 后端 `jscode2session`)直接返回**验证过的 `userid`**——这个你**一定拿得到**,不需要任何内部映射。
- 用这个 `userid` 签发短时效 token、每次请求校验 → **匿名 `curl` 没有有效 token 直接被拒**,洞就堵上了。
- 此时 `employee_code` 继续当**业务参数**从 body 里传(它本来就是"以哪个业务员视角评分"的入参),**不参与鉴权**。你不用为了堵洞去解决 employee_code 的来源问题。

所以:**先做 A**(gate 在 `userid` 上),洞立刻关闭;**B 以后有 `userid↔employee_code` 映射了再加**,防的是"内鬼查同事的客户",优先级低。

### 我需要你确认一件事(决定 B 能不能做、以及 employee_code 该不该信)
**小程序现在传给后端的那个 `employee_code`,前端是从哪拿到的?** 比如:
- 是企微 `userid` 直接当工号 / 有张映射表?→ 那 B 可做,甚至能直接从 token 派生 employee_code,body 里那个就不用信了。
- 是用户自己填/选的?→ 那 employee_code 天生不可信,只能先做 A(gate userid),B 暂时无解。
- 是另一个 CRM 登录给的?→ 那真正的身份源可能是 CRM 不是企微,鉴权得验 CRM 那个态。

---

要不要我**现在就把第 1、2 项(CORS + api_log allowlist,默认丢 GPS)落地**——单独 commit、带测试改动、跑一遍 `pytest`?第 3 项的鉴权,等你回上面那个"employee_code 从哪来"的问题,我再出完整实现方案。

---

### Turn 4  *(2026-06-23 00:51)*

**User:**

鉴权方面，调用者肯定是已经登录的用户啊，用户必须先登录企业微信才使用

**Assistant:**

你说的"调用者肯定是已登录用户"——这正是最该掰开的地方。你没说错,但它只对了一半:**"必须先登录企微"是你给*正经用户*铺的那条路上的规矩,而攻击者根本不走你这条路。**

## 关键:登录是"小程序"在拦,不是"你的接口"在拦

你的接口 `https://aisales.amer.cn/api/v1/leads/*` 是公网可达的。攻击者**不会打开你的小程序**,他直接对着这个地址发 HTTP:

```bash
# 攻击者在自己电脑上，从没碰过企业微信：
curl -X POST https://aisales.amer.cn/api/v1/leads/timeline \
  -H 'Content-Type: application/json' \
  -d '{"customer_id": "1"}'
# 再换 "2"、"3"、"4"…… 把每个客户的时间轴全捞走
```

这个请求**到你服务器、照样执行、照样返回真实客户数据**。因为你的服务器**什么都没检查**——它只是读 body 里的 JSON 然后回数据。"必须先登录企微"这条规矩,是**小程序那个界面**在执行的,你的 **API 根本不知道企微的存在**。

> 打个比方:你店门口挂了"会员才能进",正经顾客从前门刷卡进来;但 API 是**敞开的后门,直通马路,没锁**。小偷不走前门,绕到后门就进来了。那块"会员专用"的牌子,对后门没有任何约束力。

## 为什么 body 里的 `employee_code` 不算"已登录"
你现在唯一能证明身份的东西,是 body 里那个 `employee_code` 字符串。但它就是一段**任何人都能手打**的文本——`curl` 里写 `"employee_code": "000068"` 和真用户发来的一模一样,服务器分不出来。**登录这件事确实发生了,但它产生的"我是谁"这个事实,从来没以服务器能验证的形式传到你 API**。所以对 API 来说,调用者不是"已登录用户",而是"**整个互联网上任何人**"。

(顺带堵掉一个常见念头:"别人不知道我的接口地址"——不成立。域名在 DNS/TLS 证书透明日志里是公开的,小程序包能被解包、流量能被抓,路径一抓一个准;而 `customer_id` 从 1 顺着枚举,连猜都不用猜。靠"找不到"不是防护。)

## 真正的修法:让"登录"产生一个服务器能验的凭证
原则就一句:**客户端不能被信任来做访问控制,必须服务端验。** 具体落到你这套(而且你手里正好有企微这个现成身份源):

1. 小程序登录后 `wx.qy.login()` 拿 `code` → 后端 `jscode2session` 换出**验证过的 `userid`**;
2. 后端签发一个**短时效 token** 发回小程序;
3. 之后每次请求带 `Authorization: Bearer <token>`,**后端校验 token 才放行**。

改完之后,上面那条匿名 `curl` 因为**没有有效 token,直接被拒**——洞就关上了。而这一步**不需要你解决 employee_code 的来源问题**(上轮说的:gate 在 `userid` 上,employee_code 继续当业务参数传)。所以你不卡在那。

## 你可以亲手验一下(最直接)
从**任意一台机器、完全不碰企业微信**,跑一条上面的 `curl` 打你生产域名——你会拿到 **200 + 真实数据**,而不是 401/403。那一刻就能确认:现在确实是"谁来都给"。这是你自己的系统、防御性自查,放心试(跑一条就够,别循环枚举)。

---

所以回到决策:**鉴权是需要的**,"用户登录企微"挡不住它要挡的东西。

我的建议不变:
- **现在就落地** CORS + api_log allowlist(这俩和鉴权独立,先把能堵的小口堵上);
- **鉴权**是下一步的正经活,我可以出一份具体实现方案(token 中间件只作用于 `LEAD_PREFIX`、jscode2session 对接、access_token 缓存放 Redis)。

要我先把 CORS + allowlist 改了,然后给你写鉴权的实现方案吗?

---

### Turn 5  *(2026-06-23 00:54)*

**User:**

要

**Assistant:**

开始落地。先把要改的测试和 conftest 读全,避免遗漏断言。

关键发现:`test_api_log_helpers.py:107` 直接断言 `lng` 在落库 body 里——这条会随 GPS 默认丢弃而失败,必须一并翻转。其余 `lng/lat` 引用都是请求校验/响应断言,不受影响。开始改。

**1) CORS — `app/main.py`**

**2) `.env.example` — CORS 说明**

**3) api_log allowlist — `app/lead/middleware/api_log.py`** (默认丢 GPS)

**4) 翻转 `test_api_log_helpers.py` 的 GPS 断言**

**5) `test_api_log_privacy.py` — 改 docstring + 重写 tripwire 测试**

全部改完。跑格式化 + lint + 全量测试 + 类型检查。

Lint 全过。跑全量测试 + mypy(并行)。

全绿:**ruff format/check 通过、527 passed、mypy no issues**。现在按约定单独成 commit。当前在默认分支 `main`,先开分支再提交。

暂存了我改的 5 个文件;`tests/conftest.py`(会话开始前就存在的未跟踪文件)**未纳入**本提交。现在提交。

已落地并提交。下面给出**鉴权实现方案**(你要的第三部分)。

## ✅ 第 1、2 项已完成

分支 `lead-cors-apilog-hardening`,commit `6070029`,5 文件:
- **CORS**:`main.py` 改为"仅配置 `CORS_ORIGINS` 才挂、只认白名单、含 `*` 自动关 credentials";`.env.example` 同步;消掉孤儿配置。
- **api_log**:`trim_request_body` 改 allowlist(fail-closed),**GPS 默认不落库**;测试翻转。
- 验证:`ruff` 全过 · **`pytest 527 passed`** · `mypy 81 files 全绿`。

> 两个收尾提醒:① 生产 `CORS_ORIGINS` 保持留空即可,部署后 `*`+credentials 暴露面消失;② `tests/conftest.py`(会话前就有的未跟踪文件)**未纳入**本提交,你自己决定何时提。需要我把分支合回 `main` 再说。

---

# 鉴权实现方案(企微小程序登录态服务端校验)

**目标**:让 4 个 PII 接口只接受"带有效凭证"的请求,凭证由后端用**企微验证过的 `userid`**签发。匿名 `curl` 无凭证直接 401,洞关闭。

## 配置(全部带默认值,secret 留空,符合铁律#3)

| 变量 | 默认 | 用途 |
|---|---|---|
| `WECOM_AUTH_ENABLED` | `0` | **kill-switch**:0=放行(灰度/回滚),1=强制校验 |
| `WECOM_CORP_ID` | `""` | 企业 corpid |
| `WECOM_APP_SECRET` | `""` | 小程序应用 secret(换 access_token) |
| `WECOM_API_BASE` | `https://qyapi.weixin.qq.com` | 可覆盖;`trust_env` off(cn↔cn 直连,仿 crm_client) |
| `LEAD_SESSION_SECRET` | `""` | 签发自有 token 的 HMAC 密钥 |
| `LEAD_SESSION_TTL_SECONDS` | `7200` | token 有效期 |

## 分阶段落地(对应项目分层:clients / middleware / routers + 薄装配)

**Phase 0 — 配置 + .env.example**:加上表 6 个变量,`WECOM_AUTH_ENABLED` 默认 `0`。**关键:默认关,部署代码不破坏现有前端**。

**Phase 1 — 企微 client**(`app/lead/clients/wecom.py`,仿 `crm_client` 风格):
- `get_access_token()` → `/cgi-bin/gettoken`,**缓存进 Redis db2**(TTL=expires_in−300),复用 `cache.py` 现成的 singleflight 锁防击穿(access_token 有频控,全集群共用一份)。
- `jscode2session(code)` → `/cgi-bin/miniprogram/jscode2session` → 返回**验证过的 `userid`**。
- httpx + 超时/重试,`trust_env=False`。

**Phase 2 — token 签发/校验**(`app/lead/services/session_token.py`,纯函数、易单测):
- 用 **stateless HMAC**:`token = base64(payload{userid, exp}) + "." + HMAC(payload, LEAD_SESSION_SECRET)`。
- `issue(userid) -> token` / `verify(token) -> userid | None`(校验签名 + 未过期)。
- 选 stateless 的理由:校验纯 CPU、无热路径 Redis 依赖、重启不失效;短 TTL 抵消"不可吊销"。需要吊销再换 Redis 会话表。

**Phase 3 — 登录端点** `POST /api/v1/leads/session`(新 `routers/auth.py`):
- 入参 `{code}`(来自 `wx.qy.login()`)→ `jscode2session` → `userid` →(可选映射 employee_code)→ `issue()` → `envelope({token, expires_in})`。
- 此端点本身不需鉴权,但**必须有合法的单次性企微 `code`**才能换到 token,无法被刷;`code` 因 allowlist 也不会进 api_log。

**Phase 4 — 鉴权门**(FastAPI `Depends(require_session)`,挂在 4 个 PII 路由上):
- 读 `Authorization: Bearer <token>` → `verify()`;失败→ `JSONResponse(200, envelope(code=1401, "登录态失效，请重新进入"))`(沿用 lead 200-envelope 契约)。
- `WECOM_AUTH_ENABLED=0` 时依赖直接放行(灰度/回滚开关)。
- 把 `userid` 注入 request.state,供下游 + 后续 api_log 落库用。
- 选 dependency 而非 ASGI 中间件:只 4-5 个路由、显式、好单测;`/session`、`/healthz`、`/industries`(公开行业字典,无 PII)不挂。

**Phase 5 — 测试**(全 stub,保持"确定性、无外部依赖"):
- stub `wecom.jscode2session` / `get_access_token`,不发真 HTTP;
- token 纯函数:有效/过期/篡改/缺失;
- 依赖门:开关 off→放行、on+无 token→1401、on+有效 token→通过;
- `/session` 端点用桩 `jscode2session` 验签发。

**Phase 6 — 灰度上线**(这套不停机加鉴权的标准姿势,贴合你 ~50 人灰度):
1. 先部署后端,`WECOM_AUTH_ENABLED=0` → 什么都不变;
2. 前端发版:`wx.qy.login()` → `POST /session` 存 token → 所有 lead 请求加 `Authorization`;遇 401 重新 login 刷 token 重试;
3. 前端全量后,翻 `WECOM_AUTH_ENABLED=1` → 匿名调用即被拒,洞关闭。

## 还需你拍一件事:`userid ↔ employee_code` 有映射吗?

这决定能不能做"参数完整性"(防越权),但**不影响关洞**:
- **有映射** → `/session` 里把 employee_code 也写进 token,下游从 token 取(可信),并校验 body 的 employee_code 一致、不符则拒。
- **没映射** → token 只带 `userid`,照样关洞(访问控制);body 的 employee_code 暂当业务参数,以后补映射再加这层。

**可选加固**(和上面独立):host nginx 按 IP/userid 限流,进一步压枚举速率。

---

要我**按这个方案开始实现**吗(同样单独分支、全 stub 测试、跑 `pytest`+`mypy`)?顺带告诉我 `userid↔employee_code` 那件事,我好决定 token 里带不带 employee_code。另外,刚才那个 hardening 分支要不要我**合回 `main`**?

---

### Turn 6  *(2026-06-23 01:39)*

**User:**

H5 大概率和 API 同源，这是什么意思

**Assistant:**

"同源"(same-origin)是浏览器判断"两个 URL 算不算一家"的规则,直接决定**要不要走 CORS**。

## 同源 = 协议 + 域名 + 端口,三者全相同

只要这三样有**任意一个**不同,就算"跨域"。以 H5 页面 `https://aisales.amer.cn/index.html` 为基准:

| 它要调的接口 URL | 协议 | 域名 | 端口 | 结果 |
|---|---|---|---|---|
| `https://aisales.amer.cn/api/v1/leads/...` | ✓ https | ✓ 同 | ✓ 443 | ✅ **同源** |
| `http://aisales.amer.cn/api/...` | ✗ http | ✓ | (80) | ❌ 协议不同 |
| `https://api.amer.cn/...` | ✓ | ✗ 子域不同 | ✓ | ❌ 域名不同 |
| `https://aisales.amer.cn:8000/...` | ✓ | ✓ | ✗ 8000 | ❌ 端口不同 |

(注意:`aisales.amer.cn` 和 `api.amer.cn` 子域名不一样,也算跨域。)

## 为什么这跟 CORS 直接相关
浏览器的同源策略:
- **同源请求** → 浏览器直接放行,**根本不发 CORS 预检、不看 `Access-Control-Allow-Origin` 头**。CORS 中间件挂不挂都无所谓。
- **跨域请求** → 浏览器才会触发 CORS 那套检查,这时后端必须用 `CORS_ORIGINS` 把对方域名列进白名单,否则浏览器拦掉响应。

## "H5 大概率和 API 同源"具体指什么
你生产是「**nginx 同源反代**」:宿主机 nginx 用**一个域名** `aisales.amer.cn` 按路径分流——
- `/` 这类路径 → 返回 H5 静态页面
- `/api/v1/leads/...` → 反代到后端 api 容器

于是浏览器看到的**页面的源**和**接口的源**都是 `https://aisales.amer.cn`(同协议、同域名、同 443 端口)→ **同源** → 不走 CORS。所以即便你的前端真有 H5,把 CORS 中间件改成"留空不挂",它照样能调接口。

说"大概率"是因为这是从你那句"同源反代"**推断**的——我看不到你实际部署,留了余地。

**你自己一秒确认**:打开那个 H5,看地址栏域名,再开 F12 → Network 看它请求接口的域名。**两个是同一个 `https://aisales.amer.cn` → 同源 → 改 CORS 不影响。**

## 回扣一句
如果你前端是**原生企微小程序(`wx.request`)**,那压根没有"网页源"这个东西,同源/跨域都不适用,CORS 本来就不参与——**无论同不同源,改 CORS 对小程序都零影响**。"同源"这套只在"小程序里用 webview 嵌了 H5 页面"时才需要考虑。

---

### Turn 7  *(2026-06-23 01:50)*

**User:**

改 CORS 对小程序都零影响，这是为什么

**Assistant:**

一句话:**CORS 是浏览器的机制,而小程序不是浏览器,根本不跑这套**。我把这个机制拆开你就明白了。

## CORS 到底由谁执行
关键认知:**CORS 是浏览器在客户端执行的,不是服务器执行的**。服务器只是吐几个响应头(`Access-Control-Allow-Origin` 等),这些头本质是**"给浏览器看的指令"**。真正干活的是浏览器里的那段逻辑:

> "这个网页(源 A)想读源 B 的响应 —— 我(浏览器)去看看 B 的响应头允不允许 A 读;不允许,我就**把响应扣下、不交给这个网页的 JS**。"

注意:请求照样发出去、服务器照样返回,CORS 只决定**浏览器要不要把结果递给那段网页 JS**。

## 为什么小程序不吃这套
企微/微信小程序**不是浏览器里的网页**,它跑在微信的自有运行时里,网络请求走 `wx.request()` —— 这是运行时自己的 HTTP 客户端,**没有实现浏览器的同源策略**:
- 它不带网页的 `Origin`、不发 CORS 预检、**根本不看响应里的 `Access-Control-Allow-Origin` 头**。
- 它的行为更像 `curl` 或后端 HTTP 客户端:发请求 → 拿响应 → 把 body 交给你,**全程无视 CORS 头**。

所以你服务器**挂不挂、怎么改 CORS 中间件,对 `wx.request` 一点影响都没有** —— 那些头它压根不读。

> 打个比方:CORS 是**浏览器这个保安**才执行的规矩。`curl` 和 `wx.request` 身上没有这个保安,直接进门拿东西。你改保安的规矩,对"不走保安通道的人"自然零影响。

## 那小程序靠什么限制?(别和 CORS 搞混)
小程序有它**自己的一套**域名限制:你得在**微信/企微管理后台**配「服务器域名 / request 合法域名」白名单,由**微信运行时**(在腾讯那边)校验 `wx.request` 能不能打这个域名。

它和 CORS 是**两套完全独立**的东西:

| | CORS | 小程序合法域名 |
|---|---|---|
| 谁执行 | 浏览器 | 微信运行时 |
| 在哪配 | **你的服务器**(响应头) | **腾讯 mp 后台** |
| 我这次改动碰它吗 | 碰(改的就是这个) | **完全没碰** |

所以:如果哪天小程序调不通接口,要去 **mp 后台的「合法域名」**里找,跟你服务器的 CORS 配置无关 —— 而我改的恰恰只是后者。

## 结论
改 CORS = 改一份**只给浏览器看的指令**;小程序不是浏览器、不读这份指令,所以零影响。**唯一例外**还是那个:如果你在小程序里用 `web-view` 嵌了 **H5 网页**,那个 H5 是真·浏览器环境,会吃 CORS —— 这种情况才需要把 H5 的域名配进白名单。纯原生 `wx.request` 不受影响。

---

### Turn 8  *(2026-06-23 02:07)*

**User:**

合回 main 吧

**Assistant:**

合并完成。

- **main 已快进到 `6070029`**(紧接 v2.4.1),特性分支已删除,无冲突。
- **未 push**——按惯例只在你明确要时推;要我推到远端随时说。
- `tests/conftest.py` 仍未跟踪(会话前就在,非本次改动),你自己决定何时提。

剩两件挂起的事:
1. **上线提醒**:生产 `CORS_ORIGINS` 保持留空即可,部署后 `*`+credentials 暴露面就消失。(`ENABLE_API_LOGGING=1` 不用动,allowlist 已让 GPS 停止落库。)
2. **鉴权**:实现方案已给(企微登录态服务端校验,分 6 阶段、kill-switch 灰度)。我在等你两件事再开工——① `userid↔employee_code` 有没有映射(决定 token 带不带 employee_code);② 是否现在动手。

要不要我现在就开鉴权那块?

---

### Turn 9  *(2026-06-23 02:09)*

**User:**

鉴权实现方案，有个问题啊，我只是提供接口给前端而已，新配置的内容我是不知道的

**Assistant:**

这是个很实在的边界问题,而且它指向一个我之前默认了、但不该替你定的事:**企微那步换取,到底该你做还是前端做。** 先拆"不知道 config"这件事。

## "不知道新配置"分两层,别混
- **写代码**(读 `os.getenv("WECOM_...")`)——这是我干的,跟你知不知道具体值无关。
- **填值**(corpid/secret 到底是多少)——这是**部署期**的事,由**持有企微的人**塞进服务器 `.env`,跟你现在 `DB_PWD`、`DEEPSEEK_API_KEY`、`CRM_PASSPORT_SALT` 一模一样:你也不"知道"也不"发明"这些值,是运维/归属方给的。

所以"值不知道"本身不卡开发。但你这句真正点出的是另一件事——

## 企微 `code → token` 这步换取,谁来做?这决定你需不需要那些 config

| | **A. 你来换取** | **B. 前端侧换取,你只验** |
|---|---|---|
| 谁持有 corpid/secret | 你(部署期填进 `.env`) | 前端/他们的后端,**你不碰** |
| 你要写的代码 | 企微 client + `/session` + 签发 + 验签 + 门 | **只有验签 + 门** |
| 你需要的新配置 | corpid、app_secret、签发密钥… | **就一个验签密钥**(或他们的公钥) |
| 贴合"我只提供接口" | 一般 | ✅ 很贴合 |

**B 才是匹配"我只提供接口而已"的那条路**:
- 谁持有企微 secret,谁就负责登录 + 签发 token(他们本来就握着 corpid/secret);
- 你这边**完全不碰企微**,只做两件事:① 校验请求带的 token 签名对不对、没过期;② 没有/无效就 401。
- 你需要从他们那儿拿的,**只有一样**:验签用的密钥——共享密钥,或者更干净的方式,他们用非对称密钥签、给你一个**公钥**(你只能验、伪造不了)。外加约定一下 token 里 userid/employee_code 放哪。

## 但有一点 B 绕不开:**门还是得你来守**
B 不是"前端全包、你什么都不做"。**服务端强制校验这一步必须在你这** —— 这正是整件事的核心(客户端挡不住 `curl`,只有后端验才算数)。B 省掉的是"企微对接 + 持有 secret",省不掉"验签 + 拒绝无 token 请求"。

还有个前提要确认:**签发方必须真的验过企微身份**再签 token。否则他随便给谁都发,又变回剧场。

## 我需要你确认一件事(决定走 A 还是 B)
**有没有"另一个掌握企微 corpid/secret 的后端/团队"(比如前端那边、或企业 IT 的企微后端)?还是你就是唯一的后端?**
- **有** → 走 **B**:他们登录签发,你只做验签+门,从他们要一个公钥/共享密钥即可,不碰任何企微 config。
- **没有、你是唯一后端** → 只能走 **A**:那两个企微值找企微管理员要,部署期进 `.env`(和你现有 secret 一样的待遇),开发上由我搞定。

你先告诉我是哪种,我再给你对应那条路的最小实现(B 的话,你这边改动会小很多)。

---
