# Claude Code 对话记录

- **来源**: Claude Code
- **会话**: `37efff83-d964-4697-b88e-96e8bf07dfa8`
- **范围**: Turn 1-3
- **导出时间**: 2026-05-25 09:01:26

---

### Turn 1  *(2026-05-25 00:14)*

**User:**

对于这个html，"E:\html_template\安美AI销售赋能系统_全景可视化(1).html"，请问在没有这个参考文件的前提，怎么让claude code 生成类似排版的html，给我一套完整的提示词

**Assistant:**

我先把那个 HTML 读完，分析它的排版/视觉/技术栈特征，再据此抽出一套可复用的提示词。

我已经读完那份 HTML（1051 行，单文件，无依赖），核心特征是：单页/中文/绿色系/时间线交替叙事 + PSV 三段折叠 + 管理者深色玻璃拟态 + 飞轮自转 + 痛点速查矩阵。下面给你一套**自包含、可复制即用**的提示词，让 Claude Code 在不看原文件时也能产出同款排版。

---

## 一、使用方式（先看这个）

1. 把下面 `===== 提示词正文开始 =====` 到 `===== 提示词正文结束 =====` 之间**整段复制**贴给 Claude Code。
2. 把"填空区"里的 YAML 字段替换成你自己项目的素材（产品名、KPI、模块、Journey 节点等）。
3. 直接让它输出 `.html` 文件即可。

设计上做了两点取舍：
- **骨架/视觉系统/微交互**写死成精确规格（颜色 hex、尺寸 px、动画时长），保证排版高度复现。
- **主题色/节点数/文案**做成参数，方便换行业（销售赋能 → 招聘系统 → 教育产品都能套）。

---

## 二、提示词正文

```
===== 提示词正文开始 =====

# 你的角色

你是一位「商业叙事 × 信息可视化」设计师，同时是熟练的前端工程师。
你的专长：用单页 HTML 把一个复杂系统/方案/产品矩阵，浓缩成让目标读者
2 分钟内看懂的"全景可视化"页面。

# 任务

基于本文末尾「填空区」的素材，输出一份完整的、单文件、可直接用浏览器
打开的 HTML，呈现一个商业系统的全景可视化页面。

# 硬规则（不可违反）

1. 单文件：HTML、CSS、JS 全部内联在一个 .html 文件里。
2. 零外部依赖：不引入任何 CDN、Webfont、图标库、JS 框架。所有图标用 emoji。
3. 字体栈：-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif
4. 响应式：仅一个断点 @media (max-width: 768px)
5. 容器：max-width: 1000px; margin: 0 auto; padding: 20px;  背景 #FAFAFA
6. 输出：直接输出 HTML 文本，不包在 markdown 代码块里，前后不加任何说明文字。

# 设计 Tokens（按 CSS 变量声明）

:root {
  --primary:       {{主色}};       /* 默认 #43A047 */
  --primary-dark:  {{主深}};       /* 默认 #2E7D32 */
  --primary-light: {{主浅}};       /* 默认 #E8F5E9 */
  --blue:   #1565C0;  --blue-light:   #E3F2FD;
  --orange: #EF6C00;  --orange-light: #FFF3E0;
  --purple: #6A1B9A;  --purple-light: #F3E5F5;
  --teal:   #00897B;  --teal-light:   #E0F2F1;
  --red:    #C62828;
  --text:           #212121;
  --text-secondary: #757575;
  --gray-light:     #F5F5F5;
  --shadow:       0 2px 12px rgba(0,0,0,0.08);
  --shadow-hover: 0 6px 24px rgba(0,0,0,0.12);
}

- 圆角：卡片 12px / 大容器 16px / 圆形节点 50%
- 板块间距：40px；卡片间距：14–16px

# 模块色板（按模块总数生成 m1~mN，类名固定）

m1  bg #FFF8E1 / border #FFD54F / text #F57F17
m2  bg #E8F5E9 / border #81C784 / text #2E7D32
m3  bg #FFF3E0 / border #FFB74D / text #E65100
m4  bg #E3F2FD / border #64B5F6 / text #1565C0
m5  bg #F3E5F5 / border #BA68C8 / text #6A1B9A
m6  bg #FFEBEE / border #EF9A9A / text #C62828
m7  bg #E0F2F1 / border #4DB6AC / text #00695C
m8  bg #FCE4EC / border #F48FB1 / text #AD1457
m9  bg #E8EAF6 / border #7986CB / text #283593
m10 bg #FBE9E7 / border #FF8A65 / text #BF360C
m11 bg #E0F7FA / border #4DD0E1 / text #00838F
m12 bg #F9FBE7 / border #DCE775 / text #827717
m13 bg #EFEBE9 / border #A1887F / text #4E342E
m14 bg #E1F5FE / border #4FC3F7 / text #01579B

# 页面骨架（从上到下严格按此顺序）

## 1) Hero
- 渐变：linear-gradient(135deg, var(--primary-dark) 0%, var(--primary) 50%, {{主辅亮色}} 100%)
- 圆角 16 / padding 48 36 / 白字 / text-align center
- ::before 在右上角放 200×200 圆 rgba(255,255,255,0.06)，top:-60 right:-60
- ::after  在左下角放 260×260 圆 rgba(255,255,255,0.04)，bottom:-80 left:-40
- 标题 28px / 副标 15px opacity 0.9 / sub 16px opacity 0.85 margin-top 12
- 4 个 KPI（hero-stats）：display flex justify-center gap 40 margin-top 24
  · 每个 KPI：数字 32px 700 + 标签 12px opacity 0.8

## 2) Section Title + Intro（每个大块上方使用）
- .section-title：text-align center；margin 40 0 24；position relative
  · h2 inline-block padding 0 20 background #FAFAFA z-index 1 字号 22 主深色
  · ::before：top 50% left 0 right 0 height 1 background #E0E0E0
  · 效果：一条贯穿灰线被居中标题文字遮断
- .intro：max-width 700 居中；font 15 / line-height 1.8 / 次文字色

## 3) Timeline Journey（核心，重点做）

骨架：
- .journey::before：竖线 width 3 left 50% transform translateX(-50%)
  background linear-gradient(to bottom, var(--primary), var(--blue), var(--purple))
  border-radius 3
- .journey-node：position relative；margin-bottom 16；cursor pointer；display flex
  · :nth-child(odd) { flex-direction: row }
  · :nth-child(even) { flex-direction: row-reverse }
- .time-badge：absolute left 50% transform translateX(-50%) width 52 height 52
  圆形 / display flex column / center / 白字 / 自定义背景色
  · .time-icon 18px（放 emoji）
  · .time-text 10px 700
  · hover: translateX(-50%) scale(1.15)
- .journey-card：width calc(50% - 44px) / bg white 圆角 12 padding 20
  border 1px #E8E8E8 / box-shadow var(--shadow)
  · hover: translateY(-2px) + shadow-hover
  · :nth-child(odd) .journey-card { margin-right:auto }
  · :nth-child(even).journey-card { margin-left:auto }

卡片内部：
1. .card-header：标题 16px 600 + 一个 .card-tag（11px / padding 2 8 / 圆角 10 /
   浅绿背景 深绿文字，如"拜访前/中/后"）
2. .card-scene：13.5px / 次文字色 / line-height 1.7
   ⚠ 必须写成「画面感叙事」，不是产品说明（见下方"内容写作规范"）
3. .card-toggle：12px 主色 cursor pointer
   初始文案 "▶ 查看痛点 · 方案 · 价值"，展开后变 "▶ 收起"，箭头 rotate 90deg
4. .psv（折叠区）：默认 display none；.psv.open { display block; animation: fadeIn 0.3s }
   @keyframes fadeIn { from {opacity:0; transform: translateY(-8px)} to {opacity:1; transform: translateY(0)} }
   内含 3 行 .psv-row（icon + content）：
   · pain     —— icon-bg #FFEBEE，label 红色 11px 600 uppercase
   · solution —— icon-bg 主浅色，label 主深色
   · value    —— icon-bg 浅蓝，label 蓝色
   · 每行文字 13px / line-height 1.6
5. .module-tags：flex wrap gap 6 margin-top 10
   每个 .module-tag：11px / padding 3 10 / 圆角 12 / 1px solid border / 用 m{X} 配色

## 4) Manager Section
- background linear-gradient(135deg, #1A237E, #283593)
- 圆角 16 padding 32 白字 margin-top 40
- h2 20px / .sub 14px opacity 0.7 margin-bottom 20
- .manager-grid：grid-template-columns repeat(3, 1fr) gap 14
- .manager-card：bg rgba(255,255,255,0.1) backdrop-filter blur(4px)
  圆角 10 padding 18 transition 0.3
  · hover: bg rgba(255,255,255,0.18) + translateY(-2px)
  · .mc-icon 24px / .mc-title 14px 600 / .mc-desc 12px opacity 0.8 / 
  · .mc-value inline-block padding 4 10 bg rgba(76,175,80,0.3) 圆角 8 字号 12

## 5) Flywheel Section
- 外层 .flywheel-container：bg white 圆角 16 padding 32 shadow
  display flex gap 32 align-items center
- 左 .flywheel-visual：flex 0 0 320px; height 320; position relative
  · .flywheel-ring：280×280 圆 border 3px dashed var(--primary) opacity 0.3
    position absolute top 20 left 20 animation spin 30s linear infinite
    @keyframes spin { from{transform:rotate(0)} to{transform:rotate(360deg)} }
  · .flywheel-center：100×100 圆 主色 白字 13px 600 居中文字两行
    position absolute top 50% left 50% transform translate(-50%,-50%)
    box-shadow 0 4px 20px rgba(67,160,71,0.3)
  · .fw-node（6 个）：80×80 圆 居中文字两行 11px 500 shadow transition
    定位：
    .n1 top 0    left 50% transform translateX(-50%)         浅绿/绿
    .n2 top 22%  right 0                                     浅蓝/蓝
    .n3 bottom 22% right 0                                   浅紫/紫
    .n4 bottom 0 left 50% transform translateX(-50%)         浅橙/橙
    .n5 bottom 22% left 0                                    浅青/青
    .n6 top 22%  left 0                                      浅粉/粉
    · hover: scale 1.1（n1/n4 要保留 translateX(-50%)）
- 右 .flywheel-text：flex 1
  · h3 18px 主深色
  · p 14px 次色 line-height 1.8
  · 6 个 .fw-step：display flex align-items center gap 8 字号 13
    每个含 .fw-num（24×24 圆 主色 白字 12 600）+ 一句步骤
  · 末尾一句加粗主深色作结论

## 6) Summary Matrix
- .matrix-grid：grid-template-columns repeat(2, 1fr) gap 14
- .matrix-card：bg white 圆角 12 padding 20 shadow border-left 4px
  · hover: translateY(-2px) + shadow-hover
  · :nth-child(N) 给每张卡 border-left 不同颜色，且让 .mx-value 同色
    （颜色档轮换使用 primary / blue / orange / purple / teal / #C2185B /
     #283593 / #BF360C / #00838F / #827717 / #4E342E / #01579B）
- 内容：
  · .mx-pain 13px 红色 500（用😤开头）
  · .mx-solution 14px 600 文本主色（用 → 开头）
  · .mx-value 20px 700 配色
  · .mx-detail 12px 次色

## 7) Footer
- text-align center padding 32 0 16 / 字号 12 次色
- 一行主标语 14px 600 主深色 + 一行公司全称

# 微交互

- 所有卡片 hover：translateY(-2px) + shadow → shadow-hover
- 圆形节点 hover：transform scale(1.1) 0.3s
- Journey 卡片点击 togglePSV()，展开当前并关闭其他（互斥）
- Flywheel ring 自转 30s linear infinite

附 JS（直接照抄）：

function togglePSV(node) {
  const psv = node.querySelector('.psv');
  const arrow = node.querySelector('.arrow');
  const toggle = node.querySelector('.card-toggle');
  if (!psv) return;
  const isOpen = psv.classList.contains('open');
  document.querySelectorAll('.psv.open').forEach(el => {
    if (el !== psv) {
      el.classList.remove('open');
      const a = el.closest('.journey-node').querySelector('.arrow');
      const t = el.closest('.journey-node').querySelector('.card-toggle');
      if (a) a.classList.remove('open');
      if (t) t.innerHTML = '<span class="arrow">▶</span> 查看痛点 · 方案 · 价值';
    }
  });
  if (isOpen) {
    psv.classList.remove('open');
    if (arrow) arrow.classList.remove('open');
    toggle.innerHTML = '<span class="arrow">▶</span> 查看痛点 · 方案 · 价值';
  } else {
    psv.classList.add('open');
    if (arrow) arrow.classList.add('open');
    toggle.innerHTML = '<span class="arrow open">▶</span> 收起';
  }
}

# 响应式（@media max-width 768px）

- .journey::before { left: 28px }
- .journey-node, .journey-node:nth-child(even) { flex-direction: row }
- .time-badge { position: relative; left: 28px; transform: none; flex-shrink: 0 }
  hover 改为 scale(1.1) 不带 translateX
- .journey-card { width: auto; margin-left: 16px; margin-right: 0 }
- .manager-grid { grid-template-columns: 1fr }
- .matrix-grid  { grid-template-columns: 1fr }
- .flywheel-container { flex-direction: column }
- .flywheel-visual { flex: none; width: 100%; max-width: 320px; margin: 0 auto }
- .hero-stats { gap: 24px; flex-wrap: wrap }

# 内容写作规范（重要，AI 最容易在这里跑偏）

## Journey 节点 · 场景描述
- 用画面感叙事，把读者放进具体场景
- ✅ "早上到办公室，打开企微，AI 已为今天要拜访的客户准备好简报。30 分钟手动调研压缩到 3 分钟阅读。"
- ❌ "本模块为销售提供拜访前的客户简报功能，包含行业动态、产品建议等。"
- 长度 100–300 字，至少含一个具体「数字 / 对话 / 动作」细节让画面立起来。

## PSV 三段
- 痛点：用读者真实的吐槽口吻，让目标人群"对对对我就是这样"
- 方案：动词链结构 ——「自动抓取 X → 结合 Y → 生成 Z → 推送 W」，可用 ①②③ 编号
- 价值：3–4 个「·」分隔的短句，至少一个带具体数字（"+35%"、"-90%"、"30 分钟降至 3 分钟"）

## Hero KPI
- 4 个，数字 + 单位（35% / 600+ / 14 / 15），标签 ≤ 6 字

## Manager Card
- 标题 ≤ 8 字；desc ≤ 40 字说清"看什么+解决什么"；value ≤ 10 字一句价值主张

## Summary Matrix Card
- 痛点用 😤 开头；方案用 → 开头；价值用大字呈现数字/倍数/时间节省；细节一句话补因果

# 节点数量建议

- Journey：8–12 个（最佳 10 个，按时间从早到晚或从前置到后置排）
- Manager Card：6–10 个
- Summary Matrix：8–13 个
- Flywheel：固定 6 圆周节点 + 6 步骤

# 填空区（请用户提供）

产品名:
副标题/品牌行:
主标语:
主色调档位: [绿色|蓝色|紫色|橙色|自定义hex]

Hero_KPI:   # 4 项
  - { 数字: , 标签: }
  - { 数字: , 标签: }
  - { 数字: , 标签: }
  - { 数字: , 标签: }

模块清单:   # M1~MN
  - { 编号: M1, 名称: }
  - { 编号: M2, 名称: }
  # ...

Journey_主线介绍:        # 一段 80–150 字的引导文
Journey_节点:            # 按时间顺序
  - 时间标签:            # 如 "7:30" / "前一晚" / "常驻"
    emoji:               # 如 🔍
    徽章背景色:          # 可用 var(--primary) 或 hex
    卡片标题:
    阶段tag:             # 如 "拜访前"
    场景描述:            # 100–300 字画面叙事
    痛点:
    方案:
    价值:
    关联模块: [M1, M2]
  # ...

管理者视角_介绍:         # 1–2 句
管理者视角_卡片:         # 6–10 张
  - { emoji: , 标题: , desc: , value: }
  # ...

Flywheel:
  中心标语:              # 2×2 字双行（如 "数据飞轮"）
  6_圆周节点:            # 顺时针，每个 2 行 4–6 字
    - 
  6_闭环步骤:            # 描述飞轮如何闭环越用越聪明
    - 
  收尾结论:              # 一句加粗主深色

Summary_Matrix:          # 8–13 条
  - { 痛点: , 方案: , 数字价值: , 细节: }
  # ...

Footer:
  公司全称:
  slogan:

# 自检 Checklist（生成后逐项核对，不过就重做）

[ ] Hero 有 135deg 渐变 + 两个绝对定位装饰圆
[ ] Section title 横线被居中标题字遮断（不是断在两侧）
[ ] Journey 节点严格左右交替（odd 左 / even 右）
[ ] 中央时间徽章圆形居中盖在竖线上
[ ] 每个 Journey 卡片点击展开 PSV，互斥（只能开一个）
[ ] PSV 三段配色：pain 红 / solution 绿 / value 蓝
[ ] 模块 tag 至少出现填空区清单中所有 m{X} 配色
[ ] Manager 区深蓝渐变 + 卡片 backdrop-filter blur(4px)
[ ] Flywheel ring 在自转 + 6 节点对称分布
[ ] Matrix 每张卡 border-left 颜色不同
[ ] 768px 下时间线左偏 / 网格变单列 / 飞轮纵向堆叠
[ ] 所有图标用 emoji，全文件零外部资源
[ ] 文件能双击直接在浏览器打开运行

# 输出指令

不要解释，不要包代码块，直接输出完整 HTML 内容。

===== 提示词正文结束 =====
```

---

## 三、使用贴士

1. **填空区是可选的**——只填关键字段（产品名 + Journey 节点 + Matrix）就够生成完整页面，其他没填的 Claude 会基于已有信息合理补全。如果你想一次性多次复用，先把空 YAML 模板存成 `prompt-fanjing.md`，下次只改 YAML 那块。

2. **节点超过 14 个**：把"模块色板"段加几行 `m15/m16…`（按相同三件套规律续编），同时把"节点数量建议"里的 Journey 上限改高。色谱可再循环复用。

3. **想换主题**——比如改成"招聘 AI 系统全景图"、"教育产品落地页"、"内部工具产品矩阵"，只需要换填空区的内容；骨架/视觉系统几乎都能直接套用。如果题材完全不是"系统/方案/产品矩阵"（比如纯营销 landing page），骨架可能不再合适，需要在提示词最上方加一句"删除 Flywheel 板块"或"用 X 板块替换 Y"。

4. **想要暗色版**：在硬规则下方加 `# 主题模式: 暗色`，告诉它把 `#FAFAFA` 换成 `#0F0F12`、文字反色、shadow 改为发光（box-shadow 用 rgba 主色 0.2）。

5. **第一次跑出来排版不对**：最常见两个偏差—— ① Journey 卡片不交替（CSS 里漏了 `:nth-child(even) { flex-direction: row-reverse }`）；② Section title 横线没被遮断（h2 漏了 `background: #FAFAFA`）。直接贴这两条让它修就行，不必整个重生成。

---

### Turn 2  *(2026-05-25 00:24)*

**User:**

"E:\html_template\安美AI销售赋能系统_八大模块分工可视化.html"，这个文件呢

**Assistant:**

我先把这份读完，对比上一份提炼出新的设计语言。

任务直白，不需要 tasklist。继续读后半段。

读完了（602 行，与上份是同项目的"姊妹页"）。这份是**项目管理/执行视角**——数据驱动 + RACI + 任务表 + 部门筛选 + 工作量条形图，跟上份的"营销叙事视角"互补。

**两套该怎么选**：
- 上一份（全景可视化）适合 **对外宣讲 / Demo / 管理层 Pitch**——强叙事让人 2 分钟看懂愿景
- 这一份（分工可视化）适合 **项目启动会 / 团队对齐 / 进度跟踪**——强执行让人看清"谁干什么 / 几月份 / 依赖什么"

---

## 一、提示词正文（直接复制）

```
===== 提示词正文开始 =====

# 你的角色

你是一位「项目管理 × 数据驱动可视化」设计师 + 前端工程师。
你的专长：把一个跨部门、多模块、多周期的复杂项目，浓缩成一份让 PMO/团队
负责人 / 跨部门协作方一眼看清"谁主责、谁配合、什么交付物、哪个月、依赖谁"
的单页 HTML 项目分工图。

# 任务

基于本文末尾「填空区」给出的素材，输出一份完整的、单文件、可直接用浏览器
打开的 HTML，呈现一个项目的「模块分工与执行可视化」页面。

# 硬规则

1. 单文件：HTML、CSS、JS 全部内联在一个 .html 文件里。
2. 零外部依赖：不引入 CDN/Webfont/图标库/JS 框架。所有图标用 emoji。
3. 字体栈：-apple-system, "PingFang SC", "Microsoft YaHei", sans-serif
4. 响应式：仅一个断点 @media (max-width: 768px)
5. 容器：max-width: 1100px; margin: 0 auto; padding: 20px; 背景 #FAFAFA
6. ⚠ 核心要求：所有模块/工作量数据写成 JS 数据数组，DOM 由 render 函数
   动态生成。HTML 里不要手写模块卡片或工作量条。
7. 输出：直接输出完整 HTML 文本，不包在 markdown 代码块里，前后不加任何说明。

# 设计 Tokens

:root {
  --primary:      {{主色}};         /* 默认 #43A047 */
  --primary-dark: {{主深}};         /* 默认 #2E7D32 */
  --primary-light:{{主浅}};         /* 默认 #E8F5E9 */
  --blue:   #1565C0; --blue-light:   #E3F2FD;
  --orange: #EF6C00; --orange-light: #FFF3E0;
  --purple: #6A1B9A; --purple-light: #F3E5F5;
  --red:    #C62828; --red-light:    #FFEBEE;
  --teal:   #00897B; --teal-light:   #E0F2F1;
  --pink:   #C2185B; --pink-light:   #FCE4EC;
  --amber:  #F57F17; --amber-light:  #FFF8E1;
  --indigo: #283593; --indigo-light: #E8EAF6;
  --gray:   #616161; --gray-light:   #F5F5F5;
  --text:   #212121; --text-sec:     #757575;
  --shadow:       0 2px 12px rgba(0,0,0,0.08);
  --shadow-hover: 0 8px 30px rgba(0,0,0,0.14);
}

# 页面骨架（从上到下，严格此序）

## 1) Hero（简洁版）
- 渐变 linear-gradient(135deg, var(--primary-dark), var(--primary) 60%, {{主辅亮色}})
- 圆角 16 / padding 36 / 白字 / text-align center / margin-bottom 24
- 仅一个装饰圆 ::before：top -50 right -50 w 180 h 180 圆 rgba(255,255,255,0.06)
- 标题 26px / 副标 14px opacity 0.85
- 副标的格式建议："N大模块 / M个团队角色 / X个月实施周期 / 治理结构"

## 2) Stats Bar
- grid-template-columns: repeat(4, 1fr); gap 12; margin 24 0
- .stat-card：bg white 圆角 12 padding 18 text-center shadow
  · hover translateY(-3px)
  · .stat-num 28px 700（inline style 设颜色，分别用 primary/blue/orange/purple）
  · .stat-label 12px 次色 margin-top 4

## 3) Filter Bar
- flex wrap justify-center gap 8 padding 16
- bg white 圆角 12 shadow margin-bottom 24
- .filter-label "按部门筛选："13px 次色 600 self-center
- .filter-btn：padding 6 16 / 圆角 20 / 2px solid #E0E0E0 / 13px / bg white
  · hover: border 主色 + 文字主色
  · active: bg 主色 + 白字 + border 主色
- 第一个按钮固定 "全部"，初始 active
- 后续按钮：每个部门一个

## 4) Module Cards（核心，数据驱动）

外层：<div id="modules-container"></div>，由 renderModules() 注入。

每张 module-card：
- bg white 圆角 14 mb 16 shadow overflow hidden border-left 5px（模块色）
- hover: shadow → shadow-hover
- 包含 .collapsed 时隐藏 .module-body

.module-header（点击切换折叠）：
- flex align-center padding 14 18 gap 14 cursor pointer user-select none
- .module-id：44×44 圆角 10 模块色 白字 15 700（如 M1）
- .module-info（flex 1 min-width 0）：
  · .module-name 16 600
  · .module-desc 12 次色 margin-top 2
- .module-timeline-bar：flex 0 0 320px height 24
  · 灰背景圆角 12 overflow hidden
  · 内置 .module-bar-fill：absolute 按 timeline 在 1-N 月之间计算 left/width
    height 100% 圆角 12 opacity 0.7 bg 模块色 居中文字 10px 白字 600
    显示 "月X-Y" 或 "月X-Y+"（ongoing）
- .module-toggle：▶ 字符 20px #CCC
  · 展开时 transform rotate(90deg) 色 var(--primary)

.module-body padding 0 18 18，由以下组成：

① 前置依赖横条（仅当 depends 存在）：
   橙色背景 #FFF3E0 圆角 8 padding 8 12 字号 12 色 #E65100
   内容："前置依赖：M1 → M2 需先就绪"

② RACI Strip：
   bg var(--gray-light) 圆角 8 padding 10 12 mb 14 flex wrap gap 6
   .raci-chip：11px padding 3 10 圆角 12 inline-flex gap 4
     - r 类：bg #C8E6C9 / 文字 #1B5E20
     - a 类：bg #FFCDD2 / 文字 #B71C1C
     - c 类：bg #BBDEFB / 文字 #0D47A1
     - i 类：bg #F5F5F5 / 文字 #616161
   .raci-letter 700 加粗
   每个 chip 标 data-dept="部门名" 供筛选使用

③ Task Table：
   table width 100% border-collapse separate border-spacing 0 字号 12
   列宽：编号 40 / 任务项 auto / 主责 90 / 配合 100 / 交付物 100 / 周期 80
   thead th：bg gray-light padding 8 10 左对齐 600 次色 11px border-bottom 2px #E0E0E0
   tbody td：padding 10 border-bottom 1px #F0F0F0 vertical-align top line-height 1.6
   tbody tr:hover { background: #FAFFFE }
   
   每个 tr 必须有 data-owner="主责部门" data-support="配合1,配合2"，便于筛选
   
   td.task-id：600 主深色 nowrap
   td 任务项：<b>名称</b><br>+ 11px 次色描述
   .owner-tag（主责）：padding 2 8 圆角 10 11 500 nowrap
     背景/文字色查 OWNER_COLORS / OWNER_TEXT 映射（见下）
   .support-tag（配合，可多个）：padding 2 6 圆角 8 10 浅灰背景 灰文字 margin 1 2
   .deliverable：蓝色 var(--blue) 500
   .timeline-cell：11px 次色 nowrap

## 5) Section Title「各部门工作量投入（人月）」
- text-align center margin 32 0 16
- h2 inline-block padding 0 20 background #FAFAFA z-index 1 20 主深色
- ::before 贯穿灰线被字遮断

## 6) Workload Bars（数据驱动）

外层：<div id="workload-section"></div>，由 renderWorkload() 注入。

每行 .workload-bar-row：
- flex align-center gap 12 mb 10
- .workload-dept：width 120 12 500 右对齐 flex-shrink 0
- .workload-bar-bg：flex 1 height 28 bg #F0F0F0 圆角 14 overflow hidden position relative
- .workload-bar-fill：height 100% 圆角 14 部门色
  · flex align-center padding-left 10 11 白字 600
  · width = total / maxTotal * 100%
  · transition: width 0.8s ease-out
  · 文字 "X人月 (a+b+c)"
- .workload-total：50 12 次色 600

末尾追加一段 11 #999 居中小字注释，说明三个阶段含义。

## 7) Footer
- text-align center padding 24 0 12 12 次色
- 一行主标语 14 600 主深色 + 一行公司全称

# 数据结构 Schema（用户填空必看）

const MODULES = [
  {
    id: 'M1',                            // 模块编号
    name: '知识底座',                     // 模块名
    color: '#F57F17',                    // 主色（border-left + id 块 + 时间条）
    bg: '#FFF8E1',                       // 浅背景（可选用）
    desc: '一句话说明这个模块在做什么',
    timeline: [1, 4],                    // 起止月（基于实施总周期）
    ongoing: true,                       // 可选，true 时时间条文字加 "+"
    raci: {
      R: ['主责部门1', '主责部门2'],      // Responsible 主责执行
      A: ['Frank'],                     // Accountable 最终负责（通常 1 人）
      C: ['咨询部门'],                   // Consulted 需要咨询
      I: ['知会部门'],                   // Informed 仅知会
    },
    depends: ['M0'],                     // 可选，前置依赖模块编号
    tasks: [
      {
        id: '1.1',
        name: '任务标题（一行）',
        detail: '任务具体内容（30-200字）',
        owner: '主责部门名',
        support: ['配合1', '配合2'],     // 可空数组
        deliverable: '交付物',
        time: '月X-Y',
      },
      // ...
    ],
  },
  // ...
];

const WORKLOAD = [
  { dept: '部门名', total: 46, phases: [6, 16, 24], color: '#C62828' },
  // phases 是 3 阶段拆分（如 月1-4 / 月5-10 / 月11-18），求和需 = total
];

const OWNER_COLORS = {        // 主责标签的浅色背景
  '部门名': '#浅色 hex',
};
const OWNER_TEXT = {           // 主责标签的深色文字
  '部门名': '#深色 hex',
};
// 配色约定：每个部门用同色系，背景浅 / 文字深，色相区分（红/蓝/绿/紫/橙…）

# 渲染 JS（请完整使用以下代码，不要简化）

let currentDept = 'all';

function renderModules() {
  const container = document.getElementById('modules-container');
  container.innerHTML = '';
  const TOTAL_MONTHS = {{总月数}};  // 默认 18，等于 stats 里的实施周期

  MODULES.forEach((m) => {
    const card = document.createElement('div');
    card.className = 'module-card collapsed';
    card.style.borderColor = m.color;
    card.dataset.module = m.id;

    const barLeft = ((m.timeline[0] - 1) / TOTAL_MONTHS * 100).toFixed(1);
    const barWidth = ((m.timeline[1] - m.timeline[0] + 1) / TOTAL_MONTHS * 100).toFixed(1);

    let raciHtml = '';
    ['R','A','C','I'].forEach(role => {
      (m.raci[role] || []).forEach(dept => {
        raciHtml += `<span class="raci-chip ${role.toLowerCase()} dept-highlight" data-dept="${dept}"><span class="raci-letter">${role}</span> ${dept}</span>`;
      });
    });

    let taskRows = '';
    m.tasks.forEach(t => {
      const ownerBg = OWNER_COLORS[t.owner] || '#F5F5F5';
      const ownerColor = OWNER_TEXT[t.owner] || '#333';
      const supportHtml = t.support.map(s => `<span class="support-tag dept-highlight" data-dept="${s}">${s}</span>`).join('');
      taskRows += `<tr class="dept-highlight" data-owner="${t.owner}" data-support="${t.support.join(',')}">
        <td class="task-id">${t.id}</td>
        <td><b>${t.name}</b><br><span style="color:#757575;font-size:11px;">${t.detail}</span></td>
        <td><span class="owner-tag dept-highlight" data-dept="${t.owner}" style="background:${ownerBg};color:${ownerColor}">${t.owner}</span></td>
        <td>${supportHtml || '-'}</td>
        <td class="deliverable">${t.deliverable}</td>
        <td class="timeline-cell">${t.time}</td>
      </tr>`;
    });

    card.innerHTML = `
      <div class="module-header" onclick="toggleModule(this)">
        <div class="module-id" style="background:${m.color}">${m.id}</div>
        <div class="module-info">
          <div class="module-name">${m.name}</div>
          <div class="module-desc">${m.desc}</div>
        </div>
        <div class="module-timeline-bar">
          <div class="module-bar-fill" style="left:${barLeft}%;width:${barWidth}%;background:${m.color};">
            月${m.timeline[0]}-${m.timeline[1]}${m.ongoing ? '+' : ''}
          </div>
        </div>
        <div class="module-toggle">▶</div>
      </div>
      <div class="module-body">
        ${m.depends ? '<div style="margin-bottom:10px;padding:8px 12px;background:#FFF3E0;border-radius:8px;font-size:12px;color:#E65100;"><b>前置依赖：</b>' + m.depends.join(' → ') + ' 需先就绪</div>' : ''}
        <div class="raci-strip">${raciHtml}</div>
        <table class="task-table">
          <thead><tr><th style="width:40px">编号</th><th>任务项 / 具体内容</th><th style="width:90px">主责</th><th style="width:100px">配合</th><th style="width:100px">交付物</th><th style="width:80px">周期</th></tr></thead>
          <tbody>${taskRows}</tbody>
        </table>
      </div>
    `;
    container.appendChild(card);
  });
}

function toggleModule(header) {
  header.parentElement.classList.toggle('collapsed');
}

function filterDept(dept) {
  currentDept = dept;
  document.querySelectorAll('.filter-btn').forEach(btn => {
    btn.classList.toggle('active', btn.textContent.trim() === dept || (dept === 'all' && btn.textContent.trim() === '全部'));
  });

  if (dept === 'all') {
    document.querySelectorAll('.dept-highlight').forEach(el => el.classList.remove('dept-dimmed'));
    document.querySelectorAll('.module-card').forEach(c => { c.style.opacity = '1'; });
    return;
  }

  document.querySelectorAll('.module-card').forEach(card => {
    const mod = MODULES.find(m => m.id === card.dataset.module);
    const allDepts = [...(mod.raci.R||[]), ...(mod.raci.A||[]), ...(mod.raci.C||[]), ...(mod.raci.I||[])];
    const involved = allDepts.some(d => d.includes(dept) || dept.includes(d));
    card.style.opacity = involved ? '1' : '0.35';
  });

  document.querySelectorAll('.dept-highlight').forEach(el => {
    const d = el.dataset.dept || '';
    const owner = el.dataset.owner || '';
    const support = el.dataset.support || '';
    const match = d.includes(dept) || dept.includes(d) || owner.includes(dept) || support.includes(dept);
    el.classList.toggle('dept-dimmed', !match && d !== '' && (owner !== '' || support !== '' || d !== ''));
  });

  document.querySelectorAll('.task-table tbody tr').forEach(tr => {
    const owner = tr.dataset.owner || '';
    const support = tr.dataset.support || '';
    const match = owner.includes(dept) || support.includes(dept);
    tr.classList.toggle('dept-dimmed', !match);
  });
}

function renderWorkload() {
  const container = document.getElementById('workload-section');
  const maxTotal = Math.max(...WORKLOAD.map(w => w.total));
  let html = '';
  WORKLOAD.forEach(w => {
    const pct = (w.total / maxTotal * 100).toFixed(0);
    html += `
      <div class="workload-bar-row">
        <div class="workload-dept">${w.dept}</div>
        <div class="workload-bar-bg">
          <div class="workload-bar-fill" style="width:${pct}%;background:${w.color};">
            ${w.total}人月 (${w.phases.join('+')})
          </div>
        </div>
        <div class="workload-total">${w.total}人月</div>
      </div>`;
  });
  html += `<div style="text-align:center;font-size:11px;color:#999;margin-top:8px;">阶段划分：{{阶段定义文字，如 "月1-4 基础设施 / 月5-10 核心能力 / 月11-18 高阶能力"}}</div>`;
  container.innerHTML = html;
}

// 必须执行：渲染 + 首个模块默认展开
renderModules();
renderWorkload();
document.querySelector('.module-card')?.classList.remove('collapsed');

# 微交互

- 模块卡 hover：shadow → shadow-hover
- 模块卡点击 header：toggle .collapsed，▶ 旋转 90deg
- Filter Btn 点击：切换 active 状态
- 部门筛选：未涉及该部门的卡 opacity 0.35；未匹配的 chip/row 加 .dept-dimmed { opacity 0.15 }
- Workload Bar：CSS transition width 0.8s ease-out（首次渲染会有从 0 增长的动画——如果想要这个效果，可在 render 时先 width 0 再 setTimeout 设目标 width）
- 首屏：自动展开第一个模块

# 响应式（@media max-width 768px）

- .module-timeline-bar { display: none }     /* 移动端隐藏时间条，避免拥挤 */
- .stats-bar { grid-template-columns: repeat(2, 1fr) }
- .task-table { font-size: 11px }
- .workload-dept { width: 80px; font-size: 11px }

# 内容写作规范

## Module desc
- 一句话 ≤ 40 字，说清"这个模块解决什么 / 关键技术约束"
- 例："所有AI能力的地基，知识库质量直接决定推送和复盘准确度"

## Task name / detail
- name：动词开头短句（"搭建 XX"、"开发 XX 引擎"、"对接 XX"）
- detail：30-200 字，描述「具体做什么 / 怎么做 / 输出是什么」
  · 必要时用 ① ② ③ 列出子步骤
  · 必要时给出量化指标（如"识别率 >90%"、"<30g/8h续航"）
- owner：单一部门名，必须出现在 RACI.R 列表里
- support：可多个，可空
- deliverable：名词性短语（"XX 服务" / "XX 文档" / "XX 看板"）
- time：月份范围 "月X-Y"，或 "持续"，或 "月X 起，每 N 月一轮"

## RACI 填写规则
- R（Responsible）：实际干活的部门，可多个
- A（Accountable）：最终拍板/担责的人，强烈建议只有 1 个
- C（Consulted）：需要被咨询、提供输入的方
- I（Informed）：仅知会、不参与决策的方
- 一个部门在同一模块里不应同时出现在 R 和 A（除非 R=A）

# 节点数量建议

- Modules：6-14 个（典型 8-12）
- 每个 Module 的 tasks：3-7 个
- Stats Bar：固定 4 张
- Filter 部门按钮：实际出现的部门 + "全部"，建议 ≤ 10 个
- Workload Bars：6-10 个部门，phases 固定 3 阶段

# 填空区（请用户提供）

项目名:                            # 用于 Hero 标题
副标行:                            # 用于 Hero 副标，建议格式 "N模块/M角色/X月周期/治理模式"
主色档位: [绿色|蓝色|紫色|橙色|自定义]
实施总月数:                        # 默认 18，决定时间条比例
阶段定义文字:                      # 用于 workload 底部注释
                                  # 如 "月1-4 基础设施 / 月5-10 核心能力 / 月11-18 高阶能力"

Stats:  # 固定 4 项
  - { num: , label: , color: 'primary' }
  - { num: , label: , color: 'blue' }
  - { num: , label: , color: 'orange' }
  - { num: , label: , color: 'purple' }

部门清单:  # 用于 Filter Bar 按钮和 OWNER_COLORS 映射
  - { 名称: , 浅色bg: , 深色text: }

MODULES:  # 见上 schema
  - id: M1
    name: 
    color: 
    bg: 
    desc: 
    timeline: [1, 4]
    ongoing: false
    raci: { R: [], A: [], C: [], I: [] }
    depends: []   # 可选
    tasks:
      - { id: '1.1', name: , detail: , owner: , support: [], deliverable: , time: }

WORKLOAD:
  - { dept: , total: , phases: [a, b, c], color: }
  # phases 三段求和需 = total

Footer:
  公司全称: 
  slogan: 

# 自检 Checklist（生成后逐项核对）

[ ] 所有模块数据在 JS 数组里，HTML 没手写任何 .module-card
[ ] Module 时间条 left/width 比例正确（基于实施总月数计算）
[ ] 每个模块卡 border-left 颜色 = 该模块 color
[ ] RACI chip 四色对：R 绿 / A 红 / C 蓝 / I 灰
[ ] 每个 task tr 带 data-owner + data-support，便于 filterDept 工作
[ ] 主责标签背景/文字色匹配 OWNER_COLORS/OWNER_TEXT
[ ] 点击 Filter 按钮：未涉及的卡 opacity 0.35，未匹配的 chip/row 加 dimmed
[ ] 点击 module-header 能折叠/展开 + ▶ 旋转 90deg
[ ] 首屏第一个模块默认展开
[ ] 有 depends 的模块在 body 顶部显示橙色「前置依赖」横条
[ ] Workload 条形图 width 按 total/maxTotal 比例 + 文字 "X人月 (a+b+c)"
[ ] Workload 底部小字注释展示三阶段含义
[ ] 768px 下时间条隐藏 / stats 改 2 列 / task-table 字号缩小
[ ] 所有图标 emoji，零外部资源，文件可双击直接打开

# 输出指令

不要解释，不要包代码块，直接输出完整 HTML 内容。

===== 提示词正文结束 =====
```

---

## 二、使用贴士

1. **数据准备是关键**——这套 prompt 的产出质量 90% 取决于你填的 `MODULES` 数组。建议先在 Excel/Notion 里把每个模块的 `id / name / color / timeline / raci / tasks` 列清楚，再粘贴成 YAML 给 Claude。

2. **RACI 是这套页面的灵魂**，不要随便填。建议遵守：A 永远只有 1 人；R 列表的人都要在 tasks 的 owner 里至少出现一次；如果某模块所有 task 的 owner 都是同一个部门，那这个部门就是 R，不需要再列其他 R。

3. **配色映射小技巧**——OWNER_COLORS / OWNER_TEXT 不要随机配。每个部门给一个固定色系：AI 工程师红、数字信息部蓝、产品经理紫、硬件橙、战略绿、培训粉、油液 PM 黄……这样部门筛选时颜色记忆能强化。

4. **timeline 月数对齐**——所有模块的 `timeline[1]` 最大值要 ≤ Hero 副标和 Stats 里说的总月数；`renderModules` 的 `TOTAL_MONTHS` 要等于这个值，否则时间条比例会错。

5. **两套 prompt 配合使用**——同一个项目可以同时产出两份页面：用第一套对 CEO/客户讲愿景，用第二套对项目团队和 PMO 对齐执行。两份共享相同的模块清单（M1~MN）和配色，互相呼应、信息一致。

---

### Turn 3  *(2026-05-25 00:31)*

**User:**

两套 prompt 配合使用怎么做

**Assistant:**

核心思路：**维护一份「项目主数据」作为单一可信源（SSOT），两套 prompt 各自从中派生，永远不要重复编辑同一份信息**。

下面给你完整的工作流。

---

## 一、主数据模板（一份 YAML，两份页面都基于它）

```yaml
# ============ 共享部分（两套 prompt 都要用） ============

项目:
  名称: 安美AI销售赋能系统
  副标: amer 安美 · 更好的润滑保护
  主标语: 用AI武装每一位销售
  主色: '#43A047'        # primary
  主深: '#2E7D32'        # primary-dark
  主浅: '#E8F5E9'        # primary-light
  总月数: 18

部门清单:                  # 配色一旦定下，两套页面里所有出现这部门的地方都用同一对色
  - { 名称: AI工程师,         浅bg: '#FFCDD2', 深text: '#B71C1C' }
  - { 名称: 数字信息部,        浅bg: '#BBDEFB', 深text: '#0D47A1' }
  - { 名称: AI&IT产品经理,     浅bg: '#E1BEE7', 深text: '#4A148C' }
  - { 名称: 智能硬件组,        浅bg: '#FFE0B2', 深text: '#E65100' }
  - { 名称: 战略部,            浅bg: '#C8E6C9', 深text: '#1B5E20' }
  - { 名称: 经验萃取,          浅bg: '#B2DFDB', 深text: '#004D40' }
  - { 名称: 培训执行,          浅bg: '#F8BBD0', 深text: '#880E4F' }
  - { 名称: 油液化学品PM,      浅bg: '#FFF9C4', 深text: '#F57F17' }

模块清单:                  # ⚠ 这是两套页面"对得上"的根基，编号/名字/颜色必须完全一致
  - id: M1
    名称: 知识底座
    色: '#F57F17'          # 用于分工图的 border-left/id 块/时间条；全景图的 m1 tag 配色
    浅bg: '#FFF8E1'        # 配套浅色
    深text: '#F57F17'
    一句话: 所有AI能力的地基，知识库质量决定推送和复盘准确度
  - id: M2
    名称: 行业调研智能体
    色: '#2E7D32'
    浅bg: '#E8F5E9'
    深text: '#2E7D32'
    一句话: 模仿战略部已有的行业解决方案SOP，自动化调研报告生成
  # ... M3~MN

# ============ 全景图（第一套）独需 ============

全景图:
  Hero_KPI:
    - { 数字: 14,    标签: 核心模块 }
    - { 数字: 600+,  标签: 一线销售覆盖 }
    - { 数字: 15,    标签: 国家/地区 }
    - { 数字: 35%,   标签: 目标效率提升 }

  Journey_主线介绍: 跟随一位销售，从AI线索孵化到深夜复盘学习……
  Journey_节点:
    - 时间标签: 7:30
      emoji: 🔍
      徽章色: '#43A047'
      标题: AI行业研究
      阶段tag: 拜访前
      场景: 早上到办公室，打开企微，AI已为今天要拜访的客户准备好简报……
      痛点: 拜访前准备全靠经验和百度，资深销售花30分钟做功课，新人不知道查什么……
      方案: AI行业研究Agent自动抓取数据源 → 结合知识库 → 生成定制化简报 → 推送企微
      价值: 准备时间从30分钟降至3分钟 · 新人也能带专业信息上场 · 客户第一印象专业
      关联模块: [M2, M1]                  # ⚠ 必须引用主数据里已存在的模块编号
    # ... 8-12 个节点

  管理者视角:
    - { emoji: 📊, 标题: 思迈特BI驾驶舱, desc: ..., value: ... }
    # ... 6-10 张

  Flywheel:
    中心: 数据飞轮
    圆周节点: [更多/拜访数据, 更精准/商机预测, 更智能/客户孪生, 更高/转化率, 更多/成功案例, 跨区域/知识迁移]
    闭环步骤: [...]
    收尾: 飞轮每转一圈，14个模块同时变强……

  Matrix:
    - { 痛点: 拜访准备靠经验和搜索, 方案: AI行业研究Agent+知识库, 数字价值: '准备时间 -90%', 细节: 30分钟→3分钟 }
    # ... 8-13 条

# ============ 分工图（第二套）独需 ============

分工图:
  Stats:
    - { num: 14,   label: 核心模块,   color: primary }
    - { num: 10,   label: 团队角色,   color: blue }
    - { num: 130,  label: 总人月投入, color: orange }
    - { num: 18,   label: 月实施周期, color: purple }

  阶段定义: 月1-4 基础设施 / 月5-10 核心能力+智能层 / 月11-18 高阶能力+实时推送

  模块详情:                # 与"模块清单"按 id 一一对应，这里补充 RACI/tasks/depends
    - id: M1               # ⚠ 必须用主数据里已有的 id
      timeline: [1, 4]
      ongoing: true
      raci:
        R: [经验萃取, 战略部, AI工程师]
        A: [Frank, 销售总经理]
        C: [AI&IT产品经理, 培训执行, 油液化学品PM]
        I: [数字信息部]
      depends: []
      tasks:
        - { id: '1.1', name: 六脉神剑SOP结构化, detail: ..., owner: 经验萃取, support: [销售总经理, 培训执行], deliverable: 结构化SOP+话术库, time: 月1起每2月一轮 }
        # ...
    # ... M2~MN

  WORKLOAD:
    - { dept: AI工程师,         total: 46, phases: [6,16,24], color: '#C62828' }
    - { dept: 数字信息部,       total: 25, phases: [3,11,11], color: '#1565C0' }
    # ... 注意 phases 三段求和需 = total

公司:
  全称: 安美科技股份有限公司 AMER TECHNOLOGY CO.,LTD.
  slogan: 安美科技 · AI赋能每一次拜访
```

---

## 二、推荐工作流（3 步）

**Step 1 · 先把「主数据」的共享部分定稿**（30 分钟到几小时）

只填上半部分：项目信息、部门清单、模块清单（id/名称/色/一句话）。这步是项目本身的功课，不是 prompt 工程。建议跟项目相关人对一遍，编号和颜色一旦发出去就别再改。

**Step 2 · 先生成「分工图」**

把主数据里的「共享部分 + 分工图独需」字段套入第二套 prompt 的填空区，让 Claude 生成 HTML。为什么先做这个？因为 RACI/tasks/timeline 是事实，逼着你想清楚"谁主责、几月做、依赖谁"。这些事实想清楚了，下一步的故事才不会飘。

**Step 3 · 再生成「全景图」**

把主数据里的「共享部分 + 全景图独需」字段套入第一套 prompt 的填空区。写 Journey 节点的「场景描述」时，每个节点的 `关联模块: [Mx, My]` 必须引用主数据里已有的编号——这样配色自动一致，读者打开两份页面，看到"M5 紫色"就知道是同一个模块。

---

## 三、必须保持一致的项目（一致性 checklist）

打开两份页面对照，如果以下任何一项对不上，就回主数据里改、重新生成，不要在某一份 HTML 里手改：

- [ ] 模块编号 M1~MN 一一对应
- [ ] 同编号的模块名完全一致
- [ ] 同编号的模块配色（hex 值）完全一致
- [ ] 部门名称完全一致（"AI&IT产品经理" 不能写成 "AI产品经理"）
- [ ] 部门数量、总人月、实施月数 = 全景图 Hero KPI 里的对应数字
- [ ] 主标语/公司全称/slogan 完全一致

> 顺序很重要：第一次定稿后，所有后续修改都回主数据，再重新跑两套 prompt。手改某一份 HTML 后两份会越来越偏。

---

## 四、实操小技巧

1. **把主数据存成 `project-master.md`**——放在项目目录，每次让 Claude Code 读这个文件 + 提示词文件，比每次粘贴方便。

2. **让 Claude 帮你做一致性校验**——两份 HTML 生成完后，发一句："读 `quanjing.html` 和 `fengong.html`，比对所有 M1~M14 的名称和颜色，列出不一致的地方"。它能在 30 秒里发现你手抖改过的地方。

3. **配色档别超过 14 个**——M1~M14 我给的色谱是经过试色的舒适搭配，超过 14 个色相会开始打架。模块多就考虑分组（M1-M7 基础层 / M8-M14 高阶层），分两批做。

4. **想把两份页面联动**——可以在两份页面顶部加同样的导航条 `<a href="quanjing.html">愿景</a> | <a href="fengong.html">执行</a>`，做成一个"项目门户"的两个视图。

5. **数据规模超过单一 prompt 上下文了怎么办**——14 模块 × 5 任务的分工图大概接近 prompt 极限。如果你的项目有 20+ 模块，分两次生成：第一次给 M1-M10、第二次给 M11-M20，最后把生成的 JS 数据数组手工合并到一个 HTML 里。

---
