# 网站嵌入 & Token 校验 Q&A

## Q1：网站嵌入是什么意思？

网站嵌入指的是**把 fastagi 的聊天机器人（chatbot）以组件的形式嵌入到第三方业务系统的网页中**。

简单来说就是：你的业务网站右下角多了一个 AI 对话气泡按钮，用户点击后可以直接在你的网站内与 fastagi 智能体对话，而不需要跳转到 fastagi 的页面。

**类比**：类似于很多网站右下角嵌入的在线客服窗口（如 Intercom、LiveChat），只不过这里嵌入的是 AI 智能体。

---

## Q2：为什么需要 Token 校验？

核心问题：**fastagi 和你的业务系统是两套独立的用户体系**。

```
业务系统：用户 A（token_abc）  ←  fastagi 不认识这个 token
fastagi： 用户 A（token_xyz）  ←  业务系统不认识这个 token
```

Token 校验解决的就是**身份打通**的问题：

| 问题 | 如果没有 Token 校验 |
|------|---------------------|
| **身份识别** | fastagi 不知道当前用户是谁，无法提供个性化服务 |
| **安全性** | 任何人都可以伪造请求调用 fastagi 接口，存在滥用风险 |
| **会话隔离** | 无法区分不同用户的聊天记录 |

**校验流程本质上是一次"翻译"：**

1. 用户在业务系统已经登录，拥有业务系统的 token
2. fastagi 拿着这个 token 去问业务系统："这个 token 对应的是谁？"
3. 业务系统回答："这是张三，ID 是 xxx"
4. fastagi 据此生成自己系统的 token，后续用这个 token 通信

这样用户**无需在 fastagi 再登录一次**，实现了单点登录（SSO）的效果。

---

## Q3：网站嵌入是前端还是后端负责？

主要是前端，但不完全是。按职责划分：

**前端工程师负责：**
- 引入 SDK 脚本
- 初始化 ChatBot（配置参数）
- 调用 `sendMessage` 等交互逻辑
- 按钮样式、窗口大小等 UI 调整

**后端工程师负责：**
- 实现 token 校验接口（`/auth/check_token`）
- 返回 `user_name`、`user_id` 等用户信息

**简单说：** 前端负责"嵌"，后端负责"认"。

---

## Q4：对接系统和 fastagi 怎么交互的？

### fastagi 调用 token 校验接口

fastagi 收到前端传来的业务 token 后，会向对接系统发起一个 **HTTP GET 请求**：

```
GET http://{host}:{port}/auth/check_token?token=业务系统用户token
```

这就是一次普通的后端到后端的 HTTP 调用，没有什么特殊协议。

### 对接系统如何返回用户信息

对接系统收到请求后，内部逻辑大致是：

```
收到 token → 查库/查缓存（如 Redis）验证 token → 找到对应用户 → 返回 JSON
```

伪代码示例：

```python
def check_token(token):
    # 1. 从缓存或数据库中查找 token 对应的用户
    user = redis.get(token)  # 或查数据库、解析 JWT 等

    # 2. token 无效或过期
    if not user:
        return {"code": 1, "data": None, "msg": "token无效"}

    # 3. token 有效，返回用户信息
    return {
        "code": 0,
        "data": {
            "user_name": user.name,   # "张三"
            "user_id": user.id        # "xxxxx"
        },
        "msg": ""
    }
```

### fastagi 如何生成自己的 token

业界通用做法是 **JWT（JSON Web Token）**：

```
拿到 user_name + user_id
        ↓
组装 payload：{ user_id, user_name, tenant_id, exp（过期时间）... }
        ↓
用 fastagi 的密钥签名 → 生成 JWT
        ↓
返回给前端
```

前端后续的每次聊天请求都带上这个 JWT，fastagi 验签即可识别用户身份，不需要每次都再去问对接系统。

**整体交互一句话总结：** fastagi 不维护业务系统的用户数据，它通过 HTTP 调对接系统的接口"问一嘴"用户是谁，拿到答案后生成自己的凭证，后续就用自己的凭证通信。

---

## Q5：biz_auth_url 配置是什么意思？

这是 **fastagi-chat-server 的配置文件**，告诉 fastagi 去哪里校验 token。

### 单系统对接

```yaml
biz_auth_url: "http://{host}:{port}/auth/check_token"
```

只对接一个业务系统时，配置一个地址就够了。fastagi 收到 token 后，固定调这个地址去校验。

### 多系统对接

```yaml
biz_auth_urls:
  third_party_system_a: "http://系统A的地址/auth/check_token_for_a"
  third_party_system_b: "http://系统B的地址/auth/check_token_for_b"
```

如果多个业务系统都要嵌入同一个 fastagi 聊天机器人，每个系统的 token 校验接口地址不同，所以需要用 **key-value** 的方式分别配置。

**举个实际例子：**

假设公司有两个内部系统都要接入 fastagi：

```yaml
biz_auth_urls:
  oa: "http://oa.company.com/auth/check_token"
  crm: "http://crm.company.com/auth/check_token"
```

前端初始化时通过 `biz_id` 告诉 fastagi "我是哪个系统"：

```javascript
// OA 系统的前端
new ChatBot({ biz_id: "oa", ... })

// CRM 系统的前端
new ChatBot({ biz_id: "crm", ... })
```

fastagi 根据 `biz_id` 查配置，找到对应的校验地址去调用。

**本质就是一个路由表**——`biz_id` 是 key，校验接口地址是 value。
