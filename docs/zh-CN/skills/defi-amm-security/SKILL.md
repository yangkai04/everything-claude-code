---
name: defi-amm-security
description: Solidity AMM 合约、流动性池和兑换流程的安全检查清单。涵盖重入攻击、CEI 顺序、捐赠/通胀攻击、预言机操纵、滑点、管理员控制和整数运算。
origin: ECC direct-port adaptation
version: "1.0.0"
---

# DeFi AMM 安全

Solidity AMM 合约、LP 金库和兑换函数的关键漏洞模式及加固实现。

## 使用场景

- 编写或审计 Solidity AMM 或流动性池合约
- 实现持有 token 余额的兑换（swap）、存款（deposit）、提款（withdraw）、铸造（mint）或销毁（burn）流程
- 审查任何在份额或储备计算中使用 `token.balanceOf(address(this))` 的合约
- 向 DeFi 协议添加手续费设置器、暂停器、预言机更新或其他管理员函数

## 工作原理

将此文档作为检查清单加模式库使用。对照以下分类审查每个用户入口点，优先使用加固示例而非自行实现。

## 示例

### 重入攻击：强制执行 CEI 顺序

存在漏洞的写法：

```solidity
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    token.transfer(msg.sender, amount);
    balances[msg.sender] -= amount;
}
```

安全写法：

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

已有加固库时，不要自己实现守卫逻辑。

### 捐赠攻击或通胀攻击

在份额计算中直接使用 `token.balanceOf(address(this))` 会让攻击者通过在合约预期路径之外发送 token 来操纵分母。

```solidity
// 存在漏洞
function deposit(uint256 assets) external returns (uint256 shares) {
    shares = (assets * totalShares) / token.balanceOf(address(this));
}
```

```solidity
// 安全写法
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

使用内部账本并计算实际收到的 token 数量。

### 预言机操纵

现货价格可被闪电贷操纵。优先使用 TWAP（时间加权平均价格）。

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

每条兑换路径都需要调用方提供滑点参数和截止时间。

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

### 安全储备计算

```solidity
import {FullMath} from "@uniswap/v3-core/contracts/libraries/FullMath.sol";

uint256 result = FullMath.mulDiv(a, b, c);
```

对于大数储备计算，当存在溢出风险时避免使用简单的 `a * b / c`。

### 管理员控制

```solidity
import {Ownable2Step} from "@openzeppelin/contracts/access/Ownable2Step.sol";

contract MyAMM is Ownable2Step {
    function setFee(uint256 fee) external onlyOwner { ... }
    function pause() external onlyOwner { ... }
}
```

优先使用需要显式接受的所有权转移方式，并为每条特权路径设置访问控制。

## 安全检查清单

- 存在重入风险的入口点使用了 `nonReentrant`
- 遵守了 CEI 顺序（检查-生效-交互，Checks-Effects-Interactions）
- 份额计算不依赖原始的 `balanceOf(address(this))`
- ERC-20 转账使用了 `SafeERC20`
- 存款计算了实际收到的 token 数量
- 预言机读取使用了 TWAP 或其他抗操纵来源
- 兑换要求提供 `amountOutMin` 和 `deadline`
- 溢出敏感的储备计算使用了安全原语（如 `mulDiv`）
- 管理员函数设置了访问控制
- 紧急暂停机制存在且经过测试
- 上线前运行了静态分析和模糊测试

## 审计工具

```bash
pip install slither-analyzer
slither . --exclude-dependencies

echidna-test . --contract YourAMM --config echidna.yaml

forge test --fuzz-runs 10000
```
