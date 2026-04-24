---
name: skill-comply
description: 可视化技能、规则和 Agent 定义是否被实际遵守——在 3 种提示词严格程度下自动生成场景，运行 Agent，对行为序列进行分类，并生成包含完整工具调用时间线的合规率报告
origin: ECC
tools: Read, Bash
---

# skill-comply：自动化合规度量

通过以下方式衡量编码 Agent 是否真正遵循技能、规则或 Agent 定义：
1. 从任意 .md 文件自动生成预期行为序列（规格）
2. 自动生成具有递减提示词严格程度的场景（支持性 → 中性 → 竞争性）
3. 运行 `claude -p` 并通过 stream-json 捕获工具调用追踪
4. 使用 LLM（而非正则表达式）对照规格步骤对工具调用进行分类
5. 确定性地检查时序顺序
6. 生成包含规格、提示词和时间线的自包含报告

## 支持的目标

- **技能**（`skills/*/SKILL.md`）：工作流技能，如 search-first、TDD 指南
- **规则**（`rules/common/*.md`）：强制性规则，如 testing.md、security.md、git-workflow.md
- **Agent 定义**（`agents/*.md`）：验证 Agent 是否在预期时被调用（内部工作流验证暂不支持）

## 激活时机

- 用户运行 `/skill-comply <路径>`
- 用户询问"这条规则是否真的被遵守？"
- 添加新规则/技能后，验证 Agent 合规性
- 作为质量维护的定期检查

## 使用方法

```bash
# 完整运行
uv run python -m scripts.run ~/.claude/rules/common/testing.md

# 试运行（不产生费用，仅生成规格和场景）
uv run python -m scripts.run --dry-run ~/.claude/skills/search-first/SKILL.md

# 自定义模型
uv run python -m scripts.run --gen-model haiku --model sonnet <path>
```

## 核心概念：提示词独立性

衡量即使在提示词未明确支持的情况下，技能/规则是否仍被遵守。

## 报告内容

报告为自包含格式，包括：
1. 预期行为序列（自动生成的规格）
2. 场景提示词（每种严格程度级别下的请求内容）
3. 各场景的合规分数
4. 带有 LLM 分类标签的工具调用时间线

### 高级选项（可选）

对于熟悉钩子（hooks）的用户，报告还包含针对低合规度步骤的钩子推广建议。这仅供参考——核心价值在于合规可见性本身。
