# BlockNative

## 概述

BlockNative 提供区块链交易监控、Gas 估算和 Mempool 数据服务。

## 核心产品

- **Gas Platform**：实时 Gas 价格估算
- **Mempool Explorer**：交易池监控
- **On-Chain Monitor**：链上事件监控通知
- **Web3-onboard**：多钱包连接框架

## Gas 估算

```javascript
import { BlocknativeSdk } from 'bnc-sdk';

const sdk = new BlocknativeSdk({
  dappId: 'YOUR_API_KEY',
  networkId: 1,
});

// 实时 Gas 价格
const gasPrice = await sdk.gas.getGasPrice();
```

## Web3-Onboard

多钱包统一接入框架，支持 MetaMask、WalletConnect、Coinbase 等 20+ 钱包。
