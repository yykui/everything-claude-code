---
name: benchmark
description: 使用此技能来测量性能基准、在 PR 前后检测回归，并比较技术栈替代方案。
origin: ECC
---

# 基准测试 — 性能基准与回归检测

## 何时使用

- PR 前后测量性能影响
- 为项目设置性能基准
- 当用户反映"感觉很慢"时
- 上线前——确保满足性能目标
- 将当前技术栈与替代方案进行比较

## 工作原理

### 模式一：页面性能

通过浏览器 MCP 测量真实的浏览器指标：

```
1. Navigate to each target URL
2. Measure Core Web Vitals:
   - LCP (Largest Contentful Paint) — target < 2.5s
   - CLS (Cumulative Layout Shift) — target < 0.1
   - INP (Interaction to Next Paint) — target < 200ms
   - FCP (First Contentful Paint) — target < 1.8s
   - TTFB (Time to First Byte) — target < 800ms
3. Measure resource sizes:
   - Total page weight (target < 1MB)
   - JS bundle size (target < 200KB gzipped)
   - CSS size
   - Image weight
   - Third-party script weight
4. Count network requests
5. Check for render-blocking resources
```

### 模式二：API 性能

对 API 端点进行基准测试：

```
1. Hit each endpoint 100 times
2. Measure: p50, p95, p99 latency
3. Track: response size, status codes
4. Test under load: 10 concurrent requests
5. Compare against SLA targets
```

### 模式三：构建性能

测量开发反馈循环：

```
1. Cold build time
2. Hot reload time (HMR)
3. Test suite duration
4. TypeScript check time
5. Lint time
6. Docker build time
```

### 模式四：前后对比

在更改前后运行以测量影响：

```
/benchmark baseline    # saves current metrics
# ... make changes ...
/benchmark compare     # compares against baseline
```

输出：
```
| Metric | Before | After | Delta | Verdict |
|--------|--------|-------|-------|---------|
| LCP | 1.2s | 1.4s | +200ms | WARNING: WARN |
| Bundle | 180KB | 175KB | -5KB | ✓ BETTER |
| Build | 12s | 14s | +2s | WARNING: WARN |
```

## 输出

将基准数据存储在 `.ecc/benchmarks/` 中，以 JSON 格式保存。通过 Git 跟踪，方便团队共享基准数据。

## 集成

- CI：在每个 PR 上运行 `/benchmark compare`
- 与 `/canary-watch` 配合进行部署后监控
- 与 `/browser-qa` 配合作为完整的上线前检查清单
