---
name: design-system
description: 使用此技能生成或审计设计系统、检查视觉一致性，以及审查涉及样式的 PR。
origin: ECC
---

# 设计系统 — 生成与审计视觉体系

## 何时使用

- 启动需要设计系统的新项目
- 审计现有代码库的视觉一致性
- 重新设计之前 — 了解现有状态
- 当 UI 看起来"不对劲"但无法确定原因时
- 审查涉及样式的 PR

## 工作原理

### 模式一：生成设计系统

分析代码库并生成统一的设计系统：

```
1. Scan CSS/Tailwind/styled-components for existing patterns
2. Extract: colors, typography, spacing, border-radius, shadows, breakpoints
3. Research 3 competitor sites for inspiration (via browser MCP)
4. Propose a design token set (JSON + CSS custom properties)
5. Generate DESIGN.md with rationale for each decision
6. Create an interactive HTML preview page (self-contained, no deps)
```

输出：`DESIGN.md` + `design-tokens.json` + `design-preview.html`

### 模式二：视觉审计

从 10 个维度为 UI 评分（每项 0-10 分）：

```
1. Color consistency — are you using your palette or random hex values?
2. Typography hierarchy — clear h1 > h2 > h3 > body > caption?
3. Spacing rhythm — consistent scale (4px/8px/16px) or arbitrary?
4. Component consistency — do similar elements look similar?
5. Responsive behavior — fluid or broken at breakpoints?
6. Dark mode — complete or half-done?
7. Animation — purposeful or gratuitous?
8. Accessibility — contrast ratios, focus states, touch targets
9. Information density — cluttered or clean?
10. Polish — hover states, transitions, loading states, empty states
```

每个维度都会给出分数、具体示例，以及包含精确文件行号的修复建议。

### 模式三：AI 俗套检测

识别常见的 AI 生成设计模式：

```
- Gratuitous gradients on everything
- Purple-to-blue defaults
- "Glass morphism" cards with no purpose
- Rounded corners on things that shouldn't be rounded
- Excessive animations on scroll
- Generic hero with centered text over stock gradient
- Sans-serif font stack with no personality
```

## 示例

**为 SaaS 应用生成设计系统：**
```
/design-system generate --style minimal --palette earth-tones
```

**审计现有 UI：**
```
/design-system audit --url http://localhost:3000 --pages / /pricing /docs
```

**检测 AI 俗套：**
```
/design-system slop-check
```
