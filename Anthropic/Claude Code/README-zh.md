# Claude Code 系统提示词

对 Claude Code 发送给每个模型的系统提示词的逐字捕获，以及工具定义、agent/skill 列表和二进制文件中内置的技能。

捕获均为原始状态：无 MCP 服务器、无个人 `CLAUDE.md`、输出风格为 `default`、项目路径用占位符替换。

## 精简版与完整版系统提示词

Claude Code 提供两种系统提示词，并按模型选择其一。

| | 开篇章节 |
|---|---|
| **精简版** | `# Harness`、`# Communicating with the user` |
| **完整版** | `# System`、`# Doing tasks`、`# Executing actions with care`、`# Tone and style`、`# Text output` |

选择按模型固定，由二进制文件模型注册表中的 `lean_prompt` 能力标志驱动：

| 模型 | 提示词 | 文件 |
|---|---|---|
| Fable 5 | 精简版 | `claude-code-fable-5.md` |
| Opus 4.8 | 精简版 | `claude-code-opus-4.8.md` |
| Sonnet 5 | 完整版 | `claude-code-sonnet-5.md` |
| Opus 4.7 | 完整版 | `claude-code-opus-4.7.md` |
| Opus 4.6 | 完整版 | `claude-code-opus-4.6.md` |
| Sonnet 4.6 | 完整版 | `claude-code-sonnet-4.6.md` |
| Haiku 4.5 | 完整版 | `claude-code-haiku-4.5.md` |

精简版模型恰好是 **Opus 4.8、Fable 5 和 Mythos 5**，即携带 `lean_prompt` 能力标志的模型。这是模型级别的属性，而非按发布时间划分：**Sonnet 5 在 Opus 4.8 之后发布，但仍使用完整版提示词**，因为排除是按系列划分的（任何 Sonnet、任何 Haiku、Opus 4.7 及更早版本），而非按发布日期。

精简版/完整版的区分并非唯一的模型级差异：**Claude 5 系列 + Opus 4.8** 携带 `mid_conv_system` 能力，以尾部系统消息的形式接收 agent 类型和 skill 列表，而 **Claude 4.x 模型**（Opus 4.6/4.7、Sonnet 4.6、Haiku 4.5）以第一条用户消息中的 `<system-reminder>` 块接收这些列表。内容相同，注入渠道不同。

精简版提示词在 Claude Code **2.1.154**（2026-05-28，Opus 4.8 发布当天）引入，更新日志写道："精简版系统提示词现已成为除 Haiku、Sonnet 和 Opus 4.7 及更早版本之外所有模型的默认选择。"

## 注意事项

- `DesignSync` 的存在是因为捕获账号启用了 design-sync 功能。它是受控的（`isEnabled` → `tengu_slate_quill`）；没有此功能的常规安装不会有这个工具。

## 斜杠命令与内置技能

所有通过 `/name` 调用的命令都显示在同一个 `/help` 列表中，但这是两个不同的系统。关闭内置技能（`disableBundledSkills`）不会影响 `.claude/commands/`，这证明它们是独立的群体。

| | 内置（二进制中） | 用户/项目/插件（磁盘上） |
|---|---|---|
| 斜杠命令 | `type:local` / `local-jsx` / `prompt`（约 108 个） | `.claude/commands/*.md` |
| 技能 | `wu`/`Du` 注册（约 32 个） | `.claude/skills/*/SKILL.md` |

技能通过 Skill 工具由模型调用，携带 `description` + 可选的 `allowedTools` 和打包文件，受 `disableBundledSkills` 控制。普通命令（`/clear`、`/theme`、`/config`）仅由用户触发，没有面向模型的文本。这些界面已趋同，模型现在可以通过 Skill 工具调用一些内置命令（`/init`、`/review`、`/security-review`），少数条目（`init`、`review`）同时注册为 `type:"prompt"` 命令和技能。

面向模型的提示词文本通过三个渠道到达模型，各有其目录：

1. **技能** — `wu`/`Du` 注册 → `bundled-skills/`。
2. **斜杠命令提示词** — `type:"prompt"` 和携带提示词的 `local` 命令（`btw`、`compact`、`recap`、`insights`、`team-onboarding`、`session-title`/`/rename`）→ `slash-commands/`。
3. **`<system-reminder>` 注入** — 功能包装在 `<system-reminder>` 中并注入到对话回合中的提示词，不与任何命令绑定（`teammate`、`remote-plan`、`plan-multiagent`、`non-interactive`、`container-restart`、`model-switched`、`brief-mode`）→ `injected-reminders/`。

约 100 个 `local`/`local-jsx` UI 命令（`clear`、`theme`、`config`、`model`、`diff` 等）不携带面向模型的文本，未予捕获。

## 目录内容

- `claude-code-{fable-5,sonnet-5,opus-4.6,opus-4.8}.md` — 各模型的系统提示词（参见上文的精简版/完整版区分）。
- `bundled-skills/` — 编译进二进制的技能；二进制的 `disableBundledSkills` 设置恰好隐藏这一组。
- `slash-commands/` — `/` 调用命令背后的提示词。
- `injected-reminders/` — `<system-reminder>` 功能注入。
- `agents/` — 标准子代理提示词。
- `mcp-servers/` — 连接时 Claude Code 暴露的 MCP 服务器的逐服务器捕获（指令 + 工具定义）：`computer-use`、`claude-in-chrome` 以及 claude.ai 的 Google Workspace 连接器（`gmail`、`google-calendar`、`google-drive`）。在常规的无 MCP 会话中不存在。
- `grep-tool.md`、`glob-tool.md` — Grep/Glob 工具定义。**这些不在主代理工具集中**（截至 2.1.211，约 2026 年 4 月移除），仅搜索子代理（Explore 等）可用。主代理捕获有 28 个工具，其中不含这两个。
- `prompt-suggestion.md`、`claude-code-docs-assistant.md` — 其他单独捕获。
