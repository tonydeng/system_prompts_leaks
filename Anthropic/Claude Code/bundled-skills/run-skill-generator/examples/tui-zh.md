> **说明**：本文件为英文原文（`tui.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：TUI / 交互式终端应用

交互式终端应用（文本编辑器、REPL、基于 curses 的 UI）无法由智能体的 bash 工具直接驱动 — 它们接管终端。skill 必须展示如何将它们包装在 `tmux` 中，以便智能体可以发送输入、捕获输出并截图。

## tmux 模式

这是标准方法：

1. 在分离的 tmux 会话中启动 TUI
2. 用 `tmux send-keys` 发送击键
3. 用 `tmux capture-pane` 读取屏幕内容
4. 用 `tmux kill-session` 清理

skill 的 `SKILL.md` 应该将此呈现为驱动应用的主要方式。包装启动+附加序列的小 `driver.sh` 可以位于 skill 目录中，但对于大多数 TUI，skill 主体中的原始 tmux 命令就足够了。

## 示例片段

> ## 运行（交互式，用于智能体）
>
> 在 tmux 中启动 TUI：
>
> ```bash
> tmux new-session -d -s app -x 120 -y 40 './myapp'
> ```
>
> 轮询直到就绪标记出现（比固定睡眠更快 + 更可靠 — 应用启动时立即返回，如果未启动则大声失败）：
>
> ```bash
> timeout 10 bash -c 'until tmux capture-pane -t app -p | grep -q "Ready"; do sleep 0.2; done'
> tmux capture-pane -t app -p
> ```
>
> 发送输入（此示例导航到设置屏幕并切换选项）：
>
> ```bash
> tmux send-keys -t app 's'
> timeout 5 bash -c 'until tmux capture-pane -t app -p | grep -q "Settings"; do sleep 0.2; done'
> tmux send-keys -t app 'Down' 'Down' 'Space'  # 导航 + 切换
> timeout 5 bash -c 'until tmux capture-pane -t app -p | grep -qF "[x]"; do sleep 0.2; done'
> tmux capture-pane -t app -p
> ```
>
> 如果你发现自己编写了不止几个这样的轮询行，将它们拉到 skill 旁边的 `driver.sh` 中的 `wait_for()` 助手中。
>
> 退出：
>
> ```bash
> tmux send-keys -t app 'q'
> tmux kill-session -t app 2>/dev/null || true
> ```
>
> ### 键参考
>
> | 键 | 操作 |
> |---|---|
> | `j` / `k` 或 `Down` / `Up` | 导航列表 |
> | `Enter` | 选择 |
> | `s` | 设置 |
> | `q` | 退出 |

## 值得文档化的细节

- **终端大小。**某些 TUI 在小宽度处中断或隐藏内容。在 `tmux new-session -x -y` 参数中指定已知良好的大小。
- **启动时间。**轮询就绪标记（`until tmux capture-pane | grep -q X`）而不是固定的 `sleep N` — 应用启动时立即返回，如果从未启动则有用地失败。说明什么字符串意味着就绪。
- **键绑定参考。**主要键的表格。这是 TUI 的"API" — 智能体需要它来驱动应用。
- **干净退出。**显示退出击键*和* `tmux kill-session` 作为后备。
- **颜色/unicode 怪癖。**如果 `capture-pane` 输出难以阅读，注意有帮助的标志（`-e` 用于转义序列，`-J` 连接换行）。

## 还要文档化直接调用

对于交互式运行应用的人类，tmux 是过度杀伤。也包括单行命令：

> ## 运行（直接，用于人类）
>
> ```bash
> ./myapp
> ```
>
> 按 `q` 退出。
