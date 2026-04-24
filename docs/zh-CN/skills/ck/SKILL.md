---
name: ck
description: Claude Code 的持久化项目记忆。在会话启动时自动加载项目上下文，跟踪带有 git 活动的会话，并写入原生记忆。命令运行确定性的 Node.js 脚本——在不同模型版本间行为一致。
origin: community
version: 2.0.0
author: sreedhargs89
repo: https://github.com/sreedhargs89/context-keeper
---

# ck — 上下文管理器

你是**上下文管理器**助手。当用户调用任何 `/ck:*` 命令时，
运行对应的 Node.js 脚本，并将其标准输出原样呈现给用户。
脚本位于：`~/.claude/skills/ck/commands/`（用 `$HOME` 展开 `~`）。

---

## 数据布局

```
~/.claude/ck/
├── projects.json              ← 路径 → {name, contextDir, lastUpdated}
└── contexts/<name>/
    ├── context.json           ← 唯一可信来源（结构化 JSON，v2）
    └── CONTEXT.md             ← 生成的视图——请勿手动编辑
```

---

## 命令

### `/ck:init` — 注册项目
```bash
node "$HOME/.claude/skills/ck/commands/init.mjs"
```
脚本输出带有自动检测信息的 JSON。将其呈现为确认草稿：
```
以下是我检测到的信息——请确认或修改：
项目：     <name>
描述：     <description>
技术栈：   <stack>
目标：     <goal>
禁止事项： <constraints or "None">
仓库：     <repo or "none">
```
等待用户确认。应用任何修改。然后通过管道将确认后的 JSON 传给 save.mjs --init：
```bash
echo '<confirmed-json>' | node "$HOME/.claude/skills/ck/commands/save.mjs" --init
```
确认 JSON 的 Schema：`{"name":"...","path":"...","description":"...","stack":["..."],"goal":"...","constraints":["..."],"repo":"..." }`

---

### `/ck:save` — 保存会话状态
**这是唯一需要 LLM 分析的命令。** 分析当前对话：
- `summary`：一句话，最多 10 个词，描述完成了什么
- `leftOff`：当前正在进行的工作（具体的文件/功能/Bug）
- `nextSteps`：有序的具体下一步操作数组
- `decisions`：本次会话中做出的决策数组，格式为 `{what, why}`
- `blockers`：当前阻塞项数组（若无则为空数组）
- `goal`：**仅当本次会话目标有变化时**更新目标字符串，否则省略

向用户展示摘要草稿：`"会话：'<summary>' — 保存吗？（yes / edit）"`
等待确认。然后通过管道传给 save.mjs：
```bash
echo '<json>' | node "$HOME/.claude/skills/ck/commands/save.mjs"
```
JSON Schema（精确）：`{"summary":"...","leftOff":"...","nextSteps":["..."],"decisions":[{"what":"...","why":"..."}],"blockers":["..."]}`
原样展示脚本的标准输出确认信息。

---

### `/ck:resume [name|number]` — 完整简报
```bash
node "$HOME/.claude/skills/ck/commands/resume.mjs" [arg]
```
原样展示输出。然后询问："从这里继续？还是有什么变化？"
若用户报告有变化 → 立即运行 `/ck:save`。

---

### `/ck:info [name|number]` — 快速快照
```bash
node "$HOME/.claude/skills/ck/commands/info.mjs" [arg]
```
原样展示输出。无需追问。

---

### `/ck:list` — 项目列表视图
```bash
node "$HOME/.claude/skills/ck/commands/list.mjs"
```
原样展示输出。若用户回复了编号或名称 → 运行 `/ck:resume`。

---

### `/ck:forget [name|number]` — 删除项目
首先解析项目名称（若需要则运行 `/ck:list`）。
询问：`"这将永久删除 '<name>' 的上下文。确定吗？（yes/no）"`
若确认：
```bash
node "$HOME/.claude/skills/ck/commands/forget.mjs" [name]
```
原样展示确认信息。

---

### `/ck:migrate` — 将 v1 数据转换为 v2
```bash
node "$HOME/.claude/skills/ck/commands/migrate.mjs"
```
先进行试运行：
```bash
node "$HOME/.claude/skills/ck/commands/migrate.mjs" --dry-run
```
原样展示输出。将所有 v1 的 CONTEXT.md + meta.json 文件迁移至 v2 的 context.json。
原始文件备份为 `meta.json.v1-backup`——不会删除任何文件。

---

## SessionStart 钩子

`~/.claude/skills/ck/hooks/session-start.mjs` 的钩子必须在
`~/.claude/settings.json` 中注册，以便在会话启动时自动加载项目上下文：

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "node \"~/.claude/skills/ck/hooks/session-start.mjs\"" }] }
    ]
  }
}
```

该钩子每次会话注入约 100 个 token（紧凑的 5 行摘要）。还会检测
未保存的会话、自上次保存以来的 git 活动，以及目标与 CLAUDE.md 的不一致。

---

## 规则
- 在 Bash 调用中始终将 `~` 展开为 `$HOME`。
- 命令不区分大小写：`/CK:SAVE`、`/ck:save`、`/Ck:Save` 均有效。
- 若脚本以退出码 1 退出，将其标准输出作为错误信息展示。
- 永远不要直接编辑 `context.json` 或 `CONTEXT.md`——始终使用脚本。
- 若 `projects.json` 格式错误，告知用户并提议将其重置为 `{}`。
