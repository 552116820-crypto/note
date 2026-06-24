root@AI-appliancce-server ~# docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep -E '0.0.0.0:(80|443)'
docker-nginx-1               0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp
root@AI-appliancce-server ~# echo '--- compose 里的 nginx 端口 ---'
--- compose 里的 nginx 端口 ---
root@AI-appliancce-server ~# grep -nE 'EXPOSE_NGINX|"80|"443|:80|:443' /root/dify/docker/docker-compose.yaml
177:  WEAVIATE_ENDPOINT: ${WEAVIATE_ENDPOINT:-http://weaviate:8080}
573:  EXPOSE_NGINX_PORT: ${EXPOSE_NGINX_PORT:-80}
574:  EXPOSE_NGINX_SSL_PORT: ${EXPOSE_NGINX_SSL_PORT:-443}
1113:      - "{EXPOSE_NGINX_PORT:-80}:{NGINX_PORT:-80}"
1114:      - "{EXPOSE_NGINX_SSL_PORT:-443}:{NGINX_SSL_PORT:-443}"
1243:          "curl -s -f -u Administrator:password http://localhost:8091/pools/default/buckets | grep -q '\\{' || exit 1",
root@AI-appliancce-server ~# echo '--- .env 里的端口变量 ---'
--- .env 里的端口变量 ---
root@AI-appliancce-server ~# grep -nE 'EXPOSE_NGINX_(PORT|SSL_PORT)' /root/dify/docker/.env
1325:EXPOSE_NGINX_PORT=80
1326:EXPOSE_NGINX_SSL_PORT=443
root@AI-appliancce-server ~# docker exec docker-nginx-1 sh -c 'cat /etc/nginx/conf.d/.conf'
80 口：全部 301 跳到 https
server {
  listen 80;
  server_name aisales.amer.cn;
  return 301 https://$host$request_uri;
}

443 口：终结 TLS，反代到 api 容器
server {
  listen 443 ssl;
  server_name aisales.amer.cn;

  ssl_certificate     /etc/ssl/aisales.amer.cn.pem;
  ssl_certificate_key /etc/ssl/aisales.amer.cn.key;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 10m;

  location / {
  resolver 127.0.0.11 valid=10s;
  set $aisales_upstream aisales-app:8000;
  proxy_pass http://$aisales_upstream;
  include proxy.conf;
  }
}
Please do not directly edit this file. Instead, modify the .env variables related to NGINX configuration.

server {
    listen 80;
    server_name dify.amer.cn;

    location /console/api {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /api {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /v1 {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /files {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /explore {
      proxy_pass http://web:3000;
      include proxy.conf;
    }

    location /e/ {
      proxy_pass http://plugin_daemon:5002;
      proxy_set_header Dify-Hook-Url $scheme://$host$request_uri;
      include proxy.conf;
    }

    location / {
      proxy_pass http://web:3000;
      include proxy.conf;
    }

    location /mcp {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    location /triggers {
      proxy_pass http://api:5001;
      include proxy.conf;
    }

    # placeholder for acme challenge location


    # placeholder for https config defined in https.conf.template
    # Please do not directly edit this file. Instead, modify the .env variables related to NGINX configuration.

listen 443 ssl;
ssl_certificate /etc/ssl/dify.amer.cn.pem;
ssl_certificate_key /etc/ssl/dify.amer.cn.key;
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
}
root@AI-appliancce-server ~# docker exec docker-nginx-1 cat /etc/nginx/proxy.conf
Please do not directly edit this file. Instead, modify the .env variables related to NGINX configuration.

proxy_set_header Host $host;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Port $server_port;
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_buffering off;
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
root@AI-appliancce-server ~# echo '--- 证书 ---'; ls -l /root/dify/docker/nginx/ssl/ | grep -E 'dify|aisales'
--- 证书 ---
-rw------- 1 root root 1675 Jun 18 09:14 aisales.amer.cn.key
-rw-r--r-- 1 root root 3834 Jun 18 09:14 aisales.amer.cn.pem
-rw-r--r-- 1 root root 1675 May 11 18:54 dify.amer.cn.key
-rw-r--r-- 1 root root 3822 May 11 18:58 dify.amer.cn.pem
root@AI-appliancce-server ~# echo '--- 网络 ---'; docker network ls | grep docker_default
--- 网络 ---
29e0dc1d3a08   docker_default                bridge    local
root@AI-appliancce-server ~# echo '--- api 别名 ---'; docker exec docker-nginx-1 getent hosts aisales-app
--- api 别名 ---
172.24.0.12     aisales-app
root@AI-appliancce-server ~# ss -ltn | grep -E ':(8081|8444) ' && echo '⚠️ 有占用' || echo '✅ 8081/8444 空闲可用'
✅ 8081/8444 空闲可用
root@AI-appliancce-server ~# ss -ltn | grep -E ':(808^C' && echo '⚠️ 有占用' || echo '✅ 8081/8444 空闲可用'
root@AI-appliancce-server ~# ss -ltn | grep -E ':8080 ' && echo '⚠️ 有占用' || echo '✅ 8080 空闲可用'
✅ 8080 空闲可用