# 导出为独立 HTML

将当前设计导出为一个完全自包含的 HTML 文件，完全离线工作，无外部依赖。

## 工作原理

有一个确定性打包器（super_inline_html 工具）可以内联在 HTML 属性中直接引用的资源，包括 img src/srcset、source src/srcset、video/audio/track src、video poster、SVG `<image href>`/`<use href>`、link href（样式表、favicon）、script src、CSS url() 和 @import、内联 style 属性。但是，它无法发现仅在 JavaScript 或 JSX 代码中作为字符串引用的资源，例如：
- React 中设置的图片 src：`<img src={"./hero.png"} />`
- styled-component 中的背景 URL：`background: url('./pattern.svg')`
- 动态导入的脚本

你的工作是准备好 HTML 文件使打包器能捕获所有内容，然后运行它。

## 步骤 1：复制 HTML 文件并更新代码引用的资源

复制当前 HTML 文件。读取它。复制其依赖项。查看所有代码（内联脚本、导入的 JSX 文件、styled-components 等）中任何作为字符串而非 HTML 属性引用的资源 URL。这包括：
- React/JSX 中的图片 URL（`<img src={...} />`、`style={{ backgroundImage: ... }}`）
- CSS-in-JS 中的 URL（styled-components、通过 JS 设置的内联样式）
- 导入其他脚本而这些脚本本身又引用资源的 script 标签
- 任何加载资源的 fetch() 或 XMLHttpRequest 调用
- 通过编程方式设置的音频/视频源

注意：如果你在项目中使用 Anthropic API，它将无法独立运行。如果这是项目的核心功能，停止并告知用户！

## 步骤 2：添加 ext-resource-dependency meta 标签

对于步骤 1 中找到的每个资源，在 `<head>` 中添加一个 `<meta>` 标签：

```html
<meta name="ext-resource-dependency" content="<url>" data-resource-id="<id>" />
```

其中：
- `content` 是资源的 URL（相对于 HTML 文件或绝对路径）
- `data-resource-id` 是简短的唯一标识符（例如 "heroImage"、"patternSvg"）

然后更新代码，用 `window.__resources[id]` 替代硬编码的 URL。在打包文件的运行时中，`window.__resources[id]` 将包含指向内联资源数据的 blob URL。

示例：
```html
<!-- 在 <head> 中： -->
<meta name="ext-resource-dependency" content="./hero.png" data-resource-id="heroImg" />
<meta name="ext-resource-dependency" content="./pattern.svg" data-resource-id="patternBg" />

<!-- 在代码中，将： -->
<!-- <img src={"./hero.png"} /> -->
<!-- 替换为： -->
<!-- <img src={window.__resources.heroImg} /> -->
```

重要：
- `content` 中的相对路径是相对于 HTML 页面本身的
- 对于导入并自身引用资源的任何外部 script 标签，也必须这样做，这些脚本会被打包器内联，但它们的资源引用也需要提升
- 要彻底！遗漏哪怕一个资源都意味着最终文件中有损坏的图片或缺失的资产

## 步骤 3：创建缩略图（必需，打包器在没有它的情况下会拒绝文件）

创建一个轻量级 SVG 缩略图，在打包文件解包时充当启动画面。此 SVG 应是设计的简化代表性预览，例如关键形状、布局轮廓或品牌加载视觉。它不需要像素级精确，只需视觉上有代表性，让用户能立即看到有意义的内容。它会被很小地显示，所以简单图标搭配鲜艳背景色就够了。

将其作为 `<template>` 标签添加到源 HTML 中：

```html
<template id="__bundler_thumbnail" data-bg-color="#0a5e3e">
  <svg viewBox="0 0 1200 800" xmlns="http://www.w3.org/2000/svg">
    <!-- 简化的图标 -->
  </svg>
</template>
```

- 将 `data-bg-color` 设置为与页面背景色匹配
- SVG 应使用 `viewBox` 以正确进行纵横比适配缩放
- 保持简单，这只是加载占位符，不是完整复现
- 使用设计的实际颜色，使过渡感觉无缝

打包器会提取此缩略图，在解包资产时全屏显示它（以背景色进行纵横比适配），然后用真实页面替换它。当 JavaScript 被禁用时，它也作为永久回退保持可见。

## 步骤 4：运行打包器

如果你在步骤 1-3 中做了更改，先保存修改后的 HTML 文件。然后（或如果无需更改）调用：

```
super_inline_html({ input_path: "<path-to-html>", output_path: "My Deck.html" })
```

给输出文件取一个友好的名称。

## 步骤 5：验证（仅内部检查）

**首先读取工具结果**，如果有任何资产无法解析，super_inline_html 会在其输出中直接列出（"N asset(s) could not be bundled: - asset not found: ./foo.png"）。这是权威的缺失列表，修复这些引用并在打开任何内容之前重新运行。

然后用 show_html 打开打包输出以检查其是否正常工作，这是为你准备的私有验证步骤，不是交付机制。检查 get_webview_logs 中的运行时错误（JS 异常、解码失败）。如果有问题，修复源文件并重新运行。

## 步骤 6：提供下载，强制步骤

你必须使用 **present_fs_item_for_download** 直接指向内联 HTML 输出来交付最终文件。这是交出独立导出的唯一正确方式。

- 不要使用 show_html / show_to_user 作为交付步骤，它们是预览工具，不是下载工具。用户无法从中保存文件。
- 不要询问用户是否想下载，直接调用 present_fs_item_for_download。
- 如果你跳过此步骤，用户将无法获取文件。此步骤不可协商。
