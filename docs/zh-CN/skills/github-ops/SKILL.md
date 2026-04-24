---
name: github-ops
description: GitHub 仓库操作、自动化与管理。使用 gh CLI 进行问题分类、PR 管理、CI/CD 运营、版本发布管理和安全监控。当用户需要管理 GitHub issues、PR、CI 状态、版本发布、贡献者、过期条目或任何超出简单 git 命令的 GitHub 操作任务时使用。
origin: ECC
---

# GitHub 操作

管理 GitHub 仓库，重点关注社区健康、CI 可靠性和贡献者体验。

## 何时激活

- 分类 issues（分类、打标签、回复、去重）
- 管理 PR（审查状态、CI 检查、过期 PR、合并就绪情况）
- 调试 CI/CD 失败
- 准备版本发布和变更日志
- 监控 Dependabot 和安全警报
- 管理开源项目的贡献者体验
- 用户说"检查 GitHub"、"分类 issues"、"审查 PR"、"合并"、"发布"、"CI 坏了"

## 工具要求

- **gh CLI** 用于所有 GitHub API 操作
- 通过 `gh auth login` 配置仓库访问权限

## Issue 分类

按类型和优先级对每个 issue 进行分类：

**类型：** bug、feature-request、question、documentation、enhancement、duplicate、invalid、good-first-issue

**优先级：** critical（破坏性/安全）、high（重大影响）、medium（锦上添花）、low（外观问题）

### 分类工作流

1. 阅读 issue 标题、正文和评论
2. 检查是否与现有 issue 重复（按关键词搜索）
3. 通过 `gh issue edit --add-label` 应用适当的标签
4. 对于问题类 issue：起草并发布有帮助的回复
5. 对于需要更多信息的 bug：要求提供复现步骤
6. 对于适合新手的 issue：添加 `good-first-issue` 标签
7. 对于重复 issue：评论并附上原始 issue 链接，添加 `duplicate` 标签

```bash
# 搜索潜在重复 issue
gh issue list --search "keyword" --state all --limit 20

# 添加标签
gh issue edit <number> --add-label "bug,high-priority"

# 在 issue 上评论
gh issue comment <number> --body "Thanks for reporting. Could you share reproduction steps?"
```

## PR 管理

### 审查清单

1. 检查 CI 状态：`gh pr checks <number>`
2. 检查是否可合并：`gh pr view <number> --json mergeable`
3. 检查创建时间和最近活动
4. 标记超过 5 天未审查的 PR
5. 对于社区 PR：确保有测试并遵循规范

### 过期策略

- 14 天以上无活动的 issue：添加 `stale` 标签，评论询问进展
- 7 天以上无活动的 PR：评论询问是否仍在进行
- 30 天无响应后自动关闭过期 issue（添加 `closed-stale` 标签）

```bash
# 查找过期 issue（14 天以上无活动）
gh issue list --label "stale" --state open

# 查找近期无活动的 PR
gh pr list --json number,title,updatedAt --jq '.[] | select(.updatedAt < "2026-03-01")'
```

## CI/CD 操作

当 CI 失败时：

1. 检查工作流运行：`gh run view <run-id> --log-failed`
2. 识别失败步骤
3. 判断是不稳定测试还是真实失败
4. 对于真实失败：找出根本原因并建议修复方案
5. 对于不稳定测试：记录该模式供后续调查

```bash
# 列出最近失败的运行
gh run list --status failure --limit 10

# 查看失败运行日志
gh run view <run-id> --log-failed

# 重新运行失败的工作流
gh run rerun <run-id> --failed
```

## 版本发布管理

准备版本发布时：

1. 确认 main 分支所有 CI 均为绿色
2. 审查未发布的变更：`gh pr list --state merged --base main`
3. 从 PR 标题生成变更日志
4. 创建版本发布：`gh release create`

```bash
# 列出上次发布以来已合并的 PR
gh pr list --state merged --base main --search "merged:>2026-03-01"

# 创建版本发布
gh release create v1.2.0 --title "v1.2.0" --generate-notes

# 创建预发布版本
gh release create v1.3.0-rc1 --prerelease --title "v1.3.0 Release Candidate 1"
```

## 安全监控

```bash
# 检查 Dependabot 警报
gh api repos/{owner}/{repo}/dependabot/alerts --jq '.[].security_advisory.summary'

# 检查密钥扫描警报
gh api repos/{owner}/{repo}/secret-scanning/alerts --jq '.[].state'

# 审查并自动合并安全的依赖升级
gh pr list --label "dependencies" --json number,title
```

- 审查并自动合并安全的依赖升级
- 立即标记任何严重/高危警报
- 至少每周检查一次新的 Dependabot 警报

## 质量门禁

完成任何 GitHub 操作任务之前：
- 所有已分类的 issue 都有适当的标签
- 没有超过 7 天未经审查或评论的 PR
- CI 失败已被调查（不只是重新运行）
- 版本发布包含准确的变更日志
- 安全警报已被确认并跟踪
