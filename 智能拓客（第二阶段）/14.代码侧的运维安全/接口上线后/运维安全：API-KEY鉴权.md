# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `ad21a304-dd68-475d-b54e-72312ccb8362`
- **范围**: Turn 1
- **导出时间**: 2026-06-22 14:55:11

---

### Turn 1  *(2026-06-22 00:56)*

**User:**

评估是否需要为lead的接口配置API-Key 鉴权，配置生产 CORS 白名单，以及是否需要api_log 改 allowlist 脱敏。帮我看看是怎么一回事

**Assistant:**

我来调查这三个问题的现状。先并行摸清楚 lead 的路由装配、CORS/中间件、以及 api_log 的实现。

现在深入看 api_log 中间件、lead settings、路由层鉴权情况。

初步发现已经很清楚了，再补几个关键细节：4 个接口的请求体字段（判断 PII 风险）、api_log 落库的表结构、以及 .env.example 现状。

请求体字段已基本清晰。再看完整的 request.py（特别是 recommend-reason 的 score_context 内容）和 .env.example,确认脱敏面与配置现状。

看一下 .env.example 的"认证已退役"段落和 lead_api_log 表结构，确认历史背景。

已经掌握全貌。最后确认一下仓库内 nginx/compose 是否在网关层做了鉴权或 CORS（这会影响结论），再给评估。

调查完成,网关层(`nginx/default.conf`)和 compose 都没有鉴权或 CORS 处理——api 容器在 `127.0.0.1:8000` 由宿主机 nginx(aisales.amer.cn:443)反代,链路上没有任何一层做入站鉴权。下面是三项评估。

---

## 总览

| 项 | 现状 | 真实风险 | 建议 |
|---|---|---|---|
| **1. API-Key 鉴权** | 4 个 lead 接口**零鉴权**(路由层无任何 `Depends`/Header 校验);认证随 churn HTTP 面在 v2.3.0 一起退役 | 接口返回客户 PII + customer_id 可枚举,经 aisales.amer.cn 可达 → **真实暴露** | **建议做**,但形态取决于调用方 |
| **2. CORS 白名单** | `main.py` 无条件挂 CORS,默认 `*` + `allow_credentials=True`;且 `settings.cors_origins` 是孤儿配置(没人用) | 最宽松的 CORS 组合,且与代码自己的设计意图矛盾 | **建议立即修**(便宜、无歧义) |
| **3. api_log 脱敏** | denylist(只剥 `score_context` 一个字段),其余请求体原样落库;日志默认关 | 当前落库字段低敏(工号/客户id/GPS),但 denylist 对未来新字段是 fail-open | **建议做,低优先级**(加固) |

---

## 1. API-Key 鉴权:这是三项里唯一的真实暴露

**现状**:`app/lead/routers/leads.py` 和 `customer_detail.py` 的 4 个 handler 没有任何鉴权依赖,全 app 搜不到一行入站鉴权代码。`.env.example:44` 写明"认证:已退役(v2.3.0 摘除 churn HTTP 面后无消费方)"——但 churn 退役时,**lead 才是唯一剩下的 HTTP 面**,鉴权却一起被摘了,等于 lead 现在裸奔。

**为什么是真实风险**(不是纸上谈兵):
- `/score-and-info` 返回的 `LeadInfo` 里有 `contact`(主要联系人)、`phone`、`tel`、`address`(`response.py:31-34`)——**成片的客户联系人 PII**。
- `customer_id` 是顺序数字 id(`_coerce_digit_id` 只校验 `isdigit`),`/timeline`、`/recommend-reason` 只要给个 customer_id 就能拉行为时间轴 → **可枚举遍历**。
- `employee_code` 只是 body 里一个字段,任何人可伪造——它是"归属标记",不是身份凭证。
- 链路上唯一的屏障是网络:api 只绑 `127.0.0.1`,靠宿主机 nginx(aisales.amer.cn:443)反代。仓库里这层 nginx **不做鉴权**。所以"谁能打到 aisales.amer.cn,谁就能无鉴权读客户 PII"。

**关键纠结点——API-Key 是否有效,取决于调用方形态**:
- 若调用方是**服务端**(Dify workflow / 后端):key 放在服务端配置里,共享 API-Key **有效且推荐**。
- 若是**浏览器/H5 前端**(业务员在浏览器里带 GPS 用):key 在前端可被 view-source 看到,**共享 key ≈ 形同虚设**,真正的防线是网络隔离(内网/VPN)或每用户 token。
- 若是**原生 App**:介于两者之间,key 比浏览器安全些。

> compose 注释提到"保留 lead 前端现有上游",加上请求里带实时 GPS,**很可能是浏览器/H5 前端**——若如此,单纯加共享 API-Key 的收益有限,需要和"aisales.amer.cn 是否公网可达"一起判断。

---

## 2. 生产 CORS:建议立即修,这里有个明确的 bug

**现状**(`app/main.py:156-162`):
```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "*").split(","),  # 默认 ["*"]
    allow_credentials=True,
    allow_methods=["*"], allow_headers=["*"],
)
```
两个问题叠加:

1. **默认 `*` + `allow_credentials=True` 是最危险的 CORS 组合**。Starlette 在这个组合下,对带 Origin 的请求会**回显具体 Origin** 并带 `Access-Control-Allow-Credentials: true`,等价于"反射任意来源 + 允许凭证"——正是 CORS 指南明确警告的配置。当前虽无 cookie 鉴权、危害被削弱,但它仍让**任意第三方网页**能借访客浏览器(访客的网络位置)跨域读取接口返回的客户 PII。

2. **配置自相矛盾(孤儿配置)**:`settings.py:116` 定义了 `cors_origins`,注释明说"留空 = 不挂 CORS 中间件(生产走 nginx 同源反代,无需跨域)"——但 `main.py` **根本没用这个字段**,而是自己又 `os.getenv("CORS_ORIGINS", "*")` 读了一遍,默认还反过来变成 `*`。即"留空不挂中间件"这个设计意图**从未被实现**,中间件无条件挂载、默认放行一切。

**风险落地**:生产的 `CORS_ORIGINS` 在 gitignored 的 `docker-compose.override.yml` 里(我看不到)。**如果生产没显式设这个变量,现在就是 `*` + credentials 在跑**。值得上服务器核一下。

**建议**(便宜、与调用方无关都对):落实 settings 的原意——`CORS_ORIGINS` 留空就**不挂** CORS 中间件(生产同源不需要);非空才挂,且用**显式白名单**、`allow_credentials` 仅在确实用 cookie 时才开。顺手把 `main.py` 接到 `lead_settings.cors_origins`,消掉孤儿配置。

---

## 3. api_log 改 allowlist 脱敏:加固项,可做但不急

**现状**(`api_log.py:37` + `trim_request_body` :58-81):用的是 **denylist**——`_DROP_KEYS = ("score_context",)` 只剥掉 `score_context` 一个大字段,**其余请求体原样 `json.dumps` 落库**(16KB 截断)。

**当前落库的字段其实低敏**:
- 4 个接口请求体里只有 `employee_code`、`customer_id`、`lng/lat`、`radius`、分页、行业筛选;**客户 PII(姓名/电话)在响应侧,而响应不落库**(只取 `biz_code` + `result_total`)。
- 唯一偏敏感的是 `lng/lat`——业务员的实时 GPS 位置。
- 且日志**默认关**(`ENABLE_API_LOGGING=0`,`settings.py:119`)。

**为什么仍值得改 allowlist**:denylist 是 **fail-open**——明天谁加个新请求字段(手机号、备注、客户名),只要忘了加进 `_DROP_KEYS` 就会**静默落库**。尤其 `ScoreContext` 是 `extra="allow"`(`request.py:113`),天然容易夹带。改成 allowlist(只放行明确列出的 key)就是 **fail-closed / 隐私默认安全**,而且 `trim_request_body` 已经是个纯函数、有单测缝,改动很小。顺带可以在这里决定 `lng/lat` 到底留不留。

**结论**:cheap 的加固,但不是今天的洞;优先级排在 1、2 之后。

---

## 下一步取决于两个我从代码里看不到的事实

Q2、Q3 我都可以**现在就动手**(都是小改、有测试缝、与调用方无关都成立)。但 Q1(是否/如何加鉴权)严重性和方案,卡在下面两个事实上——帮我确认一下:

确认了**最坏组合:浏览器/H5 前端 + 公网可达**。这把结论显著收紧了——下面是基于这两个事实的最终判断。

## 先说最重要的一句

**现在就存在一个真实的客户 PII 泄露面**:公网任何人访问 aisales.amer.cn,无需任何凭证,即可:
- 用任意 `employee_code` + 一组坐标打 `/score-and-info`,成片拿客户**姓名/电话/固话/地址/联系人**;
- 顺序枚举 `customer_id`(1,2,3…)打 `/timeline`、`/recommend-reason`,拉任意客户的行为时间轴 + AI 理由。

这已经踩到个人信息(PIPL)红线。三项里这是唯一"洞",另两项是它的配套。

---

## 关键认知纠偏:在"公网 + 浏览器前端"下,API-Key 和 CORS 都挡不住枚举

这点必须讲透,否则容易误以为"加个 Key + 配个 CORS 白名单就安全了":

- **共享 API-Key 塞进 H5 前端 = 形同虚设**。前端是公开下发给浏览器的,Key 在 bundle / Network 面板里一眼可见。它只能挡住"无差别扫描器"和裸 `curl`,挡不住任何打开 F12 的人。~20 行能加,但别把它当防线。
- **CORS 根本不是鉴权**。CORS 只约束"别的网站能不能在受害者浏览器里读你的响应",对 `curl`/脚本**完全无效**——攻击者直接发请求枚举 PII,浏览器同源策略压根不参与。所以 CORS 白名单解决的是"第三方站点借用户浏览器读数据",不解决"有人直接刷接口"。

**真正能堵住枚举的,只有"每用户身份鉴权"**:前端登录 → 后端发短时效 token → 接口校验 token 并从 token 取 `employee_code`(而不是信任 body 里那个可伪造的字段)。这也顺带让限流/审计能落到人头上。

---

## 三项的最终判断

### 1. API-Key 鉴权 → 方向对,但"共享 Key"不够,要做就做每用户鉴权

| 方案 | 是否堵住公网枚举 | 成本 | 评价 |
|---|---|---|---|
| 共享 API-Key 进 H5 | ❌(Key 公开可见) | 极低 | 安全剧场,只挡扫描器 |
| **每用户 token(登录→校验)** | ✅ | 中(动前端+身份) | **唯一真解** |
| 边缘限流 + WAF(按 IP/工号) | ⚠️ 减缓非根治 | 低-中 | 真解落地前的止血 |

**要我帮你定方案,得先知道一件事**:业务员的 H5 现在是**怎么进来的**?是裸 H5,还是跑在某个已登录的宿主里(企业微信/CRM 门户/Dify webview)?
- 若已在**有登录态的宿主**里 → 最省力:后端校验那个宿主的 session/passport(仓库已有 `crm_passport` 概念,但它是"按天轮换的共享盐",非每用户,只能算半步),把鉴权接到现有身份上。
- 若是**裸 H5、无登录** → 需要补一套最小登录态(代价主要在前端 + 身份源)。

### 2. 生产 CORS → 现在确定要修(但定位是"补漏",不是"上锁")
- 当前 `main.py:156` 默认 `*` + `allow_credentials=True`,既是最宽松组合、又和 `settings.cors_origins` 的设计意图自相矛盾(那字段是孤儿,没人用)。
- 公网 + 浏览器前端下,**应该**配显式白名单(就你的前端域名),关掉 `*`。这能挡住"恶意站点借业务员浏览器跨域读数据"这一类。
- 但记住上面那句:它挡不住直接枚举。**别因为配了 CORS 就觉得 PII 安全了。**
- 顺手要核:生产 `docker-compose.override.yml`(在服务器、gitignored)有没有设 `CORS_ORIGINS`——**没设的话现在就是 `*` 在裸跑**。

### 3. api_log allowlist 脱敏 → 加固项,但公网下 GPS 这条更值得在意
- 改 allowlist 本身仍是低优先级 fail-closed 加固。
- 但叠加"公网"后有个具体点:一旦 `ENABLE_API_LOGGING=1`,业务员**实时 GPS(lng/lat)**就持续落库。趁改 allowlist,正好显式决定 GPS 留不留。同样要核服务器 override 里 `ENABLE_API_LOGGING` 是不是开着的。

---

## 建议的推进顺序

1. **本周先做(便宜、无歧义、我现在就能改)**:
   - CORS 改成"留空不挂中间件 / 非空走显式白名单",干掉 `*` 默认,并把 `main.py` 接到 `settings.cors_origins` 消掉孤儿配置;
   - api_log 改 allowlist,顺带定 GPS 去留;
   - (并行)上服务器核 `CORS_ORIGINS` / `ENABLE_API_LOGGING` 的实际值。
2. **真正的活(需你拍板)**:每用户鉴权——等你告诉我 H5 的宿主/登录现状,我再出具体方案。
3. **止血(可选,真解落地前)**:host nginx 加按 IP/工号限流,blunt 掉批量枚举。

要不要我**现在就把第 1 步的 CORS + allowlist 两项改了**(单独 commit、带测试)?顺便我可以列一份"上服务器要核的 3 个变量"清单给你。第 2 步的鉴权方案,你先确认下 H5 是怎么进来的。

---
