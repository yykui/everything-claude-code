---
name: safety-guard
description: 在生产系统上工作或自主运行 agent 时，使用此技能防止破坏性操作。
origin: ECC
---

# 安全防护 — 防止破坏性操作

## 何时使用

- 在生产系统上工作时
- agent 自主运行（全自动模式）时
- 希望将编辑限制在特定目录时
- 敏感操作期间（迁移、部署、数据变更）

## 工作原理

三种防护模式：

### 模式一：谨慎模式

在执行前拦截破坏性命令并发出警告：

```
监控的命令模式：
- rm -rf（尤其是 /、~ 或项目根目录）
- git push --force
- git reset --hard
- git checkout .（丢弃所有变更）
- DROP TABLE / DROP DATABASE
- docker system prune
- kubectl delete
- chmod 777
- sudo rm
- npm publish（意外发布）
- 任何带有 --no-verify 的命令
```

检测到时：显示命令的作用，请求确认，建议更安全的替代方案。

### 模式二：冻结模式

将文件编辑锁定到特定目录树：

```
/safety-guard freeze src/components/
```

任何在 `src/components/` 之外的写入/编辑操作都会被阻止并给出解释。适用于希望 agent 专注于某一区域而不触碰无关代码的场景。

### 模式三：防护模式（谨慎 + 冻结组合）

两种防护同时生效。为自主 agent 提供最高安全级别。

```
/safety-guard guard --dir src/api/ --allow-read-all
```

agent 可以读取任何内容，但只能写入 `src/api/`。破坏性命令在所有位置均被阻止。

### 解除防护

```
/safety-guard off
```

## 实现

使用 PreToolUse 钩子拦截 Bash、Write、Edit 和 MultiEdit 工具调用。在允许执行前根据活跃规则检查命令/路径。

## 集成

- 默认为 `codex -a never` 会话启用
- 与 ECC 2.0 中的可观测性风险评分配合使用
- 将所有被阻止的操作记录到 `~/.claude/safety-guard.log`
