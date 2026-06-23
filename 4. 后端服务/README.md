# 4. 后端服务

Web3 后端服务构成了 DApp 访问区块链、监听链上数据流、防范交易池攻击（MEV）的硬核底座。

## 学习路径

```
       NaaS 节点服务综合指南
                 │
      ┌──────────┴──────────┐
      ▼                     ▼
Mempool 监控、Gas预测   自建以太坊节点
与 MEV 实战指南         深度运维指南
```

> **前置知识**：建议先阅读 [3. 前端集成](../3.%20前端集成/)，理解前端/后端是如何通过 Provider 与以太坊 RPC 节点进行基本读写交互的。

## 文档索引

| 文档 | 难度 | 核心内容 |
|------|------|---------|
| [NaaS 节点服务综合指南](./Node_Services_Guide.md) | ⭐ 入门 | RPC 底层交互原理、全节点与归档节点分类、Infura/Alchemy/QuickNode 平台对比与 CU 计费拆坑、WebSocket 实时监听与断线重连、Alchemy 增强型 API（Token API/Notify Webhooks）、FallbackProvider 节点热备冗余设计 |
| [Mempool 与 MEV 服务指南](./Mempool_and_MEV_Guide.md) | ⭐⭐ 进阶 | 内存池 Mempool 核心机制（Pending排队/Nonce覆盖与替换）、Blocknative Gas预测、MEV 最大可提取价值与夹子攻击（Sandwich Attack）原理解密、Flashbots Protect 私有通道避坑与发单实战 |
| [自建以太坊节点深度运维指南](./Self_Hosted_Node_Guide.md) | ⭐⭐⭐ 深入 | 执行层客户端（Geth）+ 共识层客户端（Lighthouse）双引擎架构、Docker Compose 极速 Checkpoint 部署、硬件指标与网络拓扑、Geth 离线磁盘修剪（Pruning）、Nginx RPC 安全加固、Prometheus+Grafana看板与指标红线 |

## 生态与服务商定位

| 服务商 | 核心定位 | 推荐场景 |
|:---|:---|:---|
| **Infura** | 稳定、标准以太坊 API 访问 | 传统 DApp，对标准多链节点有高可用性要求 |
| **Alchemy** | 增强 API（NFT/Token/Trace）、Webhooks 推送 | 资产看板、充值自动入账监控、需要免配置查询 NFT |
| **Blocknative** | 实时 Mempool 分析与 Gas 深度预测 | 跨链桥、高频套利、清算机器人、抢先交易防夹 |
| **Flashbots** | 私有 RPC 交易中继，防范 MEV 套利 | 大额兑换、DEX 操作防夹、高价值资产转移安全信道 |
| **自建全节点** | 纯自主控制、免速率限制、本地归档 Trace | 交易所钱包、区块链浏览器后台、对数据隐私有极高要求 |

---

## 各子模块说明

- [Infura](./Infura/) - ConsenSys 旗下稳定的 NaaS 平台。
- [Alchemy](./Alchemy/) - 全栈增强型 Web3 节点开发套件。
- [BlockNative](./BlockNative/) - 专注于交易生命周期与 Mempool 数据。
- [自建节点](./自建节点/) - 双客户端、Docker 自动化构建方案。
