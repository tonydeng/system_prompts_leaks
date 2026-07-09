> **说明**：本文件为英文原文（`SKILL.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: run-skill-generator
description: Teach /run and /verify how to build and launch your project by creating a per-project run skill with a driver script.
---

你的工作是在 `<unit>/.claude/skills/run-<unit-name>/` 生成一个 **skill**，让未来的智能体可以从干净的机器上构建、启动和**驱动**这个项目。

skill 有两个部分，它们放在一起：

```
<unit>/.claude/skills/run-<unit-name>/
  SKILL.md      ← 面向智能体的说明 — 简短。指向驱动程序。
  driver.mjs    ← (或 driver.py, smoke.sh, … — 或无：web 应用使用现成的
                   chromium-cli，SKILL.md 中的 heredoc 就是脚本)
```

这几乎总是意味着**编写代码**，而不仅仅是文字。如果应用有任何交互界面（GUI、TUI、长时间运行的服务器、REPL），未来的智能体需要一种程序化的方式来操作它。仅靠 markdown 文件无法点击按钮 — 但有时按钮点击器已经存在：对于 web 应用，它是 `chromium-cli`；对于服务器，它是 `curl`。你现在构建（或编写）这个工具，将它与 skill 一起提交，`SKILL.md` 文档化如何使用它。

## 完成定义

当**所有**以下条件都为真时，你就完成了：

1. **你在这个容器中启动了应用并与之交互** — 不是它的测试套件，而是实际运行的应用。对于任何有 GUI 的应用，这意味着你在磁盘上有一个截图文件，是你拍摄的。
2. **交互工具已提交**在 skill 旁边。驱动脚本、REPL 包装器、冒烟测试，或 `SKILL.md` 中内联的 `chromium-cli` heredoc — 无论你在步骤 1 中用什么来驱动应用。（已升级到 `scripts/`/`e2e/`？ — 很好，指向它。使用现成 `chromium-cli` 的 web 应用？ — 内联脚本就是工具；不需要单独的文件。）
3. **`SKILL.md` 文档化工具**作为主要的智能体路径 — 未来的智能体首先阅读的部分是"运行此驱动程序 / 将这些命令管道传输到 `chromium-cli`"，而不是"运行 `npm start` 然后窗口打开"。
4. **`SKILL.md` 中的每个代码块都是你运行过的有效命令。** 本次会话。本容器。不是来自 README，不是推断的。

如果你正要编写 skill 但没有（1），**停止。** 你即将改写现有文档。该文档已经存在 — 它叫 README，你在这里的全部原因就是它不够用。

## 交付物是代码和文档

典型输出是包含两者的 skill 目录：

```
<unit>/.claude/skills/run-<unit>/
  SKILL.md         ← 简短。指向驱动程序。具有 frontmatter，让 Claude 在有人要求
                      "运行 <unit>" 或"截图 <unit>"时自动加载它。
  driver.mjs       ← (或 driver.py, smoke.sh, … — 或无：web 应用使用现成的
                      chromium-cli，SKILL.md 中的 heredoc 就是脚本)
```

驱动程序默认位于 **skill 目录内**。它们是一对 — skill 的说明和实现它们的代码。位于这里的驱动程序可以比生产代码稍微混乱一些；它是智能体工具，不是产品表面。

**升级：**如果驱动程序成长为项目自己的测试套件想要重用的东西 — 共享启动助手、真正的 e2e 工具 — 将其移动到 `scripts/` 或 `e2e/` 并更新 `SKILL.md` 以引用新路径。skill 保持不变；驱动程序找到更好的家。

确切形状取决于项目，但原则是恒定的：**驱动程序是交付物。** `SKILL.md` 是它的手册页。对于 web 应用，驱动程序已经存在 — `chromium-cli`（[examples/playwright.md](examples/playwright.md)）— skill 是运行它的脚本。对于桌面应用（[examples/electron.md](examples/electron.md)），驱动程序是 tmux 下的自定义 REPL，暴露 `launch`/`ss`/`click`/`eval`。对于服务器，驱动程序是 `curl`。无论它采取什么形状，如果没有东西能够接触到运行中的应用，skill 就是一个没人能触摸的窗口描述。

## skill 的位置

skill 位于 `<unit>/.claude/skills/run-<unit-name>/`，其中 `<unit>` 是**一个可部署事物**的目录 — 应用、服务、库。

Claude Code **原生发现**来自嵌套 `.claude/skills/` 目录的 skill：在 `<unit>` 内任何地方工作的智能体都会将 `/run-<unit-name>` 视为可用 skill，当请求匹配其描述时（例如"运行桌面应用"，"截图计费"）它会自动加载。

- **单项目仓库：**仓库根目录的 `.claude/skills/run-<repo-name>/`。
- **包含多个应用的大型仓库：**每个应用一个，并置 — `apps/billing/.claude/skills/run-billing/`、`apps/desktop/.claude/skills/run-desktop/`。
- **具有多个二进制文件的应用：**仍然是应用根目录处的**一个** skill，每个二进制文件一个部分。它们共享设置。从最接近的单二进制文件示例开始，为每个二进制文件添加 `## Run: <name>` 部分。

如果你不确定单元边界在哪里，**询问用户。**

将目录名称 slugify：小写，空格用破折号，无斜杠（`run-billing-api`，而不是 `run-billing/api`）。目录名称和 frontmatter `name:` 应该匹配 — 这是斜杠命令。

## 流程

### 0. 查找任何关于运行此应用的现有 skill

列出项目的 skill 及其描述（与 `/run` 使用的相同探测 — 用户以各种方式命名这些，所以按描述匹配，而不是名称）：

```bash
d=$PWD; while :; do
  grep -Hm1 '^description:' "$d"/.claude/skills/*/SKILL.md 2>/dev/null
  [ -e "$d/.git" ] || [ "$d" = / ] && break
  d=$(dirname "$d")
done
```

如果有一个是关于启动/驱动此应用的 — 无论它叫什么 — **改进，不要重写**：验证其声明，修复错误，添加缺失内容，保留有效内容。如果有驱动程序，重新运行它。保留其现有名称。

（还要检查遗留的 `.claude/run.md` — 此工具的早期版本生成了这些。如果你找到一个，迁移它：主体成为 skill 的 `SKILL.md` 内容，任何引用的脚本移动到 skill 目录，然后删除旧文件。）

如果不存在，决定在哪里创建它（见上文）并继续。

### 1. 发现 — 并将每个声明视为可证伪

弄清楚你要为谁编写：

- 这里的清单（`package.json`、`go.mod`、`pyproject.toml`…）并且它是一个自包含的东西 → 这就是单元。
- 看起来像 mega-repo 根目录（`apps/`、`packages/`、`services/`）→ **询问哪一个。**列出候选，让他们选择，`cd` 到那里。
- 真正模棱两可 → 询问。

调查通常的地方：`README.md`、`package.json` 脚本、`Dockerfile`、`Makefile`、`.github/workflows/`、`CONTRIBUTING.md`。CI 配置通常比 README 更准确。

**现有文档中的每个声明都是一个假设。**尤其是负面的：

| 当文档说… | 你做什么 |
|---|---|
| "需要 macOS/Windows" | 无论如何在 Linux 上启动它。应用很少拒绝启动 — 它们在缺少 `.so` 时崩溃，`apt-get` 修复它。*你主机的*密钥链/通知的原生模块可能 no-op；核心通常运行。 |
| "需要 GPU" | 尝试软件渲染。Electron/Chrome 用 `--disable-gpu` 回退。 |
| "需要付费账户 / 功能标志" | 门是你可以阅读的代码。找到它（环境变量？构建定义？SSR 嵌入的 JSON？）并为你的本地运行修补它。文档化修补。 |
| "运行 `npm start`" | 这是人类路径（生成窗口，永远等待）。找到或构建*程序化*路径 — `electron-forge start` 构建然后通过 Playwright 启动，或等效的。 |

macOS 开发人员编写的 README 中的"不支持 Linux"意味着"我从未尝试过。"你即将尝试。**如果你在这里放弃，你编写的 skill 就是带有额外步骤的 README。**

### 2. 执行 — 并构建你需要的工具

你在无头 Linux 容器中。应用将与你对抗。这种对抗就是 skill 的内容。

在进行过程中保持一个运行的 `NOTES.md`。每个错误 → 每个修复 → 最终工作的每个命令。这个草稿本成为故障排除部分。

**努力实现真正的交互：**

- **安装 + 构建。**当缺少某些东西时，注意修复它的确切 `apt-get` / `npm install`。
- **启动应用。**不是测试套件 — 应用。桌面 GUI（Electron、原生）需要 `xvfb-run` 和少量 `lib*` 包；由 `chromium-cli` 驱动的 web 应用无头运行，两者都不需要。启动超时和神秘的崩溃在此阶段是正常的。阅读堆栈跟踪，安装缺少的东西，再试一次。
- **构建工具来驱动它。**你需要一个运行应用的句柄，让你能够程序化地发送输入并观察输出。形状取决于项目（见下表）。

   **覆盖 PR 实际接触的层。**操作 CLI 用户表面的 tmux 驱动程序是 UI 更改的正确句柄 — 对于接触一个内部函数的 PR 是错误的句柄。对于后者，智能体想要 `NODE_ENV=test bun run script.ts`（或等效的）：导入函数，调用它，观察。如果这里的大多数 PR 接触内部，该直接调用路径是驱动程序的主要入口点，tmux 启动是次要的。查看最近合并的 PR：它们接触什么层？覆盖那个。

   对于 **web** 应用，`chromium-cli` 是驱动程序 — 你编写它的脚本，你不编写它（见 [examples/playwright.md](examples/playwright.md)）。对于 **桌面** GUI（Electron），编写 REPL 驱动程序（stdin 命令 → 点击/类型/截图），在 tmux 内运行它，并使用 `send-keys` / `capture-pane`。你将迭代该驱动程序 — 它开始最小（`launch`、`ss`、`quit`），并增长你到达应用有趣部分所需的任何命令。
- **端到端执行一个真正的用户流程。**点击按钮。填写表单。在 DOM 中查看结果。截图。**实际查看截图。**如果它是空白或显示错误页面，你就没有完成。
- **然后运行测试。**单元测试是健全性检查，不是主要事件。
- **干净地停止。**

**障碍是内容。**你将遇到奇怪的障碍 — 不对齐的坐标系、在此 Electron 版本上返回空的 API、隐藏你需要测试的功能的功能门。这些每一个都会在 Gotchas 中获得一个要点，并且（通常）在你的驱动程序中获得一个助手。黄金标准是充满没人能猜到的东西的 Gotchas 部分。

**驱动程序脚本与 skill 一起提交。**它不是脚手架。它是未来智能体（和人类）将驱动此应用的方式。它默认位于 skill 目录内（对于使用 `chromium-cli` 的 web 应用，这意味着在 `SKILL.md` 中内联 — heredoc 就是脚本）。如果它超出了那个 — 如果项目的真实测试套件想要从中导入 — 将其移动到 `scripts/` 或 `e2e/` 并更新 `SKILL.md` 以指向那里。

### 3. 编写 SKILL.md

简短。指向驱动程序。使用 [template.md](template.md) 作为起始结构 — 它具有 frontmatter 形状。

**Frontmatter 很重要。** `name:` 成为斜杠命令（`/run-billing`）。`description:` 是 Claude 扫描以决定是否自动加载此 skill 的内容 — 将**智能体实际会输入的动词**放入其中："run"、"start"、"build"、"test"、"screenshot"。通用描述（"billing 的有用实用程序"）不会匹配。

主体结构：

1. 一段介绍：这是什么应用，如何驱动它 — 桌面应用在 xvfb/tmux 下的 `<driver-path>`，web 应用用 `chromium-cli`，服务器用 `curl`。
2. **先决条件** — 你运行的确切 `apt-get install` 行。
3. **构建** — 确切的命令，按顺序。包括你必须应用的任何修补（功能门、配置覆盖）以及确切的 `sed` 或编辑。
4. **运行（智能体路径）** — 首先。如何启动驱动程序，它接受什么命令，截图落在哪里。如果是 REPL，显示 tmux 包装。这是下一个智能体实际将使用的部分。
5. **运行（人类路径）** — 其次，如果不同。`npm start` → 窗口打开 → Ctrl-C。简短。注意它在无头模式下无用。
6. **Gotchas** — 战斗伤痕。看起来应该工作但不工作的东西，以及解决方法。如果此部分是通用的，你还没有足够努力地战斗。
7. **故障排除** — 症状 → 修复。只有你实际遇到的错误。

保持它**已验证**（你运行了它）、**规定性**（一条路径，不是选项）、**诚实**（不稳定？慢？这样说）。

**SKILL.md 中的路径相对于 `<unit>/`，**而不是 skill 目录。如果有任何歧义，在顶部说明这一点。当驱动程序位于 skill 内时，它从 `<unit>` 的路径是 `.claude/skills/run-<unit-name>/driver.mjs` — 它很长，但是明确的。

### 4. 验证

新 shell，`cd` 到单元，逐行遵循 skill 的 `SKILL.md` 而不偏离。任何即兴创作 = 缺口。修复它。

## 项目类型模式

为你的驱动程序选择一个起始形状。这些示例与 `/run` skill 共享（当不存在项目特定的 run skill 时，使用相同的每个项目类型模式作为后备）— 如果你正在编写一个新的，示例是你的起始模板。

| 项目类型 | 驱动程序形状 | 示例 |
|---|---|---|
| Web 服务器 / API | 后台启动 + 基于 `curl` 的冒烟脚本 | [examples/server.md](examples/server.md) |
| CLI 工具 | 代表性参数冒烟脚本，检查退出代码 + 输出 | [examples/cli.md](examples/cli.md) |
| TUI / 交互式终端 | tmux 包装器：`send-keys` / `capture-pane` | [examples/tui.md](examples/tui.md) |
| Electron / 桌面 GUI | xvfb 下的 Playwright `_electron` REPL 驱动程序，截图，tmux 包装 | [examples/electron.md](examples/electron.md) |
| 浏览器驱动 | 开发服务器 + `chromium-cli` 脚本 | [examples/playwright.md](examples/playwright.md) |
| 库 / SDK | 导入和调用冒烟脚本 | [examples/library.md](examples/library.md) |

对于 web 应用，从 [examples/playwright.md](examples/playwright.md) 开始 — 用 `chromium-cli` 驱动它，不需要自定义驱动程序。对于桌面应用，从 [examples/electron.md](examples/electron.md) 开始 — 它具有完整的 `_electron` REPL 驱动程序骨架、tmux 包装和你将遇到的障碍目录。

## 包含什么

- **先决条件** — 操作系统包、运行时、工具。Ubuntu `apt-get` 行。确切的那些。
- **设置** — 安装依赖、配置、任何修补。
- **构建** — 编译/打包。
- **运行（智能体路径）** — 驱动程序。命令。截图位置。
- **直接调用** — 如果可调用：如何导入和运行内部代码而无需完整应用。绕过初始化守卫的环境变量 / 标志。许多 PR 只需要这个。
- **运行（人类路径）** — 如果有意义地不同。
- **测试** — 测试套件命令。
- **Gotchas** — 你遇到的非明显陷阱。
- **故障排除** — 错误 → 修复。
- **驱动程序本身** — 提交在 skill 目录中（或升级到 `scripts/`/`e2e/`），或对于 `chromium-cli` web 应用内联在 `SKILL.md` 中；无论哪种方式都从 `SKILL.md` 引用。

## 省略什么

- **你没有运行的任何东西。**如果 README 说 `yarn start:prod` 而你从未运行它，它就不在 skill 中。完全停止。
- **你不在平台上的文档化快乐路径。**你在 Linux 容器中。你无法验证的仅 macOS 部分是推测。提及它存在；不要详细说明。
- **详尽的选项。**一条工作路径。
- **架构文字。**那是其他文档。
- **通用故障排除。**"如果构建失败，检查你的 Node 版本" — 无用。只包括你实际遇到并修复的错误。

## 红旗 — 你即将交付错误的东西

如果出现以下情况，停止并重新考虑：

- **你没有截图** GUI 应用。你没有运行它。
- **你的 skill 没有驱动程序/冒烟脚本**可以指向，并且应用是交互式的。下一个智能体无法驱动它。（使用 `chromium-cli` 的 web 应用？ — `SKILL.md` 中的 heredoc 是驱动程序；不需要单独的文件。）
- **你的 skill 读起来像 README。**相同的结构，相同的命令，相同的警告。你改写了。
- **你的故障排除部分是通用的。**真正的执行产生特定的、奇怪的错误。通用错误 = 你没有执行。
- **你在没有尝试启动的情况下写了"不支持此平台"。**README 作者在 Mac 上。你不是。尝试。
- **一切都在第一次尝试中工作。**要么这个项目非常简单，要么你运行了测试套件并称之为完成。
