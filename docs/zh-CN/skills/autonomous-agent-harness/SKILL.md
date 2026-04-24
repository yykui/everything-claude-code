---
name: autonomous-agent-harness
description: 通过持久记忆、定时操作、计算机使用和任务队列，将 Claude Code 转变为完全自主的代理系统。通过利用 Claude Code 原生的定时任务、分发、MCP 工具和记忆，替代独立代理框架（Hermes、AutoGPT）。当用户需要持续自主运行、定时任务或自我引导的代理循环时使用。
origin: ECC
---

# 自主代理运行框架

仅使用原生功能和 MCP 服务器，将 Claude Code 转变为持久化、自我引导的代理系统。

## 何时激活

- 用户希望代理持续运行或按计划运行
- 设置定期触发的自动化工作流
- 构建跨会话记忆上下文的个人 AI 助手
- 用户说"每天运行这个"、"定期检查这个"、"持续监控"
- 希望复制 Hermes、AutoGPT 或类似自主代理框架的功能
- 需要将计算机使用与定时执行结合

## 架构

```
┌──────────────────────────────────────────────────────────────┐
│                    Claude Code 运行时                         │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐ │
│  │  定时任务 │  │  分发    │  │  记忆    │  │  计算机     │ │
│  │  调度    │  │  远程    │  │  存储    │  │  使用       │ │
│  │  任务    │  │  代理    │  │          │  │             │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬──────┘ │
│       │              │             │                │        │
│       ▼              ▼             ▼                ▼        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              ECC 技能 + 代理层                        │    │
│  │                                                      │    │
│  │  skills/     agents/     commands/     hooks/        │    │
│  └──────────────────────────────────────────────────────┘    │
│       │              │             │                │        │
│       ▼              ▼             ▼                ▼        │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              MCP 服务器层                             │    │
│  │                                                      │    │
│  │  memory    github    exa    supabase    browser-use  │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. 持久记忆

使用 Claude Code 内置记忆系统，通过 MCP 记忆服务器增强结构化数据存储。

**内置记忆**（`~/.claude/projects/*/memory/`）：
- 用户偏好、反馈、项目上下文
- 以带有 frontmatter 的 Markdown 文件存储
- 会话启动时自动加载

**MCP 记忆服务器**（结构化知识图谱）：
- 实体、关系、观察
- 可查询的图结构
- 跨会话持久化

**记忆模式：**

```
# 短期：当前会话上下文
使用 TodoWrite 进行会话内任务跟踪

# 中期：项目记忆文件
写入 ~/.claude/projects/*/memory/ 以实现跨会话回忆

# 长期：MCP 知识图谱
使用 mcp__memory__create_entities 存储永久结构化数据
使用 mcp__memory__create_relations 进行关系映射
使用 mcp__memory__add_observations 添加已知实体的新事实
```

### 2. 定时操作（Cron）

使用 Claude Code 的定时任务创建周期性代理操作。

**设置定时任务：**

```
# 通过 MCP 工具
mcp__scheduled-tasks__create_scheduled_task({
  name: "daily-pr-review",
  schedule: "0 9 * * 1-5",  # 工作日上午 9 点
  prompt: "Review all open PRs in affaan-m/everything-claude-code. For each: check CI status, review changes, flag issues. Post summary to memory.",
  project_dir: "/path/to/repo"
})

# 通过 claude -p（编程模式）
echo "Review open PRs and summarize" | claude -p --project /path/to/repo
```

**常用 Cron 模式：**

| 模式 | 调度 | 使用场景 |
|---------|----------|----------|
| 每日站会 | `0 9 * * 1-5` | 审查 PR、Issues、部署状态 |
| 每周回顾 | `0 10 * * 1` | 代码质量指标、测试覆盖率 |
| 每小时监控 | `0 * * * *` | 生产健康状态、错误率检查 |
| 夜间构建 | `0 2 * * *` | 运行完整测试套件、安全扫描 |
| 会前准备 | `*/30 * * * *` | 为即将召开的会议准备上下文 |

### 3. 分发/远程代理

远程触发 Claude Code 代理以实现事件驱动工作流。

**分发模式：**

```bash
# 从 CI/CD 触发
curl -X POST "https://api.anthropic.com/dispatch" \
  -H "Authorization: Bearer $ANTHROPIC_API_KEY" \
  -d '{"prompt": "Build failed on main. Diagnose and fix.", "project": "/repo"}'

# 从 Webhook 触发
# GitHub webhook → 分发 → Claude 代理 → 修复 → PR

# 从另一个代理触发
claude -p "Analyze the output of the security scan and create issues for findings"
```

### 4. 计算机使用

利用 Claude 的 computer-use MCP 与物理世界交互。

**能力：**
- 浏览器自动化（导航、点击、填写表单、截图）
- 桌面控制（打开应用、输入、鼠标控制）
- 超越 CLI 的文件系统操作

**运行框架中的使用场景：**
- Web UI 自动化测试
- 表单填写和数据录入
- 基于截图的监控
- 多应用工作流

### 5. 任务队列

管理跨会话边界持久存在的任务队列。

**实现：**

```
# 通过记忆持久化任务
将任务队列写入 ~/.claude/projects/*/memory/task-queue.md

# 任务格式
---
name: task-queue
type: project
description: Persistent task queue for autonomous operation
---

## 活跃任务
- [ ] PR #123：如果 CI 通过则审查并批准
- [ ] 监控部署：每 30 分钟检查 /health，持续 2 小时
- [ ] 研究：在 AI 工具领域找到 5 条线索

## 已完成
- [x] 每日站会：审查了 3 个 PR，2 个 Issue
```

## 替代 Hermes

| Hermes 组件 | ECC 等价物 | 实现方式 |
|------------------|---------------|-----|
| 网关/路由器 | Claude Code 分发 + Cron | 定时任务触发代理会话 |
| 记忆系统 | Claude 记忆 + MCP 记忆服务器 | 内置持久化 + 知识图谱 |
| 工具注册表 | MCP 服务器 | 动态加载的工具提供商 |
| 编排 | ECC 技能 + 代理 | 技能定义引导代理行为 |
| 计算机使用 | computer-use MCP | 原生浏览器和桌面控制 |
| 上下文管理器 | 会话管理 + 记忆 | ECC 2.0 会话生命周期 |
| 任务队列 | 记忆持久化任务列表 | TodoWrite + 记忆文件 |

## 安装指南

### 第一步：配置 MCP 服务器

确保以下内容在 `~/.claude.json` 中：

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

### 第二步：创建基础 Cron 任务

```bash
# 每日早报
claude -p "Create a scheduled task: every weekday at 9am, review my GitHub notifications, open PRs, and calendar. Write a morning briefing to memory."

# 持续学习
claude -p "Create a scheduled task: every Sunday at 8pm, extract patterns from this week's sessions and update the learned skills."
```

### 第三步：初始化记忆图谱

```bash
# 引导你的身份和上下文
claude -p "Create memory entities for: me (user profile), my projects, my key contacts. Add observations about current priorities."
```

### 第四步：启用计算机使用（可选）

授予 computer-use MCP 浏览器和桌面控制所需的必要权限。

## 示例工作流

### 自主 PR 审查员
```
Cron：工作时间每 30 分钟
1. 检查受监控仓库的新 PR
2. 对每个新 PR：
   - 本地拉取分支
   - 运行测试
   - 使用 code-reviewer 代理审查变更
   - 通过 GitHub MCP 发布审查评论
3. 将审查状态更新到记忆
```

### 个人研究代理
```
Cron：每天早上 6 点
1. 检查记忆中保存的搜索查询
2. 对每个查询运行 Exa 搜索
3. 总结新发现
4. 与昨天的结果进行比较
5. 将摘要写入记忆
6. 标记高优先级项目以供早间审查
```

### 会议准备代理
```
触发：每个日历事件前 30 分钟
1. 读取日历事件详情
2. 在记忆中搜索参与者的上下文
3. 拉取与参与者的近期邮件/Slack 线程
4. 准备讨论要点和议程建议
5. 将准备文档写入记忆
```

## 限制

- Cron 任务在隔离会话中运行——除非通过记忆，否则它们不与交互式会话共享上下文。
- 计算机使用需要明确的权限授予。不要假定有访问权限。
- 远程分发可能有频率限制。设计 Cron 时使用适当的时间间隔。
- 记忆文件应保持简洁。归档旧数据，而非让文件无限增长。
- 始终验证定时任务是否成功完成。在 Cron 提示词中添加错误处理。
