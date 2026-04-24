---
name: opensource-pipeline
description: "开源流水线：对私有项目进行分叉、净化和打包，以安全方式发布为公开项目。串联 3 个子 Agent（forker、sanitizer、packager）。触发词：'/opensource'、'open source this'、'make this public'、'prepare for open source'。"
origin: ECC
---

# 开源流水线技能

通过三阶段流水线安全地开源任何项目：**分叉**（剔除密钥）→ **净化**（验证干净）→ **打包**（CLAUDE.md + setup.sh + README）。

## 何时激活

- 用户说"开源这个项目"或"将其公开"
- 用户想为私有仓库准备公开发布
- 用户需要在推送到 GitHub 前剔除密钥
- 用户调用 `/opensource fork`、`/opensource verify` 或 `/opensource package`

## 命令

| 命令 | 操作 |
|------|------|
| `/opensource fork PROJECT` | 完整流水线：分叉 + 净化 + 打包 |
| `/opensource verify PROJECT` | 对现有仓库运行净化检查 |
| `/opensource package PROJECT` | 生成 CLAUDE.md + setup.sh + README |
| `/opensource list` | 显示所有暂存项目 |
| `/opensource status PROJECT` | 显示暂存项目的报告 |

## 协议

### /opensource fork PROJECT

**完整流水线——主要工作流。**

#### 第一步：收集参数

解析项目路径。如果 PROJECT 包含 `/`，视为路径（绝对路径或相对路径）。否则检查：当前工作目录、`$HOME/PROJECT`，然后询问用户。

```
SOURCE_PATH="<解析后的绝对路径>"
STAGING_PATH="$HOME/opensource-staging/${PROJECT_NAME}"
```

询问用户：
1. "哪个项目？"（如果未找到）
2. "许可证？（MIT / Apache-2.0 / GPL-3.0 / BSD-3-Clause）"
3. "GitHub 组织或用户名？"（默认：通过 `gh api user -q .login` 检测）
4. "GitHub 仓库名？"（默认：项目名）
5. "README 的描述？"（分析项目后给出建议）

#### 第二步：创建暂存目录

```bash
mkdir -p $HOME/opensource-staging/
```

#### 第三步：运行分叉子 Agent

生成 `opensource-forker` 子 Agent：

```
Agent(
  description="Fork {PROJECT} for open-source",
  subagent_type="opensource-forker",
  prompt="""
Fork project for open-source release.

Source: {SOURCE_PATH}
Target: {STAGING_PATH}
License: {chosen_license}

Follow the full forking protocol:
1. Copy files (exclude .git, node_modules, __pycache__, .venv)
2. Strip all secrets and credentials
3. Replace internal references with placeholders
4. Generate .env.example
5. Clean git history
6. Generate FORK_REPORT.md in {STAGING_PATH}/FORK_REPORT.md
"""
)
```

等待完成。读取 `{STAGING_PATH}/FORK_REPORT.md`。

#### 第四步：运行净化子 Agent

生成 `opensource-sanitizer` 子 Agent：

```
Agent(
  description="Verify {PROJECT} sanitization",
  subagent_type="opensource-sanitizer",
  prompt="""
Verify sanitization of open-source fork.

Project: {STAGING_PATH}
Source (for reference): {SOURCE_PATH}

Run ALL scan categories:
1. Secrets scan (CRITICAL)
2. PII scan (CRITICAL)
3. Internal references scan (CRITICAL)
4. Dangerous files check (CRITICAL)
5. Configuration completeness (WARNING)
6. Git history audit

Generate SANITIZATION_REPORT.md inside {STAGING_PATH}/ with PASS/FAIL verdict.
"""
)
```

等待完成。读取 `{STAGING_PATH}/SANITIZATION_REPORT.md`。

**如果失败：** 向用户展示发现的问题。询问："修复这些问题并重新扫描，还是终止？"
- 如果修复：应用修复，重新运行净化器（最多 3 次重试——3 次失败后，展示所有发现并要求用户手动修复）
- 如果终止：清理暂存目录

**如果通过或带警告通过：** 继续到第五步。

#### 第五步：运行打包子 Agent

生成 `opensource-packager` 子 Agent：

```
Agent(
  description="Package {PROJECT} for open-source",
  subagent_type="opensource-packager",
  prompt="""
Generate open-source packaging for project.

Project: {STAGING_PATH}
License: {chosen_license}
Project name: {PROJECT_NAME}
Description: {description}
GitHub repo: {github_repo}

Generate:
1. CLAUDE.md (commands, architecture, key files)
2. setup.sh (one-command bootstrap, make executable)
3. README.md (or enhance existing)
4. LICENSE
5. CONTRIBUTING.md
6. .github/ISSUE_TEMPLATE/ (bug_report.md, feature_request.md)
"""
)
```

#### 第六步：最终审查

向用户展示：
```
开源分叉已就绪：{PROJECT_NAME}

位置：{STAGING_PATH}
许可证：{license}
已生成的文件：
  - CLAUDE.md
  - setup.sh（可执行）
  - README.md
  - LICENSE
  - CONTRIBUTING.md
  - .env.example（{N} 个变量）

净化结果：{sanitization_verdict}

后续步骤：
  1. 审查：cd {STAGING_PATH}
  2. 创建仓库：gh repo create {github_org}/{github_repo} --public
  3. 推送：git remote add origin ... && git push -u origin main

继续创建 GitHub 仓库？（yes/no/先审查）
```

#### 第七步：GitHub 发布（用户批准后）

```bash
cd "{STAGING_PATH}"
gh repo create "{github_org}/{github_repo}" --public --source=. --push --description "{description}"
```

---

### /opensource verify PROJECT

独立运行净化器。解析路径：如果 PROJECT 包含 `/`，视为路径。否则检查 `$HOME/opensource-staging/PROJECT`，然后 `$HOME/PROJECT`，再检查当前目录。

```
Agent(
  subagent_type="opensource-sanitizer",
  prompt="Verify sanitization of: {resolved_path}. Run all 6 scan categories and generate SANITIZATION_REPORT.md."
)
```

---

### /opensource package PROJECT

独立运行打包器。询问"许可证？"和"描述？"，然后：

```
Agent(
  subagent_type="opensource-packager",
  prompt="Package: {resolved_path} ..."
)
```

---

### /opensource list

```bash
ls -d $HOME/opensource-staging/*/
```

显示每个项目及其流水线进度（FORK_REPORT.md、SANITIZATION_REPORT.md、CLAUDE.md 是否存在）。

---

### /opensource status PROJECT

```bash
cat $HOME/opensource-staging/${PROJECT}/SANITIZATION_REPORT.md
cat $HOME/opensource-staging/${PROJECT}/FORK_REPORT.md
```

## 暂存目录布局

```
$HOME/opensource-staging/
  my-project/
    FORK_REPORT.md           # 来自分叉子 Agent
    SANITIZATION_REPORT.md   # 来自净化子 Agent
    CLAUDE.md                # 来自打包子 Agent
    setup.sh                 # 来自打包子 Agent
    README.md                # 来自打包子 Agent
    .env.example             # 来自分叉子 Agent
    ...                      # 净化后的项目文件
```

## 反模式

- **绝不**在未获得用户批准的情况下推送到 GitHub
- **绝不**跳过净化器——它是安全关卡
- **绝不**在净化器失败且未修复所有关键发现的情况下继续
- **绝不**在暂存目录中保留 `.env`、`*.pem` 或 `credentials.json`

## 最佳实践

- 对新版本始终运行完整流水线（分叉 → 净化 → 打包）
- 暂存目录会持续存在直到被明确清理——用它进行审查
- 在任何手动修复后、发布前重新运行净化器
- 参数化密钥而不是删除它们——保留项目功能

## 相关技能

查看 `security-review` 了解净化器使用的密钥检测模式。
