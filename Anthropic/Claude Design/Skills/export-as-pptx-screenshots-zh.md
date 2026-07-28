> **说明**：本文件为英文原文（`export-as-pptx-screenshots.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 导出为 PPTX（截图模式）

将 HTML 幻灯片导出为 `.pptx`，作为全出血 PNG 图像。像素完美，不可编辑。一次 `gen_pptx` 工具调用。

## 步骤

1. `show_to_user` 展示幻灯片。
2. 调用 `gen_pptx`：

```jsonc
{
  "mode": "screenshots",
  "width": 1920, "height": 1080,
  "slides": [
    { "showJs": "goToSlide(0)", "selector": "body" },  // selector 在截图模式下未使用但必填
    { "showJs": "goToSlide(1)", "selector": "body" }
  ],
  "hideSelectors": [".nav", ".progress"],
  // 如果幻灯片包裹在 transform:scale() 容器中，在此指定它，
  // 以便幻灯片在锁定的 iframe 内强制为 width × height。
  "resetTransformSelector": ".slide-container",
  "filename": "my-deck"
}
```

仅当用户要求 Google Slides 时，才传 `"offer_google_slides": true`：导出对话框会获得"发送到 Google Slides"按钮，仅当他们点击时才上传。

`slides[].delay` 默认 600ms——过渡较慢时调高。

### 如果幻灯片使用 `<deck-stage>` 起始组件

- `resetTransformSelector: "deck-stage"`——与可编辑模式相同；组件丢弃其 shadow-DOM `transform: scale()` 使幻灯片填充锁定的 iframe。
- `slides[N].showJs`: `"document.querySelector('deck-stage').goTo(N)"`——0 索引，所以幻灯片 1 是 `goTo(0)`。
- `hideSelectors` 不必要——覆盖层和点击区域位于 shadow DOM 中，不会被捕获。

## 验证

与可编辑模式相同的标志。注意 `duplicate_adjacent`（showJs 未导航）和 `reset_selector_miss` / `slide_size_mismatch`（你的 `resetTransformSelector` 未匹配任何内容或未缩放到 width × height）。

`#speaker-notes` 的演讲者备注会自动附加。页面之后重新加载。
