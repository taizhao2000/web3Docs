# 部署流程

## 概述

智能合约部署是从开发到生产的关键步骤，需要严格遵循安全流程。

## 📖 核心文档

| 文档 | 说明 |
|------|------|
| [智能合约部署与升级成本](./智能合约部署与升级成本.md) | 部署≠执行澄清、EVM硬性限制、部署/运行/升级成本详解、代理模式对比、存储布局冲突 |
| 部署脚本示例 | 见下方 Hardhat / Foundry 部署脚本 |

## 部署阶段

```
本地测试 → 测试网部署 → 审计 → 主网部署 → 监控
```

## 部署前检查

- [ ] 合约通过所有测试
- [ ] 测试覆盖率达标
- [ ] 代码已审计
- [ ] 部署脚本已测试
- [ ] 多签钱包已配置
- [ ] 紧急暂停功能就绪
- [ ] 监控系统已部署

## Hardhat 部署脚本

```javascript
const { ethers } = require('hardhat');

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log('部署账户:', deployer.address);
  console.log('账户余额:', (await deployer.getBalance()).toString());
  
  const Token = await ethers.getContractFactory('Token');
  const token = await Token.deploy('MyToken', 'MTK', 1000000);
  await token.deployed();
  
  console.log('合约地址:', token.address);
  console.log('部署交易:', token.deployTransaction.hash);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

## Foundry 部署脚本

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";
import "../src/Token.sol";

contract DeployToken is Script {
    function run() external {
        vm.startBroadcast();
        
        Token token = new Token("MyToken", "MTK", 1000000);
        console.log("Token deployed at:", address(token));
        
        vm.stopBroadcast();
    }
}
```

```bash
forge script script/DeployToken.s.sol --rpc-url $RPC_URL --broadcast
```

## 多链部署

| 网络 | Chain ID | RPC |
|------|----------|-----|
| Ethereum Mainnet | 1 | https://mainnet.infura.io |
| Sepolia Testnet | 11155111 | https://rpc.sepolia.org |
| Polygon | 137 | https://polygon-rpc.com |
| Arbitrum | 42161 | https://arb1.arbitrum.io/rpc |
| Optimism | 10 | https://mainnet.optimism.io |
| BSC | 56 | https://bsc-dataseed.binance.org |
