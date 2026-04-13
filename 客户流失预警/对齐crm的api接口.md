  完成的工作：

  新建文件：app/shared/crm_api.py
  - 新增 CrmApiRepository，实现 CrmRepository 接口
  - 从 https://crm.amer.cn/v2/api/OutPut/getSalesmanSaleOrder 获取订单数据
  - passport 通过 MD5(当天日期 + amer_fl_output) 动态计算
  - 针对 API 每次最多返回 100 条的限制，按单个 sales_id 逐个请求，确保数据完整
  - 首次加载后缓存到内存，后续查询直接读缓存

  修改文件：app/churn/factory.py
  - 新增 REPOSITORY_TYPE=api 模式，通过环境变量配置：
    - CRM_API_URL（默认已填好接口地址）
    - CRM_API_START_DATE / CRM_API_END_DATE（必填，日期范围）
    - CRM_API_SALES_IDS（可选，逗号分隔的业务员编号）

  测试结果（test_crm_api.py）：
  - 成功从 API 加载了 2 个业务员、21 个客户、32 条订单
  - 流失预警引擎正确识别出频率异常和产品异常
  - 完整的 factory → 创建任务 → 批次检测 → 生成预警 流程全部跑通
  - 原有 80 个单元测试全部通过，无回归
