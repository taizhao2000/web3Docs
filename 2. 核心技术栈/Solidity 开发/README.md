# Solidity 开发

## 概述

Solidity 是以太坊智能合约的主要编程语言，是一种静态类型的面向对象语言。

## 基本结构

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract SimpleStorage {
    uint256 private storedValue;
    
    event ValueChanged(uint256 newValue);
    
    function store(uint256 _value) public {
        storedValue = _value;
        emit ValueChanged(_value);
    }
    
    function retrieve() public view returns (uint256) {
        return storedValue;
    }
}
```

## 数据类型

| 类型 | 关键字 | 说明 |
|------|--------|------|
| 值类型 | uint, int, bool, address | 直接存储值 |
| 引用类型 | string, array, struct, mapping | 存储引用 |
| 特殊类型 | enum, contract | 枚举和合约类型 |

## 关键特性

- **继承**：支持多重继承
- **接口**：定义合约接口
- **库**：可复用的代码片段
- **事件**：链上日志记录
- **修饰符**：函数访问控制
- **错误处理**：require/revert/custom errors

## 版本演进

- Solidity 0.4.x：早期版本
- Solidity 0.5.x：重大 API 变更
- Solidity 0.6.x：抽象合约、继承变更
- Solidity 0.7.x：小幅度改进
- Solidity 0.8.x：内置溢出检查、自定义错误
