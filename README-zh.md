<p align="center">
  <sub>鸣谢</sub>  
</p>

<p align="center">
  <a href="https://go.asgeirtj.workers.dev/latitude">
    <img src="assets/latitude-dark.png" alt="Latitude Logo" width="700"/>
  </a>
</p>

<div align="center" markdown="1">

### [让你的 AI 智能体自我修复](https://go.asgeirtj.workers.dev/latitude)  
[开源 AI 监控](https://go.asgeirtj.workers.dev/latitude)

</div>

---

> **《华盛顿邮报》**基于本仓库的提示词制作了交互式报道：[揭开 AI 背后的隐藏规则，然后用它们重写这篇文章。](https://wapo.st/49t4gSb)（2026 年 5 月 11 日）
> 
> **CEPS' AI World** 基于本仓库的文件制作了实时数据看板：[系统提示词及其在对话开始前告诉我们的信息](https://aiworld.eu/story/system-prompts-and-what-they-tell-us-about-the-chat-before-the-chat)（2026 年 7 月 10 日）

# 系统提示词泄露

泄露的系统提示词，原文照录——ChatGPT、Claude、Gemini、Grok 以及所有其他 AI 聊天机器人在你的第一条消息之前收到的隐藏指令和规则。

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/banner-dark.png">
  <source media="(prefers-color-scheme: light)" srcset=".github/banner-light.png">
  <img alt="ChatGPT 在被要求重复上述所有内容后泄露了其系统提示词" src=".github/banner-light.png">
</picture>

![最后提交](https://img.shields.io/github/last-commit/asgeirtj/system_prompts_leaks?style=flat)
[![欢迎 PR](https://img.shields.io/badge/PRs-welcome-brightgreen)](http://makeapullrequest.com)




## 最近更新

| 内容 | 日期 | 链接 |
|------|------|------|
| **Codex GPT-5.6（Sol 变体）** | 2026 年 7 月 26 日 | [Codex GPT-5.6 系统提示词（Terra/Luna）](OpenAI/Codex/gpt-5.6.md) · [Sol](OpenAI/Codex/gpt-5.6-sol.md) |
| **Grok 4.5** | 2026 年 7 月 26 日 | [Grok 4.5 系统提示词](xAI/grok-4.5.md) |
| **Claude Opus 5** | 2026 年 7 月 24 日 | [Claude Opus 5 系统提示词](Anthropic/claude-opus-5.md) · [Claude Code（Opus 5）](Anthropic/Claude%20Code/claude-code-opus-5.md) |
| **Claude Design（完整提示词 + 53 个工具 + 22 个技能 + 10 个起始组件）** | 2026 年 7 月 23 日 | [Claude Design 系统提示词](Anthropic/claude-design.md) · [技能](Anthropic/Claude%20Design/Skills) · [起始组件](Anthropic/Claude%20Design/Starter%20components) |
| **Perplexity** | 2026 年 7 月 17 日 | [Perplexity AI 系统提示词](Perplexity/perplexity-ai.md) |
| **Claude Code（新模型）** | 2026 年 7 月 16 日 | [Claude Code 系统提示词（Fable 5）](Anthropic/Claude%20Code/claude-code-fable-5.md) · [Sonnet 5](Anthropic/Claude%20Code/claude-code-sonnet-5.md) |
| **OpenCode · Pi · CommandCode** | 2026 年 7 月 16 日 | [OpenCode 系统提示词](OpenCode/opencode.md) · [Pi 系统提示词](Pi/instructions.md) · [CommandCode CLI 系统提示词](Misc/commandcode-cli.md) |
| **Kimi K2.6** | 2026 年 7 月 14 日 | [Kimi K2.6 系统提示词](Kimi/kimi-2.6.md) |
| **Perplexity Deep Research** | 2026 年 7 月 14 日 | [Perplexity Deep Research 系统提示词](Perplexity/deep-research.md) |
| **DeepSeek** | 2026 年 7 月 14 日 | [DeepSeek 系统提示词](DeepSeek/deepseek-chat.md) |
| **ChatGPT 5.6** | 2026 年 7 月 10 日 | [ChatGPT 5.6 系统提示词（Sol，extra high）](OpenAI/gpt-5.6-sol-extra-high.md) · [Codex GPT-5.6 系统提示词](OpenAI/Codex/gpt-5.6.md) |


---
![Anthropic](https://shieldcn.dev/badge/Anthropic-D97757.svg?logo=anthropic&logoColor=fff&variant=secondary&mode=light)

## Anthropic — Claude 系统提示词

### Claude.ai 系统提示词（网页版、桌面版和移动版）

| 模型 | 提示词 |
|-------|--------|
| **Claude Fable 5** | [**Claude Fable 5 系统提示词**](Anthropic/claude-fable-5.md) |
| **Claude Opus 5** | [**Claude Opus 5 系统提示词**](Anthropic/claude-opus-5.md) |
| Claude Opus 4.8 | [Claude Opus 4.8 系统提示词](Anthropic/claude-opus-4.8.md) |
| **Claude Sonnet 5** | [**Claude Sonnet 5 系统提示词**](Anthropic/claude-sonnet-5.md) |
| Claude Opus 4.7 | [Claude Opus 4.7 系统提示词](Anthropic/claude-opus-4.7.md) |
| Claude Opus 4.6 | [Claude Opus 4.6 系统提示词](Anthropic/claude-opus-4.6.md) · [无工具版](Anthropic/claude-opus-4.6-no-tools.md) |
| Claude Sonnet 4.6 | [Claude Sonnet 4.6 系统提示词](Anthropic/claude-sonnet-4.6.md) · [无工具版](Anthropic/claude-sonnet-4.6-no-tools.md) |
| 注入提醒 | [Claude.ai 注入提醒](Anthropic/anthropic_reminders.md) |

### Claude Code 系统提示词

| 组件 | 提示词 |
|-----------|--------|
| **Claude Code（Fable 5）** | [**Claude Code 系统提示词（Fable 5）**](Anthropic/Claude%20Code/claude-code-fable-5.md) |
| **Claude Code（Opus 5）** | [**Claude Code 系统提示词（Opus 5）**](Anthropic/Claude%20Code/claude-code-opus-5.md) |
| Claude Code（Opus 4.8） | [Claude Code 系统提示词（Opus 4.8）](Anthropic/Claude%20Code/claude-code-opus-4.8.md) |
| **Claude Code（Sonnet 5）** | [Claude Code 系统提示词（Sonnet 5）](Anthropic/Claude%20Code/claude-code-sonnet-5.md) |
| Claude Code（旧模型） | [Opus 4.7](Anthropic/Claude%20Code/claude-code-opus-4.7.md) · [Opus 4.6](Anthropic/Claude%20Code/claude-code-opus-4.6.md) · [Sonnet 4.6](Anthropic/Claude%20Code/claude-code-sonnet-4.6.md) · [Haiku 4.5](Anthropic/Claude%20Code/claude-code-haiku-4.5.md) |
| 子代理 | [Claude Code 子代理系统提示词](Anthropic/Claude%20Code/agents) |
| 技能和命令 | [Claude Code 内置技能](Anthropic/Claude%20Code/bundled-skills) · [斜杠命令](Anthropic/Claude%20Code/slash-commands) · [技能](Anthropic/Claude%20Code/skills) |
| 注入提醒 | [Claude Code 注入提醒](Anthropic/Claude%20Code/injected-reminders) |
| MCP 服务器 | [Claude Code MCP 服务器系统提示词](Anthropic/Claude%20Code/mcp-servers) |
| 文档助手 | [docs.claude.com 助手指令](Anthropic/Claude%20Code/claude-code-docs-assistant.md) |


### Claude 集成

| 产品 | 提示词 |
|---------|--------|
| **Claude Design** | [**Claude Design 系统提示词**](Anthropic/claude-design.md) · [技能](Anthropic/Claude%20Design/Skills) · [起始组件](Anthropic/Claude%20Design/Starter%20components) |
| **Claude Cowork** | [Claude Cowork 系统提示词](Anthropic/claude-cowork.md) · [Dispatch](Anthropic/claude-cowork-dispatch.md) |
| Claude for Microsoft 365 | [Claude for Excel](Anthropic/claude-for-excel.md) · [Claude for Word](Anthropic/claude-for-word.md) · [Claude in PowerPoint](Anthropic/claude-in-powerpoint.md) |
| Chrome 中的 Claude | [Claude in Chrome 扩展系统提示词](Anthropic/claude-in-chrome.md) |
| Claude iOS 应用 | [Claude 移动版 iOS 系统提示词](Anthropic/claude-mobile-ios.md) |

![OpenAI](https://shieldcn.dev/badge/OpenAI-412991.svg?logo=ri%3ASiOpenai&variant=secondary&mode=light)

## OpenAI — ChatGPT 系统提示词

| 模型 | 提示词 |
|-------|--------|
| **ChatGPT 5.6 Sol** | [**ChatGPT 5.6 系统提示词（Sol，extra high）**](OpenAI/gpt-5.6-sol-extra-high.md) |
| **ChatGPT 5.5 Thinking** | [**ChatGPT 5.5 Thinking 系统提示词**](OpenAI/gpt-5.5-thinking.md) |
| **ChatGPT 5.5 Instant** | [**ChatGPT 5.5 Instant 系统提示词**](OpenAI/gpt-5.5-instant.md) |
| ChatGPT 5.4 | [ChatGPT 5.4 Thinking 系统提示词](OpenAI/gpt-5.4-thinking.md) |
| ChatGPT 5.3 | [ChatGPT 5.3 Instant 系统提示词](OpenAI/gpt-5.3-instant.md) |
| ChatGPT 5.2 | [ChatGPT 5.2 Thinking 系统提示词](OpenAI/gpt-5.2-thinking.md) |
| ChatGPT 5 | [ChatGPT 5 Thinking 系统提示词](OpenAI/gpt-5-thinking.md) · [Agent 模式](OpenAI/chatgpt-gpt-5-agent-mode.md) |
| **ChatGPT Atlas** | [ChatGPT Atlas 系统提示词](OpenAI/chatgpt-atlas.md) |
| ChatGPT 4.5 | [ChatGPT 4.5 系统提示词](OpenAI/chatgpt-4.5.md) |
| ChatGPT 4o | [ChatGPT 4o 系统提示词](OpenAI/gpt-4o.md) · [弃用准备](OpenAI/ChatGPT/chatgpt-4o-deprecation-preparedness-prompt.md) |
| 语音模式 | [ChatGPT 高级语音模式系统提示词](OpenAI/gpt-4o-advanced-voice-mode.md) · [旧版语音模式](OpenAI/gpt-4o-legacy-voice-mode.md) |
| 个性 | [ChatGPT 个性指令](OpenAI/chatgpt-personality-instructions.md) |
| 记忆 | [ChatGPT 高级记忆系统提示词](OpenAI/tool-advanced-memory.md) |

### Codex 系统提示词

| 模型 | 提示词 |
|-------|--------|
| **Codex GPT-5.6** | [**Codex GPT-5.6 系统提示词（Terra/Luna）**](OpenAI/Codex/gpt-5.6.md) · [Sol](OpenAI/Codex/gpt-5.6-sol.md) |
| **Codex GPT-5.5** | [Codex GPT-5.5 系统提示词](OpenAI/Codex/gpt-5.5.md) · [完整提示词](OpenAI/Codex/codex-full.md) · [Friendly](OpenAI/Codex/personality_friendly_gpt-5.5.md) · [Pragmatic](OpenAI/Codex/personality_pragmatic_gpt-5.5.md) |
| Codex GPT-5.4 | [Codex GPT-5.4 系统提示词](OpenAI/Codex/gpt-5.4.md) · [Mini](OpenAI/Codex/gpt-5.4-mini.md) |
| Codex Spark | [Codex Spark 系统提示词](OpenAI/Codex/gpt-5.3-codex-spark.md) |
| Codex 模式 | [Plan 模式](OpenAI/Codex/plan_mode.md) · [自动审查](OpenAI/Codex/codex-auto-review.md) · [Computer use](OpenAI/Codex/computer-use.md) · [Control Chrome](OpenAI/Codex/control-chrome.md) · [应用内浏览器](OpenAI/Codex/control-in-app-browser.md) |
| 人格 | [Friendly](OpenAI/Codex/personality_friendly.md) · [Pragmatic](OpenAI/Codex/personality_pragmatic.md) |

### API 注入提示词

| 模型 | 提示词 |
|-------|--------|
| GPT-5.5 | [GPT-5.5 API 系统提示词](OpenAI/gpt-5.5-api.md) · [Pro](OpenAI/gpt-5.5-pro-api.md) |
| GPT-5.4 / 5.3 | [GPT-5.4 API](OpenAI/gpt-5.4-api.md) · [5.3 Chat](OpenAI/gpt-5.3-chat-api.md) · [5.3 Codex](OpenAI/gpt-5.3-codex-api.md) |
| o 系列和旧模型 | [o3 / o4-mini reasoning-effort 变体](OpenAI/API/) |

<details><summary>旧模型、工具和已弃用功能</summary>

| | |
|--|--|
| 旧模型 | [GPT-4.5](OpenAI/gpt-4.5.md) · [GPT-4.1](OpenAI/gpt-4.1.md) · [GPT-4.1 Mini](OpenAI/gpt-4.1-mini.md) · [o3](OpenAI/Old/o3.md) · [o4-mini](OpenAI/Old/o4-mini.md) · [GPT-5.2 Mini（免费）](OpenAI/gpt-5.2-mini-free-account.md) · [ChatGPT 4o Mini](OpenAI/Old/chatgpt-4o-mini.md) |
| 旧 4o 变体 | [4o WhatsApp](OpenAI/Old/gpt-4o-whatsapp.md) · [4o 新个性](OpenAI/4o-2025-09-03-new-personality.md) · [Monday GPT](OpenAI/Old/monday-gpt.md) |
| 旧工具 | [Canvas](OpenAI/Old/tool-canvas-canmore.md) · [图片生成](OpenAI/Old/tool-create-image-image_gen.md) · [文件搜索](OpenAI/Old/tool-file_search.md) · [Python](OpenAI/Old/tool-python-code.md) · [网页搜索](OpenAI/Old/tool-web-search.md) |
| 旧策略 | [图片安全](OpenAI/Old/prompt-image-safety-policies.md) · [图片安全（2026）](OpenAI/Old/image-safety-policies.md) · [自动化上下文](OpenAI/Old/prompt-automation-context.md) |
| 已弃用功能 | [GPT-5 个性](OpenAI/gpt-5-listener-personality.md) · [GPT-5.1 个性](OpenAI/gpt-5.1-efficient.md) · [深度研究工具](OpenAI/tool-deep-research.md) · [学习和研究](OpenAI/Old/study-and-learn.md) · [全部](OpenAI/deprecated/) |
| GPT-5.1（旧） | [Professional](OpenAI/gpt-5.1-professional.md) |

</details>

![Google Gemini](https://shieldcn.dev/badge/Google%20Gemini-8E75B2.svg?logo=googlegemini&logoColor=fff&variant=secondary&mode=light)

## Google — Gemini 系统提示词

| 模型 | 提示词 |
|-------|--------|
| **Gemini 3.5 Flash** | [**Gemini 3.5 Flash 系统提示词**](Google/gemini-3.5-flash.md) · [AI Studio](Google/gemini-3.5-flash-ai-studio.md) |
| **Gemini 3.1 Pro** | [**Gemini 3.1 Pro 系统提示词**](Google/gemini-3.1-pro.md) · [API](Google/gemini-3.1-pro-api.md) |
| **Antigravity CLI** | [**Antigravity CLI 系统提示词**](Google/antigravity-cli.md) |
| Nano / Banana 2 | [Nano Banana 2 系统提示词](Google/nano-banana-2-api.md) |
| Google Search AI 模式 | [Google Search AI Mode 系统提示词](Google/google-search-ai-mode.md) |
| Gemini CLI | [Gemini CLI 系统提示词](Google/gemini-cli.md) |
| NotebookLM | [NotebookLM 聊天系统提示词](Google/notebooklm-chat.md) |
| Jules | [Jules 系统提示词](Google/jules.md) |
| AI Studio Build | [AI Studio Build 系统提示词](Google/ai-studio-build.md) |
| Gemini 3 | [Gemini 3 Flash 系统提示词](Google/gemini-3-flash.md) · [Gemini 3 Pro](Google/gemini-3-pro.md) |
| Gemini YouTube | [Gemini YouTube 系统提示词](Google/gemini-youtube.md) |
| Gemini Diffusion | [Gemini Diffusion 系统提示词](Google/gemini-diffusion.md) |
| Chrome 中的 Gemini | [Chrome 中的 Gemini 系统提示词](Google/gemini-in-chrome.md) |
| Gemini Workspace | [Gemini Workspace 系统提示词](Google/gemini-workspace.md) |


<details><summary>旧模型和变体</summary>

| | |
|--|--|
| Gemini 2.5 Pro | [API](Google/gemini-2.5-pro-api.md) · [网页版](Google/gemini-2.5-pro-webapp.md) · [引导学习](Google/gemini-2.5-pro-guided-learning.md) |
| Gemini 2.5 Flash | [图片预览](Google/gemini-2.5-flash-image-preview.md) |
| Gemini 2.0 Flash | [网页版](Google/gemini-2.0-flash-webapp.md) |

</details>

## xAI — Grok 系统提示词

| 模型 | 提示词 |
|-------|--------|
| **Grok 4.5** | [**Grok 4.5 系统提示词**](xAI/grok-4.5.md) |
| **Grok Build** | [**Grok Build 系统提示词**（CLI Agent）](xAI/grok-build.md) |
| **Grok 4.3 Beta** | [Grok 4.3 Beta 系统提示词](xAI/grok-4.3-beta.md) |
| **Grok 4.2** | [**Grok 4.2 系统提示词**](xAI/grok-4.2.md) |
| Grok Expert | [Grok Expert 系统提示词](xAI/grok-expert.md) |

<details><summary>旧版本</summary>

| | |
|--|--|
| Grok 4.1 Beta | [Grok 4.1 Beta 系统提示词](xAI/grok-4.1-beta.md) |
| Grok 4 | [Grok 4 系统提示词](xAI/grok-4.md) · [API](xAI/grok-api.md) |
| Grok 3 | [Grok 3 系统提示词](xAI/grok-3.md) |
| Grok Account | [Grok 账户系统提示词](xAI/grok-account.md) |
| Grok 人格 | [Grok 人格提示词](xAI/grok-personas.md) |
| 安全指令 | [Grok 安全指令](xAI/grok.com-post-new-safety-instructions.md) |

</details>

## Perplexity 系统提示词

| 模型 | 提示词 |
|-------|--------|
| **Perplexity** | [**Perplexity AI 系统提示词**](Perplexity/perplexity-ai.md) |
| **Perplexity Computer** | [**Perplexity Computer 系统提示词**](Perplexity/perplexity-computer.md) |
| **Deep Research** | [**Perplexity Deep Research 系统提示词**](Perplexity/deep-research.md) |
| Comet 浏览器 | [Comet 浏览器助手系统提示词](Perplexity/comet-browser-assistant.md) |
| 语音助手 | [Perplexity 语音助手系统提示词](Perplexity/voice-assistant.md) |

## Microsoft — Copilot 系统提示词

| 产品 | 提示词 |
|---------|--------|
| GitHub Copilot | [GitHub Copilot 系统提示词](Microsoft/github-copilot.md) |
| VS Code Copilot Agent | [VS Code Copilot Agent 系统提示词](Microsoft/vscode-copilot-agent.md) |
| Copilot CLI | [Copilot CLI 系统提示词](Microsoft/copilot-cli.md) |
| **Copilot for macOS（应用）** | [**Copilot for macOS 系统提示词**](Microsoft/copilot-macos-app.md) |
| Copilot in Word | [Copilot in Word 系统提示词](Microsoft/copilot-in-microsoft-word.md) |

## Cursor 系统提示词

| 产品 | 提示词 |
|---------|--------|
| Cursor | [Cursor 系统提示词](Cursor/cursor.md) |

## Meta AI 系统提示词

| 产品 | 提示词 |
|---------|--------|
| Meta AI | [Meta AI Muse Spark 系统提示词](Meta/meta-spark.md) · [Muse Spark 1.1](Meta/muse-spark-1.1.md) |

## Mistral 系统提示词

| 产品 | 提示词 |
|---------|--------|
| Mistral Medium 3.5 (Vibe) | [Mistral Medium 3.5 系统提示词](Mistral/mistral-medium-3.5.md) |
| Mistral Code | [Mistral Code 系统提示词](Mistral/mistral-code.md) |

## Moonshot — Kimi 系统提示词

| 模型 | 提示词 |
|-------|--------|
| **Kimi K2.6** | [**Kimi K2.6 系统提示词**](Kimi/kimi-2.6.md) |

## DeepSeek

| 产品 | 提示词 |
|---------|--------|
| **DeepSeek** | [**DeepSeek 系统提示词**](DeepSeek/deepseek-chat.md)（chat.deepseek.com） |

## Z.ai — GLM

| 产品 | 提示词 |
|---------|--------|
| GLM | [GLM 不提供系统提示词——已验证并记录](GLM/README.md) |

## OpenCode 系统提示词

| 产品 | 提示词 |
|---------|--------|
| **OpenCode** | [**OpenCode 系统提示词**](OpenCode/opencode.md) · [2026 年 5 月捕获](Misc/opencode.md) |

## Pi 系统提示词

| 产品 | 提示词 |
|---------|--------|
| Pi (Inflection) | [Pi 系统提示词](Pi/instructions.md) |

## Notion AI 系统提示词

| 产品 | 提示词 |
|---------|--------|
| Notion AI | [Notion AI 系统提示词](Notion/notion-ai.md) |

## Qwen 系统提示词

| 产品 | 提示词 |
|---------|--------|
| Qwen 3.6 Plus | [Qwen 3.6 Plus 系统提示词](Qwen/qwen-3.6-plus.md) |

## 其他系统提示词

| 产品 | 提示词 |
|---------|--------|
| Amp Code (Sourcegraph) | [Amp Code 系统提示词](Misc/amp-code.md) |
| CommandCode CLI | [CommandCode CLI 系统提示词](Misc/commandcode-cli.md) |
| Devin CLI | [Devin CLI 系统提示词](Misc/devin-cli.md) |
| Docker Gordon AI | [Docker Gordon AI 系统提示词](Misc/docker-gordon-ai.md) |
| ElevenLabs Voice Agent | [ElevenLabs 语音代理系统提示词](Misc/elevenlabs-voice-agent.md) |
| Reddit Answers | [Reddit Answers 系统提示词](Misc/reddit-answers.md) |
| Warp 2.0 Agent | [Warp 2.0 Agent 系统提示词](Misc/warp-2.0-agent.md) |
| Zed AI | [Zed AI 系统提示词](Misc/zed.md) |

<details><summary>更多产品</summary>

| | |
|--|--|
| Brave Search | [Brave Search 系统提示词](Misc/brave-search.md) |
| Character AI | [Character AI 系统提示词](Misc/character-ai.md) |
| Confer | [Confer 系统提示词](Misc/confer.md) |
| Fellou Browser | [Fellou 浏览器系统提示词](Misc/fellou-browser.md) |
| Gizmo AI | [Gizmo AI 系统提示词](Misc/gizmo-ai.md) |
| Hermes | [Hermes 系统提示词](Misc/hermes.md) |
| Indus AI | [Indus AI 系统提示词](Misc/indus-ai.md) |
| Kagi Assistant | [Kagi 助手系统提示词](Misc/kagi-assistant.md) |
| MiniMax M2.5 | [MiniMax M2.5 系统提示词](Misc/minimax-m2.5.md) |
| Proton Lumo AI | [Proton Lumo AI 系统提示词](Misc/proton-lumo-ai.md) |
| Raycast AI | [Raycast AI 系统提示词](Misc/raycast-ai.md) |
| Sesame AI Maya | [Sesame AI Maya 系统提示词](Misc/sesame-ai-maya.md) |
| Stack Overflow AI Assist | [Stack Overflow AI Assist 系统提示词](Misc/stack-overflow-ai-assist.md) |
| t3.chat | [t3.chat 系统提示词](Misc/t3.chat.md) |
| t3 Code | [t3 Code 系统提示词](Misc/t3-code.md) |

</details>

---


## 联系方式

![a](https://badgen.net/email/asgeirtj/gmail.com)
[![X](https://img.shields.io/badge/@asgeirtj-black?logo=x&logoColor=white)](https://x.com/asgeirtj)


## Star 历史

<a href="https://www.star-history.com/?repos=asgeirtj%2Fsystem_prompts_leaks&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=asgeirtj/system_prompts_leaks&type=date&theme=dark&legend=top-left&sealed_token=EQ-O807pj1bSPYgKyA5jLwS5T2bqfW3b8ADNsSmVECobESl058V8OkfYQ0S0iG1iCfTLZwuDzaDNNTZ0SOb4rS8oXX-si3kZKlwgOoECQXqY0JrYhqCVdz2itd0pUv5fd-sVr5lbitvclGw1dS_piRTxiCLIDJGlJIWef3qXc8ZDE6zlhIiLbi56yv_e" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=asgeirtj/system_prompts_leaks&type=date&legend=top-left&sealed_token=EQ-O807pj1bSPYgKyA5jLwS5T2bqfW3b8ADNsSmVECobESl058V8OkfYQ0S0iG1iCfTLZwuDzaDNNTZ0SOb4rS8oXX-si3kZKlwgOoECQXqY0JrYhqCVdz2itd0pUv5fd-sVr5lbitvclGw1dS_piRTxiCLIDJGlJIWef3qXc8ZDE6zlhIiLbi56yv_e" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=asgeirtj/system_prompts_leaks&type=date&legend=top-left&sealed_token=EQ-O807pj1bSPYgKyA5jLwS5T2bqfW3b8ADNsSmVECobESl058V8OkfYQ0S0iG1iCfTLZwuDzaDNNTZ0SOb4rS8oXX-si3kZKlwgOoECQXqY0JrYhqCVdz2itd0pUv5fd-sVr5lbitvclGw1dS_piRTxiCLIDJGlJIWef3qXc8ZDE6zlhIiLbi56yv_e" />
 </picture>
</a>


<p align="center">
<a href="https://trendshift.io/repositories/14577" target="_blank"><img src="https://trendshift.io/api/badge/repositories/14577" alt="asgeirtj%2Fsystem_prompts_leaks | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>
 <a href="https://www.star-history.com/asgeirtj/system_prompts_leaks">
  <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/badge?repo=asgeirtj/system_prompts_leaks&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/badge?repo=asgeirtj/system_prompts_leaks" />
   <img alt="Star History Rank" src="https://api.star-history.com/badge?repo=asgeirtj/system_prompts_leaks" />
  </picture>
 </a>
</p>

<img alt="Claude 确认提取的系统提示词是真实的" src="https://github.com/user-attachments/assets/444e3fcc-9374-4964-afd3-069222713dc0" />
