# DEX 开发

## 概述

去中心化交易所（DEX）允许用户在无需信任第三方的情况下进行代币交易。

## AMM 模型

| 模型 | 公式 | 代表项目 |
|------|------|----------|
| 恒定乘积 | x * y = k | Uniswap V2 |
| 集中流动性 | 自定义价格区间 | Uniswap V3 |
| 恒定均值 | 多资产池 | Balancer |
| 虚拟余额 | 折价曲线 | Curve |

## Uniswap V2 核心

```solidity
// 添加流动性
function addLiquidity(
    address tokenA,
    address tokenB,
    uint amountADesired,
    uint amountBDesired,
    uint amountAMin,
    uint amountBMin,
    address to,
    uint deadline
) external returns (uint amountA, uint amountB, uint liquidity);

// 代币交换
function swapExactTokensForTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

## 开发要点

1. 价格滑点保护
2. 前端运行（MEV）防护
3. 多跳路径优化
4. 费率计算精度
