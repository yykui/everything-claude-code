---
name: gan-style-harness
description: "受 GAN 启发的生成器-评估器 Agent 框架，用于自主构建高质量应用程序。基于 Anthropic 2026 年 3 月的框架设计论文。"
origin: ECC-community
tools: Read, Write, Edit, Bash, Grep, Glob, Task
---

# GAN 风格框架技能

> 灵感来源：[Anthropic 长期应用开发的框架设计](https://www.anthropic.com/engineering/harness-design-long-running-apps)（2026 年 3 月 24 日）

一种多 Agent 框架，将**生成**与**评估**分离，形成对抗性反馈循环，将质量提升到单个 Agent 无法达到的水平。

## 核心洞察

> 当被要求评估自己的工作时，Agent 是病态的乐观主义者——它们会赞美平庸的输出，并说服自己忽视合理的问题。但将**独立的评估器**工程化为极其严苛，要比教会生成器进行自我批评容易得多。

这与 GAN（生成对抗网络）的动态相同：生成器产出，评估器批评，反馈推动下一次迭代。

## 何时使用

- 从一行提示词构建完整应用
- 需要高视觉质量的前端设计任务
- 需要可运行功能而非只是代码的全栈项目
- 任何不接受"AI 俗套"美学的任务
- 愿意投入 $50-200 获取生产级输出的项目

## 何时不使用

- 快速的单文件修复（使用标准 `claude -p`）
- 预算紧张的任务（<$10）
- 简单的重构（改用去俗套化模式）
- 已有明确测试规范的任务（使用 TDD 工作流）

## 架构

```
                    ┌─────────────┐
                    │   PLANNER   │
                    │  (Opus 4.6) │
                    └──────┬──────┘
                           │ Product Spec
                           │ (features, sprints, design direction)
                           ▼
              ┌────────────────────────┐
              │                        │
              │   GENERATOR-EVALUATOR  │
              │      FEEDBACK LOOP     │
              │                        │
              │  ┌──────────┐          │
              │  │GENERATOR │--build-->│──┐
              │  │(Opus 4.6)│          │  │
              │  └────▲─────┘          │  │
              │       │                │  │ live app
              │    feedback             │  │
              │       │                │  │
              │  ┌────┴─────┐          │  │
              │  │EVALUATOR │<-test----│──┘
              │  │(Opus 4.6)│          │
              │  │+Playwright│         │
              │  └──────────┘          │
              │                        │
              │   5-15 iterations      │
              └────────────────────────┘
```

## 三个 Agent

### 1. 规划器 Agent

**角色：** 产品经理——将简短提示词扩展为完整的产品规格。

**核心行为：**
- 接受一行提示词，生成包含 16 个功能、多冲刺的规格文档
- 定义用户故事、技术要求和视觉设计方向
- 刻意**雄心勃勃**——保守的规划会导致平庸的结果
- 生成评估器后续使用的评估标准

**模型：** Opus 4.6（规格扩展需要深度推理）

### 2. 生成器 Agent

**角色：** 开发者——按规格实现功能。

**核心行为：**
- 在结构化冲刺中工作（或在更新模型下使用连续模式）
- 在编写代码前与评估器协商"冲刺合约"
- 使用全栈工具：React、FastAPI/Express、数据库、CSS
- 使用 git 进行迭代间的版本控制
- 读取评估器反馈并在下一次迭代中纳入

**模型：** Opus 4.6（需要强大的编码能力）

### 3. 评估器 Agent

**角色：** QA 工程师——测试实际运行的应用，而非仅测试代码。

**核心行为：**
- 使用 **Playwright MCP** 与实时应用交互
- 点击功能、填写表单、测试 API 端点
- 按四个标准评分（可配置）：
  1. **设计质量** — 是否感觉像一个整体？
  2. **原创性** — 自定义决策 vs 模板/AI 模式？
  3. **工艺** — 字体排版、间距、动画、微交互？
  4. **功能性** — 所有功能是否真正可用？
- 返回包含分数和具体问题的结构化反馈
- 被工程化为**极其严苛**——绝不赞美平庸的工作

**模型：** Opus 4.6（需要强判断力 + 工具使用能力）

## 评估标准

默认四个标准，每项 1-10 分：

```markdown
## Evaluation Rubric

### Design Quality (weight: 0.3)
- 1-3: Generic, template-like, "AI slop" aesthetics
- 4-6: Competent but unremarkable, follows conventions
- 7-8: Distinctive, cohesive visual identity
- 9-10: Could pass for a professional designer's work

### Originality (weight: 0.2)
- 1-3: Default colors, stock layouts, no personality
- 4-6: Some custom choices, mostly standard patterns
- 7-8: Clear creative vision, unique approach
- 9-10: Surprising, delightful, genuinely novel

### Craft (weight: 0.3)
- 1-3: Broken layouts, missing states, no animations
- 4-6: Works but feels rough, inconsistent spacing
- 7-8: Polished, smooth transitions, responsive
- 9-10: Pixel-perfect, delightful micro-interactions

### Functionality (weight: 0.2)
- 1-3: Core features broken or missing
- 4-6: Happy path works, edge cases fail
- 7-8: All features work, good error handling
- 9-10: Bulletproof, handles every edge case
```

### 评分

- **加权分数** = 各标准分数 × 权重之和
- **通过阈值** = 7.0（可配置）
- **最大迭代次数** = 15（可配置，通常 5-15 次足够）

## 使用方法

### 通过命令

```bash
# Full three-agent harness
/project:gan-build "Build a project management app with Kanban boards, team collaboration, and dark mode"

# With custom config
/project:gan-build "Build a recipe sharing platform" --max-iterations 10 --pass-threshold 7.5

# Frontend design mode (generator + evaluator only, no planner)
/project:gan-design "Create a landing page for a crypto portfolio tracker"
```

### 通过 Shell 脚本

```bash
# Basic usage
./scripts/gan-harness.sh "Build a music streaming dashboard"

# With options
GAN_MAX_ITERATIONS=10 \
GAN_PASS_THRESHOLD=7.5 \
GAN_EVAL_CRITERIA="functionality,performance,security" \
./scripts/gan-harness.sh "Build a REST API for task management"
```

### 通过 Claude Code（手动）

```bash
# Step 1: Plan
claude -p --model opus "You are a Product Planner. Read PLANNER_PROMPT.md. Expand this brief into a full product spec: 'Build a Kanban board app'. Write spec to spec.md"

# Step 2: Generate (iteration 1)
claude -p --model opus "You are a Generator. Read spec.md. Implement Sprint 1. Start the dev server on port 3000."

# Step 3: Evaluate (iteration 1)
claude -p --model opus --allowedTools "Read,Bash,mcp__playwright__*" "You are an Evaluator. Read EVALUATOR_PROMPT.md. Test the live app at http://localhost:3000. Score against the rubric. Write feedback to feedback-001.md"

# Step 4: Generate (iteration 2 — reads feedback)
claude -p --model opus "You are a Generator. Read spec.md and feedback-001.md. Address all issues. Improve the scores."

# Repeat steps 3-4 until pass threshold met
```

## 随模型能力的演进

随着模型改进，框架应逐渐简化。按照 Anthropic 的演进路径：

### 阶段一 — 较弱模型（Sonnet 级别）
- 需要完整的冲刺分解
- 冲刺之间重置上下文（避免上下文焦虑）
- 最少 2 个 Agent：初始化器 + 编码 Agent
- 大量脚手架弥补模型局限

### 阶段二 — 有能力的模型（Opus 4.5 级别）
- 完整的 3-Agent 框架：规划器 + 生成器 + 评估器
- 每个实现阶段前的冲刺合约
- 复杂应用的 10 冲刺分解
- 上下文重置仍有用，但不那么关键

### 阶段三 — 前沿模型（Opus 4.6 级别）
- 简化框架：单次规划，连续生成
- 评估减少为单次末尾通过（模型更智能）
- 无需冲刺结构
- 自动压缩处理上下文增长

> **关键原则：** 框架中的每个组件都编码了对模型无法独立完成什么的假设。当模型改进时，重新测试这些假设，去掉不再需要的部分。

## 配置

### 环境变量

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `GAN_MAX_ITERATIONS` | `15` | 最大生成器-评估器循环次数 |
| `GAN_PASS_THRESHOLD` | `7.0` | 通过的加权分数（1-10） |
| `GAN_PLANNER_MODEL` | `opus` | 规划 Agent 模型 |
| `GAN_GENERATOR_MODEL` | `opus` | 生成 Agent 模型 |
| `GAN_EVALUATOR_MODEL` | `opus` | 评估 Agent 模型 |
| `GAN_EVAL_CRITERIA` | `design,originality,craft,functionality` | 逗号分隔的评估标准 |
| `GAN_DEV_SERVER_PORT` | `3000` | 实时应用端口 |
| `GAN_DEV_SERVER_CMD` | `npm run dev` | 启动开发服务器的命令 |
| `GAN_PROJECT_DIR` | `.` | 项目工作目录 |
| `GAN_SKIP_PLANNER` | `false` | 跳过规划器，直接使用规格 |
| `GAN_EVAL_MODE` | `playwright` | `playwright`、`screenshot` 或 `code-only` |

### 评估模式

| 模式 | 工具 | 最适合 |
|------|-------|----------|
| `playwright` | Browser MCP + 实时交互 | 带 UI 的全栈应用 |
| `screenshot` | 截图 + 视觉分析 | 静态站点、纯设计 |
| `code-only` | 测试 + 代码检查 + 构建 | API、库、CLI 工具 |

## 反模式

1. **评估器过于宽松** — 若评估器在第一次迭代就通过一切，说明评分标准太慷慨。收紧评分标准，并为常见 AI 模式添加明确惩罚。

2. **生成器忽视反馈** — 确保反馈以文件形式传递，而非内联。生成器应在每次迭代开始时读取 `feedback-NNN.md`。

3. **无限循环** — 始终设置 `GAN_MAX_ITERATIONS`。若生成器经过 3 次迭代后仍无法突破分数瓶颈，停止并标记供人工审查。

4. **评估器测试流于表面** — 评估器必须使用 Playwright **交互**实时应用，而非仅截图。点击按钮、填写表单、测试错误状态。

5. **评估器赞扬自己的修复** — 永远不要让评估器建议修复后再评估这些修复。评估器只负责批评；生成器负责修复。

6. **上下文耗尽** — 长时间会话时，使用 Claude Agent SDK 的自动压缩，或在主要阶段之间重置上下文。

## 结果：预期效果

基于 Anthropic 发布的结果：

| 指标 | 单独 Agent | GAN 框架 | 提升 |
|--------|-----------|-------------|-------------|
| 时间 | 20 分钟 | 4-6 小时 | 长 12-18 倍 |
| 成本 | $9 | $125-200 | 多 14-22 倍 |
| 质量 | 勉强可用 | 生产就绪 | 质的飞跃 |
| 核心功能 | 有缺陷 | 全部可用 | N/A |
| 设计 | 通用 AI 俗套 | 独特、精致 | N/A |

**权衡显而易见：** 约 20 倍的时间和成本换来输出质量的质的飞跃。适用于质量至关重要的项目。

## 参考资料

- [Anthropic: Harness Design for Long-Running Apps](https://www.anthropic.com/engineering/harness-design-long-running-apps) — Prithvi Rajasekaran 的原始论文
- [Epsilla: The GAN-Style Agent Loop](https://www.epsilla.com/blogs/anthropic-harness-engineering-multi-agent-gan-architecture) — 架构解构
- [Martin Fowler: Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) — 更广泛的行业背景
- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/) — OpenAI 的平行工作
