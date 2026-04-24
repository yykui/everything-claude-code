---
name: opensource-pipeline
description: "开源流水线：fork、脱敏并将私有项目打包以安全公开发布。串联 3 个 Agent（forker、sanitizer、packager）。触发词：'/opensource'、'open source this'、'make this public'、'prepare for open source'。"
origin: ECC
---

# 开源流水线技能

通过 3 阶段流水线安全地将任意项目开源：**Fork**（清除密钥）→ **脱敏**（验证干净）→ **打包**（CLAUDE.md + setup.sh + README）。

## 何时激活

- 用户说"开源这个项目"或"将其公开"
- 用户希望将私有仓库准备好公开发布
- 用户需要在推送到 GitHub 前清除密钥
- 用户调用 `/opensource fork`、`/opensource verify` 或 `/opensource package`

## 命令

| 命令 | 操作 |
|------|------|
| `/opensource fork PROJECT` | 完整流水线：fork + 脱敏 + 打包 |
| `/opensource verify PROJECT` | 对已有仓库运行脱敏检查 |
| `/opensource package PROJECT` | 生成 CLAUDE.md + setup.sh + README |
| `/opensource list` | 显示所有暂存项目 |
| `/opensource status PROJECT` | 显示暂存项目的报告 |

## 协议

### /opensource fork PROJECT

**完整流水线——主要工作流。**

#### 第一步：收集参数

解析项目路径。如果 PROJECT 包含 `/`，视为路径（绝对或相对）。否则检查：当前工作目录、`$HOME/PROJECT`，然后询问用户。

```
SOURCE_PATH="<已解析的绝对路径>"
STAGING_PATH="$HOME/opensource-staging/${PROJECT_NAME}"
```

询问用户：
1. "哪个项目？"（如未找到）
2. "许可证？（MIT / Apache-2.0 / GPL-3.0 / BSD-3-Clause）"
3. "GitHub 组织或用户名？"（默认：通过 `gh api user -q .login` 检测）
4. "GitHub 仓库名？"（默认：项目名）
5. "README 的描述？"（分析项目给出建议）

#### 第二步：创建暂存目录

```bash
mkdir -p $HOME/opensource-staging/
```

#### 第三步：运行 Forker Agent

启动 `opensource-forker` Agent：

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

#### 第四步：运行 Sanitizer Agent

启动 `opensource-sanitizer` Agent：

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

**如果 FAIL：** 向用户展示发现的问题。询问："修复这些问题并重新扫描，还是中止？"
- 如果修复：应用修复，重新运行脱敏检查（最多 3 次重试——3 次 FAIL 后，展示所有问题并请用户手动修复）
- 如果中止：清理暂存目录

**如果 PASS 或 PASS WITH WARNINGS：** 继续第五步。

#### 第五步：运行 Packager Agent

启动 `opensource-packager` Agent：

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
开源 Fork 就绪：{PROJECT_NAME}

位置：{STAGING_PATH}
许可证：{license}
已生成文件：
  - CLAUDE.md
  - setup.sh（可执行）
  - README.md
  - LICENSE
  - CONTRIBUTING.md
  - .env.example（{N} 个变量）

脱敏状态：{sanitization_verdict}

后续步骤：
  1. 审查：cd {STAGING_PATH}
  2. 创建仓库：gh repo create {github_org}/{github_repo} --public
  3. 推送：git remote add origin ... && git push -u origin main

是否继续创建 GitHub 仓库？（yes/no/review first）
```

#### 第七步：发布到 GitHub（用户批准后）

```bash
cd "{STAGING_PATH}"
gh repo create "{github_org}/{github_repo}" --public --source=. --push --description "{description}"
```

---

### /opensource verify PROJECT

独立运行脱敏检查。解析路径：如果 PROJECT 包含 `/`，视为路径。否则检查 `$HOME/opensource-staging/PROJECT`，然后 `$HOME/PROJECT`，最后检查当前目录。

```
Agent(
  subagent_type="opensource-sanitizer",
  prompt="Verify sanitization of: {resolved_path}. Run all 6 scan categories and generate SANITIZATION_REPORT.md."
)
```

---

### /opensource package PROJECT

独立运行打包。询问"许可证？"和"描述？"，然后：

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

## 暂存目录结构

```
$HOME/opensource-staging/
  my-project/
    FORK_REPORT.md           # 来自 forker agent
    SANITIZATION_REPORT.md   # 来自 sanitizer agent
    CLAUDE.md                # 来自 packager agent
    setup.sh                 # 来自 packager agent
    README.md                # 来自 packager agent
    .env.example             # 来自 forker agent
    ...                      # 已脱敏的项目文件
```

## 反模式

- **绝对不要**在未获用户批准的情况下推送到 GitHub
- **绝对不要**跳过脱敏检查——它是安全门控
- **绝对不要**在脱敏检查 FAIL 且未修复所有严重问题的情况下继续
- **绝对不要**在暂存目录中留下 `.env`、`*.pem` 或 `credentials.json`

## 最佳实践

- 新版本发布始终运行完整流水线（fork → 脱敏 → 打包）
- 暂存目录在明确清理之前持续存在——可用于审查
- 手动修复后，在发布前重新运行脱敏检查
- 将密钥参数化而非删除——保留项目功能性

## 相关技能

参见 `security-review` 了解脱敏检查使用的密钥检测模式。
