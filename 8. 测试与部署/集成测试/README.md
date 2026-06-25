# 集成测试与主网分叉

在可组合的 DeFi “乐高”生态中，没有任何协议是在孤岛运行的。本模块带你深入跨合约交互、多协议联动测试，以及最核心的主网分叉（Mainnet Forking）测试技术。

---

## 核心学习模块

> 📘 **[集成测试与主网分叉（Mainnet Forking）深度实战](./Integration_Testing_Guide.md)**
> 详细涵盖了以下核心板块：
> 1. **为什么弃用 Mock 模拟**：解析在本地 Mock Uniswap V3 或 Aave V3 极高昂的开发成本与严重失真风险。
> 2. **主网分叉底层机制**：解析惰性加载（Lazy Loading）原理与本地内存写入隔离。
> 3. **冒充大户（Impersonate Account）夺舍黑科技**：讲解如何在不获取任何私钥的情况下，利用 `vm.startPrank` 冒充主网巨鲸，并在本地划转其 DAI / USDC 进行流动性滑点测试。
> 4. **Aave V3 端到端集成测试实战**：提供一套完整的 Solidity 集成测试代码，真实分叉主网并冒充大户将 $1,000,000$ DAI 注入 Aave V3 生息池。

---

## 主网分叉防雷要点

- **锁定区块高度**：启动分叉时必须用 `--fork-block-number` 锁定一个确定的主网历史区块，避免主网出块变动导致测试丧失幂等性。
- **开启存储缓存**：在 `foundry.toml` 中配置 `no_storage_caching = false`，本地永久缓存已拉取的 Storage Slot 数据，将第二次及后续测试速度提升 20 倍。
