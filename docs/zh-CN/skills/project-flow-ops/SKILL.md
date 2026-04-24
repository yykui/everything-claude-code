---
name: project-flow-ops
description: 通过对 issue 和 PR 进行分类、关联活跃工作，在 GitHub 和 Linear 之间运营执行流程，保持 GitHub 对外公开，同时 Linear 作为内部执行层。适用于用户需要积压工作管控、PR 分类或 GitHub-to-Linear 协同的场景。
origin: ECC
---

# 项目流程运营

本技能将分散的 GitHub issue、PR 和 Linear 任务整合为统一的执行流程。

当问题是协调而非编码时，使用本技能。

## 何时使用

- 对开放 PR 或 issue 积压进行分类
- 决定哪些工作属于 Linear，哪些应保留在 GitHub
- 将活跃的 GitHub 工作与内部执行通道关联
- 将 PR 分类为合并、移植/重建、关闭或暂存
- 审查评论、CI 失败或过期 issue 是否阻塞了执行

## 运营模型

- **GitHub** 是对外公开和社区的事实来源
- **Linear** 是活跃计划工作的内部执行事实来源
- 并非每个 GitHub issue 都需要创建 Linear issue
- 仅在以下情况下创建或更新 Linear：
  - 工作处于活跃状态
  - 已委派
  - 已排期
  - 跨职能
  - 重要到需要内部追踪

## 核心工作流

### 1. 先读取公开界面

收集：

- GitHub issue 或 PR 状态
- 作者和分支状态
- 评审评论
- CI 状态
- 关联的 issue

### 2. 对工作进行分类

每个条目应落入以下状态之一：

| 状态 | 含义 |
|-------|---------|
| 合并 | 自包含、符合策略、已就绪 |
| 移植/重建 | 有价值的想法，但应在 ECC 内手动重新落地 |
| 关闭 | 方向错误、已过期、不安全或重复 |
| 暂存 | 可能有用，但当前未排期 |

### 3. 决定是否需要 Linear

仅在以下情况下创建或更新 Linear：

- 执行已主动计划
- 涉及多个仓库或工作流
- 工作需要内部所有者或排序
- issue 是更大项目通道的一部分

不要机械地镜像所有内容。

### 4. 保持两个系统一致

当工作处于活跃状态时：

- GitHub issue/PR 应公开说明正在发生什么
- Linear 应在内部追踪负责人、优先级和执行通道

当工作完成或被拒绝时：

- 将公开解决结果回写到 GitHub
- 相应地标记 Linear 任务

## 评审规则

- 永远不要仅凭标题、摘要或信任合并；要使用完整的 diff
- 外部来源的功能如果有价值但不自包含，应在 ECC 内重建
- CI 红灯意味着分类并修复或阻塞；不要假装已就绪
- 如果真正的阻塞点是产品方向，要直接说明而非掩盖在工具问题后面

## 输出格式

返回：

```text
PUBLIC STATUS
- issue / PR state
- CI / review state

CLASSIFICATION
- merge / port-rebuild / close / park
- one-paragraph rationale

LINEAR ACTION
- create / update / no Linear item needed
- project / lane if applicable

NEXT OPERATOR ACTION
- exact next move
```

## 适用场景

- "审计开放 PR 积压，告诉我哪些要合并、哪些要重建"
- "将 GitHub issue 映射到我们的 ECC 1.x 和 ECC 2.0 项目通道"
- "检查这是否需要 Linear issue，还是应该保留在 GitHub"
