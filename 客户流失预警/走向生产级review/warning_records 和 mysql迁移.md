• warning_records 全表按 customer_id 去重后还有 6031 条。

  我查的口径是：

  SELECT COUNT(DISTINCT customer_id) FROM warning_records;

  同时查到全表总行数是 8171 条，所以按 customer_id 去重会减少 2140 条重复记录。




• 已执行新增迁移：migrations/mysql/churn/007_churn_ai_analyses.sql。

  校验结果：

  - churn_ai_analyses 表已存在
  - 表结构包含迁移里的 16 个字段
  - 索引已存在：PRIMARY、ix_churn_ai_analysis_cache、ix_churn_ai_analysis_sales_customer、ix_churn_ai_analysis_input_hash
  - 当前表内数据量：0

  我用的是当前项目 .env 里的 DATABASE_URL，没有改代码文件。



• 已执行迁移：migrations/mysql/churn/008_warning_records_list_sort_index.sql。

  结果：

  - 新增索引已存在：ix_warning_records_biz_date_warning_id
  - 索引列顺序确认：biz_date, warning_id
  - 执行前该索引不存在，执行后 information_schema.STATISTICS 中有 2 个列条目
  - warning_records 当前行数：8171

  我用的是当前项目 .env 里的 DATABASE_URL，没有修改代码文件。