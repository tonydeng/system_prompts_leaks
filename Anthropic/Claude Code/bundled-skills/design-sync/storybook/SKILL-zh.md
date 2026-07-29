# Storybook 源形态

Storybook 是**保真度预言机，而非运行时**。转换器将包编译后的 `dist/` 打包为 `_ds_bundle.js`——与 claude.ai/design 智能体构建时使用的相同打包产物——并通过**编译 story 源模块本身**（hooks、fixtures、本地辅助函数——整个闭包都包含在内）来生成每个预览，每个组件导入都被解析到该打包产物（`lib/story-imports.mjs` 将包导入*和*相对组件导入重定向到 `window.<Global>`）。仓库自身的 storybook 渲染是这些预览必须匹配的基准真值：对比工具将每个 story 在参考 storybook 和对应预览渲染中并排截图，你不断迭代直到它们匹配。storybook-static 中的任何内容都不会被上传，story 代码也绝不会在构建时被求值——story 仅在浏览器中运行，针对真实的构建产物。


需要 React 18+。Playwright + chromium 是该形态的**必需依赖**（对比循环即验证环节），不是可选的。

**首次同步还是重新同步？** 重新同步的标志是配置中的 `projectId` 和 `pkg` 在本次运行开始前都已存在——此时本文档大部分内容不适用；跳转至 §7，一次驱动器运行即可路由工作，未触及的组件零成本。其他情况均需走完整流程（§2 构建 → §3 自愈 → §4 匹配 → 约定头（上传前的基础 SKILL.md）→ §6 上传），其中每个组件都被验证和评级一次——包括中止运行留下的部分配置，以及本次运行自身在基础 skill §1 中刚记录的 pin。（只有旧的 `design-sync.config.json` 存在？先移动它并提交：`mkdir -p .design-sync && mv -n design-sync.config.json .design-sync/config.json`，然后应用相同的测试。）

## 2. 构建，然后运行转换器

1. **构建 DS 包*及其工作区依赖项*。** 转换器将 `dist/` 打包到 `window.<Global>`。运行 `<pm> run build`；在 monorepo 中使用 `turbo run build --filter=<pkg>` 或 `pnpm -F "<pkg>..." build`（末尾的 `...` 是必需的——裸 `-F <pkg>` 会跳过依赖，你会看到 `Cannot find module '@scope/tokens'`）。如果 `package.json` 的 `module`/`exports['.']` 指向 TS 源码，找到实际构建入口并通过 `--entry` 传入。**在步骤 2 之前执行此操作**——storybook 经常从同级包的构建 `dist/` 导入。
2. **将参考 storybook 构建一次到 `.design-sync/sb-reference/`** ——不在 `ds-bundle/` 下（转换器在每次重建时清除 `--out`，而 storybook 构建需要数分钟；参考必须在修复循环中存活）：

   ```bash
   npx storybook build -c <storybookConfigDir> -o .design-sync/sb-reference
   ```

   从其 `package.json` 包含 storybook devDependencies 的目录运行——通常是包含 `.storybook/` 的目录；monorepo 通常有多个 storybook，选择覆盖你要同步的包的那个。**将 `-o` 设为仓库根路径**（例如 `-o "$(git rev-parse --show-toplevel)/.design-sync/sb-reference"`）：转换器和对比工具从仓库根解析 `.design-sync/`，所以在子包中用 cwd 相对路径的 `-o` 会把参考放到无人能找到的地方。直接使用 `npx storybook build`，**不要**用仓库的 `npm run build-storybook` 脚本（输出目录错误）。然后检查 `.design-sync/sb-reference/iframe.html` 是否存在且 >10KB——仅 `index.json` 存在可能意味着构建失败。

   长时间构建：**仅通过 shell 工具的后台模式**在后台运行，并等待完成通知。绝不要用裸 `&`（无法追踪——通知永远不会到来），也绝不要用 `pgrep -f '<script>'` 轮询循环（它匹配自己的命令行并一直旋转到超时）。Headless / `-p` 会话：改为同步运行长命令——那里没有任务通知重新调用，所以后台运行永远不会被恢复。

   `.gitignore` 追加项：`.design-sync/sb-reference/`、`.design-sync/learnings/`、`.design-sync/.cache/`、`.design-sync/node_modules`（fork 符号链接——每个 clone 重新创建）、`.ds-sync/`、`ds-bundle/`——构建产物、临时暂存、验证工作状态、符号链接、暂存脚本、重新生成的输出。提交的内容：持久集（非 storybook §2 中的规则，此处相同：`.design-sync/` 下未被 gitignore 的所有内容——previews/ 仅存放你编写的文件；生成的 story-module 包装器位于 `.design-sync/.cache/previews/` 并在每次构建时重新生成；转换器从不在 `previews/` 中写入或删除任何内容）。验证状态从不提交——跨机器的继承来自上传项目的 `_ds_sync.json`。仅在 story 或 DS 源变更时重建参考。
3. **编写 `.design-sync/config.json`** ——只需 `pkg` 和 `globalName`。**如果已存在，先读取它并保留已有内容**——`titleMap`、`overrides` 和 `provider` 累积了之前同步的修复。同时先读取 `.design-sync/NOTES.md`——其**重新同步风险**部分是上次运行的关注清单；重新验证这些项目，而不是假设继承覆盖了它们。`../non-storybook/SKILL.md` §2.6 中的包形态字段表完全适用；此处最相关的字段：

   | 字段 | 值 |
   |---|---|
   | `pkg` / `globalName` | `pkg` 必需；`globalName` 省略时从 `pkg` 自动推导 |
   | `shape` | `"storybook"`——锁定检测 |
   | `storybookStatic` | `".design-sync/sb-reference"`——使重新同步和对比无需标志即可找到参考 |
   | `storybookConfigDir` | `.storybook/` 目录（monorepo 场景） |
   | `buildCmd` | 重新同步时在转换器之前重新运行的命令 |
   | `titleMap` | 当 story 标题不匹配导出名时为 `{title: ExportName}`；`{title: null}` 将非视觉/内部组件从同步中完全排除 |
   | `overrides` | `{<Name>: {skip: [storyIds], cardMode: "single"\|"column", primaryStory: "<Export>", viewport: "WxH"}}`——`skip` 用于无法静态渲染的 story；`cardMode: "single"` 用于覆盖层组件（§4a.5、§5），`"column"` 用于比网格单元更宽的 story（§3 中的 `[GRID_OVERFLOW]` 行） |
   | `provider` | 对于**预览**通常不需要——`.storybook/preview` 装饰器会自动打包；仅在其失败时设置。在 §6 上传前，将装饰器提供的上下文提炼到 `cfg.provider`——README/prompt.md 的包装指导仅从配置生成（仅装饰器的包装会生成通用说明）。**设置它还会在下次构建时替换装饰器作为预览包装器**：切换后对主题组件进行 scoped-compare——不完整的提炼会退化装饰器原本渲染良好的预览，而继承的评级无法捕获这一点。格式：`{"component": "ThemeProvider", "props": {…}, "inner": {…}}`——嵌套链，最外层在前；每个 `component` 必须是打包导出。字面量 `props` 用于小标量（`"theme": "light"`）和稳定片段。对于仓库中已存在的数据——locale JSON、主题对象——**优先使用 `{"$ref": "<export>"}`**，由通过 `cfg.extraEntries` 添加的 2 行模块支持（例如 `export { default as previewI18n } from '../locales/en.json'`）：`$ref` 发出 `window.<Global>.<export>`，所以数据在打包中只存一次，并在每次构建时从源文件重新读取。对于小且稳定的内容内联副本是可以接受的，但要知道代价——字面量会复制到每个卡片的 html 中，并在源文件变更时悄无声息地腐化，所以任何较大或不断演变的内容都应放在 `$ref` 后面。`extraEntries` 的路径形式：裸名称从 `node_modules` 解析；仓库自有模块需要显式的 `./`/`../` 包相对路径（工作区边界——构建日志会记录 `! extraEntries: … skipped` 如果它逃逸了）。 |

4. **暂存脚本 + 安装转换器依赖**（隔离在 `.ds-sync/` 中，仓库 lockfile 不受影响）：

   ```bash
   mkdir -p .ds-sync && cp -r "<skill-base-dir>"/package-build.mjs "<skill-base-dir>"/package-validate.mjs "<skill-base-dir>"/resync.mjs "<skill-base-dir>"/lib "<skill-base-dir>"/storybook "<skill-base-dir>"/non-storybook .ds-sync/
   echo '{"name":"ds-sync-deps","private":true}' > .ds-sync/package.json
   (cd .ds-sync && npm i esbuild ts-morph @types/react playwright && npx playwright install chromium)
   ```

   如果 chromium 安装失败，先运行 `npx playwright install-deps chromium`；如果环境无法安装 chromium，设置 `DS_CHROMIUM_PATH=<system-chromium>`。
5. **运行转换器、验证器和对比工具**——同步执行，在第一个非零退出码时停止（对比仅在构建+验证通过后运行——§3）。大型 DS（约100+ 组件）可能需要 `NODE_OPTIONS=--max-old-space-size=<MB>` 来构建；**绝不通过 `head`/`tail` 管道构建**（管道掩盖退出码——OOM 看起来像成功）；重定向到文件再读取：

   ```bash
   node .ds-sync/package-build.mjs --config .design-sync/config.json --node-modules <pkg-node-modules> \
     --entry <built-dist-entry> --out ./ds-bundle
   node .ds-sync/package-validate.mjs ./ds-bundle
   node .ds-sync/storybook/compare.mjs --out ./ds-bundle --storybook-static .design-sync/sb-reference \
       --components <solo-phase picks>   # 将第一次对比范围限定在 §4b 的 solo 组件
   ```

   在 monorepo 中，`--node-modules` 是 DS 包自身的 `node_modules`——除非 hoisting 使其稀疏（yarn 的 `node-modules` linker 只在仓库根保留 `react`）：如果其中缺少 `react/` 或 `react-dom/`，则传入仓库根 `node_modules`。在 DS 自有源码仓库中 `node_modules/<pkg>` 不存在，因此需要 `--entry`。构建日志记录 `[ICON_PKG]` / `[TOKENS_PKG]` 自动检测，并将 `.storybook/preview` 装饰器打包为预览包装器（`preview-decorators.js`），使预览获得与 story 相同的 provider 链。

   限定第一次对比运行的范围：大型 DS 的完整捕获是数千次 chromium 导航——在 solo 阶段清除全局问题之前毫无意义（每个全局修复使每次捕获失效）。第一次全名单运行在 §4b 步骤 3 中进行——而对于超过 20 个有 story 的组件的 DS，即使那次也会按规模分批进入 §4c 的范围批次，所以唯一必须的全名单运行是 §4d 收据，它携带已评级的工作前进而不是重新捕获它们。对于有 >100 个有 story 的组件的 DS，在分派之前还要告诉用户预期规模（组件 × story），并让他们在需要时缩小范围。

## 3. 自愈循环（构建 + 验证）

修复 `[TAG]` 错误 → 重建 → 重新验证，直到两者都退出 0，**在**开始 §4 的对比循环**之前**——在打包本身损坏时像素匹配预览没有意义。共享转换器标签（`[NO_DIST]`、`[WORKSPACE_SIBLING]`、`[CSS_*]`、`[FONT_*]`、`[TOKENS_MISSING]`、`[DTS_*]`、`[RENDER*]`、…）行为与包形态完全相同——使用 `../non-storybook/SKILL.md` §3 中的表格。错误下打印为 `hypothesis:` 的行是线索，不是指令：先运行其验证步骤，如果不确认，放弃假设并从错误文本本身诊断。Storybook 特有项：

| 标签 | 症状 | 修复 |
|---|---|---|
| `[SB_REFERENCE_MISSING]` | 对比找不到 `iframe.html` | 构建参考（§2.2）；设置 `cfg.storybookStatic`。 |
| `[SB_BUILD_FAIL]` | 转换器自身的 storybook 构建失败 | 你跳过了 §2.2——自己构建参考并设置 `cfg.storybookStatic`，这样转换器就不需要自己构建。 |
| `[ZERO_MATCH]`（storybook 变体） | 没有 story 条目匹配 | 检查 storybook 配置的 `stories` glob；然后检查 `titleMap`。 |
| `[TITLE_UNMAPPED]` | N 个标题不匹配导出 | `cfg.titleMap {<title-name>: <export-name>}`。 |
| `(preview: <Name> — no story exports paired …)` | 索引 story 名称无法匹配到模块导出键（配对尝试显示名称，然后是 story ID 的尾部） | 组件显示底板卡片；修复配对——通常是自有的 `.tsx` 以可匹配的名称重新导出 story。 |
| 某个预览单元报 `undefined`-component / wrong-context 消息 | 某个 story 导入解析方式错误——相对路径、tsconfig-alias 和裸工作区导入都通过相同策略（见 `lib/story-imports.mjs` 的规则） | `cfg.storyImports.shim` / `cfg.storyImports.bundle` 子串模式按解析路径强制解析——在 fork 接口之前的廉价修复。 |
| `! preview build failed: <Name>` | story 模块未能编译（顶层 await、esbuild 无法解析的包导入、无 loader 的资源扩展名） | 阅读该行上方的 esbuild 错误。未知资源扩展名 → `cfg.storyImports.loaders`（与默认值合并，例如 `{".yaml": "text"}`）；无法解析的导入 → 接管 `.tsx` 并删除该导入。组件显示底板卡片直到修复。 |
| 某个 story 自身的样式表在其单元中缺失 | story 局部 `.css`/`.scss` 副作用导入编译为空（组件样式通过打包 css 交付）。例外：`.module.css` 会被编译——类名解析且 `_preview/<Name>.css` 自动链接 | 通常无需操作——这些样式是 storybook 页面添加的装饰。如果 story 确实依赖它们，在自有的 `.tsx` 中内联样式。 |
| `[BUNDLE_EXPORT]` | 组件不是 `window.<Global>` 上的函数 | 为子路径/图标导出添加 `extraEntries`；检查 dist 入口是否为完整构建。 |
| `[SCHEDULER_MISSING]` | dist 导入了 `scheduler` | react-dom 泄漏到 DS dist 中——检查其构建的 externals。 |
| `! preview decorator bundle failed` | 装饰器无法打包 | 手动设置 `cfg.provider`，或运行 `node .ds-sync/storybook/probe.mjs --storybook-static .design-sync/sb-reference` 从实时 storybook 推断链（用真实值替换每个 `$hint`）。 |
| 预览在加载 `_vendor/preview-decorators.js` 时报错（storybook-API `undefined` 错误） | `.storybook/preview` 导入图触及了 stub 不覆盖的 storybook-runtime 模块 | `manager-api`/`preview-api` 被功能性空操作 hooks stub，其他每个 `@storybook/*`/`msw` 模块被惰性可调用对象 stub（`fn()`、`action()`、`setupWorker()` 在模块作用域都无害求值）；如果某个其他 API 仍然崩溃，显式设置 `cfg.provider`——它完全跳过装饰器打包。 |
| 对比输出 `[ASSETS_BLOCKED]` | 捕获浏览器继承了网络沙箱 shell——story 资源（CDN 图片/字体）在**两个**面板上失败，所以评级可能虚假通过而最终用户看到不同输出 | 从有出口权限的 shell 重新运行 `package-validate.mjs` + `compare.mjs --force`：在提示时批准在没有沙箱的情况下运行命令，或将主机添加到沙箱白名单。在此输出期间不要对含图片的组件评级。 |

**增量路径（基础 SKILL.md §3）——这是开通通道的门控。** 在构建+验证首次都退出 0 时，在开始 §4 之前开通上传通道：用户在此批准一次，然后观看组件随评级进展而落地。在第一个评级批次上传之前不会上传任何内容——共享基础文件随其一起——批次推送来自 §4b/§4c。（原子路径：在 §6 之前不上传任何内容。）

## 4. 将预览匹配到 storybook

`compare.mjs` 是一个**捕获工具——它拍照，你评级。** 它不计算任何相似度启发式（像素/文本/字体分数在框架合理差异时会误导）；判断从两张真实截图中做出。编译的预览按**每个 story** 捕获——每个 story 通过 `?story=<Export>` 在完整捕获视口中单独渲染，与 storybook 框架参考侧的方式完全相同——因此同级 story 不会互相干扰（portal 堆叠、共享 radio-group 名称、焦点、容器测量）。两个输出层级：
- **临时层**（在 `ds-bundle/` 下，被重建清除）：`_screenshots/compare/<group>__<Name>.png`——每个 story 一行的表格：**真实 storybook 渲染 | 真实预览渲染**，并排。表格图像被缩小以适应；全分辨率原始图在 `…/compare/raw/`（`…__sb.png` / `…__ds.png`）——当表格太小无法自信判断时读取这些。
- **活动状态**（在 `.design-sync/.cache/compare/` 中，被 gitignore）：`<Name>.grade.json`——你的判定——以及 `<Name>.json`——捕获事实：story↔单元配对、截图路径、`previewKind`、组件的 `srcSha`（story 文件指纹）、抽查锚点。可重建——缺失只意味着"重新捕获"。脚本发出的唯一判定是事实性的：`sb-error`（story 在 storybook 中不渲染）、`unpaired`（该 story 没有预览单元）、`error`（单元抛出异常）；每个已渲染的对都是 `needs-grade`。

对比默认每个组件最多捕获 6 个 story——日志中的 `[STORY_CAP]` 标记超过此数的组件，`--max-stories <n>` 可提高上限。该上限不是评级契约的一部分：提高它只是捕获尾部 story 以进行增量评级，现有判定保持不变。需要注意的一个后果：一个被上限限制的组件如果全部评级为 `match`/`close`，在未来的同步中即使其尾部 story 从未被单独评级，也会通过上传完全验证——当那些尾部 story 携带值得验证的不同变体时提高上限。分派子智能体不得在波次中间更改它（表格覆盖的 story 集合会与编排者工作列表假设的不同）。

**跨运行状态**——第一次运行验证所有内容一次；之后，一条规则：**评级跟随你的源**——story 文件、你拥有的预览、story 集合、影响预览的配置（`provider`/`storyImports`/`extraEntries`/`overrides`/`titleMap`）以及提交的 `.design-sync/overrides/` fork。流水线变更（skill 或工具链更新重新渲染一切）由采样的 `[SPOT_CHECK]` 自动验证并保留评级；你的编辑只重新评级它们触及的内容。像素抖动永远不会搅动评级。
- *源未变* + 完全评级的 `match`/`close` → **直接跳过**（`carried forward`）：不捕获，不重新评级——即使打包、样式、storybook 或转换器本身被重建。`--force` 重新捕获一切**并清除所有评级**——系统性重新验证，不是随意的表格重新生成。
- *源已变*（编辑了 story、编辑了 `.tsx`、编辑了配置/fork）→ 重新捕获，评级清除，从新表格重新评级。`[STORY_CHANGED]` 标记代码移动的 story——那些是拥有的 `.tsx` **必须更新**的（生成的预览自动重新派生）；没有 `[STORY_CHANGED]` 的重新捕获通常只需要重新评级。
- *`[SPOT_CHECK]`* → 重新捕获指定组件**不清除其评级**；读取新表格并确认它们仍然匹配记录的评级。它可以在流水线变更后由驱动器触发——skill/工具链更新的正常验证，不是 bug。差异修复随变更集规模缩放：几个组件 → 只重新评级那些；大范围 → 停止，诊断，然后 `--force` 全量通过。`--spot-check N` 调整全量运行的随机采样（0 禁用）；`--spot-check-components A,B` 显式指定选取，在范围运行中也受支持（§7 步骤 4 的审计）。
- *`[REFERENCE_STALE?]`* → 打包已变但参考 storybook 未变。如果 DS 源已变，在评级前重建 `.design-sync/sb-reference`——过时的参考使每个评级都是与*旧*设计的比较。
- *某个 story 每次捕获渲染结果不同*（`new Date()`/`Math.random()` 内容）→ 指纹是 story 文件，所以契约稳定——但像素不稳定，而评级判断的是像素。冻结的捕获时钟稳定了日期渲染；对于真正随机的内容，在拥有的 `.tsx` 中固定值或 `cfg.overrides.<Name>.skip` 该 story 并在 NOTES.md 中添加一行。

捕获经过稳定化处理以用于评级可比性（动画快进、减少动画、冻结时钟——两个面板显示相同的稳定帧、相同的渲染日期）。这是仅用于验证：发布的预览不受影响且完全动画化。

**评级由处理组件的人完成**——solo 阶段是你，扇出阶段每个子智能体负责自己的组件。每次对比运行后：读取表格（有疑问时读取原始 PNG），**仅从图像**判断每个 story，将判定写入 `.design-sync/.cache/compare/<Name>.grade.json`（活动本地工作状态——使判定持久的是上传：上传的 `_ds_sync.json` 在每次未来同步中锚定通过上传验证的跳过，任何机器上）：

```json
{"stories": {"Default": {"verdict": "match"}, "Compact": {"verdict": "match", "basis": "sibling-trusted"}}}
{"stories": {"Loading": {"verdict": "mismatch", "note": "spinner missing — story uses MSW mock"}}}
```

（两个组件的文件：一个干净的按下方采样规则评级——`Default` 是图像判断的主 story，在无警告组件上为 `match`，这正是许可同级信任条目的依据——以及一个不匹配的，其备注驱动下一个修复。）

评级标准——以设计师关注的角度评级，观察两张渲染图：
- `match`——相同的内容、构图和样式。忽略抗锯齿模糊、滚动条碎片、5px 以下偏移和框架差异（storybook canvas 和预览页面框架不同——判断组件，而非其周围环境）。
- `close`——可辨识的相同渲染但有细微差异（略微不同的 padding、焦点环、占位文本）。**`close` 仍然是修复目标，不是退出条件：** 如果你能说出差异名称，通常也能说出旋钮名称——继续迭代。仅在一次迭代未能改善或没有可操作的原因时才接受 `close`，且备注必须同时说明*什么不对*和*你尝试了什么/为什么不可修复*（例如"焦点环颜色不同——storybook 应用了全局焦点插件，不属于 DS"）。
- `mismatch`——错误/缺失内容、无样式输出、错误变体、缺失图标/图片、默认字体。备注必须说明*什么*不同——它驱动下一个修复。

当参考侧是构建产物时——storybook 通过 UI 外壳（主题/控制切换消息）限制 story，而预览渲染真实组件——自行判断组件渲染并注明限制；渲染*多于*受限参考的预览不是 `close`。

**评级主 story，信任其余。** 一个组件的同级 story 经过相同流水线——相同的导入、相同的 provider 链、相同的 CSS——所以当其中一个忠实渲染时，其余几乎总是如此。在首次同步中，仅从图像判断组件的**主 story**（设置 `cfg.overrides.<Name>.primaryStory` 时——单模式卡片渲染的相同 story——否则是表格的第一个 story）。如果评级为 `match` 且组件干净——没有 `sb-error`/`unpaired`/`error` 单元、没有 `[PORTAL?]`、没有 `[RENDER_BLANK]`、没有空白或尺寸异常的截图——为其余 story 写入 `match` 并附基础标记，`{"verdict": "match", "basis": "sibling-trusted"}`，使记录说明每个判定是如何得出的（对比只读取 `verdict` 字符串）。一个组件的所有判定——图像判断的主 story 加上每个同级信任条目——写入其唯一的 `grade.json`：信任的同级不需要打开图像和逐 story 遍历。当组件有 portal/覆盖层、主题或 provider 敏感性、拥有的预览或任何警告时，逐 story 穷尽评级——并且始终对 §4b solo 集合进行穷尽评级，其穷尽评级正是赢得信任的基础。

捕获无论如何都会为每个 story 拍照——采样节省的是评级注意力，而非捕获时间，表格保留供日后有意的查看（§7 步骤 4 的携带评级审计使用相同的保留评级抽查路径）。这与 `[STORY_CAP]` 未评级尾部 story 相同的信任类别，被有意应用。采样从不放宽 `[FONT_MISSING]`（§4a）——该检查对对比图像无论如何都不可见。

### 4a. 修复决策树——全局优先

自上而下工作；全局修复一次修复所有组件，单组件修复只修复一个：

1. **大多数/所有组件以相同方式出错** → 全局，在配置中修复 + 全量重建：
   - 单元中的上下文/provider 错误（`use<X> must be inside <Provider>`）→ 装饰器未打包（§3 `! preview decorator bundle failed` 行）→ `cfg.provider`。
   - 所有内容无样式/默认字体 → `cfg.cssEntry`（检查构建日志中的 `[CSS_FROM_STORYBOOK]`）、`cfg.tokensPkg`、`cfg.extraFonts`。
   - **`[FONT_MISSING]`——对比循环看不到这个。** 当两侧都不提供字体时，两个面板渲染相同的 chromium 回退，所以表格看起来"匹配"而每个 claude.ai/design 用户得到错误字体——绝不要接受"两侧相同回退"作为通过。按 `../non-storybook/SKILL.md` §3 中的 `[FONT_MISSING]` 行解决；storybook 特有补充：`cfg.extraFonts` 路径以包含 `dirname(--node-modules)` 的 git 仓库为边界——monorepo 中的同级排版包按原样工作；只有在没有 `.git` 祖先时边界才缩小到 `dirname(--node-modules)`，如果你添加了参考缺少的字体，将相同的 `@font-face` 注入 `.design-sync/sb-reference/iframe.html` 使预言机在两侧用真实字体验证。
   - 所有地方图标缺失 → `cfg.extraEntries`（检查 `[ICON_PKG]`）。
2. **一个组件，`unpaired` 或 `fallback preview`** → 其 `.tsx` 缺少该 story 的单元。预览编译整个 story 模块（hooks、fixtures、本地辅助函数全部包含——闭包不是失败模式），所以原因是：配对失败（`storyName` override）、包装器构建失败（构建日志中的 `! preview build failed`）、或模块在加载时抛出异常——检查表格的 `(page)` 错误行获取真实异常（模块作用域调用了 stub 不覆盖的包）。打开包装器（生成的：`.design-sync/.cache/previews/<Name>.tsx`；拥有的：`.design-sync/previews/<Name>.tsx`），添加/重命名导出或删除有问题的导入——如果是生成的那个，将修复保存为 `.design-sync/previews/<Name>.tsx`，不带首行标记（原地缓存编辑在本机保留但被 gitignore——在全新 clone 时消失，且重新编译时从不重新评级；只有拥有的副本移动评级契约，重建会警告已编辑的缓存孪生）。Story 导入使用位置无关的 `@ds-stories/<repo-relative path>` 形式，所以文件从任一位置都能不变工作。
3. **一个组件，你评级为 `mismatch`** → 错误的 props/构图。读取 story 源码；在拥有的 `.design-sync/previews/<Name>.tsx` 中镜像它（将缓存包装器复制到那里并删除其标记行）。这是编译 story 预览的唯一杠杆。
4. **`sb-error`** → story 在 storybook 中也不渲染（数据获取、交互驱动）。将其 id 添加到 `cfg.overrides.<Name>.skip` 并在 NOTES.md 中注明原因。
5. **`[PORTAL?]` / 覆盖层组件**（Dialog/Tooltip/Toast）→ 评级已经隔离（按 story 捕获），但产品卡片渲染整个网格 html，所以打开覆盖层的 story 也会在那里覆盖同级单元。设置 `cfg.overrides.<Name>.cardMode: "single"`——卡片渲染一个 story（`primaryStory` 选取它；否则为第一个导出）在全出血包装器中包含 `position:fixed` 后代，并在卡片上声明评级视口使产品以你验证的尺寸渲染。对于仅仅太宽的 story（数据表格、全宽栏——验证标记为 `[GRID_OVERFLOW] … wide`），改用 `cardMode: "column"`：每个 story 保持完整卡片宽度，不丢弃任何内容。定向重建该组件（`preview-rebuild.mjs --components <Name>`，秒级）——**评级携带**（`cardMode`/`primaryStory` 不在评级键或标记的配置切片中）；只有 `viewport` 变更需要重新评级（它是捕获视口）并需要完整构建（它移动切片）。

**重建规则——仅重建变更能触及的内容。** 样式变更（css/fonts/tokens）重新渲染每个预览但不移动任何评级契约——评级携带。Provider、`storyImports`、`extraEntries` 和 fork 编辑是评级契约的一部分（它们改变预览挂载的内容）——受影响的评级在重建时清除并重新评级。

| 你更改了 | 重建 | 对比 |
|---|---|---|
| 仅某个预览 `.tsx` | 下方定向循环（秒级） | 范围 `--components <Name>`——其评级清除，重新评级 |
| `overrides`（`skip`/`viewport`）/ `titleMap` | 完整 `package-build.mjs` + `package-validate.mjs`（重新标记定向重建检查的配置键） | 完整 `compare.mjs`——触及的组件重新评级；携带的 `match`/`close` 组件直接跳过，仍待处理的集合获得新表格（完整构建清除了它们——下一波读取这些表格） |
| `overrides`（仅 `cardMode`/`primaryStory`） | **定向循环**（`preview-rebuild.mjs --components <Name>`，秒级）——展示键不在标记的配置切片中，所以 `[CONFIG_STALE]` 不触发；循环重新发出卡片 html 并修补其 renderHash | **不重新评级**：仅展示键不在评级契约中——评级携带；更改的卡片 html 重新发布，重新同步可能抽查它 |
| `provider` / `storyImports` / `.design-sync/overrides/` fork | 完整构建 + 验证 | 完整 `compare.mjs`——受影响的评级按上述规则重新评级 |
| css / fonts / tokens | `package-build.mjs --skip-dts` + 验证 | 完整 `compare.mjs`——廉价：携带的 `match`/`close` 组件直接跳过，所以只有待处理集合针对新样式重新捕获。评级携带——零重新评级，不是零触及：更改的字节仍然重新发布，重新同步可能将它们作为 `verification.canary` 抽查浮现 |
| `entry` / `extraEntries` | 完整构建 + 验证——绝不 `--skip-dts`（它们改变打包和导出面） | 完整 `compare.mjs`——受影响的评级重新评级 |

活动中期——§4c 波次仍待处理——将此表的"完整 `compare.mjs`"理解为*最终通过批次进行*：重建无论如何清除受影响的评级，下一波的范围运行重新捕获这些组件，§4d 收据是全名单结算（§4c 波次间步骤 2）。仅在没有剩余波次时才支付即时的全名单对比。

`--skip-dts` 跳过逐组件类型提取——大型 DS 构建的缓慢部分——并发出 stub `.d.ts` 内容，所以其验证按设计失败 `[DTS_STUBBED]`（渲染检查仍然回答"修复是否有效？"）；§4d/§6 门控的验证退出 0 要求强制最终构建不使用它。预期 stub 构建的底板卡片和 README 摘要看起来简陋——最终构建恢复它们。`--skip-dts` 仅用于修复循环迭代：上传读取的任何构建——增量批次推送（基础 SKILL.md §3）和 §6 收尾一样——必须是真正的构建，所以如果 `.ds-build-meta.json` 仍然携带 `dtsStubbed`，在推送前不带该标志重建（批次推送上传磁盘上的 `.d.ts`）。

**将配置编辑批量到一个周期中。** 在支付重建之前，扫描所有待处理表格判定和已知问题，收集它们隐含的所有配置编辑（`skip`、`titleMap` 条目、`cardMode`）并一起应用——几分钟内发现的两个编辑不应花费两个重建+验证+对比周期。

**对比运行中途崩溃**（浏览器崩溃、OOM）：它捕获的表格是有效的——先评级它们，然后重新运行；携带将重新捕获范围限定到缺口。绝不要用 `--force` 重启崩溃的运行（它清除你刚获得的评级）。

**在大型 DS 上，在支付完整重建之前验证修复是否正确**：先在一个受影响的组件上运行下方定向循环（或探测其渲染页面）——通过完整重建验证的错误猜测花费整个周期。**中间验证可以采样**：全局损坏本质上是系统性的，所以 `--render-sample 10` 以一小部分成本回答"修复是否有效？"；当任何影响渲染的内容移动时，§4d/§6 上传门控要求完整渲染检查——在锚定重新同步中，§7 驱动器自动应用该规则（层级规则在那里）。

仅 `.tsx` 的定向循环：
  ```bash
  node .ds-sync/lib/preview-rebuild.mjs --config .design-sync/config.json --node-modules <nm> --out ./ds-bundle --components <Name>
  node .ds-sync/storybook/compare.mjs --out ./ds-bundle --storybook-static .design-sync/sb-reference --components <Name>
  ```

  定向循环重新编译预览但不从源重新键定评级契约：story 文件编辑后仅运行此循环会携带旧评级直到下次完整构建或驱动器运行重新键定它——通过完整构建路由 story 编辑（驱动器自动执行）。

### 4b. Solo 阶段——一个，然后几个

不要立即分派。全局问题必须先刷新到配置中，否则每个子智能体重新发现它们。

1. **一个组件。** 选择一个简单的、story 丰富的（类 Button：多个 story，无 portal）。运行 §4a 循环直到你从图像中将每个 story 评级为 `match`——仅当迭代停止改善时才接受 `close`（上方标准）。**每个修复成为 `.design-sync/NOTES.md` 中的一条**：症状 → 根因 → 修复，非组件特定时标记 `[GENERAL]`。
2. **再选三个，追求多样性：** 一个复合/覆盖层（Dialog/Tabs），一个图标或资源密集型**其 story 加载远程图片**（这是 `[ASSETS_BLOCKED]` 金丝雀——§3 的行：网络沙箱 shell 在两个面板上使资源空白，所以评级虚假通过；在此处浮现它花费一个组件的重新捕获，在全名单通过后浮现它花费整个通过），一个主题/provider 敏感型——并确保集合跨越一个**文本密集型**组件（字体/排版 bug 在仅 Button 的 solo 中隐藏，然后使整个评级波次失效）。相同循环，solo。*增量路径：* solo 集合，一旦每个 story 评级为 `match`（或按标准的接受栏 `close`），是第一个验证批次——推送它（基础 SKILL.md §3）。
3. **第一次全名单捕获——按有 story 的组件数量规模分控。**
   - **20 或更少：** 对名单运行一次完整 `compare.mjs`。通过 shell 工具的后台模式在后台运行并等待完成通知——§2.2 的规则，在此重申因为这是被违反的地方：前台 `sleep`-轮询会阻塞唤醒你的通知，而 `pgrep -f` 循环匹配自己的命令行并旋转到超时。（Headless / `-p` 会话：改为同步运行——headless 模式没有任务通知重新调用，所以后台运行永远不会被恢复。）如果 ≥30% 的组件以*相同原因*失败，那是你遗漏的全局问题——在配置中修复并在分派前重新运行。**在重建之前批量处理列表显示的每个 skip 和配对修复**——每次重建+对比周期花费数分钟；逐个修复按每项支付该成本。
   - **超过 20：不要运行单一的全量捕获。捕获在 §4c 的批次内进行**——每个子智能体运行一个范围 `compare.mjs --components <its batch>` 并评级它刚捕获的表格。这带来三个好处：范围捕获并发运行（名单以串行扫描的时钟时间的一小部分渲染）；评级在第一批表格存在时开始而不是在最后一个组件渲染后；当一波浮现 `[GENERAL]` 问题时，风险中的工作是到目前为止评级的几个批次，而不是整个名单的捕获和评级。≥30% 相同原因检查随捕获移动——它变成波次 1 的学习审查（§4c 波次间）。你不跳过的全名单运行是 §4d 收据：到那时一切都已评级，所以它携带组件前进而不是重新捕获它们，花费秒级而非分钟级。

### 4c. 扇出——并行子智能体

将仍需工作的组件分成 5-8 个一批——在大型 DS 上（§4b 步骤 3 的 >20 门控）那是 solo 集合之外的每个组件，大多数还没有捕获表格；在小型 DS 全量捕获后是不匹配集合。将相关组件分组在一起（共享 provider、共享 fixtures——一次诊断服务于整批）。每波最多启动 4 个子智能体（Agent 工具，在一条消息中以便它们并发运行）。四个也是浏览器并发上限：每个子智能体的范围对比运行自己的 chromium，超过约 4 个并发捕获有机器级争用导致启动失败的风险。对于每个子智能体，填写此提示中的每个 `{…}` 并粘贴**当前** NOTES.md 内容（子智能体通过它继承 solo 阶段的学习）：

```text
Fix design-sync previews so they match the repo's own storybook render.
Repo: {REPO_ROOT}. Your components (yours alone): {COMPONENT_LIST}.

Why this matters: this design system is being synced to claude.ai/design, where
a design agent will build real UIs from this exact compiled bundle. The
storybook render is the proof of how each component is supposed to look; a
preview that matches it proves the component arrived intact, and one that
doesn't means every design the agent builds with it will be wrong the same way.

Artifacts per component (read these first):
- {OUT}/_screenshots/compare/<group>__<Name>.png — the true storybook render (left) vs the true preview render (right), per story. Full-res originals in {OUT}/_screenshots/compare/raw/.
- .design-sync/.cache/compare/<Name>.json — pairing facts + shot paths (no similarity scores — your eyes are the judge).
- The preview source (real JSX importing from '{PKG}'): .design-sync/previews/<Name>.tsx when owned, else the generated .design-sync/.cache/previews/<Name>.tsx. Your fixes are written to .design-sync/previews/<Name>.tsx (step 2).
- {OUT}/.stories-map.json — maps components to story ids; find each story's source file via its id in .design-sync/sb-reference/index.json (`importPath`). The story source is the authority on intended props/composition.
- .ds-sync/storybook/SKILL.md §4 — the grading rubric and fix decision tree.

First action, once for the whole batch: if any of your components has no compare sheet yet, run
  node .ds-sync/storybook/compare.mjs --out {OUT} --storybook-static {SB_REF} --components {COMPONENT_LIST}
One scoped run captures every missing sheet in your batch (one browser launch, not one per component); components already graded with unchanged sources skip automatically.

Per component (max 3 iterations):
1. Read the sheet; judge the primary story FROM THE TWO IMAGES (raw PNGs when the sheet is too small) per the §4 sampling rule — exhaustively when the component has portals, theme/provider sensitivity, an owned preview, or any warning; diagnose failures via the decision tree.
2. Copy .design-sync/.cache/previews/<Name>.tsx to .design-sync/previews/<Name>.tsx and DELETE its first-line `// @ds-preview generated …` marker (owned files live in previews/, win over the generated twin, and are durable + committed; an in-place cache edit survives rebuilds on this machine but is gitignored and vanishes on a fresh clone). The `@ds-stories/...` imports work unchanged from the new location. Mirror the story's JSX; inline story-local fixture data.
3. node .ds-sync/lib/preview-rebuild.mjs --config .design-sync/config.json --node-modules {NM} --out {OUT} --components <Name>
4. node .ds-sync/storybook/compare.mjs --out {OUT} --storybook-static {SB_REF} --components <Name>   (your edit changed the component's contract, so this clears its old grade — that's intended)
5. Re-Read the fresh sheet and Write your verdicts to .design-sync/.cache/compare/<Name>.grade.json ({"stories": {"<story>": {"verdict": "match|close|mismatch", "note": "…"}}}); siblings you trust under the §4 sampling rule get {"verdict": "match", "basis": "sibling-trusted"} — written in the same single grade.json Write, no image opens for them. Done when you grade every story match. A close story is still a fix target — if you can name the delta, try the knob for it; accept close only when an iteration didn't improve it or there's no actionable cause, and the note must say what's off AND what you tried. Blocked after 3 iterations → grade honestly (mismatch/close + note), record the exact blocker, move on.

HARD RULES — violating these corrupts other agents' work:
- Edit ONLY .design-sync/previews/{<your components>}.tsx, your components' .design-sync/.cache/compare/*.grade.json files, and .design-sync/learnings/{BATCH_ID}.md.
- NEVER edit .design-sync/config.json, .design-sync/NOTES.md, .ds-sync/, or any other component's files.
- NEVER run package-build.mjs or package-validate.mjs — they rewrite the shared bundle. preview-rebuild.mjs + compare.mjs scoped via --components are your only build commands.
- NEVER write an image-judged grade for images you haven't Read in this iteration. A sibling-trusted verdict must carry "basis": "sibling-trusted" and is allowed only when the image-judged primary story graded match and the component is warning-free (§4 sampling rule).
- A story that doesn't render in storybook either (sb-error) needs cfg.overrides.<Name>.skip; likewise [PORTAL?] needs cfg.overrides.<Name>.cardMode "single". Both are config edits you may NOT make — record them in your learnings file and final report; the orchestrator applies them. NEVER "fix" overlay bleed by neutralizing a story's open state in the .tsx — that destroys the fidelity being verified.
- If the SAME root cause appears in 2+ of your components — or even once when the cause is config-level (provider/css/font/token/import resolution) — STOP on those components: it's global. Write it to your learnings file `[GENERAL]`, report it, do not work around it per-component. Per-component fixes for a global cause are worse than waste: nothing ever machine-deletes `.design-sync/previews/`, so an owned preview you land for it persists and SHADOWS the corrected generated preview on every future build.

Learnings: append to .design-sync/learnings/{BATCH_ID}.md as you go — one bullet per discovery:
`<Component>: <symptom> → <root cause> → <fix>`, prefixed [GENERAL] if it applies beyond that component.

Known repo gotchas (read before starting):
{CURRENT_NOTES_MD_CONTENT}

Final report: per component — match/close/blocked + one-line reason; then any [GENERAL] learnings verbatim.
```

**波次间（编排者）——学习折叠是强制性的，不是可选的：**
1. 读取每个 `.design-sync/learnings/*.md`。将 `[GENERAL]` 条目提升到 `.design-sync/NOTES.md`（去重；保持简洁），然后删除你已折叠的每个学习文件。完整 `compare.mjs` 运行在任何学习文件存在时打印 `[LEARNINGS_UNMERGED]`，§4d 驱动器收据在相同条件下判定失败——被忽视的折叠不能悄无声息地发布。
2. **在下一波启动之前立即对每个 `[GENERAL]` 学习采取行动——无论多少组件显示了它。** 2/24 的发生率仍然是全局的；在未处理的 `[GENERAL]` 之上分派的波次按组件重新支付它，当配置修复最终落地时那些评级被冲掉。应用配置修复，**删除子智能体为解决相同原因而编写的任何拥有的预览**（拥有的文件从不被机器删除——留在原地它们会遮蔽修复），然后完整重建（真正的构建——步骤 3 的批次推送上传磁盘文件，所以绝不 `--skip-dts` stub）+ 验证。然后用范围 `compare.mjs --components` 在 1-2 个实际受到该问题影响的组件上证明修复有效——**不要在活动中运行全名单对比。** 重建已经清除了修复的契约变更触及的评级；那些组件只需重新加入队列，下一波的范围运行重新捕获它们，§4d 收据在最后结算整个名单。活动中*捕获*大量组件的全名单运行是症状，不是常规步骤：要么捕获的组件从未被评级（每批必须评级它捕获的所有内容），要么全局切片配置编辑清除了已获得的评级——在支付渲染时间之前诊断。
3. *增量路径：* 将本波中现在满足 §4d 评级栏的组件作为验证批次推送（基础 SKILL.md §3）——在步骤 1-2 之后，这样本波的全局修复先重建它们。
4. 下一波获得更新的 NOTES.md 内容和仍然失败的组件。最后一波后，对剩余内容重复步骤 1 并删除 `.design-sync/learnings/`。

### 4d. 完成标准 + 报告

- **一次 §7 驱动器运行即收尾收据——每条路径。** 将会话的最终构建设为驱动器（`resync.mjs`）；无锚定时省略 `--remote`（首次同步、恢复的项目）——锚定项目的完整重新验证仍然通过它。门控是驱动器的判定：`ok: true` 且 `verification.pendingGrade` 为空。其捕获范围是其工作列表的可捕获子集——首次同步时为每个有 story 的组件，重新同步时为 `changed`+`added` 集合——携带的评级跳过，所以收据花费范围通过而非完整重新捕获（不可捕获的成员通过上传分区重新发布，无需评级；通过上传验证的组件在门控之外）。驱动器自身检查 `.design-sync/learnings/` 并在任何未折叠的学习文件存在时以 `[LEARNINGS_UNMERGED]` 判定失败（`.compare-report.json` 聚合保持仅全量运行）。在此最终运行中，每个在范围内的组件应打印 `carried forward` 且零 `grade cleared`——该行就是下次同步将很快的证明。无变更运行上的清除评级意味着非确定性源输入（不稳定 story 内容）——现在追查它；驱动器触发的 `[SPOT_CHECK]` 不是那个（流水线变更被自动验证——确认表格并继续）。
- 每个在范围内的有 story 组件都有当前的 `.grade.json`，每个 story 为 `match`——或达到标准接受栏的 `close`（§4）——或通过 `cfg.overrides.<Name>.skip` 跳过并附 NOTES.md 理由。机械检查是驱动器的 `verification.pendingGrade`：列在其中的组件有 story 没有当前判定且未完成（通过上传验证的组件豁免）。
- `package-validate.mjs` 在最终重建后仍退出 0，没有未解决的 `[FONT_MISSING]`（§4a——对比预言机看不到的唯一警告）。
- 从最终的 `ds-bundle/.render-check.json`（由 `package-validate.mjs` 写入；`iterations` = 完整重建通过次数）调用 `DesignSync({method: 'report_validate', counts: {total, bad, thin, variantsIdentical, iterations}})`。在驱动器范围收据（§7）上该文件缺失（跳过层级）或仅覆盖采样——当此调用需要完整计数时先用 `--render-sample 0` 重新运行驱动器；在不上传任何内容的无变更重新同步上，跳过该调用。
- NOTES.md 有当前的**重新同步风险**部分，现在趁你仍然知道时编写：什么可能悄无声息地过时（内联到配置中的数据、中和的 story 导出、绑定到上游 API 的拥有的预览），什么仅部分验证（story 上限、接受的 `close` 理由），以及构建假设了什么（工具链版本、CDN 获取的资源）。修复记录你做了什么；此部分告诉下次运行要注意什么。
- 告诉用户：N/M 个组件评级为 match，哪些是 `close`（以及为什么可接受），哪些被跳过及原因。

## 5. 当仓库很怪——逃生舱

对不寻常仓库的首次运行一定会遇到默认值未覆盖的情况。每个启发式都有提交的 override——规则是：**绝不手动修补生成的输出；将修复放在下次运行读取的文件中。** 从失败类别到旋钮的映射：

| 仓库的怪异之处 | 旋钮 | 所在位置 |
|---|---|---|
| 非标准构建/入口（`module` 指向 TS 源码、exotic dist 布局） | `cfg.entry`、`cfg.buildCmd` | 配置 |
| CSS 由独立流水线构建 / 无 dist 副作用文件 / CSS-in-JS | `cfg.cssEntry` 如果有文件；否则依赖 `[CSS_FROM_STORYBOOK]`——转换器从 `sb-reference` 中抓取**编译后**的 CSS，这是通用兜底：无论流水线多怪，其输出都在 storybook 构建中 | 配置 |
| Token 作为独立包发布 | `cfg.tokensPkg` | 配置 |
| 来自运行时服务/专有 CDN 的字体 | `cfg.extraFonts`、`cfg.runtimeFontPrefixes` | 配置 |
| 子路径导出上的图标或组件 | `cfg.extraEntries` | 配置 |
| 命名约定（story 标题 ≠ 导出名） | `cfg.titleMap`；story↔单元配对也回退到顺序 | 配置 |
| 不会打包的装饰器/provider（仅 vite 插件、MDX、别名） | `cfg.provider`——显式链优于装饰器打包；`probe.mjs` 从实时 storybook 推断它；或在组件自身的 `.tsx` 中**内联**组合 provider（拥有的预览可以导入并包装包导出的任何内容） | 配置 / 预览 |
| 无法静态渲染的 story（MSW、数据获取、交互测试） | `cfg.overrides.<Name>.skip` + NOTES.md 一行说明原因。Skip 移除 story 的单元，但包装器仍导入整个 story 模块——如果文件在导入时崩溃（模块作用域 fetch/worker），接管 `.tsx` 并删除导入 | 配置 |
| `[PORTAL?]`——覆盖层/portal story 在网格卡片中绘制到单元外 | `cfg.overrides.<Name>.cardMode: "single"`（+ 可选 `primaryStory`、`viewport: "WxH"`）——单 story 卡片、fixed 定位包含、声明的产品视口。对比仍通过 `?story=` 评级每个 story | 配置 |
| `[GRID_OVERFLOW]`——验证测量了网格卡片的几何：`wide` = story 渲染比其单元更宽（单元裁剪在产品中裁剪它们）；`escape` = fixed/portal 内容定位到任何单元之外 | 应用警告指出的 override——`wide` → `cardMode: "column"`（每行一个 story、完整卡片宽度、所有 story 保留）；`escape` → `cardMode: "single"` + `primaryStory`。`.render-check.json` 中的结构化拷贝（`gridOverflow`、`gridOverflowCells`、`suggestedOverride`）。将每个标记的组件批量到一个定向重建中（`preview-rebuild.mjs --components A,B,C`）——仅展示编辑不触发 `[CONFIG_STALE]` 且评级携带。不要追查干净的重新验证来确认：应用的补救措施不会重新标记（single 完全豁免；column 不会重新标记 `wide`——escape 保持监控，所以后来添加的 portal story 仍然浮现）；如果想视觉确认则查看 `.review.html` | 配置 |
| `[EXPORT_COLLISION]`——同级包（图标等）导出了主包也导出的名称 | 主包赢得全局合并，所以从同级导入失败名称的 story 渲染了错误内容 | 日志指出修复：`cfg.storyImports.bundle: ["<sibling>"]` |
| `[FILE_TOO_LARGE]`——构建输出超过上传的每文件 12 MB 上限 | 通常是打包到预览或装饰器打包中的仅开发用重量级内容（语法高亮器、代码形式图标） | 现在就瘦身，在评级之前——评级后瘦身拥有的预览会重新评级该组件 |
| `[PROVIDER_UNEXPORTED]`——`cfg.provider` 组件不是打包导出 | 构建在发出任何组件预览或文档之前退出 1——输出目录不完整；修复后重建 | 使用确切的导出名，或通过 `cfg.extraEntries` 重新导出。检查读取打包自身的导出列表，所以缺失是可靠的；隐藏在打包 CommonJS 重新导出后面的名称无法枚举——那些以 `[PROVIDER_UNVERIFIED]` 警告构建；如果每个预览都失败 "Element type is invalid"，名称就是错的 |
| story 导入解析方式错误（应该 shim 时被 bundle，或反之——任何导入风格） | `cfg.storyImports.shim` / `cfg.storyImports.bundle`——按解析路径匹配的子串模式（裸包导入按**说明符** shim，不解析——为那些匹配说明符）。未知包子路径（`<pkg>/utils`）默认打包；如果某个应该骑全局，添加到 `cfg.extraEntries`。在包自身源码仓库中打包的自导入没有东西可解析——先将 `node_modules/<pkg>` → 构建的 `dist/` 符号链接 | 配置 |
| story 文件导入默认值无法加载的资源类型（`.yaml`、`?raw`、svg-as-component） | `cfg.storyImports.loaders`——与默认值合并的 esbuild loader 映射（例如 `{".yaml": "text"}`） | 配置 |
| 生成的预览有错误的 props/构图 | 将 `.design-sync/.cache/previews/<Name>.tsx` 复制到 `.design-sync/previews/<Name>.tsx` 并删除其标记行（永久拥有） | 预览 |
| 源码/文档发现遗漏（不寻常的仓库布局） | `cfg.componentSrcMap`、`cfg.docsMap`、`cfg.dtsPropsFor`、`cfg.srcDir` | 配置 |
| 更深层的问题——自定义 story 格式、exotic args 提取、CSS 变换 | fork 适配器：将打包的 lib 模块复制到 `.design-sync/overrides/<name>.mjs` 并在 `cfg.libOverrides` 中声明并附一行原因（构建双向交叉检查：`[OVERRIDE_UNDECLARED]` / `[OVERRIDE_MISSING]`）。Fork 被提交，所以重新同步自动使用它们。**`emit.mjs` 和 `bundle.mjs` 是应用契约面——绝不 fork 它们。** | `.design-sync/overrides/` |

对于**story 处理**，fork 点按关注点划分：`story-imports.mjs`（预览编译的所有导入解析策略——为逐仓库定制而构建的接口；完整构建和 `preview-rebuild.mjs` 都遵守它）、`source-storybook.mjs`（index.json 发现、title→component 映射、story 源解析 + 导出配对）、`preview-gen-storybook.mjs`（包装器模板 / composeStories 语义）、`css-fallback.mjs`（从 storybook 构建中抓取 CSS/字体）。fork 拥有损坏的*最窄*模块，保持其导出签名，并在 NOTES.md 中记录仓库的不同之处——下次同步继承所有这些。Fork 从 `.design-sync/overrides/` 加载而其同级留在暂存脚本中——将 fork 的相对导入（`./common.mjs` 等）重新指向 `../../.ds-sync/lib/`。导入裸转换器依赖（`esbuild`）的 fork 还需要 `ln -sfn ../.ds-sync/node_modules .design-sync/node_modules` 以便 node 能从 fork 的位置解析它——每个 clone 一次，不是永远一次：链接被 gitignore（`node_modules` 规则）而需要它的已提交 fork 在 clone 后存活，所以重新创建它是全新 clone 设置的一部分。

梯子的最后一档，对于真正在转换器范围之外的仓库：**上传格式是契约，不是转换器**（见基础 skill）。以仓库允许的任何方式生成布局——但 `package-validate.mjs` 和对比/评级门控对你产生的任何内容不变地适用。预言机从不被 fork。

该表中的所有内容都是提交的文件，§2.3 要求在做任何事之前读取现有配置 + NOTES.md——所以第 N+1 次重放运行 N 做出的每个决策。当你在奇怪仓库上修复某些东西时，问自己："哪个提交的文件使这下次自动？" 如果答案是无，那至少是一个 NOTES.md 条目——可能还值得在此处添加一行。

## 编写约定头（上传前）

预览验证后——无论是新编写的还是重新同步携带的——运行基础 SKILL.md 中的约定编写步骤（"编写约定头"）——它将你刚学到的使预览渲染的内容提炼到 `.design-sync/conventions.md` 中，通过 `readmeHeader` 配置键连接。顺序很重要：先编写文件并设置键，然后按基础步骤的**重建规则**重建（每条路径上的全新驱动器运行——首次同步省略 `--remote`），使生成的 README 实际携带头部，§4d 收据描述 §6 上传的构建。然后进行下方上传。

## 6. 上传

适用哪条路径由基础 skill §1 路由器决定（运行开始时 pin → 原子；否则空 → 增量，非空 → 原子）：

**增量路径**（首次同步到空项目）：计划自本文件的 §3 门控以来一直开放，验证批次已落地。在 §4d 通过且约定头步骤已运行后（基础 SKILL.md——它必须先于其重建馈送的上传），运行基础 SKILL.md §3 中的收尾——哨兵围栏 → 完整内容写入 → 对账删除 → 哨兵重新武装 → `_ds_sync.json` 最后。本节的分块、卫生和保留本地规则适用于这些写入；`projectId` 已在 §1 中记录；本节末尾的交接审计仍然适用。跳过本节其余部分的序列——那是原子路径。

**原子路径**（重新同步，或任何非空目标——它可能正在使用中，所以在一切验证后一次性更新）：下方所有内容。仅在 §4d 和约定头步骤之后（基础 SKILL.md）。`DesignSync(finalize_plan)`，`localDir: "./ds-bundle"`。

- **写入——一切，始终**（完整重新验证和重新同步一样）：`writes: ["components/**", "tokens/**", "fonts/**", "_vendor/**", "_preview/**", "guidelines/**", "_ds_bundle.js", "_ds_bundle.css", "styles.css", "README.md", "_ds_sync.json", "_ds_needs_recompile"]`。重新上传未更改的文件是幂等且廉价的。范围不足的写入列表悄无声息且永久地使项目不同步——完整写入是安全默认值。
- **删除。** 锚定重新同步：从 diff 原样复制——完全复制 `.sync-diff.json` 的 `upload.deletePaths`；绝不手动推导列表，diff 列出路径时绝不传 `[]`。无锚定（正在完全重新验证的重新采用或恢复的非空项目）：diff 看不到项目历史，所以现在——在 `finalize_plan` 之前——审查其 `list_files`，查找此构建不产生的文件，并将这些审查的路径放入计划的 `deletes`（计划中未命名的删除被拒绝）。
- **§4d 收尾收据兼作上传的真实来源。** 会话的最终构建已经是 §7 驱动器运行（§4d）；裸 `package-build.mjs` 运行清除 `.sync-diff.json`，驱动器的 diff 阶段重新生成它，所以 `deletePaths` 和 `upload.any` 描述你上传的确切字节——一次运行既是验证收据又是上传清单，之后没有单独的完整对比。
- **`upload.any === false` → 完全跳过上传**——项目已与此构建匹配。（下方的交接审计仍然适用。）
- **`_ds_sync.json` 是绝对的最终写入**——在所有内容写入、所有删除和哨兵重新武装之后，在其自身的 `write_files` 调用中。提前上传，计划中途失败使锚定为项目没有的文件担保，而确定性重建意味着没有后续同步会修复它们。
- **保留本地**：`_sb/**`（storybook-static 是参考，从不上传）、点前缀条目（`.stories-map.json`、`.compare-report.json`、`.ds-build-meta.json`、`.sb-static/`、`.sync-diff.json`）和 `_screenshots/`。`_vendor/` 和 `_preview/` **要**上传——预览卡片从它们加载 React 和编译的预览。

如果 `finalize_plan` 被拒绝，**停止**——拒绝意味着会话无法批准，不是参数错误。告诉用户什么被拒绝了并询问他们想如何继续：再次尝试批准，或自行获取验证的 `ds-bundle/` 并交互式运行上传。

计划批准后，上传是固定序列：

1. **哨兵优先**：`DesignSync(write_files, [{path: "_ds_needs_recompile", localPath: "_ds_needs_recompile"}])`——它将应用的清单/复制机制围栏在半上传状态。
2. **所有内容写入**，分块为 ≤256 文件的 `write_files` 调用，在同一 `planId` 下。服务器也按字节限制有效载荷，不仅是文件数——将二进制密集目录（fonts/、images）分到更小的块，在 500 时将块大小减半并重试。
3. **所有删除**：`DesignSync(delete_files)` 遍历 `upload.deletePaths` 中的每个路径。（无锚定：你在 `finalize_plan` 时审查到计划 `deletes` 中的路径——上方删除要点。）如果 `delete_files` 拒绝不存在的路径（底板卡片组件没有 `_preview/` 文件），去掉被拒绝的条目重试——那个 not-found 拒绝是你唯一可以继续过去的失败。
4. **哨兵重新武装，然后 `_ds_sync.json` 最后。** 锚定在删除之后也要——失败的删除会留下刷新的锚定不再能看到的远程文件。

任何其他重试无法清除的写入/删除失败意味着**停止**——不重新武装哨兵，不写 `_ds_sync.json`。未锚定的项目仅在下一次同步重新验证；半应用上传之上的新锚定是永久的。

**上传卫生**：将文件列表和分块清单保留在 `.design-sync/` 下——绝不用裸 `/tmp` 路径，那里另一个仓库同步的陈旧列表会上传错误的设计系统。在上传前立即从活跃的 `ds-bundle/` 重新生成列表，并做健全性检查：组件名称属于此设计系统，打包的 `window.<globalName>` 匹配。最后用 `DesignSync(list_files)` 确认计数。

仅在_post-upload `list_files` 计数验证后，**将 `projectId` 记录到 `.design-sync/config.json`**（如果缺失或不同——这是后备——§1 在目标确定时为每条路径记录 id，所以通常已存在；绝不能发生的是在_upload 验证之前记录 id，将配置 pin 到内容尚不真实的项目）——它 pin 哪个项目锚定未来重新同步。完成后，告诉用户：项目 URL（`https://claude.ai/design/p/<projectId>`）、组件计数、对比结果摘要以及验证退出干净。持久集（下方交接审计中的规则：`.design-sync/` 下未被 gitignore 的所有内容）必须落地到仓库以供重新同步重用每个修复；验证状态与上传的 `_ds_sync.json` 一起存在，不在 git 中。下方的交接审计涵盖提交提议。

**最后一步——审计交接。** 未来运行只有在本运行留下的一切基础上才能快速且正确；验证它，不要假设：

1. `git status`——持久集（`.design-sync/` 下未被 gitignore 的所有内容——今天 config.json、NOTES.md、`conventions.md`、`previews/`、`overrides/`；规则是契约，所以未来的持久文件按构造在集合中）是同步的仓库足迹；`sb-reference/`、`learnings/`、`.cache/`、`.ds-sync/` 被忽略。如果本次运行创建或更改了任何持久文件，**提议提交它们并开 PR**（一次提交，仅同步状态——无无关文件）。未提交的修复是下次同步没有的修复。
2. 以你是下一个智能体、对本次会话一无所知的身份重读 NOTES.md：仅凭所写内容能跳过今天的调试吗？每个拥有的预览、skip、配置旋钮和 lib fork 都应追溯到一条条目，重新同步风险部分应是当前的（§4d）。现在就写缺失的——今天花一分钟，以后花重新推导。
3. 重新同步后——无论更改了多少或重新评级了多少——保持 NOTES.md 和 git 状态与你发现它们时完全一致，除非运行产生了下次运行需要知道的内容；只有当它为未来同步增加价值时才给用户东西提交。

## 7. 重新同步——一条命令路由工作

仓库携带同步的输入（配置、拥有的预览、NOTES.md）；上传的项目携带锚定（`_ds_sync.json`）。先读 NOTES.md（重新同步风险是关注清单），然后：

1. **刷新输入。** 重新复制暂存脚本（§2.4 的 `cp -r` 行——即时；陈旧的 `.ds-sync/` 运行旧转换器对应这些指令）。每当 DS 源可能已变时重新运行 `buildCmd` **并重建 `.design-sync/sb-reference`**——它们必须一起移动；不确定时都重建（确定性构建使不必要的重建成为空操作；捕获日志中的 `[REFERENCE_STALE?]` 意味着你忘了）。全新 clone 额外项：§2.4 的依赖安装 + chromium、§2.2 的 sb-reference 构建，以及——如果仓库携带 `.design-sync/overrides/` fork 且有裸导入——`ln -sfn ../.ds-sync/node_modules .design-sync/node_modules`。
2. **获取锚定**：`DesignSync(get_file, path: "_ds_sync.json")` → 保存到 `.design-sync/.cache/remote-sync.json`。项目中无 sidecar → 首次同步范围（下方省略 `--remote`）。
3. **从仓库根运行驱动器**：

   ```sh
   node .ds-sync/resync.mjs --config .design-sync/config.json --node-modules <nm> \
     [--entry <dist-entry>] --out ./ds-bundle --remote .design-sync/.cache/remote-sync.json
   ```

   它链接构建 → diff → 验证 → 捕获（范围限定到新增 + 契约变更的组件）并打印一个判定 JSON（也写入 `ds-bundle/.resync-verdict.json`）。阶段日志流式输出到 stderr。驱动器是幂等的——修复后重新运行它。对于逐组件预览迭代改用 §4a 定向循环（秒级，不是完整构建 + 渲染检查）；驱动器重新运行是收尾收据。

   驱动器还按 diff 证明的范围限定验证的渲染检查（显式 `--render-sample` / `--no-render-check` 标志始终优先）。有健康的锚定且打包 + 样式未变时，每个未变预览的渲染输入与上次上传渲染验证（或显式接受）的字节相同——diff 将锚定 pin 到新 sidecar，`[SYNC_STALE]`/打包 sha 重新计算将渲染面 pin 到磁盘（样式由刚写入两者的构建 pin 定），重新渲染相同字节测试的是你的 chromium 安装而非构建产物。所以：完全没变 → 渲染检查**跳过**（该运行上的 `[RENDER_SKIPPED]` 警告由驱动器宣布且是预期的——不是要追查的新警告）；有东西发布但没有影响渲染的内容移动（docs/guidelines 编辑、锚定刷新）→ **采样**（`--render-sample N`，默认 10）；打包或样式已变 → **完整渲染检查**（`--render-sample 0`）——变更重新渲染每个预览，所以未变检查是安全采样时唯一会遗漏 bug 的地方。在所有情况下对比仍按其正常规则运行（携带/清除），作为这一步的确定性补充而非替代——`compare.mjs` 仍然做评级门控的判定。

4. **如果判定为 `ok: true`——完成。** 上传（§6 原子路径——以驱动器的 `.sync-diff.json` 为权威），然后在将 `projectId` 写入 config.json 之前用 post-upload `list_files` 计数验证（上方）。然后审计交接（§6 末尾）。如果 `ok: false`，其 `stages` 数组告诉你哪个阶段失败以及哪些组件需要工作——它们是 §4 的输入。在修复和重新运行驱动器之间，§4a 的定向循环是逐组件迭代的地方（秒级 vs 完整构建+渲染检查）；驱动器重新运行是收尾收据，不是迭代工具。`verification.pendingGrade` 列出需要评级的组件；`verification.canary` 列出样式/字体变更触及的组件——抽查它们的表格（§7 步骤 4）以捕获采样渲染检查可能遗漏的回归。如果驱动器自身打印 `[SPOT_CHECK]`，读取这些表格并确认评级仍然成立——流水线变更被自动验证。

   **全名单抽查**——仅当**显式**审计整个已评级名单时运行（用户请求、可疑的系统性回归、或 §4d 完成标准要求）：`compare.mjs`（不限定范围）+ 验证携带评级的每个组件。将结果与 `.design-sync/.cache/compare/*.grade.json` 中已提交的评级进行比较——任何不匹配都是回归。这是一项昂贵操作（大型 DS 需数分钟），不是常规步骤。如果抽查发现回归，**仅在**已通过上传验证的组件上——它们的评级被锚定且上次驱动器运行已通过——回退到 §4 完整流程（从§4b solo 开始）。通过上传验证的组件也在门控之外——它们的锚定是权威。

   **大范围变更后**（打包重建、storybook 配置编辑、skill 升级）：`--force` 重新捕获一切并清除所有评级——系统性重新验证。仅在变更证明成本合理时使用。

   **自动化重新同步**（CI/脚本）：`resync.mjs` 是唯一需要的命令。检查 `.resync-verdict.json` 中的 `ok` 字段——`true` → 上传，`false` → 人工干预。不要在自动化中运行 `--force`——它清除评级并需要完整人工 §4 通过。
