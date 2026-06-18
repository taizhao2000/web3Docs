# Truffle 框架

Truffle 是最早流行的以太坊开发框架，提供合约编译、部署、测试的一站式解决方案。

> **⚠️ 弃用通知**：Truffle 已被 Consensys 于 2023 年正式弃用，不再接收安全更新。**新项目请使用 [Hardhat](../Hardhat%20开发环境/) 或 [Foundry](../Foundry%20工具链/)**。本目录文档旨在帮助开发者**理解旧项目**和**完成迁移**。

## 文档索引

| 文档 | 定位 | 内容 |
|------|------|------|
| [Truffle 框架完整指南](./Truffle%20框架完整指南.md) | 理解旧项目 | 项目结构、配置详解、编译、迁移脚本、测试、Ganache、部署到测试网/主网、前端集成、命令速查 |
| [Truffle 迁移指南](./Truffle%20迁移指南.md) | 迁移到新工具 | Truffle → Hardhat / Foundry 完整转换映射、逐步迁移实战、常见坑点、工具链选择决策树 |

## 核心组件

| 组件 | 作用 | 现状 | 替代方案 |
|------|------|------|---------|
| **Truffle** | 主体框架（编译、部署、测试） | ❌ 已弃用 | Hardhat / Foundry |
| **Ganache** | 本地区块链模拟器 | ⚠️ 可用但停更 | Hardhat Network / Anvil |
| **Drizzle** | React 前端数据同步 | ❌ 已弃用 | wagmi + viem |
| **@truffle/hdwallet-provider** | HD Wallet 签名 | ⚠️ 独立维护 | ethers.js 内置 |

## 快速对比

| 特性 | Truffle | Hardhat | Foundry |
|------|---------|---------|---------|
| 语言 | JavaScript | JavaScript/TypeScript | Solidity + Rust |
| 编译速度 | 慢（solcjs） | 中等 | **快** |
| 测试速度 | 慢 | 中等 | **快** |
| Fuzz 测试 | ❌ | ⚠️ 插件 | **✅ 原生** |
| 本地链 | Ganache | 内置 | Anvil |
| 推荐度 | ❌ 不推荐新项目 | ✅ JS 生态首选 | ✅ 性能首选 |

## 迁移路线

```
Truffle (已弃用)
    │
    ├── 迁移到 Hardhat（成本最低，同为 JS 生态）
    │
    └── 迁移到 Foundry（长期收益最大，性能最优）
```

详见 [Truffle 迁移指南](./Truffle%20迁移指南.md)。
