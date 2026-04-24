---
name: git-workflow
description: Git 工作流模式，包括分支策略、提交规范、合并与变基、冲突解决，以及适用于各规模团队的协作开发最佳实践。
origin: ECC
---

# Git 工作流模式

Git 版本控制、分支策略和协作开发的最佳实践。

## 何时激活

- 为新项目建立 Git 工作流
- 决定分支策略（GitFlow、主干开发、GitHub flow）
- 编写提交信息和 PR 描述
- 解决合并冲突
- 管理发布和版本标签
- 向新团队成员传授 Git 实践

## 分支策略

### GitHub Flow（简单，适合大多数场景）

最适合持续部署和中小型团队。

```
main（受保护，始终可部署）
  │
  ├── feature/user-auth      → PR → 合并到 main
  ├── feature/payment-flow   → PR → 合并到 main
  └── fix/login-bug          → PR → 合并到 main
```

**规则：**
- `main` 始终可部署
- 从 `main` 创建功能分支
- 准备好审查时开启 Pull Request
- 审批通过且 CI 通过后，合并到 `main`
- 合并后立即部署

### 主干开发（高速团队）

最适合具有强大 CI/CD 和功能开关的团队。

```
main（主干）
  │
  ├── 短生命周期功能分支（最多 1-2 天）
  ├── 短生命周期功能分支
  └── 短生命周期功能分支
```

**规则：**
- 所有人提交到 `main` 或非常短生命周期的分支
- 使用功能开关隐藏未完成的工作
- CI 通过后方可合并
- 每天多次部署

### GitFlow（复杂，按发布周期驱动）

最适合定期发布和企业项目。

```
main（生产发布）
  │
  └── develop（集成分支）
        │
        ├── feature/user-auth
        ├── feature/payment
        │
        ├── release/1.0.0    → 合并到 main 和 develop
        │
        └── hotfix/critical  → 合并到 main 和 develop
```

**规则：**
- `main` 只包含生产就绪的代码
- `develop` 是集成分支
- 功能分支从 `develop` 创建，合并回 `develop`
- 发布分支从 `develop` 创建，合并到 `main` 和 `develop`
- 热修复分支从 `main` 创建，合并到 `main` 和 `develop`

### 如何选择

| 策略 | 团队规模 | 发布节奏 | 最适合 |
|----------|-----------|-----------------|----------|
| GitHub Flow | 任意规模 | 持续 | SaaS、Web 应用、初创公司 |
| 主干开发 | 5+ 有经验成员 | 每天多次 | 高速团队、功能开关 |
| GitFlow | 10+ | 定期 | 企业、受监管行业 |

## 提交信息

### 约定式提交格式

```
<类型>(<范围>): <主题>

[可选正文]

[可选页脚]
```

### 类型

| 类型 | 用途 | 示例 |
|------|---------|---------|
| `feat` | 新功能 | `feat(auth): add OAuth2 login` |
| `fix` | 错误修复 | `fix(api): handle null response in user endpoint` |
| `docs` | 文档 | `docs(readme): update installation instructions` |
| `style` | 格式化，无代码变更 | `style: fix indentation in login component` |
| `refactor` | 代码重构 | `refactor(db): extract connection pool to module` |
| `test` | 添加/更新测试 | `test(auth): add unit tests for token validation` |
| `chore` | 维护任务 | `chore(deps): update dependencies` |
| `perf` | 性能提升 | `perf(query): add index to users table` |
| `ci` | CI/CD 变更 | `ci: add PostgreSQL service to test workflow` |
| `revert` | 回滚之前的提交 | `revert: revert "feat(auth): add OAuth2 login"` |

### 好坏示例对比

```
# 坏：模糊，无上下文
git commit -m "fixed stuff"
git commit -m "updates"
git commit -m "WIP"

# 好：清晰、具体，说明原因
git commit -m "fix(api): retry requests on 503 Service Unavailable

The external API occasionally returns 503 errors during peak hours.
Added exponential backoff retry logic with max 3 attempts.

Closes #123"
```

### 提交信息模板

在仓库根目录创建 `.gitmessage`：

```
# <类型>(<范围>): <主题>
# # 类型: feat, fix, docs, style, refactor, test, chore, perf, ci, revert
# 范围: api, ui, db, auth 等
# 主题: 祈使语气，不加句号，最多 50 字符
#
# [可选正文] - 解释原因，而非内容
# [可选页脚] - 破坏性变更，关闭 #issue
```

通过以下命令启用：`git config commit.template .gitmessage`

## 合并 vs 变基

### 合并（保留历史）

```bash
# 创建合并提交
git checkout main
git merge feature/user-auth

# 结果：
# *   合并提交
# |\
# | * 功能分支提交
# |/
# * main 提交
```

**适用场景：**
- 将功能分支合并到 `main`
- 希望保留完整历史记录时
- 多人在该分支上协作时
- 分支已推送且其他人可能基于此开展工作时

### 变基（线性历史）

```bash
# 将功能分支提交重写到目标分支之上
git checkout feature/user-auth
git rebase main

# 结果：
# * 功能分支提交（已重写）
# * main 提交
```

**适用场景：**
- 用最新 `main` 更新本地功能分支时
- 希望保持线性、整洁的历史时
- 分支仅在本地（未推送）时
- 只有你在该分支上工作时

### 变基工作流

```bash
# 在提 PR 前，用最新 main 更新功能分支
git checkout feature/user-auth
git fetch origin
git rebase origin/main

# 解决任何冲突
# 测试应仍然通过

# 强制推送（仅当你是唯一贡献者时）
git push --force-with-lease origin feature/user-auth
```

### 不应变基的情况

```
# 永远不要对以下分支进行变基：
- 已推送到共享仓库的分支
- 其他人基于此开展工作的分支
- 受保护分支（main、develop）
- 已合并的分支

# 原因：变基会重写历史，破坏他人的工作
```

## Pull Request 工作流

### PR 标题格式

```
<类型>(<范围>): <描述>

示例：
feat(auth): add SSO support for enterprise users
fix(api): resolve race condition in order processing
docs(api): add OpenAPI specification for v2 endpoints
```

### PR 描述模板

```markdown
## 做了什么

简述此 PR 的内容。

## 为什么

说明动机和背景。

## 如何实现

值得重点说明的关键实现细节。

## 测试

- [ ] 添加/更新了单元测试
- [ ] 添加/更新了集成测试
- [ ] 已进行手动测试

## 截图（如适用）

UI 变更的前后截图。

## 检查清单

- [ ] 代码遵循项目风格规范
- [ ] 已完成自我审查
- [ ] 为复杂逻辑添加了注释
- [ ] 已更新文档
- [ ] 未引入新的警告
- [ ] 测试在本地通过
- [ ] 已关联相关 issue

Closes #123
```

### 代码审查清单

**审查者：**

- [ ] 代码是否解决了所述问题？
- [ ] 是否有未处理的边缘情况？
- [ ] 代码是否可读且易于维护？
- [ ] 是否有足够的测试？
- [ ] 是否存在安全隐患？
- [ ] 提交历史是否整洁（必要时已压缩）？

**作者：**

- [ ] 请求审查前已完成自我审查
- [ ] CI 通过（测试、Lint、类型检查）
- [ ] PR 大小合理（理想情况下 <500 行）
- [ ] 关联单一功能/修复
- [ ] 描述清晰说明了变更内容

## 冲突解决

### 识别冲突

```bash
# 合并前检查冲突
git checkout main
git merge feature/user-auth --no-commit --no-ff

# 如果有冲突，Git 会显示：
# CONFLICT (content): Merge conflict in src/auth/login.ts
# Automatic merge failed; fix conflicts and then commit the result.
```

### 解决冲突

```bash
# 查看冲突文件
git status

# 查看文件中的冲突标记
# <<<<<<< HEAD
# main 中的内容
# =======
# 功能分支中的内容
# >>>>>>> feature/user-auth

# 方式 1：手动解决
# 编辑文件，删除标记，保留正确内容

# 方式 2：使用合并工具
git mergetool

# 方式 3：接受某一方
git checkout --ours src/auth/login.ts    # 保留 main 版本
git checkout --theirs src/auth/login.ts  # 保留功能分支版本

# 解决后，暂存并提交
git add src/auth/login.ts
git commit
```

### 冲突预防策略

```bash
# 1. 保持功能分支小而短暂
# 2. 频繁对 main 进行变基
git checkout feature/user-auth
git fetch origin
git rebase origin/main

# 3. 与团队沟通哪些共享文件被修改
# 4. 使用功能开关替代长生命周期分支
# 5. 及时审查并合并 PR
```

## 分支管理

### 命名约定

```
# 功能分支
feature/user-authentication
feature/JIRA-123-payment-integration

# 错误修复
fix/login-redirect-loop
fix/456-null-pointer-exception

# 热修复（生产问题）
hotfix/critical-security-patch
hotfix/database-connection-leak

# 发布
release/1.2.0
release/2024-01-hotfix

# 实验/概念验证
experiment/new-caching-strategy
poc/graphql-migration
```

### 分支清理

```bash
# 删除已合并到 main 的本地分支
git branch --merged main | grep -v "^\*\|main" | xargs -n 1 git branch -d

# 删除已删除的远程分支的本地跟踪引用
git fetch -p

# 删除本地分支
git branch -d feature/user-auth  # 安全删除（仅在已合并时）
git branch -D feature/user-auth  # 强制删除

# 删除远程分支
git push origin --delete feature/user-auth
```

### 暂存工作流

```bash
# 保存进行中的工作
git stash push -m "WIP: user authentication"

# 列出暂存列表
git stash list

# 应用最近的暂存
git stash pop

# 应用指定暂存
git stash apply stash@{2}

# 删除暂存
git stash drop stash@{0}
```

## 发布管理

### 语义化版本

```
主版本.次版本.修订版本

主版本：破坏性变更
次版本：新功能，向后兼容
修订版本：错误修复，向后兼容

示例：
1.0.0 → 1.0.1（修订版本：错误修复）
1.0.1 → 1.1.0（次版本：新功能）
1.1.0 → 2.0.0（主版本：破坏性变更）
```

### 创建发布

```bash
# 创建带注释的标签
git tag -a v1.2.0 -m "Release v1.2.0

Features:
- Add user authentication
- Implement password reset

Fixes:
- Resolve login redirect issue

Breaking Changes:
- None"

# 将标签推送到远程
git push origin v1.2.0

# 列出标签
git tag -l

# 删除标签
git tag -d v1.2.0
git push origin --delete v1.2.0
```

### 生成变更日志

```bash
# 从提交生成变更日志
git log v1.1.0..v1.2.0 --oneline --no-merges

# 或使用 conventional-changelog
npx conventional-changelog -i CHANGELOG.md -s
```

## Git 配置

### 必要配置

```bash
# 用户身份
git config --global user.name "Your Name"
git config --global user.email "your@email.com"

# 默认分支名称
git config --global init.defaultBranch main

# 拉取行为（变基而非合并）
git config --global pull.rebase true

# 推送行为（仅推送当前分支）
git config --global push.default current

# 自动纠正拼写错误
git config --global help.autocorrect 1

# 更好的差异算法
git config --global diff.algorithm histogram

# 彩色输出
git config --global color.ui auto
```

### 实用别名

```bash
# 添加到 ~/.gitconfig
[alias]
    co = checkout
    br = branch
    ci = commit
    st = status
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = log --oneline --graph --all
    amend = commit --amend --no-edit
    wip = commit -m "WIP"
    undo = reset --soft HEAD~1
    contributors = shortlog -sn
```

### Gitignore 模式

```gitignore
# 依赖
node_modules/
vendor/

# 构建输出
dist/
build/
*.o
*.exe

# 环境文件
.env
.env.local
.env.*.local

# IDE
.idea/
.vscode/
*.swp
*.swo

# 系统文件
.DS_Store
Thumbs.db

# 日志
*.log
logs/

# 测试覆盖率
coverage/

# 缓存
.cache/
*.tsbuildinfo
```

## 常见工作流

### 开始一个新功能

```bash
# 1. 更新 main 分支
git checkout main
git pull origin main

# 2. 创建功能分支
git checkout -b feature/user-auth

# 3. 进行更改并提交
git add .
git commit -m "feat(auth): implement OAuth2 login"

# 4. 推送到远程
git push -u origin feature/user-auth

# 5. 在 GitHub/GitLab 上创建 Pull Request
```

### 用新更改更新 PR

```bash
# 1. 进行额外更改
git add .
git commit -m "feat(auth): add error handling"

# 2. 推送更新
git push origin feature/user-auth
```

### 将 Fork 与上游同步

```bash
# 1. 添加上游远程（一次性操作）
git remote add upstream https://github.com/original/repo.git

# 2. 拉取上游
git fetch upstream

# 3. 将 upstream/main 合并到你的 main
git checkout main
git merge upstream/main

# 4. 推送到你的 Fork
git push origin main
```

### 撤销错误

```bash
# 撤销最后一次提交（保留更改）
git reset --soft HEAD~1

# 撤销最后一次提交（丢弃更改）
git reset --hard HEAD~1

# 撤销已推送到远程的最后一次提交
git revert HEAD
git push origin main

# 撤销特定文件的更改
git checkout HEAD -- path/to/file

# 修改最后一次提交信息
git commit --amend -m "New message"

# 将遗漏的文件添加到最后一次提交
git add forgotten-file
git commit --amend --no-edit
```

## Git 钩子

### Pre-Commit 钩子

```bash
#!/bin/bash
# .git/hooks/pre-commit

# 运行 Lint
npm run lint || exit 1

# 运行测试
npm test || exit 1

# 检查密钥
if git diff --cached | grep -E '(password|api_key|secret)'; then
    echo "Possible secret detected. Commit aborted."
    exit 1
fi
```

### Pre-Push 钩子

```bash
#!/bin/bash
# .git/hooks/pre-push

# 运行完整测试套件
npm run test:all || exit 1

# 检查 console.log 语句
if git diff origin/main | grep -E 'console\.log'; then
    echo "Remove console.log statements before pushing."
    exit 1
fi
```

## 反模式

```
# 坏：直接提交到 main
git checkout main
git commit -m "fix bug"

# 好：使用功能分支和 PR

# 坏：提交密钥
git add .env  # 包含 API 密钥

# 好：添加到 .gitignore，使用环境变量

# 坏：巨型 PR（1000+ 行）
# 好：拆分为更小、更专注的 PR

# 坏："Update" 提交信息
git commit -m "update"
git commit -m "fix"

# 好：描述性信息
git commit -m "fix(auth): resolve redirect loop after login"

# 坏：重写公共历史
git push --force origin main

# 好：对公共分支使用 revert
git revert HEAD

# 坏：长生命周期功能分支（数周/数月）
# 好：保持分支短暂（数天），频繁变基

# 坏：提交生成的文件
git add dist/
git add node_modules/

# 好：添加到 .gitignore
```

## 快速参考

| 任务 | 命令 |
|------|---------|
| 创建分支 | `git checkout -b feature/name` |
| 切换分支 | `git checkout branch-name` |
| 删除分支 | `git branch -d branch-name` |
| 合并分支 | `git merge branch-name` |
| 变基分支 | `git rebase main` |
| 查看历史 | `git log --oneline --graph` |
| 查看更改 | `git diff` |
| 暂存更改 | `git add .` 或 `git add -p` |
| 提交 | `git commit -m "message"` |
| 推送 | `git push origin branch-name` |
| 拉取 | `git pull origin branch-name` |
| 暂存 | `git stash push -m "message"` |
| 撤销最后一次提交 | `git reset --soft HEAD~1` |
| 回滚提交 | `git revert HEAD` |
