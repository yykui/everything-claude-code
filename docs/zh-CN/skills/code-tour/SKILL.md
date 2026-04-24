---
name: code-tour
description: 创建 CodeTour `.tour` 文件——针对特定角色的、带有真实文件和行号锚点的逐步引导。适用于新成员入职导览、架构讲解、PR 导览、根因分析导览，以及结构化的"解释这是如何工作的"请求。
origin: ECC
---

# 代码导览

为代码库创建 **CodeTour** `.tour` 文件，可直接打开到真实文件和行范围。导览文件存放在 `.tours/` 目录，遵循 CodeTour 格式，而非随意编写的 Markdown 笔记。

一个好的导览是针对特定读者的叙事：
- 他们正在看什么
- 为什么重要
- 接下来应该走哪条路径

只创建 `.tour` JSON 文件。不要作为此技能的一部分修改源代码。

## 何时使用

在以下情况使用此技能：
- 用户请求代码导览、入职导览、架构讲解或 PR 导览
- 用户说"解释 X 是如何工作的"并希望得到可复用的引导产物
- 用户需要为新工程师或审查者提供快速上手路径
- 任务更适合用引导序列而非平铺式摘要来完成

示例：
- 为新维护者做入职引导
- 为某个服务或包做架构导览
- 锚定到变更文件的 PR 审查导览
- 展示故障路径的根因分析导览
- 覆盖信任边界和关键检查点的安全审查导览

## 何时不使用

| 不用 code-tour 的情况 | 替代方案 |
| --- | --- |
| 聊天中的一次性解释已足够 | 直接回答 |
| 用户想要散文文档而非 `.tour` 产物 | `documentation-lookup` 或编辑仓库文档 |
| 任务是实现或重构 | 进行实现工作 |
| 任务是宽泛的代码库入门但不需要导览产物 | `codebase-onboarding` |

## 工作流

### 1. 探索

在写任何内容之前先探索仓库：
- README 和 package/app 入口点
- 目录结构
- 相关配置文件
- 若导览以 PR 为重点，则查看变更的文件

在理解代码结构之前不要开始写步骤。

### 2. 推断读者

根据请求确定角色和深度。

| 请求形式 | 角色 | 建议深度 |
| --- | --- | --- |
| "入职"、"新成员" | `new-joiner` | 9-13 步 |
| "快速导览"、"整体感受" | `vibecoder` | 5-8 步 |
| "架构" | `architect` | 14-18 步 |
| "导览此 PR" | `pr-reviewer` | 7-11 步 |
| "为什么出问题了" | `rca-investigator` | 7-11 步 |
| "安全审查" | `security-reviewer` | 7-11 步 |
| "解释此功能如何工作" | `feature-explainer` | 7-11 步 |
| "调试此路径" | `bug-fixer` | 7-11 步 |

### 3. 阅读并验证锚点

每个文件路径和行号锚点必须是真实存在的：
- 确认文件存在
- 确认行号在范围内
- 若使用选区，验证确切的代码块
- 若文件容易变动，优先使用基于模式的锚点

永远不要猜测行号。

### 4. 编写 `.tour`

写入：

```text
.tours/<persona>-<focus>.tour
```

保持路径确定性和可读性。

### 5. 验证

完成前：
- 每个引用的路径都存在
- 每个行号或选区都有效
- 第一步锚定到真实的文件或目录
- 导览讲述了一个连贯的故事，而非文件清单

## 步骤类型

### 内容型

少用，通常仅用于结尾步骤：

```json
{ "title": "Next Steps", "description": "You can now trace the request path end to end." }
```

不要让第一步只有内容，没有锚点。

### 目录型

用于引导读者了解某个模块：

```json
{ "directory": "src/services", "title": "Service Layer", "description": "The core orchestration logic lives here." }
```

### 文件 + 行号型

这是默认的步骤类型：

```json
{ "file": "src/auth/middleware.ts", "line": 42, "title": "Auth Gate", "description": "Every protected request passes here first." }
```

### 选区型

当某个代码块比整个文件更重要时使用：

```json
{
  "file": "src/core/pipeline.ts",
  "selection": {
    "start": { "line": 15, "character": 0 },
    "end": { "line": 34, "character": 0 }
  },
  "title": "Request Pipeline",
  "description": "This block wires validation, auth, and downstream execution."
}
```

### 模式型

当精确行号可能漂移时使用：

```json
{ "file": "src/app.ts", "pattern": "export default class App", "title": "Application Entry" }
```

### URI 型

在有帮助时用于 PR、Issue 或文档：

```json
{ "uri": "https://github.com/org/repo/pull/456", "title": "The PR" }
```

## 写作规则：SMIG

每个描述都应回答：
- **情境（Situation）**：读者正在看什么
- **机制（Mechanism）**：它是如何工作的
- **含义（Implication）**：对该角色读者为何重要
- **陷阱（Gotcha）**：聪明的读者可能会遗漏什么

保持描述简洁、具体，并基于实际代码。

## 叙事结构

除非任务明显需要不同结构，否则使用以下弧线：
1. 定向
2. 模块地图
3. 核心执行路径
4. 边界情况或陷阱
5. 结尾 / 下一步行动

导览应该像一条路径，而不是一份清单。

## 示例

```json
{
  "$schema": "https://aka.ms/codetour-schema",
  "title": "API Service Tour",
  "description": "Walkthrough of the request path for the payments service.",
  "ref": "main",
  "steps": [
    {
      "directory": "src",
      "title": "Source Root",
      "description": "All runtime code for the service starts here."
    },
    {
      "file": "src/server.ts",
      "line": 12,
      "title": "Entry Point",
      "description": "The server boots here and wires middleware before any route is reached."
    },
    {
      "file": "src/routes/payments.ts",
      "line": 8,
      "title": "Payment Routes",
      "description": "Every payments request enters through this router before hitting service logic."
    },
    {
      "title": "Next Steps",
      "description": "You can now follow any payment request end to end with the main anchors in place."
    }
  ]
}
```

## 反模式

| 反模式 | 修复方式 |
| --- | --- |
| 平铺文件列表 | 讲述步骤之间有依赖关系的故事 |
| 泛泛的描述 | 指明具体的代码路径或模式 |
| 猜测锚点 | 先验证每个文件和行号 |
| 快速导览步骤过多 | 大力削减 |
| 第一步只有内容 | 将第一步锚定到真实文件或目录 |
| 角色不匹配 | 为实际读者写作，而非泛泛的工程师 |

## 最佳实践

- 步骤数量与仓库规模和角色深度成比例
- 目录步骤用于定向，文件步骤用于实质内容
- PR 导览优先覆盖变更的文件
- 对于单体仓库，限定在相关包的范围内，而非导览所有内容
- 结尾告诉读者现在能做什么，而非总结回顾

## 相关技能

- `codebase-onboarding`
- `coding-standards`
- `council`
- 官方上游格式：`microsoft/codetour`
