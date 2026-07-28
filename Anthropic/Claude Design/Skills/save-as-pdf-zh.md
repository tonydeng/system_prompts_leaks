# 保存为 PDF

将当前 HTML 设计重新排版为分页的、适合打印的 PDF。"即时"导出已经以设计的原始像素尺寸给用户一个 PDF——这条路径适用于用户需要真正的分页页面时。

**不要将页面光栅化为 PDF。** 绝不使用 jsPDF、html2canvas、dom-to-image 或任何其他画布/截图转 PDF 方案——它们产生模糊、不可选择、过大的输出，也不要自己生成 PDF 二进制数据。PDF 导出基于打印：一份打印就绪的副本交给 `show_pdf_export_dialog`，浏览器自身的打印引擎渲染出清晰、可选择、基于文本的页面。让副本打印就绪的唯一支持方式是拥有其打印几何的组件——文档用的 doc_page 起始组件，或已经基于 `<deck-stage>` 或 `<doc-page>` 构建的源文件。不要手写 `@page` 规则或打印 CSS 重置。

## 步骤

1. **阅读当前 HTML 设计文件**以了解其结构和内容。每次 PDF 请求都要重新阅读，即使你之前在本对话中读过或制作过打印副本——用户可能在此期间更改了内容或调整了数值（调整面板会写入源文件）。

2. **编写打印副本。** 始终从你刚读的源文件重新编写——之前请求留下的 `-print` 副本是过时快照，复用或仅部分更新它会把过时数值带入 PDF。打印文件路径是源路径在扩展名前插入 `-print`——同目录、同文件名。如果源文件是 `slides/deck.html`，写 `slides/deck-print.html`；如果源文件是 `web/index.html`，写 `web/index-print.html`。**不要**使用演示标题或项目名作为文件名，**不要**在源文件位于子目录时写到项目根目录——目录深度的任何变化都会破坏所有相对 URL（`@font-face` `src: url(...)`、`<img src>`、`<link href>`、CSS `background: url(...)`），导致打印标签页显示缺失图片和系统字体回退。

   **如果源文件已经基于 `<deck-stage>` 或 `<doc-page>` 构建，副本就是源文件加上内容级打印规则。** 两个组件都拥有其打印几何——绝不添加 `@page` 规则或重新排布其布局。对于 `<deck-stage>` 演示文稿，在**每个**直接子级幻灯片上设置 `data-deck-active`（不只是当前那个），这样基于 `[data-deck-active]` 的入场样式在每一页都能解析——每张幻灯片已经是一页。对于 `<doc-page>` 文档，结构上无需做任何事。

   **否则，将内容重建为基于 doc_page 起始组件的分页文档。** 调用 `copy_starter_component` 并传入 `kind: "doc_page.js"`（每个项目一次——组件文件会持久化），然后将打印副本写为 `<doc-page size="letter" margin="0.75in">` 文档（用户明显非美国时用 `size="a4"`）：将源文件的内容作为普通流式 HTML 倒入其中，保持设计的排版、颜色和图像完整。组件拥有页面、分页和所有打印几何——**不要**自己写 `@page` 规则、打印 CSS 重置、页面卡片 div 或 `break-after: page` 假页面。只在某个部分真正开始新章节时使用 `break-before: page`；长表格加 `<thead>` 让表头在每页重复。

   **固定画布设计（海报、社交图片、信息图）也通过 doc_page 重建，有一个决策点：页面。** 按其真实尺寸打印——`<doc-page width="18in" height="24in" margin="0">`，页面即设计——或缩放到标准纸张——`<doc-page size="letter" content-width="960px" content-height="1440px">`，内容以其创作尺寸布局，组件缩放以适应可打印区域。当用户的意图从请求中不清楚时，在导出前用通俗的话问——按海报尺寸打印，还是适配到 letter 纸张？——绝不用自己的 CSS transform 手动缩放；组件负责缩放。

   在每份副本中，添加 color-adjust 规则使背景和颜色与预览匹配——**不要**从设计中去除背景：
   ```css
   * { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
   ```

   **将跳转动画置于终态。** **不要**使用 `animation: none`（那会让淡入恢复到隐藏基态）。相反，将每个动画冻结在最后一帧并禁用过渡：
   ```css
   * { animation-delay: -99s !important; animation-duration: .001s !important;
       animation-iteration-count: 1 !important; animation-fill-mode: both !important;
       animation-play-state: running !important; transition-duration: 0s !important; }
   ```

3. **测试文件**：用 `show_html` 展示，确保没有 JS 错误。除非被要求，无需截图。

4. **调用 `show_pdf_export_dialog` 工具**，传入打印就绪文件的项目相对路径。调用此工具时，打印触发代码会自动注入到打印副本中——**不要**自己写自动打印或 `window.print()` 脚本。除非导出来自用户自己的 Export 点击，否则工具本身不会打开任何东西：它呈现一个导出对话框，用户在其中点击 **Continue to export** 才会打开打印视图。工具结果会说明发生了什么——当它说对话框在等待用户时，打印视图**尚未**打开；如实说明，绝不声称已打开。如果文件（及其打印源）不是基于打印的，工具会带原因失败——步骤 2 的 doc_page 重建正是保持副本合格的关键。

## 重要说明

- 目标是一个能在真实页面上干净打印的文件——doc_page 组件负责分页；你的工作是内容
- 保持视觉保真度——保持设计的排版、颜色和图像完整
- 对于 `<deck-stage>` 演示文稿，每张幻灯片留在各自页面；`<doc-page>` 文档自行流动和分页
- 对于提示驱动的导出，`show_pdf_export_dialog` 等待用户：导出对话框的 **Continue to export** 点击打开打印视图，在此之前什么都没打开——报告工具结果描述的状态，而非你期望的状态
- `-print.html` 是打印标签页的管道，不是交付物——`show_pdf_export_dialog` 是唯一的交付步骤。**不要**对它调用 `present_fs_item_for_download`；其相对资源路径只能通过项目文件服务器解析，独立打开时会失效。
