---
name: jira-integration
description: 当需要获取 Jira 工单、分析需求、更新工单状态、添加评论或转换问题状态时使用此技能。提供通过 MCP 或直接 REST 调用的 Jira API 模式。
origin: ECC
---

# Jira 集成技能

直接从 AI 编程工作流中获取、分析和更新 Jira 工单。支持 **基于 MCP**（推荐）和**直接 REST API** 两种方式。

## 何时激活

- 获取 Jira 工单以理解需求
- 从工单中提取可测试的验收标准
- 向 Jira 问题添加进度评论
- 转换工单状态（待处理 → 进行中 → 已完成）
- 将合并请求或分支关联到 Jira 问题
- 通过 JQL 查询搜索问题

## 前置条件

### 方案 A：MCP 服务器（推荐）

安装 `mcp-atlassian` MCP 服务器。这将直接向 AI 智能体暴露 Jira 工具。

**要求：**
- Python 3.10+
- `uvx`（来自 `uv`），通过包管理器或官方 `uv` 安装文档安装

**添加到您的 MCP 配置**（例如 `~/.claude.json` → `mcpServers`）：

```json
{
  "jira": {
    "command": "uvx",
    "args": ["mcp-atlassian==0.21.0"],
    "env": {
      "JIRA_URL": "https://YOUR_ORG.atlassian.net",
      "JIRA_EMAIL": "your.email@example.com",
      "JIRA_API_TOKEN": "your-api-token"
    },
    "description": "Jira issue tracking — search, create, update, comment, transition"
  }
}
```

> **安全提示：** 切勿硬编码密钥。优先在系统环境（或密钥管理器）中设置 `JIRA_URL`、`JIRA_EMAIL` 和 `JIRA_API_TOKEN`。MCP `env` 块仅用于本地未提交的配置文件。

**获取 Jira API 令牌：**
1. 访问 <https://id.atlassian.com/manage-profile/security/api-tokens>
2. 点击**创建 API 令牌**
3. 复制令牌——存储在您的环境中，切勿放入源代码

### 方案 B：直接 REST API

如果 MCP 不可用，可通过 `curl` 或辅助脚本直接使用 Jira REST API v3。

**所需环境变量：**

| 变量 | 描述 |
|------|------|
| `JIRA_URL` | 您的 Jira 实例 URL（例如 `https://yourorg.atlassian.net`） |
| `JIRA_EMAIL` | 您的 Atlassian 账户邮箱 |
| `JIRA_API_TOKEN` | 来自 id.atlassian.com 的 API 令牌 |

将这些变量存储在 shell 环境、密钥管理器或未跟踪的本地 env 文件中。不要提交到仓库。

## MCP 工具参考

当 `mcp-atlassian` MCP 服务器配置完成后，以下工具可用：

| 工具 | 用途 | 示例 |
|------|------|------|
| `jira_search` | JQL 查询 | `project = PROJ AND status = "In Progress"` |
| `jira_get_issue` | 通过工单键获取完整问题详情 | `PROJ-1234` |
| `jira_create_issue` | 创建问题（任务、Bug、故事、Epic） | 新 Bug 报告 |
| `jira_update_issue` | 更新字段（摘要、描述、负责人） | 更改负责人 |
| `jira_transition_issue` | 更改状态 | 移动到"审查中" |
| `jira_add_comment` | 添加评论 | 进度更新 |
| `jira_get_sprint_issues` | 列出 Sprint 中的问题 | 活动 Sprint 审查 |
| `jira_create_issue_link` | 关联问题（阻塞、关联） | 依赖追踪 |
| `jira_get_issue_development_info` | 查看关联的 PR、分支、提交 | 开发上下文 |

> **提示：** 在转换状态之前，始终调用 `jira_get_transitions`——每个项目工作流的转换 ID 各不相同。

## 直接 REST API 参考

### 获取工单

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/3/issue/PROJ-1234" | jq '{
    key: .key,
    summary: .fields.summary,
    status: .fields.status.name,
    priority: .fields.priority.name,
    type: .fields.issuetype.name,
    assignee: .fields.assignee.displayName,
    labels: .fields.labels,
    description: .fields.description
  }'
```

### 获取评论

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  "$JIRA_URL/rest/api/3/issue/PROJ-1234?fields=comment" | jq '.fields.comment.comments[] | {
    author: .author.displayName,
    created: .created[:10],
    body: .body
  }'
```

### 添加评论

```bash
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "body": {
      "version": 1,
      "type": "doc",
      "content": [{
        "type": "paragraph",
        "content": [{"type": "text", "text": "Your comment here"}]
      }]
    }
  }' \
  "$JIRA_URL/rest/api/3/issue/PROJ-1234/comment"
```

### 转换工单状态

```bash
# 1. 获取可用的转换
curl -s -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  "$JIRA_URL/rest/api/3/issue/PROJ-1234/transitions" | jq '.transitions[] | {id, name: .name}'

# 2. 执行转换（替换 TRANSITION_ID）
curl -s -X POST -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"transition": {"id": "TRANSITION_ID"}}' \
  "$JIRA_URL/rest/api/3/issue/PROJ-1234/transitions"
```

### 使用 JQL 搜索

```bash
curl -s -G -u "$JIRA_EMAIL:$JIRA_API_TOKEN" \
  --data-urlencode "jql=project = PROJ AND status = 'In Progress'" \
  "$JIRA_URL/rest/api/3/search"
```

## 分析工单

在为开发或测试自动化获取工单时，提取以下内容：

### 1. 可测试的需求
- **功能需求** — 功能做什么
- **验收标准** — 必须满足的条件
- **可测试行为** — 特定操作和预期结果
- **用户角色** — 谁使用此功能及其权限
- **数据需求** — 需要什么数据
- **集成点** — 涉及的 API、服务或系统

### 2. 所需的测试类型
- **单元测试** — 独立的函数和工具
- **集成测试** — API 端点和服务交互
- **E2E 测试** — 面向用户的 UI 流程
- **API 测试** — 端点契约和错误处理

### 3. 边界情况和错误场景
- 无效输入（空值、过长、特殊字符）
- 未授权访问
- 网络故障或超时
- 并发用户或竞态条件
- 边界条件
- 缺失或空数据
- 状态转换（后退导航、刷新等）

### 4. 结构化分析输出

```
Ticket: PROJ-1234
Summary: [ticket title]
Status: [current status]
Priority: [High/Medium/Low]
Test Types: Unit, Integration, E2E

Requirements:
1. [requirement 1]
2. [requirement 2]

Acceptance Criteria:
- [ ] [criterion 1]
- [ ] [criterion 2]

Test Scenarios:
- Happy Path: [description]
- Error Case: [description]
- Edge Case: [description]

Test Data Needed:
- [data item 1]
- [data item 2]

Dependencies:
- [dependency 1]
- [dependency 2]
```

## 更新工单

### 何时更新

| 工作流步骤 | Jira 更新 |
|-----------|-----------|
| 开始工作 | 转换到"进行中" |
| 测试已编写 | 评论附上测试覆盖率摘要 |
| 分支已创建 | 评论附上分支名称 |
| PR/MR 已创建 | 评论附上链接，关联问题 |
| 测试通过 | 评论附上结果摘要 |
| PR/MR 已合并 | 转换到"已完成"或"审查中" |

### 评论模板

**开始工作：**
```
Starting implementation for this ticket.
Branch: feat/PROJ-1234-feature-name
```

**测试已实现：**
```
Automated tests implemented:

Unit Tests:
- [test file 1] — [what it covers]
- [test file 2] — [what it covers]

Integration Tests:
- [test file] — [endpoints/flows covered]

All tests passing locally. Coverage: XX%
```

**PR 已创建：**
```
Pull request created:
[PR Title](https://github.com/org/repo/pull/XXX)

Ready for review.
```

**工作已完成：**
```
Implementation complete.

PR merged: [link]
Test results: All passing (X/Y)
Coverage: XX%
```

## 安全指南

- **切勿硬编码** Jira API 令牌到源代码或技能文件中
- **始终使用**环境变量或密钥管理器
- 在每个项目中将 **`.env`** 添加到 `.gitignore`
- 如果令牌在 git 历史记录中暴露，立即**轮换令牌**
- 使用**最小权限** API 令牌，限定在所需项目范围内
- 在进行 API 调用之前**验证**凭据是否已设置——快速失败并给出清晰提示

## 故障排除

| 错误 | 原因 | 解决方法 |
|------|------|----------|
| `401 Unauthorized` | API 令牌无效或已过期 | 在 id.atlassian.com 重新生成 |
| `403 Forbidden` | 令牌缺少项目权限 | 检查令牌范围和项目访问权限 |
| `404 Not Found` | 工单键或基础 URL 错误 | 验证 `JIRA_URL` 和工单键 |
| `spawn uvx ENOENT` | IDE 无法在 PATH 中找到 `uvx` | 使用完整路径（如 `~/.local/bin/uvx`）或在 `~/.zprofile` 中设置 PATH |
| 连接超时 | 网络/VPN 问题 | 检查 VPN 连接和防火墙规则 |

## 最佳实践

- 随时更新 Jira，而不是最后一口气全部更新
- 评论简洁但信息丰富
- 使用链接而非复制——指向 PR、测试报告和仪表板
- 如需他人意见，使用 @提及
- 开始工作前检查关联问题以了解完整功能范围
- 如果验收标准模糊，在编写代码之前请求澄清
