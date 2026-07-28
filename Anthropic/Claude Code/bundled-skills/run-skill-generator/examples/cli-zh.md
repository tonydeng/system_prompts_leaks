> **说明**：本文件为英文原文（`cli.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：CLI 工具

CLI 是最简单的情况——通常没有后台进程要管理、没有端口、没有生命周期。技能聚焦于**安装**、**代表性调用**和**测试**。

## 什么重要

- **如何将二进制放到 `PATH` 上。** 全局安装？通过 `npx`/`uv run` 运行？构建到 `./target/release/foo`？要明确。
- **两三个覆盖主要用例的示例调用。** 包含预期输出，让读者知道它工作了。
- **退出代码**（若有意意义，如 linter 在发现时返回 1）。
- **Stdin 行为**（若工具从 stdin 读取）。

## 示例片段

> ---
> name: run-mytool
> description: Build, install, and run mytool. Use when asked to run mytool, test it, or verify it's installed correctly.
> ---
>
> ## Setup
>
> ```bash
> pip install -e .
> ```
>
> This puts `mytool` on PATH. Verify:
>
> ```bash
> mytool --version
> # → mytool 0.3.1
> ```
>
> ## Run
>
> Process a single file:
>
> ```bash
> mytool process input.json
> # → Processed 42 records, wrote output.json
> ```
>
> Read from stdin, write to stdout:
>
> ```bash
> cat input.json | mytool process -
> ```
>
> Lint a directory (exits non-zero on problems):
>
> ```bash
> mytool lint ./src
> echo $?  # 0 if clean, 1 if issues found
> ```
>
> ## Test
>
> ```bash
> pytest
> ```

## 保持简短

CLI 的运行技能可以非常紧凑。不要用每个标志填充——`--help` 输出已覆盖那些。只展示足够让智能体 (a) 构建它、(b) 确认它工作、(c) 运行测试的内容。
