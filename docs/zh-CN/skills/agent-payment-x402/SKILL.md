---
name: agent-payment-x402
description: 为 AI 代理添加 x402 支付执行能力——包含每任务预算、支出控制和通过 MCP 工具实现的非托管钱包。当代理需要为 API、服务或其他代理付费时使用。
origin: community
---

# 代理支付执行（x402）

为 AI 代理启用内置支出控制的自主支付能力。使用 x402 HTTP 支付协议和 MCP 工具，代理无需托管风险即可为外部服务、API 或其他代理付费。

## 何时使用

当以下情况时使用：代理需要为 API 调用付费、购买服务、向另一个代理结算、执行每任务支出限制，或管理非托管钱包。可与 cost-aware-llm-pipeline 和 security-review 技能自然配合使用。

## 工作原理

### x402 协议
x402 将 HTTP 402（需要付款）扩展为机器可协商的流程。当服务器返回 `402` 时，代理的支付工具会自动协商价格、检查预算、签署交易并重试——无需人工介入。

### 支出控制
每次支付工具调用都会执行 `SpendingPolicy`（支出策略）：
- **每任务预算** — 单次代理操作的最大支出
- **每会话预算** — 整个会话的累计限额
- **白名单收款方** — 限制代理可付款的地址/服务
- **频率限制** — 每分钟/小时最大交易数

### 非托管钱包
代理通过 ERC-4337 智能账户持有自己的密钥。编排器在委托前设置策略；代理只能在限额内支出。无共享资金池，无托管风险。

## MCP 集成

支付层通过标准 MCP 工具暴露接口，可插入任何 Claude Code 或代理运行框架。

> **安全说明**：始终固定软件包版本。此工具管理私钥——使用未固定版本的 `npx` 安装会引入供应链风险。

```json
{
  "mcpServers": {
    "agentpay": {
      "command": "npx",
      "args": ["agentwallet-sdk@6.0.0"]
    }
  }
}
```

### 可用工具（代理可调用）

| 工具 | 用途 |
|------|---------|
| `get_balance` | 查询代理钱包余额 |
| `send_payment` | 向地址或 ENS 发送付款 |
| `check_spending` | 查询剩余预算 |
| `list_transactions` | 所有付款的审计追踪 |

> **注意**：支出策略由**编排器**在委托给代理之前设置——而非由代理本身设置。这可防止代理提升自己的支出限额。通过编排层或任务前钩子中的 `set_policy` 配置策略，切勿将其作为代理可调用的工具。

## 示例

### 在 MCP 客户端中执行预算控制

构建调用 agentpay MCP 服务器的编排器时，在分发付费工具调用前执行预算限制。

> **前提条件**：在添加 MCP 配置之前先安装软件包——在非交互式环境中，不带 `-y` 的 `npx` 会提示确认，导致服务器挂起：`npm install -g agentwallet-sdk@6.0.0`

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function main() {
  // 1. 在构建 transport 之前验证凭据。
  //    缺少密钥必须立即失败——切勿在没有认证的情况下启动子进程。
  const walletKey = process.env.WALLET_PRIVATE_KEY;
  if (!walletKey) {
    throw new Error("WALLET_PRIVATE_KEY is not set — refusing to start payment server");
  }

  // 通过 stdio transport 连接到 agentpay MCP 服务器。
  // 只白名单服务器需要的环境变量——切勿将整个 process.env
  // 转发给管理私钥的第三方子进程。
  const transport = new StdioClientTransport({
    command: "npx",
    args: ["agentwallet-sdk@6.0.0"],
    env: {
      PATH: process.env.PATH ?? "",
      NODE_ENV: process.env.NODE_ENV ?? "production",
      WALLET_PRIVATE_KEY: walletKey,
    },
  });
  const agentpay = new Client({ name: "orchestrator", version: "1.0.0" });
  await agentpay.connect(transport);

  // 2. 在委托给代理之前设置支出策略。
  //    始终验证成功——静默失败意味着没有任何控制生效。
  const policyResult = await agentpay.callTool({
    name: "set_policy",
    arguments: {
      per_task_budget: 0.50,
      per_session_budget: 5.00,
      allowlisted_recipients: ["api.example.com"],
    },
  });
  if (policyResult.isError) {
    throw new Error(
      `Failed to set spending policy — do not delegate: ${JSON.stringify(policyResult.content)}`
    );
  }

  // 3. 在任何付费操作之前使用 preToolCheck
  await preToolCheck(agentpay, 0.01);
}

// 工具前钩子：具有四条不同错误路径的失败关闭预算控制。
async function preToolCheck(agentpay: Client, apiCost: number): Promise<void> {
  // 路径 1：拒绝无效输入（NaN/Infinity 会绕过 < 比较）
  if (!Number.isFinite(apiCost) || apiCost < 0) {
    throw new Error(`Invalid apiCost: ${apiCost} — action blocked`);
  }

  // 路径 2：transport/连接故障
  let result;
  try {
    result = await agentpay.callTool({ name: "check_spending" });
  } catch (err) {
    throw new Error(`Payment service unreachable — action blocked: ${err}`);
  }

  // 路径 3：工具返回错误（例如认证失败、钱包未初始化）
  if (result.isError) {
    throw new Error(
      `check_spending failed — action blocked: ${JSON.stringify(result.content)}`
    );
  }

  // 路径 4：解析并验证响应格式
  let remaining: number;
  try {
    const parsed = JSON.parse(
      (result.content as Array<{ text: string }>)[0].text
    );
    if (!Number.isFinite(parsed?.remaining)) {
      throw new TypeError("missing or non-finite 'remaining' field");
    }
    remaining = parsed.remaining;
  } catch (err) {
    throw new Error(
      `check_spending returned unexpected format — action blocked: ${err}`
    );
  }

  // 路径 5：预算超出
  if (remaining < apiCost) {
    throw new Error(
      `Budget exceeded: need $${apiCost} but only $${remaining} remaining`
    );
  }
}

main().catch((err) => {
  console.error(err);
  process.exitCode = 1;
});
```

## 最佳实践

- **委托前设置预算**：派生子代理时，通过编排层附加 SpendingPolicy。切勿赋予代理无限支出权限。
- **固定依赖版本**：在 MCP 配置中始终指定精确版本（例如 `agentwallet-sdk@6.0.0`）。部署到生产环境前验证软件包完整性。
- **审计追踪**：在任务后钩子中使用 `list_transactions` 记录支出内容和原因。
- **失败关闭**：如果支付工具不可访问，阻止付费操作——不要回退到无计量访问。
- **配合 security-review 使用**：支付工具是高权限工具。应像对待 shell 访问一样严格审查。
- **先用测试网测试**：开发使用 Base Sepolia；生产使用 Base 主网。

## 生产参考

- **npm**：[`agentwallet-sdk`](https://www.npmjs.com/package/agentwallet-sdk)
- **已合并到 NVIDIA NeMo Agent Toolkit**：[PR #17](https://github.com/NVIDIA/NeMo-Agent-Toolkit-Examples/pull/17) — NVIDIA 代理示例的 x402 支付工具
- **协议规范**：[x402.org](https://x402.org)
