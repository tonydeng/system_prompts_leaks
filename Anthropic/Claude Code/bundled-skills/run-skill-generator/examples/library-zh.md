> **说明**：本文件为英文原文（`library.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：库 / SDK

库在流程意义上没有"运行"步骤 — 没有服务器要启动，没有 CLI 要调用。对于库，run skill 是关于：

1. 从源代码**构建**库
2. **运行测试套件**
3. **最小工作示例**，它练习库并证明它已正确安装

保持简短。模板的构建和测试部分完成大部分工作。

## 冒烟测试示例

主要的库特定添加是一个小程序（或 REPL 片段），它导入库并做一件真实的事情。这就是智能体确认"是的，库可用"的方式：

> ## 验证
>
> ```bash
> python -c '
> from mylib import Client
> c = Client()
> print(c.ping())
> '
> # → pong
> ```

或者对于编译语言：

> ```bash
> cat > /tmp/smoke.go <<GO
> package main
> import "example.com/mylib"
> func main() { println(mylib.Version()) }
> GO
> go run /tmp/smoke.go
> # → v1.2.3
> ```

## 示例片段

> ---
> name: run-mylib
> description: Build, install, and test mylib from source. Use when asked to verify mylib works, run its tests, or build a distribution.
> ---
>
> `mylib` 是一个 Python 库 — "运行"它意味着从源代码构建并执行测试套件。
>
> ## 设置
>
> ```bash
> pip install -e '.[dev]'
> ```
>
> ## 验证
>
> ```bash
> python -c 'import mylib; print(mylib.__version__)'
> # → 2.1.0
> ```
>
> ## 测试
>
> ```bash
> pytest
> ```
>
> 测试子集：`pytest tests/unit/`。带覆盖率：`pytest --cov=mylib`。
>
> ## 构建（分发）
>
> ```bash
> pip install build
> python -m build
> # → dist/mylib-2.1.0-py3-none-any.whl
> ```

## 值得文档化的事情

- **开发模式与安装模式。** `pip install -e .` 与 `pip install .` — 如果行为不同，说明何时使用哪个。
- **可选依赖。** `[dev]`、`[test]`、`[docs]` extras 以及何时需要每个。
- **生成的代码。**如果有代码生成步骤（protobuf、OpenAPI 客户端），文档化它 — 它几乎总是从 README 中缺失。
