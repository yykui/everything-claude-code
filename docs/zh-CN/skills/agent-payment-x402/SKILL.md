---
name: agent-payment-x402
description: 为 AI 智能体添加 x402 支付执行能力——每任务预算、支出控制和通过 MCP 工具实现的非托管钱包。当智能体需要为 API、服务或其他智能体付费时使用。
origin: community
---

# 智能体支付执行（x402）

为 AI 智能体启用内置支出控制的自主支付能力。使用 x402 HTTP 支付协议和 MCP 工具，使智能体能够为外部服务、API 或其他智能体付费，同时无托管风险。

## 何时使用

当以下情况时使用：你的智能体需要为 API 调用付费、购买服务、与其他智能体结算、强制执行每任务支出限制，或管理非托管钱包。自然地与 cost-aware-llm-pipeline 和 security-review 技能配合使用。

## 工作原理

### x402 协议
x402 将 HTTP 402（需要付款）扩展为机器可协商的流程。当服务器返回 `402` 时，智能体的支付工具会自动协商价格、检查预算、签署交易并重试——无需人工干预。

### 支出控制
每次支付工具调用都强制执行 `SpendingPolicy`：
- **每任务预算** — 单个智能体操作的最大支出
- **每会话预算** — 整个会话的累计限额
- **白名单接收方** — 限制智能体可以向哪些地址/服务付款
- **频率限制** — 每分钟/小时最大交易数

### 非托管钱包
智能体通过 ERC-4337 智能账户持有自己的密钥。编排器在委托前设置策略；智能体只能在限额内支出。无资金池，无托管风险。

## MCP 集成

支付层提供标准 MCP 工具，可接入任何 Claude Code 或智能体运行时设置。

> **安全说明**：始终固定软件包版本。此工具管理私钥——未固定的 `npx` 安装会引入供应链风险。

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

### 可用工具（智能体可调用）

| 工具 | 用途 |
|------|---------|
| `get_balance` | 检查智能体钱包余额 |
| `send_payment` | 向地址或 ENS 发送付款 |
| `check_spending` | 查询剩余预算 |
| `list_transactions` | 所有付款的审计跟踪 |

> **注意**：支出策略由**编排器**在委托给智能体之前设置——而非由智能体自身设置。这可防止智能体自行提升支出限额。通过编排层或预任务钩子中的 `set_policy` 配置策略，绝不将其作为智能体可调用的工具。

## 示例

### MCP 客户端中的预算强制执行

在构建调用 agentpay MCP 服务器的编排器时，在分发付费工具调用之前强制执行预算。

> **前提条件**：在添加 MCP 配置之前安装软件包——不带 `-y` 的 `npx` 在非交互式环境中会提示确认，导致服务器挂起：`npm install -g agentwallet-sdk@6.0.0`

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function main() {
  // 1. Validate credentials before constructing the transport.
  //    A missing key must fail immediately — never let the subprocess start without auth.
  const walletKey = process.env.WALLET_PRIVATE_KEY;
  if (!walletKey) {
    throw new Error("WALLET_PRIVATE_KEY is not set — refusing to start payment server");
  }

  // Connect to the agentpay MCP server via stdio transport.
  // Whitelist only the env vars the server needs — never forward all of process.env
  // to a third-party subprocess that manages private keys.
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

  // 2. Set spending policy before delegating to the agent.
  //    Always verify success — a silent failure means no controls are active.
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

  // 3. Use preToolCheck before any paid action
  await preToolCheck(agentpay, 0.01);
}

// Pre-tool hook: fail-closed budget enforcement with four distinct error paths.
async function preToolCheck(agentpay: Client, apiCost: number): Promise<void> {
  // Path 1: Reject invalid input (NaN/Infinity bypass the < comparison)
  if (!Number.isFinite(apiCost) || apiCost < 0) {
    throw new Error(`Invalid apiCost: ${apiCost} — action blocked`);
  }

  // Path 2: Transport/connectivity failure
  let result;
  try {
    result = await agentpay.callTool({ name: "check_spending" });
  } catch (err) {
    throw new Error(`Payment service unreachable — action blocked: ${err}`);
  }

  // Path 3: Tool returned an error (e.g., auth failure, wallet not initialised)
  if (result.isError) {
    throw new Error(
      `check_spending failed — action blocked: ${JSON.stringify(result.content)}`
    );
  }

  // Path 4: Parse and validate the response shape
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

  // Path 5: Budget exceeded
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

- **委托前设置预算**：生成子智能体时，通过编排层附加 SpendingPolicy。切勿给智能体无限支出权限。
- **固定依赖版本**：始终在 MCP 配置中指定精确版本（如 `agentwallet-sdk@6.0.0`）。在部署到生产环境之前验证软件包完整性。
- **审计跟踪**：在后任务钩子中使用 `list_transactions` 记录支出内容和原因。
- **失败关闭**：如果支付工具不可达，阻止付费操作——不要回退到无计量访问。
- **与 security-review 配合**：支付工具是高权限工具。应用与 shell 访问相同的审查标准。
- **先用测试网测试**：开发时使用 Base Sepolia；生产环境切换到 Base 主网。

## 生产参考

- **npm**: [`agentwallet-sdk`](https://www.npmjs.com/package/agentwallet-sdk)
- **已并入 NVIDIA NeMo Agent Toolkit**: [PR #17](https://github.com/NVIDIA/NeMo-Agent-Toolkit-Examples/pull/17) — NVIDIA 智能体示例的 x402 支付工具
- **协议规范**: [x402.org](https://x402.org)
