---
name: token-budget-advisor
description: >-
  在回答之前为用户提供关于消耗多少响应深度的知情选择。当用户明确希望控制响应长度、深度或 token 预算时使用本技能。
  触发条件："token budget"、"token count"、"token usage"、"token limit"、
  "response length"、"answer depth"、"short version"、"brief answer"、
  "detailed answer"、"exhaustive answer"、"respuesta corta vs larga"、
  "cuántos tokens"、"ahorrar tokens"、"responde al 50%"、"dame la versión
  corta"、"quiero controlar cuánto usas"，或用户明确要求控制答案大小或深度的明显变体。
  不触发条件：用户在当前会话中已指定级别（保持该级别），请求明显是单词答案，或
  "token" 指的是认证/会话/支付 token 而非响应大小。
origin: community
---

# Token Budget Advisor（TBA）

在 Claude 回答**之前**拦截响应流，为用户提供响应深度的选择。

## 何时使用

- 用户希望控制响应的长度或详细程度
- 用户提到 token、预算、深度或响应长度
- 用户说"短版本"、"tldr"、"brief"、"al 25%"、"exhaustive"等
- 用户想要事先选择深度/详细程度

**不触发**条件：用户本次会话中已设置级别（静默保持），或答案明显只有一行。

## 工作原理

### 第一步——估算输入 token 数

使用仓库的规范上下文预算启发式方法在脑中估算提示的 token 数量。

使用与 [context-budget](../context-budget/SKILL.md) 相同的校准指导：

- 散文：`单词数 × 1.3`
- 代码密集型或混合/代码块：`字符数 / 4`

对于混合内容，使用主要内容类型并保持估算启发式方法。

### 第二步——按复杂度估算响应大小

对提示进行分类，然后应用乘数范围获取完整响应窗口：

| 复杂度 | 乘数范围 | 示例提示 |
|--------|----------|----------|
| 简单 | 3× – 8× | "X 是什么？"、是/否、单一事实 |
| 中等 | 8× – 20× | "X 是如何工作的？" |
| 中高 | 10× – 25× | 带上下文的代码请求 |
| 复杂 | 15× – 40× | 多部分分析、比较、架构 |
| 创意 | 10× – 30× | 故事、文章、叙述写作 |

响应窗口 = `input_tokens × mult_min` 到 `input_tokens × mult_max`（但不超过模型配置的输出 token 限制）。

### 第三步——呈现深度选项

在回答**之前**使用实际估算数字呈现此块：

```
Analyzing your prompt...

Input: ~[N] tokens  |  Type: [type]  |  Complexity: [level]  |  Language: [lang]

Choose your depth level:

[1] Essential   (25%)  ->  ~[tokens]   Direct answer only, no preamble
[2] Moderate    (50%)  ->  ~[tokens]   Answer + context + 1 example
[3] Detailed    (75%)  ->  ~[tokens]   Full answer with alternatives
[4] Exhaustive (100%)  ->  ~[tokens]   Everything, no limits

Which level? (1-4 or say "25% depth", "50% depth", "75% depth", "100% depth")

Precision: heuristic estimate ~85-90% accuracy (±15%).
```

各级别 token 估算（在响应窗口内）：
- 25%  → `min + (max - min) × 0.25`
- 50%  → `min + (max - min) × 0.50`
- 75%  → `min + (max - min) × 0.75`
- 100% → `max`

### 第四步——按选定级别响应

| 级别 | 目标长度 | 包含 | 省略 |
|------|----------|------|------|
| 25% 基本 | 最多 2-4 句 | 直接答案、关键结论 | 上下文、示例、细微差别、替代方案 |
| 50% 适中 | 1-3 段 | 答案 + 必要上下文 + 1 个示例 | 深度分析、边缘情况、参考资料 |
| 75% 详细 | 结构化响应 | 多个示例、优缺点、替代方案 | 极端边缘情况、详尽参考资料 |
| 100% 详尽 | 无限制 | 全部——完整分析、所有代码、所有视角 | 无 |

## 快捷方式——跳过提问

如果用户已明确表示级别，直接在该级别响应，无需提问：

| 用户说的话 | 级别 |
|-----------|------|
| "1" / "25% depth" / "short version" / "brief answer" / "tldr" | 25% |
| "2" / "50% depth" / "moderate depth" / "balanced answer" | 50% |
| "3" / "75% depth" / "detailed answer" / "thorough answer" | 75% |
| "4" / "100% depth" / "exhaustive answer" / "full deep dive" | 100% |

如果用户在会话早期设置了级别，**静默保持**该级别用于后续响应，除非他们更改。

## 精度说明

本技能使用启发式估算——没有真实的分词器。准确率约 85-90%，误差 ±15%。始终显示免责声明。

## 示例

### 触发条件

- "先给我短版本。"
- "你的答案会用多少 token？"
- "以 50% 深度回应。"
- "我要详尽的答案，不要摘要。"
- "Dame la version corta y luego la detallada."

### 不触发条件

- "JWT token 是什么？"
- "结账流程使用支付 token。"
- "这正常吗？"
- "完成重构。"
- 用户在本次会话中已选择深度后的后续问题

## 来源

来自 [TBA — Token Budget Advisor for Claude Code](https://github.com/Xabilimon1/Token-Budget-Advisor-Claude-Code-) 的独立技能。
原始项目还附带一个 Python 估算脚本，但本仓库保持技能自包含且仅使用启发式方法。
