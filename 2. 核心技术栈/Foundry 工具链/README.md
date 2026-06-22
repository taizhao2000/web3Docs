# Foundry 工具链

Foundry 是用 Rust 编写的极速以太坊应用开发工具链，原生支持 Solidity 测试和 Fuzz 测试。

## 学习路径

```
Foundry 工具链完整指南
         │
         ├─ Forge：编译、测试（含 fuzz/invariant）、部署
         ├─ Cast：链上交互命令行
         ├─ Anvil：本地测试节点
         └─ Chisel：Solidity REPL
```

> **前置知识**：建议先阅读 [Solidity 语法基础](../Solidity%20开发/Solidity%20语法基础.md) 和 [Solidity 进阶模式](../Solidity%20开发/Solidity%20进阶模式.md)。

## 文档索引

| 文档 | 难度 | 内容 |
|------|------|------|
| [Foundry 工具链完整指南](./Foundry%20工具链完整指南.md) | ⭐⭐ 进阶 | 安装（foundryup）、项目初始化、foundry.toml 配置详解、Forge 编译/依赖管理/测试（单元/Fuzz/不变式）/部署脚本（forge script）、Cast 链上交互实战、Anvil 本地节点、Chisel REPL、Cheatcode 速查表、与 Hardhat 共存配置 |

## 四大核心工具

| 工具 | 作用 | 类比 |
|------|------|------|
| **Forge** | 测试框架、构建管道、部署 | Hardhat 全部功能 |
| **Cast** | 链上交互命令行 | ethers.js CLI 版 |
| **Anvil** | 本地测试节点 | Ganache / Hardhat Node |
| **Chisel** | Solidity REPL | Node.js REPL |

## 快速开始

```bash
# 安装
curl -L https://foundry.paradigm.xyz | bash && foundryup

# 创建项目
forge init my-dapp && cd my-dapp

# 编译、测试、部署
forge build
forge test
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast
```

## Foundry vs Hardhat

| 维度 | Foundry | Hardhat |
|------|---------|---------|
| 语言 | Rust + Solidity | Node.js + TypeScript |
| 测试语言 | **Solidity（原生）** | JavaScript/TypeScript |
| 测试速度 | **极快（直接在 EVM 中）** | 中等（JS → RPC → 链） |
| Fuzz 测试 | **✅ 原生支持** | ⚠️ 需插件 |
| 不变式测试 | **✅ 原生支持** | ❌ 需插件 |
| 部署 | forge script | JS/TS 脚本 |
| 依赖管理 | Git submodules | npm |
| 推荐场景 | 合约核心逻辑测试 | 全栈项目、前端集成 |

> **两者可以共存！** 推荐用 Foundry 做合约测试，Hardhat 做部署和前端集成。

## 相关章节

- [Solidity 开发](../Solidity%20开发/) — 合约语言语法与模式
- [Hardhat 开发环境](../Hardhat%20开发环境/) — JS/TS 生态开发框架，可与 Foundry 共存
- [Truffle 框架](../Truffle%20框架/) — 已弃用的旧框架，附迁移指南
