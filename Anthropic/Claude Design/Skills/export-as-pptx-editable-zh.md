# 导出为 PPTX（可编辑）

将 HTML 幻灯片导出为包含原生 PowerPoint 对象（可编辑文本、形状、图像）的 `.pptx` 文件。一次 `gen_pptx` 工具调用完成所有操作：捕获、字体处理、生成、下载。

## 你需要做什么

1. **了解这份幻灯片。** 你很可能编写了它。如果没有，用 `read_file` 读取 HTML 来查找：幻灯片选择器、如何导航（函数名？类切换？）、使用什么字体、是否有缩放包裹器。
2. **`show_to_user`** 展示幻灯片，使其出现在用户预览中。
3. **调用 `gen_pptx`**，使用以下输入。
4. **阅读结果中的验证标志**，决定是否需要重试。

## gen_pptx 输入

```jsonc
{
  "width": 1920, "height": 1080,   // CSS 像素 — 匹配幻灯片的尺寸
  "slides": [                      // 每张幻灯片一个条目，按顺序
    { "showJs": "goToSlide(0)", "selector": ".slide.active" },
    { "showJs": "goToSlide(1)", "selector": ".slide.active" }
    // 如果所有幻灯片同时在 DOM 中且无需导航：
    //   { "selector": ".slide:nth-child(1)" }, { "selector": ".slide:nth-child(2)" }
  ],
  "hideSelectors": [".nav", ".progress", "[data-omelette-chrome]", "[data-noncommentable]"],
  // 如果幻灯片用 transform:scale() 容器包裹，在此指定。
  // gen_pptx 会清除 transform 并强制设置此元素的 width/height。
  "resetTransformSelector": ".slide-container",
  // 字体处理 — 根据底部的指令选择一种策略。
  // 替换在捕获之前进行，使布局正确重排。
  "googleFontImports": ["Poppins", "Lora"],
  "fontSwaps": [{ "from": "BrandSans", "to": "Poppins" }],
  // 或 fontSwaps: [{from:"BrandSans", to:"Arial"}] 使用 Web 安全字体。
  // 或两者都省略，保持品牌字体原样。
  "filename": "my-deck"
}
```

当且仅当用户要求 Google Slides 时，还需传 `"offer_google_slides": true`：导出对话框会增加一个 "Send to Google Slides" 按钮，仅在用户点击时才上传。

`slides[].showJs` 在 iframe 内作为同步表达式运行，不要 `await`。如果你的幻灯片导航函数是异步的，不带 await 调用它即可；每张幻灯片的 `delay`（默认 600ms）覆盖过渡时间。CSS 过渡较长的幻灯片可增大 `delay`。

### 如果幻灯片使用 `<deck-stage>` 起始组件

- `resetTransformSelector: "deck-stage"` — 导出器在其上设置 `noscale` 属性，组件观察到后移除其 shadow-DOM 中的 `transform: scale()`。无法通过其他方式访问缩放画布。
- `slides[N].showJs`: `"document.querySelector('deck-stage').goTo(N)"` — 从 0 开始索引，所以第 1 张幻灯片是 `goTo(0)`。
- `slides[N].selector`: `"deck-stage > [data-deck-active]"`。
- `hideSelectors` 不需要，覆盖层和点击区域在 shadow DOM 中，不会被捕获。

## 演讲者备注

从 `<script type="application/json" id="speaker-notes">` 自动读取并按索引附加。你无需传递它们。

## 验证标志

结果列出标志。**这些是警告而非错误**，阅读每条消息并判断对此幻灯片是否为预期情况：

- `duplicate_adjacent` / `duplicate_majority` — 幻灯片捕获内容完全相同。几乎总是意味着 `showJs` 未导航。检查函数名，尝试更长的 `delay`，或检查幻灯片使用 0 索引还是 1 索引。
- `slide_size_mismatch` — 捕获区域不匹配 width/height。选择器可能匹配了包裹器，或需要设置 `resetTransformSelector`。
- `notes_uniform_nonempty` — 每条演讲者备注都是相同的字符串。可能是占位符。如果是有意为之则无妨。
- `notes_count_mismatch` — #speaker-notes 长度 ≠ 幻灯片数量。备注按索引附加，尾部会错位。
- `no_speaker_notes` — 幻灯片没有 #speaker-notes 标签。如果没有备注则属预期。
- `fonts_timeout` — fonts.ready 耗时 >8 秒。字体 URL 可能不可达。
- `font_swap_failed` — 一个或多个 `fontSwaps` 目标未加载（拼写错误的字体族名，或 Google Fonts 不提供该字体），导致幻灯片用回退字体布局而文件中指定的是替换字体。用修正或不同的字体族名重试，或回退到 Web 安全字体。无论后续做什么，直接告诉用户哪些字体无法应用，例如 "提醒：导出时无法加载 Poppins，幻灯片使用了替代字体，文字换行可能不同。要我换个字体试试吗？"
- `images_failed` — 图片在捕获前未解码。通常是 404 或 CORS 问题。
- `reset_selector_miss` — 你的 `resetTransformSelector` 未匹配到任何元素。

如果标志看起来是真正的问题，修正输入并重试。如果是预期的（幻灯片确实没有备注、两张幻灯片确实相同），告诉用户下载已开始，继续操作。

**向用户说明标志时：** 这些名称和消息是内部诊断信息，不要原样传达。如果一切正常，完全不要提及验证，只确认下载。如果有看起来真正不对的地方，用通俗语言描述，不提标志名称或技术细节，例如 "糟糕，演讲者备注可能没有正确导出。" 而非 "我收到了 no_speaker_notes 标志"，或 "有几张幻灯片可能捕获了相同内容，我来修正导航并重试。" 而非引用 `duplicate_adjacent`。

捕获后页面自动重新加载，DOM 变更（隐藏的界面元素、字体替换）会被还原。

## 字体策略

阅读本提示词末尾的指令并将其转换为输入：

| 指令 | 输入 |
|---|---|
| 品牌字体原样保留 | 省略 `googleFontImports` 和 `fontSwaps` |
| Web 安全字体替换 | `fontSwaps: [{from:"EachCustomFont", to:"Arial"}]`（或 Georgia 用于衬线体，Courier New 用于等宽体） |
| Google Fonts 替换 | `googleFontImports: ["Poppins","Lora"]` + `fontSwaps: [{from:"EachCustomFont", to:"Poppins"}]` |

系统字体（Arial、Helvetica、Georgia、Times、Courier、sans-serif 等）保持不动。
