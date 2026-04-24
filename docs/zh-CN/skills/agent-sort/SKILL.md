---
name: agent-sort
description: 通过将技能、命令、规则、钩子和附加项分类到 DAILY（每日）和 LIBRARY（库）两个桶中，利用并行仓库感知审查来构建基于证据的特定仓库 ECC 安装计划。当 ECC 应精简为项目实际所需而非加载完整包时使用。
origin: ECC
---

# 智能体排序

当仓库需要特定于项目的 ECC 界面而非默认完整安装时，使用此技能。

目标不是猜测什么"感觉有用"，而是用实际代码库的证据对 ECC 组件进行分类。

## 何时使用

- 项目只需要 ECC 的子集，完整安装噪音太多
- 仓库技术栈清晰，但没有人想逐一手动整理技能
- 团队希望基于 grep 证据而非主观意见做出可重复的安装决策
- 需要将始终加载的每日工作流界面与可搜索的库/参考界面分开
- 仓库已漂移到错误的语言、规则或钩子集，需要清理

## 不可违背的规则

- 以当前仓库为真相来源，而非通用偏好
- 每个 DAILY 决策必须引用具体的仓库证据
- LIBRARY 不意味着"删除"；它意味着"保持可访问但不默认加载"
- 不安装当前仓库无法使用的钩子、规则或脚本
- 优先使用 ECC 原生界面；不引入第二个安装系统

## 输出

按顺序生成以下产物：

1. DAILY 清单
2. LIBRARY 清单
3. 安装计划
4. 验证报告
5. 可选的 `skill-library` 路由器（如果项目需要）

## 分类模型

仅使用两个桶：

- `DAILY`
  - 应在此仓库的每个会话中加载
  - 与仓库的语言、框架、工作流或操作界面强匹配
- `LIBRARY`
  - 有保留价值，但不值得默认加载
  - 应通过搜索、路由技能或选择性手动使用保持可访问

## 证据来源

在进行任何分类之前使用仓库本地证据：

- 文件扩展名
- 包管理器和锁文件
- 框架配置
- CI 和钩子配置
- 构建/测试脚本
- 导入和依赖清单
- 明确描述技术栈的仓库文档

有用的命令包括：

```bash
rg --files
rg -n "typescript|react|next|supabase|django|spring|flutter|swift"
cat package.json
cat pyproject.toml
cat Cargo.toml
cat pubspec.yaml
cat go.mod
```

## 并行审查通道

如果并行子智能体可用，将审查分为以下通道：

1. 智能体
   - 分类 `agents/*`
2. 技能
   - 分类 `skills/*`
3. 命令
   - 分类 `commands/*`
4. 规则
   - 分类 `rules/*`
5. 钩子和脚本
   - 分类钩子界面、MCP 健康检查、辅助脚本和操作系统兼容性
6. 附加项
   - 分类上下文、示例、MCP 配置、模板和指导文档

如果子智能体不可用，按顺序运行相同的通道。

## 核心工作流

### 1. 读取仓库

在分类任何内容之前建立真实的技术栈：

- 使用的语言
- 使用的框架
- 主要包管理器
- 测试栈
- 代码检查/格式化栈
- 部署/运行时界面
- 已存在的操作集成

### 2. 构建证据表

对于每个候选界面，记录：

- 组件路径
- 组件类型
- 建议的桶
- 仓库证据
- 简短理由

使用此格式：

```text
skills/frontend-patterns | skill | DAILY | 84 .tsx files, next.config.ts present | core frontend stack
skills/django-patterns   | skill | LIBRARY | no .py files, no pyproject.toml       | not active in this repo
rules/typescript/*       | rules | DAILY | package.json + tsconfig.json            | active TS repo
rules/python/*           | rules | LIBRARY | zero Python source files             | keep accessible only
```

### 3. 决定 DAILY vs LIBRARY

提升到 `DAILY` 当：

- 仓库明确使用匹配的技术栈
- 组件足够通用，可以帮助每个会话
- 仓库已依赖对应的运行时或工作流

降级到 `LIBRARY` 当：

- 组件不在技术栈范围内
- 仓库以后可能需要它，但不是每天
- 它增加了上下文开销而没有即时相关性

### 4. 构建安装计划

将分类转化为操作：

- DAILY 技能 -> 安装或保留在 `.claude/skills/`
- DAILY 命令 -> 仅在仍然有用时保留为显式垫片
- DAILY 规则 -> 仅安装匹配的语言集
- DAILY 钩子/脚本 -> 只保留兼容的
- LIBRARY 界面 -> 通过搜索或 `skill-library` 保持可访问

如果仓库已使用选择性安装，更新该计划而不是创建另一个系统。

### 5. 创建可选库路由器

如果项目需要可搜索的库界面，创建：

- `.claude/skills/skill-library/SKILL.md`

该路由器应包含：

- DAILY vs LIBRARY 的简短说明
- 分组的触发关键词
- 库引用所在位置

不要在路由器中复制每个技能的完整内容。

### 6. 验证结果

应用计划后，验证：

- 每个 DAILY 文件存在于预期位置
- 过时的语言规则未被留为活跃状态
- 不兼容的钩子未被安装
- 最终安装实际上与仓库技术栈匹配

返回简洁报告，包含：

- DAILY 数量
- LIBRARY 数量
- 已删除的过时界面
- 待解决问题

## 交接

如果下一步是交互式安装或修复，交接给：

- `configure-ecc`

如果下一步是重叠清理或目录审查，交接给：

- `skill-stocktake`

如果下一步是更广泛的上下文精简，交接给：

- `strategic-compact`

## 输出格式

按此顺序返回结果：

```text
STACK
- language/framework/runtime summary

DAILY
- always-loaded items with evidence

LIBRARY
- searchable/reference items with evidence

INSTALL PLAN
- what should be installed, removed, or routed

VERIFICATION
- checks run and remaining gaps
```
