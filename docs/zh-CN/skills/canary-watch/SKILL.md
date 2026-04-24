---
name: canary-watch
description: 使用本技能在部署、合并或依赖升级后监控已部署 URL 的回归情况。
origin: ECC
---

# Canary Watch — 部署后监控

## 何时使用

- 部署到生产或预发布环境后
- 合并高风险 PR 后
- 想验证修复是否真正生效时
- 发布窗口期间的持续监控
- 依赖升级后

## 工作原理

监控已部署 URL 的回归情况。循环运行，直到手动停止或监控窗口到期。

### 监控内容

```
1. HTTP 状态 — 页面是否返回 200？
2. 控制台错误 — 是否出现之前没有的新错误？
3. 网络失败 — API 调用失败、5xx 响应？
4. 性能 — LCP/CLS/INP 相比基线是否有回归？
5. 内容 — 关键元素是否消失？（h1、导航、页脚、CTA）
6. API 健康 — 关键端点是否在 SLA 内响应？
```

### 监控模式

**快速检查**（默认）：单次扫描，报告结果
```
/canary-watch https://myapp.com
```

**持续监控**：每 N 分钟检查一次，持续 M 小时
```
/canary-watch https://myapp.com --interval 5m --duration 2h
```

**对比模式**：对比预发布与生产环境
```
/canary-watch --compare https://staging.myapp.com https://myapp.com
```

### 告警阈值

```yaml
critical:  # 立即告警
  - HTTP 状态 != 200
  - 控制台错误数 > 5（仅新增错误）
  - LCP > 4s
  - API 端点返回 5xx

warning:   # 在报告中标记
  - LCP 相比基线增加 > 500ms
  - CLS > 0.1
  - 新增控制台警告
  - 响应时间 > 基线的 2 倍

info:      # 仅记录日志
  - 轻微性能波动
  - 新增网络请求（添加了第三方脚本？）
```

### 通知

当超过关键阈值时：
- 桌面通知（macOS/Linux）
- 可选：Slack/Discord Webhook
- 记录到 `~/.claude/canary-watch.log`

## 输出

```markdown
## Canary 报告 — myapp.com — 2026-03-23 03:15 PST

### 状态：健康 ✓

| 检查项 | 结果 | 基线 | 差值 |
|-------|--------|----------|-------|
| HTTP | 200 ✓ | 200 | — |
| 控制台错误 | 0 ✓ | 0 | — |
| LCP | 1.8s ✓ | 1.6s | +200ms |
| CLS | 0.01 ✓ | 0.01 | — |
| API /health | 145ms ✓ | 120ms | +25ms |

### 未检测到回归。部署状态正常。
```

## 集成

配合使用：
- `/browser-qa` 进行部署前验证
- 钩子：将其添加为 `git push` 的 PostToolUse 钩子，在部署后自动检查
- CI：在 GitHub Actions 的部署步骤后运行
