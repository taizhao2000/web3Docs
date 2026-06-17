# Truffle 框架

## 概述

Truffle 是最早流行的以太坊开发框架，提供合约编译、部署、测试的一站式解决方案。

> **注意**：Truffle 已被 Consensys 弃用，建议新项目使用 Hardhat 或 Foundry。

## 核心组件

- **Truffle**：开发框架
- **Ganache**：本地区块链模拟器
- **Drizzle**：前端数据同步库

## 项目结构

```
my-project/
├── contracts/         # 智能合约
├── migrations/        # 迁移脚本
├── test/              # 测试文件
└── truffle-config.js   # 配置文件
```

## 迁移指南

从 Truffle 迁移到 Hardhat/Foundry 的主要注意事项：

1. 迁移脚本对应关系不同
2. 测试框架 API 差异
3. 配置文件格式不同
4. 插件生态需要重新选择
