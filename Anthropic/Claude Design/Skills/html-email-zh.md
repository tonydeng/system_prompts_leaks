> **说明**：本文件为英文原文（`html-email.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# HTML 邮件

设计 HTML 邮件作为一个自包含 .html 文件，能在真实邮件客户端中存活。邮件渲染不是浏览器渲染——Gmail、Outlook 和 Apple Mail 各自剥离或破坏不同的东西，下面的规则是在三者中可靠存活的内容。当此处的规则与正常网页设计直觉冲突时，规则胜出。

布局和样式：
- 用嵌套 <table role="presentation" cellpadding="0" cellspacing="0" border="0"> 构建——不用 flexbox、grid、float、position。一个居中的包装表格，max-width 600px，单列流（堆叠行胜过并排列）。
- 在所样式化的元素上内联每个样式。<head> 中的 <style> 块可额外携带无法内联的内容（媒体查询、深色模式调整）——一些客户端完全丢弃它，所以邮件必须仅凭内联样式正确阅读。
- 任何地方都不要 JavaScript（普遍被剥离）。不要外部样式表。不要 web 字体——使用邮件安全字体栈（Arial、Helvetica、Georgia、Verdana、Tahoma、'Courier New'）配通用回退。
- 用彩色表格单元格、边框、间隔单元格和字体构建视觉设计——而非图像。这里无处托管项目图像：引用的项目文件对收件人不存在。如果图像至关重要，留一个清晰标记的占位单元格配 alt 文本，告诉用户在发送前替换为托管的 https URL。
- 按钮是"防弹"的：带 padding 的 <td> 配 bgcolor 和内联 border-radius，<a> 填充它配 display:block 和内联 color——永不用图像，永不用样式化的 <button>。

重要的客户端怪癖：
- Outlook（Word 引擎）：给每个表格/单元格显式宽度；用 mso-line-height-rule:exactly 设置 line-height；将 Outlook 专用修复包裹在 <!--[if mso]> … <![endif]--> 条件注释中。
- Gmail 裁剪超过 ~100KB HTML 的消息——保持远低于此。
- 添加 <meta name="color-scheme" content="light dark"> 并选择能在深色模式反转中存活的颜色（避免纯 #000/#fff 背景；在中色调填充上测试文本）。

可送达性和可访问性：
- <body> 中的第一个元素：隐藏的预览头 span（~85 字符），在主题行旁边预览。
- 任何图像加 alt 文本，<html> 加 lang，真实 <a href> 链接（无死 # 锚点），页脚带合理的退订行和邮政地址（对任何营销性质的内容）。

以 600px 展示设计；在回复中提及该文件是发送就绪的 HTML，用户可放入其邮件工具。
