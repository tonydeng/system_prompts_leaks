> **说明**：本文件为英文原文（`handoff-to-claude-code.md`）的中文译注版，辅助理解。英文原文为权威来源，任何冲突以英文原文为准。

# 移交给 Claude Code

创建一个全面的交接包，使使用 Claude Code 的开发者能够在真实代码库中实现此设计。

## 步骤

1. **在项目目录中创建交接文件夹**：
   ```
   mkdir -p <project-folder>/design_handoff_<feature-name>/
   ```
   使用从设计中派生的描述性功能名称（如 `design_handoff_onboarding_flow`、`design_handoff_settings_redesign`）。

2. **在交接文件夹中创建 README.md**，包含以下部分：

### README.md 结构

```markdown
# Handoff: <Feature Name>

## Overview
Brief description of what this design is for and what it accomplishes.

## About the Design Files
State clearly that the files in this bundle are **design references created in HTML** — prototypes showing intended look and behavior, not production code to copy directly. Explain that the task is to **recreate these HTML designs in the target codebase's existing environment** (React, Vue, SwiftUI, native, etc.) using its established patterns and libraries — or, if no environment exists yet, to choose the most appropriate framework for the project and implement the designs there.

## Fidelity
State clearly whether the mocks/prototypes created in this conversation are:
- **High-fidelity (hifi)**: Pixel-perfect mockups with final colors, typography, spacing, and interactions. The developer should recreate the UI pixel-perfectly using the codebase's existing libraries and patterns.
- **Low-fidelity (lofi)**: Wireframes or rough layouts showing structure and flow. The developer should use these as a guide for layout and functionality but apply the codebase's existing design system for styling.

## Screens / Views
For each screen or view in the design:
- **Name**: What this screen is called
- **Purpose**: What the user does here
- **Layout**: Detailed description of the layout (grid structure, flex directions, widths, heights, margins, padding)
- **Components**: List each UI component with:
  - Position and size
  - Colors (exact hex values if hifi)
  - Typography (font family, size, weight, line-height, letter-spacing)
  - Border radius, shadows, borders
  - Hover/active/focus states
  - Content/copy (exact text used)

## Interactions & Behavior
- Click handlers and navigation flows
- Animations and transitions (duration, easing, properties)
- Hover states
- Loading states
- Error states
- Form validation rules
- Responsive behavior (if applicable)

## State Management
- What state variables are needed
- State transitions and their triggers
- Any data fetching requirements

## Design Tokens
List all design values used:
- Colors (with hex values)
- Spacing scale
- Typography scale
- Border radius values
- Shadow values

## Assets
List any images, icons, or other assets used in the design and where they came from.

## Files
List the HTML/CSS/JS files in the project that contain the design, so the developer can reference them.
```

3. **将相关设计文件复制到**交接文件夹中（HTML 原型、任何组件文件等）

4. **使用 `present_fs_item_for_download` 工具**，传入交接文件夹路径，以便用户将其作为 zip 下载。

## 重要说明

- 对尺寸、颜色和排版要极其精确，开发者将依赖此文档
- 确保 README 开头声明打包的 HTML 文件是**设计参考**，用户描述的行为应理解为在目标应用的现有环境中重新创建这些设计（如果没有现有环境则选择最合适的框架），而非直接发布 HTML
- 如果设计使用了 Anthropic 品牌资产，提及他们应在代码库中使用现有的品牌系统
- 创建后，询问用户是否需要包含设计的截图。默认不包含。
- README 应自足，未参与本次对话的开发者应能仅凭 README 实现该设计
