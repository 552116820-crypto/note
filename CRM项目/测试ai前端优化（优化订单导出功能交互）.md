• 如果你说的是之前那版“订单导出交互优化”，入口在销售单列表页，不在报表页。

  怎么体验

  1. 登录 http://localhost:9528/adminVueHtml/#/。
  2. 从左侧菜单进入“销售单列表”。
     说明：这个菜单是后台动态下发的，不是前端静态写死路由，代码在 VueAdmin/src/store/modules/permission.js:22 和
     VueAdmin/src/store/modules/permission.js:79。
  3. 在列表右上角点“导出”，按钮位置在 VueAdmin/src/views/sale_order/sale_order_list.vue:94。
  4. 会弹出“订单导出”窗口，你可以体验这些交互：
      - 字段分组勾选
      - 全选/取消全选
      - 同步/异步导出切换
      - 异步导出的进度态 UI
        代码在 VueAdmin/src/views/sale_order/sale_order_list.vue:651。
  5. 小数据量时用“直接下载”更容易走通；页面默认也是 total <= 2000 预选同步，> 2000 预选异步，逻辑在 VueAdmin/src/views/
     sale_order/sale_order_list.vue:1273。

  当前这套本地环境能体验到哪一步
  我刚实测过：

  - 老的同步导出接口 /SaleOrder/orderExport 是通的，后端在 app/adminApi/controller/order/SaleOrder.php:2118。
  - 前端新交互调用的 3 个接口：
      - /SaleOrder/exportFields
      - /SaleOrder/createExportTask
      - /SaleOrder/exportTaskStatus
        代码在 VueAdmin/src/api/sale_order.js:303
        但我现在从本地 9528 代理实测都是 404。

  所以你现在可以真实体验的是：

  - 导出按钮
  - 导出弹窗
  - 字段选择
  - 全选/取消全选
  - 导出方式切换
  - 同步导出下载

  异步导出这条链路目前在你本地还跑不通。不过字段列表接口即使失败，前端也会回退到默认字段，所以弹窗本身还能看，代码在
  VueAdmin/src/views/sale_order/sale_order_list.vue:1288。

  如果你要，我可以继续把这 3 个后端接口和路由补上，让你在 Docker 里把异步导出也完整体验起来。


• 已经把左侧菜单里的“销售单列表”入口改到新版页面了。

  你现在这样体验：

  1. 退出登录后重新登录，或者强制刷新一次让菜单重新拉取。
  2. 点击左侧菜单“销售单列表”，它会打开 #/sale_order/sale_order_list_v2（也可以直接访问 http://localhost:9528/adminVueHtml/#/sale_order/sale_order_list_v2）。
  3. 进页面后右上角会有绿色“导出”按钮（下载图标）。

  改动点在后端菜单返回重写 app/adminApi/controller/system/AdminUser.php 和前端新增包装页 VueAdmin/src/views/sale_order/sale_order_list_v2.vue，创建订单返回也已跳到 v2：VueAdmin/src/views/sale_order/
  create.vue:2170。如果你点菜单还是进不去，把当前地址栏的 #/... 路径发我。


• 我刚才在查“为什么这页没数据”，结论是我前面把入口切错了。

  你看到的空页，其实是财务核单那套页面；你这个账号走那条接口时，后端权限会被收缩成空集，SQL 实际会落成 WHERE 0 = 1，所以
  必然查不出数据。现在我已经把 #/sale_order/sale_order_list_v2 改成真正的销售单列表数据源了，文件在 VueAdmin/src/views/
  sale_order/sale_order_list_v2.vue，并且把导出弹窗交互挂到了 VueAdmin/src/views/sale_order/sale_order_list_two.vue:106
  和 VueAdmin/src/views/sale_order/sale_order_list_two.vue:188。

  你现在刷新一下页面，再进“销售单列表”试试。正常的话，这次应该走原来的销售单数据，同时右上角能看到“导出”按钮和导出弹窗。
  补充一点：同步导出和弹窗交互现在可体验；如果你手动切到“异步导出”，本地后端那 2 个任务接口还没接上，可能会提示创建任务
  失败。你刷新后告诉我这次列表里有没有数据。
