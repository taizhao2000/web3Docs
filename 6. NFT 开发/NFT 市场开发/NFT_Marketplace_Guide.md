# NFT 交易市场合约开发与 Seaport 架构剖析

> NFT 交易市场（Marketplace）是 NFT 资产流动的核心枢纽。本指南将剖析传统链上托管模式与现代链下挂单签名（EIP-712/Seaport）模式的架构演进，并提供一个集成了 ERC-2981 链上版税分成、平台佣金结算和防重入安全的生产级链上 NFT 交易市场合约。

---

## 1. NFT 交易市场两大架构演进

NFT 交易市场的主要任务是**实现“一手交钱，一手交货”（原子化交换：代币与 NFT 的无信托兑换）**。技术架构经历了两大阶段演进：

### 1.1 阶段一：链上托管模式（早期低效方案）
- **流程**：卖家挂单时，调用 `safeTransferFrom` 将 NFT 转移到交易市场合约中锁存托管（Escrow）；买家付钱时，合约将钱付给卖家，并将 NFT 从托管合约转给买家。
- **弊端**：
  1. **极度耗 Gas**：挂单需要一次链上 Transfer 交易（$60,000+$ Gas），下架退回又需要一次 Transfer 交易。
  2. **丧失使用权**：在托管期间，用户的 NFT 不在自己的钱包里，无法享受社区空投（Airdrop）、无法展示、无法作为社交头像（PFP）、无法质押（Staking）。

### 1.2 阶段二：授权撮合模式（现代黄金标准）
- **流程**：卖家挂单时，只需在本地单次或全局调用 `setApprovalForAll(Marketplace, true)`，将 NFT 转移权单次授权给市场，**NFT 依然保留在卖家钱包中**。当真正成交的一瞬间，买家调用市场合约，合约利用 `transferFrom` 将 NFT 强行扣划并转移给买家。
- **优点**：挂单和取消挂单都可以实现**链下无 Gas 签名化管理**。

---

## 2. 现代交易市场的黑科技：EIP-712 链下签名挂单（LooksRare/Seaport）

为了免除卖家挂单和频繁改价时的 Gas 费成本，现代 NFT 市场（OpenSea 的 Seaport 协议、LooksRare）普遍引入了 **EIP-712 类型化签名挂单机制**。

### 2.1 链下签名挂单极速业务流

```
 [1. 链下挂单] ──────────────────> 卖家在本地浏览器上（MetaMask）对 EIP-712 挂单订单体进行私钥签名。
                                  挂单信息包含：[卖家, NFT地址, tokenId, 售价 1 ETH, 到期时间]。
                                  【无需链上发交易，0 Gas！】
                                        │
                                        ▼
 [2. 上架数据库] ───────────────> 签名与订单数据免费发送并存储到 OpenSea/LooksRare 后端数据库。
                                        │
                                        ▼
 [3. 链上购买] ──────────────────> 买家浏览网页，看中了此挂单，发起链上购买交易。
                                  网页将订单数据与卖家的签名打包传给【买家交易参数】。
                                        │
                                        ▼
 [4. 智能合约校验与原子交换] ─────> 买家调用 `purchase(Order, Signature)` 并附带 1 ETH。
                                  市场合约利用 `ecrecover` 校验该订单确实由卖家签名且未到期。
                                  合约执行代扣：从卖家钱包扣划 NFT 给买家，将买家的 1 ETH 分给卖家与国库。
```

---

## 3. 生产级 `NFTMarketplace` 合约实现（含版税与平台分成）

以下是一个符合生产环境要求的、集成了 **ERC-2981 链上版税标准** 和 **平台手续费分成** 的授权撮合交易市场合约。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/common/ERC2981/ERC2981.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

interface IERC2981Royalties {
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external
        view
        returns (address receiver, uint256 royaltyAmount);
}

contract NFTMarketplace is ReentrancyGuard, Ownable {
    // 1. 结构体与事件定义
    struct Listing {
        address seller;     // 卖家地址
        uint256 price;      // 挂单价格 (wei)
        bool active;        // 是否在售
    }

    // 平台手续费率 (万分位基数，如 2.5% = 250)
    uint256 public platformFeeBps = 250; 
    address public platformFeeRecipient;  // 平台收费国库地址

    uint256 private constant BPS_DENOMINATOR = 10000;

    // NFT 合约地址 => (Token ID => 挂单详情)
    mapping(address => mapping(uint256 => Listing)) public listings;

    event ItemListed(address indexed seller, address indexed nftAddress, uint256 indexed tokenId, uint256 price);
    event ItemCanceled(address indexed seller, address indexed nftAddress, uint256 indexed tokenId);
    event ItemSold(address indexed buyer, address indexed nftAddress, uint256 indexed tokenId, uint256 price, uint256 royaltyPaid);

    constructor(address _feeRecipient) {
        require(_feeRecipient != address(0), "Invalid recipient");
        platformFeeRecipient = _feeRecipient;
    }

    /**
     * @notice 挂单 NFT (采用授权转移模式)
     */
    function listFormat(
        address nftAddress,
        uint256 tokenId,
        uint256 price
    ) external {
        require(price > 0, "Price must be greater than zero");
        
        IERC721 nft = IERC721(nftAddress);
        // 安全检查 1：挂单人必须是 NFT 的真实所有者
        require(nft.ownerOf(tokenId) == msg.sender, "Not the owner");
        // 安全检查 2：市场合约必须已获得该 NFT 的划转授权
        require(
            nft.isApprovedForAll(msg.sender, address(this)) || nft.getApproved(tokenId) == address(this),
            "Marketplace has no transfer approval"
        );

        listings[nftAddress][tokenId] = Listing({
            seller: msg.sender,
            price: price,
            active: true
        });

        emit ItemListed(msg.sender, nftAddress, tokenId, price);
    }

    /**
     * @notice 取消挂单
     */
    function cancelFormat(address nftAddress, uint256 tokenId) external nonReentrant {
        Listing memory listedItem = listings[nftAddress][tokenId];
        require(listedItem.active, "Item not listed");
        require(listedItem.seller == msg.sender, "Not the seller");

        delete listings[nftAddress][tokenId];

        emit ItemCanceled(msg.sender, nftAddress, tokenId);
    }

    /**
     * @notice 购买 NFT 并自动分账（含：1. 扣划平台佣金、2. 提取 ERC-2981 链上版税、3. 支付卖家余款）
     */
    function purchaseFormat(address nftAddress, uint256 tokenId) external payable nonReentrant {
        Listing memory listedItem = listings[nftAddress][tokenId];
        // 安全校验
        require(listedItem.active, "Item is not listed for sale");
        require(msg.value >= listedItem.price, "Insufficient payment");

        // 1. 锁死订单状态，防重入清算
        listings[nftAddress][tokenId].active = false;

        uint256 totalPrice = listedItem.price;
        uint256 platformFee = (totalPrice * platformFeeBps) / BPS_DENOMINATOR;
        uint256 royaltyAmount = 0;
        address royaltyReceiver;

        // 2. 动态查询与提取 ERC-2981 链上版税
        try IERC2981Royalties(nftAddress).royaltyInfo(tokenId, totalPrice) returns (address receiver, uint256 amount) {
            if (receiver != address(0) && amount > 0) {
                royaltyReceiver = receiver;
                royaltyAmount = amount;
            }
        } catch {
            // 若 NFT 未集成 ERC-2981 接口，则静默跳过，版税为 0
        }

        // 3. 安全计算与资金分配
        uint256 sellerProceeds = totalPrice - platformFee - royaltyAmount;
        require(sellerProceeds + platformFee + royaltyAmount == totalPrice, "Math inconsistency");

        // (a) 分账：划转平台手续费
        if (platformFee > 0) {
            (bool successFee, ) = payable(platformFeeRecipient).call{value: platformFee}("");
            require(successFee, "Platform fee payout failed");
        }

        // (b) 分账：划转原创作者版税
        if (royaltyAmount > 0) {
            (bool successRoyalty, ) = payable(royaltyReceiver).call{value: royaltyAmount}("");
            require(successRoyalty, "Royalty payout failed");
        }

        // (c) 分账：划转卖家纯收益
        (bool successSeller, ) = payable(listedItem.seller).call{value: sellerProceeds}("");
        require(successSeller, "Seller payout failed");

        // 4. 清除并释放 Listings 内存
        delete listings[nftAddress][tokenId];

        // 5. 执行代币扣划转账（将 NFT 从卖家钱包直划至买家钱包）
        IERC721(nftAddress).safeTransferFrom(listedItem.seller, msg.sender, tokenId);

        // 6. 若买家超额付钱（多付了 ETH），全额原路退回
        uint256 refund = msg.value - totalPrice;
        if (refund > 0) {
            (bool successRefund, ) = payable(msg.sender).call{value: refund}("");
            require(successRefund, "Refund failed");
        }

        emit ItemSold(msg.sender, nftAddress, tokenId, totalPrice, royaltyAmount);
    }

    /**
     * @notice 管理员更新平台收费标准
     */
    function updatePlatformFeeBps(uint256 _newBps) external onlyOwner {
        require(_newBps <= 1000, "Fee limit exceeded (Max 10%)"); // 限制最高收费为 10%
        platformFeeBps = _newBps;
    }

    /**
     * @notice 管理员更新国库收取地址
     */
    function updatePlatformFeeRecipient(address _newRecipient) external onlyOwner {
        require(_newRecipient != address(0), "Invalid address");
        platformFeeRecipient = _newRecipient;
    }
}
```

---

## 4. 交易市场核心漏洞与安全加固指南

在交易市场的开发和审计实践中，有 3 类黑客攻击最为频繁，必须采取针对性的防御手段：

### 4.1 挂单签名重放攻击（Signature Replay）
- **漏洞风险**：在 EIP-712 链下签名挂单中，如果签名订单体中没有加入 **`chainId`** 和 **`marketplaceContractAddress`**（也就是未定义 `DOMAIN_SEPARATOR`），攻击者可以将用户在以太坊主网上生成的优惠挂单签名，拿到以太坊 L2（如 Arbitrum/Polygon）或分叉链上去重新执行购买（重放），低价盗走用户的 NFT。
- **加固方案**：必须遵循严格的 EIP-712 签名标准，生成具有强链 ID 隔离和强合约地址隔离的 `DOMAIN_SEPARATOR`，确保一签一用，跨链失效。

### 4.2 重入安全（Reentrancy）
- **漏洞风险**：在 `purchase` 分账结算时，如果合约先向卖家发送以太币，由于外部调用（`call{value: ...}`）会触发卖家账户的回调函数（`fallback` / `receive`），黑客可以在卖家回调中重入调用市场的 `purchase` 函数。此时，因为挂单状态（`Listing.active`）还来不及更新，会导致重复购买或双花退款。
- **加固方案**：
  1. 引入 OpenZeppelin 的 `nonReentrant` 保护锁。
  2. 严格遵守 **Checks-Effects-Interactions（检查-效果-交互）** 模式。在向任何外部账户划转资金前，**必须先将挂单的 `active` 状态置为 `false` 并清除挂单账目**。

### 4.3 零价格或精度下溢截断溢出
- **漏洞风险**：在计算平台分成和版税比例时，如果代币价格极小（例如只有几个 wei），`(Price * FeeBps) / 10000` 会因为整数除法精度截断而产生 `0` 值。如果算术中包含多次减法（如 `Price - PlatformFee - Royalty`），需做好安全防溢出判定，防止结果下溢（Underflow）导致卖家获得超大额代币。
- **加固方案**：采用 Solidity 0.8.x 默认的溢出检测，或者使用 SafeMath/Require 进行精确校验。
