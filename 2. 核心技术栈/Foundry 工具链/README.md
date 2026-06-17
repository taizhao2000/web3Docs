# Foundry 工具链

## 概述

Foundry 是用 Rust 编写的极速以太坊应用开发工具链，包含 Forge、Cast、Anvil 和 Chisel 四个核心工具。

## 核心工具

| 工具 | 功能 |
|------|------|
| **Forge** | 测试框架、构建管道 |
| **Cast** | 链上交互命令行工具 |
| **Anvil** | 本地测试节点 |
| **Chisel** | Solidity REPL |

## 项目结构

```
my-project/
├── src/           # 智能合约
├── test/          # 测试文件（Solidity）
├── script/        # 部署脚本
├── lib/           # 依赖库
└── foundry.toml   # 配置文件
```

## 常用命令

```bash
forge init              # 初始化项目
forge build             # 编译合约
forge test              # 运行测试
forge script            # 运行部署脚本
forge install           # 安装依赖
cast call               # 读取合约数据
cast send               # 发送交易
anvil                   # 启动本地节点
```

## 与 Hardhat 对比

| 特性 | Foundry | Hardhat |
|------|---------|---------|
| 语言 | Rust | Node.js |
| 测试语言 | Solidity | JavaScript/TypeScript |
| 编译速度 | 极快 | 较慢 |
| 生态成熟度 | 快速增长 | 成熟 |
