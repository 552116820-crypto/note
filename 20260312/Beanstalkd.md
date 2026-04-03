• Beanstalkd 是一个很轻量的消息队列服务，常用来做“异步任务”。

  你可以把它理解成一个任务中转站：

  - 业务系统先把任务塞进去
  - 后台 worker 再慢慢取出来处理
  - 用户请求不用一直等任务做完

  在 CRM 里常见用途：

  - 发短信
  - 发邮件
  - 同步客户数据
  - 生成报表
  - 批量导入导出
  - 其他耗时操作

  它的基本角色有两个：

  - producer：往队列里放任务
  - consumer/worker：从队列里取任务并执行

  优点：

  - 非常简单
  - 性能高
  - 部署容易
  - 适合做短任务队列

  局限：

  - 功能比 RabbitMQ、Kafka 简单很多
  - 不适合复杂消息路由
  - 不适合大规模流式数据场景

  一个典型流程是：

  1. 用户在 CRM 点“批量发送短信”
  2. Web 服务把每条短信任务丢进 Beanstalkd
  3. PHP/Python worker 从队列中取任务
  4. worker 调短信接口并回写结果


• 那就继续说实用一点的。

  和 Redis 队列的区别

  Beanstalkd 更像“专门做任务队列”的工具，Redis 是“通用内存数据库，顺便也能做队列”。

  主要区别：

  - Beanstalkd 天生就是任务队列
      - 有 ready / reserved / delayed / buried 这些任务状态
      - worker 取到任务后，不 delete 就不会算完成
      - 支持优先级、延迟执行、超时重试
  - Redis 更通用
      - 你可以用 list、stream、pub/sub 自己拼队列
      - 生态更强，但很多可靠性细节要自己补
  - 老 CRM / 老 PHP 项目里，Beanstalkd 很常见
      - 配置简单
      - 协议简单
      - worker 模型很直接

  如果只是“后台任务排队执行”，Beanstalkd 很合适。
  如果还要缓存、分布式锁、排行榜、发布订阅，Redis 更全面。

  在 CRM 里一般怎么用

  常见结构就是 3 层：

  1. Web 应用把任务丢进队列
  2. Beanstalkd 存任务
  3. 一个或多个 worker 持续消费任务

  例如：

  - 用户提交“批量发送短信”
  - PHP 把每条短信写入 sms_queue
  - Python/PHP worker 从 sms_queue 取任务
  - 调第三方短信接口
  - 成功就 delete
  - 失败就 release 或 bury

  部署上通常只需要：

  - 一个 beanstalkd 服务，默认端口 11300
  - 一个或多个 worker 进程
  - 用 supervisor、systemd 或 Docker 保活 worker

  如果你用 Docker，最小启动可以是：

  docker run -d --name beanstalkd -p 11300:11300 schickling/beanstalkd

  应用里连接地址就是：

  127.0.0.1:11300

  如果在 docker compose 里，通常写服务名：

  beanstalkd:11300

  Python 2.7 怎么连

  老项目里常用 beanstalkc。

  安装：

  pip install beanstalkc

  生产者示例：

  # -*- coding: utf-8 -*-
  import json
  import beanstalkc

  beanstalk = beanstalkc.Connection(host='127.0.0.1', port=11300)
  beanstalk.use('sms_queue')

  job_data = {
      'mobile': '13800138000',
      'message': 'hello',
  }

  beanstalk.put(
      json.dumps(job_data),
      priority=1024,
      delay=0,
      ttr=60
  )

  print 'job pushed'

  消费者示例：

  # -*- coding: utf-8 -*-
  import json
  import time
  import beanstalkc

  beanstalk = beanstalkc.Connection(host='127.0.0.1', port=11300)
  beanstalk.watch('sms_queue')
  beanstalk.ignore('default')

  while True:
      job = beanstalk.reserve(timeout=5)
      if job is None:
          continue

      try:
          data = json.loads(job.body)
          print 'processing:', data

          # 在这里写你的业务逻辑
          # 比如发短信、发邮件、同步 CRM 数据

          job.delete()
          print 'done'
      except Exception as e:
          print 'error:', str(e)
          job.release(delay=10)
          time.sleep(1)

  几个关键概念

  - tube
      - 类似队列名，比如 sms_queue、email_queue
  - put
      - 投递任务
  - reserve
      - worker 取任务
  - delete
      - 任务处理成功，删除
  - release
      - 任务失败，重新放回队列
  - bury
      - 任务有问题，先挂起，等人工处理
  - ttr
      - worker 拿到任务后，允许占用的最长处理时间

  适不适合你的 CRM

  如果这个 CRM 是老 PHP / Python 系统，而且环境文档明确写了 Beanstalkd，通常说明它就是拿来跑异步任务的，不建议删掉。很多老系统对它有直接依赖。



• 没有绝对“哪个好”，看场景。

  对你这个 CRM 场景，Beanstalkd 更合适；如果你是在做新的事件平台、数据总线、日志流处理，才考虑 Kafka。

  - Beanstalkd 更像“任务队列”。官方定位就是简单快速的 work queue，支持任务的 ready / reserved / delayed / buried 状态、延迟、超时 TTR、重试，适合发短信、发邮件、报表、导入导出这类后台异步任务。
  - Kafka 更像“事件流平台”。官方文档强调 topic 持久保留、multi-producer / multi-subscriber、分区扩展、consumer group、offset 管理，适合一个事件被多个系统消费、保留后反复回放、做流处理和数据管道。

  实际选型可以直接这么判：

  - 选 Beanstalkd：老 CRM、PHP/Python 项目、异步任务队列、部署要简单。
  - 选 Kafka：多系统订阅同一事件、需要消息保留和重放、吞吐量很大、要做事件驱动架构或数据流平台。
  - 不要把 Kafka 当成 Beanstalkd 的“高级替代品”，两者模型不同。

  对你现在这个系统，我建议继续用 Beanstalkd。环境文档已经明确点名它，这通常说明系统就是按“简单后台任务队列”设计的，直接换成 Kafka 往往不是配置问题，而是架构重做。

  运维复杂度这一点是我根据官方文档能力范围做的推断：Beanstalkd 官方写明“没有配置文件，只有少量命令行参数”；Kafka 官方文档则涉及 topics、partitions、consumer groups、offsets 和 group 管理，所以落地和维护明显更
  重。
