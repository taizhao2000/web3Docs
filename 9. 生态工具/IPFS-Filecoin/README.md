# IPFS-Filecoin 去中心化存储

本模块详细探讨了 Web3 大体积文件（如 NFT 多媒体图片、前端静态网站包、PDF 合约报告）的去中心化存储技术，解决了传统中心化 HTTP 存储易发生 404 丢失、单点宕机及随意被篡改的风险。

---

## 核心学习模块

我们在此专题中准备了高保真的实战指南与生产级 API 上传代码：

> 📘 **[去中心化存储 IPFS-Filecoin 与 Pinata 实战](./IPFS_Filecoin_Guide.md)**
> 深入探索和掌握以下去中心化存储技术：
> 1. **内容寻址（Content Addressing）与 CID**：理解 IPFS 唯一哈希寻址如何从根本上防范资产内容被暗中修改。
> 2. **IPFS、Filecoin 与 Arweave 对比**：辨析传输协议、FIL 时空复制证明经济激励层、以及 Arweave 一次付费永久存储（Permaweb）的共识。
> 3. **Pinning 钉住服务与 Pinata**：解析本地节点断网危机，详解云端常备节点的重大价值。
> 4. **生产级上传脚本实战**：提供一个完整的 Node.js/TypeScript 脚本，演示如何将本地多媒体上传、自动在内存中动态组装 OpenSea 规范的 JSON、并将 JSON 整体上传生成最终 baseURI 写入合约。

---

## 核心开发红线

- **不要硬编码 JWT**：在上传脚本中，Pinata API Token / JWT 必须通过 `process.env.PINATA_JWT` 从环境变量中安全读取，绝对不能硬编码提交到公共 GitHub 仓库上，否则必然面临配额被盗刷。
- **协议头必须是 `ipfs://`**：拼装 JSON 元数据时，代币媒体 URL 必须声明为 `ipfs://Qm...`，由 OpenSea 或 MetaMask 钱包自主就近路由解析。
