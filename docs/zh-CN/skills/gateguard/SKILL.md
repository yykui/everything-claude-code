---
name: gateguard
description: 强制事实的关卡，阻断 Edit/Write/Bash（包括 MultiEdit）并要求进行具体调查（导入者、数据模式、用户指令）才允许操作。与未设关卡的 Agent 相比，可测量地将输出质量提升 +2.25 分。
origin: community
---

# GateGuard — 强制事实的行动前关卡

一个 PreToolUse 钩子，强制 Claude 在编辑前先调查。它不是要求自我评估（"你确定吗？"），而是要求具体事实。调查行为本身创造了自我评估永远无法带来的上下文感知。

## 何时激活

- 处理任何文件编辑会影响多个模块的代码库
- 具有特定模式或日期格式的数据文件项目
- AI 生成的代码必须匹配现有模式的团队
- Claude 倾向于猜测而非调查的任何工作流

## 核心概念

LLM 自我评估不起作用。问"你是否违反了任何策略？"，答案永远是"没有"。这已通过实验验证。

但问"列出所有导入该模块的文件"会强制 LLM 运行 Grep 和 Read。调查行为本身创造了能改变输出的上下文。

**三阶段关卡：**

```
1. DENY  — block the first Edit/Write/Bash attempt
2. FORCE — tell the model exactly which facts to gather
3. ALLOW — permit retry after facts are presented
```

没有竞品同时做到这三点。大多数止步于拒绝。

## 证据

两次独立 A/B 测试，相同 Agent，相同任务：

| 任务 | 有关卡 | 无关卡 | 差距 |
| --- | --- | --- | --- |
| 分析模块 | 8.0/10 | 6.5/10 | +1.5 |
| Webhook 验证器 | 10.0/10 | 7.0/10 | +3.0 |
| **平均** | **9.0** | **6.75** | **+2.25** |

两个 Agent 都能产出可运行且通过测试的代码。差距在于设计深度。

## 关卡类型

### Edit / MultiEdit 关卡（每个文件首次编辑）

MultiEdit 以相同方式处理——批次中的每个文件单独设关卡。

```
Before editing {file_path}, present these facts:

1. List ALL files that import/require this file (use Grep)
2. List the public functions/classes affected by this change
3. If this file reads/writes data files, show field names, structure,
   and date format (use redacted or synthetic values, not raw production data)
4. Quote the user's current instruction verbatim
```

### Write 关卡（首次创建新文件）

```
Before creating {file_path}, present these facts:

1. Name the file(s) and line(s) that will call this new file
2. Confirm no existing file serves the same purpose (use Glob)
3. If this file reads/writes data files, show field names, structure,
   and date format (use redacted or synthetic values, not raw production data)
4. Quote the user's current instruction verbatim
```

### 破坏性 Bash 关卡（每次破坏性命令）

触发条件：`rm -rf`、`git reset --hard`、`git push --force`、`drop table` 等。

```
1. List all files/data this command will modify or delete
2. Write a one-line rollback procedure
3. Quote the user's current instruction verbatim
```

### 常规 Bash 关卡（每会话一次）

```
1. The current user request in one sentence
2. What this specific command verifies or produces
```

## 快速开始

### 方式 A：使用 ECC 钩子（零安装）

`scripts/hooks/gateguard-fact-force.js` 中的钩子已包含在本插件中。通过 hooks.json 启用。

### 方式 B：带配置的完整包

```bash
pip install gateguard-ai
gateguard init
```

这会添加 `.gateguard.yml` 用于项目级配置（自定义消息、忽略路径、关卡开关）。

## 反模式

- **不要用自我评估代替关卡。** "你确定吗？"永远得到"确定"。这已通过实验验证。
- **不要跳过数据模式检查。** 两次 A/B 测试中，两个 Agent 都假设日期使用 ISO-8601，而真实数据使用 `%Y/%m/%d %H:%M`。检查数据结构（使用脱敏值）可以防止整类 bug。
- **不要对每个 Bash 命令都设关卡。** 常规 bash 每会话关卡一次，破坏性 bash 每次关卡。这种平衡在避免减速的同时捕获真实风险。

## 最佳实践

- 让关卡自然触发。不要试图预先回答关卡问题——调查本身才是提升质量的关键。
- 针对你的领域自定义关卡消息。若项目有特定约定，将其添加到关卡提示中。
- 使用 `.gateguard.yml` 忽略 `.venv/`、`node_modules/`、`.git/` 等路径。

## 相关技能

- `safety-guard` — 运行时安全检查（互补，不重叠）
- `code-reviewer` — 编辑后审查（GateGuard 是编辑前调查）
