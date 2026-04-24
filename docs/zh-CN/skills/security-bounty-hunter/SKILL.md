---
name: security-bounty-hunter
description: 在代码仓库中搜寻可利用的、值得提交赏金的安全问题。专注于可远程到达的漏洞，这些漏洞符合真实报告的要求，而非仅限于本地的噪音性发现。
origin: ECC direct-port adaptation
version: "1.0.0"
---

# 安全赏金猎人

当目标是为负责任披露或赏金提交进行实际漏洞发现时使用本技能，而非进行宽泛的最佳实践评审。

## 何时使用

- 扫描代码仓库以发现可利用的漏洞
- 准备 Huntr、HackerOne 或类似赏金提交
- 需要评判"这是否真的值钱？"而非"这在理论上是否不安全？"

## 工作原理

偏向于远程可达、用户可控的攻击路径，丢弃平台通常以"信息性"或"超出范围"拒绝的模式。

## 范围内模式

这些是持续产生价值的问题类型：

| 模式 | CWE | 典型影响 |
| --- | --- | --- |
| 通过用户可控 URL 的 SSRF | CWE-918 | 内网访问、云元数据窃取 |
| 中间件或 API 守卫中的认证绕过 | CWE-287 | 未授权账户或数据访问 |
| 远程反序列化或上传到 RCE 路径 | CWE-502 | 代码执行 |
| 可达端点中的 SQL 注入 | CWE-89 | 数据外泄、认证绕过、数据销毁 |
| 请求处理程序中的命令注入 | CWE-78 | 代码执行 |
| 文件服务路径中的路径遍历 | CWE-22 | 任意文件读写 |
| 自动触发的 XSS | CWE-79 | 会话劫持、管理员账户入侵 |

## 跳过这些

这些通常信噪比低或超出赏金范围，除非程序另有说明：

- 无远程路径的仅本地 `pickle.loads`、`torch.load` 或等效操作
- 仅 CLI 工具中的 `eval()` 或 `exec()`
- 完全硬编码命令上的 `shell=True`
- 单独的缺失安全头
- 无利用影响的通用限速投诉
- 需要受害者手动粘贴代码的自我 XSS
- 不属于目标程序范围的 CI/CD 注入
- 演示、示例或仅测试代码

## 工作流

1. 先检查范围：程序规则、SECURITY.md、披露渠道和排除项。
2. 找到真实入口点：HTTP 处理程序、上传、后台任务、webhook、解析器和集成端点。
3. 在有帮助的地方运行静态工具，但仅将其作为分类输入。
4. 端到端阅读真实代码路径。
5. 证明用户控制到达了有意义的汇聚点。
6. 用最小的安全 PoC 确认可利用性和影响。
7. 在起草报告前检查重复项。

## 示例分类循环

```bash
semgrep --config=auto --severity=ERROR --severity=WARNING --json
```

然后手动过滤：

- 丢弃测试、演示、fixture、外部引入代码
- 丢弃仅本地或不可达的路径
- 仅保留具有明确网络或用户可控路由的发现

## 报告结构

```markdown
## Description
[What the vulnerability is and why it matters]

## Vulnerable Code
[File path, line range, and a small snippet]

## Proof of Concept
[Minimal working request or script]

## Impact
[What the attacker can achieve]

## Affected Version
[Version, commit, or deployment target tested]
```

## 质量关卡

提交前：

- 代码路径可从真实用户或网络边界到达
- 输入是真正的用户可控
- 汇聚点有意义且可利用
- PoC 可正常工作
- 该问题尚未被公告、CVE 或开放工单覆盖
- 目标实际在赏金程序的范围内
