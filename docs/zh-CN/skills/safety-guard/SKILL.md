---
name: safety-guard
description: 在生产系统上工作或自主运行代理时，使用本技能防止破坏性操作。
origin: ECC
---

# 安全防护——防止破坏性操作

## 何时使用

- 在生产系统上工作时
- 代理以自主模式（全自动模式）运行时
- 需要将编辑限制在特定目录时
- 进行敏感操作时（迁移、部署、数据变更）

## 工作原理

三种保护模式：

### 模式 1：谨慎模式

在执行前拦截破坏性命令并发出警告：

```
Watched patterns:
- rm -rf (especially /, ~, or project root)
- git push --force
- git reset --hard
- git checkout . (discard all changes)
- DROP TABLE / DROP DATABASE
- docker system prune
- kubectl delete
- chmod 777
- sudo rm
- npm publish (accidental publishes)
- Any command with --no-verify
```

检测到时：显示命令的作用，要求确认，建议更安全的替代方案。

### 模式 2：冻结模式

将文件编辑锁定到特定目录树：

```
/safety-guard freeze src/components/
```

任何对 `src/components/` 以外的写入/编辑都将被阻止并附带解释。适用于希望代理专注于某个区域而不接触无关代码的场景。

### 模式 3：守卫模式（谨慎模式 + 冻结模式的组合）

两种保护均处于激活状态。自主代理的最高安全级别。

```
/safety-guard guard --dir src/api/ --allow-read-all
```

代理可以读取任何内容，但只能写入 `src/api/`。破坏性命令在任何地方都被阻止。

### 解锁

```
/safety-guard off
```

## 实现

使用 PreToolUse 钩子拦截 Bash、Write、Edit 和 MultiEdit 工具调用。在允许执行之前，根据激活的规则检查命令/路径。

## 集成

- 默认为 `codex -a never` 会话启用
- 与 ECC 2.0 中的可观测性风险评分配合使用
- 将所有被阻止的操作记录到 `~/.claude/safety-guard.log`
