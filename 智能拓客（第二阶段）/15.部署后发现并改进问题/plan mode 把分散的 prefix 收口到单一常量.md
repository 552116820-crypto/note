# 收口 lead HTTP 前缀到单一常量(彻底版)

## Context(为什么做)

`/api/v1/leads`(lead HTTP 命名空间)是**单一事实**,但当前被独立硬编码了 **3 份 prefix + 5 份全路径**,散在 4 个文件:

- `app/main.py:51` `LEAD_PREFIX = "/api/v1/leads"` —— 喂给体积限制中间件(165)+ 两个按路径门控的异常处理器(214 校验、259 兜底)
    
- `app/lead/routers/leads.py:15` 与 `customer_detail.py:15` —— 各自 `APIRouter(prefix="/api/v1/leads")`,**根本不看 `LEAD_PREFIX`**
    
- `app/lead/middleware/api_log.py:27-34` —— 4 条全路径(`_LOGGED`)+ 1 条(`_RESULT_TOTAL_PATHS`)
    

问题不是"重复不好看",而是 **`LEAD_PREFIX` 是个假 SSOT**:它有名字、有注释、门控着中间件,任何人都会当它是前缀的权威来源——但真正注册路由的两个 APIRouter 各抄各的字面量。一旦未来改前缀(如上 `/api/v2/leads`)只改了其中一边:**路由照常工作、healthz 绿、冒烟过,但按前缀门控的行为静默失配**——请求体上限(413/code=1002)失效、参数校验从 200-envelope 退回裸 422(**违反铁律 #6 契约**)、兜底异常 envelope 回退。全程不抛错,而三件套门禁(ruff/mypy/pytest)都看不见这个"3 个字面量应该相等"的不变量。

收口把"靠人记得 3 个地方"换成"编译器维持的单一符号",并以最低成本拆掉 v2 迁移风险(**无需**建版本化框架)。

## 最优方案

**常量归宿:新建 `app/lead/constants.py`**,只放一行纯常量、不 import 任何东西:

# lead HTTP 命名空间前缀(无尾斜杠);app 运行时路由/中间件/异常门控的单一来源。  
# 注:scripts/lead/{http_smoke,load_test}.py 另有独立字面量副本(外部测试工具、不在运行时 import 图内),  
#     改前缀时需手动同步这两个脚本(本次不收口它们)。  
LEAD_PREFIX = "/api/v1/leads"

归宿决策(已由 Explore 验证):

- **不放** `config/settings.py`:它是 env 驱动的 `Settings(BaseModel)`,语义是"可配置项";前缀是契约、不该 env 可配,且 import 它带 `load_env()`+`Settings()` 副作用,而两个 router 当前**不依赖** settings,放进去平白新增依赖边。
    
- **不放** `config/constants.py`:`config/__init__.py` import 了 settings,经 config 包导入会被副作用连坐。
    
- **`app/lead/constants.py` 自身零 import** → 数学上不可能成环、零副作用,routers/middleware/main 均可自由 import(`app/lead/__init__` 仅 docstring,`app/lead` 不反向 import `app.main`)。
    
- **复用既有名 `LEAD_PREFIX`**(不另起 `LEAD_API_PREFIX`):main.py 的 165/209/214/259 沿用同名,一字不改,全仓单一名。
    

**5 处引用全部指向该常量:**

1. **新建** `app/lead/constants.py` —— 定义 `LEAD_PREFIX`。
    
2. `app/main.py:51` —— 删本地 def,改为 `from app.lead.constants import LEAD_PREFIX`;165/209/214/259 不动。
    
3. `app/lead/routers/leads.py:15` —— `APIRouter(prefix=LEAD_PREFIX, tags=["leads"])` + 顶部 import。
    
4. `app/lead/routers/customer_detail.py:15` —— 同上(`tags=["customer-detail"]`)。
    
5. `app/lead/middleware/api_log.py:21,27-34` —— 顶部 import,全路径改 f-string 派生(两处均为精确成员判断,逐字节一致):
    
    _LOGGED = {("POST", f"{LEAD_PREFIX}/score-and-info"), ("POST", f"{LEAD_PREFIX}/skip"),  
               ("POST", f"{LEAD_PREFIX}/timeline"), ("POST", f"{LEAD_PREFIX}/recommend-reason")}  
    _RESULT_TOTAL_PATHS = {f"{LEAD_PREFIX}/score-and-info"}
    

**关键不变量:** 常量值恰为 `"/api/v1/leads"`(无尾斜杠),同时满足 main.py 的 `startswith`、router prefix 拼接、api_log 的 f-string 拼接三处逐字节一致。

> 注:三个文件的**模块 docstring / 注释**里仍会出现 `/api/v1/leads/...` 字样——那是文档说明、不影响行为,保留即可,不属功能性重复。

## 触碰文件清单

|文件|改动|
|---|---|
|`app/lead/constants.py`|**新建**,1 行常量|
|`app/main.py`|24 行区加 import、删 51 行本地 def(165/209/214/259 不动)|
|`app/lead/routers/leads.py`|15 行 prefix 引用常量 + import|
|`app/lead/routers/customer_detail.py`|15 行同上|
|`app/lead/middleware/api_log.py`|21 行 import + 27-34 行 f-string 派生|
|**测试**|**0 改**(无任何测试 import 这些常量,全用字面量,值不变)|
|`scripts/lead/`|**本次不改**:`http_smoke.py`/`load_test.py` 有独立字面量副本(外部工具、不在运行时 import 图),仅在常量 docstring 标注其存在|

## 验证(端到端)

1. `python3 -m ruff format app && python3 -m ruff check --fix app` —— 格式/lint 全绿
    
2. `python3 -m mypy` —— 类型全绿(新模块纳入 app/ 覆盖)
    
3. `python3 -m pytest -q` —— **559 基线全绿、0 测试改动**(逐字节零行为变化的硬证据)
    
4. `grep -rn '"/api/v1/leads"' app/` —— 可执行字面量应**只**剩 `app/lead/constants.py` 一处(routers/api_log/main 的全消失;docstring/注释不算)
    
5. (可选)`docker compose up -d --build && make smoke` —— 全栈冒烟确认 lead 接口 / envelope / 体积限制行为不变
    

> 本 plan 已经 Codex 独立读码审查:循环 import 安全、逐字节行为不变、测试零改、归宿最优、命名合理——核心断言全部成立、无必须修正的硬伤。唯一补充即上方的 `scripts/` 副本边界。

## 预期效果

- **行为:逐字节不变。** 运行中的 app 对所有请求产出与改动前完全相同的响应;559 测试零改动全过即为证。
    
- **结构:`/api/v1/leads` 在 app 运行时唯一可执行定义**落在 `app/lead/constants.py`;路由注册、体积限制门控、envelope 作用域、api_log 白名单全部引用同一符号(`scripts/` 下的副本属外部工具、不在运行时 import 图,见上方边界说明)。
    
- **假 SSOT 陷阱消除:** 改前缀只需动一处常量,所有消费方自动跟随;删常量 → ImportError(编译期暴露,不再是静默漂移)。
    
- **v2 迁移去风险(原 P4):** 未来上 `/api/v2/leads` 时,app 运行时的路由 + 体积限制 + envelope 门控 + api_log 路径随单一常量一起迁移,不再有"路由到了 v2、契约门控还停在 v1"的静默契约破损——而这一切**无需**引入版本化框架。(`scripts/lead/` 两个测试工具的副本仍需手动同步。)
    

## 会用到的能力

**扫描范围(本次会话可见):**

- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
    
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily
    
- Agent types (7): claude, claude-code-guide, codex:codex-rescue, Explore, general-purpose, Plan, statusline-setup
    

**会用:**

- Agent Explore:已用于查清常量归宿 / 导入图成环风险 / api_log 路径结构 / 测试引用(Phase 1 已完成)
    

**考虑过但不用:**

- Agent Plan:设计已在 Explore 阶段定死(归宿、派生、命名均无剩余权衡),属单符号提取类改动,再派 Plan agent 冗余
    
- Skill code-review / simplify:5 文件机械提取、输出字面恒等、测试零改,diff 一眼可核,不必上评审编排(要复查直接读 diff)
    
- Skill security-review:不碰鉴权与契约值、envelope 行为逐字节不变,无安全面变化