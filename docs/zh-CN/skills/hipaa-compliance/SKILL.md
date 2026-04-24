---
name: hipaa-compliance
description: 针对医疗隐私和安全工作的 HIPAA 专项入口。当任务明确涉及 HIPAA、PHI 处理、受保护实体、BAA、违规态势或美国医疗合规要求时使用。
origin: ECC direct-port adaptation
version: "1.0.0"
---

# HIPAA 合规

当任务明确涉及美国医疗合规时，将此技能作为 HIPAA 专项入口使用。本技能有意保持精简和规范：

- `healthcare-phi-compliance` 仍然是 PHI/PII 处理、数据分类、审计日志、加密和泄露预防的主要实现技能。
- `healthcare-reviewer` 仍然是当代码、架构或产品行为需要具备医疗意识的二次审查时的专项审查者。
- `security-review` 仍然适用于一般的身份验证、输入处理、密钥、API 和部署安全加固。

## 何时使用

- 请求明确提及 HIPAA、PHI、受保护实体、业务伙伴或 BAA
- 构建或审查存储、处理、导出或传输 PHI 的美国医疗软件
- 评估日志、分析、LLM 提示、存储或支持工作流是否产生 HIPAA 暴露风险
- 设计面向患者或临床医生的系统，其中最小必要访问权限和可审计性至关重要

## 工作原理

将 HIPAA 视为叠加在更广泛医疗隐私技能之上的覆盖层：

1. 从 `healthcare-phi-compliance` 开始了解具体实现规则。
2. 应用 HIPAA 专项决策门：
   - 这是 PHI 数据吗？
   - 这个参与者是受保护实体还是业务伙伴？
   - 供应商或模型提供商在接触数据之前是否需要 BAA？
   - 访问是否限制在最小必要范围内？
   - 读取/写入/导出事件是否可审计？
3. 如果任务影响患者安全、临床工作流或受监管的生产架构，则升级到 `healthcare-reviewer`。

## HIPAA 专项护栏

- 切勿将 PHI 放入日志、分析事件、崩溃报告、提示或客户端可见的错误字符串中。
- 切勿在 URL、浏览器存储、截图或复制的示例载荷中暴露 PHI。
- 对 PHI 的读取和写入需要经过身份验证的访问、范围化授权和审计跟踪。
- 将第三方 SaaS、可观测性工具、支持工具和 LLM 提供商默认视为受阻，直到 BAA 状态和数据边界明确为止。
- 遵循最小必要访问原则：合适的用户只能看到完成任务所需的最小 PHI 片段。
- 优先使用不透明的内部 ID，而非姓名、病历号、电话号码、地址或其他标识符。

## 示例

### 示例 1：以 HIPAA 为框架的产品请求

用户请求：

> 为我们的临床医生仪表板添加 AI 生成的就诊摘要。我们为美国诊所提供服务，需要保持 HIPAA 合规。

响应模式：

- 激活 `hipaa-compliance`
- 使用 `healthcare-phi-compliance` 审查 PHI 流转、日志记录、存储和提示边界
- 在发送任何 PHI 之前，验证摘要提供商是否已签订 BAA
- 如果摘要影响临床决策，则升级到 `healthcare-reviewer`

### 示例 2：供应商/工具决策

用户请求：

> 我们可以将支持记录和患者消息发送到我们的分析平台吗？

响应模式：

- 假设这些消息可能包含 PHI
- 除非分析供应商已获批用于 HIPAA 受限工作负载且数据路径已最小化，否则阻止该设计
- 尽可能要求脱敏或使用非 PHI 事件模型

## 相关技能

- `healthcare-phi-compliance`
- `healthcare-reviewer`
- `healthcare-emr-patterns`
- `healthcare-eval-harness`
- `security-review`
