› 书写skill.md一定得用英文吗


• 不一定。

  按当前 skill 规范，真正有硬约束的是 name：
  只能用小写字母、数字、连字符，所以通常还是英文 slug，例如 maintain-dify-crm-customer-workflows。

  其他内容不必强制英文：
  description 可以写中文，SKILL.md 正文也可以写中文，agents/openai.yaml 里的 display_name、short_description、
  default_prompt 也都可以写中文。

  实际建议是：

  - name 保持英文 slug
  - description 用你希望触发 skill 的语言写
  - 如果日常提问中英混用，description 最好写成中英混合，提升触发概率
  - 正文按团队最顺手的语言写，中文完全可以



• 有，分成“硬性规范”和“推荐规范”最清楚。依据是 skill-creator/SKILL.md 和 openai_yaml.md。

  硬性规范

  - 每个 skill 至少要有一个 SKILL.md。
  - SKILL.md 的 YAML frontmatter 只能有两个字段：name 和 description。
  - name 只能用小写字母、数字、连字符，且目录名要和 skill 名一致，例如 my-skill。
  - description 是主要触发条件，必须写清楚“这个 skill 做什么”以及“什么时候该用它”。
  - SKILL.md 正文要写成操作性说明，规范要求偏祈使/不定式风格，也就是直接写“做什么”。
  - 完成后要跑校验：quick_validate.py <skill-folder>。

  推荐规范

  - SKILL.md 保持精简，核心流程留在正文，细节拆到 references/、可执行逻辑拆到 scripts/、模板资源放 assets/。
  - 正文最好控制在 500 行以内，避免上下文过重。
  - 不要堆无关文档，比如 README.md、CHANGELOG.md、INSTALLATION_GUIDE.md 这类通常不该放进 skill。
  - 如果有多种变体，只在 SKILL.md 里放选择规则和主流程，具体变体放到引用文件。
  - 更新 skill 时，agents/openai.yaml 最好同步维护，但它是推荐项，不是必需项。

  agents/openai.yaml 规范

  - 这是推荐文件，不是强制文件。
  - 字符串值要加引号，键名不要加引号。
  - default_prompt 必须显式包含 $skill-name。
  - 常见字段是 display_name、short_description、default_prompt。

  语言规范

  - name 必须是英文 slug。
  - description、正文、openai.yaml 展示文案都可以中文、英文或中英双语。
  - 如果希望触发更稳，description 最好覆盖用户真实会说的话，中英混合通常更实用。
