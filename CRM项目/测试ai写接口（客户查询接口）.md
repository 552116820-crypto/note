• 我总共做了这些事：

  - 梳理项目结构，确认客户模块、路由方式、返回结构和前端调用点，最终决定不新建接口，而是增强现有的 app/adminApi/
    controller/customer/Customer.php:66 里的 customerList()。
  - 检查现有 customerList() 的能力，确认它原本已有分页和大量筛选，但排序是写死的，前端传了 sort 实际后端没接。
  - 在 app/adminApi/controller/customer/Customer.php:267 给客户查询接口补上了排序支持，新增了：
      - app/adminApi/controller/customer/Customer.php:351
      - app/adminApi/controller/customer/Customer.php:382
  - 给排序做了白名单，只允许指定字段排序，非法字段自动回退到默认 -id，避免任意 order by 注入。
  - 修正了两个查询问题：
      - tax_register_status=3 的判断修正为“工商税号为空”(app/adminApi/controller/customer/Customer.php:85)
      - description 查询改成分组条件，避免 whereOr 破坏其他筛选(app/adminApi/controller/customer/Customer.php:103)
  - 为了保持页面默认展示顺序不变，把依赖该接口的前端默认排序从 +id 改成了 -id：
      - VueAdmin/src/views/customer/list.vue:501
      - VueAdmin/src/views/customer/clerk_customer_list.vue:391
      - VueAdmin/src/views/customer/clerk_customer_list_new.vue:393
      - VueAdmin/src/views/common/role.vue:108
      - VueAdmin/src/views/common/user.vue:108
  - 输出了接口文档到 docs/customer-query-api.md:1，包含分页、筛选、排序字段、返回结构、错误示例和调用示例。
  - 在你起好的 Docker 环境里做了真实联调测试：
      - 真实登录拿 X-Token
      - 实测分页
      - 实测排序
      - 实测 id、full_name、description/scope 筛选
      - 实测 full_name 长度不足 2 的报错
  - 最后还帮你解释了测试时的两个占位符问题：
      - {host} 要换成真实地址
      - {token} 要填登录接口返回的 data.token


 加了哪些代码

  - 在 app/adminApi/controller/customer/Customer.php:267 给 customerList() 加了排序入口，支持 sort=+id、sort=-created 这种格式。
  - 新增了排序解析方法 app/adminApi/controller/customer/Customer.php:351，做两件事：
      - 解析 +field / -field
      - 对非法排序字段自动回退到默认 customer.id desc
  - 新增了排序白名单方法 app/adminApi/controller/customer/Customer.php:382，目前允许这些字段排序：
      - id
      - created
      - full_name
      - code
      - ncc_code
      - type
      - is_protect
      - salesman_customer_created
      - bs_employee_name
      - bs_department_name
      - payment_ncc_name
      - payment_name
  - 修正了 tax_register_status=3 的判断，app/adminApi/controller/customer/Customer.php:85 现在查的是 enterprise.creditNo is null，也就是“工商税号为空”。
  - 修正了 description 筛选写法，app/adminApi/controller/customer/Customer.php:103 改成了分组条件，避免原来的 whereOr 把后面的筛选条件冲掉。
  - 为了不让前端默认排序被反转，把几个依赖这个接口的页面默认值从 +id 改成了 -id：
      - VueAdmin/src/views/customer/list.vue:501
      - VueAdmin/src/views/customer/clerk_customer_list.vue:391
      - VueAdmin/src/views/customer/clerk_customer_list_new.vue:393
      - VueAdmin/src/views/common/role.vue:108
      - VueAdmin/src/views/common/user.vue:108

  接口文档我放在了 docs/customer-query-api.md:1。



现在我有一个接口开发任务需要你完成，请你开发客户查询接口（含分页、筛选、排序，输出接口文档）

• 完成情况

  - 后端客户查询接口已在 app/adminApi/controller/customer/Customer.php:267 补齐排序能力，支持 sort=+field/-field，并做了
    排序白名单解析，具体字段映射在 app/adminApi/controller/customer/Customer.php:350。
  - 顺手修了两个筛选问题：tax_register_status=3 现在会正确筛“工商税号为空”(app/adminApi/controller/customer/
    Customer.php:85)，description 查询也改成了分组 OR，避免把后续筛选条件冲掉(app/adminApi/controller/customer/
    Customer.php:103)。
  - 为了不改变现有页面默认展示顺序，把依赖这个接口的默认排序统一改成了 -id：VueAdmin/src/views/customer/list.vue:501、
    VueAdmin/src/views/customer/clerk_customer_list.vue:391、VueAdmin/src/views/customer/
    clerk_customer_list_new.vue:393、VueAdmin/src/views/common/role.vue:108、VueAdmin/src/views/common/user.vue:108。

  接口文档已输出到 docs/customer-query-api.md:1，里面包含调用地址、鉴权方式、分页/筛选参数、排序字段白名单、返回示例和
  curl 示例。

  验证

  - 已确认 customer 和 common 相关页面里不再保留默认 sort: '+id'。
  - 当前环境没有 php 可执行文件，没法跑 php -l 或实际接口联调测试。



• 符合要求。我已经通过真实 HTTP 请求在 Docker 环境里做了端到端验证，访问地址是
  http://127.0.0.1:8080/adminApi/Customer/customerList，先走登录接口拿到 X-Token，再实际请求客户查询接口。

  实测结果

  - 分页通过：page=1&limit=5 返回 5 条，当前测试账号权限范围内 total=151；page=2&limit=5 也返回 5 条，且和第 1 页无重
    复。
  - 排序通过：
      - 默认排序为 -id，返回 332397, 332050, 331411, 330760, 330330
      - sort=+id 返回升序 15223, 15239, 16175, 16236, 19242
      - sort=+created 返回 created 升序
      - 非法排序字段 sort=+drop_table 会回退到默认 -id
  - 筛选通过：
      - id=332397 只返回目标客户
      - full_name=延锋 能命中目标客户，返回 28 条
      - description=产销 能命中目标客户 275863，说明 description/scope 联合筛选正常
  - 边界校验通过：
      - full_name=延 返回 50000，错误信息为“客户名称长度小于2”

  当前实测实现对应后端在 app/adminApi/controller/customer/Customer.php:66，接口文档在 docs/customer-query-api.md:1。




› {token} 这个填什么


• 填登录接口返回的 JWT。

  先调用登录接口：

  curl -X POST 'http://127.0.0.1:8080/adminApi/Login/accountLogin' \
    -H 'Content-Type: application/x-www-form-urlencoded' \
    --data 'username=15050276771&password=123456'

  返回里会有：

  {
    "code": 20000,
    "data": {
      "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiIiLCJhdWQiOiIiLCJpYXQiOjE3NzM4ODY0OTgsIm5iZiI6MTc3Mzg4NjQ5OCwiZXhwIjoxNzczOTcyODk4LCJkYXRhIjp7InRva2VuX3ZhbGlkaXR5X3RpbWUiOjE3NzM5NzI4OTgsImFkbWluVG9rZW4iOiI3ZjA0ZDIzODk4MzcxOWJkNDM1ZmZiOGZkNTAxMDlmZCJ9fQ.6WKMAteKUYyiHSLCl2e_oQl92dOW_PfpMS8XjH37irQ"
    }
  }

  把这个 data.token 的值填到请求头 X-Token 里：

  curl -G 'http://127.0.0.1:8080/adminApi/Customer/customerList' \
    -H 'X-Token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...' \
    --data-urlencode 'page=1' \
    --data-urlencode 'limit=20'

  注意：

  - 不是填 adminToken
  - 不是填数据库里的 token 字段
  - 是登录接口响应里的 data.token

  如果你要，我可以直接给你一条“登录并提取 token，再调用客户查询”的可执行命令。




› {token} 这个填什么


• 填登录接口返回的 JWT。

  先调用登录接口：

  curl -X POST 'http://127.0.0.1:8080/adminApi/Login/accountLogin' \
    -H 'Content-Type: application/x-www-form-urlencoded' \
    --data 'username=15050276771&password=123456'

  返回里会有：

  {
    "code": 20000,
    "data": {
      "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."
    }
  }

  把这个 data.token 的值填到请求头 X-Token 里：

  curl -G 'http://127.0.0.1:8080/adminApi/Customer/customerList' \
    -H 'X-Token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...' \
    --data-urlencode 'page=1' \
    --data-urlencode 'limit=20'

  注意：

  - 不是填 adminToken
  - 不是填数据库里的 token 字段
  - 是登录接口响应里的 data.token



```json
{

    "code": 20000,

    "message": "",

    "time": "2026-03-19 09:06:29",

    "data": {

        "items": [

            {

                "id": 332397,

                "code": null,

                "ncc_code": "250316934",

                "type": 2,

                "full_name": "延锋彼欧汽车外饰系统有限公司西安分公司",

                "is_protect": 1,

                "created": 1753342931,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": null,

                "tax_register": "91610132MAEMN7KC2U",

                "payment_method_id": 17,

                "street": "陕西省西安市经济技术开发区渭阳路33号1栋",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": null,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1753342931,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "款到发货",

                "payment_name": "款到发货,开收据",

                "create_employee_name": "岑德姑",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 332050,

                "code": null,

                "ncc_code": "220316612",

                "type": 1,

                "full_name": "延锋彼欧(辽宁)汽车外饰系统有限公司",

                "is_protect": 1,

                "created": 1753068525,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 39,

                "tax_register": "91210104MA7GTW413G",

                "payment_method_id": 18,

                "street": "辽宁省沈阳市大东区轩顺路6号",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": null,

                "print_template": "",

                "hide_type": "A",

                "delivery_note_code": null,

                "description": "",

                "salesman_customer_created": 1753068525,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结90天",

                "payment_name": "月结90天,开收据",

                "create_employee_name": "岑德姑",

                "creditNo": null,

                "scope": null,

                "type_str": "成交",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 331411,

                "code": "",

                "ncc_code": "230316045",

                "type": 2,

                "full_name": "常州市愉宝传动科技有限公司",

                "is_protect": 1,

                "created": 1752462928,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 11,

                "tax_register": "91320413MACQ9AFUXK",

                "payment_method_id": 0,

                "street": "江苏省常州市金坛区鑫城大道2898号",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": null,

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1752463164,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": null,

                "payment_name": null,

                "create_employee_name": "钱伟伟",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 330760,

                "code": "",

                "ncc_code": "240315449",

                "type": 2,

                "full_name": "裕钦金属表面科技（泰州）有限公司",

                "is_protect": 1,

                "created": 1751620033,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 11,

                "tax_register": "91321283MAE58UCN76",

                "payment_method_id": 17,

                "street": "泰兴市滨江镇西江路8号102-3",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1751620180,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "款到发货",

                "payment_name": "款到发货,开收据",

                "create_employee_name": "钱伟伟",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 330330,

                "code": null,

                "ncc_code": "210315115",

                "type": 2,

                "full_name": "东莞市兴为智能科技有限公司",

                "is_protect": 1,

                "created": 1751341529,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": null,

                "tax_register": "91441900MA56936D3F",

                "payment_method_id": 9,

                "street": "广东省东莞市常平镇桥梓路25号5栋601室",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1751342514,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结60天",

                "payment_name": "月结60天,开13%增值税普通发票",

                "create_employee_name": "岑德姑",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 326473,

                "code": "",

                "ncc_code": "240311639",

                "type": 2,

                "full_name": "裕钦精密拉深技术（湖州）有限公司",

                "is_protect": 1,

                "created": 1747196748,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 11,

                "tax_register": "91330522MAE6R9TUXY",

                "payment_method_id": 17,

                "street": "浙江省湖州市长兴县煤山镇国家级开发区绿色制造产业园浙能产业园创新工场",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1747196824,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "款到发货",

                "payment_name": "款到发货,开收据",

                "create_employee_name": "钱伟伟",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 322639,

                "code": "",

                "ncc_code": "110308229",

                "type": 2,

                "full_name": "浙江欧博电子有限公司",

                "is_protect": 1,

                "created": 1742528601,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 54,

                "tax_register": "9133018358652314XL",

                "payment_method_id": 0,

                "street": "杭州富阳区银湖街道银湖村祝家交界岭99-10号1楼",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1742528886,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": null,

                "payment_name": null,

                "create_employee_name": "黄瑶",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 316837,

                "code": "",

                "ncc_code": "210303095",

                "type": 2,

                "full_name": "浙江优格科技有限公司",

                "is_protect": null,

                "created": 1731938222,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 53,

                "tax_register": "91330521MA2JJJPG33",

                "payment_method_id": 0,

                "street": "浙江省湖州市德清县禹越镇星河路750号",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 1,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1761141627,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": null,

                "payment_name": null,

                "create_employee_name": "周金铃",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 313648,

                "code": null,

                "ncc_code": "240300206",

                "type": 2,

                "full_name": "天空之星科技（东莞市）有限公司",

                "is_protect": 1,

                "created": 1726714556,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": null,

                "tax_register": "91441900MADKM8YG8E",

                "payment_method_id": 17,

                "street": "广东省东莞市虎门镇怀德路243号2栋204室",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1726732335,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "款到发货",

                "payment_name": "款到发货,开收据",

                "create_employee_name": "陈金杏",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 312336,

                "code": null,

                "ncc_code": "230299022",

                "type": 2,

                "full_name": "苏州裕钦精密成型有限公司",

                "is_protect": 1,

                "created": 1724656270,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 42,

                "tax_register": "91320581MAD2CMEU23",

                "payment_method_id": 17,

                "street": "江苏省苏州市常熟市古里镇镇南路29号",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": null,

                "print_template": "",

                "hide_type": "A",

                "delivery_note_code": null,

                "description": "",

                "salesman_customer_created": 1724656270,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "款到发货",

                "payment_name": "款到发货,开收据",

                "create_employee_name": "陈金杏",

                "creditNo": null,

                "scope": null,

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 306230,

                "code": null,

                "ncc_code": "240293607",

                "type": 1,

                "full_name": "日照凯瑞金属科技有限公司",

                "is_protect": 1,

                "created": 1715914541,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 52,

                "tax_register": "91371102MADGXLC17U",

                "payment_method_id": 9,

                "street": "山东省日照市东港区后村镇曙光路66号科创中心1021室",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": null,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1715916074,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结60天",

                "payment_name": "月结60天,开13%增值税普通发票",

                "create_employee_name": "孙慧玲",

                "creditNo": null,

                "scope": null,

                "type_str": "成交",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 297794,

                "code": null,

                "ncc_code": "230285974",

                "type": 1,

                "full_name": "安徽华迎汽车零部件有限公司",

                "is_protect": 1,

                "created": 1701072955,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 31,

                "tax_register": "91340422MA8QMKGH0K",

                "payment_method_id": 9,

                "street": "安徽省淮南市寿县新桥国际产业园科技大道9号",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": "",

                "hide_type": "A",

                "delivery_note_code": null,

                "description": "",

                "salesman_customer_created": 1701073049,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结60天",

                "payment_name": "月结60天,开13%增值税普通发票",

                "create_employee_name": "陈金杏",

                "creditNo": null,

                "scope": null,

                "type_str": "成交",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 293909,

                "code": null,

                "ncc_code": "220282555",

                "type": 1,

                "full_name": "岳阳龙文精密机械有限公司",

                "is_protect": 1,

                "created": 1695009735,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 52,

                "tax_register": "91430600MAC185W438",

                "payment_method_id": 9,

                "street": "湖南省城陵矶新港区哈工大机器人(岳阳)军民融合创新中心",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 0,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1695010021,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结60天",

                "payment_name": "月结60天,开13%增值税普通发票",

                "create_employee_name": "陈金杏",

                "creditNo": null,

                "scope": null,

                "type_str": "成交",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 275863,

                "code": "",

                "ncc_code": "170266696",

                "type": 2,

                "full_name": "东莞市乾德智能科技有限公司",

                "is_protect": 1,

                "created": 1668657101,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 0,

                "tax_register": null,

                "payment_method_id": 0,

                "street": "广东省东莞市常平镇常田路426号2栋202室",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 1,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1751340748,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": null,

                "payment_name": null,

                "create_employee_name": "钱伟伟",

                "creditNo": "91441900MA4WLKY33Q",

                "scope": "产销、研发：电子智能产品、电子产品、电子配件；批发业、零售业；货物进出口、技术进出口。(依法须经批准的项目，经相关部门批准后方可开展经营活动)〓",

                "type_str": "潜在（A类）",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 268749,

                "code": "",

                "ncc_code": "210260559",

                "type": 1,

                "full_name": "正雄（常熟）动力系统有限公司",

                "is_protect": 0,

                "created": 1662511224,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 30,

                "tax_register": "91320581MA27FA4B17",

                "payment_method_id": 18,

                "street": "常熟经济技术开发区通联路10号1幢",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 2,

                "print_template": null,

                "hide_type": null,

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1662511370,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结90天",

                "payment_name": "月结90天,开收据",

                "create_employee_name": "胡志",

                "creditNo": "91320581MA27FA4B17",

                "scope": "一般项目：电机及其控制系统研发；机械设备研发；工业自动控制系统装置制造；工业自动控制系统装置销售；新能源原动设备制造；通用设备制造（不含特种设备制造）；电力电子元器件制造；电力电子元器件销售；输配电及控制设备制造；电子元器件与机电组件设备制造；电子元器件与机电组件设备销售；智能输配电及控制设备销售；配电开关控制设备研发；配电开关控制设备制造；配电开关控制设备销售；机械电气设备制造；机械电气设备销售；电器辅件制造；照明器具制造；照明器具销售；电工器材制造；电工器材销售；汽车零部件及配件制造；充电控制设备租赁；机动车充电销售；五金产品研发；五金产品制造；五金产品零售；合成材料销售；新能源汽车电附件销售；特种陶瓷制品制造；特种陶瓷制品销售；技术服务、技术开发、技术咨询、技术交流、技术转让、技术推广；新材料技术研发；科技推广和应用服务；新兴能源技术研发；工程和技术研究和试验发展；电池制造；电池销售；新能源原动设备销售；塑料制品制造；塑料制品销售（除依法须经批准的项目外，凭营业执照依法自主开展经营活动）",

                "type_str": "成交",

                "assign_time": "2025-08-22 15:35:15",

                "check_status_text": "待审核",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 262936,

                "code": "",

                "ncc_code": "210255672",

                "type": 1,

                "full_name": "亚郎医疗器械(浙江)有限公司",

                "is_protect": 0,

                "created": 1658715284,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 54,

                "tax_register": "91330421MA2JHQTY44",

                "payment_method_id": 11,

                "street": "浙江省嘉兴市嘉善县罗星街道科技大道2699号8号楼",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 2,

                "print_template": null,

                "hide_type": null,

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1658715537,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结60天",

                "payment_name": "月结60天,开13%增值税专用发票",

                "create_employee_name": "冯志豪",

                "creditNo": "91330421MA2JHQTY44",

                "scope": "一般项目：技术服务、技术开发、技术咨询、技术交流、技术转让、技术推广；医学研究和试验发展；第二类医疗器械销售；第一类医疗器械销售；通用零部件制造；通用设备制造（不含特种设备制造）(除依法须经批准的项目外，凭营业执照依法自主开展经营活动)。",

                "type_str": "成交",

                "assign_time": "2025-08-22 15:35:15",

                "check_status_text": "待审核",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 261463,

                "code": "",

                "ncc_code": "220254469",

                "type": 2,

                "full_name": "马鞍山市朗科精密机械有限公司",

                "is_protect": 1,

                "created": 1657708245,

                "is_void": 0,

                "abbr": "",

                "amer_industry_id": 54,

                "tax_register": "91340506MA8NYQTT0A",

                "payment_method_id": 9,

                "street": "横望工业园3栋",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 2,

                "print_template": null,

                "hide_type": "A",

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1754700063,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结60天",

                "payment_name": "月结60天,开13%增值税普通发票",

                "create_employee_name": "吴飞",

                "creditNo": "91340506MA8NYQTT0A",

                "scope": "一般项目：新能源原动设备制造；新能源原动设备销售；电动机制造；齿轮及齿轮减、变速箱制造；齿轮及齿轮减、变速箱销售；工业机器人制造；工业机器人销售；有色金属合金制造；有色金属铸造；有色金属合金销售（除许可业务外，可自主依法经营法律法规非禁止或限制的项目）",

                "type_str": "潜在（A类）",

                "assign_time": "2025-08-22 15:35:15",

                "check_status_text": "待审核",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 210936,

                "code": "ZSBE耀陆实业有限",

                "ncc_code": "210207784",

                "type": 1,

                "full_name": "上海耀陆实业有限公司",

                "is_protect": 0,

                "created": 1644805938,

                "is_void": 0,

                "abbr": "耀陆实业有限",

                "amer_industry_id": 52,

                "tax_register": "91310113MA7GHQJJ4R",

                "payment_method_id": 24,

                "street": "上海市宝山区淞兴路163号1幢5层B区",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 2,

                "print_template": "不打单价(不打印金额，打印付款方式)",

                "hide_type": null,

                "delivery_note_code": "02001",

                "description": null,

                "salesman_customer_created": 1654069016,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "货到收款",

                "payment_name": "货到收款,开13%增值税专用发票",

                "create_employee_name": "潘静",

                "creditNo": "91310113MA7GHQJJ4R",

                "scope": "一般项目：润滑油销售；石油制品销售（不含危险化学品）；包装材料及制品销售；汽车零配件零售；锻件及粉末冶金制品销售；模具销售；塑料制品销售；纸制品销售；会议及展览服务；企业管理咨询；信息咨询服务（不含许可类信息咨询服务）；普通机械设备安装服务；软件开发；工程塑料及合成树脂销售；玻璃纤维增强塑料制品销售；塑料加工专用设备销售；食品用塑料包装容器工具制品销售；金属制品修理；新型金属功能材料销售；劳动保护用品销售；电力电子元器件销售；专用设备修理；通用设备修理；电气设备修理；办公用品销售；计算机软硬件及辅助设备零售；信息系统集成服务。（除依法须经批准的项目外，凭营业执照依法自主开展经营活动）",

                "type_str": "成交",

                "assign_time": "2025-08-22 15:42:36",

                "check_status_text": "待审核",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 206163,

                "code": "JSQC米德勒激光技",

                "ncc_code": "210016888",

                "type": 1,

                "full_name": "米德勒激光技术(苏州)有限公司",

                "is_protect": 1,

                "created": 1640064631,

                "is_void": 0,

                "abbr": "米德勒激光技",

                "amer_industry_id": 37,

                "tax_register": "91320583MA26URE45Q",

                "payment_method_id": 22,

                "street": "昆山开发区饮马路88号7号厂房",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 2,

                "print_template": null,

                "hide_type": null,

                "delivery_note_code": null,

                "description": null,

                "salesman_customer_created": 1676277673,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "月结90天",

                "payment_name": "月结90天,开13%增值税专用发票",

                "create_employee_name": "潘静",

                "creditNo": "91320583MA26URE45Q",

                "scope": "一般项目：激光打标加工；集成电路芯片设计及服务；集成电路设计；工业设计服务；汽车零部件及配件制造；金属切割及焊接设备制造；金属切割及焊接设备销售；汽车零部件研发；金属制品研发（除依法须经批准的项目外，凭营业执照依法自主开展经营活动）",

                "type_str": "成交",

                "assign_time": "2025-08-22 15:42:36",

                "check_status_text": "待审核",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            },

            {

                "id": 194223,

                "code": "ZSMI波维化学试剂",

                "ncc_code": "040019590",

                "type": 1,

                "full_name": "上海波维化学试剂有限公司",

                "is_protect": 0,

                "created": 1633682404,

                "is_void": 0,

                "abbr": "波维化学试剂",

                "amer_industry_id": 51,

                "tax_register": "91310115770204659U",

                "payment_method_id": 21,

                "street": "上海市闵行区都庄路4226号3幢429室",

                "is_invoice": 0,

                "create_customer_type": 1,

                "sea_type": 2,

                "print_template": "不打单价(不打印金额，打印付款方式)",

                "hide_type": "A",

                "delivery_note_code": "02001",

                "description": null,

                "salesman_customer_created": 1742968649,

                "department_code": "A0503041302",

                "employee_code": "000068",

                "bs_employee_name": "蒋先丙",

                "bs_department_name": "蒋先丙",

                "bs_department_system_code": "A050304",

                "payment_ncc_name": "款到发货",

                "payment_name": "款到发货,开13%增值税专用发票",

                "create_employee_name": "潘静",

                "creditNo": "91310115770204659U",

                "scope": "一般项目：化学试剂、化工产品（不含许可类化工产品）、玻璃仪器、实验仪器的销售。（除依法须经批准的项目外，凭营业执照依法自主开展经营活动）",

                "type_str": "成交",

                "assign_time": "",

                "check_status_text": "未派发/未提审",

                "bs_department_system_name": "东部二路军",

                "tag_list": []

            }

        ],

        "total": 151

    }

}
```

