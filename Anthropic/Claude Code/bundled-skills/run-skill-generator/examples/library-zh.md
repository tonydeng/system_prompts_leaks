> **说明**：本文件为英文原文（`library.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# 示例：库 / SDK

库没有进程意义上的"运行"步骤——没有服务器要启动，没有 CLI 要调用。对库而言，运行技能关注：

1. 从源码**构建**库
2. **运行测试套件**
3. 一个练习库并证明其已正确安装的**最小工作示例**

保持简短。模板的 Build 和 Test 部分承担大部分工作。

## 冒烟测试示例

主要的库特定补充是一个小程序（或 REPL 片段），导入库并做一件真实的事。这是智能体确认"是的，库可用"的方式：

> ## Verify
>
> ```bash
> python -c '
> from mylib import Client
> c = Client()
> print(c.ping())
> '
> # → pong
> ```

或对于编译型语言：

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
> `mylib` 是一个 Python 库——"运行"它意味着从源码构建并执行测试套件。
>
> ## Setup
>
> ```bash
> pip install -e '.[dev]'
> ```
>
> ## Verify
>
> ```bash
> python -c 'import mylib; print(mylib.__version__)'
> # → 2.1.0
> ```
>
> ## Test
>
> ```bash
> pytest
> ```
>
> Subset of tests: `pytest tests/unit/`. With coverage: `pytest --cov=mylib`.
>
> ## Build (distribution)
>
> ```bash
> pip install build
> python -m build
> # → dist/mylib-2.1.0-py3-none-any.whl
> ```

## 需要考虑记录的事项

- **开发模式 vs 安装模式。** `pip install -e .` vs `pip install .`——若行为不同，说明各自用途。
- **可选依赖。** `[dev]`、`[test]`、`[docs]` extras 及各自何时需要。
- **生成代码。** 若有代码生成步骤（protobuf、OpenAPI 客户端），记录它——它几乎总是从 README 中缺失。
