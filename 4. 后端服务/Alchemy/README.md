# Alchemy 节点服务

Alchemy 是全栈区块链开发平台，提供超高性能的节点 RPC 服务以及独创的增强型 APIs。

## 详细指南

请参阅核心指南和实战：
- **[NaaS 节点服务综合指南](../Node_Services_Guide.md)**：深入剖析 Alchemy 的计算单元（CU）计费模式、Token/NFT 增强 API 怎么用，以及如何用 **Notify Webhooks** 实现账户充值秒级通知。

## 快速调用示例（Token Balance）

```typescript
import { Alchemy, Network } from "alchemy-sdk";

const config = {
  apiKey: "YOUR_ALCHEMY_KEY",
  network: Network.ETH_MAINNET,
};
const alchemy = new Alchemy(config);

// 快速获取某钱包持有的所有 ERC20 余额
const balances = await alchemy.core.getTokenBalances("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");
```

## 相关章节
- [Infura 节点服务](../Infura/) — 标准节点服务
- [Mempool 与 MEV 服务指南](../Mempool_and_MEV_Guide.md) — 合约高并发交易发送中的 Gas 优化
