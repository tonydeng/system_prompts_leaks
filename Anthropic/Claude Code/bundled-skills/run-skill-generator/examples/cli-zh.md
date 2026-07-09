> **说明**：本文件为英文原文（`cli.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：CLI 工具

CLI 是最简单的情况 — 通常没有后台进程要管理，没有端口，没有生命周期。skill 专注于**安装**、**代表性调用**和**测试**。

## 重要的是什么

- **如何将二进制文件放在 `PATH` 上。**全局安装？通过 `npx`/`uv run` 运行？构建到 `./target/release/foo`？要明确。
- **两个或三个示例调用**，涵盖主要用例。包括预期输出，以便读者可以判断它是否工作。
- **退出代码**如果它们有意义（例如 linter 在发现问题时返回 1）。
- **Stdin 行为**如果工具从 stdin 读取。

## 示例片段

> ---
> name: run-mytool
> description: Build, install, and run mytool. Use when asked to run mytool, test it, or verify it's installed correctly.
> ---
>
> ## 设置
>
> ```bash
> pip install -e .
> ```
>
> 这将 `mytool` 放在 PATH 上。验证：
>
> ```bash
> mytool --version
> # → mytool 0.3.1
> ```
>
> ## 运行
>
> 处理单个文件：
>
> ```bash
> mytool process input.json
> # → Processed 42 records, wrote output.json
> ```
>
> 从 stdin 读取，写入 stdout：
>
> ```bash
> cat input.json | mytool process -
> ```
>
> Lint 目录（有问题时退出非零）：
>
> ```bash
> mytool lint ./src
> echo $?  # 0 如果干净，1 如果发现问题
> ```
>
> ## 测试
>
> ```bash
> pytest
> ```

## 保持简短

CLI 的 run skill 可以非常紧凑。不要用每个标志填充它 — `--help` 输出涵盖那个。只显示足够的内容，以便智能体可以（a）构建它，（b）确认它工作，（c）运行测试。
