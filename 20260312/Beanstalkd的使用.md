• 那我接着讲这 3 个部分。

  1. Beanstalkd 和其他队列的区别

  Beanstalkd

  - 定位是“简单任务队列”
  - 适合异步 job：发邮件、发短信、生成缩略图、订单后处理
  - 部署轻，学习成本低
  - 数据主要在内存里，重启和持久化能力不如专业消息系统强

  Redis

  - Redis 本身不是专门的消息队列，但常被拿来做队列
  - 生态更广，功能更多
  - 如果只是简单队列，Redis 也能做；如果需要更完整的 job 模型，常配合 BullMQ、RQ、Celery 等

  RabbitMQ

  - 更标准的消息中间件
  - 路由、确认、重试、死信等能力更完整
  - 适合企业级消息流转，复杂度也更高

  Kafka

  - 更偏“流式平台”和高吞吐消息系统
  - 适合日志、事件流、数据管道
  - 不适合把它当成 Beanstalkd 这种简单后台任务队列来理解

  一句话：

  - 想要简单、快、轻量：Beanstalkd
  - 想要复杂路由和强消息能力：RabbitMQ
  - 想要大规模事件流：Kafka
  - 已经大量使用 Redis，任务也不复杂：Redis + 队列库

  2. Beanstalkd 的基本使用

  核心概念：

  - put：投递任务
  - reserve：取任务
  - delete：处理成功后删除
  - release：处理失败后放回队列
  - bury：彻底标记为失败，暂不再消费
  - kick：把 buried/delayed 的任务重新激活
  - tube：队列名

  典型流程：

  生产者：

  use email
  put 1024 0 60 11
  hello world

  消费者：

  watch email
  reserve
  delete <job-id>

  参数大意：

  - priority：优先级，数字越小优先级越高
  - delay：延迟多少秒执行
  - ttr：worker 拿到任务后，允许处理的最长时间

  常见场景：

  - 立即任务：delay = 0
  - 延迟任务：比如 30 秒后重试
  - 失败重试：worker 失败后 release
  - 人工排查失败任务：bury

  3. 怎么接入

  PHP

  - 常见客户端会提供 put、reserve、delete 这些封装
  - Web 请求里负责投递任务
  - CLI worker 常驻进程负责消费任务

  Python

  - 用客户端库连接 beanstalkd
  - 一个脚本持续 reserve，拿到任务后执行
  - 成功 delete，失败 release 或 bury

  Go

  - Go 很适合写常驻 worker
  - 启动多个 goroutine 消费不同 tube
  - 适合高并发处理简单任务

  一个非常简化的伪代码：

  生产者：

  queue.put({"type": "send_email", "user_id": 123})

  消费者：

  while True:
      job = queue.reserve()
      try:
          handle(job)
          job.delete()
      except Exception:
          job.release(delay=30)

  什么时候适合用

  - 你的需求就是“把慢任务从主流程拆出去”
  - 不想引入太重的消息系统
  - 可以接受它是一个偏轻量、偏简单的方案

  什么时候不太适合

  - 你需要很强的持久化和高可靠投递保证
  - 你需要复杂消息路由、多订阅模型
  - 你要做大规模事件流处理

  如果你愿意，我下一步可以直接给你一份：

  1. Beanstalkd 的安装和启动命令
  2. 一个最小可运行的生产者/消费者示例
  3. 它和 MySQL 配合的常见架构图




• 下面按“架构”和“Python 用法”两部分说。

  常见架构
  最常见的思路是：MySQL 负责存业务真相，Beanstalkd 只负责搬运异步任务，Worker 负责执行任务并回写 MySQL。

  简单版架构图：

  [Web/API]
     |
     | 1. 写业务数据
     v
   [MySQL]
     |
     | 2. 投递异步任务
     v
  [Beanstalkd]  --->  [Worker x N]  --->  [邮件/短信/第三方接口]
                           |
                           | 3. 更新任务结果/业务状态
                           v
                         [MySQL]

  典型例子：

  - 用户下单
  - 主流程先把订单写进 MySQL
  - 然后往 Beanstalkd 放一个 send_order_email 或 sync_inventory job
  - Worker 取出 job 执行
  - 成功后更新 MySQL 里的状态或日志

  这个方案简单，但有一个经典问题：

  - 如果 MySQL 提交成功，但 put job 失败，任务会丢
  - 如果 job 放进去了，但数据库事务回滚了，worker 可能处理到“脏任务”

  更稳的工程做法是 Outbox 模式。这是实践推导，不是 Beanstalkd 自带功能。

  更稳版架构图：

  [Web/API]
     |
     | 1. 同一事务里写两张表
     v
  [MySQL]
     |-- orders / users / payments
     |-- job_outbox(status=pending, payload=...)
     |
     v
  [Dispatcher / Poller]
     |
     | 2. 扫描 pending outbox，投递到队列
     v
  [Beanstalkd]  --->  [Worker x N]
                           |
                           | 3. 执行业务，更新 MySQL
                           v
                         [MySQL]

  这种方式的好处：

  - 业务数据和“待投递任务”一起提交，避免数据库和队列不一致
  - 可以做失败重试、人工补偿、任务审计
  - 更适合订单、支付、通知这类要追踪的场景

  落地时建议记住 3 点：

  - MySQL 是真相来源，Beanstalkd 只是异步通道
  - Worker 要做幂等，比如 update ... where status != 'DONE'
  - 长任务要注意 TTR，必要时用 touch 续期

  Python 中怎么用
  在 Python 里，通常直接用客户端库连 Beanstalkd。我这里用 greenstalk 举例。它的文档直接对应 beanstalkd 协议，适合写 producer / consumer。根据 PyPI，截至 2026-03-12，greenstalk 最新版是 2.1.1，发布时间是 2025-
  10-05，要求 Python >= 3.8。

  先安装：

  pip install greenstalk pymysql

  生产者示例，把任务放进 tube：

  import json
  import greenstalk

  payload = {
      "type": "send_welcome_email",
      "user_id": 123,
  }

  with greenstalk.Client(("127.0.0.1", 11300), use="email_jobs") as client:
      job_id = client.put(
          json.dumps(payload),
          priority=65536,
          delay=0,
          ttr=60,
      )
      print("job_id =", job_id)

  消费者示例，取任务并处理：

  import json
  import greenstalk
  import pymysql

  db = pymysql.connect(
      host="127.0.0.1",
      user="root",
      password="root",
      database="demo",
      autocommit=False,
      charset="utf8mb4",
  )

  with greenstalk.Client(("127.0.0.1", 11300), watch=["email_jobs"]) as client:
      while True:
          try:
              job = client.reserve(timeout=5)
          except greenstalk.TimedOutError:
              continue

          try:
              payload = json.loads(job.body)

              with db.cursor() as cur:
                  cur.execute(
                      """
                      update users
                      set email_sent = 1
                      where id = %s and email_sent = 0
                      """,
                      (payload["user_id"],),
                  )
              db.commit()

              client.delete(job)
          except (pymysql.err.OperationalError, ConnectionError):
              db.rollback()
              client.release(job, delay=30)
          except Exception:
              db.rollback()
              client.bury(job)

  这段代码里：

  - reserve() 取任务
  - 成功后 delete(job)
  - 可重试错误用 release(job, delay=30)
  - 不确定错误用 bury(job)，留着排查

  如果任务执行时间可能超过 ttr，中途要续命：

  client.touch(job)

  和 MySQL 一起用时的推荐分工

  - Web/API：写业务数据，必要时写 outbox
  - Dispatcher：把 outbox 推到 Beanstalkd
  - Worker：消费 job，调外部服务，更新 MySQL
  - MySQL 表里建议至少有：业务状态、重试次数、最后错误信息、更新时间
