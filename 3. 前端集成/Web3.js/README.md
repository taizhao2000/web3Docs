# Web3.js

## 概述

Web3.js 是以太坊生态最经典的 JavaScript 库，用于与以太坊节点交互。

## 安装

```bash
npm install web3
```

## 基本用法

```javascript
const Web3 = require('web3');

// 连接节点
const web3 = new Web3('https://mainnet.infura.io/v3/YOUR_KEY');

// 查询余额
const balance = await web3.eth.getBalance('0x...');

// 调用合约
const contract = new web3.eth.Contract(abi, address);
const result = await contract.methods.getValue().call();

// 发送交易
await contract.methods.setValue(42).send({ from: userAddress });
```

## 核心模块

| 模块 | 功能 |
|------|------|
| web3.eth | 以太坊区块链交互 |
| web3.utils | 工具函数（编码、单位转换等） |
| web3.shh | Whisper 协议（已弃用） |
| web3.bzz | Swarm 存储（已弃用） |
