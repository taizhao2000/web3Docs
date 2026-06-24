# NFT 元数据与去中心化盲盒

NFT 的元数据决定了其图像、音视频以及在 OpenSea 等交易市场中展示的属性标签。本模块带你深入探讨去中心化存储、动态 NFT 以及抗审查、抗科学家提前窥探的公平盲盒揭示机制。

---

## 核心学习模块

我们在此专题中准备了高保真的实战指南与 Chainlink 集成合约：

> 📘 **[NFT 元数据管理：动态 NFT、IPFS 存储、VRF 盲盒机制与 Reveal 实战](./NFT_Metadata_Guide.md)**
> 深入探讨以下核心原理：
> 1. **元数据存储抉择**：链下中心化 HTTP、分布式 IPFS / Arweave、以及 100% 具备审查抗性的**链上 SVG Base64 矢量图硬编码编码硬渲染**技术。
> 2. **动态 NFT (dNFT) 架构**：结合 **Chainlink Automation（Keepers）** 与外部预言机，在现实事件发生时动态改变 NFT 的等级与外观属性。
> 3. **无管理员作弊的 Chainlink VRF 公平盲盒**：详解为什么传统延迟揭示会面临“科学家偷看”漏洞，并提供基于真随机数进行 **循环平移位移（Offset Scheme）** 的盲盒洗牌算法。
> 4. **生产级 RevealNFT 智能合约**：提供完整集成 OpenZeppelin ERC721 与 Chainlink VRF v2 的盲盒揭示与随机数位移洗牌代码实现。

---

## 核心要点速查

- **tokenURI**：ERC721 合约的核心只读函数，用于向外界返回带有 JSON 结构数据的 URI 地址。
- **Chainlink VRF (Verifiable Random Function)**：链上可验证随机数，提供零知识证明（ZKP）以确保生成的随机数绝不可预测且无法被项目方操纵。
- **On-chain SVG Base64**：通过将 SVG 代码和属性转化为 Base64 编码，实现不依赖任何 Web 服务器的“艺术品与合约共存亡”。
