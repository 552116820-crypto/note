# Churn Warning Engine 代码审查与修复记录 (Codex)

> 记录时间: 2026-04-10  
> 项目路径: `/mnt/e/churn-warning-engine`  
> 用户指定保存路径: `E:\note\客户流失预警\review by codex 4.md`  
> 主要参考文件: `E:\note\客户流失预警\review by claude 6.md`

---

## 一、用户提供的仓库说明

用户在会话开始时提供了仓库级说明，核心约束如下：

- `app/` 存放运行时代码。
- `app/engine.py` 负责异常检测逻辑。
- `app/service.py` 负责任务编排。
- `app/repository.py` 提供内存仓储。
- `app/db.py` 提供可选 MySQL 仓储。
- `app/main.py` 是 FastAPI 入口。
- `app/tasks.py` 定义 Celery 定时任务。
- `app/config.py` 统一管理环境变量驱动的配置。
- `tests/` 存放 pytest 测试。
- `churn_warning_engine.egg-info/` 和 `__pycache__/` 属于生成产物，不应作为源码修改。

常用命令：

```bash
pip install -e .[dev]
pip install -e .[all]
uvicorn app.main:app --reload
pytest -q
celery -A app.tasks worker --loglevel=info --beat
```

编码和测试约定：

- Python 3.10+。
- 遵循 PEP 8，4 空格缩进。
- 模块、函数、变量使用 `snake_case`。
- 类名使用 `PascalCase`。
- 环境变量使用 `UPPER_SNAKE_CASE`。
- 新增逻辑沿用 service/repository 分层。
- 测试使用 `pytest`。
- 测试文件命名为 `tests/test_*.py`。

安全配置提醒：

- `DATABASE_URL`、`CELERY_BROKER_URL`、`CELERY_RESULT_BACKEND` 等敏感信息不得提交。
- 只有安装可选 MySQL 依赖且数据库表结构可用时，才启用 `REPOSITORY_TYPE=mysql`。

---

## 二、用户第一次请求

用户请求：

> `E:\note\客户流失预警\review by claude 6.md` 这 claude 审查代码的结果，你看一下

处理方式：

1. 读取 Claude 审查文件。
2. 对照当前代码确认每条问题是否真实存在。
3. 运行测试确认当前基线。
4. 给出优先级判断。

读取到的 Claude 审查要点：

- 项目整体质量较高，分层清晰。
- 测试结果为 `56 passed`。
- 建议补 `.gitignore`。
- 建议移除 `churn_warning_engine.egg-info/`。
- 建议为 engine/service 层添加日志。
- 建议 Celery beat 加防重叠机制。
- 建议 `batch_id` 不依赖普通冒号分割。
- 建议说明 `detect_batch` 里的 `anomaly_count` 语义。
- 建议补充一批边界测试。
- 建议给 MySQL 锁名中的 SHA1 加注释。

我对照代码后的判断：

### 优先级较高、值得立即修复

1. `.gitignore` 不完整  
   当前 `.gitignore` 只包含：

   ```gitignore
   /.claude
   /.cursor
   .pytest_cache/
   __pycache__/
   *.py[cod]
   ```

   缺少 `.env`、`*.egg-info/`、`build/`、`dist/`。

2. `churn_warning_engine.egg-info/` 已经被 git 跟踪  
   通过 `git ls-files churn_warning_engine.egg-info .gitignore explain.md` 确认，`egg-info` 下 5 个文件已被跟踪：

   ```text
   churn_warning_engine.egg-info/PKG-INFO
   churn_warning_engine.egg-info/SOURCES.txt
   churn_warning_engine.egg-info/dependency_links.txt
   churn_warning_engine.egg-info/requires.txt
   churn_warning_engine.egg-info/top_level.txt
   ```

3. `anomaly_count` 语义容易误导  
   `app/service.py` 中 `anomaly_count` 是检测出的原始异常数，而 `warning_ids` 是保存或复用后的预警 ID。重跑时可能 `anomaly_count > 0`，但不会新增预警。

4. MySQL 写入缺少显式 rollback  
   当前依赖 session 上下文清理。不是明显 bug，但生产环境可考虑加固。

5. Celery 定时任务重叠是生产风险  
   当前 `app/tasks.py` 没有任务级锁。去重能减少重复预警，但不能避免重复扫描、重复 job、数据库压力。

### 低优先级或不必急着改

- `_prediction_confidence([])` 空列表分支基本是防御代码，不是 bug。
- `predict_cycle` 中 `days > 0` 过滤因为前面 `sorted(set(...))` 已经去重，确实冗余，但无害。
- `InMemoryRepository.get_job` 的 `KeyError` 信息不如 MySQL 友好，但 API 层已经转成 404。
- `batch_id.split(":")` 当前因为 job_id 是 UUID，不含冒号，所以不是当前 bug；但改成 `rsplit(":", 1)` 成本很低。
- SHA1 用于 MySQL 锁名不是安全用途，可以接受，建议加注释。
- `OrderedDict` 可用普通 dict 替代，但不是问题。

基线测试结果：

```text
56 passed in 0.43s
```

我给用户的建议：

先做一批小而确定的修复：

- 补 `.gitignore`。
- 移除已跟踪的 `egg-info`。
- 补边界测试。
- 把 `batch_id` 改成 `rsplit`。
- 给 SHA1 锁名加注释。

日志、分页、Celery 防重叠可以作为第二批，因为涉及接口行为或运维策略。

---

## 三、用户第二次请求

用户请求：

> 修复补 .gitignore、移除已跟踪的 egg-info、补上述边界测试、把 batch_id 改成 rsplit、给 SHA1 锁名加注释

执行的改动如下。

### 1. 补全 `.gitignore`

文件：`.gitignore`

新增内容：

```gitignore
.env
*.egg-info/
build/
dist/
```

最终相关内容：

```gitignore
/.claude
/.cursor
.env
.pytest_cache/
__pycache__/
*.py[cod]
*.egg-info/
build/
dist/
```

### 2. 移除已跟踪的 `egg-info`

执行：

```bash
git rm -r churn_warning_engine.egg-info
```

输出：

```text
rm 'churn_warning_engine.egg-info/PKG-INFO'
rm 'churn_warning_engine.egg-info/SOURCES.txt'
rm 'churn_warning_engine.egg-info/dependency_links.txt'
rm 'churn_warning_engine.egg-info/requires.txt'
rm 'churn_warning_engine.egg-info/top_level.txt'
```

之后通过命令确认：

```bash
git ls-files churn_warning_engine.egg-info
```

结果无输出，表示已不再被 git 跟踪。

### 3. 将 `batch_id` 解析改成 `rsplit`

文件：`app/service.py`

修改前：

```python
parts = batch_id.split(":")
```

修改后：

```python
parts = batch_id.rsplit(":", 1)
```

目的：

- 只从右侧切一次。
- 当前 UUID job_id 不含冒号，但这样对未来格式变化更稳健。

### 4. 给 MySQL 锁名中的 SHA1 加注释

文件：`app/db.py`

新增注释：

```python
# Non-cryptographic use: shorten an arbitrary lock identity for MySQL.
return f"warning:{hashlib.sha1(raw.encode('utf-8')).hexdigest()}"
```

目的：

- 明确 SHA1 不是用于密码学安全场景。
- 这里只是将任意锁身份缩短为 MySQL lock name。

### 5. 补测试：`_severity_from_ratio` 边界

文件：`tests/test_engine.py`

新增测试：

```python
class TestSeverity:
    def test_ratio_boundary_medium(self):
        assert _severity_from_ratio(1.49) == "low"
        assert _severity_from_ratio(1.5) == "medium"

    def test_ratio_boundary_high(self):
        assert _severity_from_ratio(1.99) == "medium"
        assert _severity_from_ratio(2.0) == "high"
```

覆盖：

- `ratio == 1.5` 时为 `medium`。
- `ratio == 2.0` 时为 `high`。

### 6. 补测试：多个关键产品同时触发

文件：`tests/test_engine.py`

新增测试：

```python
def test_multiple_key_products_can_trigger(self):
    s = Settings(product_multiplier=1.4, key_product_threshold=0.3)
    c = Customer(3, "多产品客户", 8003, "销售丙")
    orders = [
        _order(
            1, 3, 8003, date(2025, 10, 1), 30000,
            [("P1", "关键产品1", 15000), ("P2", "关键产品2", 15000)],
        ),
        _order(
            2, 3, 8003, date(2025, 10, 29), 30000,
            [("P1", "关键产品1", 15000), ("P2", "关键产品2", 15000)],
        ),
        _order(
            3, 3, 8003, date(2025, 11, 26), 30000,
            [("P1", "关键产品1", 15000), ("P2", "关键产品2", 15000)],
        ),
        _order(
            4, 3, 8003, date(2025, 12, 24), 30000,
            [("P1", "关键产品1", 15000), ("P2", "关键产品2", 15000)],
        ),
    ]

    warnings = detect_product_anomaly(c, orders, date(2026, 4, 7), s)

    product_ids = {w.feature_summary["product_id"] for w in warnings}
    assert product_ids == {"P1", "P2"}
```

覆盖：

- 同一个客户多个关键产品都满足阈值和偏离条件时，应返回多条产品预警。

### 7. 补测试：配置边界

文件：`tests/test_config.py`

新增测试：

```python
def test_settings_accepts_minimum_order_count():
    assert Settings(min_order_count=2).min_order_count == 2


def test_settings_rejects_invalid_volume_rise_threshold():
    with pytest.raises(ValueError, match="volume_rise_threshold"):
        Settings(volume_rise_threshold=0)
```

覆盖：

- `min_order_count=2` 是最小合法值。
- `volume_rise_threshold=0` 非法。

### 8. 补测试：warning key

新增文件：`tests/test_warning_keys.py`

内容：

```python
from __future__ import annotations

from app.warning_keys import (
    ACTIVE_WARNING_SLOT,
    ENRICHED_STATUS,
    MAX_DEDUP_KEY_LENGTH,
    PENDING_ENRICH_STATUS,
    RESOLVED_WARNING_STATUS,
    active_slot_for_status,
    make_warning_dedup_key,
)


def test_long_dedup_key_is_shortened_to_storage_limit():
    product_id = "P-" + ("X" * 200)

    key = make_warning_dedup_key("product", product_id=product_id)

    assert len(key) == MAX_DEDUP_KEY_LENGTH
    assert key.startswith("product:P-")


def test_active_slot_for_status():
    assert active_slot_for_status(PENDING_ENRICH_STATUS) == ACTIVE_WARNING_SLOT
    assert active_slot_for_status(ENRICHED_STATUS) == ACTIVE_WARNING_SLOT
    assert active_slot_for_status(RESOLVED_WARNING_STATUS) is None
```

覆盖：

- 超长 dedup key 会被压缩到 MySQL 字段长度限制。
- `pending_enrich` 和 `enriched` 是活跃状态。
- `resolved` 不占用活跃去重槽。

---

## 四、测试结果

修复后第一次测试：

```text
63 passed in 0.51s
```

整理测试代码格式后再次测试：

```text
63 passed in 0.40s
```

最终测试：

```text
63 passed in 0.39s
```

测试数量变化：

- 修复前：`56 passed`
- 修复后：`63 passed`
- 新增测试数：7

---

## 五、最终工作区状态

当时 `git status --short` 显示：

```text
 M .gitignore
 M app/db.py
 M app/service.py
D  churn_warning_engine.egg-info/PKG-INFO
D  churn_warning_engine.egg-info/SOURCES.txt
D  churn_warning_engine.egg-info/dependency_links.txt
D  churn_warning_engine.egg-info/requires.txt
D  churn_warning_engine.egg-info/top_level.txt
 M tests/test_config.py
 M tests/test_engine.py
?? tests/test_warning_keys.py
```

`git diff --stat HEAD` 显示：

```text
 .gitignore                                         |  4 +++
 app/db.py                                          |  1 +
 app/service.py                                     |  2 +-
 churn_warning_engine.egg-info/PKG-INFO             | 18 -----------
 churn_warning_engine.egg-info/SOURCES.txt          | 17 ----------
 churn_warning_engine.egg-info/dependency_links.txt |  1 -
 churn_warning_engine.egg-info/requires.txt         | 17 ----------
 churn_warning_engine.egg-info/top_level.txt        |  1 -
 tests/test_config.py                               |  9 ++++++
 tests/test_engine.py                               | 37 ++++++++++++++++++++++
 10 files changed, 52 insertions(+), 55 deletions(-)
```

说明：

- `tests/test_warning_keys.py` 是新增未跟踪文件，所以没有出现在普通 `git diff --stat HEAD` 里。
- `churn_warning_engine.egg-info` 删除已通过 `git rm` 加入索引。

---

## 六、用户第三次请求：LF/CRLF 提示是什么意思

用户问题：

> LF will be replaced by CRLF the next time Git touches it 这是什么意思

解释：

这表示文件当前换行符是 LF，也就是 Unix/Linux 风格的 `\n`；但 Git 配置认为工作区文件应该使用 CRLF，也就是 Windows 风格的 `\r\n`。所以下次 Git 重新写这个文件时，会把 LF 自动换成 CRLF。

这通常不是错误，只是提示。常见原因是 Windows 环境中设置了：

```bash
git config core.autocrlf true
```

Git 行为大致如下：

- 提交到仓库时，通常规范成 LF。
- 检出到工作区时，可能转换成 CRLF。
- 因此提示文件换行符可能被自动转换。

建议检查配置：

```bash
git config --show-origin --get core.autocrlf
git config --show-origin --get core.eol
git ls-files --eol
```

如果项目主要是 Python，建议统一使用 LF，避免跨平台 diff 噪音。项目级处理可以添加 `.gitattributes`：

```gitattributes
* text=auto
*.py text eol=lf
*.md text eol=lf
*.toml text eol=lf
*.sql text eol=lf
*.yml text eol=lf
*.yaml text eol=lf
```

然后执行：

```bash
git add --renormalize .
```

总结：

- 这个 warning 不代表代码错误。
- 它主要提醒 Git 可能会自动转换换行符。
- Python 项目建议用 `.gitattributes` 固定 LF。

---

## 七、用户第四次请求

用户请求：

> 把这次会话的所有内容存进 `E:\note\客户流失预警\review by codex 4.md`

处理结果：

- 目标文件存在，大小为 0。
- 已将本轮会话的主要内容、结论、代码变更、测试结果和解释整理保存到该文件。
