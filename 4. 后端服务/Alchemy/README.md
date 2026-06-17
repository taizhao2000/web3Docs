# Alchemy

## 概述

Alchemy 是全栈区块链开发平台，提供节点服务、NFT API、通知等高级功能。

## 核心功能

- **增强型 API**：超越标准 RPC 的高级接口
- **NFT API**：NFT 元数据、持有者查询
- **Transact**：交易发送和 Gas 优化
- **Notify**：链上事件通知
- **Crunch**：归档数据分析

## 特色 API

```javascript
// 获取 NFT 持有者
const owners = await alchemy.nft.getOwnersForContract(address);

// 获取代币余额（含价格）
const balances = await alchemy.core.getTokenBalances(address);

// Gas 优化交易
const tx = await alchemy.transact.sendPrivateTransaction(signedTx);
```

## 与 Infura 对比

| 特性 | Alchemy | Infura |
|------|---------|--------|
| 标准 RPC | ✅ | ✅ |
| 增强 API | ✅ | ❌ |
| NFT API | ✅ | ❌ |
| 交易加速 | ✅ | ❌ |
| 免费额度 | 300M CU/月 | 100K 请求/天 |
