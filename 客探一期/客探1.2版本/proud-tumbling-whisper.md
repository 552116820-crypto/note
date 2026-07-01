# 日志草稿功能 - 整体架构方案设计

## Context

**问题**: 业务员通过企微H5应用调用Dify API生成拜访总结，但生成的总结目前只在页面上展示，没有持久化到CRM系统。需求是将每天所有拜访总结按客户分类汇总为"日志草稿"，用户确认后手动保存为正式日志。

**现状**:
- CRM后端(ThinkPHP 6) 已有 `PeopleDiaryModel`(工作日志) 和 `CustomerVisitModel`(拜访记录)
- 企微H5应用(独立项目) 已通过调Dify API获取拜访总结，前端自己渲染结果
- 两个系统之间目前没有草稿相关的数据流

**目标**: 实现 Dify生成总结 → 自动存为草稿 → 用户手动确认保存 的完整链路

---

## 整体数据流

```
用户输入客户名称
    ↓
H5前端调用 Dify Workflow API
    ↓
Dify返回拜访总结（含待办事项）
    ↓
H5前端展示总结给用户
    ↓ （同时）
H5前端调用 CRM后端API → 保存/追加到日志草稿
    ↓
用户进入"日志"页面 → 看到按客户分类的草稿列表
    ↓
用户点击某客户草稿 → 查看详情 → 点"保存" → 草稿变为正式日志
```

**核心思路：不需要复杂的跨页面通信机制。** H5前端在拿到Dify结果后，直接调CRM后端API存草稿；日志页面从后端拉取草稿数据。两个页面通过后端数据库解耦，不需要直接通信。

---

## 一、后端设计（CRM ThinkPHP）

### 1.1 数据库方案

**方案：新建 `diary_draft` 草稿表**（推荐，不影响现有日志逻辑）

```sql
CREATE TABLE `diary_draft` (
  `id` int(11) UNSIGNED NOT NULL AUTO_INCREMENT,
  `employee_code` varchar(50) NOT NULL COMMENT '业务员工号',
  `customer_id` int(11) NOT NULL COMMENT '客户ID',
  `customer_name` varchar(255) NOT NULL COMMENT '客户名称',
  `date` date NOT NULL COMMENT '日期（当天）',
  `content` text NOT NULL COMMENT '拜访总结内容（JSON数组，每次追加一条）',
  `todo_items` text DEFAULT NULL COMMENT '待办事项（JSON数组）',
  `status` tinyint(1) NOT NULL DEFAULT 0 COMMENT '0=草稿 1=已保存为正式日志',
  `diary_id` int(11) DEFAULT NULL COMMENT '保存后关联的正式日志ID',
  `created` datetime DEFAULT CURRENT_TIMESTAMP,
  `updated` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_employee_customer_date` (`employee_code`, `customer_id`, `date`),
  KEY `idx_employee_date_status` (`employee_code`, `date`, `status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='日志草稿表';
```

**关键设计**:
- `UNIQUE KEY (employee_code, customer_id, date)` — 保证每个业务员、每个客户、每天只有一条草稿，新的总结追加到 `content` 字段
- `content` 字段存JSON数组，每次Dify生成新总结时 append 一条，实现"自动叠加"
- `status` 区分草稿和已保存状态

### 1.2 API接口设计

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 追加草稿 | POST | `/DiaryDraft/append` | Dify生成总结后调用，追加到当天该客户的草稿 |
| 草稿列表 | GET | `/DiaryDraft/list` | 获取当前用户当天所有草稿（按客户分组） |
| 草稿详情 | GET | `/DiaryDraft/detail` | 获取某条草稿的完整内容 |
| 保存草稿 | POST | `/DiaryDraft/save` | 将草稿转为正式日志（写入PeopleDiaryModel） |
| 编辑草稿 | POST | `/DiaryDraft/update` | 用户保存前可以编辑草稿内容 |

#### 追加草稿 `/DiaryDraft/append`

```
请求参数：
{
  "customer_id": 123,
  "customer_name": "XX公司",
  "summary": "本次拜访总结内容...",
  "todo_items": ["待办1", "待办2"]
}

逻辑：
1. 根据 employee_code + customer_id + 当天日期 查找草稿
2. 如果存在且status=0 → 将summary追加到content JSON数组，合并todo_items
3. 如果不存在 → 新建一条草稿记录
4. 如果存在且status=1 → 返回提示"该客户今日日志已保存"
```

#### 保存草稿 `/DiaryDraft/save`

```
请求参数：
{
  "draft_id": 1,
  "content": "（用户可能编辑过的最终内容）"
}

逻辑：
1. 查找草稿，校验权限（只能保存自己的草稿）
2. 将内容写入 PeopleDiaryModel（创建正式日志）
3. 更新草稿 status=1, diary_id=新日志ID
```

### 1.3 后端文件结构

```
app/
├── adminApi/controller/report/
│   └── DiaryDraftCtrl.php          # 草稿API控制器（新建）
├── model/people/
│   └── DiaryDraftModel.php         # 草稿模型（新建）
└── service/
    └── DiaryDraftService.php       # 草稿业务逻辑（新建）
```

---

## 二、H5前端设计

### 2.1 Dify总结生成后的处理

在现有调用Dify API获取拜访总结的逻辑中，增加一步：

```
// 伪代码 - 在Dify返回结果后
async function onDifySummaryGenerated(customerInfo, difyResult) {
  // 1. 页面展示总结（已有逻辑）
  displaySummary(difyResult);

  // 2. 新增：调CRM后端API存草稿
  await api.post('/DiaryDraft/append', {
    customer_id: customerInfo.id,
    customer_name: customerInfo.name,
    summary: difyResult.summary,
    todo_items: difyResult.todoItems
  });

  // 3. 可选：显示一个提示 "已自动保存到日志草稿"
  showToast('已保存到日志草稿');
}
```

### 2.2 日志草稿页面

**草稿列表页**：展示当天所有草稿，按客户名称分类

```
┌──────────────────────────┐
│  📋 今日日志草稿          │
│  2026-03-20              │
├──────────────────────────┤
│  XX科技有限公司     草稿   │
│  拜访3次 | 待办2项        │
│  最后更新 14:30          │
├──────────────────────────┤
│  YY制造集团         草稿   │
│  拜访1次 | 待办1项        │
│  最后更新 10:15          │
├──────────────────────────┤
│  ZZ贸易公司        已保存  │
│  拜访2次 | 待办3项        │
│  保存于 11:00            │
└──────────────────────────┘
```

**草稿详情页**：查看某客户的所有拜访总结，可编辑，点"保存"提交

```
┌──────────────────────────┐
│  ← XX科技有限公司         │
├──────────────────────────┤
│  ▸ 第1次拜访 09:30       │
│  本次拜访了张经理...      │
│                          │
│  ▸ 第2次拜访 11:00       │
│  跟进了上次的报价...      │
│                          │
│  ▸ 第3次拜访 14:30       │
│  确认了合同条款...        │
│                          │
│  ── 待办事项 ──          │
│  □ 明天发送修改后的报价    │
│  □ 下周安排技术对接       │
│                          │
│  ┌────────────────────┐  │
│  │    保存为正式日志    │  │
│  └────────────────────┘  │
└──────────────────────────┘
```

### 2.3 页面间无需直接通信

**为什么不需要跨页面通信：**
- Dify总结页面：生成后调API存草稿 → 完事
- 日志草稿页面：每次进入时从后端拉最新数据 → 始终是最新的
- 两个页面通过后端数据库间接通信，简单可靠
- 如果想更实时，日志页面可以用 `onShow`/`visibilitychange` 事件在页面重新可见时刷新数据

---

## 三、关键注意事项

1. **并发安全**：`append` 接口需要处理并发（同一业务员快速生成多个总结），建议用数据库唯一索引 + `INSERT ... ON DUPLICATE KEY UPDATE` 或加锁
2. **内容格式**：`content` 字段存JSON数组，每条记录包含 `{ time, summary, source }` 方便展示
3. **跨天处理**：以 `date` 字段按天隔离，次日自动是新草稿
4. **权限控制**：复用现有 `TokenCheck` 中间件，从 `adminInfo` 获取 `employee_code`
5. **历史草稿**：未保存的草稿保留在数据库中，可以后续查看或提醒用户

---

## 四、实现优先级

1. **第一步**：建表 + 后端 `append`/`list`/`detail`/`save` 四个API
2. **第二步**：H5前端在Dify返回结果后调 `append` 接口
3. **第三步**：H5前端新增日志草稿列表页和详情页
4. **第四步**：完善细节（编辑草稿、未保存提醒、历史草稿查看等）

---

## 五、验证方式

1. 调Dify生成一条拜访总结 → 检查数据库 `diary_draft` 表是否有记录
2. 对同一客户再生成一条 → 检查是否追加到同一条草稿的 `content` 数组
3. 进入日志页面 → 确认能看到按客户分类的草稿列表
4. 点击保存 → 确认 `diary_draft.status` 变为1，且 `people_diary` 表新增一条正式日志
5. 保存后再对该客户生成总结 → 确认提示"今日日志已保存"或创建新草稿（根据业务需求定）
