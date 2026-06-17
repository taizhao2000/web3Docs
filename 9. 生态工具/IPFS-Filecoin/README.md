# IPFS/Filecoin

## IPFS 概述

IPFS（InterPlanetary File System）是一种点对点的分布式文件系统，使用内容寻址存储和检索数据。

## 核心概念

- **CID（Content Identifier）**：内容的唯一标识符
- **DAG**：有向无环图，数据组织方式
- **Pinning**：固定数据，防止被垃圾回收
- **Gateway**：HTTP 网关，通过 URL 访问 IPFS 内容

## 在 Web3 中的应用

| 用途 | 说明 |
|------|------|
| NFT 元数据 | 存储 NFT 的属性和图片 |
| DApp 前端 | 去中心化托管前端代码 |
| 数据存储 | 存储大文件，链上存哈希 |
| 内容分发 | 抗审查的内容发布 |

## 常用工具

| 工具 | 说明 |
|------|------|
| Pinata | IPFS Pinning 服务 |
| nft.storage | 专为 NFT 设计的 IPFS 存储 |
| web3.storage | IPFS + Filecoin 存储 |
| ipfs-http-client | JavaScript IPFS 客户端 |

## Filecoin 概述

Filecoin 是基于 IPFS 的去中心化存储网络，通过经济激励机制确保数据的持久存储。

| 特性 | IPFS | Filecoin |
|------|------|----------|
| 存储 | 临时 | 持久（付费） |
| 激励 | 无 | FIL 代币 |
| 证明 | 无 | 复制证明+时空证明 |
| 保证 | 无 | 经济担保 |
