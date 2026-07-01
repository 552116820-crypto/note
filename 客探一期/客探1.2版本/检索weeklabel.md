● 方案：在知识检索前动态计算 week_label

  整体流程：

  用户输入
    ↓
  LLM 节点（提取时间意图）
    ↓
  代码节点（计算 week_label）
    ↓
  知识检索节点（元数据过滤 week_label = 变量）
    ↓
  生成周报

  1. LLM 节点 — 提取时间意图

  提示词示例：

  当前日期：{{#sys.current_date#}}
  用户输入：{{#sys.query#}}

  请判断用户想要哪一周的周报，返回一个整数 offset：
  - "这周"/"本周" → 0
  - "上周" → -1
  - "上上周" → -2
  - "下周" → 1
  - 如果用户指定了具体周（如"第12周"/"2026W12"），直接返回该周标签

  仅输出一个整数或周标签，不要输出其他内容。

  2. 代码节点 — 计算 week_label

  from datetime import datetime, timedelta

  def main(llm_output: str) -> dict:
      llm_output = llm_output.strip()

      # 如果LLM直接返回了周标签（如 2026W12）
      if 'W' in llm_output:
          return {"week_label": llm_output}

      # 否则按 offset 计算
      offset = int(llm_output)
      today = datetime.now()
      target = today + timedelta(weeks=offset)
      year, week, _ = target.isocalendar()
      week_label = f"{year}W{week:02d}"

      return {"week_label": week_label}

  3. 知识检索节点 — 元数据过滤

  在知识检索节点的元数据过滤配置中：

  字段：week_label
  操作符：等于
  值：{{#code_node.week_label#}}

  这样 week_label 就是动态的，会根据用户的输入自动计算。

  补充

  如果你不想依赖 LLM 来判断意图（担心不稳定），也可以：
  - 给用户提供下拉选项（本周 / 上周 / 指定周）作为会话变量
  - 直接用代码节点根据选项计算，完全不需要 LLM 判断



● 抱歉，sys.current_date 不一定是 Dify 内置的系统变量。Dify 内置的系统变量主要有
  sys.query、sys.user_id、sys.conversation_id 等。

  解决方法

  在 LLM 节点之前加一个代码节点来获取当前日期：

  from datetime import datetime

  def main() -> dict:
      today = datetime.now().strftime("%Y-%m-%d")
      return {"current_date": today}

  然后在 LLM 节点的提示词中引用这个代码节点的输出：

  当前日期：{{#code_node.current_date#}}
  用户输入：{{#sys.query#}}

  请判断用户想要哪一周的周报，返回一个整数 offset：
  - "这周"/"本周" → 0
  - "上周" → -1
  ...

  或者更简单——把两步合并成一个代码节点，直接计算出 week_label，跳过 LLM 判断：

  from datetime import datetime, timedelta
  import re

  def main(query: str) -> dict:
      today = datetime.now()

      # 匹配具体周标签，如 2026W12
      match = re.search(r'(\d{4})W(\d{1,2})', query)
      if match:
          return {"week_label": f"{match.group(1)}W{int(match.group(2)):02d}"}

      # 关键词匹配
      if '上上周' in query:
          offset = -2
      elif '上周' in query:
          offset = -1
      elif '下周' in query:
          offset = 1
      else:
          offset = 0  # 默认本周

      target = today + timedelta(weeks=offset)
      year, week, _ = target.isocalendar()
      return {"week_label": f"{year}W{week:02d}"}

  这样用纯代码就能搞定，不需要额外调用 LLM，更稳定也更快。
