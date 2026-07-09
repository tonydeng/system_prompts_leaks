> **说明**：本文件为英文原文（`template.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: run-<unit-name>
description: Build, run, and drive <unit-name>. Use when asked to start <unit-name>, run its tests, build it, take a screenshot of its UI, or interact with the running app.
---

<一句话描述：这是什么以及智能体如何驱动它。在这里命名句柄 — 对于桌面应用是"通过 xvfb 下的 `.claude/skills/run-<unit-name>/driver.mjs` 驱动它"，对于 web 应用是"启动开发服务器然后通过 `chromium-cli` 驱动它" — 这样智能体知道首先在哪里查看。>

<如果单元不在仓库根目录：>
下面的所有路径都相对于 `<unit-dir>/`。

## 先决条件

<系统级要求。你运行的确切 `apt-get install` 行 — 不是通用列表，是实际工作的那个。针对 Ubuntu。>

```bash
sudo apt-get update
sudo apt-get install -y <packages-you-actually-installed>
```

<如果运行时版本很重要：>

```bash
# 示例：通过 nvm 的 Node 20，通过 uv 的 Python 3.12，等等。
```

## 设置

<克隆后的一次性设置：安装依赖、配置、应用任何修补（功能门覆盖、配置存根）以及确切的命令。>

```bash
<commands>
```

<环境变量 — 必需与可选，带有合理的默认值：>

```bash
export FOO_API_KEY=...   # 必需 — 从 <where> 获取
export BAR_MODE=dev      # 可选 — 默认是 prod
```

## 构建

<如果没有单独的构建步骤则跳过。否则确切的命令：>

```bash
<command>
```

## 运行（智能体路径）

<这是未来的智能体实际使用的部分。如果你构建了驱动程序/REPL/冒烟脚本，这将文档化如何启动它以及它做什么。如果应用足够简单，`curl` 或单行命令就足够了，那么单行命令就在这里。>

```bash
<launch-the-driver-or-smoke-script>
```

<对于 REPL 风格的驱动程序，显示 tmux 包装。在 send-keys 和 capture-pane 之间轮询就绪标记 — 比固定睡眠更快，并且如果未就绪会大声失败，而不是捕获半渲染的屏幕：>

```bash
tmux new-session -d -s app -x 200 -y 50
tmux send-keys -t app '<launch command>' Enter
timeout 30 bash -c 'until tmux capture-pane -t app -p | grep -q "<ready-marker>"; do sleep 0.2; done'
tmux send-keys -t app '<first driver command>' Enter
tmux capture-pane -t app -p
```

<工件落在哪里（截图、日志）— 绝对路径：>

截图 → `/tmp/shots/`。日志 → `/tmp/<app>.log`。

<如果驱动程序有命令，一个表格：>

| 命令 | 它做什么 |
|---|---|
| `<cmd>` | <描述> |

## 运行（人类路径）

<如果与智能体路径有意义地不同。简短 — 智能体不会使用这个，人类可以弄清楚。>

```bash
<command>   # → <发生什么>。<如何停止>。
```

## 测试

```bash
<command>
```

<预期结果 — "N 个套件通过"，或特定的已知不稳定测试。>

---

<下面的可选部分 — 仅在相关时包含，并且仅包含你实际遇到的内容，而不是通用建议。>

## Gotchas

<非明显的陷阱。看起来应该工作但不工作的东西，以及解决方法。如果此部分是通用的，删除它。>

- **<特定事物>** — <为什么它中断> → <改为做什么>

## 故障排除

<症状 → 修复。只有你实际遇到的错误。>

- **<确切错误消息或症状>**：<原因>。<修复>。

<!---

关于上述 FRONTMATTER 的说明：
- 在 `name:` 和 `description:` 中替换 <unit-name>。`name:` 成为斜杠命令（/run-<unit-name>）并且必须与目录名称匹配。
- `description:` 是 Claude 扫描以决定是否自动加载此 skill 的内容。保留动词 — "start"、"run"、"build"、"test"、"screenshot" — 它们是询问的智能体实际会输入的内容。

关于驱动程序的说明：
- 如果你编写了驱动程序脚本，它默认位于同一目录中（此文件旁边）。从运行部分引用它。
- 对于 web 应用，通常没有驱动程序文件 — 运行部分中的 `chromium-cli` heredoc 就是工具。
- 如果驱动程序成长为项目测试套件想要的东西 — 共享启动助手、真正的 e2e 工具 — 将其移动到单元中的 scripts/ 或 e2e/，并更新此处的路径。skill 保持不变。

在提交之前删除从上面的 `---` 开始的所有内容。--->
