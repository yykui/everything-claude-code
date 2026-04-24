---
name: gan-style-harness
description: "受 GAN 启发的生成器-评估器代理架构，用于自主构建高质量应用程序。基于 Anthropic 2026 年 3 月发布的架构设计论文。"
origin: ECC-community
tools: Read, Write, Edit, Bash, Grep, Glob, Task
---

# GAN 风格架构技能

> 灵感来源于 [Anthropic 长期运行应用程序开发的架构设计](https://www.anthropic.com/engineering/harness-design-long-running-apps)（2026 年 3 月 24 日）

一种多代理架构，将**生成**与**评估**分离，形成对抗性反馈循环，驱动质量远超单一代理所能达到的水平。

## 核心洞察

> 当被要求评估自己的工作时，代理是病态的乐观主义者——它们会赞扬平庸的输出，并说服自己忽视真实的问题。但将一个**独立的评估器**设计为无情严格的批评者，比教会生成器进行自我批评要容易得多。

这与 GAN（生成对抗网络）的动态机制相同：生成器生产，评估器批评，反馈驱动下一次迭代。

## 何时使用

- 从一行提示词构建完整应用程序
- 需要高视觉质量的前端设计任务
- 需要可用功能而非仅有代码的全栈项目
- 任何不能接受"AI 风格"输出的任务
- 愿意投入 $50-200 以获得生产级质量的项目

## 不适用场景

- 快速的单文件修复（使用标准 `claude -p`）
- 预算紧张（<$10）的任务
- 简单的重构（改用去滥用化模式）
- 已有测试且规格明确的任务（使用 TDD 工作流）

## 架构

```
                    ┌─────────────┐
                    │   PLANNER   │
                    │  (Opus 4.6) │
                    └──────┬──────┘
                           │ 产品规格
                           │ (功能、迭代周期、设计方向)
                           ▼
              ┌────────────────────────┐
              │                        │
              │   生成器-评估器         │
              │      反馈循环           │
              │                        │
              │  ┌──────────┐          │
              │  │ 生成器   │--构建-->│──┐
              │  │(Opus 4.6)│          │  │
              │  └────▲─────┘          │  │
              │       │                │  │ 运行中的应用
              │    反馈                 │  │
              │       │                │  │
              │  ┌────┴─────┐          │  │
              │  │ 评估器   │<-测试----│──┘
              │  │(Opus 4.6)│          │
              │  │+Playwright│         │
              │  └──────────┘          │
              │                        │
              │   5-15 次迭代          │
              └────────────────────────┘
```

## 三个代理

### 1. 规划器代理

**角色：** 产品经理——将简短提示词扩展为完整的产品规格。

**关键行为：**
- 接收一行提示词，生成包含 16 个功能、多迭代周期的规格
- 定义用户故事、技术需求和视觉设计方向
- 刻意保持**雄心勃勃**——保守的规划会导致平淡的结果
- 生成评估器后续使用的评估标准

**模型：** Opus 4.6（规格扩展需要深度推理）

### 2. 生成器代理

**角色：** 开发者——按规格实现功能。

**关键行为：**
- 以结构化的迭代周期工作（或新型模型使用持续模式）
- 在编写代码前与评估器协商"迭代契约"
- 使用全栈工具：React、FastAPI/Express、数据库、CSS
- 使用 git 管理迭代间的版本控制
- 读取评估器反馈并在下一次迭代中采纳

**模型：** Opus 4.6（需要强大的编码能力）

### 3. 评估器代理

**角色：** QA 工程师——测试运行中的应用，而非仅测试代码。

**关键行为：**
- 使用 **Playwright MCP** 与实时应用交互
- 点击功能、填写表单、测试 API 端点
- 按四项标准评分（可配置）：
  1. **设计质量** — 整体是否感觉统一？
  2. **原创性** — 自定义决策与模板/AI 模式的对比？
  3. **工艺** — 字体、间距、动画、微交互？
  4. **功能性** — 所有功能是否真正可用？
- 返回带评分和具体问题的结构化反馈
- 被设计为**无情严格**——从不赞扬平庸的工作

**模型：** Opus 4.6（需要强大的判断力 + 工具使用）

## 评估标准

默认四项标准，每项 1-10 分：

```markdown
## 评估评分标准

### 设计质量（权重：0.3）
- 1-3：通用、模板化、"AI 风格"美学
- 4-6：胜任但平淡，遵循惯例
- 7-8：独特、统一的视觉标识
- 9-10：可媲美专业设计师的作品

### 原创性（权重：0.2）
- 1-3：默认颜色、库存布局、无个性
- 4-6：一些自定义选择，主要是标准模式
- 7-8：清晰的创意愿景，独特方法
- 9-10：令人惊喜、愉悦，真正新颖

### 工艺（权重：0.3）
- 1-3：布局损坏、缺少状态处理、无动画
- 4-6：可用但感觉粗糙，间距不一致
- 7-8：精致，过渡流畅，响应式
- 9-10：像素完美，令人愉悦的微交互

### 功能性（权重：0.2）
- 1-3：核心功能损坏或缺失
- 4-6：主路径可用，边缘情况失败
- 7-8：所有功能可用，良好的错误处理
- 9-10：坚不可摧，处理每种边缘情况
```

### 评分规则

- **加权评分** = 各项（标准分数 × 权重）之和
- **通过阈值** = 7.0（可配置）
- **最大迭代次数** = 15（可配置，通常 5-15 次已足够）

## 使用方法

### 通过命令

```bash
# 完整三代理架构
/project:gan-build "Build a project management app with Kanban boards, team collaboration, and dark mode"

# 自定义配置
/project:gan-build "Build a recipe sharing platform" --max-iterations 10 --pass-threshold 7.5

# 前端设计模式（仅生成器 + 评估器，无规划器）
/project:gan-design "Create a landing page for a crypto portfolio tracker"
```

### 通过 Shell 脚本

```bash
# 基本用法
./scripts/gan-harness.sh "Build a music streaming dashboard"

# 带选项
GAN_MAX_ITERATIONS=10 \
GAN_PASS_THRESHOLD=7.5 \
GAN_EVAL_CRITERIA="functionality,performance,security" \
./scripts/gan-harness.sh "Build a REST API for task management"
```

### 通过 Claude Code（手动）

```bash
# 步骤 1：规划
claude -p --model opus "You are a Product Planner. Read PLANNER_PROMPT.md. Expand this brief into a full product spec: 'Build a Kanban board app'. Write spec to spec.md"

# 步骤 2：生成（第 1 次迭代）
claude -p --model opus "You are a Generator. Read spec.md. Implement Sprint 1. Start the dev server on port 3000."

# 步骤 3：评估（第 1 次迭代）
claude -p --model opus --allowedTools "Read,Bash,mcp__playwright__*" "You are an Evaluator. Read EVALUATOR_PROMPT.md. Test the live app at http://localhost:3000. Score against the rubric. Write feedback to feedback-001.md"

# 步骤 4：生成（第 2 次迭代——读取反馈）
claude -p --model opus "You are a Generator. Read spec.md and feedback-001.md. Address all issues. Improve the scores."

# 重复步骤 3-4 直至通过阈值
```

## 随模型能力演进

随着模型改进，架构应当简化。遵循 Anthropic 的演进路线：

### 阶段 1 — 较弱模型（Sonnet 级）
- 需要完整的迭代周期分解
- 迭代周期间重置上下文（避免上下文焦虑）
- 最少 2 个代理：初始化器 + 编码代理
- 重度脚手架弥补模型局限性

### 阶段 2 — 有能力的模型（Opus 4.5 级）
- 完整 3 代理架构：规划器 + 生成器 + 评估器
- 每个实现阶段前协商迭代契约
- 复杂应用的 10 迭代周期分解
- 上下文重置仍有用但不那么关键

### 阶段 3 — 前沿模型（Opus 4.6 级）
- 简化架构：单次规划，持续生成
- 评估简化为单次最终评审（模型更智能）
- 无需迭代周期结构
- 自动压缩处理上下文增长

> **核心原则：** 每个架构组件都编码了对模型无法独立完成的事情的假设。当模型改进时，重新测试这些假设。去掉不再需要的部分。

## 配置

### 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `GAN_MAX_ITERATIONS` | `15` | 生成器-评估器最大循环次数 |
| `GAN_PASS_THRESHOLD` | `7.0` | 通过的加权分数（1-10） |
| `GAN_PLANNER_MODEL` | `opus` | 规划代理使用的模型 |
| `GAN_GENERATOR_MODEL` | `opus` | 生成代理使用的模型 |
| `GAN_EVALUATOR_MODEL` | `opus` | 评估代理使用的模型 |
| `GAN_EVAL_CRITERIA` | `design,originality,craft,functionality` | 逗号分隔的评估标准 |
| `GAN_DEV_SERVER_PORT` | `3000` | 运行应用的端口 |
| `GAN_DEV_SERVER_CMD` | `npm run dev` | 启动开发服务器的命令 |
| `GAN_PROJECT_DIR` | `.` | 项目工作目录 |
| `GAN_SKIP_PLANNER` | `false` | 跳过规划器，直接使用规格 |
| `GAN_EVAL_MODE` | `playwright` | `playwright`、`screenshot` 或 `code-only` |

### 评估模式

| 模式 | 工具 | 最适合 |
|------|-------|----------|
| `playwright` | 浏览器 MCP + 实时交互 | 带 UI 的全栈应用 |
| `screenshot` | 截图 + 视觉分析 | 静态网站、纯设计 |
| `code-only` | 测试 + Lint + 构建 | API、库、CLI 工具 |

## 反模式

1. **评估器过于宽松** — 如果评估器在第 1 次迭代就通过一切，说明你的评分标准太宽松了。收紧评分标准，并为常见 AI 模式添加明确的扣分项。

2. **生成器忽略反馈** — 确保反馈以文件形式传递，而非内联传入。生成器应在每次迭代开始时读取 `feedback-NNN.md`。

3. **无限循环** — 始终设置 `GAN_MAX_ITERATIONS`。如果生成器在 3 次迭代后仍无法突破评分瓶颈，停止并标记为人工审查。

4. **评估器测试过于表面** — 评估器必须使用 Playwright **与**实时应用**交互**，而非仅截图。点击按钮、填写表单、测试错误状态。

5. **评估器评估自己建议的修复** — 永远不要让评估器既提出修复建议又评估这些修复。评估器只批评，生成器负责修复。

6. **上下文耗尽** — 对于长时间会话，使用 Claude Agent SDK 的自动压缩功能，或在主要阶段间重置上下文。

## 预期结果

基于 Anthropic 公布的结果：

| 指标 | 单一代理 | GAN 架构 | 提升 |
|--------|-----------|-------------|-------------|
| 时间 | 20 分钟 | 4-6 小时 | 慢 12-18 倍 |
| 成本 | $9 | $125-200 | 贵 14-22 倍 |
| 质量 | 勉强可用 | 生产就绪 | 质的飞跃 |
| 核心功能 | 有缺陷 | 全部可用 | N/A |
| 设计 | 通用 AI 风格 | 独特、精致 | N/A |

**权衡取舍很明显：** 以约 20 倍的时间和成本换取输出质量的质的提升。适用于质量至关重要的项目。

## 参考资料

- [Anthropic：长期运行应用程序的架构设计](https://www.anthropic.com/engineering/harness-design-long-running-apps) — Prithvi Rajasekaran 的原始论文
- [Epsilla：GAN 风格代理循环](https://www.epsilla.com/blogs/anthropic-harness-engineering-multi-agent-gan-architecture) — 架构解构
- [Martin Fowler：架构工程](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) — 更广泛的行业背景
- [OpenAI：架构工程](https://openai.com/index/harness-engineering/) — OpenAI 的并行工作
