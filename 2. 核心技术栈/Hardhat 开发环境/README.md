# Hardhat 开发环境

Hardhat 是专业的以太坊开发环境，用于编译、部署、测试和调试智能合约。JavaScript/TypeScript 生态中最成熟的 Web3 开发框架。

## 学习路径

```
Hardhat 开发环境完整指南 → Hardhat 高级配置与插件
         │                        │
         │                        └─ 插件生态、自定义任务、部署管理、CI/CD
         └─ 安装、配置、编译、Hardhat Network、部署、测试、调试
```

> **前置知识**：建议先阅读 [Solidity 语法基础](../Solidity%20开发/Solidity%20语法基础.md)，理解 Solidity 的基本语法。

## 文档索引

| 文档 | 难度 | 内容 |
|------|------|------|
| [Hardhat 开发环境完整指南](./Hardhat%20开发环境完整指南.md) | ⭐ 入门 | 安装与初始化、hardhat.config.ts 配置详解、合约编译、Hardhat Network（内置开发链/主网Fork/impersonate/时间操作）、传统脚本部署与 Ignition 声明式部署、测试框架（ethers.js v6 + chai）、console.log 调试、Etherscan 验证、命令速查、与 Foundry 共存 |
| [Hardhat 高级配置与插件](./Hardhat%20高级配置与插件.md) | ⭐⭐ 进阶 | 工具包/插件详解（toolbox/gas-reporter/coverage/upgrades/verify）、自定义任务（task）系统、Hardhat Network 高级功能（挖矿/日志/状态修改）、hardhat-deploy 部署管理、CI/CD 集成（GitHub Actions）、常见问题排查 |

## 核心特性

- **内置 Hardhat Network**：无需启动 Ganache，测试时自动启动
- **Solidity console.log**：在合约中直接使用 `console.log` 调试
- **TypeScript 原生支持**：完整的类型提示和编译时检查
- **主网 Fork**：在本地模拟主网状态，操作真实合约
- **Mocha + Chai 测试**：熟悉的 JS 测试体验
- **丰富的插件生态**：Gas 报告、覆盖率、代理升级、Etherscan 验证

## 快速开始

```bash
mkdir my-dapp && cd my-dapp
npm init -y
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npx hardhat init    # 选择 TypeScript 项目
npx hardhat compile # 编译合约
npx hardhat test    # 运行测试
```

## 相关章节

- [Solidity 开发](../Solidity%20开发/) — 合约语言语法与模式
- [Foundry 工具链](../Foundry%20工具链/) — 高性能替代方案，可与 Hardhat 共存
- [Truffle 框架](../Truffle%20框架/) — 已弃用的旧框架，附迁移指南
- [智能合约部署与升级成本](../../8.%20测试与部署/部署流程/智能合约部署与升级成本.md) — 部署条件与升级模式详解
