---
name: evm-token-decimals
description: 防止 EVM 链上的静默小数点不匹配错误。涵盖运行时小数位查询、链感知缓存、跨链桥接代币精度漂移，以及机器人、仪表板和 DeFi 工具中的安全归一化处理。
origin: ECC direct-port adaptation
version: "1.0.0"
---

# EVM 代币小数位

静默的小数点不匹配是最容易导致余额或美元价值偏差几个数量级却不抛出任何错误的问题之一。

## 何时使用

- 在 Python、TypeScript 或 Solidity 中读取 ERC-20 余额
- 从链上余额计算法币价值
- 跨多条 EVM 链比较代币数量
- 处理跨链桥接资产
- 构建投资组合追踪器、机器人或聚合器

## 工作原理

永远不要假设稳定币在任何地方都使用相同的小数位。在运行时查询 `decimals()`，按 `(chain_id, token_address)` 缓存，并使用高精度数学进行价值计算。

## 示例

### 运行时查询小数位

```python
from decimal import Decimal
from web3 import Web3

ERC20_ABI = [
    {"name": "decimals", "type": "function", "inputs": [],
     "outputs": [{"type": "uint8"}], "stateMutability": "view"},
    {"name": "balanceOf", "type": "function",
     "inputs": [{"name": "account", "type": "address"}],
     "outputs": [{"type": "uint256"}], "stateMutability": "view"},
]

def get_token_balance(w3: Web3, token_address: str, wallet: str) -> Decimal:
    contract = w3.eth.contract(
        address=Web3.to_checksum_address(token_address),
        abi=ERC20_ABI,
    )
    decimals = contract.functions.decimals().call()
    raw = contract.functions.balanceOf(Web3.to_checksum_address(wallet)).call()
    return Decimal(raw) / Decimal(10 ** decimals)
```

不要硬编码 `1_000_000`，因为某个代币在其他地方通常有 6 位小数以外的精度。

### 按链和代币缓存

```python
from functools import lru_cache

@lru_cache(maxsize=512)
def get_decimals(chain_id: int, token_address: str) -> int:
    w3 = get_web3_for_chain(chain_id)
    contract = w3.eth.contract(
        address=Web3.to_checksum_address(token_address),
        abi=ERC20_ABI,
    )
    return contract.functions.decimals().call()
```

### 防御性处理异常代币

```python
try:
    decimals = contract.functions.decimals().call()
except Exception:
    logging.warning(
        "decimals() reverted on %s (chain %s), defaulting to 18",
        token_address,
        chain_id,
    )
    decimals = 18
```

记录回退行为并保持可见性。老旧或非标准代币依然存在。

### 在 Solidity 中归一化为 18 位小数的 WAD

```solidity
interface IERC20Metadata {
    function decimals() external view returns (uint8);
}

function normalizeToWad(address token, uint256 amount) internal view returns (uint256) {
    uint8 d = IERC20Metadata(token).decimals();
    if (d == 18) return amount;
    if (d < 18) return amount * 10 ** (18 - d);
    return amount / 10 ** (d - 18);
}
```

### 使用 ethers 的 TypeScript 示例

```typescript
import { Contract, formatUnits } from 'ethers';

const ERC20_ABI = [
  'function decimals() view returns (uint8)',
  'function balanceOf(address) view returns (uint256)',
];

async function getBalance(provider: any, tokenAddress: string, wallet: string): Promise<string> {
  const token = new Contract(tokenAddress, ERC20_ABI, provider);
  const [decimals, raw] = await Promise.all([
    token.decimals(),
    token.balanceOf(wallet),
  ]);
  return formatUnits(raw, decimals);
}
```

### 快速链上检查

```bash
cast call <token_address> "decimals()(uint8)" --rpc-url <rpc>
```

## 规则

- 始终在运行时查询 `decimals()`
- 按链加代币地址缓存，而非按代币符号
- 使用 `Decimal`、`BigInt` 或等效的精确数学，而非浮点数
- 桥接或包装器变更后重新查询小数位
- 在比较或定价前，统一对内部账目进行归一化处理
