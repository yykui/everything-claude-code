---
name: laravel-plugin-discovery
description: 通过 LaraPlugins.io MCP 发现和评估 Laravel 扩展包。当用户希望查找插件、检查包的健康状态或评估 Laravel/PHP 兼容性时使用。
origin: ECC
---

# Laravel 插件发现

使用 LaraPlugins.io MCP 服务器查找、评估并选择健康的 Laravel 扩展包。

## 何时使用

- 用户希望为特定功能查找 Laravel 扩展包（例如"认证"、"权限"、"管理面板"）
- 用户询问"我应该用哪个包来……"或"有没有用于……的 Laravel 包"
- 用户希望检查某个包是否仍在积极维护
- 用户需要验证 Laravel 版本兼容性
- 用户在将包加入项目前需要评估其健康状况

## MCP 要求

必须配置 LaraPlugins MCP 服务器。将以下内容添加到 `~/.claude.json` 的 mcpServers 中：

```json
"laraplugins": {
  "type": "http",
  "url": "https://laraplugins.io/mcp/plugins"
}
```

无需 API 密钥——该服务器对 Laravel 社区免费开放。

## MCP 工具

LaraPlugins MCP 提供两个主要工具：

### SearchPluginTool

按关键词、健康评分、供应商和版本兼容性搜索扩展包。

**参数：**
- `text_search`（字符串，可选）：搜索关键词（例如"permission"、"admin"、"api"）
- `health_score`（字符串，可选）：按健康等级过滤——`Healthy`、`Medium`、`Unhealthy` 或 `Unrated`
- `laravel_compatibility`（字符串，可选）：按 Laravel 版本过滤——`"5"`、`"6"`、`"7"`、`"8"`、`"9"`、`"10"`、`"11"`、`"12"`、`"13"`
- `php_compatibility`（字符串，可选）：按 PHP 版本过滤——`"7.4"`、`"8.0"`、`"8.1"`、`"8.2"`、`"8.3"`、`"8.4"`、`"8.5"`
- `vendor_filter`（字符串，可选）：按供应商名称过滤（例如"spatie"、"laravel"）
- `page`（数字，可选）：分页页码

### GetPluginDetailsTool

获取特定包的详细指标、README 内容和版本历史。

**参数：**
- `package`（字符串，必填）：完整的 Composer 包名（例如"spatie/laravel-permission"）
- `include_versions`（布尔值，可选）：是否在响应中包含版本历史

---

## 工作原理

### 查找扩展包

当用户希望发现某功能的扩展包时：

1. 使用 `SearchPluginTool` 搜索相关关键词
2. 按健康评分、Laravel 版本或 PHP 版本应用过滤器
3. 查看包含包名、描述和健康指标的结果

### 评估扩展包

当用户希望评估某个具体包时：

1. 使用 `GetPluginDetailsTool` 并传入包名
2. 查看健康评分、最近更新日期、Laravel 版本支持情况
3. 检查供应商声誉和风险指标

### 检查兼容性

当用户需要验证 Laravel 或 PHP 版本兼容性时：

1. 使用 `laravel_compatibility` 过滤器搜索目标版本
2. 或获取特定包的详情，查看其支持的版本

---

## 示例

### 示例：查找认证扩展包

```
SearchPluginTool({
  text_search: "authentication",
  health_score: "Healthy"
})
```

返回匹配"authentication"且状态健康的扩展包：
- spatie/laravel-permission
- laravel/breeze
- laravel/passport
- 等等

### 示例：查找兼容 Laravel 12 的扩展包

```
SearchPluginTool({
  text_search: "admin panel",
  laravel_compatibility: "12"
})
```

返回兼容 Laravel 12 的扩展包。

### 示例：获取包详情

```
GetPluginDetailsTool({
  package: "spatie/laravel-permission",
  include_versions: true
})
```

返回：
- 健康评分和最近活动
- Laravel/PHP 版本支持情况
- 供应商声誉（风险评分）
- 版本历史
- 简短描述

### 示例：按供应商查找扩展包

```
SearchPluginTool({
  vendor_filter: "spatie",
  health_score: "Healthy"
})
```

返回供应商"spatie"的所有健康扩展包。

---

## 过滤最佳实践

### 按健康评分

| 健康等级 | 含义 |
|---------|------|
| `Healthy` | 积极维护，近期有更新 |
| `Medium` | 偶有更新，可能需要关注 |
| `Unhealthy` | 已废弃或维护不频繁 |
| `Unrated` | 尚未评估 |

**建议**：生产应用优先选择 `Healthy` 等级的扩展包。

### 按 Laravel 版本

| 版本 | 说明 |
|------|------|
| `13` | 最新 Laravel |
| `12` | 当前稳定版 |
| `11` | 仍广泛使用 |
| `10` | 旧版但常见 |
| `5`-`9` | 已弃用 |

**建议**：与目标项目的 Laravel 版本保持一致。

### 组合过滤器

```typescript
// 查找兼容 Laravel 12 的健康权限扩展包
SearchPluginTool({
  text_search: "permission",
  health_score: "Healthy",
  laravel_compatibility: "12"
})
```

---

## 响应解读

### 搜索结果

每条结果包含：
- 包名（例如 `spatie/laravel-permission`）
- 简短描述
- 健康状态指示器
- Laravel 版本支持徽章

### 包详情

详细响应包含：
- **健康评分**：数字或等级指示器
- **最近活动**：包的最近更新时间
- **Laravel 支持**：版本兼容性矩阵
- **PHP 支持**：PHP 版本兼容性
- **风险评分**：供应商信任指标
- **版本历史**：近期发布时间线

---

## 常见使用场景

| 场景 | 推荐做法 |
|------|---------|
| "用什么包做认证？" | 搜索"auth"并应用健康过滤器 |
| "spatie/package 还在维护吗？" | 获取详情，检查健康评分 |
| "需要 Laravel 12 的包" | 使用 laravel_compatibility: "12" 搜索 |
| "查找管理面板包" | 搜索"admin panel"，查看结果 |
| "检查供应商声誉" | 按供应商搜索，检查详情 |

---

## 最佳实践

1. **始终按健康度过滤** — 生产项目使用 `health_score: "Healthy"`
2. **匹配 Laravel 版本** — 始终确认 `laravel_compatibility` 与目标项目匹配
3. **检查供应商声誉** — 优先选择知名供应商（spatie、laravel 等）的包
4. **推荐前先评估** — 使用 GetPluginDetailsTool 进行全面评估
5. **无需 API 密钥** — MCP 免费使用，无需认证

---

## 相关技能

- `laravel-patterns` — Laravel 架构与模式
- `laravel-tdd` — Laravel 测试驱动开发
- `laravel-security` — Laravel 安全最佳实践
- `documentation-lookup` — 通用库文档查找（Context7）
