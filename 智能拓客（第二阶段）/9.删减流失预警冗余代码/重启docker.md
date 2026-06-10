› docker部署在服务器上需要拉取新代码并重新启动，对吧？


• 对，需要。

  因为这次改了代码里的 ORM/schema/service，并且数据库已执行 020 删除了 job_id 列，所以服务器上的 Docker 服务必须拉取新代
  码并重新构建/重启。否则旧镜像还会尝试读写 job_id，会报数据库字段不存在。

  大致流程：

  git pull
  docker compose build
  docker compose up -d

  如果镜像不是本机 build，而是 CI/CD 推镜像，则服务器应拉新镜像后重启：

  docker compose pull
  docker compose up -d

  重启后建议看一下日志，确认 Celery 和 API 都正常：

  docker compose logs -f
