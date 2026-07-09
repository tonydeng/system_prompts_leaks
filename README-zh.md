> **据《华盛顿邮报》报道：** [揭开 AI 背后的隐藏规则，然后用它们重写这篇文章。](https://wapo.st/49t4gSb)（2026 年 5 月 11 日）

# 系统提示词泄露

本仓库旨在记录所有 AI 聊天机器人的系统提示词指令——包括 Claude、ChatGPT、Gemini 等。

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/banner-dark.png">
  <source media="(prefers-color-scheme: light)" srcset=".github/banner-light.png">
  <img alt="ChatGPT 在被要求重复上述所有内容后泄露了其系统提示词" src=".github/banner-light.png">
</picture>

![最后提交](https://img.shields.io/github/last-commit/asgeirtj/system_prompts_leaks?style=flat)
[![欢迎 PR](https://img.shields.io/badge/PRs-welcome-brightgreen)](http://makeapullrequest.com)

> 🆕 **[差异对比：Claude Opus 4.8 → Claude Fable 5](https://www.diffchecker.com/QJn9jFNk/)** —— 查看 Anthropic 最新模型的 claude.ai 系统提示词具体变化

## 最近更新

| 内容 | 日期 | 链接 |
|------|------|------|
| **Claude Sonnet 5** | 2026 年 7 月 1 日 | [系统提示词](Anthropic/claude-sonnet-5.md) |
| **Claude Design（Opus 4.8 — 完整提示词 + 48 个工具 + 16 个技能 + 9 个起始来源）** | 2026 年 6 月 26 日 | [系统提示词](Anthropic/claude-design.md) |
| **GitHub Copilot for macOS（应用）** | 2026 年 6 月 18 日 | [系统提示词](Microsoft/copilot-macos-app.md) |
| **GPT-5.5 Codex（完整提示词）** | 2026 年 6 月 18 日 | [系统提示词](OpenAI/Codex/gpt-5.5.md) |
| **Claude Fable 5** | 2026 年 6 月 9 日 | [系统提示词](Anthropic/claude-fable-5.md) · [与 Opus 4.8 对比](https://www.diffchecker.com/QJn9jFNk/) |
| **Claude Opus 4.8** | 2026 年 6 月 9 日 | [系统提示词](Anthropic/claude-opus-4.8.md) · [官方版](Anthropic/Official/2026-05-28-claude-opus-4.8.md) |
| **Claude Code Glob 和 Grep 工具** | 2026 年 6 月 9 日 | [Glob](Anthropic/Claude%20Code/glob-tool.md) · [Grep](Anthropic/Claude%20Code/grep-tool.md) |
| **Claude Code（Opus 4.8）** | 2026 年 5 月 28 日 | [系统提示词](Anthropic/Claude%20Code/claude-code-opus-4.8.md) |
| **Claude Code 和 Cowork** | 2026 年 5 月 28 日 | [Claude Code](Anthropic/Claude%20Code/claude-code-opus-4.6.md) · [Cowork](Anthropic/claude-cowork.md) · [Cowork Dispatch](Anthropic/claude-cowork-dispatch.md) |
| **GPT-5.5** | 2026 年 5 月 24 日 | [Thinking](OpenAI/gpt-5.5-thinking.md) · [Instant](OpenAI/gpt-5.5-instant.md) · [API](OpenAI/gpt-5.5-api.md) · [Pro API](OpenAI/gpt-5.5-pro-api.md) |
| **Perplexity Computer** | 2026 年 5 月 21 日 | [系统提示词](Perplexity/perplexity-computer.md) |
| **VS Code Copilot Agent** | 2026 年 5 月 21 日 | [系统提示词](Microsoft/vscode-copilot-agent.md) |
| **Docker Gordon AI** | 2026 年 5 月 21 日 | [系统提示词](Misc/docker-gordon-ai.md) |
| **Gemini 3.5 Flash** | 2026 年 5 月 20 日 | [系统提示词](Google/gemini-3.5-flash.md) · [AI Studio](Google/gemini-3.5-flash-ai-studio.md) · [工具](Google/gemini-3.5-flash-tools.json) |
| **Antigravity CLI** | 2026 年 5 月 20 日 | [系统提示词](Google/antigravity-cli.md) |
| **Zed AI** | 2026 年 5 月 16 日 | [系统提示词](Misc/zed.md) |
| **Grok Expert** | 2026 年 5 月 11 日 | [系统提示词](xAI/grok-expert.md) |

---
![Anthropic](https://shieldcn.dev/badge/Anthropic-D97757.svg?logo=anthropic&logoColor=fff&variant=secondary&mode=light)

## Anthropic — Claude

| 模型 | 提示词 |
|-------|--------|
| **Claude Fable 5** | [**系统提示词**](Anthropic/claude-fable-5.md) |
| **Claude Opus 4.8** | [**系统提示词**](Anthropic/claude-opus-4.8.md) |
| **Claude Sonnet 5** | [**系统提示词**](Anthropic/claude-sonnet-5.md) |
| **Claude Code（Opus 4.8）** | [**系统提示词**](Anthropic/Claude%20Code/claude-code-opus-4.8.md) |
| **Claude Opus 4.7** | [**系统提示词**](Anthropic/claude-opus-4.7.md) |
| **Claude Code（Opus 4.6）** | [**系统提示词**](Anthropic/Claude%20Code/claude-code-opus-4.6.md) |
| **Claude Opus 4.6** | [**系统提示词**](Anthropic/claude-opus-4.6.md) |
| **Claude Sonnet 4.6** | [**系统提示词**](Anthropic/claude-sonnet-4.6.md) |
| Claude.ai | [Anthropic 提醒](Anthropic/anthropic_reminders.md) |

<details><summary>集成、官方提示词和旧版本</summary>

| | |
|--|--|
| 集成 | [Cowork](Anthropic/claude-cowork.md) · [Cowork Dispatch](Anthropic/claude-cowork-dispatch.md) · [Desktop Code](Anthropic/claude-desktop-code.md) · [Design](Anthropic/claude-design.md) · [Mobile iOS](Anthropic/claude-mobile-ios.md) · [In Chrome](Anthropic/claude-in-chrome.md) · [For Excel](Anthropic/claude-for-excel.md) · [For Word](Anthropic/claude-for-word.md) · [In PowerPoint](Anthropic/claude-in-powerpoint.md) · [Default Styles](Anthropic/default-styles.md) |
| Claude Code 额外 | [Glob 工具](Anthropic/Claude%20Code/glob-tool.md) · [Grep 工具](Anthropic/Claude%20Code/grep-tool.md) · [Deferred 工具](Anthropic/Claude%20Code/deferred-tools.md) · [文档助手](Anthropic/Claude%20Code/claude-code-docs-assistant.md) · [Bundled skills](Anthropic/Claude%20Code/bundled-skills/) |
| 已发布（`claude_behavior` 以发布日期为准，不更新） | [Opus 4.8](Anthropic/Official/2026-05-28-claude-opus-4.8.md) · [Opus 4.7](Anthropic/Official/2026-04-16-claude-opus-4.7.md) · [Opus 4.6](Anthropic/Official/2026-02-05-claude-opus-4.6.md) · [Sonnet 4.6](Anthropic/Official/2026-02-17-claude-sonnet-4.6.md) · [所有版本](Anthropic/Official/) |
| 无工具版 | [Opus 4.6](Anthropic/claude-opus-4.6-no-tools.md) · [Sonnet 4.6](Anthropic/claude-sonnet-4.6-no-tools.md) |
| 原始提示词 | [Opus 4.6](Anthropic/raw/claude-opus-4.6-raw.md) · [Opus 4.6（无工具）](Anthropic/raw/claude-opus-4.6-no-tools-raw.md) · [Sonnet 4.6](Anthropic/raw/claude-sonnet-4.6-raw.md) · [Sonnet 4.6（无工具）](Anthropic/raw/claude-sonnet-4.6-no-tools-raw.md) |
| 可视化 | [Visualization](Anthropic/visualize.md) |
| Opus 4.5 | [系统提示词](Anthropic/old/claude-opus-4.5.md) |
| Sonnet 4.5 | [系统提示词](Anthropic/old/claude-4.5-sonnet.md) |
| Sonnet 4 | [系统提示词](Anthropic/old/claude-sonnet-4.md) |
| Opus 4.1 Thinking | [系统提示词](Anthropic/old/claude-4.1-opus-thinking.md) |
| Sonnet 3.7 | [系统提示词](Anthropic/old/claude-3.7-sonnet.md) · [含工具](Anthropic/old/claude-3.7-sonnet-w-tools.md) · [完整含工具](Anthropic/old/claude-3.7-full-system-message-with-all-tools.md) · [人类可读](Anthropic/old/claude-3.7-sonnet-full-system-message-humanreadable.md) |

</details>

![OpenAI](https://shieldcn.dev/badge/OpenAI-412991.svg?logo=ri%3ASiOpenai&variant=secondary&mode=light)

## OpenAI — ChatGPT

| 模型 | 提示词 |
|-------|--------|
| **GPT-5.5** | [**Thinking**](OpenAI/gpt-5.5-thinking.md) · [**Instant**](OpenAI/gpt-5.5-instant.md) · [API](OpenAI/gpt-5.5-api.md) · [Pro API](OpenAI/gpt-5.5-pro-api.md) · [**Codex**](OpenAI/Codex/gpt-5.5.md) · [Friendly](OpenAI/Codex/personality_friendly_gpt-5.5.md) · [Pragmatic](OpenAI/Codex/personality_pragmatic_gpt-5.5.md) |
| **GPT-5.4** | [**API**](OpenAI/gpt-5.4-api.md) · [**Thinking**](OpenAI/gpt-5.4-thinking.md) · [**Codex**](OpenAI/Codex/gpt-5.4.md) · [Codex Mini](OpenAI/Codex/gpt-5.4-mini.md) |
| **GPT-5.3** | [**Codex**](OpenAI/Codex/gpt-5.3-codex.md) · [Spark](OpenAI/Codex/gpt-5.3-codex-spark.md) · [Codex API](OpenAI/gpt-5.3-codex-api.md) · [Chat API](OpenAI/gpt-5.3-chat-api.md) · [Instant](OpenAI/gpt-5.3-instant.md) |
| **Codex CLI** | [各模型提示词](OpenAI/Codex/) · [Spark](OpenAI/Codex/gpt-5.3-codex-spark.md) · [Plan mode](OpenAI/Codex/plan_mode.md) · [Personas](OpenAI/Codex/personality_friendly.md) · [Auto-review](OpenAI/Codex/codex-auto-review.md) |
| **工具** | [Web search](OpenAI/tool-web-search.md) · [Deep research](OpenAI/tool-deep-research.md) · [Python](OpenAI/tool-python.md) · [Python code](OpenAI/tool-python-code.md) · [Canvas](OpenAI/tool-canvas-canmore.md) · [Image gen](OpenAI/tool-create-image-image_gen.md) · [Memory](OpenAI/tool-memory-bio.md) · [Advanced memory](OpenAI/tool-advanced-memory.md) · [File search](OpenAI/tool-file_search.md) |
| **策略** | [Image safety](OpenAI/prompt-image-safety-policies.md) · [Automation context](OpenAI/prompt-automation-context.md) |

<details><summary>旧模型和变体</summary>

| | |
|--|--|
| GPT-5.2 | [Mini（免费）](OpenAI/gpt-5.2-mini-free-account.md) · [Thinking](OpenAI/gpt-5.2-thinking.md) · [Codex](OpenAI/Codex/gpt-5.2-codex.md) |
| o4-mini | [系统提示词](OpenAI/o4-mini.md) · [High](OpenAI/o4-mini-high.md) |
| o3 | [系统提示词](OpenAI/o3.md) |
| ChatGPT Atlas | [系统提示词](OpenAI/chatgpt-atlas.md) |
| GPT-5.1 个性 | [Default](OpenAI/gpt-5.1-default.md) · [Friendly](OpenAI/gpt-5.1-friendly.md) · [Professional](OpenAI/gpt-5.1-professional.md) · [Candid](OpenAI/gpt-5.1-candid.md) · [Cynical](OpenAI/gpt-5.1-cynical.md) · [Efficient](OpenAI/gpt-5.1-efficient.md) · [Nerdy](OpenAI/gpt-5.1-nerdy.md) · [Quirky](OpenAI/gpt-5.1-quirky.md) |
| GPT-5 | [Agent mode](OpenAI/chatgpt-gpt-5-agent-mode.md) · [Thinking](OpenAI/gpt-5-thinking.md) · [Cynic](OpenAI/gpt-5-cynic-personality.md) · [Listener](OpenAI/gpt-5-listener-personality.md) · [Nerdy](OpenAI/gpt-5-nerdy-personality.md) · [Robot](OpenAI/gpt-5-robot-personality.md) · [Codex](OpenAI/Codex/gpt-5-codex.md) · [Codex Mini](OpenAI/Codex/gpt-5-codex-mini.md) |
| GPT-4.5 | [系统提示词](OpenAI/gpt-4.5.md) |
| GPT-4.1 | [Full](OpenAI/gpt-4.1.md) · [Mini](OpenAI/gpt-4.1-mini.md) |
| GPT-4o | [系统提示词](OpenAI/gpt-4o.md) · [WhatsApp](OpenAI/gpt-4o-whatsapp.md) · [Advanced voice](OpenAI/gpt-4o-advanced-voice-mode.md) · [Legacy voice](OpenAI/gpt-4o-legacy-voice-mode.md) |
| Monday GPT | [系统提示词](OpenAI/monday-gpt.md) |
| GPT-4o 新个性 | [系统提示词](OpenAI/4o-2025-09-03-new-personality.md) |
| Study and learn | [系统提示词](OpenAI/study-and-learn.md) |
| Image safety policies | [系统提示词](OpenAI/image-safety-policies.md) |
| API 变体 | [GPT-5 reasoning (high)](OpenAI/API/gpt-5-reasoning-effort-high-api.md) · [o3 high](OpenAI/API/o3-high-api.md) · [o3 med](OpenAI/API/o3-medium-api.md) · [o3 low](OpenAI/API/o3-low-api.md) · [o4-mini high](OpenAI/API/o4-mini-high.md) · [o4-mini med](OpenAI/API/o4-mini-medium-api.md) · [o4-mini low](OpenAI/API/o4-mini-low-api.md) |
| 旧 o4-mini | [系统提示词](OpenAI/Old/chatgpt.com-o4-mini.md) |
| Codex（旧版） | [GPT-5](OpenAI/Codex/gpt-5.md) · [GPT-5.1](OpenAI/Codex/gpt-5.1.md) · [GPT-5.1 Codex](OpenAI/Codex/gpt-5.1-codex.md) · [GPT-5.1 Mini](OpenAI/Codex/gpt-5.1-codex-mini.md) · [GPT-5.1 Max](OpenAI/Codex/gpt-5.1-codex-max.md) · [GPT-5.2](OpenAI/Codex/gpt-5.2.md) · [5.2 Friendly](OpenAI/Codex/personality_friendly_gpt-5.2-codex.md) · [5.2 Pragmatic](OpenAI/Codex/personality_pragmatic_gpt-5.2-codex.md) |

</details>

![Google Gemini](https://shieldcn.dev/badge/Google%20Gemini-8E75B2.svg?logo=googlegemini&logoColor=fff&variant=secondary&mode=light)

## Google — Gemini

| 模型 | 提示词 |
|-------|--------|
| **Gemini 3.5 Flash** | [**系统提示词**](Google/gemini-3.5-flash.md) · [AI Studio](Google/gemini-3.5-flash-ai-studio.md) · [工具](Google/gemini-3.5-flash-tools.json) |
| **Gemini 3.1 Pro** | [**系统提示词**](Google/gemini-3.1-pro.md) · [API](Google/gemini-3.1-pro-api.md) |
| Gemini CLI | [系统提示词](Google/gemini-cli.md) |
| Antigravity CLI | [系统提示词](Google/antigravity-cli.md) |
| Jules | [系统提示词](Google/jules.md) |

<details><summary>旧模型和变体</summary>

| | |
|--|--|
| Gemini 3 | [Flash](Google/gemini-3-flash.md) · [Pro](Google/gemini-3-pro.md) |
| Gemini Diffusion | [系统提示词](Google/gemini-diffusion.md) |
| Google Search AI Mode | [系统提示词](Google/google-search-ai-mode.md) |
| Gemini YouTube | [系统提示词](Google/gemini-youtube.md) |
| Gemini in Chrome | [系统提示词](Google/gemini-in-chrome.md) |
| Gemini Workspace | [系统提示词](Google/gemini-workspace.md) |
| Gemini 2.5 Pro | [API](Google/gemini-2.5-pro-api.md) · [Webapp](Google/gemini-2.5-pro-webapp.md) · [Guided learning](Google/gemini-2.5-pro-guided-learning.md) |
| Gemini 2.5 Flash | [Image preview](Google/gemini-2.5-flash-image-preview.md) |
| Gemini 2.0 Flash | [Webapp](Google/gemini-2.0-flash-webapp.md) |
| AI Studio Build | [系统提示词](Google/ai-studio-build.md) |
| Nano / Banana 2 | [系统提示词](Google/nano-banana-2-api.md) |
| NotebookLM | [Chat](Google/notebooklm-chat.md) |

</details>

## xAI — Grok

| 模型 | 提示词 |
|-------|--------|
| **Grok Build** | [**系统提示词（CLI Agent）**](xAI/grok-build.md) |
| **Grok 4.3 Beta** | [系统提示词](xAI/grok-4.3-beta.md) |
| **Grok 4.2** | [**系统提示词**](xAI/grok-4.2.md) |
| Grok Expert | [系统提示词](xAI/grok-expert.md) |

<details><summary>旧版本</summary>

| | |
|--|--|
| Grok 4.1 Beta | [系统提示词](xAI/grok-4.1-beta.md) |
| Grok 4 | [系统提示词](xAI/grok-4.md) · [API](xAI/grok-api.md) |
| Grok 3 | [系统提示词](xAI/grok-3.md) |
| Grok Account | [系统提示词](xAI/grok-account.md) |
| Grok Personas | [Personas](xAI/grok-personas.md) |
| Safety Instructions | [Post-new](xAI/grok.com-post-new-safety-instructions.md) |

</details>

## Perplexity

| 模型 | 提示词 |
|-------|--------|
| **Perplexity Computer** | [**系统提示词**](Perplexity/perplexity-computer.md) |
| Comet Browser | [系统提示词](Perplexity/comet-browser-assistant.md) |
| Voice Assistant | [系统提示词](Perplexity/voice-assistant.md) |

## Microsoft — Copilot

| 产品 | 提示词 |
|---------|--------|
| GitHub Copilot | [系统提示词](Microsoft/github-copilot.md) |
| VS Code Copilot Agent | [系统提示词](Microsoft/vscode-copilot-agent.md) |
| Copilot CLI | [系统提示词](Microsoft/copilot-cli.md) |
| **Copilot for macOS（应用）** | [**系统提示词**](Microsoft/copilot-macos-app.md) |
| Copilot in Word | [系统提示词](Microsoft/copilot-in-microsoft-word.md) |

## Cursor

| 产品 | 提示词 |
|---------|--------|
| Cursor | [系统提示词](Cursor/cursor.md) |

## Meta

| 产品 | 提示词 |
|---------|--------|
| Meta AI | [系统提示词](Meta/meta-ai.md) |

## Mistral

| 产品 | 提示词 |
|---------|--------|
| Le Chat | [系统提示词](Mistral/le-chat.md) |

## Notion

| 产品 | 提示词 |
|---------|--------|
| Notion AI | [系统提示词](Notion/notion-ai.md) |

## Qwen

| 产品 | 提示词 |
|---------|--------|
| Qwen 3.6 Plus | [系统提示词](Qwen/qwen-3.6-plus.md) |

## Misc（其他）

| 产品 | 提示词 |
|---------|--------|
| Amp Code (Sourcegraph) | [系统提示词](Misc/amp-code.md) |
| Docker Gordon AI | [系统提示词](Misc/docker-gordon-ai.md) |
| ElevenLabs Voice Agent | [系统提示词](Misc/elevenlabs-voice-agent.md) |
| OpenCode | [系统提示词](Misc/opencode.md) |
| Reddit Answers | [系统提示词](Misc/reddit-answers.md) |
| Warp 2.0 Agent | [系统提示词](Misc/warp-2.0-agent.md) |
| Zed AI | [系统提示词](Misc/zed.md) |

<details><summary>更多产品</summary>

| | |
|--|--|
| Brave Search | [系统提示词](Misc/brave-search.md) |
| Character AI | [系统提示词](Misc/character-ai.md) |
| Confer | [系统提示词](Misc/confer.md) |
| Fellou Browser | [系统提示词](Misc/fellou-browser.md) |
| Gizmo AI | [系统提示词](Misc/gizmo-ai.md) |
| Hermes | [系统提示词](Misc/hermes.md) |
| Indus AI | [系统提示词](Misc/indus-ai.md) |
| Kagi Assistant | [系统提示词](Misc/kagi-assistant.md) |
| MiniMax M2.5 | [系统提示词](Misc/minimax-m2.5.md) |
| Proton Lumo AI | [系统提示词](Misc/proton-lumo-ai.md) |
| Raycast AI | [系统提示词](Misc/raycast-ai.md) |
| Sesame AI Maya | [系统提示词](Misc/sesame-ai-maya.md) |
| t3.chat | [系统提示词](Misc/t3.chat.md) |
| t3 Code | [系统提示词](Misc/t3-code.md) |

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