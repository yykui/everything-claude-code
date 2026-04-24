---
name: canary-watch
description: 使用此技能在部署、合并或依赖升级后监控已部署 URL 的回归情况。
origin: ECC
---

# 金丝雀监控 — 部署后监控

## 何时使用

- 部署到生产或预发布环境后
- 合并高风险 PR 后
- 当你想验证修复是否真正修复了问题时
- 上线窗口期间的持续监控
- 依赖升级后

## 工作原理

监控已部署 URL 的回归情况。循环运行直到停止或监控窗口到期。

### 监控内容

```
1. HTTP Status — is the page returning 200?
2. Console Errors — new errors that weren't there before?
3. Network Failures — failed API calls, 5xx responses?
4. Performance — LCP/CLS/INP regression vs baseline?
5. Content — did key elements disappear? (h1, nav, footer, CTA)
6. API Health — are critical endpoints responding within SLA?
```

### 监控模式

**快速检查**（默认）：单次通过，报告结果
```
/canary-watch https://myapp.com
```

**持续监控**：每 N 分钟检查一次，持续 M 小时
```
/canary-watch https://myapp.com --interval 5m --duration 2h
```

**差异模式**：比较预发布与生产环境
```
/canary-watch --compare https://staging.myapp.com https://myapp.com
```

### 告警阈值

```yaml
critical:  # immediate alert
  - HTTP status != 200
  - Console error count > 5 (new errors only)
  - LCP > 4s
  - API endpoint returns 5xx

warning:   # flag in report
  - LCP increased > 500ms from baseline
  - CLS > 0.1
  - New console warnings
  - Response time > 2x baseline

info:      # log only
  - Minor performance variance
  - New network requests (third-party scripts added?)
```

### 通知

当达到严重阈值时：
- 桌面通知（macOS/Linux）
- 可选：Slack/Discord Webhook
- 记录到 `~/.claude/canary-watch.log`

## 输出

```markdown
## Canary Report — myapp.com — 2026-03-23 03:15 PST

### Status: HEALTHY ✓

| Check | Result | Baseline | Delta |
|-------|--------|----------|-------|
| HTTP | 200 ✓ | 200 | — |
| Console errors | 0 ✓ | 0 | — |
| LCP | 1.8s ✓ | 1.6s | +200ms |
| CLS | 0.01 ✓ | 0.01 | — |
| API /health | 145ms ✓ | 120ms | +25ms |

### No regressions detected. Deploy is clean.
```

## 集成

与以下配合使用：
- `/browser-qa` 用于部署前验证
- 钩子：在 `git push` 上添加为 PostToolUse 钩子，以在部署后自动检查
- CI：在部署步骤后在 GitHub Actions 中运行
