• SSH 地址就是 Git 仓库的另一种克隆地址，给 git clone 用的，不依赖你网页里的登录状态，而是用 SSH 密钥认证。

  常见格式是：

  git@服务器:命名空间/仓库.git

  或者带端口：

  ssh://git@服务器:端口/命名空间/仓库.git

  你这个仓库的 SSH 地址大概率会长这样：

  git@113.78.134.101:dwyane_mo/crm_manage_system_2.0_full.git

  如果服务器 SSH 不是默认 22 端口，就会变成：

  ssh://git@113.78.134.101:端口/dwyane_mo/crm_manage_system_2.0_full.git

  它和 HTTP 地址的区别：

  - HTTP：http://...，通常要用户名/密码或 token
  - SSH：git@...，通常要先在本机配好 SSH key，并把公钥加到 GitLab 账号里

  你可以在仓库页面点 Clone，通常会看到：

  - Clone with SSH
  - Clone with HTTP
