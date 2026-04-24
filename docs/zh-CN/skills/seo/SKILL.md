---
name: seo
description: 对技术 SEO、页面优化、结构化数据、核心网页指标和内容策略进行审计、规划和实施改进。适用于用户希望提升搜索可见性、进行 SEO 修复、添加 schema 标记、处理站点地图/robots 工作或进行关键词映射的场景。
origin: ECC
---

# SEO

通过技术正确性、性能和内容相关性提升搜索可见性，而非依赖技巧。

## 何时使用

在以下情况下使用本技能：
- 审计抓取性、可索引性、规范标签或重定向
- 改善标题标签、元描述和标题结构
- 添加或验证结构化数据
- 改善核心网页指标
- 进行关键词研究并将关键词映射到 URL
- 规划内部链接或站点地图/robots 变更

## 工作原理

### 原则

1. 在内容优化之前修复技术障碍。
2. 一个页面应有一个明确的主要搜索意图。
3. 优先考虑长期质量信号，而非操纵性模式。
4. 移动优先假设很重要，因为索引是移动优先的。
5. 建议应针对具体页面且可执行。

### 技术 SEO 检查清单

#### 抓取性

- `robots.txt` 应允许重要页面并屏蔽低价值界面
- 重要页面不应意外设置 `noindex`
- 重要页面应在较浅的点击深度内可达
- 避免超过两次跳转的重定向链
- 规范标签应自洽且不形成循环

#### 可索引性

- 首选 URL 格式应保持一致
- 多语言页面若使用则需要正确的 hreflang
- 站点地图应反映预期的公开界面
- 没有重复 URL 应在无规范控制的情况下竞争

#### 性能

- LCP < 2.5s
- INP < 200ms
- CLS < 0.1
- 常见修复：预加载首屏资源、减少渲染阻塞工作、预留布局空间、精简重量级 JS

#### 结构化数据

- 首页：在适当时使用组织或企业 schema
- 编辑页面：`Article` / `BlogPosting`
- 产品页面：`Product` 和 `Offer`
- 内部页面：`BreadcrumbList`
- 问答部分：仅在内容真正匹配时使用 `FAQPage`

### 页面规则

#### 标题标签

- 目标约 50-60 个字符
- 将主要关键词或概念放在靠前位置
- 标题应对人类可读，而非为爬虫堆砌

#### 元描述

- 目标约 120-160 个字符
- 如实描述页面内容
- 自然包含主要主题

#### 标题结构

- 一个清晰的 `H1`
- `H2` 和 `H3` 应反映实际内容层级
- 不要仅为视觉样式而跳过结构

### 关键词映射

1. 定义搜索意图
2. 收集现实的关键词变体
3. 按意图匹配度、潜在价值和竞争程度排序
4. 将一个主要关键词/主题映射到一个 URL
5. 检测并避免关键词蚕食

### 内部链接

- 从权重强的页面链接到你想排名的页面
- 使用描述性锚文本
- 在可以使用更具体锚文本时避免通用锚文本
- 从新页面回链到相关的现有页面

## 示例

### 标题公式

```text
Primary Topic - Specific Modifier | Brand
```

### 元描述公式

```text
Action + topic + value proposition + one supporting detail
```

### JSON-LD 示例

```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "Page Title Here",
  "author": {
    "@type": "Person",
    "name": "Author Name"
  },
  "publisher": {
    "@type": "Organization",
    "name": "Brand Name"
  }
}
```

### 审计输出格式

```text
[HIGH] Duplicate title tags on product pages
Location: src/routes/products/[slug].tsx
Issue: Dynamic titles collapse to the same default string, which weakens relevance and creates duplicate signals.
Fix: Generate a unique title per product using the product name and primary category.
```

## 反模式

| 反模式 | 修复方法 |
| --- | --- |
| 关键词堆砌 | 以用户为先写作 |
| 内容薄弱的近重复页面 | 合并或使其差异化 |
| 与实际内容不符的 schema | 让 schema 与实际内容相匹配 |
| 不查看实际页面就给出内容建议 | 先阅读真实页面 |
| 通用"改善 SEO"输出 | 将每条建议与具体页面或资产关联 |

## 相关技能

- `seo-specialist`
- `frontend-patterns`
- `brand-voice`
- `market-research`
