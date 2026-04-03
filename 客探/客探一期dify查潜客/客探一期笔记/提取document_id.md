• 你这段响应里所谓的 document_id 就是 body 解析后的 document.id：

  - 路径：(HTTP输出).body  ->  JSON解析  ->  document  ->  id
  - 你这条样例的值就是：55091175-e716-4460-820f-9869e1971ea2

  因为 Dify 的 HTTP 节点里 body 往往是“字符串形式的 JSON”（你这里就是 "{\"document\":{...}}"），所以最稳的方式是：HTTP 后面接一个 Code 节点解析并输出
  document_id。

  ———

  ## 方案：HTTP → Code（解析）→ 取 document_id

  ### 1）Code 节点输入参数绑定

  在 Code 节点里配置输入变量（名字你可以自定义，下面是推荐）：

  - status_code ← 选择 你的HTTP节点.status_code
  - body ← 选择 你的HTTP节点.body

  ### 2）Code 节点 Output Variables（输出变量）

  为避免你之前遇到的验证报错，建议你只定义这两个输出（或确保它们存在）：

  - result：Boolean
  - document_id：String

  > 如果你 Output Variables 里配置了哪些字段，return {} 就必须包含这些字段（并且类型匹配），否则就会报 Output result is missing / Not all output parameters
  > are validated。

  ### 3）Code 节点代码（Python，直接可用）

  把下面代码完整粘进去（注意不要把 Traceback 文本也粘进去）：

  import json

  def main(status_code: int, body) -> dict:
      # 兜底：默认输出（保证输出参数永远齐全）
      result = False
      document_id = ""

      if int(status_code) != 200:
          return {"result": result, "document_id": document_id}

      payload = body
      if isinstance(payload, str):
          s = payload.strip()
          if s:
              payload = json.loads(s)
          else:
              payload = {}

      if isinstance(payload, dict):
          document_id = (payload.get("document") or {}).get("id") or ""

      result = bool(document_id)
      return {"result": result, "document_id": str(document_id)}

  ### 4）后续怎么用

  - 在 If-Else 节点判断：Code.result == true（表示已拿到 document_id）
  - 后续要引用文档 id：直接用 Code.document_id
  - 推荐再用一个 Variable Assigner 把 document_id 存成会话变量（比如 doc_id），后面更新/检索都复用。

  ———

  ## 小提醒：你这个创建响应里文档还没可检索

  你响应里有：

  - indexing_status: "waiting"
  - display_status: "queuing"

  这说明文档刚创建完正在排队/索引中，马上检索可能搜不到。通常要轮询（再发一次“查询文档详情/列表”）直到 indexing_status == "completed" 才算真正可检索。

  如果你告诉我你现在 Code 节点的 Output Variables 列表（有哪些字段），我也可以把 return 的结构按你那份列表“逐项补齐”，保证不再触发验证类报错。
