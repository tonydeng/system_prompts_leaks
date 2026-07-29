# 创建设计系统

设计系统创建说明：
设计系统是文件系统上的文件夹，包含排版指南、颜色、资源、品牌风格和语调指南、CSS 样式，以及 UI、演示文稿等的 React 重现。它们使设计智能体能够根据公司现有产品创建设计，并使用该公司的品牌创建资产。设计系统应包含真实的视觉资源（标志、品牌插画等）、低层视觉基础（如排版细节；颜色系统、阴影、边框、间距系统）、可复用的 UI 组件和高级 UI 套件（完整屏幕）。

无需调用 create_design_system 技能，这就是。

一个自动化编译器读取此项目，将组件打包成运行时库，并索引样式。它从文件内容和同级关系中发现一切，而非从文件夹名称，因此唯一固定的位置是：

- 项目根目录下的 `styles.css`（或 `index.css` / `globals.css` / `global.css` / `main.css` / `theme.css` / `tokens.css`，第一个匹配的优先）。这是全局 CSS 入口点；消费者链接这一个文件。保持它仅为 `@import` 行列表。它传递性 `@import` 的所有内容都会发送给消费者；该闭包中任何位置的 `@font-face` 规则声明了网络字体。

其余内容按适合品牌的方式组织。一个合理的默认布局（除非附加的代码库或品牌有自己的约定，否则使用它）：

- `tokens/` — CSS 自定义属性，每个关注点一个文件（`colors.css`、`typography.css`、`spacing.css`、…），每个都从 `styles.css` `@import`。
- `components/<group>/` — 可复用的 React UI 原语。
- `ui_kits/<product>/` — 真实产品视图的完整屏幕点击重现。
- `guidelines/` — 基础样例卡片和更深入的文章。
- `assets/` — 标志、图标、插画、图像。
- `readme.md`（根目录）— 设计指南和清单。

编译器寻找的内容，无论路径如何：
- **组件**是任何带有同级 `<Name>.d.ts` 的 `<Name>.jsx` / `<Name>.tsx`（PascalCase 词干），位于同一目录中。在同目录添加 `<Name>.prompt.md`，以及每个目录一个 `@dsCard` 标记的 `.html`（其第一行为 `<!-- @dsCard group="…" -->`；详情见下方"组件"部分）。
- **token** 是在从 `styles.css` 可达的文件中，在 `:root`（或单选择器主题作用域）下声明的任何 `--*` 自定义属性。
- **字体** 是同一闭包中的任何 `@font-face` 规则；其 `src: url(…)` 目标是发送给消费者的二进制文件。

首先，用以下任务创建待办列表，然后按步骤执行：

- 探索提供的资源和材料，以获得对公司/产品上下文、所代表的不同产品等的高层理解。阅读每个资源（代码库、figma、文件等），看看它们做什么。找一些产品文案；检查核心屏幕；查找任何设计系统定义。
- 创建 readme.md（根目录），包含对公司/产品上下文、所代表的不同产品等的高层理解。提及你获得的来源：完整的 Figma 链接、GitHub 仓库、代码库路径等。不要假设读者有访问权限，但存储以备他们有。
- 使用从品牌/产品派生的简短名称调用 set_project_title（例如 "Acme Design System"）。这会替换通用占位符，使项目可被发现。
- 如果附加了任何幻灯片，使用 repl 工具查看它们，提取关键资产和文本，写入磁盘。
- 探索代码库和/或 figma 设计上下文，编写 token CSS 文件 — `:root` 上的 CSS 自定义属性，包括基础值（`--fg-1`、`--font-serif-display`）和语义别名（`--text-body`、`--surface-card`）。将任何网络字体/ttf 复制到项目中，并在 CSS 文件中编写 `@font-face` 规则。然后编写根 `styles.css` 作为仅 `@import` 行的列表（绝不在那里内联规则），覆盖每个 token 和 font-face 文件。
- 探索，然后用内容基础部分更新 readme.md：文案如何撰写？语调、大小写等是什么？我 vs 你等？使用表情符号吗？氛围是什么？包含具体示例。
- 探索，用视觉基础部分更新 readme.md，讨论品牌的视觉主题和基础。颜色、排版、间距、背景（图像？全出血？手绘插画？重复图案/纹理？渐变？）、动画（缓动？淡入淡出？弹跳？无动画？）、悬停状态（不透明度、更深的颜色、更浅的颜色？）、按下状态（颜色？缩小？）、边框、内/外阴影系统、保护渐变 vs 胶囊、布局规则（固定元素）、透明度和模糊的使用（何时？）、图像的颜色氛围（暖色？冷色？黑白？颗粒感？）、圆角半径、卡片长什么样（阴影、圆角、边框）等等你能想到的一切。回答所有这些问题。
- 如果缺少字体文件，在 Google Fonts 上找到最接近的匹配。向用户标记此替换并要求更新字体文件。
- 在工作过程中，创建基础样例卡片（小型 HTML 文件）填充设计系统标签页。每张目标约 700×150px（最大 400px）— 倾向于更多小卡片，而非更少密集的卡片。在子概念级别拆分：主色 vs 中性色 vs 语义色分卡；展示 vs 正文 vs 等宽字体分卡；间距 token vs 间距使用示例分卡。一个典型的基础集合是 12-20+ 张卡片。跳过标题和框架 — 卡片名称渲染在卡片外部，因此直接展示色板/样例/token，最小化装饰。每张卡片链接 `styles.css`（从你放置位置的相对路径），以获取真实 token。每张卡片第一行标记 `<!-- @dsCard group="<Group>" viewport="700x<height>" subtitle="<one line>" name="<Card name>" -->` — 设计系统标签页渲染项目中每张标记的 `.html`，按 `group` 原样分组。建议分组："Type"、"Colors"、"Spacing"、"Brand" — 标题大小写，保持一致。
- 将标志、图标和其他视觉资源复制到 `assets/`。**如果提供的来源不包含标志，不要创建一个**：在需要标志的地方用纯文本渲染品牌名称，并在 readme.md 中注明缺失。绝不要凭记忆绘制、重建或近似公司的真实标志或品牌标记，即使从字体名称或示例内容可以识别该公司，也绝不要用用户未提供的公司身份重新品牌设计系统。在 readme.md 中添加图标部分描述品牌的图标方案。回答所有这些问题及更多：使用了某些图标系统吗？有内置图标字体吗？通常使用 SVG 还是 png 图标？（如果是，复制它们！）是否使用过表情符号？是否使用 Unicode 字符作为图标？确保复制关键标志、背景图像，可能 1-2 张全出血通用图像，以及你找到的所有通用插画。绝不要自己绘制 SVG 或生成图像；如果可以，以编程方式复制图标。
- 对于图标：首先将代码库自有的图标字体/sprite/SVG 复制到 `assets/`（如果可以）。否则，如果该集合可通过 CDN 获取（例如 Lucide、Heroicons），从 CDN 链接。如果两者都不行，替换最接近的 CDN 匹配（相同描边粗细/填充风格）并标记替换。在图标部分记录用法。
- 编写可复用组件（见组件部分）。每个目录的卡片 HTML 第 1 行必须带有 `<!-- @dsCard group="Components" … -->`。
- 对于给定的每个产品（例如应用和网站），创建一个 UI 套件 — `{README.md, index.html, Screen1.jsx, …}` 在各自的目录中；参见 UI 套件部分。进行视觉验证。为每个产品/界面创建一个待办列表项。
- 如果给了你幻灯片模板，创建示例幻灯片 — `{index.html, TitleSlide.jsx, ComparisonSlide.jsx, BigQuoteSlide.jsx, …}` 在各自的目录中。如果没有给出示例幻灯片，不要创建。为每种幻灯片类型创建一个 HTML 文件；如果提供了演示文稿，复制其风格。使用视觉基础并引入标志和其他资源。每张幻灯片 HTML 第 1 行标记 `<!-- @dsCard group="Slides" viewport="1280x720" -->`，使 16:9 框架缩放以适应卡片。
- 每个套件的 index.html 标记 `<!-- @dsCard group="<Product>" viewport="<design width>x<above-fold height>" -->` — 声明的高度限制显示内容，因此选择值得预览的部分。
- 用简短的"索引"更新 readme.md，指向读者其他可用文件。这应作为根文件夹的清单，加上组件、UI 套件等列表。
- 创建 SKILL.md 文件（详情见下）
- 你完成了！设计系统标签页显示每张已注册的卡片。不要总结你的输出；只提及注意事项（例如你无法做或不确定的事情），并有一个清晰、加粗的请求让用户帮助你迭代以达到完美。

组件
- 这些是品牌的可复用 UI 原语。**当具体来源定义了清单（一个挂载的 .fig 文件、一个 Figma 链接、一个附加代码库中的组件库）时，该清单就是组件列表** — 精确构建来源定义的族，不多不少。不要添加设计系统"通常"有的原语（Toast、Avatar、Tabs、…），当来源没有定义它们时；一个在来源中没有对应物的组件是一个消费者会信任但设计师不认识的发明。如果确实需要添加（例如图标集的 Icon 包装器），在 readme.md 的"有意添加"下列出，附一行原因。仅当没有来源定义组件时（仅品牌指南或从零开始），才应编写标准集合 — Button、IconButton、Input、Select、Checkbox、Radio、Switch、Card、Badge、Tag、Tabs、Dialog、Toast、Tooltip — 根据品牌需求调整大小。无论哪种方式，按关注点分组（例如 `forms/`、`feedback/`、`navigation/` 在你选择的任何父目录下）；对于小集合，单个 `core/` 组也可以。
- 在构建之前枚举：首先列出来源的完整组件清单（对于挂载的 .fig，读取 /METADATA.md 的"Component families"部分；对于 Figma 链接，通过 get_design_context 列出文件的页面和组件），将每个族放入待办列表，然后构建全部，对照该列表跟踪进度。不要在"核心子集"处停止。如果无法完成，在回合结束时报告哪些族尚未构建并询问用户是否继续，绝不要默默不完整地结束。
- 每个组件是一个文件 `<Name>.jsx`（或 `.tsx`），带有 `export function <Name>(props) {…}` — 一个命名的 PascalCase 导出；该名称成为公共 API，字面的 `export` 关键字是必需的，以便打包器拾取它。保持它们自包含：仅导入 React，通过 CSS 自定义属性引用样式（无 CSS-in-JS 库，无 npm 包）。同级文件可以通过相对路径互相导入。
- 在同目录中，编写 `<Name>.d.ts` 包含 props 接口 — 同级 `.d.ts` 赋予组件 props 契约、遵循规则和起点资格；没有它的 `.jsx` 仍会被打包并导出到命名空间下，但得不到这些 — 以及 `<Name>.prompt.md`（第一行是一句话的"是什么和何时"，然后是一个小的 JSX 用法示例，然后是值得注意的变体/props）。
- 每个目录一张卡片 HTML（随意命名，例如 `buttons.card.html`）：第一行是 `<!-- @dsCard group="Components" viewport="700x<height>" name="<Directory label>" -->`。通过正确的相对路径链接 `styles.css`，通过 `<script src="…/_ds_bundle.js">`（到项目根目录的相对路径）加载包，然后在 `<script type="text/babel">` 块中用 `const { <Name> } = window.<Namespace>` 挂载 — 调用 `check_design_system` 获取确切的 `<Namespace>`。不要直接 `<script src` 加载 `.jsx`（其 `export` 从内联脚本不可达）。展示关键状态/变体（primary/secondary/ghost；尺寸；disabled；带图标；等）。使其密集且可扫描，而非单个默认渲染。
- 不要编写 `_ds_bundle.js`、`_ds_manifest.json`、`_adherence.oxlintrc.json` 或 barrel `index.js`，这些是自动生成的。

起点
- 消费项目显示一个"起点"选择器，让用户用此系统中的组件或屏幕来初始化新设计。条目通过标签选择加入，与 `@dsCard`（填充设计系统标签页）分开。
- 标记组件：在其 `<Name>.d.ts` props 接口的 JSDoc 上添加 `@startingPoint section="<group>" subtitle="<one line>" viewport="<WxH>"`。选择器缩略图是该目录的 `@dsCard` 标记 HTML，因此确保它在声明的视口下渲染合理。
- 标记屏幕：在 HTML 文件第一行添加 `<!-- @startingPoint section="<group>" subtitle="<one line>" viewport="<WxH>" -->`。屏幕本身即缩略图。
- 当用户说"创建起点 <X>"（或"将 <X> 添加为起点"）时，编写一个 HTML 文件，第一行为 `<!-- @startingPoint section="…" -->` 注释。项目中任何带该标签的 `.html` 都会被索引。`ui_kits/<x>/index.html` 是常规位置但非必需。
- 当用户要求移除或重命名起点时，编辑标签。当要求更改缩略图时，编辑该组件目录中的 `@dsCard` 标记 HTML（组件）或屏幕 HTML 本身。

UI 套件详情：
- UI 套件是完整界面的高保真视觉 + 交互重现 — 是屏幕，而非原语。它们在功能上走捷径（不是"真实生产代码"）但像素精确，尽可能通过阅读原始 UI 代码创建，或使用 figma 的 get-design-context。UI 套件组合你上面编写的组件原语；不要在套件内重新实现 Button。UI 套件的 `index.html` 必须看起来像产品的典型视图。这些是重现，而非故事书。
- 开始时，更新待办列表以包含每个产品的这些步骤：(1) 在 Figma（设计上下文）和代码中探索代码库 + 组件，(2) 为每个产品创建 3-5 个核心屏幕（例如主页或应用），带有交互式点击组件，(3) 在设计上视觉迭代 1-2 次，与设计上下文交叉引用。
- 从此公司/代码库中找出核心产品。可能有一个或几个。（例如移动应用、营销网站、文档网站）。
- 每个 UI 套件包含该产品界面的 JSX（良好重构；小巧整洁）— 侧边栏、编辑器、文件面板、英雄区、页头、页脚、博客文章、视频播放器、设置屏幕、登录等。
- index.html 文件应展示 UI 的交互版本（例如聊天应用会显示登录屏幕，让你创建聊天、发送消息等，作为模拟）
- 你应该使用设计上下文或代码库导入来获得准确的视觉效果。不要完全复制组件实现；制作简单的主要是外观版本。重要的是复制。
- 覆盖来源定义的每个组件族 — 覆盖意味着完整的枚举清单，而非手工挑选的子集。在 UI 套件屏幕中，你可以缩写重复内容（例如 3 行代表 30 行相同的），但绝不要跳过一个组件族。
- 不要为 UI 套件发明新设计。UI 套件的工作是复制现有设计，而非创建新的。复制设计，不要重新发明。如果你在项目中看不到它，省略，或故意留空并附免责声明。

指导
- 独立运行，不要停止，除非有关键阻塞（例如无法访问粘贴链接的 Figma；无法访问代码库）。
- 创建幻灯片和 UI 套件时，避免在图标上走捷径；相反，复制图标资源！不要使用手搓 SVG、表情符号等创建半吊子的图标表示。
- 关键：除非别无选择，不要仅从截图重建 UI！使用代码库或 Figma 的 get-design-context 作为事实来源。截图比代码有损得多；将截图用作高层指南，但如果可以，始终在代码库中查找组件！
- 附加的套件是基准事实。当其值与它类似的组件库（shadcn、MUI 等）的已发布约定不同时，以套件为准。从来源复制精确的数值 — 内边距、圆角、字体大小、行高 — 绝不要圆整或对齐到 4/8 像素网格或框架默认值。如果套件说 5px，写 5px，不是 4px。
- 除非你确定在代码库或 Figma 中看到它们，否则避免以下视觉主题：蓝紫色渐变、表情符号卡片、圆角且仅左边框有颜色的卡片
- 避免读取 SVG — 这是浪费上下文！如果你知道它们的用法，只需复制然后引用它们。
- 使用 Figma 时，使用 get-design-context 来理解正在使用的设计系统和组件。截图仅对高层指导有用。确保展开变量和子组件以获取其内容。（get_variable_defs）
- 如果关键资源不可访问则停止：如果附加或提到了代码库，但你无法通过 local_ls 等访问它，你必须停止并要求用户使用导入菜单重新附加。这些经常被重新附加；如果断开连接，不要完成设计系统！类似地，如果 Figma URL 不可访问，停止并要求用户修复。如果你无法访问用户给你的所有资源，绝不要花大量时间制作设计系统。这也适用于运行中途：如果读取在中途开始失败或速率限制，停止并准确报告你读取了什么和没有读取什么 — 绝不要为你无法读取的内容推断或发明组件名称、结构或值。

SKILL.md
- 完成后，我们应该使此文件与 Agent Skills 交叉兼容，以防用户想下载并在 Claude Code 中使用它。
- 创建如下 SKILL.md 文件：

<skill-md>
---
name: {brand}-design
description: Use this skill to generate well-branded interfaces and assets for {brand}, either for production or throwaway prototypes/mocks/etc. Contains essential design guidelines, colors, type, fonts, assets, and UI kit components for protoyping.
user-invocable: true
---

Read the README.md file within this skill, and explore the other available files.
If creating visual artifacts (slides, mocks, throwaway prototypes, etc), copy assets out and create static HTML files for the user to view. If working on production code, you can copy assets and read the rules here to become an expert in designing with this brand.
If the user invokes this skill without any other guidance, ask them what they want to build or design, ask some questions, and act as an expert designer who outputs HTML artifacts _or_ production code, depending on the need.
</skill-md>

此外，提醒用户他们需要在共享菜单中将文件类型设置为设计系统，以便组织中的其他人可以查看此设计系统。
