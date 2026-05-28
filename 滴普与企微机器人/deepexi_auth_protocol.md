# Deepexi 私有认证协议文档



• 按你贴的文档，这个 Deepexi 本质上是一个“企业内部专用的 SSO 登录协议”，不是通用标准协议。

  它的核心流程是：

  1. 业务系统把用户重定向到统一认证中心
  2. 认证中心登录成功后，带着 code 和 state 回调业务系统
  3. 业务系统拿 code 去认证中心直接换“用户信息”，然后本地建立登录态

  所以它更像是：

  - “定制化的登录回调协议”
  - 或者“简化版的授权码登录流程”

  而不是完整意义上的 OAuth2。

  更准确地说，文档里拿它和 OAuth2 比，其实有点混淆概念：

  - OAuth2 的核心是“授权”，即客户端拿到 access token 去访问受保护资源
  - 企业 SSO/登录更常对标的是 OIDC，也就是 OAuth2 之上的身份认证层
  - Deepexi 这里做的是“拿 code 直接换用户身份”，所以更接近“私有版 OIDC 登录”，只是没有标准化的 token/id_token/userinfo
    体系

  可以这样理解两者差别：

  Deepexi 的优点：

  - 流程短，接入简单
  - 适合纯内网、同一企业内系统互信
  - 业务系统只关心“这个人是谁”，不用处理 token 生命周期
  - Deepexi 更适合同一个企业内部、只做登录、不做开放授权的场景

  一句话总结：

  Deepexi = 面向企业内部 SSO 的自定义“code 换用户信息”协议
  OAuth2/OIDC = 面向标准化授权/认证生态的通用协议体系

  如果你的场景只是“内部系统单点登录”，Deepexi 这种方案是合理的。
  如果你的场景要做开放平台、细粒度授权、跨系统标准接入，优先 OAuth2 + OIDC。

  如果你愿意，我可以下一步把这套 Deepexi 流程画成一张时序图，再顺手指出它在安全上至少应该补哪些校验。



## 1. 协议概述

Deepexi 是一种简化的私有认证协议，相比标准 OAuth2 协议省略了 token 交换步骤，适用于企业内部 SSO（单点登录）场景。

### 协议特点

- **简化流程**：省略 token 交换步骤，直接从 code 换取用户信息
- **灵活配置**：支持自定义参数映射、请求头、响应字段映射等
- **安全可靠**：支持 state 参数验证，防止 CSRF 攻击
- **易于集成**：业务系统只需配置即可使用，无需关心底层实现

### 与 OAuth2 的对比

| 特性 | OAuth2 | Deepexi |
|------|--------|---------|
| 流程步骤 | 4 步（授权 → 回调 → 换 token → 换用户信息） | 3 步（授权 → 回调 → 换用户信息） |
| Token 交换 | 需要 | 不需要 |
| 适用场景 | 标准 OAuth2 提供商 | 企业内部 SSO |

---

## 2. 认证流程

### 流程图

```
sequenceDiagram
    participant 用户
    participant 业务系统
    participant 提供方登录页
    participant 提供方API

    用户->>业务系统: 1. 请求登录
    业务系统->>提供方登录页: 2. 跳转到 auth_url?redirect_uri=xxx&state=xxx
    用户->>提供方登录页: 3. 输入账号密码登录
    提供方登录页->>业务系统: 4. 跳转到 redirect_uri?code=xxx&state=xxx
    业务系统->>提供方API: 5. 调用 user_info_url?code=xxx
    提供方API->>业务系统: 6. 返回用户信息
    业务系统->>业务系统: 7. 根据用户信息构造 JWT
```

### 详细步骤

| 步骤 | 操作 | 说明 |
|------|------|------|
| 1. 用户请求登录 | 用户点击登录按钮 | 业务系统生成随机 `state` 参数（用于 CSRF 防护） |
| 2. 跳转到登录页面 | 业务系统构建授权 URL | `auth_url?redirect_uri=xxx&state=xxx`，浏览器跳转到提供方登录页 |
| 3. 用户登录 | 用户输入账号密码 | 提供方验证用户身份 |
| 4. 回调到业务系统 | 提供方跳转到 `redirect_uri` | URL 带上 `code` 和 `state` 参数（code 的值由提供方决定） |
| 5. 获取用户信息 | 业务系统提取 `code` | 使用 code 调用 `user_info_url` 获取用户信息 |
| 6. 返回用户信息 | 提供方验证 code 有效性 | 返回用户信息（JSON 格式） |
| 7. 构造 JWT | 业务系统构造 JWT | 根据返回的用户信息完成登录流程 |

---

## 3. 配置说明

在 `oauth_config.yaml` 中添加 Deepexi 提供商配置：

```yaml
auth_config:
  identity_providers:
    - type: deepexi
      id: internal_sso
      name: 企业SSO登录
      logo: https://sso.company.com/logo.png
      enabled: true
      config:
        # 必需配置
        auth_url: https://sso.company.com/oauth/authorize
        user_info_url: https://sso.company.com/api/userinfo

        # 可选配置
        tenant_id: fastagi
        role: user

        # 回调参数映射
        callback_param_mapping:
          code: code
          state: state
          error: error
          error_description: error_description

        # 安全配置
        validate_state: true

        # 请求配置
        user_info_headers:
          Content-Type: application/json

        user_info_params:
          code: ${code}

        # 响应字段映射
        response_mapping:
          user_id: user_id
          username: username
          nickname: nickname
          email: email
          avatar: avatar
          role: role
```

### 配置项详解

#### 必需配置

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `auth_url` | 授权端点（登录页面）URL | `https://sso.company.com/oauth/authorize` |
| `user_info_url` | 用户信息端点 URL，用于用 code 换取用户信息 | `https://sso.company.com/api/userinfo` |

**`auth_url` 注意事项：**
- 业务系统会在此 URL 后自动添加 `redirect_uri` 和 `state` 参数
- 支持在 URL 中使用 `${redirect_uri}` 和 `${state}` 占位符

**`user_info_url` 注意事项：**
- 支持在 URL 中使用 `${code}` 占位符
- 如果不在 URL 中使用占位符，code 会通过 `user_info_params` 传递
