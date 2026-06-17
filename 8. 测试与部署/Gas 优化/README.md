# Gas 优化

## 概述

Gas 优化是智能合约开发中降低交易成本的关键环节，直接影响用户体验和协议可持续性。

## 存储优化

| 优化策略 | 节省 | 说明 |
|----------|------|------|
| 打包存储变量 | ~20,000 Gas | 将多个小变量放入同一存储槽 |
| 使用 calldata | ~200 Gas/字 | 只读参数使用 calldata |
| 缓存存储变量 | ~100 Gas | 循环中缓存 `storage` 变量到 `memory` |
| 使用 mapping | - | 比 array 更省 Gas |
| 短字符串 | ~20,000 Gas | 32 字节以内用 bytes32 替代 string |

## 代码示例

### 变量打包

```solidity
// ❌ 占用 2 个存储槽
struct Bad {
    uint256 a;  // 槽 0
    uint8 b;    // 槽 1
}

// ✅ 占用 1 个存储槽
struct Good {
    uint256 a;  // 槽 0
    uint8 b;    // 仍在槽 0（打包在一起）
}
```

### 缓存存储变量

```solidity
// ❌ 每次循环都读存储
uint256 total;
for (uint i = 0; i < array.length; i++) {
    total += array[i];
}

// ✅ 缓存到内存
uint256 total;
uint256 len = array.length; // 缓存 length
for (uint i = 0; i < len; ) {
    total += array[i];
    unchecked { ++i; } // 取消溢出检查
}
```

### 使用 Custom Errors

```solidity
// ❌ 字符串错误（更贵）
require(balance > 0, "Insufficient balance");

// ✅ Custom Error（更省）
error InsufficientBalance();
if (balance <= 0) revert InsufficientBalance();
```

## Gas 消耗参考

| 操作 | Gas |
|------|-----|
| 交易基础成本 | 21,000 |
| SSTORE（新值） | 20,000 |
| SSTORE（更新） | 5,000 |
| SLOAD | 2,100 |
| LOG（1 topic） | 1,125 |
| CALL | 700+ |
