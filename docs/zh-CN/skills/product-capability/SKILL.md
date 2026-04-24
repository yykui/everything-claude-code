---
name: product-capability
description: 将 PRD 意图、路线图需求或产品讨论转化为可落地的能力计划，在多服务工作启动前明确约束条件、不变量、接口和待解决决策。适用于用户需要 ECC 原生 PRD-to-SRS 通道而非模糊规划文字的场景。
origin: ECC
---

# 产品能力

本技能将产品意图转化为明确的工程约束。

当问题不是"我们应该构建什么？"，而是"实现开始前必须满足哪些条件？"时，使用本技能。

## 何时使用

- 存在 PRD、路线图条目、讨论或创始人笔记，但实现约束仍是隐式的
- 某项功能跨越多个服务、仓库或团队，编码前需要能力契约
- 产品意图清晰，但架构、数据、生命周期或策略影响仍不明确
- 高级工程师在评审时不断重申相同的隐含假设
- 需要一个可跨工具和会话复用的制品

## 规范制品

如果仓库存在持久化的产品上下文文件（如 `PRODUCT.md`、`docs/product/` 或程序规范目录），请在那里更新。

如果尚不存在能力清单，请使用以下模板创建：

- `docs/examples/product-capability-template.md`

目标不是创建另一个规划层级，而是让隐藏的能力约束变得持久且可复用。

## 不可妥协的规则

- 不要凭空捏造产品事实。明确标注未解决的问题。
- 将用户可见的承诺与实现细节分离。
- 明确指出哪些是固定策略，哪些是架构偏好，哪些仍是开放问题。
- 如果请求与现有仓库约束冲突，应直接说明，而不是掩盖矛盾。
- 优先使用一个可复用的能力制品，而非分散的临时笔记。

## 输入

仅读取必要内容：

1. 产品意图
   - issue、讨论、PRD、路线图说明、创始人消息
2. 当前架构
   - 相关仓库文档、契约、schema、路由、现有工作流
3. 现有能力上下文
   - `PRODUCT.md`、设计文档、RFC、迁移说明、运营模型文档
4. 交付约束
   - 认证、计费、合规、发布策略、向后兼容性、性能、评审策略

## 核心工作流

### 1. 重述能力

将需求压缩为一句精确的陈述：

- 用户或操作者是谁
- 交付后新增了什么能力
- 由此产生了什么结果变化

如果这句陈述含糊，实现将会偏离方向。

### 2. 解析能力约束

提取实现前必须成立的约束：

- 业务规则
- 范围边界
- 不变量
- 信任边界
- 数据所有权
- 生命周期转换
- 发布/迁移需求
- 失败与恢复预期

这些内容通常只存在于高级工程师的记忆中。

### 3. 定义面向实现的契约

生成 SRS 风格的能力计划，包含：

- 能力摘要
- 明确的非目标
- 参与者与交互界面
- 必需的状态与转换
- 接口/输入/输出
- 数据模型影响
- 安全/计费/策略约束
- 可观测性与运营需求
- 阻塞实现的开放问题

### 4. 转化为执行

以明确的交接作为结束：

- 已准备好直接实现
- 需要先进行架构评审
- 需要先进行产品澄清

如有必要，指向下一个 ECC 原生通道：

- `project-flow-ops`
- `workspace-surface-audit`
- `api-connector-builder`
- `dashboard-builder`
- `tdd-workflow`
- `verification-loop`

## 输出格式

按以下顺序返回结果：

```text
CAPABILITY
- one-paragraph restatement

CONSTRAINTS
- fixed rules, invariants, and boundaries

IMPLEMENTATION CONTRACT
- actors
- surfaces
- states and transitions
- interface/data implications

NON-GOALS
- what this lane explicitly does not own

OPEN QUESTIONS
- blockers or product decisions still required

HANDOFF
- what should happen next and which ECC lane should take it
```

## 良好结果

- 产品意图现在已足够具体，可以直接实现，而无需在 PR 中途重新发现隐藏约束。
- 工程评审拥有持久制品，而非依赖记忆或 Slack 上下文。
- 生成的计划可在 Claude Code、Codex、Cursor、OpenCode 和 ECC 2.0 规划界面中复用。
