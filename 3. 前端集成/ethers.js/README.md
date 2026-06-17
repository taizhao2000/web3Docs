# ethers.js

## 概述

ethers.js 是一个轻量级、安全的以太坊交互库，API 设计更现代，推荐用于新项目。

## 安装

```bash
npm install ethers
```

## 基本用法（ethers v6）

```javascript
const { ethers } = require('ethers');

// 连接节点
const provider = new ethers.JsonRpcProvider('https://mainnet.infura.io/v3/YOUR_KEY');

// 查询余额
const balance = await provider.getBalance('0x...');

// 合约交互
const contract = new ethers.Contract(address, abi, provider);
const value = await contract.getValue();

// 签名者交互
const signer = await provider.getSigner();
const contractWithSigner = contract.connect(signer);
const tx = await contractWithSigner.setValue(42);
await tx.wait();
```

## Web3.js vs ethers.js

| 特性 | ethers.js | Web3.js |
|------|-----------|---------|
| 包大小 | ~100KB | ~300KB |
| API 风格 | 函数式 | 面向对象 |
| 类型支持 | TypeScript 原生 | 需类型声明 |
| 交易处理 | 更安全 | 较灵活 |
| 推荐度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
