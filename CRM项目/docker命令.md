• 这个项目常用的是这几条。

  启动后端整套容器：

  docker compose --env-file .compose.env up -d

  停止后端整套容器：

  docker compose --env-file .compose.env down

  查看容器状态：

  docker compose --env-file .compose.env ps

  只启动某个服务，比如 php：

  docker compose --env-file .compose.env up -d php

  只停止某个服务，比如 php：

  docker compose --env-file .compose.env stop php

  前端 node14 这个不是常驻服务，通常是临时跑开发服务器：

  启动前端开发服务：

  docker compose --env-file .compose.env run --rm -p 9528:9528 node14 npm run dev

  停止前端开发服务：
  直接在它那个终端里按 Ctrl+C

  如果前端容器挂在后台没退出，也可以手动停：

  docker ps --filter "publish=9528"
  docker stop <容器ID>

  如果你要，我也可以顺手给你整理一版“这个项目从零启动到访问”的完整命令顺序。



• 按这套顺序就行。

  先启动后端容器：

  cd ~/projects/crm_manage_system
  docker compose --env-file .compose.env up -d

  查看后端状态：

  docker compose --env-file .compose.env ps

  进入前端目录安装依赖，首次或依赖变更时执行：

  docker compose --env-file .compose.env run --rm node14 npm install

  启动前端开发服务：

  docker compose --env-file .compose.env run --rm -p 9528:9528 node14 npm run dev

  启动后访问：

  - 后端入口：http://localhost:8080
  - 前端入口：http://localhost:9528/adminVueHtml/
  - 前端登录页：http://localhost:9528/adminVueHtml/#/login

  停止方式：

  后端整套停止：

  docker compose --env-file .compose.env down

  前端开发服务停止：
  在跑 npm run dev 的终端里按 Ctrl+C

  如果 9528 被旧前端容器占用，先停掉：

  docker ps -q --filter "publish=9528" | xargs -r docker stop
