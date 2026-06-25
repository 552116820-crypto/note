# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `424d37f2-14ae-4885-8ae8-c6f2cd0f3ff6`
- **范围**: Turn 1-10
- **导出时间**: 2026-06-25 16:43:37

---

### Turn 1  *(2026-06-25 07:52)*

**User:**

帮我确认一下，### 问题/风险

**P1 — `app/lead/config/settings.py` Settings 初始化方式存在设计问题**：`settings.py:17-113` 在 `class Settings(BaseModel)` 的 field 默认值处直接调 `os.getenv()`，绕过 pydantic-settings 机制。这意味着 Settings 字段在类定义时就被固化（模块 import 时）——`monkeypatch` 环境变量对已实例化的 `settings` 对象无效，测试必须 `monkeypatch.setattr(settings, ...)` 而非 `monkeypatch.setenv()`。虽然已知问题，但这是一个隐患：如果有测试改了 env 变量期望 settings 跟着变，会静默失效。

**P2 — `app/churn/service.py:_frequency_warnings/_volume_warnings/_product_warnings`（第43-107行）三个私有方法签名几乎一致**（都接 `customer, orders, biz_date, *, settings_by_key, customer_configs, global_configs`），有 DRY 改进空间，但目前的 `_warning_detector()` 派发表（第127-133行）已经做了合理的多态封装，可接受。

**P3 — `crm_api_parsing.py` 中的多 key 解析（第13-33行）**：`_CUSTOMER_API_ID_KEYS`/`_CUSTOMER_API_NAME_KEYS`/`_CUSTOMER_API_SALES_NAME_KEYS` 是硬编码的 CRM 字段别名列表（6/5/6 个别名），这是针对真实 CRM API 字段不统一的防御，但没有注释说明各别名对应哪个 CRM 接口版本，未来维护者无法判断哪些别名还在生产中被使用。

**P4 — `engine.py`（第1-27行）是纯转发层**：导出 `_ewma`/`_prediction_confidence`/`_remove_iqr_outliers` 等下划线开头的"私有"函数。这些函数在 `__all__` 里出现，意味着它们成了公有 API——但名字仍有下划线前缀（暗示私有），命名与 re-export 语义矛盾。测试直接从 `engine` import 这些函数（测试合理），但函数名本身应去掉下划线或改注释澄清。

**Assistant:**

我来逐一核实这四个问题。先把涉及的文件读出来。

先看了 P1 的 `settings.py`。现在并行读其余三个文件。

四个文件都已读完。让我再跑几个验证点：测试里实际怎么 monkeypatch settings（P1）、测试是否真的从 `engine` import 那些下划线函数（P4），以及确认 P2 的行号现状。

P1 的关键对照点已经浮现。再确认一下 churn 配置是怎么读 env 的（用来对比 lead 为什么不一样），然后给结论。

四个点全部核实完毕。结论：**P1–P4 全部成立**，其中 P1 比描述更严重一档，P2 的行号有出入但实质对。逐条给证据。

---

## P1 — 成立（且比描述更强）✅

**确认的事实：**
- `settings.py:22` 是 `class Settings(BaseModel)`——纯 pydantic `BaseModel`，不是 `BaseSettings`；docstring 第 3 行明说 pydantic-settings 没装。
- 字段默认值（24–131 行）全部在**类体执行时（= import 时）**调 `os.getenv()` / `_flag()`，值被烤进类定义。

**描述里轻描淡写、实际更狠的一点：**
不只是"已实例化的 `settings` 对象"无效——**连 `Settings()` 重新构造一个新实例都读不到新 env**，因为默认值在 class 定义时就固化进字段了。所以 `get_settings.cache_clear()` 也救不了；`@lru_cache`(134) 在这里只是单例，根本不是"刷新"手段。唯一能改的杠杆就是 `setattr(settings, ...)`。

**代码库实证 reviewer 说的就是现状：** 所有 lead 测试改配置都走 `monkeypatch.setattr(rs.settings/api_log.settings, ...)`，没有一个 lead 测试用 `setenv` 去动这些字段。

**让"隐患"变具体的关键对照**（同一个 repo 里行为不一致，这才是真正容易踩的地方）：
| 配置 | 读 env 时机 | `setenv` 有效？ |
|------|------------|----------------|
| lead `Settings` | class-def / import 时一次性 | ❌ 静默失效 |
| churn `load_churn_settings()` | 函数体内每次调用读（`config.py:147+`） | ✅ `test_config.py`/`test_startup_isolation.py` 都靠它 |
| shared auth `API_KEYS` | 每请求读 | ✅ `test_auth.py` 靠它 |

也就是说，一个习惯了 churn / auth "setenv 就生效"的维护者，把同样写法搬到 lead 测试里，会撞上 reviewer 警告的那种**静默失效**。

**一点补正（scope）：** 生产不受影响——生产 env 在进程启动前就设好、import 时正确读到。所以这纯粹是**测试期 / 运行时可变性**的 footgun，不是线上 bug。reviewer 定性为"隐患"是准确的。

---

## P2 — 成立，但**行号有出入**，结论同意 ✅

- reviewer 引用的 43–107 / 127–133 **对不上当前文件**：三方法实际在 `58-82 / 84-101 / 103-120`，派发表在 `170-179`。差 15–40 行，应该是 review 时看的是更早的 revision。**实质不变。**
- 签名"几乎一致"属实。但有一个真实差异值得点出：`_frequency_warnings` 额外从 `config` 取 `manual_cycle_days` 并透传（75-81），而 volume/product 直接丢弃 config（`_, _`）。所以**naive 的 DRY 折叠并不免费**——得给 frequency 开特例。这反而**加固**了 reviewer "可接受"的结论。
- `_warning_detector` 派发表（170-179）确实是合理的多态封装。同意：可接受、低优先级。

---

## P3 — 成立 ✅

别名数量**精确对上**：ID 6 个、NAME 5 个、SALES_NAME 6 个。确实零注释说明来历。

补一个能佐证 reviewer 担忧的观察：这些别名肉眼可见是**三套命名规范混在一起**——snake_case（`customer_id`）、小写连写（`customername`）、拼音缩写（`khid`=客户id、`khmc`=客户名称）。这种混杂强烈暗示对应不同 CRM 版本/取数源，正是"未来维护者判断不了哪些还在生产用"的那个维护性缺口。准确，文档/可维护性级别，低优先。

---

## P4 — 成立 ✅

- engine.py 是纯兼容转发层（docstring 1-6 自己就这么说）。
- `__all__`(25-34) 里有 **4 个下划线函数**：`_ewma` / `_prediction_confidence` / `_remove_iqr_outliers` / `_severity_from_ratio`。
- **de-facto 公有已证实**：`tests/churn/test_engine.py:9-13` 直接从 `app.churn.engine` import 这 4 个并断言（52-100）；生产侧 `service.py:13` 也从 engine import 公有 `detect_*`。所以"下划线命名却被 `__all__` 导出 + 外部 import"的语义矛盾是真的。

补一个对修法有用的细节：这些下划线名在它们的**老家模块**（`anomaly_strategies.cycle_prediction` / `.support`）里其实是合法的模块私有内部件——矛盾**只在 engine.py 用 `__all__` 把它们提升为公有时才产生**。所以最干净的修法不止"去下划线 / 加注释"，还有第三条：engine `__all__` 不再 re-export 这些内部件、让测试改从老家模块 import。三选一，加注释是最小成本那条。

---

需要的话我可以把 P1（统一成 churn 那种 `load_*()` 工厂函数、让 `setenv` 在 lead 也生效）或 P4（收敛 `__all__`）任意一条落成具体改动——都能单独成 commit。要不要动？

---

### Turn 2  *(2026-06-25 08:00)*

**User:**

这些值得改吗？

**Assistant:**

一句话先给结论:**四个都不是 bug,没有一个"必须改";真正值得顺手做的只有两行注释,其余不值得。**

理由是它们的成本结构差异很大——P2/P3/P4 全在 churn,受铁律#2 约束(单独 commit + ~50 人灰度验证),碰逻辑的改动等于为"代码洁癖"付灰度税。

## 逐条判断

| 项 | 值得改? | 一句话理由 |
|----|--------|-----------|
| **P1** lead settings 固化 | 🟡 加注释值得,重写不值得 | 隐患真实但"静默"被高估;结构重写 ROI 低 |
| **P2** 三方法 DRY | ❌ 不改 | reviewer 自己说"可接受",且 frequency 有特例、折叠不免费 |
| **P3** CRM 别名无注释 | 🟢 顺手加一行总括注释 | 成本≈0、纯增量、不碰逻辑 |
| **P4** engine `__all__` 下划线 | ❌ 不改 | 纯 cosmetic,零功能收益却要付 churn 灰度成本 |

## 几个关键的"为什么不值得"

**P1 的"静默"其实被夸大了。** 想清楚触发链:如果某测试 `setenv("MAX_CANDIDATES","5")` 然后断言依赖 `settings.max_candidates==5`——env 不生效 → 值还是 20000 → **断言直接红 → 开发者当场改用 `setattr`**。这不是静默。真正静默的窗口很窄:只有当测试**不直接断言那个值**、而是测下游行为、且下游对默认值也碰巧通过时才会发生。概率中低。所以隐患成立,但不紧迫。

结构性修复(改成 churn 那种 `load_settings()` 工厂)还有传染性:全项目 `from ...settings import settings` 都吃模块级单例,改干净是大改,为一个没造成实际问题的隐患引入回归风险,不划算。**最小有效护栏 = 在 `get_settings` 处加一行注释**:"env 在 import 时固化,测试请 `monkeypatch.setattr(settings, ...)`"。

**P2/P4 碰的是 churn 核心路径。** 为了消除三个 20 行相似方法、或让 `__all__` 命名好看,去动检测派发 / 兼容导出层,然后走一遍灰度——收益(可读性)远抵不过成本(铁律#2 的验证开销 + 回归面)。明确不动。

**P3 是四个里唯一"纯赚"的**,但有个前提:别去考古每个别名的精确来历(`khid`/`clientcode` 各对应哪个 CRM 版本,这信息可能已不可得,强写注释反而误导)。只加一句**总括**:"历史上见过的 CRM 字段别名,大小写/拼音混排因对接过多个版本与数据源"——把"为什么这么杂"讲清楚就够了。

## Bottom line

如果你想清掉这次 review,我建议只做**两行注释**(P1 的测试约定 + P3 的别名总括),其余记为"已评估、可接受"即可。这两条:
- 不碰任何逻辑,P3 虽在 churn 但只是注释、不触发铁律#2 的灰度要求;
- 可以合成一个 `docs/chore` 小 commit。

要我把这两行注释加上吗?还是你更想连这两条也先记进 review 结论、暂不动代码?

---

### Turn 3  *(2026-06-25 08:06)*

**User:**

要

**Assistant:**

好,这就加。两条独立,一并改。

两条注释加好了。跑一下门禁确认格式/lint(主要看中文注释行长有没有触发 E501)。

门禁全绿,纯增量 6 行、零逻辑改动:

- `app/lead/config/settings.py` +3:`class Settings` 上方说明"默认值 import 时固化、测试用 `setattr` 不能用 `setenv`",并点名对照 churn 的 `load_churn_settings()`。
- `app/churn/crm_api_parsing.py` +3:别名集合上方总括"多 CRM 版本/数据源混排、来历不可考、只增不删"。

P2/P4 按结论不动。

## 关于 commit

我**还没 commit**(你只说了"加注释")。要落地的话有两个项目约定得照顾:

1. **在 main 上** → 按规矩先开个分支。
2. **铁律#2** → `crm_api_parsing.py` 在 `app/churn/` 下,算 churn 改动,字面要求"单独成 commit、别塞进 lead 顺手带"。虽然是纯注释、不触发灰度,但拆开更稳(也尊重 bisect 习惯)。

所以建议:开一条 `docs/review-followup` 分支,拆**两个 commit**——一个 lead 注释、一个 churn 注释。

要我按这个开分支 + 两个 commit 吗?还是你自己来?

---

### Turn 4  *(2026-06-25 08:11)*

**User:**

那就在main上拆成两个commit

---

### Turn 5  *(2026-06-25 08:11)*

**User:**

那就在主分支main上拆成两个commit

**Assistant:**

明白,按你说的直接在 main 上提交,churn 与 lead 各一个 commit。

两个 commit 都进去了,各只含一个文件。确认一下收尾状态:

完成。两个 commit 已落在 main,工作区干净:

| commit | scope | 文件 | 改动 |
|--------|-------|------|------|
| `d46621b` | lead | `settings.py` | +3 行注释:默认值 import 时固化、测试用 `setattr` |
| `602f352` | churn | `crm_api_parsing.py` | +3 行注释:别名多源混排、只增不删 |

churn 与 lead 各自单独成 commit,满足铁律#2;纯注释、零逻辑改动,基线 559 不受影响,无需跑测试。

这次 review 的四条到此:**P1/P3 已用注释固化结论,P2/P4 评估后判定不值得改**。还没 `push`——要推到远程的话我再跟一条(不过按之前 memory,你这边远程同步还没完全就绪,需要的话告诉我)。

---

### Turn 6  *(2026-06-25 08:15)*

**User:**

按之前 memory,你这边远程同步还没完全就绪，这是什么意思？我已经建立远程了啊，是不是memory太旧了

**Assistant:**

你说得对,我不该凭 memory 下判断——直接看真实状态。

真相出来了——**你是对的,memory 过时了**。当前实际状态:

**远程其实有两个,都建好了:**
- `github` → `github.com/552116820-crypto/amer_sales_assist.git`
- `origin` → `113.78.134.101:927/.../amer_sales_assist.git`(你自建/内网那台)

我那条 memory 是 2026-06-12 深度合并时写的"遗留尾巴",里面把"git 远程"列成未办项,到现在(6-25)两周过去已经不成立了。我刚才那句"远程同步还没完全就绪"就是直接背了旧 memory,没核实——这是我的错。

不过有个**真实的细节**值得你知道,跟"建没建远程"无关:

```
* main 602f352   ← 本地 HEAD（含刚才两个 docs commit）
## main          ← 注意没有 [origin/main]，本地 main 没设 upstream tracking
origin/main = 4c98757 (cut v2.4.4)  ← 落后本地 2 个 commit
```

所以:
1. **本地 `main` 没有 upstream tracking** → 这就是为什么 `git status` 不显示 ahead/behind,也是为什么我得手动看才知道领先 2 个。push 时要显式 `git push origin main`(或加 `-u` 一次性把 tracking 设上)。
2. **那两个 docs commit 还没推到任何远程**,origin/main 还停在上次 release。

让我先确认下 github 那边 ref 的位置,然后把 memory 修正掉。

两个远程都建好了、都 fetch 过,`github/main` 和 `origin/main` 都停在 `4c98757`(上次 release),本地领先它俩各 2 个 commit。我那条 memory 确实是旧的,我来修正它。先读一下原文。

这条 memory 把"git 远程"列成遗留待办,我来修正它(顺带核实过的留痕,没核实的项不动)。

memory 正文已修正,再更新 index 里那行 hook。

改完了。memory 那条过时判断已经修正——你是对的,它是 6-12 写的,两周前的状态,我刚才不该直接背它。三处都更新了(memory 正文 frontmatter + body、MEMORY.md 索引行),把"git 远程未建"改成"已建,2026-06-25 复核",没核实的项(服务器切换/装 make)标了"未复核"没乱动。

**当前真实状态汇总:**

| 项 | 状态 |
|----|------|
| `github` 远程 | ✅ 已建,`github/main` = `4c98757` |
| `origin` 远程(自建) | ✅ 已建,`origin/main` = `4c98757` |
| 本地 `main` | 领先两个远程各 **2 commit**(刚才两个 docs) |
| upstream tracking | ❌ 未设 → `git status` 才不显示 ahead/behind |

所以"还没推"是真的,但"远程没建好"是假的——我混淆了。那两个 docs commit 现在哪个远程都还没有。

要推的话,因为 main 没设 upstream,得显式指定。你有两个远程,我需要你定:
- **只推 `origin`**(自建那台)?`git push origin main`
- **只推 `github`**?`git push github main`
- **两个都推**?
- 顺带要不要 `-u` 把某个设成默认 upstream(以后 `git push` 不用再带远程名)?

告诉我推哪个、要不要设 upstream,我就执行;或者你想自己推也行。

---

### Turn 7  *(2026-06-25 08:18)*

**User:**

你这个memory是放在哪里的，应该是项目级的memory才对吧

**Assistant:**

好问题,这里有个容易混的点,我拆开讲。

**它在哪:**
```
/home/huangch001002/.claude/projects/-mnt-e-amer-sales-assist/memory/
```
那串 `-mnt-e-amer-sales-assist` 就是本项目路径 `/mnt/e/amer_sales_assist` 的编码。

**"项目级"这个词的两层含义,它满足一层、不满足另一层:**

| 维度 | 这套 auto-memory | 你可能以为的"项目级" |
|------|-----------------|---------------------|
| **作用域**(哪个项目加载) | ✅ 按项目隔离——只有在 `/mnt/e/amer_sales_assist` 下开会话才加载这套,换项目就是另一套 | ✅ 一致 |
| **物理位置** | 在你 home 的 `~/.claude/` 下,**不在 repo 里** | ❌ 你大概以为在项目目录内 |
| **进不进 git** | ❌ 不跟踪、不 commit/push、团队看不到 | 你可能期望进版本库 |
| **跨机器/换 clone** | ❌ 只属于你这台机器+这个用户,重装/别人 clone 就没了 | 你可能期望跟着 repo 走 |

所以它是**"按项目隔离、但用户私有、不进版本库"**的记忆——设计上就放在 home 下,不是放错位置。它的定位是"我和你在这个项目协作时的私人上下文/偏好",不是团队共享的事实源。

**如果你的真实诉求是"这些项目状态该团队可见、跟着 repo 走"**,那就不该靠 auto-memory,而要落到 repo 内的文件:
- `CLAUDE.md`(项目指令,已在 git 里、每次会话加载)——适合规则/铁律/命令;
- `docs/`(runbook / handover 那类)——适合"远程已建、push 待办、upstream 未设"这种运维状态。

刚才那条"遗留尾巴"正是个例子:它记的"git 远程/服务器切换/装 make"其实是**项目运维事实**,放我私有 memory 里 → 只有我这个会话看得到、还容易过时(就像刚才)。如果挪进 `docs/runbook` 或 handover,团队和换机都能看到、也能随 commit 留痕。

要不要我把这类项目事实从我的私有 memory 迁一份进项目文档?那样就真成"项目级"了,不依赖我的 home 目录。

---

### Turn 8  *(2026-06-25 08:23)*

**User:**

/home/huangch001002/.claude/projects/-mnt-e-amer-sales-assist/memory/，它这里的memory还有什么价值吗？

**Assistant:**

让我先看一眼这 9 条 memory 各自是什么类型——价值高低恰恰取决于此,空谈不如直接体检。

类型出来了:**7 条 project、2 条 feedback**。这个分布恰好说明问题——价值高低基本沿着这条线分。

**一句话回答:有价值,但价值高度集中在一类东西上,其余的正在贬值(刚才那次过时就是贬值的证据)。**

价值锚点是这个:**这个目录的用处不在"记录项目",在"记住你"**——你否决过什么、为什么不做某件事、踩过哪个坑别再踩。这部分恰恰是 `CLAUDE.md`/`docs` 不适合放的(那是团队事实,不是"你某天怎么拍板的"),可又必须跨会话记住,否则我每次都会重新建议你已经否决的事、重新问你答过的问题。这是 auto-memory **不可替代**的地方。

反过来,"项目是什么 / 已完成什么"——代码、git log、CHANGELOG、docs 已经是 source of truth,memory 里再存一份就是副本,会过时、会和 repo 打架。我的 memory 指南其实也明写了"别存 repo 已经记录的东西",但有几条违反了。

**给你这 9 条做个体检:**

| 价值 | 条目 | 判断 |
|------|------|------|
| 🟢 不可替代 | `churn-test-strategy`(别再提真库测试)、`deps-upgrade-deferred`(决定不升)、`doc-sync`(否决 Unreleased)、`lead-max-candidates`(调参约束) | 都是**你的决策/否决/踩坑**,防我重复犯错——留 |
| 🟢 有用但放错层 | `codex-zombie-resume`(WSL Codex 恢复técnica) | 是**跨项目**技巧,该进全局 memory 而非项目级 |
| 🟡 会过时,该迁 repo | `merge-pending-tail`(远程/服务器/make)、`aisales-https-deploy`(架构)、`lead-api-auth-plan`(计划) | 项目运维/架构事实 → 迁 `docs/`,放这里只会漂移 |
| 🔴 与 git 重复 | `structure-review`、`merge` 的"已完成"清单 | git log + CHANGELOG 已是 source of truth,基本是副本 |

所以真正发挥 auto-memory 价值的,是 🟢 那几条"决策内核";🟡🔴 加起来快一半,是它在贬值的部分。

要不要我顺手做一次清理:🔴 删掉(git 已有)、🟡 迁进 `docs/`(顺便就把"远程已建/push 待办"落进 runbook,真正项目级)、🟢 保留并把夹在 project 里的决策提纯成 feedback?一次性把这个目录收敛到"只剩记住你的东西"。

---

### Turn 9  *(2026-06-25 08:30)*

**User:**

先删掉git已有的memory；
对于会过时，该迁repo的memory，应当先审核一下它们的内容，看写了什么
codex-zombie-resume，具体写了什么？
churn-test-strategy，deps-upgrade-deferred，doc-sync，lead-max-candidates，这几个不可替代的meomory写了什么？

**Assistant:**

明确了。我先把要审核/展示的文件都读出来(删 `structure-review` 之前也要核实它确实纯属 git 重复、没夹带独有决策),然后再执行删除。

8 个文件一起读:

在删之前先核实——结果发现 `structure-review` **不是纯 git 重复**:第 23 行那段"协作/踩坑"(codex 揪出 3 处真问题、WSL 下 codex 卡死、**机械大重构交 codex 务必自己跑 pytest/mypy/ruff 独立复核、别盲信它自报全绿**)是经验教训,git 记不了,而且跟 `codex-zombie-resume` 主题重叠。这条我先不盲删,等会儿单独跟你确认。

继续把其余 7 个读出来给你审核:

典型的🟢——记的就是"用户 2026-06-24 权衡后暂不做真库测试、别再重提"。继续把剩下 5 个一次读完:

剩下 4 个一次并行读完(🟢 两条 + 🟡 两条):

最后 3 个:

9 个全读完了。读完有个**重要更正**要先告诉你:我上一轮基于索引 hook 的"🔴可删/🟡迁走"分类,逐条看完**大半站不住**——几乎每条都夹带了 git 记不了的决策/经验,能无脑整删的其实一条都没有。逐项交代。

## 一、"删 git 已有的"——核实后:没有能安全整删的

两个🔴候选都夹带了 git 之外的东西,删前必须先抢救:

**`structure-review-2026-06-13`** — 99% 是已完成清单(A组5+B组4 commits、hash 全标了、507 绿),git log/CHANGELOG 确实是 source of truth。**但**第 23 行那段协作踩坑不是 git 能记的,而且和 `codex-zombie-resume` 同主题:
> codex 揪出 3 处真问题、WSL 下 codex 卡死 ~4min;**机械大重构交 codex 执行,务必自己跑 pytest/mypy/ruff+grep 独立复核,别盲信它自报全绿**

**`merge-completed-pending-tail`** — "已完成"清单 git 有,但整条还含运维遗留(远程/服务器/make,刚修过远程那块),不能整删。

→ 这步我**没硬删**(memory 不在 git,删了找不回)。正确做法是**瘦身**:structure-review 把协作段并进 codex 那条后删本体;merge 删已完成清单、留遗留+踩坑。

## 二、🟡 两条审核(你要看内容再定)——读完发现都不是简单"迁走"

**`aisales-https-deploy`** — **迁不进 repo**:它自己写明 edge-gateway/override"仅在服务器、不在 repo、fresh clone 会丢",你已有 note(`D:\note\…13.部署https`)。三块内容:
- ① 架构(edge 中立网关按 SNI 分流 dify/aisales、Dify 让出 80/443→8081/8444)— Dify 一升级就可能变,**易过时**
- ② 🔴运维:Dify 升级后查端口、aisales 证书 **2026-09-14 到期**、`ROLLBACK.sh`
- ③ 排障结论:"比 :8080 慢" = 客户端 DNS ~12s + 冷算 ~10s,**不是 HTTPS**(别再当 nginx 问题查)

价值在②③(到期提醒 + 防重复查错方向);①交给你的 note。

**`lead-api-auth-and-ratelimit-plan`** — **不是"已完成可删"**:鉴权已落地(`32714b3`,git 有),但**限流是未落地设计 + XFF 查证**,git 完全没有:
> `TRUSTED_PROXY_HOPS=1`(关键纠正——edge 直连 uvicorn、绕过 compose nginx,只一跳;**填 2 会全员同桶、限流废掉**);仅剩"edge 443 前有无云 LB"待 4G 实测

下次做限流直接救命。建议:删已落地鉴权流水、留限流设计+XFF 结论。

## 三、🟢 不可替代的内核(你问的内容)

| memory | 写了什么(精炼) |
|--------|--------------|
| **codex-zombie-resume** | WSL 下 Codex worker 中途死、companion `status` 仍显示 running(僵尸)。恢复三步:`ps -p <pid>` 空 + broker.sock 没了=确认真死 → companion `cancel <task-id>` → `task --resume-last "你上轮已完成…直接输出报告"` 接回原 thread、不重跑。 |
| **churn-test-strategy** | 2026-06-24 补 churn 测试选**确定性优先**(httpx 原生 Response 测 HTTP 解析 + MySQL dialect 编译断言测 upsert,不连真库)。核心护栏:**用户权衡后明确暂不做真库 integration,即便 CI 已具备条件——别再因"CI 有了"重提**;不碰 conftest 的 `_block_real_mysql`。 |
| **deps-upgrade-deferred** | 2026-06-23 盘点:ruff/mypy/pytest + fastapi/starlette/redis 都落后一个大版本,**用户决定暂不升、维持保守 pin**。actionable:ruff 升必须**三处同步**(pyproject+pre-commit+ci.yml);starlette 7 条 CVE 已评为不可利用/低危并 `--ignore-vuln` 挂账,**升级时记得删挂账**恢复拦截。 |
| **doc-sync-on-migration-change** | cut release/加迁移时散落的同步点:测试基线 N 要同步 **9 个文件**(不止 CLAUDE.md)、版本号代码两处+文档两处(handover/technical-design 常漏)、§18.1 env 默认值镜像表。**偏好:不用 CHANGELOG Unreleased 段、勿擅自定版本号**。(注:line 31 也夹了过时的"远程遗留",得顺手修。) |
| **lead-max-candidates-tuning** | MAX_CANDIDATES 调大的代价链(blob 体积/冷算/db2 内存线性放大)+ 三硬约束(锁窗口/Redis 512MB `volatile-lru` 才是真天花板/首屏延迟),**30000 = 不联动其它 env 的安全上界**。2026-06-17 生产实测 redis 仅 1.77M/512M、1003 偶发 → 选定 20000→30000。下次再问调这个值直接引此约束。 |

## 更正上一轮 + 建议

我上轮说"🟡🔴快一半在贬值"**估重了**。真相:**贬值的只是每条内部的流水段(已完成 commits 清单、已落地代码描述),不是整条**;几乎每条都夹带了 git 记不了的决策/经验/排障结论。所以这目录价值比我上轮说的高——它确实在"记住你",只是和流水混着。

正确动作不是"删几条"而是"**每条瘦身**":砍 git 已有的流水段、留决策内核、顺手修过时点(远程遗留出现在 merge + doc-sync 两条)。具体清单:

1. `structure-review`:协作段并入 `codex-zombie-resume` → 删本体
2. `merge`:删已完成清单,留遗留+踩坑
3. `aisales`:精简到运维②+排障③
4. `lead-api-auth`:删已落地鉴权,留限流设计+XFF,转 `feedback`
5. `doc-sync`:修掉 line 31 过时远程遗留
6. 🟢 其余:保留

要我按这 6 条动手吗?还是你先挑几条做?

---

### Turn 10  *(2026-06-25 08:36)*

**User:**

要

**Assistant:**

开始按 6 条动手。先 grep 一遍 memory 之间的 `[[链接]]` 关系,确保删/改时不留死链。

链接图确认:`structure-review` 没有任何反向引用,删它不留死链;其余指向 `merge` 的链接(doc-sync/codex/lead-max-candidates)都还有效(merge 保留)。开始按 6 条执行——独立文件一并处理:

协作经验已并入 codex 那条。现在并行:删 structure-review、重写 merge/aisales/lead-api-auth 瘦身版、修 doc-sync 过时行、更新索引。

最后更新索引——删 structure-review 行,并把抢救进 codex 那条的协作经验补进它的 hook:

6 条全部执行完毕。做一次一致性核验——确认条目数对齐、无死链、所有引用都指向存在的文件:

清理完成,8 条全部核验通过——索引对齐、零死链、`structure-review` 引用清空。逐条交代做了什么:

| 条目 | 动作 | 结果 |
|------|------|------|
| `structure-review` | **删除**,内核抢救 | 协作经验(别盲信 codex 自报全绿、机械重构自己跑门禁)已并入 `codex-zombie-resume`;已完成清单交给 git log |
| `merge` | **瘦身** 33→22 行 | 砍掉 v2.2/v2.3 已完成叙事 + 候选任务进度(git 有);留遗留待办(远程已建/服务器/main.py 泛化)+ WSL 环境备忘 |
| `aisales` | **瘦身** 23→18 行 | 砍掉易过时架构细节(交用户 note);留运维(Dify 抢端口/证书 9-14/回滚)+ 排障结论(慢=DNS 非 HTTPS) |
| `lead-api-auth` | **瘦身** 24→18 行 | 砍掉已落地鉴权实现(git 有);留未落地限流设计 + XFF 关键结论(`HOPS=1`) |
| `doc-sync` | **修过时** | line 31 "git 远程仍是遗留尾巴" → "已建,main 未设 upstream" |
| `codex-zombie-resume` | **并入 + 索引 hook 更新** | 多了一段大重构协作经验 |
| 🟢 churn-test / deps / lead-max-candidates | 保留不动 | 决策内核,不可替代 |

顺带把 `merge`/`aisales`/`lead-api-auth` 三条的 frontmatter `description` 也改成反映瘦身后定位(recall 时更准)。

**最终状态:8 条,每条都收敛到"git 记不了的东西"**——你的决策/否决、调参约束、排障结论、协作踩坑。流水叙事全部让给 git log/CHANGELOG。这些都是我私有 memory 的整理,不进版本库,所以没有 commit 这一说。

---

另:之前那两个 docs commit(`d46621b` lead 注释 / `602f352` churn 注释)**仍未 push**,两个远程都还停在 release 点。需要推的时候说一声推哪个(`origin` / `github` / 都推),不急。

---
