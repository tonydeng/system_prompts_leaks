# 捆绑技能

编译到 Claude Code 二进制中的技能——`disableBundledSkills` 设置（及 `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS`）隐藏的集合。"Bundled"是 Anthropic 自己的术语。从二进制中逐字提取；捕获反映 Claude Code 2.1.211。

## 布局约定

技能在随附伴随文件时存储为**带 `SKILL.md` 的文件夹**，在无伴随的单文件时存储为**扁平的 `<name>.md`**。

| 文件夹（SKILL.md + 伴随文件） | 随附 |
|---|---|
| `claude-api/` | 按语言的 SDK 文档 + `shared/` 参考 |
| `dataviz/` | `references/` + 调色板验证器 `scripts/` |
| `design-sync/` | `lib/` + `scripts/` + storybook/non-storybook 变体 |
| `code-review/` | 按努力层级的提示词 + `report-findings-tool.md` |
| `run-skill-generator/` | `examples/` + `template.md` |
| `claude-code-docs/` | `references/` |
| `deep-research/` | 一个工作流 `scripts/` 文件 |

其他一切都是单文件技能（`verify.md`、`simplify.md`、`loop.md`、`batch.md`、`update-config.md`、`morning.md`、……）。

在 Claude Code 自身中，每个技能都是带 `SKILL.md` 的目录（`~/.claude/skills/<name>/SKILL.md` 是磁盘上的规范）。单文件技能在此被扁平化为 `<name>.md` 以提高可读性；内容相同。

`artifacts/` 是"文件夹 = 一个技能"的唯一例外：它是 Artifact 工具家族（`artifact-design`、`artifact-dashboard`、`artifact-data-table`、`artifact-explainer`、`artifact-report`、`plan-artifact` 和 `artifact-capabilities/`）的**分组容器**，不是单个技能。

## 并非所有这些都已启用

若干技能随附但被门控关闭（在 Statsig 实验或环境标志之后），因此它们不出现在标准安装的 `/help` 中——如 `design-sync`、`claude-code-docs`、`plan-artifact`、`artifact-*` 模板、`code-walkthrough`、`pr-explainer`、`morning`。它们在此被捕获是因为无论门控状态如何，它们都被编译到二进制中。标准账户在 `/help` 中看到的 14 个是用户可调用、未门控的子集。
