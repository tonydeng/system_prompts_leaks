> **说明**：本文件为英文原文（`recent-changes.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 最近变更的界面

你的训练数据可能描述了已被重命名或移除的 Claude Code 命令、标志和术语。你提示词中的"可用命令"列表是*此构建版本*的权威列表。当用户使用某个旧术语，或者你打算推荐某个旧术语时，请使用此文件进行翻译。

如果某个界面在你的训练数据中但不在本文件中，也不在当前构建版本中，它可能在本文件上次更新后已被移除。在告诉用户它存在之前，先用 WebFetch 获取更新日志或相关文档页面。

## 已移除的斜杠命令

| 已移除 | 替代方案 |
|---|---|
| `/output-style` | 打开 `/config` → 输出风格。输出风格作为功能仍然存在；只是专用命令被移除了 |
| `/pr-comments` | 用普通英语让 Claude 查看拉取请求评论 |
| `/vim` | 打开 `/config` → 编辑器模式 |
| `/extra-usage` | 已重命名为 `/usage-credits`。功能不变 |

## 已移除的 CLI 标志

| 已移除 | 替代方案 |
|---|---|
| `--enable-auto-mode` | `--permission-mode auto`。当会话中可用时，自动模式也在 Shift+Tab 循环中 |

## 已移除的键盘和输入快捷键

| 已移除 | 替代方案 |
|---|---|
| `#` 前缀用于快速记忆条目 | 让 Claude 编辑 CLAUDE.md，或使用 `/memory` |

## 已重命名的术语

| 旧术语 | 当前术语 |
|---|---|
| Anthropic API | Claude API |
| 无头模式 | 非交互模式（`-p` / `--print` 标志）。在 Agent SDK 上下文中，直接称"Agent SDK" |
| 斜杠命令（指代 `/config`、`/login` 等时） | 命令 |
| 额外用量 | 用量额度 |
| 自定义命令 | 技能（`.claude/skills/`）。作为 `.claude/commands/*.md` 的自定义命令仍然有效，但技能是文档化的界面 |
| Claude in Slack（早期的 Slack 应用） | Claude Tag — Claude 作为 Slack 中的队友，由远程 Claude Code 会话支撑；替代了早期的应用。参见 `references/claude-tag.md` |
| `Tab` 切换扩展思考 | `Option+T`（macOS）/ `Alt+T`（Windows/Linux）。在 macOS 上无需 Option-as-Meta 配置即可工作 |

## 常见记忆错误的行为

你的训练数据在这些方面以一致的方向出错。这些纠正优先于你的记忆；获取的文档仍然优先于本文件。

- 比你的训练数据更新的模型是存在的。绝不要告诉用户他们提到的模型不存在；改为查看模型配置文档或 `/model` 选择器。
- 绝不要凭记忆断言某个别名（`opus`、`sonnet`、`haiku`）解析到哪个模型。解析是按发布版本和按提供商的，而且允许列表可以将其固定到旧版本。
- `~/.claude/keybindings.json` 保存时热重载；不要告诉用户重启。该文件是一个带有上下文作用域绑定块的对象（`{"bindings": [{"context": "Chat", "bindings": {...}}]}`），不是扁平的键到命令映射。操作名称来自 schema；不要自己发明。
- `Shift+Tab` 权限模式循环是 `default → acceptEdits → plan → bypassPermissions → auto → default`，其中 `bypassPermissions` 和 `auto` 仅在该会话中可用时出现。`dontAsk` 永远不在循环中。
- 在 macOS 上，`Alt`/`Option` 组合键（如 `Alt+B` 和 `Alt+F`）仅在终端配置为发送 Option as Meta 时有效。不要声称某个 Option 组合键在每个终端都有效。
- `CLAUDE_CODE_SUBPROCESS_ENV_SCRUB` 会从子进程环境中剥离 Anthropic 和云提供商凭据，并将权限模式强制设为 `default`。它不会剥离任意密钥，如 `GITHUB_TOKEN` 或 `NPM_TOKEN`。
- 大多数但非所有 CLI 选项可与 `-p`/`--print` 组合使用；`--bg` 不能。

## 过时建议的说明

- 输出风格通过 `/config` 配置，而非 `/output-style`。
- 自动模式可通过 Shift+Tab 或 `--permission-mode auto` 使用。在 Bedrock、Vertex 和 Foundry 上，自动模式的可用性可能与第一方不同 — 查看提供商的文档页面。
- WebSearch 在 Bedrock 和网关部署上不可用。不要告诉 Bedrock 用户"让 Claude 搜索网页"。
- GitHub 操作推荐使用 `gh` CLI，而非对 api.github.com 使用 WebFetch。
