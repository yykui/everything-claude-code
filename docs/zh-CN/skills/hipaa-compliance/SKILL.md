---
name: hipaa-compliance
description: 医疗隐私和安全工作中专属于 HIPAA 的入口技能。当任务明确涉及 HIPAA、PHI 处理、受保护实体、BAA、违规态势或美国医疗合规要求时使用。
origin: ECC direct-port adaptation
version: "1.0.0"
---

# HIPAA 合规

当任务明确涉及美国医疗合规时，将本技能作为 HIPAA 专属入口使用。本技能有意保持精简和规范：

- `healthcare-phi-compliance` 仍是处理 PHI/PII、数据分类、审计日志、加密和泄露防护的主要实现技能。
- `healthcare-reviewer` 仍是当代码、架构或产品行为需要具备医疗知识的二次审查时使用的专项审查员。
- `security-review` 仍适用于通用的身份验证、输入处理、密钥管理、API 和部署加固。

## 使用时机

- 请求中明确提到 HIPAA、PHI、受保护实体、业务伙伴或 BAA
- 构建或审查存储、处理、导出或传输 PHI 的美国医疗软件
- 评估日志记录、分析工具、LLM 提示词、存储或支持工作流是否产生 HIPAA 暴露风险
- 设计面向患者或临床医生的系统，其中最小必要访问原则和可审计性至关重要

## 工作原理

将 HIPAA 视为叠加在更广泛医疗隐私技能之上的合规层：

1. 从 `healthcare-phi-compliance` 入手，获取具体的实现规则。
2. 应用 HIPAA 专属决策门禁：
   - 该数据是否属于 PHI？
   - 该主体是否为受保护实体或业务伙伴？
   - 供应商或模型提供商在接触数据之前是否需要签署 BAA？
   - 访问是否限制在最小必要范围内？
   - 读取/写入/导出事件是否可审计？
3. 如果任务影响患者安全、临床工作流或受监管的生产架构，升级至 `healthcare-reviewer`。

## HIPAA 专属安全准则

- 绝对不要将 PHI 放入日志、分析事件、崩溃报告、提示词或客户端可见的错误字符串中。
- 绝对不要在 URL、浏览器存储、截图或复制的示例载荷中暴露 PHI。
- PHI 的读取和写入操作必须经过认证访问、范围授权和审计追踪。
- 在 BAA 状态和数据边界明确之前，第三方 SaaS、可观测性工具、支持工具和 LLM 提供商默认视为禁止使用。
- 遵循最小必要访问原则：合适的用户只能看到完成任务所需的最小 PHI 切片。
- 优先使用不透明的内部 ID，而非姓名、MRN、电话号码、地址或其他标识符。

## 示例

### 示例 1：以 HIPAA 为框架的产品需求

用户请求：

> 在我们的临床医生仪表板上添加 AI 生成的就诊摘要。我们服务于美国诊所，需要保持 HIPAA 合规。

响应模式：

- 激活 `hipaa-compliance`
- 使用 `healthcare-phi-compliance` 审查 PHI 的流转、日志记录、存储和提示词边界
- 在发送任何 PHI 之前，验证摘要生成提供商是否已签署 BAA
- 如果摘要会影响临床决策，升级至 `healthcare-reviewer`

### 示例 2：供应商/工具决策

用户请求：

> 我们能否将支持对话记录和患者消息发送到我们的分析平台？

响应模式：

- 假设这些消息可能包含 PHI
- 除非分析供应商已获批处理 HIPAA 相关工作负载且数据路径已最小化，否则阻断该设计
- 尽可能要求对数据进行脱敏处理，或采用不含 PHI 的事件模型

## 相关技能

- `healthcare-phi-compliance`
- `healthcare-reviewer`
- `healthcare-emr-patterns`
- `healthcare-eval-harness`
- `security-review`
