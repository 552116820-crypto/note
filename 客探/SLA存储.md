• 不要放在知识库里。SLA写回建议放在业务数据层（库/表），分两层存：

  1. 运行明细表（强烈推荐，主存）

  - 一次流程一条记录，字段含：run_id、trace_id、status(success/duplicate/degraded/failed)、latency_ms、idempotency_key、error_code、alert_sent、created_at
  - 用于追溯、复盘、审计
  2. SLA聚合表（看板用）

  - 按天/小时聚合：成功率、失败率、P95耗时、告警数
  - 用于SLA看板展示，不替代明细表

  3. 原始证据归档（可选）

  - 请求/响应片段、告警payload、验收报告文件（对象存储或文件目录）
  - 用于审计取证

  结合你当前项目：

  - 你本地的 acceptance_reports/run_*/acceptance_report.json 是“验收证据”，不是线上SLA主数据源。
  - 线上建议：工作流最后一个“写回节点”调用内部API，把数据写进 MySQL/PG 的 workflow_run_ledger（明细）和 sla_daily_metrics（聚合）。


• 你判断得对，这个工作流当前没有把 SLA 写入你的业务数据库。

  现在这版里，“SLA写回”实际是两件事，不是 DB 持久化：

  1. 写回到 conversation variables（会话变量）

  - 先由 1773000010001 幂等与SLA初始化 生成 idempotency_key / trace_id / sla_start_ms。
  - 再由 1773000010004 记录幂等与起始SLA 写入会话变量：
      - last_idempotency_key
      - sla_last_trace_id
      - sla_last_query_norm
      - sla_last_start_ms
  - 后续按状态分支计算：
      - 1773000010101 SLA成功计算
      - 1773000010103 SLA降级计算
      - 1773000010105 SLA失败计算
      - 1773000010107 SLA重复计算
  - 对应 SLA写回-* assigner 节点把结果写回：
      - sla_last_status
      - sla_last_error_node
      - sla_last_latency_ms
      - sla_last_finished_at

  2. 失败时发送告警 webhook
  - 1773000001001 生产告警组装
  - 1773000001002 生产告警Webhook（URL 是 {{#env.alert_webhook_url#}}）
  - 这一步是通知，不是入库。
 
  所以你在工作流图里看不到“数据库节点/SQL”，是因为目前设计就是“会话变量写回 + 告警推送”，不包含“外部 SLA 台账入库”。
 
  参考快照文件（我刚按节点配置核对过）：
 
  - .tmp_draft_697.json
 
 
  如果你要“真正 SLA 写库”（可做看板和审计统计），下一步要加一个 http-request 节点调用你内部写库 API（或直接连数据库服务），把上面这些 sla_last_* 字段和
  idempotency_key/trace_id 一并落表。
