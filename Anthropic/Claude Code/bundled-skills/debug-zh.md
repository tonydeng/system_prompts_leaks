> **说明**：本文件为英文原文（`debug.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以原文为准。占位符（如 `{model_name}`）保持原样不译。

---
name: debug
description: 通过启用调试日志、读取日志和建议修复来调试当前 Claude Code 会话中的问题。
---

# Debug 技能

帮助用户调试他们在当前 Claude Code 会话中遇到的问题。

## 调试日志刚刚启用

此会话的调试日志直到现在都是关闭的。在此 /debug 调用之前没有捕获任何内容。

告诉用户调试日志现在在 `{debug_log_path}` 处处于活动状态，要求他们重现问题，然后重新读取日志。如果他们无法重现，他们也可以使用 `claude --debug` 重启以从启动捕获日志。

## 会话调试日志

当前会话的调试日志位于：`{debug_log_path}`

尚不存在日志文件。

对于额外上下文，在整个文件中 grep [ERROR] 和 [WARN] 行。

## 守护进程

未找到守护进程锁定或状态文件——后台守护进程似乎未运行。如果问题涉及后台会话或 `claude agents`，守护进程日志（如果有）位于 `{user_home}/.claude/daemon.log`。

## 问题描述

用户未描述具体问题。读取调试日志并总结任何错误、警告或值得注意的问题。

## 设置

记住设置位于：
* 用户 - {user_home}/.claude/settings.json
* 项目 - {working_directory}/.claude/settings.json
* 本地 - {working_directory}/.claude/settings.local.json

## 指令

1. 审查用户的问题描述
2. 最后 20 行显示调试文件格式。在整个文件中查找 [ERROR] 和 [WARN] 条目、堆栈跟踪和失败模式
3. 考虑启动 claude-code-guide 子代理以了解相关的 Claude Code 功能
4. 用简单的语言解释你发现的内容
5. 建议具体的修复或下一步