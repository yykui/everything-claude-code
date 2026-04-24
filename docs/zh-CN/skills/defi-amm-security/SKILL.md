---
name: defi-amm-security
description: Solidity AMM 合约、流动性池和交换流程的安全检查清单。涵盖重入攻击、CEI 顺序、捐赠或通胀攻击、预言机操纵、滑点、管理员控制和整数运算。
origin: ECC direct-port adaptation
version: "1.0.0"
---

# DeFi AMM 安全

Solidity AMM 合约、LP 金库和交换函数的关键漏洞模式及加固实现。

## 何时使用

- 编写或审计 Solidity AMM 或流动性池合约
- 实现持有代币余额的交换、存入、提取、铸造或销毁流程
- 审查在份额或储备计算中使用 `token.balanceOf(address(this))` 的合约
- 向 DeFi 协议添加手续费设置、暂停、预言机更新或其他管理员函数

## 工作原理

将本技能作为检查清单加模式库使用。对照以下类别审查每个用户入口点，并优先使用加固示例而非手写实现。

## 示例

### 重入攻击：强制执行 CEI 顺序

存在漏洞：

```solidity
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    token.transfer(msg.sender, amount);
    balances[msg.sender] -= amount;
}
```

安全实现：

```solidity
import {ReentrancyGuard} from "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

using SafeERC20 for IERC20;

function withdraw(uint256 amount) external nonReentrant {
    require(balances[msg.sender] >= amount, "Insufficient");
    balances[msg.sender] -= amount;
    token.safeTransfer(msg.sender, amount);
}
```

当存在加固库时，不要自己编写守卫。

### 捐赠或通胀攻击

在份额计算中直接使用 `token.balanceOf(address(this))` 会让攻击者通过在预期路径之外向合约发送代币来操纵分母。

```solidity
// 存在漏洞
function deposit(uint256 assets) external returns (uint256 shares) {
    shares = (assets * totalShares) / token.balanceOf(address(this));
}
```

```solidity
// 安全实现
uint256 private _totalAssets;

function deposit(uint256 assets) external nonReentrant returns (uint256 shares) {
    uint256 balBefore = token.balanceOf(address(this));
    token.safeTransferFrom(msg.sender, address(this), assets);
    uint256 received = token.balanceOf(address(this)) - balBefore;

    shares = totalShares == 0 ? received : (received * totalShares) / _totalAssets;
    _totalAssets += received;
    totalShares += shares;
}
```

跟踪内部账目，并测量实际收到的代币数量。

### 预言机操纵

即时价格容易被闪电贷操纵。优先使用 TWAP。

```solidity
uint32[] memory secondsAgos = new uint32[](2);
secondsAgos[0] = 1800;
secondsAgos[1] = 0;
(int56[] memory tickCumulatives,) = IUniswapV3Pool(pool).observe(secondsAgos);
int24 twapTick = int24(
    (tickCumulatives[1] - tickCumulatives[0]) / int56(uint56(30 minutes))
);
uint160 sqrtPriceX96 = TickMath.getSqrtRatioAtTick(twapTick);
```

### 滑点保护

每条交换路径都需要调用方提供滑点参数和截止时间。

```solidity
function swap(
    uint256 amountIn,
    uint256 amountOutMin,
    uint256 deadline
) external returns (uint256 amountOut) {
    require(block.timestamp <= deadline, "Expired");
    amountOut = _calculateOut(amountIn);
    require(amountOut >= amountOutMin, "Slippage exceeded");
    _executeSwap(amountIn, amountOut);
}
```

### 安全的储备运算

```solidity
import {FullMath} from "@uniswap/v3-core/contracts/libraries/FullMath.sol";

uint256 result = FullMath.mulDiv(a, b, c);
```

对于大型储备运算，当存在溢出风险时，避免使用简单的 `a * b / c`。

### 管理员控制

```solidity
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

contract MyAMM is Ownable2Step {
    function setFee(uint256 fee) external onlyOwner { ... }
    function pause() external onlyOwner { ... }
}
```

优先对所有权转移使用显式确认，并对每个特权路径设置门控。

## 安全检查清单

- 存在重入漏洞的入口点使用了 `nonReentrant`
- CEI 顺序得到遵守
- 份额计算不依赖原始的 `balanceOf(address(this))`
- ERC-20 转账使用 `SafeERC20`
- 存入操作测量实际收到的代币
- 预言机读取使用 TWAP 或其他抗操纵来源
- 交换需要 `amountOutMin` 和 `deadline`
- 对溢出敏感的储备运算使用 `mulDiv` 等安全原语
- 管理员函数有访问控制
- 紧急暂停机制存在且经过测试
- 上线前已运行静态分析和模糊测试

## 审计工具

```bash
pip install slither-analyzer
slither . --exclude-dependencies

echidna-test . --contract YourAMM --config echidna.yaml

forge test --fuzz-runs 10000
```
