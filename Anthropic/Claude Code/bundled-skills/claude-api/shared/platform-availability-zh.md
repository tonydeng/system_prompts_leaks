> **说明**：本文件为英文原文（`platform-availability.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# 平台可用性

哪些功能在哪些提供商平台上可用。**此表是本技能中的唯一事实来源**，其他各功能章节指向此处而非重复说明可用性。在为第三方平台（Bedrock、Vertex、Foundry）或 AWS 上的 Claude Platform 编写代码时，请先查看此表；某个功能在该处不受支持意味着需要使用第一方 Claude API 接口或其他方法。

列说明：**1P** = 第一方 Claude API，**P-AWS** = AWS 上的 Claude Platform（Anthropic 运营，当日同步），**Bedrock** = Amazon Bedrock，**Vertex** = Google Cloud Vertex AI，**Foundry** = Microsoft Foundry。✅ = 正式可用，β = 测试版，❌ = 不支持。

| 功能 | 1P | P-AWS | Bedrock | Vertex | Foundry | 备注 |
|---|---|---|---|---|---|---|
| Messages、流式传输、工具使用 | ✅ | ✅ | ✅ | ✅ | ✅ | 核心 API |
| PDF 输入 | ✅ | ✅ | ✅ | ✅ | β | |
| 结构化输出 / 严格工具使用 | ✅ | ✅ | ✅ | ✅ | β | |
| 自适应思考 / 推理力度 | ✅ | ✅ | ✅ | ✅ | β | |
| 扩展思考 | ✅ | ✅ | ✅ | ✅ | β | |
| 提示词缓存（5分钟、1小时） | ✅ | ✅ | ✅ | ✅ | β | |
| 自动提示词缓存 | ✅ | ✅ | ❌ | ❌ | β | |
| Token 计数 | ✅ | ✅ | ✅ | ✅ | β | |
| 引用 | ✅ | ✅ | ✅ | ✅ | β | |
| 搜索结果内容块 | ✅ | ✅ | ✅ | ✅ | β | |
| 细粒度工具流式传输 | ✅ | ✅ | ✅ | ✅ | ✅ | |
| 压缩 | β | β | β | β | β | |
| 上下文编辑 | β | β | β | β | β | |
| 上下文窗口（1M） | ✅ | ✅ | ✅ | ✅ | β | |
| `inference_geo`（数据驻留） | ✅ | ✅ | ❌ | ❌ | ❌ | |
| **服务端工具** | | | | | | |
| &nbsp;&nbsp;Web 搜索 | ✅ | ✅ | ❌ | ✅ | β | Vertex：仅基础 `web_search_20250305`（无 `_20260209` 动态过滤） |
| &nbsp;&nbsp;Web 抓取 | ✅ | ✅ | ❌ | ❌ | β | |
| &nbsp;&nbsp;代码执行 | ✅ | ✅ | ❌ | ❌ | β | |
| &nbsp;&nbsp;工具搜索 | ✅ | ✅ | ✅ | ✅ | β | Bedrock：仅 InvokeModel API，非 Converse |
| &nbsp;&nbsp;Advisor 工具 | β | β | ❌ | ❌ | ❌ | |
| **客户端实现的工具** | | | | | | |
| &nbsp;&nbsp;Bash、文本编辑器、记忆 | ✅ | ✅ | ✅ | ✅ | β | |
| &nbsp;&nbsp;计算机使用 | β | β | β | β | β | |
| **智能体 / 编排** | | | | | | |
| &nbsp;&nbsp;Agent Skills（Messages API） | β | β | ❌ | ❌ | β | |
| &nbsp;&nbsp;编程式工具调用 | ✅ | ✅ | ❌ | ❌ | β | |
| &nbsp;&nbsp;MCP 连接器 | β | β | ❌ | ❌ | β | |
| &nbsp;&nbsp;Managed Agents | β | β | ❌ | ❌ | ❌ | Foundry ❌ 为推断（Foundry 文档中未提及） |
| &nbsp;&nbsp;自托管沙箱 | β | β | ❌ | ❌ | ❌ | P-AWS：不支持 `GET /v1/environments/{id}/work` 列表端点；其他 work 端点可用 |
| **API 端点** | | | | | | |
| &nbsp;&nbsp;Message Batches | ✅ | ✅ | ❌ | ❌ | ❌ | |
| &nbsp;&nbsp;Files API | β | β | ❌ | ❌ | β | |
| &nbsp;&nbsp;Models API | ✅ | ✅ | ❌ | ❌ | ❌ | |
| **其他** | | | | | | |
| &nbsp;&nbsp;对话中系统消息 | ✅ | ✅ | ❌ | ❌ | ❌ | 仅 Claude Opus 4.8 |
| &nbsp;&nbsp;快速模式 | β | ❌ | ❌ | ❌ | ❌ | 研究预览，测试版 `fast-mode-2026-02-01`，仅第一方 API |
| &nbsp;&nbsp;缓存诊断 | β | ❌ | ❌ | ❌ | ❌ | 仅第一方 API |
| &nbsp;&nbsp;任务预算 | β | β | ❌ | ❌ | ❌ | Beta header `task-budgets-2026-03-13`；第三方可用性未记录，假设不支持 |
