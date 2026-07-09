> **说明**：本文件为英文原文（`grep-tool.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# `Grep` 工具

一个强大的**内容搜索**工具，构建在 [ripgrep](https://github.com/BurntSushi/ripgrep)（`rg`）之上。它回答这个问题：*"哪些文件包含匹配此模式的文本，匹配行是什么？"*

Grep 搜索文件**内部**。（它的兄弟工具 `Glob` 搜索文件**名称**。见 `glob-tool.md`。）

---

## 何时使用它

- 查找函数、变量、类或字符串的定义或使用位置。
- 在代码库中定位模式的所有出现。
- 计算某事出现的次数。
- 任何时候你会在 shell 中到达 `grep`、`rg`、`egrep` 或 `grep -r`。

> **重要：**始终优先使用此工具而不是通过 Bash 工具运行 `grep`/`rg`。它是专门构建的，默认尊重 `.gitignore`，并以干净、结构化的形式返回结果。不鼓励在 Bash 中运行 `grep`。

---

## 参数

| 参数 | 类型 | 必需 | 描述 |
|-------------- | ------- | -------- | --------------------------------------------------------------------------------------------------- |
| `pattern` | string | **是** | 要搜索的正则表达式。支持完整的正则表达式语法（例如 `log.*Error`、`function\s+\w+`）。 |
| `path` | string | 否 | 要搜索的文件或目录。默认为当前工作目录。 |
| `glob` | string | 否 | 用于过滤搜索哪些文件的 glob 模式（例如 `*.js`、`*.{ts,tsx}`）。映射到 ripgrep 的 `--glob`。 |
| `type` | string | 否 | 要搜索的文件类型（例如 `js`、`py`、`rust`、`go`）。对于标准类型，通常比 `glob` 更有效。 |
| `output_mode` | string | 否 | `content`、`files_with_matches`（默认）或 `count` 之一。控制返回的内容（见下文）。 |
| `-i` | boolean | 否 | 不区分大小写的搜索。 |
| `-n` | boolean | 否 | 显示行号。仅当 `output_mode` 为 `content` 时适用。 |
| `-A` | number | 否 | 在每个匹配后显示的上下文行数。（仅 `content` 模式。） |
| `-B` | number | 否 | 在每个匹配前显示的上下文行数。（仅 `content` 模式。） |
| `-C` | number | 否 | 在每个匹配之前和之后显示的上下文行数。（仅 `content` 模式。） |
| `multiline` | boolean | 否 | 启用多行模式，以便 `.` 匹配换行符并且模式可以跨越行。默认 `false`。 |
| `head_limit` | number | 否 | 将输出限制为前 N 行/条目（像管道传输到 `head -N`）。适用于所有输出模式。 |

---

## 输出模式解释

`output_mode` 参数改变你得到的*内容*。选择正确的保持结果集中：

- **`files_with_matches`** *（默认）* — 仅返回包含至少一个匹配的文件路径列表。当你只需要知道某物*在哪里*时最佳。最便宜的输出。

- **`content`** — 返回实际的匹配行（以及通过 `-A`/`-B`/`-C` 的可选周围上下文）。这是支持 `-n`、`-A`、`-B` 和 `-C` 的模式。当你需要*读取*匹配而不仅仅是定位文件时使用它。

- **`count`** — 返回每个文件的匹配数。使用它在深入之前衡量模式的传播程度。

---

## 正则表达式说明

- `pattern` 是正则表达式，**不是**字面字符串。像 `.`、`(`、`)`、`{`、`[`、`*`、`+`、`?`、`|`、`\` 这样的字符具有特殊含义。
- 要字面匹配它们，用反斜杠转义。示例：要找到 Go 中的字面文本 `interface{}`，将模式写为 `interface\{\}`。
- Ripgrep 使用 Rust 正则表达式引擎。默认情况下，模式**逐行**匹配 — 除非设置 `multiline: true`，否则模式不能跨越多行。

---

## 示例

**查找每个提到 `TODO` 的文件（仅文件列表）：**
```json
{ "pattern": "TODO" }
```

**显示带有行号的匹配行，仅在 JavaScript 文件中：**
```json
{
  "pattern": "useState",
  "glob": "*.js",
  "output_mode": "content",
  "-n": true
}
```

**不区分大小写的搜索，带有 3 行周围上下文：**
```json
{
  "pattern": "deprecated",
  "output_mode": "content",
  "-i": true,
  "-C": 3
}
```

**计算 `console.log` 在每个文件中出现的频率，在 TypeScript 源中：**
```json
{
  "pattern": "console\\.log",
  "type": "ts",
  "output_mode": "count"
}
```

**多行搜索 — 跨越几行的 `struct` 块：**
```json
{
  "pattern": "struct \\{[\\s\\S]*?name",
  "multiline": true,
  "output_mode": "content"
}
```

---

## 提示和 gotchas

- **尽早过滤。**将 `pattern` 与 `glob` 或 `type` 结合比搜索所有内容并排序噪音要快得多和干净得多。
- **`type` vs `glob`。** `type` 使用 ripgrep 的内置语言定义（它知道哪些扩展名属于 `python`、`rust` 等）。`glob` 给你对文件模式的精确控制。使用哪个更清楚地表达你的意图。
- **JSON 中的转义。**因为模式通过 JSON 传输，正则表达式中的反斜杠必须在 JSON 字符串中写为 `\\`（例如 `\\.` 来匹配字面点）。
- **`.gitignore` 感知。**像 ripgrep 一样，工具默认跳过被忽略的文件（例如 `node_modules`）— 通常是你想要的。
- **对于探索，与 `Glob` 配对。**使用 `Glob` 按名称发现候选文件，然后 `Grep` 搜索它们的内容 — 或将两者都交给智能体进行广泛的扇出搜索。
