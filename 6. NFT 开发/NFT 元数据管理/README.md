# NFT 元数据管理

## 概述

NFT 元数据是描述 NFT 属性的 JSON 数据，通常存储在链下（IPFS 或中心化服务器）。

## 元数据标准（ERC-721）

```json
{
  "name": "My NFT #1",
  "description": "A unique digital artwork",
  "image": "ipfs://Qm.../image.png",
  "attributes": [
    {
      "trait_type": "Background",
      "value": "Blue"
    },
    {
      "trait_type": "Rarity",
      "value": "Legendary"
    },
    {
      "display_type": "number",
      "trait_type": "Generation",
      "value": 1
    }
  ]
}
```

## 存储方案

| 方案 | 优点 | 缺点 |
|------|------|------|
| IPFS | 去中心化、内容寻址 | 需要_pin_服务 |
| Arweave | 永久存储 | 成本较高 |
| 中心化服务器 | 快速、灵活 | 单点故障风险 |
| 链上存储 | 完全去中心化 | Gas 成本极高 |

## 最佳实践

1. **使用 IPFS + Pinning 服务**（如 Pinata、nft.storage）
2. **在合约中使用 `ipfs://` URI** 而非 HTTP 网关
3. **图像和元数据都存储在 IPFS**
4. **使用确定性 CID** 确保可验证
5. **预留元数据冻结功能**
