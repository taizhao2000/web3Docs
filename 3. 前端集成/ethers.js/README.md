# ethers.js

ethers.js 是一个轻量级、安全的以太坊交互库，API 设计更现代，推荐用于新项目。

## 文档索引

| 文档 | 内容 |
|------|------|
| [ethers.js v6 完整指南](./ethers.js%20v6%20完整指南.md) | Provider/Signer/Contract 体系、合约交互（读/写/事件/过滤）、交易管理、格式化与单位转换、签名验证、React 集成 Hook、v5→v6 迁移对照表 |

## 快速开始

```bash
npm install ethers
```

```typescript
import { BrowserProvider, Contract, formatEther, parseEther } from "ethers";

const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
const contract = new Contract(address, abi, signer);

// 读取
const balance = await contract.balanceOf(userAddress);

// 写入
const tx = await contract.transfer(to, parseEther("1.0"));
await tx.wait();
```

## 相关章节

- [MetaMask 集成](../MetaMask%20集成/) — 浏览器钱包接入，ethers.js BrowserProvider 的底层
- [Web3.js 与现代前端栈](../Web3.js/) — viem vs ethers.js 对比，技术栈选择
