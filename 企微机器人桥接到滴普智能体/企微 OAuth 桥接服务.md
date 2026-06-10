• 我按“第三方服务商应用 + Deepexi 接入方”这个最常见场景，直接给你一版可落地的桥接服务设计稿。

• 按“企微第三方服务商应用 ServiceApp + Deepexi 作为消费方”来设计，最稳。

  方案
  做一个独立桥接服务 wecom-sso-bridge，对 Deepexi 暴露两类接口：

  1. GET /oauth/authorize
      - 给 Deepexi 配成 auth_url
      - 入参：redirect_uri、state
      - 动作：
          - 校验 redirect_uri
          - 落库一条登录会话
          - 302 跳转到企微登录页

  https://login.work.weixin.qq.com/wwlogin/sso/login?login_type=ServiceApp&appid={SuiteID}&redirect_uri={bridge_callback}&state={opaque_state}
  2. GET /wecom/callback
      - 企微回调地址
      - 入参：code、state
      - 动作：
          - 校验 state
          - 用 suite_access_token + code 调企微 getuserinfo3rd
          - 拿到 corpid、userid、open_userid
          - 查本地绑定关系，映射成系统用户
          - 生成一次性 bridge_code
          - 302 跳回 Deepexi：{redirect_uri}?code={bridge_code}&state={original_state}
  3. GET /api/userinfo
      - 给 Deepexi 配成 user_info_url
      - 入参：code={bridge_code}
      - 动作：
          - 校验一次性 bridge_code
          - 返回标准用户信息 JSON
  4. GET /oauth/admin-entry
      - 处理企微管理端“业务设置”入口
      - 入参：auth_code
      - 动作：
          - 和 /wecom/callback 类似，区别只是消费的是管理端带来的 auth_code
          - 企微文档说明，开启新的登录授权应用后，这个 auth_code 也可用同一身份接口消费

  时序

  1. Deepexi 跳 auth_url
  2. 桥接服务存 redirect_uri/state
  3. 跳企微登录页
  4. 用户在企微确认登录
  5. 企微回调桥接服务 code/state
  6. 桥接服务换取企微身份
  7. 桥接服务生成 bridge_code
  8. 桥接服务回跳 Deepexi ?code=bridge_code&state=...
  9. Deepexi 调 user_info_url
  10. 桥接服务返回用户 JSON
  11. Deepexi 自己签发 JWT/登录态

  表结构
  sso_login_session

  - id
  - provider = wecom
  - deepexi_redirect_uri
  - deepexi_state
  - bridge_state
  - wecom_code
  - bridge_code
  - status：INIT / WECOM_OK / CONSUMED / FAILED
  - expire_at
  - created_at

  sso_account_binding

  - id
  - provider = wecom
  - corpid
  - open_userid
  - userid
  - local_user_id
  - tenant_id
  - enabled
  - 唯一索引：(provider, corpid, open_userid)

  建议绑定键优先用 corpid + open_userid，不要只用 userid。

  Deepexi 配置

  auth_config:
    identity_providers:
      - type: deepexi
        id: wecom_sso
        name: 企业微信登录
        enabled: true
        config:
          auth_url: https://sso.example.com/oauth/authorize
          user_info_url: https://sso.example.com/api/userinfo
          validate_state: true
          user_info_params:
            code: ${code}
          response_mapping:
            user_id: user_id
            username: username
            nickname: nickname
            email: email
            avatar: avatar
            role: role

  /api/userinfo 返回建议

  {
    "user_id": "u_10086",
    "username": "zhangsan",
    "nickname": "张三",
    "email": "zhangsan@example.com",
    "avatar": "https://cdn.example.com/avatar/u_10086.png",
    "role": "user"
  }

  实现要点

  - suite_access_token 不能写死，桥接服务里要做缓存刷新
  - code、bridge_code 都必须一次性消费，TTL 建议 5 分钟
  - state 必须校验
  - 所有回调和登录域名必须走 HTTPS
  - 企微可信域名必须和实际 redirect_uri 严格一致
  - 管理端入口和网页登录入口都统一落到桥接服务，别让 Deepexi 直接碰企微接口

  接口骨架

  GET /oauth/authorize?redirect_uri=...&state=...
  302 -> WeCom login

  GET /wecom/callback?code=...&state=...
  302 -> {redirect_uri}?code={bridge_code}&state={state}

  GET /oauth/admin-entry?auth_code=...
  302 -> {redirect_uri}?code={bridge_code}&state={state}

  GET /api/userinfo?code={bridge_code}
  200 application/json
