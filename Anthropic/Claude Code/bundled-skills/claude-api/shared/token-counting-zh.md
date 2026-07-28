> **说明**：本文件为英文原文（`token-counting.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# Token 计数

使用 `count_tokens` 端点（`POST /v1/messages/count_tokens`）获取针对 Claude 模型的准确 token 计数。Token 计数是**模型特定的**——传入你将用于推理的同一模型 ID。

**不要使用 `tiktoken`。** 它是 OpenAI 的分词器。它在典型文本上少计 Claude token 约 15-20%，在代码或非英语输入上少计更多。任何来自 `tiktoken`、`gpt-tokenizer` 或类似工具的估计对 Claude 都是错误的。

## 计数文件或字符串

```python
from anthropic import Anthropic

client = Anthropic()
resp = client.messages.count_tokens(
    model="claude-opus-4-8",
    messages=[{"role": "user", "content": open("CLAUDE.md").read()}],
)
print(resp.input_tokens)
```

TypeScript：`await client.messages.countTokens({model, messages})` → `.input_tokens`。其他 SDK 见 `{lang}/claude-api/README.md`。

## CLI

```sh
ant messages count-tokens --model claude-opus-4-8 \
  --message '{role: user, content: "@./CLAUDE.md"}' \
  --transform input_tokens -r
```

## 对比文件的两个版本

端点是无状态的——分别计数每个版本并相减：

```python
from anthropic import Anthropic
import subprocess

client = Anthropic()
def count(text: str) -> int:
    return client.messages.count_tokens(
        model="claude-opus-4-8",
        messages=[{"role": "user", "content": text}],
    ).input_tokens

before = subprocess.check_output(["git", "show", "HEAD:CLAUDE.md"], text=True)
after = open("CLAUDE.md").read()
print(count(after) - count(before))
```

完整文档：见 `shared/live-sources.md` 中的 Token Counting 条目。
