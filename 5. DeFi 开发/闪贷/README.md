# 闪贷

## 概述

闪贷（Flash Loan）是一种无需抵押的瞬时贷款，借款和还款在同一个交易内完成。如果还款失败，整笔交易回滚。

## 核心原理

```
交易开始 → 借入资产 → 执行套利/清算操作 → 归还资产+手续费 → 交易结束
                                                              ↑
                                                    如果归还失败，整个交易回滚
```

## 使用场景

| 场景 | 说明 |
|------|------|
| 套利 | 跨 DEX 价格差异 |
| 清算 | 借闪贷清算不足抵押的借款 |
| 抵押品置换 | 替换 DeFi 仓位的抵押品 |
| 自我清算 | 无需已有资金即可清算 |

## Aave 闪贷示例

```solidity
// 1. 调用闪贷
function executeFlashLoan() external {
    address[] memory assets = new address[](1);
    assets[0] = address(DAI);
    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 1000000 * 1e18; // 100万 DAI
    
    LENDING_POOL.flashLoan(
        address(this),  // 接收者
        assets,
        amounts,
        new uint256[](1), // 利率模式
        address(this),   // onBehalfOf
        "",              // 参数
        0                // referralCode
    );
}

// 2. 实现回调
function executeOperation(
    address[] calldata assets,
    uint256[] calldata amounts,
    uint256[] calldata premiums,
    address initiator,
    bytes calldata params
) external returns (bool) {
    // 在这里执行套利逻辑
    
    // 归还借款 + 手续费
    for (uint i = 0; i < assets.length; i++) {
        uint amountOwing = amounts[i] + premiums[i];
        IERC20(assets[i]).approve(address(LENDING_POOL), amountOwing);
    }
    return true;
}
```
