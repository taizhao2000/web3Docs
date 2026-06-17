# Hardhat 开发环境

## 概述

Hardhat 是专业的以太坊开发环境，用于编译、部署、测试和调试智能合约。

## 核心功能

- **编译**：自动编译 Solidity 合约
- **测试**：基于 Mocha 的测试框架
- **部署**：脚本化部署流程
- **调试**：Solidity 堆栈追踪和 console.log
- **网络**：内置 Hardhat 网络（本地测试链）

## 项目结构

```
my-project/
├── contracts/        # 智能合约
├── scripts/          # 部署脚本
├── test/             # 测试文件
├── hardhat.config.ts # 配置文件
└── artifacts/        # 编译产物（自动生成）
```

## 常用命令

```bash
npx hardhat compile      # 编译合约
npx hardhat test         # 运行测试
npx hardhat run scripts/deploy.ts  # 运行部署脚本
npx hardhat node         # 启动本地节点
npx hardhat console      # 交互式控制台
```

## 常用插件

| 插件 | 功能 |
|------|------|
| @nomicfoundation/hardhat-toolbox | 工具集合 |
| hardhat-gas-reporter | Gas 消耗报告 |
| solidity-coverage | 代码覆盖率 |
| @openzeppelin/hardhat-upgrades | 代理升级支持 |
