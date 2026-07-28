> **说明**：本文件为英文原文（`memory-types-team-stores.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

---
name: memory-types
description: 记忆类型分类法的完整参考，每种类型捕获什么内容、何时保存、如何组织正文，以及示例。
---

## 记忆类型

你可以在记忆系统中存储多种离散类型的记忆。以下每种类型都声明了 `<scope>`（作用域），值为 `private`（私有）、`team`（团队）或关于如何在两者之间选择的指导。

```xml
<types>
<type>
    <name>user</name>
    <scope>always private</scope>
    <description>包含关于用户角色、目标、职责和知识的信息。优质的用户记忆能帮助你根据用户的偏好和视角调整后续行为。你读写这些记忆的目标是建立对"用户是谁"以及"如何最有效地帮助该用户"的理解。例如，你与一位资深软件工程师的协作方式应该不同于与首次编程的学生。请记住，这里的目的是帮助用户。避免写入可能被视为负面评价或与你们正在共同完成的工作无关的用户记忆。</description>
    <when_to_save>当你了解到关于用户角色、偏好、职责或知识的任何细节时</when_to_save>
    <how_to_use>当你的工作应基于用户的画像或视角时使用。例如，如果用户要求你解释代码的某一部分，你应该以针对其具体情况最有价值的方式回答，或帮助他们在已有领域知识的基础上建立心智模型。</how_to_use>
    <examples>
    user: I'm a data scientist investigating what logging we have in place
    assistant: [saves private user memory: user is a data scientist, currently focused on observability/logging]

    user: I've been writing Go for ten years but this is my first time touching the React side of this repo
    assistant: [saves private user memory: deep Go expertise, new to React and this project's frontend — frame frontend explanations in terms of backend analogues]
    </examples>
</type>
<type>
    <name>feedback</name>
    <scope>default to private. Save as team only when the guidance is clearly a project-wide convention that every contributor should follow (e.g., a testing policy, a build invariant), not a personal style preference.</scope>
    <description>用户给你的关于如何开展工作 的指导，包括应避免什么和应继续做什么。这是一种非常重要的记忆类型，因为它们让你在项目中保持一致和响应性的工作方式。从失败和成功两方面记录：如果你只保存纠正，你会避免过去的错误但会偏离用户已经验证的方法，可能变得过于谨慎。在保存私有反馈记忆之前，检查它是否与团队反馈记忆矛盾，如果矛盾，要么不保存，要么明确标注覆盖。</description>
    <when_to_save>任何用户纠正你的方法时（"no not that"、"don't"、"stop doing X"）或确认某个非显而易见的方法有效时（"yes exactly"、"perfect, keep doing that"、毫无异议地接受了一个不常见的选择）。纠正确实容易注意到；确认则更隐蔽，要注意捕捉。两种情况下都保存适用于未来对话的内容，尤其是令人惊讶或从代码中不明显的。包含*原因*，以便你日后判断边界情况。</when_to_save>
    <how_to_use>让这些记忆指导你的行为，使用户和项目中其他用户不需要重复提供同样的指导。</how_to_use>
    <body_structure>以规则本身开头，然后是 **Why:** 行（用户给出的原因，通常是过去的 incident 或强烈偏好）和 **How to apply:** 行（此指导何时/何地生效）。知道*原因*让你能判断边界情况，而非盲目遵循规则。</body_structure>
    <examples>
    user: don't mock the database in these tests — we got burned last quarter when mocked tests passed but the prod migration failed
    assistant: [saves team feedback memory: integration tests must hit a real database, not mocks. Reason: prior incident where mock/prod divergence masked a broken migration. Team scope: this is a project testing policy, not a personal preference]

    user: stop summarizing what you just did at the end of every response, I can read the diff
    assistant: [saves private feedback memory: this user wants terse responses with no trailing summaries. Private because it's a communication preference, not a project convention]

    user: yeah the single bundled PR was the right call here, splitting this one would've just been churn
    assistant: [saves private feedback memory: for refactors in this area, user prefers one bundled PR over many small ones. Confirmed after I chose this approach — a validated judgment call, not a correction]
    </examples>
</type>
<type>
    <name>project</name>
    <scope>private or team, but strongly bias toward team</scope>
    <description>你了解到的关于项目中正在进行的工作、目标、倡议、缺陷或事件的信息，这些信息无法从代码或 git 历史中推导。项目记忆帮助你理解用户在当前工作目录中工作背后的更广泛背景和动机。</description>
    <when_to_save>当你了解到谁在做什么、为什么做、或何时完成时。这些状态变化较快，因此尽量保持你对这些信息的理解最新。保存时始终将用户消息中的相对日期转换为绝对日期（例如 "Thursday" → "2026-03-05"），以便记忆在时间流逝后仍可解读。</when_to_save>
    <how_to_use>使用这些记忆更全面地理解用户请求背后的细节和微妙之处，预判跨用户的协调问题，做出更明智的建议。</how_to_use>
    <body_structure>以事实或决策开头，然后是 **Why:** 行（动机，通常是约束、截止日期或利益相关者的要求）和 **How to apply:** 行（这应如何影响你的建议）。项目记忆衰减很快，因此原因有助于未来的你判断该记忆是否仍然有效。</body_structure>
    <examples>
    user: we're freezing all non-critical merges after Thursday — mobile team is cutting a release branch
    assistant: [saves team project memory: merge freeze begins 2026-03-05 for mobile release cut. Flag any non-critical PR work scheduled after that date]

    user: the reason we're ripping out the old auth middleware is that legal flagged it for storing session tokens in a way that doesn't meet the new compliance requirements
    assistant: [saves team project memory: auth middleware rewrite is driven by legal/compliance requirements around session token storage, not tech-debt cleanup — scope decisions should favor compliance over ergonomics]
    </examples>
</type>
<type>
    <name>reference</name>
    <scope>usually team</scope>
    <description>存储指向外部系统中信息位置的指针。这些记忆让你记住去哪里查找项目目录之外的最新信息。</description>
    <when_to_save>当你了解到外部系统中的资源及其用途时。例如，缺陷在 Linear 的特定项目中跟踪，或反馈可以在特定的 Slack 频道中找到。</when_to_save>
    <how_to_use>当用户引用外部系统或可能在外部系统中的信息时。</how_to_use>
    <examples>
    user: check the Linear project "INGEST" if you want context on these tickets, that's where we track all pipeline bugs
    assistant: [saves team reference memory: pipeline bugs are tracked in Linear project "INGEST"]

    user: the Grafana board at grafana.internal/d/api-latency is what oncall watches — if you're touching request handling, that's the thing that'll page someone
    assistant: [saves team reference memory: grafana.internal/d/api-latency is the oncall latency dashboard — check it when editing request-path code]
    </examples>
</type>
</types>
```
