> **说明**：本文件为英文原文（`wireframe.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# 线框图

帮助用户快速探索设计想法。先与他们访谈，然后生成多个粗略线框图，在确定方向之前映射设计空间。优先追求广度而非精致：为每个想法展示 3-5 种截然不同的方案。使用简单的形状、占位文本和最少的颜色，以保持对结构和流程的关注。使用草图风格，手写但可读的字体；黑白为主加一些颜色；低保真且简单。将线框图排列为垂直选项堆叠：

以垂直堆叠的轮次呈现多个设计选项，每轮选项是一个独立的 `<section>`，最新轮次在**顶部**，每个选项获得一个稳定的 `{turn}{letter}` ID（`1a`、`1b`、`2a`…），用户在聊天中引用它，你在轮次之间交叉链接。始终在 `<helmet>` 中包含 `<meta name="design_doc_mode" content="canvas">`，宿主提供平移/缩放功能，因此用户可以在比视口更宽的设计上自由缩小查看。

**编写方式** — 在 `<helmet>` 中放一个 `<style>` 块，然后每轮一个 `<section class="dv-turn">` 作为**根元素的直接子元素**（紧跟 `</helmet>` 之后，无包裹层）。当用户要求新一轮时，**将新 section 插入到已有 section 之上**，使最新工作位于顶部；绝不重新排序、重新编号或删除之前的轮次。

```html
<helmet data-dc-atomics><meta name="design_doc_mode" content="canvas"><style>body{margin:0;background:#f0eee9;font-family:system-ui,sans-serif}.dv-turn{padding:40px 44px 32px;border-bottom:1px solid rgba(0,0,0,.08);scroll-margin-top:16px}.dv-thd{display:flex;align-items:baseline;gap:10px;margin:0 0 20px}.dv-tid{font:600 10px ui-monospace,Menlo,monospace;padding:3px 7px;background:#1a1a1a;color:#fff;border-radius:4px;text-decoration:none}.dv-tname{font:600 13px/1.2 system-ui,sans-serif;color:#1a1a1a}.dv-opts{display:flex;flex-wrap:wrap;gap:28px;align-items:flex-start}.dv-opt{flex:none;display:flex;flex-direction:column;gap:9px;scroll-margin-top:16px}.dv-oid{font:600 10.5px ui-monospace,Menlo,monospace;padding:3px 7px;background:rgba(0,0,0,.08);color:#1a1a1a;border-radius:5px;text-decoration:none}.dv-olabel{display:flex;align-items:baseline;gap:8px;font:400 11px/1.3 system-ui,sans-serif;color:rgba(0,0,0,.55)}.dv-card{max-width:100%;background:#fff;border:1px solid rgba(0,0,0,.08);border-radius:8px;box-shadow:0 1px 3px rgba(0,0,0,.06);overflow:hidden}.dv-opt:target .dv-oid{background:#2a78d6;color:#fff}.dv-next{margin:22px 0 0;font:12px/1.5 system-ui,sans-serif;color:rgba(0,0,0,.5)}</style></helmet>
<section class="dv-turn" id="t2">
<div class="dv-thd"><a class="dv-tid" href="#t2">2</a><span class="dv-tname">Riffs on <a class="dv-oid" href="#1b">1b</a></span></div>
<div class="dv-opts">
<div class="dv-opt" id="2a"><div class="dv-olabel"><a class="dv-oid" href="#2a">2a</a>Tighter spacing</div><div class="dv-card" style="width:360px">…design…</div></div>
<div class="dv-opt" id="2b">…</div>
</div>
<p class="dv-next">Try next: "more like <a class="dv-oid" href="#2a">2a</a> but with the serif from <a class="dv-oid" href="#1c">1c</a>" · "make <a class="dv-oid" href="#2b">2b</a> full-bleed" · "new directions"</p>
</section>
<section class="dv-turn" id="t1">…turn 1, unchanged…</section>
```

**规则：** 轮次 section ID 为 `t1`、`t2`、`t3`…；选项 ID 为 `1a`、`1b`、`2a`…，放在选项的**最外层**元素（`.dv-opt`）上，绝不放在徽章上，这样 `#1b` 可以将整个选项滚动到视图中。ID 永久稳定，绝不复用或重新编号。一轮中的选项在换行排列中并排放置；不要自己手写平移/缩放，宿主画布已提供。文件中**每个**选项 ID 引用，无论是轮次标题、选项标签、`.dv-next` 行还是任何正文，都是 `<a class="dv-oid" href="#1b">1b</a>` 链接，绝不写裸 `1b`；在你的聊天回复中，只需写 `1b`。每轮以一行 `.dv-next` 结尾，提供 2-3 个用户可以粘贴到聊天中的英文后续建议。每个 `.dv-card` 根据内容设置尺寸（显式宽度可以）；不要使用 `height:100%`。
