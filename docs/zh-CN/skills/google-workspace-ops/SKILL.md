---
name: google-workspace-ops
description: 将 Google Drive、Docs、Sheets 和 Slides 作为统一工作流平面来操作，用于计划、追踪表格、演示文稿和共享文档。当用户需要查找、汇总、编辑、迁移或整理 Google Workspace 资产，而无需直接使用原始工具调用时使用。
origin: ECC
---

# Google Workspace 操作

此技能用于将共享文档、电子表格和演示文稿作为工作系统进行操作，而不只是孤立地编辑某个文件。

## 何时使用

- 用户需要查找文档、表格或演示文稿并就地更新
- 整合存储在 Google Drive 中的计划、追踪表格、笔记或客户列表
- 清理或重构共享电子表格
- 导入、修复或重新格式化 Google Slides 演示文稿
- 从 Docs、Sheets 或 Slides 生成摘要以支持决策

## 首选工具界面

以 Google Drive 作为入口，然后切换到合适的专项工具：

- Google Docs 用于文字密集型文档
- Google Sheets 用于表格工作、公式和图表
- Google Slides 用于演示文稿、导入、模板迁移和整理

不要仅凭文件名猜测结构，先进行检查。

## 工作流

### 1. 查找资产

从 Drive 搜索界面开始，定位：

- 确切的文件
- 同级资产
- 可能的重复文件
- 最近修改的版本

如果多个文档看起来相似，通过标题、所有者、修改时间或文件夹来确认。

### 2. 编辑前先检查

进行更改之前：

- 汇总当前结构
- 识别标签页、标题或幻灯片数量
- 判断任务是局部清理还是结构性重构

选择能够安全完成工作的最小化工具。

### 3. 精准编辑

- 对于 Docs：使用基于索引的编辑，而非模糊的重写
- 对于 Sheets：在明确的标签页和范围上操作
- 对于 Slides：区分内容编辑与视觉整理或模板迁移

如果请求的工作对视觉或布局敏感，应通过反复检查和验证来迭代，而非一次性大规模盲目更新。

### 4. 保持工作系统整洁

当文件是更大工作流的一部分时，还需要关注：

- 重复的追踪表格
- 过时的演示文稿
- 过时文档与规范文档的对比
- 资产是否应该归档、合并或重命名

## 输出格式

使用：

```text
ASSET
- file name
- type
- why this is the right file

CURRENT STATE
- structure summary
- key problems or blockers

ACTION
- edits made or recommended

FOLLOW-UPS
- archive / merge / duplicate cleanup / next file to update
```

## 典型使用场景

- "找到当前的计划文档并进行精简"
- "清理这份客户电子表格，并显示流失风险行"
- "将此演示文稿导入 Slides 并使其看起来专业"
- "找到当前的追踪表格，而不是过时的副本"
