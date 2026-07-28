> **说明**：本文件为英文原文（`claude-platform-on-aws.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# AWS 上的 Claude Platform

通过 AWS 基础设施访问 Claude Developer Platform 的 **Anthropic 运营**接入，使用 SigV4 身份验证、AWS IAM 访问控制和 AWS Marketplace 计费。由于由 Anthropic 运营，**API 接口与第一方保持当日同步**，各功能例外情况请参阅 `shared/platform-availability.md`（唯一事实来源；不要依赖此处的内联例外列表）。模型 ID 为裸第一方字符串（`claude-opus-4-8`、`claude-sonnet-5`），**无提供商前缀**。

> **与 Amazon Bedrock 不同。** Bedrock 是合作伙伴运营的（AWS 运行该服务；发布计划各异，功能子集，`anthropic.` 前缀的模型 ID）。AWS 上的 Claude Platform 和 Bedrock 共存；根据你需要的是具有完整 Anthropic API 同步的 AWS 原生 IAM/计费（本页）还是 Bedrock 自身的生态系统来选择。

---

## 客户端与安装

| 语言 | 安装 | 客户端 |
|---|---|---|
| Python | `pip install -U "anthropic[aws]"` | `from anthropic import AnthropicAWS` → `AnthropicAWS()` |
| TypeScript | `npm install @anthropic-ai/aws-sdk` | `import AnthropicAws from "@anthropic-ai/aws-sdk"` → `new AnthropicAws()` |
| Go | `go get github.com/anthropics/anthropic-sdk-go` | `import anthropicaws "github.com/anthropics/anthropic-sdk-go/aws"` → `anthropicaws.NewClient(ctx, anthropicaws.ClientConfig{})` |
| C# | `dotnet add package Anthropic.Aws` | `new AnthropicAwsClient()` |
| Java | 参见 `shared/live-sources.md` 中的 SDK 仓库 | 参见 `shared/live-sources.md` 中的 SDK 仓库 |
| Ruby | `gem install anthropic aws-sdk-core` | 参见 `shared/live-sources.md` 中的 SDK 仓库 |
| PHP | `composer require anthropic-ai/sdk aws/aws-sdk-php` | 参见 `shared/live-sources.md` 中的 SDK 仓库 |

构造完成后，**像使用 `Anthropic()` 一样使用该客户端**，`client.messages.create(...)`、`client.beta.sessions.*` 等，使用裸模型 ID。

```python
from anthropic import AnthropicAWS

client = AnthropicAWS()  # region + workspace_id from env; see below
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

---

## 必需配置

两个值必须可用（构造函数参数或环境变量），**两者都没有默认回退值**：

| 值 | 环境变量 | 备注 |
|---|---|---|
| AWS 区域 | `AWS_REGION` | 必需。与 `AnthropicBedrock` 不同，没有 `us-east-1` 回退。 |
| 工作区 ID | `ANTHROPIC_AWS_WORKSPACE_ID` | 必需。将请求路由到你的 Claude 工作区。 |

端点模式：`https://aws-external-anthropic.{region}.api.aws/v1/...`。请求使用服务名称 `aws-external-anthropic` 进行 SigV4 签名。

## 身份验证

客户端通过标准优先级链解析 AWS 凭证：显式构造函数参数 → 环境变量（`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`）→ 共享配置文件 → 假定角色 / 实例元数据。

**短期 API 密钥**也受支持，适用于 SigV4 不实际的情况（如浏览器、简单脚本）。使用各语言的 token 生成器包生成一个；将其作为 `api_key` 传递给客户端。有效期为请求时长、底层凭证过期时间与 **12 小时**中的较小者。有关包名称和 IAM 详情，请 WebFetch `shared/live-sources.md` 中 AWS 上的 Claude Platform 页面。

---

## 告知用户的内容

- 将其视为第一方：本技能的每个章节均适用不变。**不要**应用 Bedrock 的功能可用性掩码。
- 模型 ID 为裸 ID（`claude-opus-4-8`）。**不要**添加 `anthropic.` 前缀。
- 缺少 region 或 `workspace_id` 会在客户端构造时抛出异常（不会发送请求）。**403** 表示请求已到达服务器，请检查是否为**错误的** `workspace_id` 或主体上缺少 IAM 操作。参见 `shared/live-sources.md` 中的 IAM 操作参考。
