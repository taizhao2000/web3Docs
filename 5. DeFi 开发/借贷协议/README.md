# 借贷协议

## 概述

去中心化借贷协议允许用户以加密资产作为抵押品借入其他资产，或存入资产赚取利息。

## 核心机制

- **超额抵押**：抵押品价值必须大于借款价值
- **清算**：抵押率不足时触发清算
- **利率模型**：基于供需的动态利率
- **利率跳跃模型**：不同利用率区间不同利率

## 代表项目

| 项目 | 特点 |
|------|------|
| Aave | 闪电贷、多利率模型 |
| Compound | 算法利率、cToken |
| MakerDAO | 超额抵押稳定币 |
| Euler | 无许可借贷市场 |

## 关键合约接口

```solidity
// 存入资产
function deposit(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;

// 借出资产
function borrow(address asset, uint256 amount, uint256 interestRateMode, uint16 referralCode, address onBehalfOf) external;

// 偿还借款
function repay(address asset, uint256 amount, uint256 rateMode, address onBehalfOf) external returns (uint256);

// 清算
function liquidationCall(address collateral, address debt, address user, uint256 debtToCover, bool receiveAToken) external;
```
