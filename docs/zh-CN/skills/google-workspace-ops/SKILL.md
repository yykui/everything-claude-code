---
name: google-workspace-ops
description: 将 Google Drive、Docs、Sheets 和 Slides 作为统一工作流界面，用于计划文档、跟踪表、演示文稿和共享文档的操作。当用户需要查找、总结、编辑、迁移或清理 Google Workspace 资产，而无需手动执行底层工具调用时使用。
origin: ECC
---

# Google Workspace 运营

本技能用于将共享文档、电子表格和演示文稿作为工作系统进行操作，而非孤立地编辑单个文件。

## 使用时机

- 用户需要查找文档、表格或演示文稿并就地更新
- 整合存储在 Google Drive 中的计划、跟踪表、备注或客户列表
- 清理或重构共享电子表格
- 导入、修复或重新格式化 Google Slides 演示文稿
- 从 Docs、Sheets 或 Slides 中提取摘要以支持决策

## 首选工具界面

以 Google Drive 为入口，再切换到合适的专项工具：

- Google Docs 用于文字密集型文档
- Google Sheets 用于表格数据、公式和图表
- Google Slides 用于演示文稿、导入、模板迁移和清理

不要仅凭文件名猜测结构，务必先检查。

## 工作流程

### 1. 查找资产

从 Drive 搜索界面入手，定位：

- 目标文件
- 同级资产
- 可能的重复文件
- 最近修改的版本

如果多个文档看起来相似，通过标题、所有者、修改时间或文件夹加以确认。

### 2. 先检查后编辑

在进行任何更改之前：

- 总结当前结构
- 识别标签页、标题或幻灯片数量
- 判断任务是局部清理还是结构性重构

选择能安全完成工作的最小工具。

### 3. 精准编辑

- 对于 Docs：使用基于索引的精确编辑，而非模糊的整体改写
- 对于 Sheets：操作明确的标签页和范围
- 对于 Slides：区分内容编辑与视觉清理或模板迁移

如果请求的工作对视觉或布局敏感，请通过检查和验证的迭代方式进行，而非一次性盲目更新。

### 4. 保持工作系统整洁

当文件是更大工作流的一部分时，同时关注：

- 重复的跟踪表
- 过期的演示文稿
- 陈旧文档与权威文档的对比
- 资产是否应归档、合并或重命名

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

- "找到当前的规划文档并精简它"
- "清理这个客户电子表格，给我看流失风险行"
- "将这个演示文稿导入 Slides 并使其看起来专业"
- "找到当前的跟踪表，不是那个过期的重复版本"
