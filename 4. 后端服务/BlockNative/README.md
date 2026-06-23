# BlockNative 与 Mempool 服务

Blocknative 专注于提供区块链交易全生命周期追踪、精准的实时 Gas 估算，以及公开交易池（Mempool）监控数据。

## 详细指南

请参阅核心指南：
- **[Mempool 监控、Gas 预测与 MEV 实战指南](../Mempool_and_MEV_Guide.md)**：深入解释 Mempool Pending 交易积压、EIP-1559 费率调节、夹子机器人抢跑原理，以及如何接入 **Flashbots Protect** 获得私有防夹打包保护。

## 特色能力
1. **Gas Platform**：比节点提供的标准 RPC 更高频和精准的 Gas 价预测，置信度达 99%。
2. **Mempool Explorer**：捕获尚未进入区块的 Pending 交易和 Internal Call 细节。
3. **Web3-onboard**：多钱包连接库，自动适配移动端与桌面端钱包。

## 相关章节
- [NaaS 节点服务综合指南](../Node_Services_Guide.md) — 节点网络基础
- [自建以太坊节点深度运维指南](../Self_Hosted_Node_Guide.md) — 监控和管理本地节点的 txpool 队列
