> **说明**：本文件为英文原文（`live-sources.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 实时文档来源

用于获取当前 Claude Code 文档的 WebFetch URL。当打包的参考资料和提示词中的实时构建配置无法回答问题，或用户询问实时构建快照未覆盖的行为、内部机制或主题时，使用这些 URL。

Mintlify 为每个页面同时提供 `.md` 和 `.mdx`；优先使用 `.md` 以获取干净的内容。`.md` 格式仅用于获取：为用户链接页面时去掉末尾的 `.md` 使他们看到渲染后的页面。

## 从这里开始

| 主题 | URL | 提取提示 |
|---|---|---|
| 页面索引（所有页面 + 标题） | `https://code.claude.com/docs/en/claude_code_docs_map.md` | "查找涵盖 `<topic>` 的页面并返回其 URL" |
| 更新日志 | `https://code.claude.com/docs/en/changelog.md` | "提取自版本 `<X.Y.Z>` 以来的变更" |

## 配置

| 主题 | URL | 提取提示 |
|---|---|---|
| 设置参考 | `https://code.claude.com/docs/en/settings.md` | "提取 `<setting>` 的设置键、类型、作用域和默认值" |
| CLI 参考（标志） | `https://code.claude.com/docs/en/cli-reference.md` | "提取 `<flag>` 的标志、参数和功能" |
| 权限和规则 | `https://code.claude.com/docs/en/permissions.md` | "提取 `<tool>` 的权限规则语法和示例" |
| 记忆（CLAUDE.md） | `https://code.claude.com/docs/en/memory.md` | "提取如何使用和组织 CLAUDE.md" |
| `.claude/` 目录布局 | `https://code.claude.com/docs/en/claude-directory.md` | "提取 .claude 目录中各文件的用途" |
| 环境变量 | `https://code.claude.com/docs/en/env-vars.md` | "提取 `<variable>` 的环境变量名、类型和效果" |

## 可扩展性

| 主题 | URL | 提取提示 |
|---|---|---|
| Hooks | `https://code.claude.com/docs/en/hooks.md` | "提取 `<hook event>` 的钩子事件名、JSON schema 和配置" |
| Skills | `https://code.claude.com/docs/en/skills.md` | "提取如何创建和组织 skill" |
| Subagents | `https://code.claude.com/docs/en/sub-agents.md` | "提取如何定义和配置子代理" |
| MCP 服务器 | `https://code.claude.com/docs/en/mcp.md` | "提取如何添加、配置和认证 MCP 服务器" |
| 插件 | `https://code.claude.com/docs/en/plugins.md` | "提取如何安装和开发插件" |
| 输出样式 | `https://code.claude.com/docs/en/output-styles.md` | "提取如何创建和应用输出样式" |

## 工作流和界面

| 主题 | URL | 提取提示 |
|---|---|---|
| 命令参考 | `https://code.claude.com/docs/en/commands.md` | "提取 /`<command>` 的命令名、语法和描述" |
| 交互模式（快捷键） | `https://code.claude.com/docs/en/interactive-mode.md` | "提取 `<action>` 的键盘快捷键" |
| 常见工作流 | `https://code.claude.com/docs/en/common-workflows.md` | "提取 `<task>` 的工作流步骤" |
| GitHub Actions | `https://code.claude.com/docs/en/github-actions.md` | "提取如何在 GitHub Actions 中设置 Claude Code" |
| 网页版 Claude Code | `https://code.claude.com/docs/en/claude-code-on-the-web.md` | "提取远程会话如何工作及可配置内容" |
| VS Code 集成 | `https://code.claude.com/docs/en/vs-code.md` | "提取如何设置和使用 VS Code 扩展" |
| JetBrains 集成 | `https://code.claude.com/docs/en/jetbrains.md` | "提取如何设置和使用 JetBrains 插件" |

## 部署和安全

| 主题 | URL | 提取提示 |
|---|---|---|
| Amazon Bedrock | `https://code.claude.com/docs/en/amazon-bedrock.md` | "提取 Bedrock 上的设置、认证和功能差异" |
| Google Vertex AI | `https://code.claude.com/docs/en/google-vertex-ai.md` | "提取 Vertex 上的设置、认证和功能差异" |
| Microsoft Foundry | `https://code.claude.com/docs/en/microsoft-foundry.md` | "提取 Foundry 上的设置、认证和功能差异" |
| 沙箱 | `https://code.claude.com/docs/en/sandboxing.md` | "提取沙箱如何工作及如何配置" |
| 安全 | `https://code.claude.com/docs/en/security.md` | "提取安全模型和信任边界" |
| 网络配置 | `https://code.claude.com/docs/en/network-config.md` | "提取代理、防火墙和离线配置" |
| 费用和追踪 | `https://code.claude.com/docs/en/costs.md` | "提取费用如何计算及如何追踪" |

## Slack 中的 Claude（Claude Tag）

先读 `references/claude-tag.md`，它是此界面的离线参考基准。然后获取：

| 主题 | URL | 提取提示 |
|---|---|---|
| Claude Tag（Slack 中作为团队成员的 Claude，组织管理） | `https://claude.com/docs/claude-tag/overview.md` | "提取 Claude Tag 是什么、套餐可用性以及组织所有者如何启用和配置" |
| 所有 Claude Tag 页面（claude.com 文档域的索引） | `https://claude.com/docs/llms.txt` | "查找涵盖 `<topic>` 的 Claude Tag 页面并返回其 URL" |
| 组织所有者设置走查（配对 Slack、连接工具、支出限额、启动） | `https://claude.com/docs/claude-tag/admins/setup-overview.md` | "提取启用 Claude Tag 的设置步骤和前提条件" |
| 终端用户入门 | `https://claude.com/docs/claude-tag/users/getting-started.md` | "提取 Slack 用户如何开始使用 Claude Tag" |
| 从更早的 "Claude in Slack" 应用迁移 | `https://claude.com/docs/claude-tag/admins/migrate-from-earlier.md` | "提取从旧应用迁移到 Claude Tag 时工作区的变化" |

## Agent SDK

如需使用 Claude Agent SDK（Python 或 TypeScript）构建自定义代理，文档属于 Claude API 文档的一部分。获取 `https://platform.claude.com/llms.txt` 查找合适的页面，或使用 `/claude-api` 技能，该技能深入覆盖了 SDK。
