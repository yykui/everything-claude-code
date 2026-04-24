---
name: autonomous-agent-harness
description: 将 Claude Code 转变为具有持久记忆、计划操作、计算机使用和任务队列的完全自主智能体系统。通过利用 Claude Code 的原生 cron、dispatch、MCP 工具和记忆来替代独立智能体框架（Hermes、AutoGPT）。当用户希望持续自主操作、计划任务或自我导向智能体循环时使用。
origin: ECC
---

# 自主智能体运行时

仅使用原生功能和 MCP 服务器将 Claude Code 变成一个持久的、自我导向的智能体系统。

## 何时激活

- 用户希望智能体持续运行或按计划运行
- 设置定期触发的自动化工作流
- 构建跨会话记忆上下文的个人 AI 助手
- 用户说"每天运行这个"、"定期检查这个"、"持续监控"
- 希望复制 Hermes、AutoGPT 或类似自主智能体框架的功能
- 需要将计算机使用与计划执行相结合

## 架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Claude Code Runtime                        │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │  Crons   │  │ Dispatch │  │ Memory   │  │ Computer    │ │
│  │ Schedule │  │ Remote   │  │ Store    │  │ Use         │ │
│  │ Tasks    │  │ Agents   │  │          │  │             │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘ │
│       │              │             │                │        │
│       ▼              ▼             ▼                ▼        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              ECC Skill + Agent Layer                  │    │
│  │                                                      │    │
│  │  skills/     agents/     commands/     hooks/        │    │
│  └──────────────────────────────────────────────────────┘    │
│       │              │             │                │        │
│       ▼              ▼             ▼                ▼        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              MCP Server Layer                        │    │
│  │                                                      │    │
│  │  memory    github    exa    supabase    browser-use  │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. 持久记忆

使用 Claude Code 内置记忆系统，并通过 MCP 记忆服务器增强结构化数据存储。

**内置记忆**（`~/.claude/projects/*/memory/`）：
- 用户偏好、反馈、项目上下文
- 以带前置信息的 Markdown 文件存储
- 会话开始时自动加载

**MCP 记忆服务器**（结构化知识图谱）：
- 实体、关系、观察
- 可查询的图结构
- 跨会话持久化

**记忆模式：**

```
# Short-term: current session context
Use TodoWrite for in-session task tracking

# Medium-term: project memory files
Write to ~/.claude/projects/*/memory/ for cross-session recall

# Long-term: MCP knowledge graph
Use mcp__memory__create_entities for permanent structured data
Use mcp__memory__create_relations for relationship mapping
Use mcp__memory__add_observations for new facts about known entities
```

### 2. 计划操作（Cron）

使用 Claude Code 的计划任务创建周期性智能体操作。

**设置 cron：**

```
# Via MCP tool
mcp__scheduled-tasks__create_scheduled_task({
  name: "daily-pr-review",
  schedule: "0 9 * * 1-5",  # 9 AM weekdays
  prompt: "Review all open PRs in affaan-m/everything-claude-code. For each: check CI status, review changes, flag issues. Post summary to memory.",
  project_dir: "/path/to/repo"
})

# Via claude -p (programmatic mode)
echo "Review open PRs and summarize" | claude -p --project /path/to/repo
```

**常用 cron 模式：**

| 模式 | 计划 | 使用场景 |
|---------|----------|----------|
| 每日站会 | `0 9 * * 1-5` | 审查 PR、问题、部署状态 |
| 每周回顾 | `0 10 * * 1` | 代码质量指标、测试覆盖率 |
| 每小时监控 | `0 * * * *` | 生产健康状态、错误率检查 |
| 夜间构建 | `0 2 * * *` | 运行完整测试套件、安全扫描 |
| 会前准备 | `*/30 * * * *` | 为即将到来的会议准备上下文 |

### 3. Dispatch / 远程智能体

远程触发 Claude Code 智能体以实现事件驱动工作流。

**Dispatch 模式：**

```bash
# Trigger from CI/CD
curl -X POST "https://api.anthropic.com/dispatch" \
  -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  -d '{"prompt": "Build failed on main. Diagnose and fix.", "project": "/repo"}'

# Trigger from webhook
# GitHub webhook → dispatch → Claude agent → fix → PR

# Trigger from another agent
claude -p "Analyze the output of the security scan and create issues for findings"
```

### 4. 计算机使用

利用 Claude 的 computer-use MCP 与物理世界交互。

**能力：**
- 浏览器自动化（导航、点击、填写表单、截图）
- 桌面控制（打开应用、输入、鼠标控制）
- CLI 之外的文件系统操作

**在运行时中的使用场景：**
- Web UI 自动化测试
- 表单填写和数据输入
- 基于截图的监控
- 多应用工作流

### 5. 任务队列

管理跨会话边界的持久任务队列。

**实现：**

```
# Task persistence via memory
Write task queue to ~/.claude/projects/*/memory/task-queue.md

# Task format
---
name: task-queue
type: project
description: Persistent task queue for autonomous operation
---

## Active Tasks
- [ ] PR #123: Review and approve if CI green
- [ ] Monitor deploy: check /health every 30 min for 2 hours
- [ ] Research: Find 5 leads in AI tooling space

## Completed
- [x] Daily standup: reviewed 3 PRs, 2 issues
```

## 替换 Hermes

| Hermes 组件 | ECC 等价物 | 方式 |
|------------------|---------------|-----|
| 网关/路由 | Claude Code dispatch + crons | 计划任务触发智能体会话 |
| 记忆系统 | Claude 记忆 + MCP 记忆服务器 | 内置持久化 + 知识图谱 |
| 工具注册表 | MCP 服务器 | 动态加载的工具提供者 |
| 编排 | ECC 技能 + 智能体 | 技能定义指导智能体行为 |
| 计算机使用 | computer-use MCP | 原生浏览器和桌面控制 |
| 上下文管理器 | 会话管理 + 记忆 | ECC 2.0 会话生命周期 |
| 任务队列 | 记忆持久化任务列表 | TodoWrite + 记忆文件 |

## 设置指南

### 第一步：配置 MCP 服务器

确保这些在 `~/.claude.json` 中：

```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@anthropic/memory-mcp-server"]
    },
    "scheduled-tasks": {
      "command": "npx",
      "args": ["-y", "@anthropic/scheduled-tasks-mcp-server"]
    },
    "computer-use": {
      "command": "npx",
      "args": ["-y", "@anthropic/computer-use-mcp-server"]
    }
  }
}
```

### 第二步：创建基础 Cron

```bash
# Daily morning briefing
claude -p "Create a scheduled task: every weekday at 9am, review my GitHub notifications, open PRs, and calendar. Write a morning briefing to memory."

# Continuous learning
claude -p "Create a scheduled task: every Sunday at 8pm, extract patterns from this week's sessions and update the learned skills."
```

### 第三步：初始化记忆图谱

```bash
# Bootstrap your identity and context
claude -p "Create memory entities for: me (user profile), my projects, my key contacts. Add observations about current priorities."
```

### 第四步：启用计算机使用（可选）

授予 computer-use MCP 浏览器和桌面控制所需的必要权限。

## 工作流示例

### 自主 PR 审查器
```
Cron: every 30 min during work hours
1. Check for new PRs on watched repos
2. For each new PR:
   - Pull branch locally
   - Run tests
   - Review changes with code-reviewer agent
   - Post review comments via GitHub MCP
3. Update memory with review status
```

### 个人研究智能体
```
Cron: daily at 6 AM
1. Check saved search queries in memory
2. Run Exa searches for each query
3. Summarize new findings
4. Compare against yesterday's results
5. Write digest to memory
6. Flag high-priority items for morning review
```

### 会议准备智能体
```
Trigger: 30 min before each calendar event
1. Read calendar event details
2. Search memory for context on attendees
3. Pull recent email/Slack threads with attendees
4. Prepare talking points and agenda suggestions
5. Write prep doc to memory
```

## 约束

- Cron 任务在独立会话中运行——除非通过记忆，否则它们不与交互式会话共享上下文。
- 计算机使用需要明确的权限授予。不要假设访问权限。
- 远程 dispatch 可能有频率限制。设计 cron 时使用适当的间隔。
- 记忆文件应保持简洁。归档旧数据，而不是让文件无限增长。
- 始终验证计划任务是否成功完成。在 cron 提示中添加错误处理。
