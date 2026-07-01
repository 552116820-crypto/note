
## 需求

在小程序的AI助手中，用户使用"拜访记录"功能时，AI生成的输出内容可以存进拜访日志模块，作为草稿保存，用户确认后再正式提交。

## 实现思路

整体流程：

```
用户使用AI助手(拜访记录功能)
        ↓
Dify工作流生成内容
        ↓
前端拿到AI输出结果
        ↓
用户点击"存为草稿" / 自动保存
        ↓
调用拜访日志的创建接口（status: draft）
        ↓
内容进入拜访日志列表，标记为草稿
```

## 关键设计点

### 1. 前端侧

AI对话完成后，提供一个"存为草稿"按钮，点击后调用拜访日志的保存接口。

```javascript
// 伪代码示例
async function saveAsDraft(aiOutput) {
  await request({
    url: '/api/visit-log',
    method: 'POST',
    data: {
      content: aiOutput,       // AI生成的内容
      status: 'draft',         // 草稿状态
      source: 'ai_assistant',  // 标记来源
      visit_date: '...',       // 拜访相关字段
      customer_id: '...',
    }
  })
}
```

### 2. 后端侧
  
拜访日志表加一个 `status` 字段区分草稿和已发布。

```sql
-- 拜访日志表增加字段
status ENUM('draft', 'published') DEFAULT 'draft'
source VARCHAR(20)  -- 'manual' / 'ai_assistant'，标记内容来源
```

### 3. 用户体验策略

| 策略   | 说明                | 适合场景      |
| ---- | ----------------- | --------- |
| 手动保存 | 用户确认AI内容后点击"存为草稿" | 用户需要先审核内容 |
| 自动保存 | AI生成完成后自动写入草稿     | 希望流程更顺滑   |
|      |                   |           |
|      |                   |           |

建议用**手动保存**，因为AI生成的内容用户通常需要review和编辑。

### 4. 草稿编辑

用户进入拜访日志看到草稿后，可以编辑修改，确认无误后再正式提交（status → published）。

## Dify 对接方式

Dify工作流的输出可以通过两种方式对接：

- **前端中转**：小程序前端调Dify API拿到结果 → 再调自己后端的保存接口（推荐，更灵活）
- **Dify HTTP节点回调**：在Dify工作流末尾加一个HTTP请求节点，直接把结果POST到后端接口（适合全自动场景）
