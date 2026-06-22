# 3. 前端集成

Web3 前端开发的核心库和钱包集成方案。

## 学习路径

```
MetaMask 集成 → ethers.js v6 → WalletConnect v2
      │              │               │
      │              │               └─ 移动端钱包连接
      │              └─ DApp 与合约交互的核心库
      └─ 浏览器钱包接入（EIP-6963/EIP-1193）
      
Web3.js 与现代前端栈（wagmi + viem）
      │
      └─ 理解旧项目 + 迁移到现代前端栈
```

> **前置知识**：建议先阅读 [Solidity 语法基础](../2.%20核心技术栈/Solidity%20开发/Solidity%20语法基础.md)，理解合约的函数和事件。

## 文档索引

| 文档 | 难度 | 内容 |
|------|------|------|
| [MetaMask 集成完整指南](./MetaMask%20集成/MetaMask%20集成完整指南.md) | ⭐ 入门 | EIP-6963 钱包发现、EIP-1193 标准、连接/断开/账户切换/网络切换、添加自定义链、签名（personal_sign/EIP-712）、交易发送、错误码处理、完整 React Hook |
| [ethers.js v6 完整指南](./ethers.js/ethers.js%20v6%20完整指南.md) | ⭐⭐ 进阶 | Provider/Signer/Contract 体系、合约交互（读/写/事件/过滤）、交易管理、格式化与单位转换、签名验证、React 集成 Hook、v5→v6 迁移对照表 |
| [WalletConnect v2 完整指南](./WalletConnect/WalletConnect%20v2%20完整指南.md) | ⭐⭐ 进阶 | v2 架构（Relay/Pairing/Session）、DApp 端集成（SignClient 初始化/连接/签名/断开）、React Hook、与 wagmi 集成 |
| [Web3.js 与现代前端栈指南](./Web3.js/Web3.js%20与现代前端栈指南.md) | ⭐⭐ 进阶 | Web3.js v4 用法、viem vs ethers.js 对比、wagmi + viem + RainbowKit 实战、技术栈选择决策树、Web3.js→ethers.js 迁移对照表 |

## 技术栈推荐

| 场景 | 推荐方案 |
|------|---------|
| 新 React DApp | **wagmi + viem + RainbowKit** |
| 新非 React 项目 | **viem** 或 **ethers.js v6** |
| 维护旧 ethers.js 项目 | 保持 ethers.js v6 |
| 维护旧 Web3.js 项目 | 迁移到 ethers.js 或 viem |

## 相关章节

- [Solidity 开发](../2.%20核心技术栈/Solidity%20开发/) — 合约语言语法，ethers.js 的 ABI 对应合约接口
- [Hardhat 开发环境](../2.%20核心技术栈/Hardhat%20开发环境/) — 编译产出的 ABI 和字节码用于前端集成
- [后端服务](../4.%20后端服务/) — RPC 节点服务（Infura/Alchemy）
