# NFT 市场开发

## 概述

NFT 市场是用户铸造、展示、买卖 NFT 的平台。

## 核心功能

1. **挂牌（Listing）**：卖家设置价格出售 NFT
2. **出价（Bidding）**：买家对 NFT 出价
3. **成交（Sale）**：匹配买卖并完成交易
4. **拍卖（Auction）**：英式/荷兰式拍卖

## 交易模式

| 模式 | 说明 | 代表 |
|------|------|------|
| 直接出售 | 固定价格 | OpenSea |
| 英式拍卖 | 逐步加价 | Foundation |
| 荷兰式拍卖 | 价格递减 | Art Blocks |
| 捆绑出售 | 多 NFT 打包 | OpenSea Bundle |

## 核心合约设计

```solidity
contract NFTMarketplace {
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool active;
    }
    
    mapping(uint256 => Listing) public listings;
    uint256 public listingCount;
    
    function listNFT(address nftContract, uint256 tokenId, uint256 price) external {
        IERC721(nftContract).transferFrom(msg.sender, address(this), tokenId);
        listings[listingCount] = Listing(msg.sender, nftContract, tokenId, price, true);
        listingCount++;
    }
    
    function buyNFT(uint256 listingId) external payable {
        Listing storage listing = listings[listingId];
        require(listing.active && msg.value >= listing.price);
        listing.active = false;
        IERC721(listing.nftContract).transferFrom(address(this), msg.sender, listing.tokenId);
        payable(listing.seller).transfer(msg.value);
    }
}
```

## 版税实现

使用 ERC-2981 标准或 Seaport 协议实现创作者版税。
