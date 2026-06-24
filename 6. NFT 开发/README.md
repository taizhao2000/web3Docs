# 6. NFT 开发 (Non-Fungible Tokens)

非同质化代币（NFT）是 Web3 生态中代表数字资产所有权、链上唯一身份凭证和半同质化游戏道具的底层技术支撑。

本章节系统性地整理了 NFT 开发的核心技术树，涵盖代币标准、元数据治理以及去中心化交易平台的完整实现。

---

## 知识子树导航

| 核心专题 | 主要内容简介 | 核心文档链接 |
| :--- | :--- | :--- |
| **NFT 核心与扩展标准** | 深入对比 ERC-721 与 ERC-1155。深度解密 Azuki ERC-721A 批量铸造 Gas 优化机制。包含租赁（ERC-4907）与版税（ERC-2981）生产级智能合约。 | [NFT 核心标准深度指南](./NFT%20标准%20(ERC721-ERC1155)/NFT_Standards_Guide.md) |
| **元数据与盲盒开发** | 链上 SVG 渲染 vs 链下存储。动态 NFT 与 Chainlink Automation 集成。利用 Chainlink VRF（可验证随机数）设计公平防抢跑的盲盒平移洗牌机制。 | [NFT 元数据与盲盒深度指南](./NFT%20元数据管理/NFT_Metadata_Guide.md) |
| **交易市场构建** | 传统托管模式与现代“授权划转”模式对比。LooksRare/Seaport 链下 EIP-712 签名无 Gas 挂单原理解密。编写带有版税分配、平台分成、防重入攻击的交易市场合约。 | [NFT 交易市场深度指南](./NFT%20市场开发/NFT_Marketplace_Guide.md) |

---

## 核心开发路线图

1. **掌握 Gas 优化的艺术**：NFT 是以太坊交易中最活跃的板块之一。在开发大供应量（如 10k PFP）项目时，必须掌握 **ERC-721A** 的存储延迟初始化（Lazy Initialization）和 Slot 紧凑打包（Bit-packing）位运算。
2. **保障真随机公平性**：拒绝使用传统的 `block.timestamp` 或 `block.difficulty` 作为盲盒随机数源（容易被矿工/科学家操纵）。涉及稀有度洗牌，必须通过 **Chainlink VRF** 安全获取链上可验证随机数。
3. **理解原子交换**：深入理解 NFT 交易市场的挂单和划扣机制，掌握链下 EIP-712 签名的格式和验签机制，设计安全的 Checks-Effects-Interactions 数据流防范重入黑客攻击。
