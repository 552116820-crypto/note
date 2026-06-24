# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `5ad7b38c-0b19-4b81-b2fc-094bfa5b34b6`
- **范围**: Turn 2-10
- **导出时间**: 2026-06-24 09:37:28

---

---

### Turn 2  *(2026-06-24 00:20)*

**User:**

帮我补 churn CRM HTTP 测试(respx mock HTML/302/重试)+ mysql_repo MySQL 专属 SQL(ON DUPLICATE KEY)集成测试

**Assistant:**

这是 codex 改进项 #2 的落地。动手前我先把现状摸清——尤其要注意两个约束：① `conftest.py` 的 `_block_real_mysql` autouse 守卫会让**任何真连 MySQL 的测试炸掉**(538 套件的"无外部依赖"契约);② 铁律2 要求 churn 改动单独成 commit。先读 HTTP 层与 repo 层的真实实现和现有测试风格。

现在我对两个任务的现状和约束都清楚了，关键判断如下：

**CRM HTTP 测试** — 现有 `test_crm_api.py` 用 `_FakeResponse`(鸭子类型)桩掉了真实 httpx 栈,所以 `.json()`/`.raise_for_status()` 的真实行为从未被测。我**不引入 respx**(项目依赖很保守),改用 httpx **原生** `Response` 构造真实响应对象 + monkeypatch `httpx.post`——这样 `.json()` 对 HTML 会真的抛 `JSONDecodeError`、302 会真的触发 `raise_for_status`,正是 codex 想覆盖的真实路径,且零新依赖。

**mysql_repo upsert** — 这里有硬约束:`conftest.py` 的 `_block_real_mysql` autouse + "538 全确定性无外部依赖"契约,意味着**默认套件里不能真连 MySQL**。我先交付**确定性**的方言级验证(FakeEngine 捕获 `save_detection_run` 走的语句 + 用 MySQL dialect 编译断言 `ON DUPLICATE KEY UPDATE` 覆盖所有可变列),真连库的 integration 套件涉及改"焊死的守卫",留到最后跟你确认。

先写两个确定性测试文件:

两个测试文件已写好，先单独跑它们验证我对 httpx 真实行为和 MySQL 方言编译的假设：

9/9 通过——我对 httpx 真实行为和 MySQL 方言编译的假设都验证了。现在跑全量门禁(pytest 基线 + ruff + mypy)确认没破坏既有测试、新文件符合规范:

全绿:**ruff ✓、mypy ✓ (81 files)、pytest 547 passed** (538 → 547,新增 9 个全部确定性)。那 4 个 warning 是既有的 SQLite/Python3.12 `DeprecationWarning`(codex 也点过),与新测试无关。

基线从 538 变 547,文档里的基线数字需要同步(否则就是我们评审时批评的那种文档漂移)。先定位所有引用点:

9/9 新测试通过、547 全绿、ruff/mypy 干净。基线 538→547 是铁定事实(无论后续做不做真库 integration,`547 passed` 都成立),先把 7 处文档基线同步掉,避免文档漂移:

7 处文档基线已同步,无 538 残留。

到这里 **codex 三类 HTTP 场景(HTML/302/重试)+ MySQL 专属 SQL(ON DUPLICATE KEY)** 的**确定性**测试已经交付完成。剩下一个需要你拍板的范围决策——关于"真连 MySQL 的 integration 套件"。

我先说清现状再问:mysql_repo 的 upsert 现在有了**方言级确定性验证**(用 MySQL dialect 编译 `save_detection_run` 实际执行的语句,断言 `ON DUPLICATE KEY UPDATE` 覆盖全部可变列、主键不被更新、非 MySQL 方言不误走该分支)。它能抓住**最可能发生的回归**(加列漏同步、方言退化、分支选错)。但它**不验证"插入→再 upsert 真的不主键冲突"这一执行语义**——那个只有真 MySQL 能测,而真连库会撞上 `conftest` 的 `_block_real_mysql` 守卫和"538/547 全确定性"契约。

要不要再往前走一步,取决于你:

用户选了**就此收尾**——保留确定性方言验证,不碰焊死的守卫去做无 CI 时跑不起来的真库 integration。这是务实的判断。任务到此完成,我确认一下完整改动面:

全部确认:2 个新测试文件(churn-only,无跨模块依赖,符合铁律1/2)+ 7 处文档基线同步。任务完成。

我把这次的测试策略决策记一条 memory(未来遇到"真实集成路径只有 fake 覆盖"的洞时可参考):

完成。给你这次补测试的交付总结。

## 交付内容

**两个新测试文件,9 个测试,全部确定性、churn-only:**

**① `tests/churn/test_crm_api_http.py`(7 个)** — 补上 codex 指出的"真实 HTTP 栈从未被测"的洞。关键决策:**没引入 respx**(项目依赖很保守),改用 httpx **原生 `Response`** 构造真实响应 + monkeypatch `httpx.post`,所以测的是**真的** `.json()`/`.raise_for_status()` 行为:
- HTML 错误页(200+`<html>`)→ 真实 `.json()` 抛 → 归一成可读 `RuntimeError`
- 302 跳转 → 验证 httpx 默认不跟随、`raise_for_status` 抛 `HTTPStatusError`(不被静默当成功)
- 合法 JSON 但非对象(数组)→ `RuntimeError`
- TransportError 重试:瞬时失败可恢复 / 连续失败耗尽 4 次后上抛

**② `tests/churn/test_mysql_repo_dialect.py`(2 个)** — 补 `ON DUPLICATE KEY UPDATE` 这条 SQLite 测试永远走不到的 MySQL 分支。用 **MySQL dialect 编译** `save_detection_run` 实际执行的语句,断言:upsert 覆盖**全部 9 个可变列**(防加列漏同步)、主键 `run_id` 不被更新、非 MySQL 方言不误走该分支。

## 验证(门禁全绿)

| 项 | 结果 |
|----|----|
| pytest | **547 passed**(538→547,新增 9 个) |
| ruff | All checks passed |
| mypy | Success, 81 files |
| 铁律1 | 新测试无跨模块 import ✓ |

7 处文档基线(`CLAUDE.md`/`README`/`Makefile`/4 篇 docs)已同步 538→547。

## 一个诚实的边界

按你的选择,**只做了确定性验证,没碰真库**。方言断言能抓住最可能的回归(漏列/方言退化/分支选错),但**"插入→再 upsert 真的不主键冲突"这个执行语义没被验证**——那个只有真 MySQL 能测。等 CI(codex 改进项1)落地后,再给 `conftest` 守卫开 `@pytest.mark.integration` 豁免口子补上即可(已记进 memory)。

## 提示

这是一笔干净的 **churn-only** 改动(2 新测试 + 文档基线),符合铁律2"churn 改动单独成 commit"。要我帮你 commit 吗?需要的话我会按你的 commit 规范写。

---

### Turn 3  *(2026-06-24 01:10)*

**User:**

我其实已经把CI做好了，你看看

**Assistant:**

啊——这是我必须纠正的地方。我和 codex 评审时都判"无 CI",但你 memory 里明确写着 CI 在 commit `97b77e0/37f459a/39fbf20` 就落地了,而且我会话开头的 git 快照最新 commit 其实是 `e67c89e ci(gitlab)...`。问题出在我评审时跑 `ls .github` 那一刻的 git HEAD 停在 `5eefa0a`(CI commit 之前),于是看了个空——**"无 CI"是基于错位 HEAD 的误判,不成立**。

让我看当前真实状态和你的 CI 配置:

真相清楚了:对话期间你把 CI 那串 commit(`97b77e0→44b4f89`)引到了当前分支,HEAD 从 `5eefa0a` 前进到 `44b4f89`,`.github/workflows/ci.yml` 现在确实在。我评审那刻 HEAD 还停在 CI 之前,所以"无 CI"是**错的**,正式更正——codex 改进项#1 你其实早做完了,GitLab CI 还试过又因自建 runner 没权限删掉(`44b4f89`),只留 GitHub Actions。

我认真读一下你的 CI 和 pre-commit 配置:

这个 CI 配置质量很高。我先核两件事:① 我刚改基线 538→547 时**漏了 ci.yml 的注释**(评审时我以为没有 .github,sed 没带上它),得补一致;② 确认我新增的 2 个测试 + 文档改动**能通过你的 CI**(尤其 CI 把 ruff pin 在 0.14.14,要确认我本地 format 结果和它一致,不会触发 `format --check` 假报错)。

并行修注释 + 复刻 CI 的精确门禁验证:

API Error: Connection closed mid-response. The response above may be incomplete.

---

### Turn 4  *(2026-06-24 01:18)*

**User:**

API Error: Connection closed mid-response. The response above may be incomplete.怎么回事

**Assistant:**

先回答你的问题:

**"Connection closed mid-response" 是客户端(Claude Code)和 Anthropic API 之间的网络连接被中途掐断**,属于传输层抖动,**跟你的代码、CI、这次改动都无关**。关键是:**那一轮的工具调用其实都执行完了、结果都返回了**,断的只是我后面那段总结文字。所以没有任何工作丢失或写坏。我接着把被打断的收尾给你。

## 好消息:CI 验证全部通过

复刻你 CI 的精确门禁,我新增的 2 个测试 + 文档改动**能干净通过**:

| CI 步骤 | 本地复刻结果 |
|--------|------|
| ruff 版本 | 本地 **0.14.14 == CI pin 0.14.14** → format 零漂移 |
| `make lint`(ruff check) | All checks passed ✓ |
| `ruff format --check` | 144 files already formatted ✓ |
| `make typecheck`(mypy) | Success, 81 files ✓ |
| `make test`(pytest) | 547 passed ✓ |

我还顺手补了一处一致性:`ci.yml` 注释里两处"基线 538"→"547"(我评审时 HEAD 错位没看到 .github,改基线时漏了这个文件,现已对齐)。

## 你的 CI 配置 review(你要的"看看")

**质量很高,和项目其它部分一样有"决策留痕"的风格。** 几个真正做得好的点:

- **复用 Makefile target**(`make lint/typecheck/test`)当 CI 步骤——CI 和本地跑同一组命令,单一真相源,杜绝"本地绿 CI 红"的漂移。这是很多项目做不到的。
- **ruff 三处 pin 同步**(pyproject `[dev]` / pre-commit rev / ci.yml 都钉 0.14.14),还专门写了注释防 `format --check` 因 patch 漂移假报错——我实测本地正好也是 0.14.14,验证了这个设计有效。
- **Python 3.12 对齐生产镜像**,避免门禁环境和生产跑不同解释器。
- **pip-audit 逐条 `--ignore-vuln` 挂账**,每条注释写清为何豁免(starlette 7 条已按本项目用法核实不可利用)、何时该删——负责任,且 memory 里也兜了"升级时删 ignore"的提醒。

**可选的锦上添花**(都不是必须,看你要不要):

1. **`concurrency: cancel-in-progress`** — 同一 PR 连推几次会并发跑多个 run,加这个取消旧 run 省 runner。
2. **`permissions: contents: read`** — 收紧 GITHUB_TOKEN,这个 CI 不需要写权限。
3. **mypy 没 pin 版本**(只 ruff pin 了)——mypy 1.x patch 升级偶尔会多报类型错让 CI 红而本地绿,在意的话可以也 pin。
4. **没有 `docker build` 验证**——生产是 Docker 部署,Dockerfile 改动可能四道门禁全绿但镜像构建挂;加一句 `docker build .` 成本很低。

---

那么对整个评审做个**正式更正**:codex 改进清单里 **#1(CI)你早做完了**,**#2(集成测试)这次补上了**。把当时"无 CI"那条划掉——那是我评审时 git HEAD 停在 CI commit 之前导致的误判,不是你的缺口。

需要我把上面 4 个 CI 小改进里的哪几个直接动手加上吗?还是先把这次的 churn 测试 + 文档 + ci.yml 注释一起 commit?

---

### Turn 5  *(2026-06-24 01:20)*

**User:**

给 conftest 守卫开 @pytest.mark.integration 豁免口子补上，你之前说的这个是什么意思

**Assistant:**

好问题,而且问得正是时候——因为**前提变了**。我先把这个机制讲清楚,再说为什么现在它的性价比和我上次评估时不一样了。

## 这个"豁免口子"是什么意思

你的 `tests/conftest.py` 里有个**全局守卫**(autouse,每个测试自动生效):

```python
@pytest.fixture(autouse=True)
def _block_real_mysql(monkeypatch):
    import pymysql
    def _blocked(*args, **kwargs):
        raise RuntimeError("TEST GUARD: 真实 pymysql.connect() 被调用……")
    monkeypatch.setattr(pymysql, "connect", _blocked)
```

它把 `pymysql.connect` 换成一个"一被调用就炸"的函数,**焊死"全部测试确定性、无外部依赖"的契约**——任何测试想偷偷连真库,立刻 RuntimeError 暴露,而不是静默读写生产库。

**矛盾点**:真库 integration 测试**天生就要**真的 `pymysql.connect` 到一个 MySQL。可这个守卫是 `autouse`、对**所有**测试生效,所以 integration 测试会被自己人(守卫)当场炸掉,根本跑不起来。

**"开豁免口子"= 让守卫认得 integration 测试并放它过**。pytest 的 fixture 能拿到 `request`,据此判断当前测试有没有标 `@pytest.mark.integration`:

```python
@pytest.fixture(autouse=True)
def _block_real_mysql(request, monkeypatch):
    if request.node.get_closest_marker("integration"):
        return                       # ← 标了 integration 的测试：放行真连
    import pymysql                    #   其余所有测试：照旧焊死
    ...
```

只多了 `request` 参数 + 两行判断。配套还有三件小事:
1. **`pyproject.toml` 注册 marker**(否则 pytest 报 unknown marker);
2. **integration 测试默认 skip**:文件里检查环境变量,没设 `CHURN_TEST_MYSQL_URL` 就 `pytest.skip(...)`——所以本地 `pytest -q` 默认**不碰库**,547 全确定性基线不变;
3. **只有显式 opt-in 才真跑**:设了那个 env(或 CI 里配了 MySQL service container)+ `pytest -m integration`。

**为什么安全**:默认 547 个测试**没有一个**标 integration,守卫对它们 100% 照旧;integration 默认 skip。"刻意、可控地连测试库"和"意外连到生产库"是两回事,这个口子不破坏契约。

## 为什么现在该重新考虑做它

上次我推荐"**就此收尾、不做**",**核心理由就一条:"当前项目无 CI,真库 integration 只能默认 skip、基本永远不跑"**——写了等于死代码。

**但现在你的 CI 已经有了,这条理由直接失效。** GitHub Actions 里加一个带 MySQL 的 job 是标准操作:

```yaml
services:
  mysql:
    image: mysql:8
    env: { MYSQL_ROOT_PASSWORD: root, MYSQL_DATABASE: churn_test }
    options: >-
      --health-cmd="mysqladmin ping -proot" --health-interval=5s
      --health-timeout=3s --health-retries=10
    ports: ["3306:3306"]
# step: CHURN_TEST_MYSQL_URL=mysql+pymysql://root:root@127.0.0.1:3306/churn_test pytest -m integration
```

这样 integration 测试**每次 PR 都真在 MySQL 上跑**,真正补上"`ON DUPLICATE KEY UPDATE` 插入→再 upsert 不主键冲突"这个我现在只用方言断言间接覆盖的执行语义。它不再是"永远 skip 的摆设"。

所以结论变了:**有了 CI,这个 integration 套件从"不值得"变成了"值得"。**

要不要我现在就动手把它做全?一笔做完是这几样:
1. `conftest` 加 integration 豁免(上面那两行);
2. `pyproject` 注册 marker;
3. 新增 `tests/churn/test_mysql_repo_integration.py`(默认 skip,真库时验证 upsert 执行语义);
4. `ci.yml` 加 MySQL service 的 job 跑 `-m integration`。

仍是干净的 churn-only 改动,可单独 commit。你说做我就做。

---

### Turn 6  *(2026-06-24 01:24)*

**User:**

现在main主分支上commit吧

---

### Turn 7  *(2026-06-24 01:24)*

**User:**

先在main主分支上commit吧

**Assistant:**

在 main 上 commit。我先确认当前分支和完整改动范围(确保只 commit 这次任务的东西、且确实是干净的 churn-only 变更),再落 commit。

确认干净:分支 `main`、改动全是这次任务的(2 新测试 + 8 处基线 538→547 同步,含 ci.yml 注释),`ci.yml` 的 diff 纯注释、无逻辑改动。是一笔 churn-only 变更,符合铁律2。落 commit:

Commit 落定。

**`9104e26`** 已在 `main`,工作区干净:
- 10 files changed(+283/−15):2 个新测试文件 + 8 处基线 538→547 同步
- 一笔 churn-only 变更,可单独回滚/bisect(铁律2)

两点收尾:

1. **没有 push**(你只说 commit)。要推的话告我——`git push` 到 main 会触发你那条 `on: push: branches:[main]` 的 CI 实跑一遍(ruff/mypy/pytest 547/pip-audit),正好当一次真实门禁验证。

2. **还悬着一件**:你刚问的真库 integration 套件(conftest 豁免 + MySQL service job)。**现在 CI 有了、它从"不值得"变"值得"**,但这笔 commit 没含它——这次只提交了确定性测试部分。要做的话我接着开第二笔 churn commit。

要我现在 push,还是先把 integration 套件做了一起推?

---

### Turn 8  *(2026-06-24 01:27)*

**User:**

 integration是用来做什么的

**Assistant:**

一句话:**integration 测试就是把你的代码真的连到一个真实的 MySQL 上、跑真实的 SQL,验证"代码 + SQL + 数据库 schema + MySQL 引擎行为"这一整条链端到端是对的。** 这一层是我已经写的确定性测试**天然够不到**的。

## 它和我已写的测试,分工不一样

我那个 `test_mysql_repo_dialect.py` 做的事是:把 `save_detection_run` 生成的语句**编译成一段 SQL 字符串**,然后检查这段字符串长得对不对(有没有 `ON DUPLICATE KEY UPDATE`、覆盖了哪些列)。**它从头到尾没把这条 SQL 发给任何数据库。**

| | 确定性方言断言(已写) | 真库 integration(没写) |
|---|---|---|
| 验证什么 | SQL 字符串**生成得对不对** | SQL 在真 MySQL 上**跑得通、语义对不对** |
| 连数据库吗 | 不连 | 真连一个 MySQL |
| 能抓 | 漏列、方言退化、分支选错 | 代码与真实 schema 不一致、约束缺失、引擎行为 |

## 举两个"只有 integration 能抓"的 bug

**例一:有人给 `churn_detection_runs` 加了个新列 `notes`,在代码 INSERT 里加了 `notes=...`,但忘了写迁移脚本给真实表加这一列。**
- 确定性测试:✅ **通过**——它只看代码生成的字符串,字符串里有 `notes`,断言就过了,它根本不知道线上那张表没有这列。
- 真库 integration:❌ **当场失败**——真的 INSERT 进 MySQL 时报 `Unknown column 'notes'`。

**例二:`ON DUPLICATE KEY UPDATE` 要生效,表上 `run_id` 必须真的是主键。如果建表迁移漏设了主键约束——**
- upsert 永远不会"更新",同一个 `run_id` 第二次会插成**重复行**,检测记录就脏了。
- 确定性测试:✅ 通过(SQL 字符串没毛病)。
- 真库 integration:❌ 抓得到(第二次 save 后一查,发现是两行不是一行)。

`save_detection_run` 这整个 `ON DUPLICATE KEY UPDATE` 分支的**全部意义**就是"插入或更新、不要主键冲突"——而这个语义**只有真 MySQL 能验证**。同理还有 utf8mb4 中文、JSON 列、`read_text_column` 的 MEDIUMTEXT 分块读、`GET_LOCK` 命名锁——都是 SQLite/编译断言碰不到的 MySQL 专属行为。

## 放到你这个项目的语境

`CLAUDE.md` 铁律2 自己写了:churn 真实集成路径"只有 fake 测试覆盖,**pytest 全绿 ≠ 线上安全**,故改动须经 **~50 业务员灰度**验证"。

也就是说——**现在这条链唯一的安全网,是上线后让 50 个业务员人肉试、出问题再报**。integration 测试的作用,就是**把这张"人肉灰度网"部分前移成 CI 里的自动检查**:在合并前就发现"这条 SQL 在真 MySQL 上挂了 / upsert 语义错了",而不是等业务员上线踩。

## 诚实的边界

它不是免费的,也不是万能的:
- **成本**:CI 里要跑一个 MySQL service container、测试默认 skip(本地 `pytest -q` 不连库)、多一套要维护的用例。
- **局限**:测试库 ≠ 生产库的真实数据;而且如果测试库**也用同一套迁移脚本建表**(应该这么做),那"迁移脚本本身写错"它能抓,但"测试库和生产库迁移不同步"它抓不到。它验证的是"代码 ↔ 迁移脚本"一致,不是"代码 ↔ 生产现状"一致。

---

所以回到你的选择:**有了 CI,它才第一次具备"每次 PR 自动真跑"的条件**(之前无 CI 只能永远 skip,所以我上次说不值得)。要不要做,取决于你觉得"churn 改动前自动验证 SQL 真能在 MySQL 上跑"这件事,值不值上面那点 CI 维护成本。要做我就把四件套(conftest 豁免 + marker + integration 用例 + CI 的 MySQL job)开成第二笔 churn commit;不做也完全合理——继续靠方言断言 + 50 人灰度兜着。

---

### Turn 9  *(2026-06-24 01:33)*

**User:**

暂时不做

**Assistant:**

明白,**integration 暂不做**——继续靠方言断言 + 50 人灰度兜底,完全合理的取舍。

我顺手把这个决策更新进 memory(免得下次会话我又因为"CI 有了"重新来劝你做一遍)。先读一下之前那条:

记好了——memory 已更新:**评审"无 CI"是 HEAD 错位误判已更正,且明确标注"CI 已具备条件但用户权衡后仍暂不做 integration,别再重提"**,免得以后反复劝你。

那这次就到这。当前状态:

- **commit `9104e26`** 在本地 `main`(确定性测试 9 个 + 基线 538→547 同步),工作区干净
- integration:**不做**,继续方言断言 + 50 人灰度兜底
- **还没 push**

就剩一个动作没做:**要不要 `git push`?** 推上去会触发你那条 `on: push:[main]` 的 CI 实跑一遍(ruff/mypy/pytest 547/pip-audit),算一次真实门禁验证。要推你说一声,我就执行;不推就先这样搁着。

---


# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `800be2d1-975f-4da9-85c1-d67e695af966`
- **范围**: Turn 1
- **导出时间**: 2026-06-24 09:48:33

---

### Turn 1  *(2026-06-24 01:46)*

**User:**

mysql upsert 方言断言是什么意思

**Assistant:**

要点都在你那条 commit（9104e26）和测试文件 `tests/churn/test_mysql_repo_dialect.py` 里。拆开"mysql upsert 方言断言"这几个字：

## 逐词拆

- **upsert** = `INSERT ... ON DUPLICATE KEY UPDATE`。`save_detection_run` 写检测run时，run_id 已存在就更新、不存在就插入——这是一条语句完成"插或更"。
- **方言（dialect）** = SQLAlchemy 对不同数据库 SQL 方言的抽象。同样的 upsert，MySQL 编译成 `ON DUPLICATE KEY UPDATE`，Postgres 是 `ON CONFLICT DO UPDATE`,SQLite 根本不走这条。
- **断言** = 测试里的 `assert`。

合起来：**用 MySQL 方言把代码实际要执行的那条语句编译成 SQL 字符串，然后断言它确实是 MySQL 专属的 upsert，且形状正确——全程不连真库。**

## 为什么要单独补这个测试

关键在 `test_mysql_repo_dialect.py:3-5` 的注释：现有 `test_mysql_repo.py` 跑的是 SQLite 内存库，而 `save_detection_run` 里 **MySQL 才走 upsert，SQLite 走 `Session.merge()` 旁路**。也就是说 upsert 那条 MySQL 专属代码路径，在原来的 SQLite 测试里**一次都没被执行过**——pytest 全绿但那段 SQL 没人验证。这正好踩中铁律#2 说的"churn 真实集成路径只有 fake 覆盖"。

## 这两条断言具体在断什么

核心三步在 `test_mysql_repo_dialect.py:88-104`：

1. 造一个 `dialect.name == "mysql"` 的 `_FakeEngine`，逼 `save_detection_run` 选中 MySQL 分支;
2. 捕获它实际要执行的语句对象，`compile(dialect=mysql.dialect())` 编成 SQL 串;
3. 对这串 SQL 下断言。

抓的是三类最现实的回归：
- **方言退化**——必须出现 `ON DUPLICATE KEY UPDATE`，不能悄悄退成普通 `INSERT`（`:101`）;
- **加列漏同步**——`_MUTABLE_COLUMNS` 那 9 个可变列每个都得出现在更新子句里，防止以后给表加字段却忘了同步进 `on_duplicate_key_update`（`:102-103`）;
- **主键误入**——`RUN_ID` 不能出现在更新子句（`:104`，主键不该被更新）。

外加一条对偶守卫 `:107-140`：非 MySQL 方言（SQLite）**绝不能误走** upsert 分支，否则会对 SQLite 发它不认的方言。它给的 fake engine 故意不提供 `begin()`，一旦误入 MySQL 分支就 `AttributeError` 把回归炸出来。

## 它"不"保证什么

这是**确定性测试**——只验证"代码生成的 SQL 长得对"，不验证"这条 SQL 在真 MySQL 上插入→再 upsert 真的不冲突"。后者的执行语义由 MySQL 自己保证，需要真库 integration（CI 配 MySQL service container），commit 里也写了那块待 CI 就绪再补。你的记忆 [[churn-test-strategy-deterministic]] 里也记了这个权衡:暂不做真库 integration。

简单说:**这是"读编译出来的 SQL 字符串来体检",不是"连库跑一遍"——便宜、零依赖、能卡住绝大多数改坏 upsert 的改动。**

---
