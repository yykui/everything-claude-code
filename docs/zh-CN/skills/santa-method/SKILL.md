---
name: santa-method
description: 多 agent 对抗性验证与收敛循环。两个独立的审查 agent 必须全部通过，输出才能发布。
origin: "Ronald Skelton - Founder, RapportScore.ai"
---

# Santa 方法

多 agent 对抗性验证框架。列一张清单，检查两遍。如果有问题，修复直到完美。

核心洞察：单个 agent 审查自己的输出，会共享产生该输出时相同的偏见、知识盲区和系统性错误。两个没有共享上下文的独立审查者可以打破这种失败模式。

## 何时激活

在以下情况调用此技能：
- 输出将被发布、部署或供终端用户使用时
- 必须执行合规、监管或品牌约束时
- 代码在没有人工审查的情况下直接发布到生产环境时
- 内容准确性很重要（技术文档、教育材料、面向客户的文案）时
- 大规模批量生成时，抽样检查会遗漏系统性问题
- 幻觉风险较高（声明、统计数据、API 引用、法律语言）时

**请勿用于**内部草稿、探索性研究或具有确定性验证的任务（这些应使用构建/测试/lint 流水线）。

## 架构

```
┌─────────────┐
│   生成器     │  阶段 1：列清单
│  （Agent A） │  生成可交付物
└──────┬───────┘
       │ 输出
       ▼
┌──────────────────────────────┐
│       双重独立审查             │  阶段 2：检查两遍
│                                │
│  ┌───────────┐ ┌───────────┐  │  两个 agent，相同评估标准，
│  │ 审查者 B  │ │ 审查者 C  │  │  无共享上下文
│  └─────┬─────┘ └─────┬─────┘  │
│        │              │        │
└────────┼──────────────┼────────┘
         │              │
         ▼              ▼
┌──────────────────────────────┐
│         评判门控              │  阶段 3：好还是坏
│                                │
│  B 通过 且 C 通过 → 好         │  必须全部通过。
│  否则 → 有问题                 │  没有例外。
└──────┬──────────────┬─────────┘
       │              │
      好            有问题
       │              │
       ▼              ▼
   [ 发布 ]    ┌─────────────┐
               │   修复循环   │  阶段 4：修复直到完美
               │              │
               │ iteration++  │  收集所有标记问题。
               │ if i > MAX:  │  修复所有问题。
               │   升级处理   │  重新运行两个审查者。
               │ else:        │  循环直到收敛。
               │  goto 阶段 2 │
               └──────────────┘
```

## 阶段详情

### 阶段 1：列清单（生成）

执行主要任务。不改变正常的生成工作流。Santa 方法是生成后的验证层，而非生成策略。

```python
# 生成器正常运行
output = generate(task_spec)
```

### 阶段 2：检查两遍（独立双重审查）

并行生成两个审查 agent。关键不变量：

1. **上下文隔离** — 两个审查者都不能看到对方的评估
2. **相同评估标准** — 两者收到相同的评估准则
3. **相同输入** — 两者都收到原始规格和生成的输出
4. **结构化输出** — 每个审查者返回类型化的评判，而非散文

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
# 并行生成审查者（Claude Code 子 agent）
review_b = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa Reviewer B")
review_c = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa Reviewer C")

# 两者并发运行 — 互不可见
```

### 评估标准设计

评估标准是最重要的输入。模糊的标准产生模糊的审查。每条标准必须有客观的通过/失败条件。

| 标准 | 通过条件 | 失败信号 |
|------|---------|---------|
| 事实准确性 | 所有声明可根据来源材料或常识验证 | 捏造的统计数据、错误的版本号、不存在的 API |
| 无幻觉 | 无虚构实体、引言、URL 或参考资料 | 指向不存在页面的链接、无来源的引文 |
| 完整性 | 规格中的每项需求都已涉及 | 缺少章节、跳过边界情况、覆盖不完整 |
| 合规性 | 通过所有项目特定约束 | 使用了禁用词汇、违反语气要求、违反监管规定 |
| 内部一致性 | 输出内无矛盾 | A 节说 X，B 节说非 X |
| 技术正确性 | 代码可编译/运行，算法合理 | 语法错误、逻辑缺陷、错误的复杂度声明 |

#### 领域特定评估标准扩展

**内容/营销：**
- 品牌声音一致性
- SEO 要求满足（关键词密度、meta 标签、结构）
- 无竞争对手商标误用
- CTA 存在且链接正确

**代码：**
- 类型安全（无 `any` 泄漏，正确的空值处理）
- 错误处理覆盖率
- 安全性（代码中无密钥、输入验证、注入防护）
- 新路径的测试覆盖率

**合规敏感（受监管、法律、金融）：**
- 无结果保证或未经证实的声明
- 必需的免责声明已存在
- 仅使用已批准的术语
- 符合司法管辖区的语言要求

### 阶段 3：好还是坏（评判门控）

```python
def santa_verdict(review_b, review_c):
    """两个审查者必须全部通过。不允许部分通过。"""
    if review_b.verdict == "PASS" and review_c.verdict == "PASS":
        return "NICE"  # 发布

    # 合并两个审查者的标记问题，去重
    all_issues = dedupe(review_b.critical_issues + review_c.critical_issues)
    all_suggestions = dedupe(review_b.suggestions + review_c.suggestions)

    return "NAUGHTY", all_issues, all_suggestions
```

为什么必须全部通过：如果只有一个审查者发现了问题，该问题是真实存在的。另一个审查者的盲区正是 Santa 方法存在的原因。

### 阶段 4：修复直到完美（收敛循环）

```python
MAX_ITERATIONS = 3

for iteration in range(MAX_ITERATIONS):
    verdict, issues, suggestions = santa_verdict(review_b, review_c)

    if verdict == "NICE":
        log_santa_result(output, iteration, "passed")
        return ship(output)

    # 修复所有关键问题（建议可选）
    output = fix_agent.execute(
        output=output,
        issues=issues,
        instruction="Fix ONLY the flagged issues. Do not refactor or add unrequested changes."
    )

    # 对修复后的输出重新运行两个审查者（全新 agent，不记得上一轮）
    review_b = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))
    review_c = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))

# 已用尽迭代次数 — 升级处理
log_santa_result(output, MAX_ITERATIONS, "escalated")
escalate_to_human(output, issues)
```

关键：每轮审查使用**全新 agent**。审查者不得保留上一轮的记忆，因为先验上下文会造成锚定偏差。

## 实现模式

### 模式 A：Claude Code 子 agent（推荐）

子 agent 提供真正的上下文隔离。每个审查者都是一个独立进程，没有共享状态。

```bash
# 在 Claude Code 会话中，使用 Agent 工具生成审查者
# 两个 agent 并行运行以提高速度
```

```python
# Agent 工具调用伪代码
reviewer_b = Agent(
    description="Santa Review B",
    prompt=f"Review this output for quality...\n\nRUBRIC:\n{rubric}\n\nOUTPUT:\n{output}"
)
reviewer_c = Agent(
    description="Santa Review C",
    prompt=f"Review this output for quality...\n\nRUBRIC:\n{rubric}\n\nOUTPUT:\n{output}"
)
```

### 模式 B：顺序内联（备用方案）

当子 agent 不可用时，通过显式上下文重置模拟隔离：

1. 生成输出
2. 新上下文："你是审查者 1。仅根据此评估标准评估。找出问题。"
3. 逐字记录发现
4. 完全清除上下文
5. 新上下文："你是审查者 2。仅根据此评估标准评估。找出问题。"
6. 对比两份审查，修复，重复

子 agent 模式严格优于内联模拟 — 内联模拟存在审查者之间上下文渗漏的风险。

### 模式 C：批量抽样

对于大批量（100+ 条目），对每条目执行完整 Santa 流程代价过高。使用分层抽样：

1. 对随机样本运行 Santa（批量的 10-15%，最少 5 条）
2. 按类型分类失败（幻觉、合规、完整性等）
3. 如果出现系统性问题，对整批应用针对性修复
4. 对修复后的批量重新抽样并验证
5. 持续直到干净的样本通过

```python
import random

def santa_batch(items, rubric, sample_rate=0.15):
    sample = random.sample(items, max(5, int(len(items) * sample_rate)))

    for item in sample:
        result = santa_full(item, rubric)
        if result.verdict == "NAUGHTY":
            pattern = classify_failure(result.issues)
            items = batch_fix(items, pattern)  # 修复所有匹配模式的条目
            return santa_batch(items, rubric)   # 重新抽样

    return items  # 干净的样本 → 发布批量
```

## 失败模式与缓解措施

| 失败模式 | 症状 | 缓解措施 |
|---------|------|---------|
| 无限循环 | 审查者在修复后持续发现新问题 | 最大迭代上限（3 次）。升级处理。 |
| 橡皮图章 | 两个审查者都批准所有内容 | 对抗性提示："你的工作是找问题，而非批准。" |
| 主观漂移 | 审查者标记风格偏好而非错误 | 仅使用具有客观通过/失败条件的严格标准 |
| 修复回归 | 修复问题 A 引入了问题 B | 每轮使用全新审查者以捕获回归 |
| 审查者一致性偏差 | 两个审查者遗漏同一问题 | 通过独立性缓解，但未完全消除。对关键输出，添加第三个审查者或人工抽查。 |
| 成本爆炸 | 大型输出迭代过多 | 批量抽样模式。每次验证循环设置预算上限。 |

## 与其他技能的集成

| 技能 | 关系 |
|------|------|
| Verification Loop | 用于确定性检查（构建、lint、测试）。Santa 用于语义检查（准确性、幻觉）。先运行 verification-loop，再运行 Santa。 |
| Eval Harness | Santa 方法结果反馈到评估指标。跨 Santa 运行追踪 pass@k 以衡量生成器质量。 |
| Continuous Learning v2 | Santa 发现成为本能。反复在同一标准上失败 → 习得避免该模式的行为。 |
| Strategic Compact | 在压缩前运行 Santa。不要在验证中途丢失审查上下文。 |

## 指标

追踪以下指标以衡量 Santa 方法的有效性：

- **首轮通过率**：第 1 轮即通过 Santa 的输出比例（目标：>70%）
- **平均收敛迭代次数**：达到完美的平均轮数（目标：<1.5）
- **问题分类**：失败类型分布（幻觉 vs 完整性 vs 合规）
- **审查者一致性**：两个审查者都标记的问题比例 vs 仅一个标记（一致性低 = 标准需要收紧）
- **漏逃率**：发布后发现的本应被 Santa 捕获的问题（目标：0）

## 成本分析

Santa 方法每个验证周期的 token 成本约为纯生成成本的 2-3 倍。对大多数高风险输出而言，这是值得的：

```
Santa 的成本 = （生成 token）+ 2×（每轮审查 token）×（平均轮数）
不用 Santa 的成本 = （声誉损失）+（纠正工作量）+（信任侵蚀）
```

对于批量操作，抽样模式将成本降低至完整验证的约 15-20%，同时捕获 >90% 的系统性问题。
