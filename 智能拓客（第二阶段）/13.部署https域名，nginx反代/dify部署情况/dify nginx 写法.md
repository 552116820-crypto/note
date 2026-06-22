```
[root@AI-appliancce-server ~]# docker exec docker-nginx-1 cat /etc/nginx/conf.d/default.conf
# Please do not directly edit this file. Instead, modify the .env variables related to NGINX configuration.

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
[root@AI-appliancce-server ~]# ls -la /root/dify/docker/volumes/certbot/conf/live/
total 8
drwxr-xr-x 2 root root 4096 Jan  8 10:26 .
drwxr-xr-x 3 root root 4096 Jan  8 10:26 ..
[root@AI-appliancce-server ~]# docker ps --format '{{.Names}}\t{{.Ports}}'
wecom_deepexi_aibot
churn-warning-engine-nginx-1    0.0.0.0:8080->80/tcp
churn-warning-engine-api-1      8000/tcp
churn-warning-engine-celery-1   8000/tcp
churn-warning-engine-redis-1    6379/tcp
docker-nginx-1  0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp
docker-api-1    5001/tcp
docker-worker-1 5001/tcp
docker-worker_beat-1    5001/tcp
docker-plugin_daemon-1  0.0.0.0:5003->5003/tcp, :::5003->5003/tcp
docker-web-1    3000/tcp
docker-ssrf_proxy-1     3128/tcp
docker-sandbox-1
docker-redis-1  6379/tcp
docker-db_postgres-1    5432/tcp
docker-weaviate-1
```
