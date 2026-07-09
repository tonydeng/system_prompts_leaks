> **说明**：本文件为英文原文（`low.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

`low effort → 1 diff pass → no verify → ≤4 findings`

## 回合 1 — 读取

一次工具调用：读取统一 diff（`git diff @{upstream}...HEAD; git diff HEAD` 以覆盖已提交和未提交的更改，或 `git diff main...HEAD` / 作为参数传递的目标）。跳过测试/fixture hunk（`test/`、`spec/`、`__tests__/`、`*_test.*`、`*.test.*`、`fixtures/`、`testdata/`）——在此级别不审查测试文件更改。无子代理，无完整文件读取。

## 回合 2 — 发现

标记仅从 hunk 可见的运行时正确性 bug：倒置/错误的条件、差一错误、相邻行显示值可能为空的 null/undefined 解引用、被移除的守卫、假值零检查、缺失的 `await`、错误的变量复制粘贴、应该传播但在 catch 中被吞掉的错误。还要标记——仍然仅从 hunk——复制 diff 上下文中可见的现有辅助函数的新代码，以及 diff 遗留的死代码。

**不要**标记风格、命名、性能、缺失测试，或 hunk 之外的任何内容。

输出最多 **4 个发现**，最严重优先，每个一行：`path/to/file.ext:123 — what's wrong and the concrete failure`。如果没有符合条件的，准确输出 `(none)`。
