# 把 Dify 从 443 解耦：edge 中立网关方案

## Context（为什么做）

现状是**方案③（寄生）**:`aisales.amer.cn` 挂在 Dify 的 `docker-nginx-1`(占宿主 80/443)下—— 它的 nginx 配置写在 `/root/dify/docker/nginx/conf.d/aisales.amer.cn.conf`、证书放在 Dify 的 ssl 目录、 api 容器还被接进了 Dify 的 `docker_default` 网络(别名 `aisales-app`)。后果:配置/证书寄居在 Dify 树里 (Dify 重部署/升级可能抹掉)、Dify nginx 停机时 aisales 一起停、所有权混乱。

目标:让 `aisales.amer.cn` 和 `dify.amer.cn` 在**同一个 443 入口**下被**平级分流**,谁都不寄生谁。 （已确认:单公网 IP，故走 edge 中立网关；那一步动 Dify 由你自己在低峰窗口执行。）

**一个绕不开的事实**:一个 IP 上 443 只能有一个监听者。真解耦 = 让一台**中立 nginx**占 443、按域名(SNI) 分流,Dify 让出宿主 80/443、退到内网。这一步动 Dify（几秒中断）是解耦的唯一代价点，可逆。

## 目标架构

                              ┌ dify.amer.cn   → docker-nginx-1:80   (Dify，退到内网，配置/证书不动)  
浏览器 ─80/443─→ [edge-gateway] ┤  
                 中立独立 nginx   └ aisales.amer.cn → aisales-app:8000 (= amer_sales_assist-api-1)

- edge = 新的独立 compose 项目 `/root/edge-gateway/`，只干一件事:占 443、按域名分流。
    
- edge 只接 **`docker_default` 一个网络**就够:`docker-nginx-1` 和 `aisales-app` 都在里面（沿用现网已验证的别名路径，最低风险）。
    
- `:8080`（churn 业务员，`amer_sales_assist-nginx-1`）全程不动。
    

## 关键设计决策

1. **L7 终结 TLS**:edge 用 Dify/aisales 两张证书终结浏览器 TLS，再以 HTTP 回源。比 L4 透传简单、可 `curl` 调试、保留真实客户端 IP（`X-Forwarded-For`）。代价:edge 是新的共享入口（它挂则两域名同挂）、Dify 多一跳——这是单 IP 多域名的固有成本，换来配置归位与所有权清爽。
    
2. **证书零拷贝**:edge **只读挂载** `/root/dify/docker/nginx/ssl`（两张证书已都在此）。续期只改原文件 + reload edge，无副本、无"忘了同步"隐患。
    
3. **逐字复用 `proxy.conf`**:aisales 路径 `include` 一份和现网完全一致的 `proxy.conf`（体检时从 `docker-nginx-1` 抓取），行为零变化。
    
4. **resolver 动态解析**:回源用 `set $u ...; proxy_pass http://$u;` + `resolver 127.0.0.11 valid=10s;`，容器重建换 IP 自愈、不缓存死。
    
5. **api 留在 docker_default**（沿用现网别名 `aisales-app`）:复用已验证路径，cutover 最小化。深度解耦（把 api 移出 Dify 网络）列为可选后续，非本次必须。
    

## 路线对比：edge(B) vs 第二个 IP(A)——代价到底落在哪

> 纠正一处易误解的说法：edge 方案里 **aisales 流量并不"绕回 Dify 找 api"**。三种方案下 aisales 都是「**一层 nginx → api(uvicorn:8000)**」，层数完全一样，只是那层 nginx 是谁不同（api 容器内是 uvicorn，自己不带 nginx）：

当前③ ： 浏览器 ─443─→ [Dify nginx]      ──→ api(uvicorn:8000)  
edge B ： 浏览器 ─443─→ [edge nginx]      ──→ api(uvicorn:8000)   ← 不经过 :8080 那台 nginx  
二IP A ： 浏览器 ─443─→ [你自己的 nginx]   ──→ api(uvicorn:8000)

|维度|edge 网关（B，本计划）|第二个 IP（A）|
|---|---|---|
|**aisales 路径**|edge nginx → api（一层；可做到完全不碰 Dify 网络）|自己 nginx → api（一层）|
|**dify 路径**|浏览器 → **edge** → Dify nginx → dify-api（**dify 多穿一层 edge**）|浏览器 → Dify nginx → dify-api（不变）|
|**共享单点**|edge 占唯一 443，**它挂则两域名同挂**|无；dify 走 IP1、aisales 走 IP2，各回各家|
|**动 Dify**|让出宿主 80/443（重一点）|仅把 `0.0.0.0:443` 收窄到自己 IP（轻）|
|**新增成本**|无（不花钱、不求管理员，DNS/证书已齐）|一个 EIP（常年计费）+ 管理员配辅助私网IP/DNS|
|**新建容器**|是（edge）|否（复用现有 `amer_sales_assist-nginx-1`）|

**结论**：edge 的代价落在 **dify（多一跳）和"共享前门"** 上，**不在 aisales**；换来零成本、不求人、今天就能自己做完。A 用一个 EIP + 管理员配合换掉"共享前门"，是终局最干净的形态，但更重。

## 执行步骤（你自己做，分步贴回，我确认再走下一步）

### 第 0 步 — 体检（只读，先做；确认 3 个未知量）

# A. 现在谁占 80/443（应只有 docker-nginx-1）  
docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep -E '0.0.0.0:(80|443)'  
​  
# B. Dify nginx 的 host 端口是写死还是 .env 变量 → 决定第 3 步怎么改  
grep -rnE 'EXPOSE_NGINX_(PORT|SSL_PORT)' /root/dify/docker/.env 2>/dev/null  
grep -nE 'EXPOSE_NGINX|:80"|:443"' /root/dify/docker/docker-compose.yaml  
​  
# C. Dify nginx 的 :80 行为（直服务 or 强跳 443）→ 决定 edge 回源 :80 还是 :443  
docker exec docker-nginx-1 nginx -T 2>/dev/null | awk '/listen 80/{f=1} f{print} f&&/}/{exit}'  
​  
# D. 抓现网 proxy.conf（edge 逐字复用）  
docker exec docker-nginx-1 cat /etc/nginx/proxy.conf  
​  
# E. 证书 / 网络 / 别名都在位  
ls -l /root/dify/docker/nginx/ssl/ | grep -E 'dify|aisales'  
docker network ls | grep docker_default  
docker exec docker-nginx-1 getent hosts aisales-app

> **C 是关键分叉**:若 `:80` 直接服务 Dify → edge 回源 `http://docker-nginx-1:80`（主路径）。 若 `:80` 无条件 `301` 跳 443 → 改回源 `https://docker-nginx-1:443` + `proxy_ssl_server_name on;`（fallback，避免跳转环）。体检后定。

### 第 1 步 — 建 edge 文件（不动现网，443 还被 Dify 占着）

目录 `/root/edge-gateway/`，结构:

docker-compose.yml  
proxy.conf            # 第 0 步 D 抓到的内容，逐字粘贴  
conf.d/00-global.conf  
conf.d/01-http.conf  
conf.d/dify.conf  
conf.d/aisales.conf

`docker-compose.yml`:

services:  
  gateway:  
    image: nginx:1.27-alpine  
    container_name: edge-gateway  
    restart: always  
    ports: ["80:80", "443:443"]  
    volumes:  
      - ./conf.d:/etc/nginx/conf.d:ro  
      - ./proxy.conf:/etc/nginx/proxy.conf:ro  
      - /root/dify/docker/nginx/ssl:/etc/nginx/ssl:ro   # 证书零拷贝，续期自动反映  
    networks: [dify_net]  
networks:  
  dify_net: { external: true, name: docker_default }

`conf.d/00-global.conf`:

map $http_upgrade $connection_upgrade { default upgrade; '' close; }  
client_max_body_size 100m;

`conf.d/01-http.conf`（80 统一跳 443）:

server { listen 80 default_server; server_name _; return 301 https://$host$request_uri; }

`conf.d/dify.conf`（回源目标按第 0 步 C 定稿）:

server {  
    listen 443 ssl; http2 on;  
    server_name dify.amer.cn;  
    ssl_certificate /etc/nginx/ssl/dify.amer.cn.pem;  
    ssl_certificate_key /etc/nginx/ssl/dify.amer.cn.key;  
    ssl_protocols TLSv1.2 TLSv1.3;  
    location / {  
        resolver 127.0.0.11 valid=10s;  
        set $u docker-nginx-1:80;  
        proxy_pass http://$u;  
        proxy_set_header Host $host;  
        proxy_set_header X-Real-IP $remote_addr;  
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
        proxy_set_header X-Forwarded-Proto $scheme;  
        proxy_http_version 1.1;  
        proxy_set_header Upgrade $http_upgrade;  
        proxy_set_header Connection $connection_upgrade;   # Dify 控制台 WebSocket  
        proxy_read_timeout 3600s;  
    }  
}

`conf.d/aisales.conf`（逐字复用 proxy.conf，行为同现网）:

server {
    listen 443 ssl; http2 on;
    server_name aisales.amer.cn;
    ssl_certificate /etc/nginx/ssl/aisales.amer.cn.pem;
    ssl_certificate_key /etc/nginx/ssl/aisales.amer.cn.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    location / {
        resolver 127.0.0.11 valid=10s;
        set $u aisales-app:8000;
        proxy_pass http://$u;
        include proxy.conf;
    }
}

### 第 2 步 — 校验（仍不动现网）

cd /root/edge-gateway
docker compose config >/dev/null && echo "compose OK"
# 临时跑一个一次性容器只做 nginx -t（不占端口、不影响现网）：
docker run --rm -v "$PWD/conf.d:/etc/nginx/conf.d:ro" -v "$PWD/proxy.conf:/etc/nginx/proxy.conf:ro" \
  -v /root/dify/docker/nginx/ssl:/etc/nginx/ssl:ro nginx:1.27-alpine nginx -t

### 第 3 步 — 低峰切换（连贯执行，dify+aisales 闪断几秒）

# 1) Dify 让出 host 80/443（先备份！）
cd /root/dify/docker && cp docker-compose.yaml docker-compose.yaml.bak
#   按第 0 步 B 结果：写死则注释掉 nginx 的 80/443 host 映射；变量则在 .env 处理
docker compose up -d nginx        # 重建 dify nginx，释放宿主 80/443

# 2) 立刻起 edge 占 80/443
cd /root/edge-gateway && docker compose up -d

### 第 4 步 — 验证（见下方验证清单）

### 第 5 步 — 清理（验证通过后）

# 旧寄生配置职责已搬到 edge，删掉 + reload（Dify 自身 default.conf 不动）
rm /root/dify/docker/nginx/conf.d/aisales.amer.cn.conf
docker exec docker-nginx-1 nginx -t && docker exec docker-nginx-1 nginx -s reload

## 回滚预案（~30 秒退回方案③）

cd /root/edge-gateway && docker compose down                 # 让出 80/443
cd /root/dify/docker && cp docker-compose.yaml.bak docker-compose.yaml
docker compose up -d nginx                                   # dify 重占 80/443
# 若已删 aisales.amer.cn.conf：从 git 或备份恢复到 conf.d 后
docker exec docker-nginx-1 nginx -s reload

## 验证清单

docker ps --format 'table {{.Names}}\t{{.Ports}}' | grep -E 'edge|docker-nginx-1'  # edge 占 80/443；docker-nginx-1 不再有 0.0.0.0:443
curl -i  https://aisales.amer.cn/healthz      # 200 + {"status":"ok",...}
curl -I  https://dify.amer.cn                 # 200/302，证书无报错
curl -sf http://127.0.0.1:8080/healthz && echo ":8080 OK"   # churn 现网未受影响
curl -I  http://aisales.amer.cn/healthz       # 301 → https

浏览器另测:`https://dify.amer.cn` 控制台能登录、对话流式不断（WebSocket/SSE）；`https://aisales.amer.cn` 绿锁无警告。

## 风险与注意

- **唯一危险步是第 3 步动 Dify 端口**:已备份 + 可秒级回滚 + 选低峰。其余步骤都不碰现网。
    
- **Dify 升级可能还原它的 `docker-compose.yaml`**（端口映射回来 → 和 edge 抢 443）。记一笔；或改用 Dify 的 override 置空端口降低被覆盖风险。
    
- **证书续期**:aisales 那张 DigiCert **2026-09-14 到期**。续期 = 覆盖 `/root/dify/docker/nginx/ssl/aisales.amer.cn.{pem,key}` + `docker exec edge-gateway nginx -s reload`（不再是 reload dify nginx）。dify 证书续期后同样补一条 reload edge。
    
- **proxy.conf 内含 `$connection_upgrade`** 时，map 已在 `00-global.conf` 定义，不会报未定义变量。
    

## 遗留 / 后续（可选，非本次必须）

- **深度解耦**:把 api 移出 `docker_default`（撤掉现网 override 的 `aisales-app` 别名），改让 edge 多接 `amer_sales_assist_default` 网络、回源 `api:8000`。api 彻底脱离 Dify 网络。低风险、可择日单独做。
    
- **若将来拿到第二个公网 IP**:可演进到更彻底的物理隔离（aisales 用自己的 nginx 独占第二个 IP 的 443，Dify 完全不让出 443）。
    

## 会用到的能力

**扫描范围**（本次会话可见）：

- Skills (21): ogsm-report-writer, deep-research, codex:rescue, codex:setup, frontend-design:frontend-design, codex:codex-cli-runtime, codex:codex-result-handling, codex:gpt-5-4-prompting, update-config, keybindings-help, verify, code-review, simplify, fewer-permission-prompts, loop, schedule, claude-api, run, init, review, security-review
    
- MCP namespaces (4): chrome-devtools, firecrawl, jina, tavily
    

**会用**：

- 内置 Read/Bash（只读）：读 `D:\note\...` 笔记 + 本地仓库核对服务名/网络/nginx 配置；执行阶段把命令交给你在服务器上跑。本任务不依赖任何 Skill/MCP——全是服务器侧 nginx/docker 运维，由你在远端执行。
    

**考虑过但不用**：

- jina / tavily / firecrawl / WebFetch（网研 Dify 的 `EXPOSE_NGINX_*` 端口约定）：改用第 0 步 B 直接读服务器真实文件，比查通用模板更准。
    
- chrome-devtools（浏览器实测 dify 控制台 WS）：验证在切换后由你本机浏览器做——内网域名我这侧到不了。