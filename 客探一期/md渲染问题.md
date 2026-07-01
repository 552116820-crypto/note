● 这是一个常见的 Markdown 渲染问题。原因是 Dify API 返回的文本中，列表项之间使用了 \n\n（双换行），前端 Markdown
  解析器会把双换行的列表项当作**松散列表（loose list）**来渲染，每个 <li> 内部会包裹 <p> 标签，导致额外的段落间距。

  解决方案（任选其一）：

  1. 前端 CSS 修复（最简单）：
  li p {
    margin: 0;
  }

  2. 在前端渲染前预处理文本，去掉列表项之间的多余空行：
  // 将列表项之间的双换行替换为单换行
  const processed = text.replace(/(\n\d+\..*)\n\n(\d+\.)/g, '$1\n$2');
  // 或者更通用的处理
  const processed = text.replace(/(\n[-*\d].*)\n\n([-*\d])/g, '$1\n$2');

  3. 在 Dify 的 prompt 中要求输出时列表项之间不要加空行。

  核心原因就是：Markdown 规范中，列表项之间有空行 = 松散列表（每项包 <p>），没空行 = 紧凑列表（不包 <p>）。Dify
  预览可能做了特殊处理，但标准 Markdown 解析器（如 marked、markdown-it）会严格按规范渲染。

  推荐方案 1，改一行 CSS 就能解决。
