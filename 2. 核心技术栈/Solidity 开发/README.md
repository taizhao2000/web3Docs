# Solidity 开发

Solidity 是以太坊智能合约的主要编程语言，是一种静态类型的面向对象语言。

## 学习路径

```
语法基础 → 进阶模式 → 数据布局与存储
   │           │              │
   │           │              └─ 理解 EVM 存储模型，写出 Gas 高效的代码
   │           └─ 继承、接口、库、代理模式、设计原则
   └─ 类型系统、函数、事件、错误处理、全局变量
```

> **前置知识**：建议先阅读 [智能合约的本质](../../1.%20基础概念/智能合约/智能合约的本质.md)，理解智能合约与普通程序的根本区别。

## 文档索引

| 文档 | 难度 | 内容 |
|------|------|------|
| [Solidity 语法基础](./Solidity%20语法基础.md) | ⭐ 入门 | 值类型/引用类型、变量与作用域、函数修饰符、事件、错误处理、全局变量 |
| [Solidity 进阶模式](./Solidity%20进阶模式.md) | ⭐⭐ 进阶 | 继承与多态、接口与抽象合约、库、合约工厂、状态机、代理升级模式、ERC-20 完整实现 |
| [Solidity 数据布局与存储](./Solidity%20数据布局与存储.md) | ⭐⭐⭐ 深入 | Storage slot 机制、存储打包、Mapping/数组存储、calldata/memory/storage 对比、Gas 优化、字节码理解 |

## 版本演进

| 版本 | 关键变化 | 影响 |
|------|---------|------|
| 0.4.x | 早期版本 | 很多旧合约仍在此版本 |
| 0.5.x | 重大 API 变更 | 构造函数改为 `constructor()` |
| 0.6.x | 抽象合约、继承变更 | virtual/override 关键字引入 |
| 0.7.x | 小幅度改进 | `now` → `block.timestamp` |
| 0.8.x | **内置溢出检查、自定义错误** | SafeMath 不再需要，Gas 优化新范式 |

## 快速示例

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

## 相关章节

- [智能合约的本质](../../1.%20基础概念/智能合约/智能合约的本质.md) — 理解智能合约与普通程序的根本区别
- [智能合约部署与升级成本](../../8.%20测试与部署/部署流程/智能合约部署与升级成本.md) — 部署条件、EVM 限制、升级模式详解
- [Hardhat 开发环境](../Hardhat%20开发环境/) — 编译、测试、部署工具链
- [Foundry 工具链](../Foundry%20工具链/) — 高速 Solidity 开发工具链
