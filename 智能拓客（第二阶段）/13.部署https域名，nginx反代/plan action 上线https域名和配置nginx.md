## 线一:立刻催管理员(上线瓶颈,先发)

1. **DNS**:`aisales.amer.cn` → 本机公网 IP。
2. **证书**:签一张 `aisales.amer.cn` 单域名证书,要 `aisales.amer.cn.pem`(fullchain)+ `aisales.amer.cn.key` 两个文件。

## 线二:你自己做(不用等管理员)

接入 Dify 的网络。** 你的 `docker-compose.yml` 加这些(其余全不动):

```yaml
services:
  api:
    # ... 现有配置全部保留 ...
    networks:
      default: {}            # 保留:继续和 redis 通信
      dify:
        aliases: [aisales-app] # 新增:Dify 的 nginx 用这个名字找你
    # ports: "127.0.0.1:8000:8000" 这行可删——方案③走 docker 网络,不需要宿主端口了(删了更安全)

networks:
  dify:
    external: true
    name: docker_default     # Dify 的网络
```

部署到服务器后启动(**注意 `--build`,否则静默用旧镜像**;确保服务器上该目录有 `.env`):

```bash
docker compose up -d --build
# 验证 app 起来了、且 Dify 的 nginx 能解析到它:
docker exec docker-nginx-1 getent hosts aisales-app    # 能返回一个 IP 就对了
```

## 线三:证书到位后,几分钟收尾

**1. 放证书**到 Dify 的 ssl 目录(宿主):

```bash
# 把管理员给的两个文件放到这(= 容器内 /etc/ssl/)
/root/dify/docker/nginx/ssl/aisales.amer.cn.pem
/root/dify/docker/nginx/ssl/aisales.amer.cn.key
```

**2. 新建** `/root/dify/docker/nginx/conf.d/aisales.amer.cn.conf`:

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name aisales.amer.cn;

    ssl_certificate     /etc/ssl/aisales.amer.cn.pem;
    ssl_certificate_key /etc/ssl/aisales.amer.cn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    location / {
        resolver 127.0.0.11 valid=10s;       # docker 内置 DNS:app 重启换 IP 也跟得上,不会 502
        set $aisales_upstream aisales-app:8000;
        proxy_pass http://$aisales_upstream;
        include proxy.conf;                   # 复用 Dify 那套转发头,不用自己写
    }
}
```

**3. 测试 + 重载**(nginx 在容器里,不是宿主):

```bash
docker exec docker-nginx-1 nginx -t          # 必须先过
docker exec docker-nginx-1 nginx -s reload   # 平滑重载,对 dify.amer.cn 零影响
```

**4. 验证**:

```bash
curl -I https://aisales.amer.cn
```
