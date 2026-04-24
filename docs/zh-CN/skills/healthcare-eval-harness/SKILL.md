---
name: healthcare-eval-harness
description: 医疗应用部署的患者安全评估框架。针对 CDSS 准确性、PHI 暴露、临床工作流完整性和集成合规性的自动化测试套件。在安全测试失败时阻断部署。
origin: Health1 Super Speciality Hospitals — contributed by Dr. Keyur Patel
version: "1.0.0"
---

# 医疗评估框架——患者安全验证

医疗应用部署的自动化验证系统。任何一个 CRITICAL 失败即阻断部署。患者安全不可妥协。

> **注意：** 示例使用 Jest 作为参考测试运行器。请根据您的框架（Vitest、pytest、PHPUnit 等）适配命令——测试类别和通过阈值与框架无关。

## 使用时机

- 在任何 EMR/EHR 应用部署之前
- 修改 CDSS 逻辑后（药物相互作用、剂量验证、评分）
- 更改涉及患者数据的数据库 schema 后
- 修改身份验证或访问控制后
- 为医疗应用配置 CI/CD 流水线时
- 解决临床模块中的合并冲突后

## 工作原理

评估框架按顺序运行五个测试类别。前三类（CDSS 准确性、PHI 暴露、数据完整性）是需要 100% 通过率的 CRITICAL 门禁——任何一个失败均阻断部署。其余两类（临床工作流、集成合规）是需要 95% 以上通过率的 HIGH 门禁。

每个类别映射到一个 Jest 测试路径模式。CI 流水线对 CRITICAL 门禁使用 `--bail`（首次失败即停止），并通过 `--coverage --coverageThreshold` 强制执行覆盖率阈值。

### 评估类别

**1. CDSS 准确性（CRITICAL——要求 100%）**

测试所有临床决策支持逻辑：药物相互作用对（双向）、剂量验证规则、与已发布规范一致的临床评分、无假阴性、无静默失败。

```bash
npx jest --testPathPattern='tests/cdss' --bail --ci --coverage
```

**2. PHI 暴露（CRITICAL——要求 100%）**

测试受保护健康信息泄露：API 错误响应、控制台输出、URL 参数、浏览器存储、跨机构隔离、未认证访问、服务角色密钥缺失。

```bash
npx jest --testPathPattern='tests/security/phi' --bail --ci
```

**3. 数据完整性（CRITICAL——要求 100%）**

测试临床数据安全：锁定就诊记录、审计追踪条目、级联删除保护、并发编辑处理、无孤立记录。

```bash
npx jest --testPathPattern='tests/data-integrity' --bail --ci
```

**4. 临床工作流（HIGH——要求 95% 以上）**

测试端到端流程：就诊生命周期、模板渲染、用药集合、药物/诊断搜索、处方 PDF、红旗告警。

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$tmp_json" || true
total=$(jq '.numTotalTests // 0' "$tmp_json")
passed=$(jq '.numPassedTests // 0' "$tmp_json")
if [ "$total" -eq 0 ]; then
  echo "No clinical tests found" >&2
  exit 1
fi
rate=$(echo "scale=2; $passed * 100 / $total" | bc)
echo "Clinical pass rate: ${rate}% ($passed/$total)"
```

**5. 集成合规（HIGH——要求 95% 以上）**

测试外部系统：HL7 消息解析（v2.x）、FHIR 验证、化验结果映射、异常消息处理。

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/integration' --ci --json --outputFile="$tmp_json" || true
total=$(jq '.numTotalTests // 0' "$tmp_json")
passed=$(jq '.numPassedTests // 0' "$tmp_json")
if [ "$total" -eq 0 ]; then
  echo "No integration tests found" >&2
  exit 1
fi
rate=$(echo "scale=2; $passed * 100 / $total" | bc)
echo "Integration pass rate: ${rate}% ($passed/$total)"
```

### 通过/失败矩阵

| 类别 | 阈值 | 失败时 |
|------|------|--------|
| CDSS 准确性 | 100% | **阻断部署** |
| PHI 暴露 | 100% | **阻断部署** |
| 数据完整性 | 100% | **阻断部署** |
| 临床工作流 | 95% 以上 | 警告，经审查后允许 |
| 集成合规 | 95% 以上 | 警告，经审查后允许 |

### CI/CD 集成

```yaml
name: Healthcare Safety Gate
on: [push, pull_request]

jobs:
  safety-gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci

      # CRITICAL gates — 100% required, bail on first failure
      - name: CDSS Accuracy
        run: npx jest --testPathPattern='tests/cdss' --bail --ci --coverage --coverageThreshold='{"global":{"branches":80,"functions":80,"lines":80}}'

      - name: PHI Exposure Check
        run: npx jest --testPathPattern='tests/security/phi' --bail --ci

      - name: Data Integrity
        run: npx jest --testPathPattern='tests/data-integrity' --bail --ci

      # HIGH gates — 95%+ required, custom threshold check
      # HIGH gates — 95%+ required
      - name: Clinical Workflows
        run: |
          TMP_JSON=$(mktemp)
          npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$TMP_JSON" || true
          TOTAL=$(jq '.numTotalTests // 0' "$TMP_JSON")
          PASSED=$(jq '.numPassedTests // 0' "$TMP_JSON")
          if [ "$TOTAL" -eq 0 ]; then
            echo "::error::No clinical tests found"; exit 1
          fi
          RATE=$(echo "scale=2; $PASSED * 100 / $TOTAL" | bc)
          echo "Pass rate: ${RATE}% ($PASSED/$TOTAL)"
          if (( $(echo "$RATE < 95" | bc -l) )); then
            echo "::warning::Clinical pass rate ${RATE}% below 95%"
          fi

      - name: Integration Compliance
        run: |
          TMP_JSON=$(mktemp)
          npx jest --testPathPattern='tests/integration' --ci --json --outputFile="$TMP_JSON" || true
          TOTAL=$(jq '.numTotalTests // 0' "$TMP_JSON")
          PASSED=$(jq '.numPassedTests // 0' "$TMP_JSON")
          if [ "$TOTAL" -eq 0 ]; then
            echo "::error::No integration tests found"; exit 1
          fi
          RATE=$(echo "scale=2; $PASSED * 100 / $TOTAL" | bc)
          echo "Pass rate: ${RATE}% ($PASSED/$TOTAL)"
          if (( $(echo "$RATE < 95" | bc -l) )); then
            echo "::warning::Integration pass rate ${RATE}% below 95%"
          fi
```

### 反模式

- 以"上次已通过"为由跳过 CDSS 测试
- 将 CRITICAL 阈值设置低于 100%
- 对 CRITICAL 测试套件使用 `--no-bail`
- 在集成测试中模拟 CDSS 引擎（必须测试真实逻辑）
- 在安全门禁为红色时允许部署
- 对 CDSS 套件运行测试时不加 `--coverage`

## 示例

### 示例 1：本地运行所有 CRITICAL 门禁

```bash
npx jest --testPathPattern='tests/cdss' --bail --ci --coverage && \
npx jest --testPathPattern='tests/security/phi' --bail --ci && \
npx jest --testPathPattern='tests/data-integrity' --bail --ci
```

### 示例 2：检查 HIGH 门禁通过率

```bash
tmp_json=$(mktemp)
npx jest --testPathPattern='tests/clinical' --ci --json --outputFile="$tmp_json" || true
jq '{
  passed: (.numPassedTests // 0),
  total: (.numTotalTests // 0),
  rate: (if (.numTotalTests // 0) == 0 then 0 else ((.numPassedTests // 0) / (.numTotalTests // 1) * 100) end)
}' "$tmp_json"
# Expected: { "passed": 21, "total": 22, "rate": 95.45 }
```

### 示例 3：评估报告

```
## 医疗评估：2026-03-27 [提交 abc1234]

### 患者安全：通过

| 类别 | 测试数 | 通过 | 失败 | 状态 |
|------|--------|------|------|------|
| CDSS 准确性 | 39 | 39 | 0 | 通过 |
| PHI 暴露 | 8 | 8 | 0 | 通过 |
| 数据完整性 | 12 | 12 | 0 | 通过 |
| 临床工作流 | 22 | 21 | 1 | 95.5% 通过 |
| 集成合规 | 6 | 6 | 0 | 通过 |

### 覆盖率：84%（目标：80% 以上）
### 结论：可安全部署
```
