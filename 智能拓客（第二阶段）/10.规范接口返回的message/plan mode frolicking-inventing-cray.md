# 计划：成功响应 `message` 从 `"ok"` 改为空串 `""`

## Context（为什么改）

前端对接时反馈：成功响应里固定的 `"message": "ok"` 不要。原因——`message` 在前后端约定里是**给用户看的提示文案**（失败时做 toast/弹窗），而"成功与否"前端只看 `code === 0`（文档 §2 已明确）。成功时的 `"ok"`：判定上没用、显示上有害（英文魔法值，前端若统一渲染 message 会把 "ok" 漏给用户，还得专门写代码过滤）。

改成空串后契约变为：**成功 `message=""`（无内容可展示），失败 `message` 仍是可读原因**。这是 §2 定义的**全局约定**，五个接口的成功响应全部经同一个 `envelope()` 产生，所以代码改动集中、文档与测试需同步。

> **Codex 复审结论**：核心 2 行改动、唯一会挂的测试、文档 9 处行号均**确认无误**；复审纠正了 3 处（schemas 改动的定性、rationale 笔记路径、同文档内需避开的非 message "ok" 行），已并入下方。

---

## 改动一：后端代码（2 行，单一来源）

所有成功响应都走 `envelope()` 的默认 `message`，改默认值即全量生效：

1. **`app/errors.py:5`** — `def envelope(data=None, code=0, message: str = "ok")` → `message: str = ""`
   - **运行时唯一来源**。5 个接口（score-and-info / skip / industries / timeline / recommend-reason）的成功响应全部继承此默认。`app/main.py` 里的错误处理都显式传 `message=`，不受影响。
2. **`app/schemas/response.py:60`** — `ApiEnvelope.message: str = "ok"` → `""`（**可选 / 仅一致性**）
   - Codex 复审纠正：`ApiEnvelope` 当前**未接任何 `response_model`**（`app/routers/leads.py:17` 注释 "response_model is intentionally NOT enforced"），改它**既不影响运行时、也不影响 OpenAPI**。纯粹让这个"规范 envelope 模型"不与真实响应漂移（日后若真接上 `response_model`，示例不会又冒出 "ok"）。可改可不改，建议顺手改。

**严禁误改**（含 "ok" 但与我方 envelope 无关）：
- `app/clients/crm.py:32,91`（`_OK_STRS`、`("success","ok")`）= 校验**上游 CRM** 返回的 message，是外部系统契约。
- `/healthz` 的 `status:"ok"`、timeline 的 `sources:{orders:"ok"}` = 业务数据字段。

---

## 改动二：测试（1 处必改 + 3 处可选）

- **必改** `tests/test_api_boundary.py:143` — 全等断言 `r.json() == {"code": 0, "message": "ok", "data": None}` → 把 `"message"` 改成 `""`。这是唯一会因改动失败的测试；改完它即作为**契约回归守卫**（锁定 /skip 成功 message=""）。
- **可选（一致性）** `tests/test_api_log_helpers.py:58,71,88` — fixture 里的 `"message": "ok"` 是模拟响应、断言不读 message（中间件只读 `code`/`data.pagination.total`，见 `app/middleware/api_log.py:97,111,134`），不会失败；为让测试数据贴合真实响应可一并改成 `""`。
- **严禁误改** `tests/test_crm_client.py:71` — 测的是上游 CRM 校验函数 `_check_state`，非我方 envelope。

---

## 改动三：文档同步（仅 1 个文件，9 处）

**`/mnt/d/note/智能拓客（第二阶段）/6.接口测试与优化/接口文档，对接前端.md`**：

- **7 处 JSON 示例** `"message": "ok"` → `"message": ""`：行 **22、156、227、255、326、341、410**。
  其中**行 326 顺手删掉行尾注释 `// message 不要ok`**（改完注释失去意义）。
- **行 45** 错误码表 `| `0` | 200 | ok | 成功 | …` 的 message 列 → 改为 `""`（建议写 `""` 而非留空白格，避免表格歧义）。
- **行 28** 字段说明 `提示信息（成功为 `"ok"`，失败为可读原因）` → `提示信息（成功为空串 `""`，失败为可读原因）`。
- 顺手把头部 `更新日期：2026-06-06` 更到 `2026-06-10`，可加一行变更说明（成功 `message` 由 `"ok"` 改空串）。

**同一文件内严防误改**（Codex 标注，这些 "ok" 不是 envelope message）：
- 行 **333、343** —— timeline `sources` 的数据值 `"ok"`/`"error"`，保留。
- 行 **440** —— `/healthz` 的 `status:"ok"`，保留。

**其它文档保持原样**：
- `/mnt/d/note/智能拓客（第二阶段）/接口返回的message字段.md`（**注意在上级目录**，不是 `6.接口测试与优化/` 下——原计划路径写错，已更正）= 本次需求来龙去脉的记录（解释"为什么不要 ok"、含前后对比示例）；机械替换会破坏其说明含义，留作存档不动。
- `6.接口测试与优化/接口：加行业筛选.md` 经确认无 envelope message 示例，无需改。

---

## 验证（全程离线，无需库/Redis——envelope 是纯函数）

1. `cd /mnt/e/smart_lead_generation && python3 -m pytest tests/test_api_boundary.py -q` —— 确认改后的全等断言通过（`message==""`）。
2. `python3 -m pytest tests/ -q` —— 跑全量单测（约 266 个），确认无回归；若别处还有未发现的成功 message 断言，这一步会暴露。
3. `grep -rn '"message": *"ok"\|message: str = "ok"' app/` —— 改完应**无命中**（envelope 已全量迁移）。
4. `grep -rn '"ok"' app/clients/crm.py` —— 仍应看到 `_OK_STRS` / `("success","ok")`，这是上游 CRM 校验，**预期保留**（确认没误删）。

---

## 会用到的能力

**扫描范围**（本次会话可见）：
- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily

**会用**：
- Agent Explore：规划阶段已用 3 个并行 Explore 定位 envelope 源头、受影响测试、文档出现点（已完成）。
- 执行阶段仅内置 Read/Edit/Bash：改 2 行代码 + 1 处测试 + 文档；`python3 -m pytest` 离线验证。

**考虑过但不用**：
- Skill `verify`：本可起服务打真实接口看 `message=""`；但 `test_api_boundary` 用 TestClient 已能离线锁定该契约，且起服务要先修业务库隧道 MTU/连库，成本高 → 用 pytest 不用 verify。
- Skill `code-review` / `simplify`：改动仅 2 行默认值 + 同步文档/测试，无需额外质量审查。
