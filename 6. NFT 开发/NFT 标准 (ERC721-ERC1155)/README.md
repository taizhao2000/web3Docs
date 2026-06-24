# NFT 核心标准与 Gas 优化

本模块详细探讨了非同质化代币（NFT）的标准演进、游戏代币（SFT）设计以及在大规模发售中压榨以太坊 Gas 费极限的底层开发技术。

---

## 核心学习模块

我们在此专题中准备了极具深度的底层对比与生产级实战指南：

> 📘 **[NFT 标准深度解析：ERC721、ERC1155 与 ERC721A 极低 Gas 优化指南](./NFT_Standards_Guide.md)**
> 深入探索以下核心技术栈：
> 1. **设计哲学与存储模型对比**：解析 ERC-721 的所有权 Mapping（`tokenId => address`）与 ERC-1155 的嵌套 Mapping（`tokenId => Holder => Balance`）的存储及 Gas 差异。
> 2. **ERC-721A 批量铸造优化黑科技**：深度解密 Azuki 首创的**所有权延迟初始化（Lazy Initialization）**、**余额合并计算**以及 **Bit-packing（位运算紧凑打包）** 底层机制。
> 3. **游戏资产租赁（ERC-4907）**：编写带自动到期（无需链上主动清算交易）的使用权与所有权分离协议。
> 4. **链上通用版税（ERC-2981）**：实现符合跨交易平台自动版税调用的标准接口合约。

---

## 核心名词速览

- **SFT (Semi-Fungible Token)**：半同质化代币，通过单个 ID 承载多个副本余额，是 GameFi 道具设计的黄金标准。
- **Lazy Initialization**：延迟初始化，把批量 Mint 时的写存储消耗转移到未来发生的 Transfer 交易中，在 Mint 时实现极致低 Gas。
- **SupportsInterface (ERC-165)**：合约多接口自省支持判定，是检测 NFT 是否集成租赁或版税标准的核心链上机制。
