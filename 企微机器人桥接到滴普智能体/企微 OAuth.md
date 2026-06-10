• 看下来，这件事能做，且最稳的做法是把企微登录包装成一个“Deepexi 认证提供方”。

  结论
  Deepexi 协议本质上是简化版 OAuth：不换 token，只走 auth_url -> 回调 code -> user_info_url。
  企微单点登录也能产出 code，但企微后端换身份时还依赖动态的 suite_access_token/应用类型差异，所以不建议把企微接口直接硬
  配进 Deepexi；推荐加一层 SSO 适配服务。

  Deepexi 协议怎么用
  Amer 那份 auth_guide_id 文档里，Deepexi 的核心就是这 3 步：

  1. 业务系统跳到提供方 auth_url，并带上 redirect_uri、state
  2. 提供方登录成功后回跳 redirect_uri?code=xxx&state=xxx
  3. 业务系统再用这个 code 调 user_info_url 换用户信息，然后由业务系统自己构造 JWT/登录态

  文档里还明确了这些点：

  - 提供方至少要提供两个能力：auth_url 和 user_info_url
  - 支持 callback_param_mapping，所以提供方回调参数名不一定非得叫 code
  - 支持 user_info_headers、user_info_params、response_mapping
  - code 要一次性、短时有效，建议 5 到 10 分钟
  - state 要校验，防 CSRF
  - 全链路应走 HTTPS

  企微单点登录怎么接
  如果你们是“第三方/服务商应用”场景，官方流程是：

  - 前端登录入口用企微 Web 登录
      - 登录链接是
        https://login.work.weixin.qq.com/wwlogin/sso/login?...
      - 关键参数：
          - login_type=ServiceApp
          - appid=SuiteID
          - redirect_uri
          - state
  - 用户授权后，企微回调
    redirect_uri?code=CODE&state=STATE
  - 服务端再调用
    GET https://qyapi.weixin.qq.com/cgi-bin/service/auth/getuserinfo3rd?suite_access_token=...&code=...
  - 返回的关键身份字段是：
      - corpid
      - userid
      - open_userid

  另外还有一个官方支持的入口：

  - 管理员从企微管理端第三方应用“业务设置”进入时，企微会带 auth_code
  - 文档说明：如果已开启新的“登录授权应用”，这个 auth_code 也可以用同一个“获取登录用户身份”接口消费

  如果你们是“企业自建/代开发应用”而不是第三方服务商：

  - 登录入口要改成 login_type=CorpApp
  - appid=CorpID
  - agentid=AgentID
  - 不能用 getuserinfo3rd，要走企微自建应用对应的“获取访问用户身份”能力

  推荐方案
  下面这个方案是我基于两边文档做的工程化推断，也是最可落地的方案：

  - 做一个 wecom-sso-bridge 适配层，对外表现成 Deepexi provider
  - GET /oauth/authorize
      - 这是给 Deepexi 配的 auth_url
      - 接收 Deepexi 传来的 redirect_uri 和 state
      - 服务端保存上下文后，302 到企微登录页
  - GET /wecom/callback
      - 接收企微返回的 code 或管理端场景的 auth_code
      - 服务端调用企微接口换身份
      - 根据 corpid + open_userid 做本地账号绑定
      - 生成你们自己的“一次性 bridge_code”
      - 再跳回 Deepexi：redirect_uri?code=bridge_code&state=原state
  - GET/POST /api/userinfo
      - 这是给 Deepexi 配的 user_info_url
      - Deepexi 拿 bridge_code 来换标准用户 JSON
      - 返回 Deepexi 所需字段，Deepexi 自己再构造 JWT

  Deepexi 侧配置可以做成这样：

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

  返回给 Deepexi 的用户信息建议这样归一化：

  - user_id：推荐用本地用户 ID，或者至少用 wecom:{corpid}:{open_userid}
  - username：可映射本地账号名
  - nickname/email/avatar：企微这个接口默认不给全量资料，建议从你们本地用户中心补
  - role：建议从你们本地 RBAC 取，不要直接依赖企微字段

  为什么不建议“直接把企微接口填进 Deepexi 配置”

  - suite_access_token 是动态的，不能长期写死在 user_info_url
  - 企微返回的是“身份标识”，不是完整业务用户资料
  - 管理端 auth_code 和网页登录 code 两种入口，直接硬配会很别扭
  - 你们大概率还需要做本地账号绑定、租户隔离、角色映射

  落地注意点

  - 第三方服务商场景，绑定键优先用 corpid + open_userid，不要只用裸 userid
  - redirect_uri 域名必须和企微可信域名严格匹配，否则会报 50001 或登录域名校验错误
  - code 一次性使用，5 分钟内有效，桥接层自己的 bridge_code 也建议同样处理
  - 如果页面是在企微内置 WebView 打开，官方建议走 OAuth 授权登录，不要走 Web 登录组件
  - 企微桌面端快速登录还有版本、HTTPS、JSSDK 版本限制
