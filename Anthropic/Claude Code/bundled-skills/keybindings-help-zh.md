> **说明**：本文件为英文原文（`keybindings-help.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: keybindings-help
description: 自定义键盘快捷键、重新绑定键、添加和弦绑定或修改 ~/.claude/keybindings.json。
---

# 键盘绑定技能

创建或修改 `~/.claude/keybindings.json` 以自定义键盘快捷键。

## 关键：写入前先读取

**始终先读取 `~/.claude/keybindings.json`**（它可能还不存在）。将更改与现有绑定合并——永远不要替换整个文件。

- 使用 **Edit** 工具修改现有文件
- 仅当文件尚不存在时使用 **Write** 工具

## 文件格式

```json
{
  "$schema": "https://www.schemastore.org/claude-code-keybindings.json",
  "$docs": "https://code.claude.com/docs/en/keybindings",
  "bindings": [
    {
      "context": "Chat",
      "bindings": {
        "ctrl+e": "chat:externalEditor"
      }
    }
  ]
}
```

始终包括 `$schema` 和 `$docs` 字段。

## 击键语法

**修饰符**（用 `+` 组合）：
- `ctrl`（别名：`control`）
- `alt`（别名：`opt`、`option`）——注意：`alt` 和 `meta` 在终端中是相同的
- `shift`
- `meta`（别名：`cmd`、`command`）

**特殊键**：`escape`/`esc`、`enter`/`return`、`tab`、`space`、`backspace`、`delete`、`up`、`down`、`left`、`right`

**和弦**：空格分隔的击键，例如 `ctrl+k ctrl+s`（击键之间 1 秒超时）

**示例**：`ctrl+shift+p`、`alt+enter`、`ctrl+k ctrl+n`

## 解除默认快捷键绑定

将键设置为 `null` 以删除其默认绑定：

```json
{
  "context": "Chat",
  "bindings": {
    "ctrl+s": null
  }
}
```

## 用户绑定如何与默认值交互

- 用户绑定是**附加的**——它们附加在默认绑定之后
- 要将绑定**移动**到不同的键：解除旧键的绑定（`null`）并添加新绑定
- 仅当用户想要更改该上下文中的某些内容时，上下文才需要出现在用户的文件中

## 常见模式

### 重新绑定键
要将外部编辑器快捷键从 `ctrl+g` 更改为 `ctrl+e`：
```json
{
  "context": "Chat",
  "bindings": {
    "ctrl+g": null,
    "ctrl+e": "chat:externalEditor"
  }
}
```

### 添加和弦绑定
```json
{
  "context": "Global",
  "bindings": {
    "ctrl+k ctrl+t": "app:toggleTodos"
  }
}
```

## 行为规则

1. 仅包括用户想要更改的上下文（最小覆盖）
2. 验证操作和上下文来自下面的已知列表
3. 如果用户选择与保留快捷键或常见工具（如 tmux (`ctrl+b`) 和 screen (`ctrl+a`)）冲突的键，主动警告用户
4. 为现有操作添加新绑定时，新绑定是附加的（现有默认值仍然有效，除非明确解除绑定）
5. 要完全替换默认绑定，解除旧键的绑定并添加新键

## 使用 /doctor 验证

`/doctor` 命令包括一个"键盘绑定配置问题"部分，用于验证 `~/.claude/keybindings.json`。

### 常见问题和修复

| 问题 | 原因 | 修复 |
| --- | --- | --- |
| `keybindings.json 必须有一个 "bindings" 数组` | 缺少包装对象 | 将绑定包装在 `{ "bindings": [...] }` 中 |
| `"bindings" 必须是一个数组` | `bindings` 不是数组 | 将 `"bindings"` 设置为数组：`[{ context: ..., bindings: ... }]` |
| `未知上下文 "X"` | 拼写错误或无效的上下文名称 | 使用可用上下文表中的确切上下文名称 |
| `Y 绑定中的重复键 "X"` | 在一个上下文中两次定义相同的键 | 删除重复项；JSON 仅使用最后一个值 |
| `"X" 可能不起作用：...` | 键与终端/OS 保留快捷键冲突 | 选择不同的键（参见保留快捷键部分） |
| `无法解析击键 "X"` | 无效的键语法 | 检查语法：在修饰符之间使用 `+`，有效的键名称 |
| `"X" 的无效操作` | 操作值不是字符串或 null | 操作必须是字符串，如 `"app:help"` 或 `null` 以解除绑定 |

### 示例 /doctor 输出

```
键盘绑定配置问题
位置：~/.claude/keybindings.json
  └ [错误] 未知上下文 "chat"
    → 有效上下文：Global、Chat、Autocomplete、...
  └ [警告] "ctrl+c" 可能不起作用：终端中断 (SIGINT)
```

**错误**阻止绑定工作，必须修复。**警告**指示潜在冲突，但绑定可能仍然有效。

## 保留快捷键

### 不可重新绑定（错误）
- `ctrl+c` — 无法重新绑定 - 用于中断/退出（硬编码）
- `ctrl+d` — 无法重新绑定 - 用于退出（硬编码）
- `ctrl+m` — 无法重新绑定 - 在终端中与 Enter 相同（两者都发送 CR）
- `capslock` — Caps Lock 不会传递到终端应用程序

### 终端保留（错误/警告）
- `ctrl+z` — Unix 进程挂起 (SIGTSTP)（可能冲突）
- `ctrl+\` — 终端退出信号 (SIGQUIT)（将不起作用）

### macOS 保留（错误）
- `cmd+c` — macOS 系统复制
- `cmd+v` — macOS 系统粘贴
- `cmd+x` — macOS 系统剪切
- `cmd+q` — macOS 退出应用程序
- `cmd+w` — macOS 关闭窗口/标签页
- `cmd+tab` — macOS 应用程序切换器
- `cmd+space` — macOS Spotlight

## 可用上下文

| 上下文 | 描述 |
| --- | --- |
| `Global` | 在任何地方都活动，无论焦点如何 |
| `Chat` | 当聊天输入聚焦时 |
| `Autocomplete` | 当自动完成菜单可见时 |
| `Confirmation` | 当显示确认/权限对话框时 |
| `Help` | 当帮助覆盖打开时 |
| `Transcript` | 当查看转录时 |
| `HistorySearch` | 当搜索命令历史时 (ctrl+r) |
| `Task` | 当任务/代理在前台运行时 |
| `ThemePicker` | 当主题选择器打开时 |
| `Settings` | 当设置菜单打开时 |
| `Tabs` | 当标签页导航活动时 |
| `Attachments` | 当在选择对话框中导航图像附件时 |
| `Footer` | 当页脚指示器聚焦时 |
| `MessageSelector` | 当消息选择器（倒带）打开时 |
| `DiffDialog` | 当差异对话框打开时 |
| `ModelPicker` | 当模型选择器打开时 |
| `Select` | 当选择/列表组件聚焦时 |
| `Plugin` | 当插件对话框打开时 |
| `Scroll` | 当可滚动视图聚焦时（全屏布局） |
| `Doctor` | 当 /doctor 诊断屏幕打开时 |

## 可用操作

| 操作 | 默认键 | 上下文 |
| --- | --- | --- |
| `app:interrupt` | `ctrl+c` | Global |
| `app:exit` | `ctrl+d` | Global |
| `app:toggleTodos` | `ctrl+t` | Global |
| `app:toggleTranscript` | `ctrl+o` | Global |
| `app:toggleBrief` | `ctrl+shift+b` | Global |
| `app:toggleTeammatePreview` | `ctrl+shift+o` | Global |
| `app:toggleTerminal` | (无) | Global |
| `app:redraw` | (无) | Global |
| `app:openFrame` | (无) | Global |
| `history:search` | `ctrl+r` | Global |
| `history:previous` | `up` | Chat |
| `history:next` | `down` | Chat |
| `chat:cancel` | `escape` | Chat |
| `chat:killAgents` | `ctrl+x ctrl+k` | Chat |
| `chat:cycleMode` | `shift+tab` | Chat |
| `chat:modelPicker` | `meta+p` | Chat |
| `chat:fastMode` | `meta+o` | Chat |
| `chat:thinkingToggle` | `meta+t` | Chat |
| `chat:workflowKeywordToggle` | `meta+w` | Chat |
| `chat:submit` | `enter` | Chat |
| `chat:newline` | `ctrl+j` | Chat |
| `chat:undo` | `ctrl+_`、`ctrl+-`、`ctrl+shift+-`、`ctrl+shift+_` | Chat |
| `chat:externalEditor` | `ctrl+x ctrl+e`、`ctrl+g` | Chat |
| `chat:stash` | `ctrl+s` | Chat |
| `chat:imagePaste` | `ctrl+v` | Chat |
| `chat:clearInput` | `ctrl+l` | Chat |
| `chat:clearScreen` | `cmd+k` | Chat |
| `autocomplete:accept` | `tab` | Autocomplete |
| `autocomplete:dismiss` | `escape` | Autocomplete |
| `autocomplete:previous` | `up` | Autocomplete |
| `autocomplete:next` | `down` | Autocomplete |
| `confirm:yes` | `y`、`enter` | Confirmation |
| `confirm:no` | `escape`、`n`、`escape` | Settings |
| `confirm:previous` | `up` | Confirmation |
| `confirm:next` | `down` | Confirmation |
| `confirm:nextField` | `tab` | Confirmation |
| `confirm:previousField` | (无) | Confirmation |
| `confirm:cycleMode` | `shift+tab` | Confirmation |
| `confirm:toggle` | `space` | Confirmation |
| `confirm:toggleExplanation` | `ctrl+e` | Confirmation |
| `tabs:next` | `tab`、`right` | Tabs |
| `tabs:previous` | `shift+tab`、`left` | Tabs |
| `transcript:toggleShowAll` | `ctrl+e` | Transcript |
| `transcript:exit` | `ctrl+c`、`escape`、`q` | Transcript |
| `historySearch:next` | `ctrl+r` | HistorySearch |
| `historySearch:accept` | `escape`、`tab` | HistorySearch |
| `historySearch:cancel` | `ctrl+c` | HistorySearch |
| `historySearch:execute` | `enter` | HistorySearch |
| `historySearch:cycleScope` | `ctrl+s` | HistorySearch |
| `task:background` | `ctrl+b` | Task |
| `theme:toggleSyntaxHighlighting` | `ctrl+t` | ThemePicker |
| `theme:editCustom` | `ctrl+e` | ThemePicker |
| `help:dismiss` | `escape` | Help |
| `attachments:next` | `right` | Attachments |
| `attachments:previous` | `left` | Attachments |
| `attachments:remove` | `backspace`、`delete` | Attachments |
| `attachments:exit` | `down`、`escape` | Attachments |
| `footer:up` | `up`、`ctrl+p` | Footer |
| `footer:down` | `down`、`ctrl+n` | Footer |
| `footer:next` | `right` | Footer |
| `footer:previous` | `left` | Footer |
| `footer:openSelected` | `enter` | Footer |
| `footer:clearSelection` | `escape` | Footer |
| `footer:close` | `x` | Footer |
| `messageSelector:up` | `up`、`k`、`ctrl+p` | MessageSelector |
| `messageSelector:down` | `down`、`j`、`ctrl+n` | MessageSelector |
| `messageSelector:top` | `ctrl+up`、`shift+up`、`meta+up`、`shift+k` | MessageSelector |
| `messageSelector:bottom` | `ctrl+down`、`shift+down`、`meta+down`、`shift+j` | MessageSelector |
| `messageSelector:select` | `enter` | MessageSelector |
| `diff:dismiss` | `escape` | DiffDialog |
| `diff:previousSource` | `left` | DiffDialog |
| `diff:nextSource` | `right` | DiffDialog |
| `diff:back` | (无) | DiffDialog |
| `diff:viewDetails` | `enter` | DiffDialog |
| `diff:previousFile` | `up`、`k` | DiffDialog |
| `diff:nextFile` | `down`、`j` | DiffDialog |
| `modelPicker:decreaseEffort` | `left` | ModelPicker |
| `modelPicker:increaseEffort` | `right` | ModelPicker |
| `modelPicker:thisSessionOnly` | `s` | ModelPicker |
| `select:next` | `down`、`j`、`ctrl+n`、`down`、`j`、`ctrl+n` | Settings |
| `select:previous` | `up`、`k`、`ctrl+p`、`up`、`k`、`ctrl+p` | Settings |
| `select:pageUp` | `pageup` | Select |
| `select:pageDown` | `pagedown` | Select |
| `select:first` | `home` | Select |
| `select:last` | `end` | Select |
| `select:accept` | `space`、`enter` | Settings |
| `select:cancel` | `escape` | Select |
| `plugin:toggle` | `space` | Plugin |
| `plugin:install` | `i` | Plugin |
| `plugin:favorite` | `f` | Plugin |
| `doctor:fix` | `f` | Doctor |
| `permission:toggleDebug` | (无) | Confirmation |
| `settings:search` | `/` | Settings |
| `settings:retry` | `r` | Settings |
| `settings:close` | `enter` | Settings |
| `settings:periodDay` | `d` | Settings |
| `settings:periodWeek` | `w` | Settings |
| `settings:sortByTokens` | `t` | Settings |
| `voice:pushToTalk` | `space` | Chat |
| `scroll:pageUp` | `pageup`、`pageup` | Scroll |
| `scroll:pageDown` | `pagedown`、`pagedown` | Scroll |
| `scroll:lineUp` | `ctrl+p`、`k`、`up`、`wheelup` | Transcript |
| `scroll:lineDown` | `ctrl+n`、`j`、`down`、`wheeldown` | Transcript |
| `scroll:top` | `g`、`home`、`ctrl+home`、`g`、`home` | Transcript |
| `scroll:bottom` | `shift+g`、`end`、`ctrl+end`、`shift+g`、`end` | Transcript |
| `scroll:halfPageUp` | `ctrl+u`、`ctrl+u` | Settings |
| `scroll:halfPageDown` | `ctrl+d`、`ctrl+d` | Settings |
| `scroll:fullPageUp` | `ctrl+b`、`b`、`shift+space`、`b` | Transcript |
| `scroll:fullPageDown` | `ctrl+f`、`space`、`space` | Transcript |
| `selection:copy` | `ctrl+shift+c`、`cmd+c` | Scroll |
| `selection:clear` | (无) | Unknown |
| `selection:extendLeft` | `shift+left` | Scroll |
| `selection:extendRight` | `shift+right` | Scroll |
| `selection:extendUp` | `shift+up` | Scroll |
| `selection:extendDown` | `shift+down` | Scroll |
| `selection:extendLineStart` | `shift+home` | Scroll |
| `selection:extendLineEnd` | `shift+end` | Scroll |