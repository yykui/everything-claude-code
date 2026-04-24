---
name: jira-integration
description: 当需要获取 Jira 工单、分析需求、更新工单状态、添加评论或转换问题状态时使用本技能。通过 MCP 或直接 REST 调用提供 Jira API 操作模式。
origin: ECC
---

# Jira 集成技能

直接在 AI 编码工作流中获取、分析和更新 Jira 工单。支持 **基于 MCP**（推荐）和**直接 REST API** 两种方式。

## 激活时机

- 获取 Jira 工单以了解需求
- 从工单中提取可测试的验收标准
- 向 Jira 问题添加进度评论
- 转换工单状态（待处理 → 进行中 → 已完成）
- 将合并请求或分支关联到 Jira 问题
- 使用 JQL 查询搜索问题

## 前置条件

### 方式 A：MCP 服务器（推荐）

安装 `mcp-atlassian` MCP 服务器，该服务器将 Jira 工具直接暴露给 AI 代理。

**要求：**
- Python 3.10+
- `uvx`（来自 `uv`），通过包管理器或官方 `uv` 安装文档安装

**添加到 MCP 配置**（例如 `~/.claude.json` → `mcpServers`）：

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

> **安全提示：** 永远不要硬编码密钥。优先将 `JIRA_URL`、`JIRA_EMAIL` 和 `JIRA_API_TOKEN` 设置为系统环境变量（或密钥管理器中）。MCP 的 `env` 块仅适用于本地未提交的配置文件。

**获取 Jira API 令牌：**
1. 访问 <https://id.atlassian.com/manage-profile/security/api-tokens>
2. 点击**创建 API 令牌**
3. 复制令牌——存储在环境变量中，切勿写入源代码

### 方式 B：直接 REST API

如果 MCP 不可用，可通过 `curl` 或辅助脚本直接使用 Jira REST API v3。

**所需环境变量：**

| 变量 | 描述 |
|------|------|
| `JIRA_URL` | 您的 Jira 实例 URL（例如 `https://yourorg.atlassian.net`） |
| `JIRA_EMAIL` | 您的 Atlassian 账户邮箱 |
| `JIRA_API_TOKEN` | 从 id.atlassian.com 获取的 API 令牌 |

将这些变量存储在 shell 环境、密钥管理器或未跟踪的本地 env 文件中。请勿提交到仓库。

## MCP 工具参考

配置 `mcp-atlassian` MCP 服务器后，可使用以下工具：

| 工具 | 用途 | 示例 |
|------|------|------|
| `jira_search` | JQL 查询 | `project = PROJ AND status = "In Progress"` |
| `jira_get_issue` | 按工单号获取完整问题详情 | `PROJ-1234` |
| `jira_create_issue` | 创建问题（任务、缺陷、用户故事、史诗） | 新建缺陷报告 |
| `jira_update_issue` | 更新字段（标题、描述、经办人） | 更改经办人 |
| `jira_transition_issue` | 变更状态 | 移至"审查中" |
| `jira_add_comment` | 添加评论 | 进度更新 |
| `jira_get_sprint_issues` | 列出 sprint 中的问题 | 活跃 sprint 回顾 |
| `jira_create_issue_link` | 关联问题（阻塞、关联） | 依赖关系追踪 |
| `jira_get_issue_development_info` | 查看关联的 PR、分支、提交 | 开发上下文 |

> **提示：** 在执行状态转换之前，始终调用 `jira_get_transitions`——不同项目工作流的转换 ID 各不相同。

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

获取工单用于开发或测试自动化时，提取以下内容：

### 1. 可测试的需求
- **功能需求** — 功能的作用
- **验收标准** — 必须满足的条件
- **可测试行为** — 具体操作和预期结果
- **用户角色** — 使用该功能的用户及其权限
- **数据需求** — 所需数据
- **集成点** — 涉及的 API、服务或系统

### 2. 所需测试类型
- **单元测试** — 独立函数和工具
- **集成测试** — API 端点和服务交互
- **端到端测试** — 面向用户的 UI 流程
- **API 测试** — 端点契约和错误处理

### 3. 边界情况和错误场景
- 无效输入（空值、过长、特殊字符）
- 未授权访问
- 网络故障或超时
- 并发用户或竞态条件
- 边界值
- 缺失或空数据
- 状态转换（返回导航、刷新等）

### 4. 结构化分析输出

```
Ticket: PROJ-1234
Summary: [工单标题]
Status: [当前状态]
Priority: [高/中/低]
Test Types: Unit, Integration, E2E

Requirements:
1. [需求 1]
2. [需求 2]

Acceptance Criteria:
- [ ] [标准 1]
- [ ] [标准 2]

Test Scenarios:
- Happy Path: [描述]
- Error Case: [描述]
- Edge Case: [描述]

Test Data Needed:
- [数据项 1]
- [数据项 2]

Dependencies:
- [依赖项 1]
- [依赖项 2]
```

## 更新工单

### 何时更新

| 工作流步骤 | Jira 更新 |
|-----------|-----------|
| 开始工作 | 转换至"进行中" |
| 测试已编写 | 评论测试覆盖情况摘要 |
| 分支已创建 | 评论分支名称 |
| PR/MR 已创建 | 评论附链接，关联问题 |
| 测试通过 | 评论测试结果摘要 |
| PR/MR 已合并 | 转换至"已完成"或"审查中" |

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
- [测试文件 1] — [覆盖内容]
- [测试文件 2] — [覆盖内容]

Integration Tests:
- [测试文件] — [覆盖的端点/流程]

All tests passing locally. Coverage: XX%
```

**PR 已创建：**
```
Pull request created:
[PR Title](https://github.com/org/repo/pull/XXX)

Ready for review.
```

**工作完成：**
```
Implementation complete.

PR merged: [链接]
Test results: All passing (X/Y)
Coverage: XX%
```

## 安全准则

- **绝不硬编码** Jira API 令牌到源代码或技能文件中
- **始终使用**环境变量或密钥管理器
- 每个项目都将 **`.env` 加入 `.gitignore`**
- 如果令牌在 git 历史记录中暴露，**立即轮换**
- 使用范围限定到所需项目的**最小权限** API 令牌
- 在进行 API 调用之前**验证**凭据已设置——快速失败并给出清晰的提示信息

## 故障排除

| 错误 | 原因 | 解决方法 |
|------|------|---------|
| `401 Unauthorized` | API 令牌无效或已过期 | 在 id.atlassian.com 重新生成 |
| `403 Forbidden` | 令牌缺少项目权限 | 检查令牌范围和项目访问权限 |
| `404 Not Found` | 工单号或基础 URL 有误 | 验证 `JIRA_URL` 和工单号 |
| `spawn uvx ENOENT` | IDE 无法在 PATH 中找到 `uvx` | 使用完整路径（如 `~/.local/bin/uvx`）或在 `~/.zprofile` 中设置 PATH |
| 连接超时 | 网络/VPN 问题 | 检查 VPN 连接和防火墙规则 |

## 最佳实践

- 随时更新 Jira，而非在最后集中更新
- 保持评论简洁但信息完整
- 通过链接而非复制——指向 PR、测试报告和仪表板
- 如需他人输入，使用 @ 提及
- 开始工作前查看关联问题，了解完整的功能范围
- 如果验收标准模糊，在编写代码之前要求澄清
