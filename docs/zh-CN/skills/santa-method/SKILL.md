---
name: santa-method
description: "多代理对抗性验证与收敛循环。两个独立的评审代理必须均通过，输出才能发布。"
origin: "Ronald Skelton - Founder, RapportScore.ai"
---

# 圣诞老人方法

多代理对抗性验证框架。列一份清单，检查两遍。如果有问题，修复直到没问题为止。

核心洞察：单个代理审查自己的输出会共享产生输出时相同的偏见、知识盲区和系统性错误。两个没有共享上下文的独立评审者可以打破这种失败模式。

## 何时激活

在以下情况下调用本技能：
- 输出将被发布、部署或被终端用户使用
- 必须执行合规、监管或品牌约束
- 代码在没有人工审查的情况下上线生产
- 内容准确性至关重要（技术文档、教育材料、面向客户的文案）
- 大规模批量生成时抽查会遗漏系统性模式
- 幻觉风险较高（主张、统计数据、API 引用、法律语言）

不要用于内部草稿、探索性研究，或具有确定性验证的任务（这些使用构建/测试/lint 流水线）。

## 架构

```
┌─────────────┐
│  GENERATOR   │  Phase 1: Make a List
│  (Agent A)   │  Produce the deliverable
└──────┬───────┘
       │ output
       ▼
┌──────────────────────────────┐
│     DUAL INDEPENDENT REVIEW   │  Phase 2: Check It Twice
│                                │
│  ┌───────────┐ ┌───────────┐  │  Two agents, same rubric,
│  │ Reviewer B │ │ Reviewer C │  │  no shared context
│  └─────┬─────┘ └─────┬─────┘  │
│        │              │        │
└────────┼──────────────┼────────┘
         │              │
         ▼              ▼
┌──────────────────────────────┐
│        VERDICT GATE           │  Phase 3: Naughty or Nice
│                                │
│  B passes AND C passes → NICE  │  Both must pass.
│  Otherwise → NAUGHTY           │  No exceptions.
└──────┬──────────────┬─────────┘
       │              │
    NICE           NAUGHTY
       │              │
       ▼              ▼
   [ SHIP ]    ┌─────────────┐
               │  FIX CYCLE   │  Phase 4: Fix Until Nice
               │              │
               │ iteration++  │  Collect all flags.
               │ if i > MAX:  │  Fix all issues.
               │   escalate   │  Re-run both reviewers.
               │ else:        │  Loop until convergence.
               │   goto Ph.2  │
               └──────────────┘
```

## 阶段详情

### 阶段 1：列清单（生成）

执行主要任务。不更改你的正常生成工作流。圣诞老人方法是生成后的验证层，而非生成策略。

```python
# The generator runs as normal
output = generate(task_spec)
```

### 阶段 2：检查两遍（独立双重评审）

并行启动两个评审代理。关键不变量：

1. **上下文隔离**——两个评审者互不看对方的评估
2. **相同评判标准**——两者收到相同的评估准则
3. **相同输入**——两者收到原始规范和生成的输出
4. **结构化输出**——每个返回类型化评定结果，而非散文

```python
REVIEWER_PROMPT = """
You are an independent quality reviewer. You have NOT seen any other review of this output.

## Task Specification
{task_spec}

## Output Under Review
{output}

## Evaluation Rubric
{rubric}

## Instructions
Evaluate the output against EACH rubric criterion. For each:
- PASS: criterion fully met, no issues
- FAIL: specific issue found (cite the exact problem)

Return your assessment as structured JSON:
{
  "verdict": "PASS" | "FAIL",
  "checks": [
    {"criterion": "...", "result": "PASS|FAIL", "detail": "..."}
  ],
  "critical_issues": ["..."],   // blockers that must be fixed
  "suggestions": ["..."]         // non-blocking improvements
}

Be rigorous. Your job is to find problems, not to approve.
"""
```

```python
# Spawn reviewers in parallel (Claude Code subagents)
review_b = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa Reviewer B")
review_c = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa Reviewer C")

# Both run concurrently — neither sees the other
```

### 评判标准设计

评判标准是最重要的输入。模糊的标准产生模糊的评审。每个准则必须有客观的通过/失败条件。

| 准则 | 通过条件 | 失败信号 |
|-----------|---------------|----------------|
| 事实准确性 | 所有主张可根据来源材料或常识验证 | 捏造的统计数据、错误的版本号、不存在的 API |
| 无幻觉 | 无虚构的实体、引用、URL 或参考文献 | 链接到不存在的页面、无来源的归因引用 |
| 完整性 | 规范中的每个需求都有所涉及 | 缺少章节、跳过边界情况、覆盖不完整 |
| 合规性 | 通过所有项目特定约束 | 使用了禁用词汇、语气违规、法规不合规 |
| 内部一致性 | 输出内无矛盾 | A 节说 X，B 节说非 X |
| 技术正确性 | 代码可编译/运行，算法合理 | 语法错误、逻辑缺陷、错误的复杂度声明 |

#### 领域专项评判标准扩展

**内容/营销：**
- 品牌语调遵从性
- SEO 需求满足（关键词密度、元标签、结构）
- 无竞争对手商标滥用
- CTA 存在且链接正确

**代码：**
- 类型安全（无 `any` 泄漏，正确的空值处理）
- 错误处理覆盖
- 安全性（代码中无秘密、输入验证、注入防护）
- 新路径的测试覆盖

**合规敏感（受监管、法律、金融）：**
- 无结果保证或未经证实的主张
- 必需的免责声明存在
- 仅使用批准的术语
- 适合司法管辖区的语言

### 阶段 3：好坏判定（评定关卡）

```python
def santa_verdict(review_b, review_c):
    """Both reviewers must pass. No partial credit."""
    if review_b.verdict == "PASS" and review_c.verdict == "PASS":
        return "NICE"  # Ship it

    # Merge flags from both reviewers, deduplicate
    all_issues = dedupe(review_b.critical_issues + review_c.critical_issues)
    all_suggestions = dedupe(review_b.suggestions + review_c.suggestions)

    return "NAUGHTY", all_issues, all_suggestions
```

为什么两者都必须通过：如果只有一个评审者发现了问题，那个问题就是真实存在的。另一个评审者的盲区正是圣诞老人方法要消除的失败模式。

### 阶段 4：修复直到通过（收敛循环）

```python
MAX_ITERATIONS = 3

for iteration in range(MAX_ITERATIONS):
    verdict, issues, suggestions = santa_verdict(review_b, review_c)

    if verdict == "NICE":
        log_santa_result(output, iteration, "passed")
        return ship(output)

    # Fix all critical issues (suggestions are optional)
    output = fix_agent.execute(
        output=output,
        issues=issues,
        instruction="Fix ONLY the flagged issues. Do not refactor or add unrequested changes."
    )

    # Re-run BOTH reviewers on fixed output (fresh agents, no memory of previous round)
    review_b = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))
    review_c = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))

# Exhausted iterations — escalate
log_santa_result(output, MAX_ITERATIONS, "escalated")
escalate_to_human(output, issues)
```

关键：每轮评审使用**全新的代理**。评审者不得携带前几轮的记忆，因为先前的上下文会产生锚定偏差。

## 实现模式

### 模式 A：Claude Code 子代理（推荐）

子代理提供真正的上下文隔离。每个评审者是独立的进程，没有共享状态。

```bash
# In a Claude Code session, use the Agent tool to spawn reviewers
# Both agents run in parallel for speed
```

```python
# Pseudocode for Agent tool invocation
reviewer_b = Agent(
    description="Santa Review B",
    prompt=f"Review this output for quality...\n\nRUBRIC:\n{rubric}\n\nOUTPUT:\n{output}"
)
reviewer_c = Agent(
    description="Santa Review C",
    prompt=f"Review this output for quality...\n\nRUBRIC:\n{rubric}\n\nOUTPUT:\n{output}"
)
```

### 模式 B：顺序内联（备选方案）

当子代理不可用时，通过显式上下文重置来模拟隔离：

1. 生成输出
2. 新上下文："你是评审者 1。仅根据此评判标准评估。寻找问题。"
3. 逐字记录发现
4. 完全清除上下文
5. 新上下文："你是评审者 2。仅根据此评判标准评估。寻找问题。"
6. 比较两份评审，修复，重复

子代理模式严格优于内联模拟——内联模拟有评审者间上下文泄漏的风险。

### 模式 C：批量采样

对于大批量（100+ 条目），对每个条目进行完整的圣诞老人验证成本过高。使用分层采样：

1. 对随机样本运行圣诞老人验证（批次的 10-15%，最少 5 条）
2. 按类型对失败进行分类（幻觉、合规、完整性等）
3. 如果出现系统性模式，对整批应用针对性修复
4. 对修复后的批次重新采样和重新验证
5. 持续循环直到干净样本通过

```python
import random

def santa_batch(items, rubric, sample_rate=0.15):
    sample = random.sample(items, max(5, int(len(items) * sample_rate)))

    for item in sample:
        result = santa_full(item, rubric)
        if result.verdict == "NAUGHTY":
            pattern = classify_failure(result.issues)
            items = batch_fix(items, pattern)  # Fix all items matching pattern
            return santa_batch(items, rubric)   # Re-sample

    return items  # Clean sample → ship batch
```

## 失败模式与缓解措施

| 失败模式 | 症状 | 缓解措施 |
|-------------|---------|------------|
| 无限循环 | 评审者在修复后持续发现新问题 | 最大迭代上限（3 次）。上报。 |
| 橡皮图章 | 两个评审者都通过一切 | 对抗性提示："你的工作是发现问题，而非批准。" |
| 主观漂移 | 评审者标记风格偏好而非错误 | 使用只有客观通过/失败标准的严格评判标准 |
| 修复回归 | 修复问题 A 导致问题 B | 每轮新评审者发现回归 |
| 评审者共识偏差 | 两个评审者都遗漏了相同的问题 | 通过独立性缓解，但无法完全消除。对于关键输出，添加第三个评审者或人工抽查。 |
| 成本爆炸 | 大型输出迭代次数过多 | 批量采样模式。每次验证周期的预算上限。 |

## 与其他技能的集成

| 技能 | 关系 |
|-------|-------------|
| 验证循环 | 用于确定性检查（构建、lint、测试）。圣诞老人用于语义检查（准确性、幻觉）。先运行验证循环，再运行圣诞老人。 |
| 评估框架 | 圣诞老人方法结果输入评估指标。追踪圣诞老人运行中的 pass@k 以衡量生成器质量变化。 |
| 持续学习 v2 | 圣诞老人发现成为直觉。相同准则反复失败 → 学习避免该模式的行为。 |
| 战略压缩 | 在压缩前运行圣诞老人。不要在验证中途丢失评审上下文。 |

## 指标

追踪这些指标以衡量圣诞老人方法的有效性：

- **首次通过率**：第 1 轮就通过圣诞老人的输出百分比（目标：>70%）
- **平均收敛迭代次数**：达到"通过"状态的平均轮数（目标：<1.5）
- **问题分类**：失败类型的分布（幻觉 vs 完整性 vs 合规）
- **评审者一致性**：两个评审者都标记的问题百分比 vs 只有一个标记的（低一致性 = 评判标准需要收紧）
- **漏网率**：发布后发现的圣诞老人应该捕获的问题（目标：0）

## 成本分析

每个验证周期，圣诞老人方法的 token 成本约为仅生成阶段的 2-3 倍。对于大多数高风险输出，这是值得的：

```
Cost of Santa = (generation tokens) + 2×(review tokens per round) × (avg rounds)
Cost of NOT Santa = (reputation damage) + (correction effort) + (trust erosion)
```

对于批量操作，采样模式将成本降低到完整验证的约 15-20%，同时捕获超过 90% 的系统性问题。
