> **说明**：本文件为英文原文（`error-codes.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

# HTTP 错误码参考

本文档记录了 Claude API 返回的 HTTP 错误码、常见原因以及如何处理它们。如需特定语言的错误处理示例，请参见 `python/` 或 `typescript/` 文件夹。

## 错误码摘要

| 代码 | 错误类型 | 可重试 | 常见原因 |
| ---- | ----------------------- | --------- | ------------------------------------ |
| 400  | `invalid_request_error` | 否 | 请求格式或参数无效 |
| 401  | `authentication_error`  | 否 | API 密钥无效或缺失 |
| 403  | `permission_error`      | 否 | API 密钥缺少权限 |
| 404  | `not_found_error`       | 否 | 端点或模型 ID 无效 |
| 413  | `request_too_large`     | 否 | 请求超出大小限制 |
| 429  | `rate_limit_error`      | 是 | 请求过多 |
| 500  | `api_error`             | 是 | Anthropic 服务问题 |
| 529  | `overloaded_error`      | 是 | API 暂时过载 |

## 详细错误信息

### 400 Bad Request

**原因：**

- 请求体中的 JSON 格式错误
- 缺少必需参数（`model`、`max_tokens`、`messages`）
- 参数类型无效（例如需要整数却传了字符串）
- messages 数组为空
- messages 未在 user/assistant 之间交替

**错误示例：**

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "messages: roles must alternate between \"user\" and \"assistant\""
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

**修复：** 发送前验证请求结构。检查以下各项：

- `model` 是有效的模型 ID
- `max_tokens` 是正整数
- `messages` 数组非空且正确交替

---

### 401 Unauthorized

**原因：**

- 缺少 `x-api-key` 头或 `Authorization` 头
- API 密钥格式无效
- API 密钥已被吊销或删除
- 通过 `x-api-key` 发送 OAuth bearer token 而非 `Authorization: Bearer`
- 同时设置了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`，SDK 会发送两个头，API 拒绝请求

**修复：** 设置 `ANTHROPIC_API_KEY`，或运行 `ant auth login` 并将客户端构造函数留空。对于使用 OAuth token 的原始 HTTP 请求，使用 `Authorization: Bearer <token>`（而非 `x-api-key:`）。

---

### 403 Forbidden

**原因：**

- API 密钥无权访问所请求的模型
- 组织级限制
- 尝试在无 beta 访问权限的情况下访问 beta 功能

**修复：** 在 Console 中检查你的 API 密钥权限。你可能需要不同的 API 密钥或申请访问特定功能。

---

### 404 Not Found

**原因：**

- 模型 ID 拼写错误（例如 `claude-sonnet-4.6` 而非 `claude-sonnet-4-6`）
- 使用了已弃用的模型 ID
- API 端点无效

**修复：** 使用模型文档中的确切模型 ID。可以使用别名（例如 `claude-opus-4-8`）。

---

### 413 Request Too Large

**原因：**

- 请求体超出最大大小
- 输入中的 token 过多
- 图片数据过大

**修复：** 减小输入大小，截断对话历史、压缩/调整图片大小，或将大文档拆分为多个块。

---

### 400 验证错误

部分 400 错误专门与参数验证相关：

- `max_tokens` 超过模型限制
- 无效的 `temperature` 值（必须为 0.0-1.0）
- 扩展思维中 `budget_tokens` >= `max_tokens`
- 无效的工具定义 schema

**Fable 5 / Opus 4.8 / 4.7 上的模型特定 400 错误：**

- `temperature`、`top_p`、`top_k` 已移除，发送其中任何一个会返回 400。删除该参数；参见 `shared/model-migration.md` → 各 SDK 语法参考。
- `thinking: {type: "enabled", budget_tokens: N}` 已移除，发送会返回 400。改用 `thinking: {type: "adaptive"}`。
- **仅 Fable 5：**显式的 `thinking: {type: "disabled"}` 会返回 400（在 Opus 4.8/4.7 上可接受）。改为完全省略 `thinking` 参数。
- **仅 Fable 5：**如果组织设置了零数据保留（ZDR），或任何低于所需 30 天的保留设置，那么所有 Fable 5 请求都会返回 `400 invalid_request_error`，即使 payload 完全有效。在调试请求体之前，先检查组织的保留配置。

**旧模型（Opus 4.6 及更早版本）上扩展思维的常见错误：**

```
# 错误：budget_tokens 必须 < max_tokens
thinking: budget_tokens=10000, max_tokens=1000  → Error!

# 正确
thinking: budget_tokens=10000, max_tokens=16000
```

---

### 429 Rate Limited

**原因：**

- 超出每分钟请求数（RPM）
- 超出每分钟 token 数（TPM）
- 超出每天 token 数（TPD）

**需检查的头：**

- `retry-after`：重试前需等待的秒数
- `x-ratelimit-limit-*`：你的限制
- `x-ratelimit-remaining-*`：剩余配额

**修复：** Anthropic SDK 会自动以指数退避方式重试 429 和 5xx 错误（默认：`max_retries=2`）。如需自定义重试行为，请参见特定语言的错误处理示例。

---

### 500 Internal Server Error

**原因：**

- 暂时的 Anthropic 服务问题
- API 处理中的 bug

**修复：** 使用指数退避重试。如果持续出现，请检查 [status.anthropic.com](https://status.anthropic.com)。

---

### 529 Overloaded

**原因：**

- API 需求过高
- 服务容量已达上限

**修复：** 使用指数退避重试。考虑使用不同的模型（Haiku 通常负载较低）、分散请求时间，或实现请求队列。

---

## 常见错误与修复

| 错误 | 报错 | 修复 |
| ------------------------------- | ---------------- | ------------------------------------------------------- |
| 在 Fable 5 / Opus 4.8 / 4.7 上使用 `temperature`/`top_p`/`top_k` | 400 | 删除该参数（参见 `shared/model-migration.md`）  |
| 在 Fable 5 / Opus 4.8 / 4.7 上使用 `budget_tokens` | 400  | 使用 `thinking: {type: "adaptive"}`                      |
| 在 Fable 5 上使用 `thinking: {type: "disabled"}` | 400    | 完全省略 `thinking` 参数（在 Opus 4.8/4.7 上可接受） |
| 组织设置为 ZDR / 保留期低于 30 天（Fable 5） | 每个请求都返回 400 | 修复组织的数据保留配置，payload 不是问题 |
| `budget_tokens` >= `max_tokens`（旧模型） | 400 | 确保 `budget_tokens` < `max_tokens`                  |
| 模型 ID 拼写错误                | 404              | 使用有效模型 ID 如 `claude-opus-4-8`               |
| 第一条消息是 `assistant`    | 400              | 第一条消息必须是 `user`                            |
| 连续相同角色的消息  | 400              | 交替 `user` 和 `assistant`                        |
| API 密钥写在代码中                 | 401（密钥泄露） | 使用环境变量                                |
| 需要自定义重试              | 429/5xx          | SDK 自动重试；用 `max_retries` 自定义 |
| 各 SDK 中的类型化异常 |

**始终使用 SDK 的类型化异常类**，而非用字符串匹配检查错误消息。每个 HTTP 状态码在每个 SDK 中映射到特定的异常类。

### 按语言的异常类名

| HTTP | Python (`anthropic.*`) / TypeScript (`Anthropic.*`) | Ruby (`Anthropic::Errors::*`) | Java (`com.anthropic.errors.*`) | C# | PHP (`Anthropic\Core\Exceptions\*`) |
|---|---|---|---|---|---|
| 400 | `BadRequestError` | `BadRequestError` | `BadRequestException` | `AnthropicBadRequestException` | `BadRequestException` |
| 401 | `AuthenticationError` | `AuthenticationError` | `UnauthorizedException` | `AnthropicUnauthorizedException` | `AuthenticationException` |
| 403 | `PermissionDeniedError` | `PermissionDeniedError` | `PermissionDeniedException` | `AnthropicForbiddenException` | `PermissionDeniedException` |
| 404 | `NotFoundError` | `NotFoundError` | `NotFoundException` | `AnthropicNotFoundException` | `NotFoundException` |
| 422 | `UnprocessableEntityError` | `UnprocessableEntityError` | `UnprocessableEntityException` | `AnthropicUnprocessableEntityException` | `UnprocessableEntityException` |
| 429 | `RateLimitError` | `RateLimitError` | `RateLimitException` | `AnthropicRateLimitException` | `RateLimitException` |
| ≥500 | `InternalServerError` | `InternalServerError` | `InternalServerException` | `Anthropic5xxException` | `InternalServerException` |
| 网络 | `APIConnectionError` | `APIConnectionError` | `AnthropicIoException` | `AnthropicIOException` | `APIConnectionException` |
| 基类 | `APIError`（两者通用）；`APIStatusError`（仅 Python） | `APIStatusError` / `APIError` | `AnthropicServiceException` | `AnthropicApiException` | `APIStatusException` / `APIException` |

Ruby 和 PHP 的类位于专用的 errors 命名空间中，写 `Anthropic::Errors::RateLimitError` 和 `Anthropic\Core\Exceptions\RateLimitException`（而非裸写 `Anthropic::RateLimitError`）。所有 4xx C# 异常也继承自 `Anthropic4xxException`。

### 先捕获最具体的，形成链式结构

从最具体的子类到基类排列 `catch`/`except`/`rescue` 子句，为每个你以不同方式处理的类别设置单独的子句，可重试的（429、≥500、网络）与非可重试的（4xx）分开。SDK 为每个状态码定义了不同的类正是为此目的；单一的宽泛 catch-all 会丢弃这些信息。

```python
try:
    msg = client.messages.create(...)
except anthropic.NotFoundError as e:          # 404 — 例如模型 ID 错误
    ...
except anthropic.RateLimitError as e:         # 429 — 退避并重试
    ...
except anthropic.APIStatusError as e:         # 任何其他非 2xx HTTP 响应
    print(e.status_code, e.message)
except anthropic.APIConnectionError as e:     # 响应前的网络故障
    ...
```

相同的链式结构适用于每个 SDK：TypeScript `instanceof Anthropic.NotFoundError` → `RateLimitError` → `APIConnectionError` → `APIError`（在 `APIError` 之前检查 `APIConnectionError`，因为在 TypeScript SDK 中它是 `APIError` 的子类，不同于 Python 中它们是平级的）；Ruby `rescue Anthropic::Errors::NotFoundError` → `…::RateLimitError` → `…::APIStatusError`；Java `catch (NotFoundException) … catch (RateLimitException) … catch (AnthropicServiceException)`；C# `catch (AnthropicNotFoundException) … catch (AnthropicRateLimitException) … catch (AnthropicApiException)`；PHP `catch (NotFoundException) … catch (RateLimitException) … catch (APIStatusException)`。

### Go — `errors.As` 然后按状态分支

Go SDK 对所有非 2xx 响应返回单一的 `*anthropic.Error`。用 `errors.As` 解包，然后按 `StatusCode` 分支：

```go
_, err := client.Messages.New(ctx, params)
if err != nil {
    var apierr *anthropic.Error
    if errors.As(err, &apierr) {
        switch apierr.StatusCode {
        case 404:
            // 模型 ID / 资源错误
        case 429:
            // 退避并重试
        default:
            // 其他 API 错误 — apierr.StatusCode, apierr.RequestID
        }
    } else {
        // 传输层错误（*url.Error 包装 *net.OpError 等）
    }
}
```

### 错误 `.type` 字段

所有 `APIStatusError` 子类现在都暴露了 `.type` 属性（Python：`.type`，TypeScript：`.type`，Java：`.errorType()`，Go：`.Type()`，Ruby：`.type`，PHP：`.type`），返回 API 错误类型字符串（例如 `"invalid_request_error"`、`"authentication_error"`、`"rate_limit_error"`、`"overloaded_error"`）。当你需要比 HTTP 状态码更细粒度的分类时使用它，例如区分 `"billing_error"` 和 `"permission_error"`（两者都映射到 403）。

```python
except anthropic.APIStatusError as e:
    if e.type == "rate_limit_error":
        # 处理速率限制
    elif e.type == "overloaded_error":
        # 处理过载
```
