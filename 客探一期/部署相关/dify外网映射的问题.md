• 外网映射可以按这套最稳的方式做：公网网关(Nginx) -> 私网 Dify API。
  你现在已有 https://dify.amer.cn/apps，通常只需要把 /v1/* 也通过网关放出来。

  目标

  1. 公网调用地址：https://dify.amer.cn/v1/chat-messages
  2. 实际转发到内网：http://10.x.x.x:5001/v1/chat-messages（示例）
  步骤

  3. 确认 Dify API 在内网可用
     在 Dify 所在机器执行：

     curl -i http://127.0.0.1:5001/v1/chat-messages
     返回 401/404 都行，说明服务活着且端口通。
     返回 401/404 都行，说明服务活着且端口通。
  4. 准备公网入口
     用已有的反向代理机器（或同一台机器），确保有公网 IP，443 已开放。
  5. DNS 配置
     把 dify.amer.cn（或新子域 api.dify.amer.cn）A 记录指向公网入口 IP。
  6. 配置 Nginx 反向代理（关键）
     location /v1/ 转发到内网 Dify，并开启 SSE 必要参数：
     server {
       listen 443 ssl;
       server_name dify.amer.cn;
 
       # ssl_certificate /path/fullchain.pem;
       # ssl_certificate_key /path/privkey.pem;
 
       location /v1/ {
         proxy_pass http://10.0.2.15:5001;   # 你的内网Dify API地址
         proxy_http_version 1.1;
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header X-Forwarded-Proto $scheme;
         # 流式返回(SSE)必须关缓冲
         proxy_buffering off;
         proxy_request_buffering off;
         proxy_read_timeout 3600s;
         proxy_send_timeout 3600s;
         add_header X-Accel-Buffering no;
       }
     }
  5. 如果前端是不同域名，再加 CORS
     只放行你的前端域名，不要 *：
 
     if ($request_method = OPTIONS) {
       add_header Access-Control-Allow-Origin "https://web.amer.cn" always;
       add_header Access-Control-Allow-Headers "Authorization,Content-Type" always;
       add_header Access-Control-Allow-Methods "GET,POST,OPTIONS" always;
       return 204;
     }
     add_header Access-Control-Allow-Origin "https://web.amer.cn" always;
     add_header Access-Control-Allow-Headers "Authorization,Content-Type" always;
     add_header Access-Control-Allow-Methods "GET,POST,OPTIONS" always;
  6. 云防火墙/安全组收口
      - 网关机器：只开放 443 给公网。
      - Dify 机器：5001 只允许网关内网 IP 访问，拒绝公网。
  7. 重载并验证

     nginx -t && systemctl reload nginx

     外网验证（用手机热点）：
     curl -i 'https://dify.amer.cn/v1/chat-messages' \
       -H 'Authorization: Bearer app-xxxx' \
       -H 'Content-Type: application/json' \
       -d '{"query":"ping","response_mode":"blocking","user":"u_test"}'
 
  常见坑
 
  1. apps 可访问但 /v1 404：网关没配 location /v1/ 或路径改写错。
  2. 流式不实时：没关 proxy_buffering，或前面还有 CDN 缓冲。
  3. 浏览器报 CORS：前端域名没放行。
  4. 前端直连泄露 Key：只用于 Demo，正式要后端代理。
 

  如果你愿意，我可以按你当前域名和可能的部署方式（单机 Docker / k8s ingress）给你一份“可直接粘贴”的完整 Nginx 配置（含 HTTP->HTTPS 跳转、证书路径、CORS 白名 
  单）。
