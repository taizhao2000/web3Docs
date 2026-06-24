# NFT 标准深度解析：ERC721、ERC1155 与 ERC721A 极低 Gas 优化指南

> 非同质化代币（NFT）不仅代表艺术品和收藏品，更是 Web3 资产所有权、链上身份凭证以及游戏内半同质化资产（SFT）的核心技术载体。本指南将深度对比 ERC-721 与 ERC-1155 的底层存储模型，解密 Azuki 首创的 ERC-721A 批量铸造 Gas 优化黑科技，并提供可直接运行的租赁（ERC-4907）与版税（ERC-2981）生产级合约实现。

---

## 1. ERC-721 与 ERC-1155 的设计哲学与底层存储对比

在以太坊中，NFT 有两套最主流的规范标准：**ERC-721**（单资产单所有权）与 **ERC-1155**（多资产多份额，即半同质化代币 SFT）。

### 1.1 数据结构与存储开销对比
这两者在智能合约底层的 Mapping 设计和数据隔离上存在根本性差异：

#### ERC-721 (独一无二所有权)
底层通过将每一个唯一的 `tokenId` 显式映射到一个特定的 `address` 来管理：
```solidity
// ERC-721 核心映射
mapping(uint256 => address) private _owners;       // 每个 NFT 只有唯一的主人
mapping(address => uint256) private _balances;     // 记录用户拥有的 NFT 数量
```
- **特点**：每个 NFT 的 `tokenId` 都是独一无二的，适合加密艺术品、虚拟地产等。

#### ERC-1155 (半同质化代币 - SFT)
底层采用双重嵌套映射，允许单个 `tokenId` 拥有多个副本和多个所有者：
```solidity
// ERC-1155 核心映射
mapping(uint256 => mapping(address => uint256)) private _balances; // tokenId -> 账户 -> 余额
```
- **特点**：对于某个 `tokenId`（如游戏中的“铁剑”），可以有成千上万个用户同时持有不同数量的该物品，从而巧妙地把同质化代币（ERC20）与非同质化代币（ERC721）融合在一起。

### 1.2 多维对比矩阵

| 特性维度 | ERC-721 | ERC-1155 |
| :--- | :--- | :--- |
| **代币性质** | 完全非同质化（Non-Fungible） | 半同质化（Semi-Fungible, SFT） |
| **底层映射结构** | `tokenId => Owner` | `tokenId => (Holder => Balance)` |
| **元数据存储** | 每个 Token 可通过 `tokenURI` 映射到独立 JSON | 使用全局模板化 URI（如 `https://ipfs.io/ipfs/{id}.json`） |
| **多代币批量转账**| 必须拆分为多次交易，或多次循环调用 `transferFrom` | 支持 `safeBatchTransferFrom`，单笔交易批量转账多个 Token 类别 |
| **Gas 效率** | 频繁交易或批量管理时 Gas 极高 | 极佳的 Gas 效率（适合游戏内海量道具） |
| **适用场景** | 数字艺术品、PFP 社交头像、域名、DeFi LP 头寸 | 游戏装备、游戏道具、多卡牌收集、多权益门票 |

---

## 2. ERC-721A：Azuki 团队首创的批量铸造 Gas 优化机制

在 2022 年初的 NFT 牛市中，Azuki 团队推出了 **ERC-721A**，解决了以太坊网络在 NFT 批量发售（Batch Minting）时高昂的 Gas 费痛点。

### 2.1 传统 ERC-721（OpenZeppelin）的批量铸造痛点
当用户调用 `mint(10)` 一次性批量铸造 10 个 NFT 时，OpenZeppelin 的实现需要在 `_owners` 和 `_balances` 两个 mapping 中执行 10 次写入：
- 每次修改未被初始化的 Storage Slot 需要花费 $20,000$ Gas。
- 批量铸造 10 个 NFT 需要花费至少 $200,000+$ Gas，造成严重的网络拥堵与高昂的交易费。

### 2.2 ERC-721A 的三大核心优化黑科技

```
 传统 OZ 批量铸造 (100 - 102) 底层所有权分配：
 [Token 100] --> Owner: Alice
 [Token 101] --> Owner: Alice
 [Token 102] --> Owner: Alice

 ERC-721A 批量铸造 (100 - 102) 优化：
 [Token 100] --> Owner: Alice  (仅在此写入一次所有权 Slot)
 [Token 101] --> Owner: 0x00...00 (默认为空，延迟初始化)
 [Token 102] --> Owner: 0x00...00 (默认为空，延迟初始化)
```

#### 优化 1：所有权延迟初始化（Shared Ownership / Lazy Initialization）
在批量铸造时，如果要将 ID 为 $100$ 到 $109$ 的 10 个 NFT 铸造给 Alice：
- ERC-721A **仅将 ID 100 的所有者写入 Alice**，而 101 到 109 的所有权位置全部保持默认为 `address(0)`。
- **查询逻辑**：当调用 `ownerOf(105)` 时，算法会从 105 开始**递减向上追溯**，直到发现第一个非空的所有权记录（即 ID 100 对应的 Alice）。此时便能推断出 105 的所有者同样是 Alice。
- **转移开销转移**：在 `mint` 时完全不需要为 101-109 写入所有权数据，极大地降低了铸造 Gas。代价是将这部分写开销转移到了未来的 `transfer` 阶段（当 Alice 转移其中某一只时，合约会将分割出来的连续区间重新写好）。

#### 优化 2：移除 `_balances` 的高频冗余更新
当 Alice 批量铸造 10 个 NFT 时，合约在全局 `_balances` 映射中仅对 Alice 的余额执行 **1 次追加写加法**（即 `balance += 10`），而不是 10 次独立的递增修改。

#### 优化 3：位运算与紧凑打包（Bit-packing）
ERC-721A 不使用复杂的结构体数组，而是使用单条 256 位（1 Slot）数据通过**位运算**打包：
- **地址 (Address)**: 占 160 位
- **铸造时间戳 (Timestamp)**: 占 64 位
- **销毁标记 (Burned)**: 占 1 位
- **额外元数据 (Extra Data)**: 占 31 位
通过 `shl`（左移）和 `or`（按位或）位运算符在单次 SSTORE 中存入，从而压榨每一个 Gas 开销。

---

## 3. ERC-4907：租赁 NFT（所有权与使用权分离）

随着链上游戏（GameFi）和元宇宙的发展，“租借装备/道具”成为了刚需。**ERC-4907** 作为 ERC-721 的官方扩展协议，通过引入**带自动过期时间的使用权角色（User Role）**，实现了所有权（Ownership）与使用权（Usership）的优雅分离。

### 3.1 ERC-4907 核心机制
- 传统租赁通常需要承租人向中介智能合约交纳高额押金或进行复杂的托管。
- ERC-4907 直接在代币层增加一个使用权持有者字段 `user` 和一个过期时间戳 `expires`。
- **无需链上主动退租**：当时间戳到达过期值后，`user` 的使用权将自动失效（链下及链上检测逻辑判定其到期），**不需要任何链上清算交易，不耗费任何 Gas 费**，实现了“自动清算”。

### 3.2 生产级 ERC-4907 智能合约实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

interface IERC4907 {
    // 租赁使用权发生改变时触发的事件
    event UpdateUser(uint256 indexed tokenId, address indexed user, uint64 expires);

    /**
     * @notice 设置 NFT 的使用权和过期时间
     * @param tokenId 目标代币 ID
     * @param user 承租人/使用者地址
     * @param expires 过期时间戳 (UNIX 时间)
     */
    function setUser(uint256 tokenId, address user, uint64 expires) external;

    /**
     * @notice 获取当前的使用者
     */
    function userOf(uint256 tokenId) external view returns (address);

    /**
     * @notice 获取使用者的过期时间
     */
    function userExpires(uint256 tokenId) external view returns (uint256);
}

contract ERC4907 is ERC721, IERC4907 {
    struct UserInfo {
        address user;   // 使用人地址
        uint64 expires; // 到期时间戳
    }

    // tokenId => 使用人信息
    mapping(uint256 => UserInfo) private _users;

    constructor(string memory name_, string memory symbol_) ERC721(name_, symbol_) {}

    /**
     * @dev 设置使用者角色，必须是 Owner 或经授权的地址才能调用
     */
    function setUser(uint256 tokenId, address user, uint64 expires) public virtual override {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Caller is not owner nor approved");
        
        UserInfo storage info = _users[tokenId];
        info.user = user;
        info.expires = expires;

        emit UpdateUser(tokenId, user, expires);
    }

    /**
     * @dev 返回该 NFT 的当前实际使用者（已考虑过期时间自动判定）
     */
    function userOf(uint256 tokenId) public view virtual override returns (address) {
        if (uint64(block.timestamp) <= _users[tokenId].expires) {
            return _users[tokenId].user;
        }
        return address(0); // 过期或未设置则退回 address(0)
    }

    /**
     * @dev 返回过期时间戳
     */
    function userExpires(uint256 tokenId) public view virtual override returns (uint256) {
        return _users[tokenId].expires;
    }

    /**
     * @dev 在代币转移（Transfer）时，必须清除使用权（重置为 0），防止买家买到带有旧租约的 NFT
     */
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 firstTokenId,
        uint256 batchSize
    ) internal virtual override {
        super._beforeTokenTransfer(from, to, firstTokenId, batchSize);

        if (from != to && _users[firstTokenId].user != address(0)) {
            delete _users[firstTokenId];
            emit UpdateUser(firstTokenId, address(0), 0);
        }
    }
}
```

---

## 4. ERC-2981：通用 NFT 版税标准

在早期的 NFT 市场中，版税是在交易中心层（例如 OpenSea、LooksRare）通过链下中心化设置的，极易产生碎片化以及“无版权市场白嫖”问题。**ERC-2981** 作为通用的链上版税标准，为跨平台资产流转提供了统一的版税计算和结算接口。

### 4.1 版税设计的核心理念
- **链上不可强制扣款**：由于 ERC721 的 `transferFrom` 只执行代币转移，无法得知这笔转移是不是一笔真实交易（还是用户仅仅换钱包），因此**无法在转账层强制扣除版税**。
- **标准化版税信息返回**：ERC-2981 提供了一个只读函数 `royaltyInfo`。任何交易市场（DEX/Marketplace）在执行成交匹配时，均应调用此接口，获取版税接收者及应付版税，并在撮合结算时将版税转给作者。

### 4.2 生产级 ERC-2981 智能合约集成实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/utils/introspection/IERC165.sol";

interface IERC2981 is IERC165 {
    /**
     * @notice 查询版税分配
     * @param tokenId 待结算的代币 ID
     * @param salePrice 资产真实的销售总金额
     * @return receiver 版税接收者的国库地址
     * @return royaltyAmount 应该提取支付的版税金额
     */
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external
        view
        returns (address receiver, uint256 royaltyAmount);
}

contract ERC2981NFT is ERC721, IERC2981 {
    address private _royaltyReceiver; // 版税收取地址
    uint96 private _royaltyFraction;  // 版税占比基数（如 5% = 500，采用万分位基基数 10000）

    uint256 private constant FEE_DENOMINATOR = 10000;

    constructor(
        string memory name,
        string memory symbol,
        address royaltyReceiver_,
        uint96 royaltyFraction_
    ) ERC721(name, symbol) {
        _royaltyReceiver = royaltyReceiver_;
        _royaltyFraction = royaltyFraction_;
    }

    /**
     * @dev 满足 ERC-2981 规范，返回计算好的版税数额
     */
    function royaltyInfo(uint256 tokenId, uint256 salePrice)
        external
        view
        override
        returns (address receiver, uint256 royaltyAmount)
    {
        // 验证代币存在性
        require(_exists(tokenId), "ERC2981NFT: Royalty query for nonexistent token");
        
        uint256 amount = (salePrice * _royaltyFraction) / FEE_DENOMINATOR;
        return (_royaltyReceiver, amount);
    }

    /**
     * @dev 必须显式声明支持的接口，满足 ERC-165（多接口支持判定机制）
     */
    function supportsInterface(bytes4 interfaceId)
        public
        view
        virtual
        override(ERC721, IERC165)
        returns (bool)
    {
        return interfaceId == type(IERC2981).interfaceId || super.supportsInterface(interfaceId);
    }
}
```

---

## 5. NFT 标准选择决策树

当开发一个新的 Web3 项目时，可以根据以下决策指引选择最合适的 NFT 开发底层：

```
                           你的项目是什么类型？
                                    │
         ┌──────────────────────────┼──────────────────────────┐
         ▼                          ▼                          ▼
   【独一无二艺术品】          【高频、批量 Mint 的 PFP】     【大型 GameFi 道具/卡牌】
         │                          │                          │
         ▼                          ▼                          ▼
   【ERC-721】                  【ERC-721A】               【ERC-1155】
   (经典 OpenZeppelin)         (Azuki 极低 Gas 优化)       (支持半同质化与多代币划转)
         │                          │                          │
  是否需支持租赁/版税？        是否需支持租赁/版税？        是否需支持版税？
         │                          │                          │
 ┌───────┴───────┐          ┌───────┴───────┐          ┌───────┴───────┐
 ▼               ▼          ▼               ▼          ▼               ▼
【纯ERC-721】  【集成ERC-2981】 【纯ERC-721A】 【集成ERC-4907】 【纯ERC-1155】 【集成ERC-2981】
               【集成ERC-4907】               【集成ERC-2981】
```
