# HTTPS 上线最终小结 + “慢”的根因排查（结论：DNS，非 HTTPS）

> 目标域名：`https://aisales.amer.cn`（替代旧入口 `http://8.138.104.14:8080`）
> 服务器：`8.138.104.14`（`root@AI-appliancce-server`，与 Dify 同一台）
> 完成日期：2026-06-18

---

## 0. 一句话结论

1. **HTTPS 上线成功**：`https://aisales.amer.cn` 由 Dify 的 nginx 在 443 终结 TLS，直接反代到 amer 的 api 容器，正常可用（`/healthz`、`/api/v1/leads/score-and-info` 实测 200）。
2. **“感觉慢”和 HTTPS / nginx / 证书无关**，是两件互不相干的事叠加：
   - ① **客户端 DNS 解析慢 ~12 秒**——你笔记本当前用的 DNS 服务器的问题，换成 `223.5.5.5` 后只要 0.4s。
   - ② **后端冷算 ~10 秒**——score-and-info 首次地理候选查询，5 分钟缓存内再查就秒回（~17ms）。
   - HTTPS 本身的额外开销只有 **~1.4s**（TLS 握手 + DNS 解析），且会被浏览器 keep-alive / DNS 缓存摊掉。

---

## 1. 最终拓扑

```
浏览器 ──HTTPS 443──▶ aisales.amer.cn (→ 8.138.104.14)
        │                       （和 dify.amer.cn 同一个 IP，靠 SNI 分流）
        ▼
   Dify nginx (docker-nginx-1)   443 终结 TLS，按 server_name 分流
        │  proxy_pass http://aisales-app:8000   ← 直连 api 容器（绕开 amer 自己的 nginx）
        ▼
   amer api 容器 (amer_sales_assist-api-1, uvicorn --forwarded-allow-ips *)

旧入口 http://8.138.104.14:8080（amer nginx）仍保留，内部联调用；正式对外用 aisales.amer.cn
```

为什么“直连 api 容器”而不是“经 amer nginx”：少一跳、更简单。唯一代价是 amer nginx 注入的 `X-Request-ID` 没了，但 app 自己会兜底生成 UUID（`app/shared/middleware.py:29`），且 app 不读 `X-Real-IP`，功能零影响。

---

## 2. 这台服务器的真实坐标（交接 / 排查必备）

| 项 | 值 |
|---|---|
| app 仓库目录 | `/root/amer_sales_assist` |
| api 容器 | `amer_sales_assist-api-1`（expose 8000，另绑 127.0.0.1:8000 给旧 lead 前端） |
| amer nginx 容器 | `amer_sales_assist-nginx-1`（`0.0.0.0:8080->80`） |
| Dify nginx 容器 | `docker-nginx-1`（`0.0.0.0:443/80`，TLS 边缘） |
| Dify docker 网络 | `docker_default` |
| api 在 Dify 网的别名 | `aisales-app` |
| 证书（宿主机） | `/root/dify/docker/nginx/ssl/aisales.amer.cn.{pem,key}` |
| 证书（容器内路径） | `/etc/ssl/aisales.amer.cn.{pem,key}`（mount 映射，已证实有效） |
| nginx 站点配置 | `/root/dify/docker/nginx/conf.d/aisales.amer.cn.conf` |
| compose override | `/root/amer_sales_assist/docker-compose.override.yml`（已在 .gitignore） |
| 证书类型 / 有效期 | DigiCert DV（**公网可信**，fullchain 2 张）；**2026-06-17 → 2026-09-14** |
| DNS | `aisales.amer.cn  A → 8.138.104.14`，TTL 600，托管在阿里云 AliDNS |
| CORS | 服务器 `.env` 未钉死（默认 `*`），**无需改** |

---

## 3. 实际执行的步骤（可复现 / 可回滚）

### Step 1 — 上传证书并校验（复制粘贴法）
证书 `.pem`/`.key` 在本地，用“粘贴进文件”上传到 `/root/dify/docker/nginx/ssl/`，然后：
```bash
chmod 600 /root/dify/docker/nginx/ssl/aisales.amer.cn.key
chmod 644 /root/dify/docker/nginx/ssl/aisales.amer.cn.pem
# 配对校验：两行 md5 必须一致（实测一致 ✅）
openssl x509 -noout -pubkey -in /root/dify/docker/nginx/ssl/aisales.amer.cn.pem | openssl md5
openssl pkey  -pubout      -in /root/dify/docker/nginx/ssl/aisales.amer.cn.key | openssl md5
# fullchain：≥2 才含中间证书（实测 2 ✅）
grep -c "BEGIN CERTIFICATE" /root/dify/docker/nginx/ssl/aisales.amer.cn.pem
```

### Step 2 — 让 Dify nginx 能找到 api（零停机）
用运行时 `network connect`，不重建容器、`:8080` 不断：
```bash
docker network connect --alias aisales-app docker_default amer_sales_assist-api-1
docker exec docker-nginx-1 getent hosts aisales-app    # 返回 172.x IP 即成功（实测 172.24.0.12 ✅）
```
> ⚠️ 这是**临时**挂载，重建容器/重新部署后会丢——持久化见 Step 6 / 待办②。

### Step 3 — 写 nginx 站点配置并重载
`/root/dify/docker/nginx/conf.d/aisales.amer.cn.conf` 内容（**当前生效版本**）：
```nginx
# http → 全部 301 到 https
server {
    listen 80;
    server_name aisales.amer.cn;
    return 301 https://$host$request_uri;
}
# 443 终结 TLS，反代到 api 容器
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
```
```bash
docker exec docker-nginx-1 nginx -t            # 必须 ok / successful
docker exec docker-nginx-1 nginx -s reload     # 对 dify.amer.cn 零影响（fail-safe）
```

### Step 4 — 验证（DNS 没切也能本地验）
```bash
# 路由通不通（强制指本机）
curl -ki --resolve aisales.amer.cn:443:127.0.0.1 https://aisales.amer.cn/healthz   # 实测 200 ✅
# 证书链是否公网可信（去掉 -k，仍 200 即可信）✅
curl -i  --resolve aisales.amer.cn:443:127.0.0.1 https://aisales.amer.cn/healthz
```

### Step 5 — DNS（已就绪）
`aisales.amer.cn` 的 A 记录已指向 `8.138.104.14`（TTL 600，AliDNS）。443 安全组本就为 Dify 开着，无需额外动。

### Step 6 — 持久化 override（让网络配置不随重建丢失）
`/root/amer_sales_assist/docker-compose.override.yml`（**仅本机，已被 .gitignore，勿提交**）：
```yaml
services:
  api:
    networks:
      default: {}                 # 保留：继续和 redis / 本项目 nginx 通信
      dify:
        aliases: [aisales-app]     # Dify nginx 用这名字找 api
networks:
  dify:
    external: true
    name: docker_default
```
> 终端粘贴会吞缩进，所以当时是用单行 JSON 写的（JSON 也是合法 YAML，docker compose 认）：
> `printf '%s\n' '{"services":{"api":{"networks":{"default":{},"dify":{"aliases":["aisales-app"]}}}},"networks":{"dify":{"external":true,"name":"docker_default"}}}' > /root/amer_sales_assist/docker-compose.override.yml`

---

## 4. 踩过的坑（教训）

1. **服务名是 `api` 不是 `app`**：`app` 只是 uvicorn 的模块名，compose 服务叫 `api`。原 plan 写错会直接挂。
2. **nginx 变量名前后要一致**：`set $x ...; proxy_pass http://$x;`，原 plan 一处拼成 `leads_upstream`、另一处 `aisales_upstream`，`nginx -t` 直接报错。
3. **终端粘贴会吞字符 / 吞缩进**：heredoc 多次把 YAML 缩进吃没（`mapping key "dify" already defined`）。**对策：能用单行就单行；YAML 改用 JSON / flow 风格（无缩进依赖）；证书/配置用 `printf` 一次性写。**
4. **`proxy_buffering` 不能重复**：`include proxy.conf;` 里已有 `proxy_buffering off;`，再加一行 `proxy_buffering on;` → nginx 报 `directive is duplicate`（不是“后者覆盖”）。而且这条对 14KB 小响应根本没用——是排查时的弯路，已撤销。
5. **amer nginx 的 `proxy_pass http://api:8000;` 没有 resolver**：它把 api 的 IP 缓存死了。一旦 `docker compose up -d` 重建 api（换 IP），`:8080` 会 502，**重建后必须紧跟 `docker compose restart nginx`** 让它重新解析。

---

## 5. “慢”的排查全过程与证据

| 阶段 | 做法 | 结果 / 结论 |
|---|---|---|
| 怀疑 buffering | 看 Dify `proxy.conf` 有 `proxy_buffering off` | 红鲱鱼：响应仅 ~14KB，本机传输瞬间完成；且加 `on` 触发 duplicate 报错 |
| 本机 A/B | curl 真实 POST（本机环回） | HTTP 000004 冷算 **3.06s**；HTTPS 000004（命中缓存）**0.017s**；HTTPS 000098 冷算 **1.28s** → 缓存真实存在（180×），但解释不了 Postman 现象 |
| healthz A/B（Postman） | 对比无后端开销的 `/healthz` | HTTP 552ms vs HTTPS 1.93s → HTTPS 传输开销仅 **~1.4s**，不是 10s；`getent` 无 AAAA，排除 IPv6 |
| Postman 瀑布图（同 body，HTTPS 两次） | 看分阶段耗时 | 第一次 21.63s = **DNS 11.04s** + 冷算 TTFB 9.95s + SSL 0.64s；第二次 660ms（DNS/连接/缓存全命中）→ **DNS 是大头** |
| PowerShell DNS 实测 | `Resolve-DnsName` 对比 | `aisales` 默认解析器 **12.04s**；`dify` 也 **12.04s**（→ 非子域问题）；`aisales` 走 `8.8.8.8` 仅 **0.44s**（→ 权威 DNS 与 A 记录都正常）。**实锤：笔记本默认 DNS 服务器超时重试** |

**代码佐证**（`app/lead`）：
- score-and-info 缓存键 = `(employee_code, lng, lat, radius_km)`，TTL 默认 **300s**（`cache/store.py:56`、`config/settings.py:64`）。
- 冷算（地理候选查询）固有耗时 **~7–10s**（`config/settings.py:71` 注释明示）；score-and-info 不调大模型（LLM 在接口四 `detail_cache`，TTL 3600s）。

---

## 6. 待办 / 收尾（重要，别漏）

- [ ] **① 确认 nginx 配置干净**（排查时加过的 `proxy_buffering on;` 若有残留会挡住下次 reload）：
  ```bash
  docker exec docker-nginx-1 nginx -t
  # 若报 duplicate：
  sed -i '/proxy_buffering on;/d' /root/dify/docker/nginx/conf.d/aisales.amer.cn.conf
  docker exec docker-nginx-1 nginx -t
  ```
- [ ] **② 持久化网络（重要）**：当前 api 接入 `docker_default` 是运行时临时挂的，**重建容器/重新部署/可能重启后会丢，aisales.amer.cn 会断**。挑低峰执行（会重建 api、几秒抖动）：
  ```bash
  cd /root/amer_sales_assist
  docker compose config >/dev/null && echo OK          # 先验 override 语法
  docker compose up -d && docker compose restart nginx  # 重建 api 后必须紧跟 restart nginx（见坑#5）
  # 复验
  docker exec docker-nginx-1 getent hosts aisales-app
  curl -ki --resolve aisales.amer.cn:443:127.0.0.1 https://aisales.amer.cn/healthz
  curl -sf http://127.0.0.1:8080/healthz >/dev/null && echo ":8080 正常"
  ```
- [ ] **③ 证书续期**：DigiCert DV 90 天，**2026-09-14 前**手动续（非 certbot 托管）。重复 Step 1 粘贴新 pem/key + `docker exec docker-nginx-1 nginx -s reload`。**建议在 9 月初设日历提醒。**
- [ ] **④ 客户端 DNS**：你笔记本默认 DNS 解 amer.cn 要 12s。
  - 个人/测试机：把 DNS 改成 `223.5.5.5,223.6.6.6`（阿里公共 DNS，与 amer.cn 同源、国内快）。
  - **若 ~50 个业务员共用同一个慢的公司 DNS** → 他们首次访问也会卡 ~12s（之后按 TTL 600 缓存 10 分钟）→ 需 **公司 DNS 管理员**把上游转发指到快的解析器。
- [ ] **⑤（可选改进）** 给 amer nginx 也加 resolver，让它在 api 换 IP 时自愈，省掉坑#5 那个“重建后手动 restart nginx”。改 `app/amer_sales_assist` 仓库的 `nginx/default.conf`：把 `proxy_pass http://api:8000;` 换成 `resolver 127.0.0.11 valid=10s; set $u api:8000; proxy_pass http://$u;`。

---

## 7. 给 DNS 管理员的一句话（如需）

> `aisales.amer.cn` 的 A 记录（→8.138.104.14，AliDNS）本身正确，公共解析器（8.8.8.8）0.4s 就能解出。但我们内网/笔记本默认 DNS 解 `amer.cn`（含 `dify.amer.cn`）普遍要 ~12s（超时重试）。请检查内网 DNS 解析器对外部域名（特别是 AliDNS 托管的 amer.cn）的转发/可达性，建议上游转发到 `223.5.5.5`。

---

*排查链路：买缓冲(误) → 缓存(部分对) → 传输开销(仅 1.4s) → DNS(12s, 真因) + 后端冷算(10s)。HTTPS / nginx / 证书部署本身完全正确，无需改动。*
