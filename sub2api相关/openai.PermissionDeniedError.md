现在看到真正的问题了。debug 日志显示：

  1. openai_token_cache_get_failed — OpenAI 账户 (id=2) 的 OAuth token 无效或为空
  2. account_select_failed: no available OpenAI accounts — 没有可用的上游 OpenAI 账户
  3. DNS 失败: lookup chatgpt.com: server misbehaving — 容器内无法解析 chatgpt.com

  核心问题不是余额，而是你的上游 OpenAI 账户没有配置有效的 OAuth 登录。 Sub2api 是把 ChatGPT/Claude 的订阅账号转成 API
  的工具，你需要先在管理后台绑定一个有效的 ChatGPT 或 Claude 订阅账号（通过 OAuth 登录授权），它才能转发请求到上游。

  你现在有可用的 ChatGPT 或 Claude 订阅账号吗？如果有，在管理后台 http://localhost:8080
  的"账户管理"页面添加/重新授权即可。