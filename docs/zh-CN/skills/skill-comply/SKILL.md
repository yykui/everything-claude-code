---
name: skill-comply
description: 可视化技能、规则和 Agent 定义是否被实际遵守——在三种提示严格程度下自动生成场景，运行 Agent，分类行为序列，并生成包含完整工具调用时间线的合规率报告
origin: ECC
tools: Read, Bash
---

# skill-comply：自动化合规性测量

通过以下方式测量编码 Agent 是否实际遵循技能、规则或 Agent 定义：
1. 从任意 .md 文件自动生成预期行为序列（规格说明）
2. 按提示严格程度递减自动生成场景（支持性 → 中性 → 竞争性）
3. 运行 `claude -p` 并通过 stream-json 捕获工具调用跟踪
4. 使用 LLM（而非正则表达式）将工具调用与规格步骤进行分类匹配
5. 确定性地检查时间顺序
6. 生成包含规格说明、提示和时间线的独立报告

## 支持的目标

- **技能**（`skills/*/SKILL.md`）：工作流技能，如 search-first、TDD 指南
- **规则**（`rules/common/*.md`）：强制规则，如 testing.md、security.md、git-workflow.md
- **Agent 定义**（`agents/*.md`）：验证 Agent 是否在预期时被调用（暂不支持内部工作流验证）

## 何时激活

- 用户运行 `/skill-comply <路径>`
- 用户询问"这条规则是否真的被遵守？"
- 添加新规则/技能后，验证 Agent 合规性
- 作为质量维护的定期检查

## 用法

```bash
# 完整运行
uv run python -m scripts.run ~/.claude/rules/common/testing.md

# 演习（无费用，仅生成规格说明和场景）
uv run python -m scripts.run --dry-run ~/.claude/skills/search-first/SKILL.md

# 自定义模型
uv run python -m scripts.run --gen-model haiku --model sonnet <path>
```

## 核心概念：提示独立性

测量即使提示中没有明确支持，技能/规则是否仍被遵守。

## 报告内容

报告是独立的，包含：
1. 预期行为序列（自动生成的规格说明）
2. 场景提示（每个严格程度级别的提问内容）
3. 每个场景的合规分数
4. 带有 LLM 分类标签的工具调用时间线

### 高级功能（可选）

对于熟悉 hooks 的用户，报告还包含针对低合规步骤的 hook 推广建议。此为参考信息——主要价值在于合规性的可见性本身。
