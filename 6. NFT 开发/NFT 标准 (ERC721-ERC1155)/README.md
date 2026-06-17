# NFT 标准 (ERC721/ERC1155)

## ERC-721

最常用的 NFT 标准，每个代币具有唯一 ID。

### 核心接口

```solidity
interface IERC721 {
    function balanceOf(address owner) external view returns (uint256);
    function ownerOf(uint256 tokenId) external view returns (address);
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    function approve(address to, uint256 tokenId) external;
    function setApprovalForAll(address operator, bool approved) external;
}
```

### 扩展标准

| 标准 | 说明 |
|------|------|
| ERC-721A | Azuki 优化的批量铸造（省 Gas） |
| ERC-4907 | 租赁 NFT（分离使用权和所有权） |
| ERC-2981 | NFT 版税标准 |

## ERC-1155

多代币标准，同一合约可管理同质化和非同质化代币。

### 核心接口

```solidity
interface IERC1155 {
    function balanceOf(address account, uint256 id) external view returns (uint256);
    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids) external view returns (uint256[] memory);
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data) external;
    function safeBatchTransferFrom(address from, address to, uint256[] calldata ids, uint256[] calldata amounts, bytes calldata data) external;
}
```

## 标准对比

| 特性 | ERC-721 | ERC-1155 |
|------|---------|----------|
| 代币类型 | 仅非同质化 | 同质化+非同质化 |
| 批量操作 | 需逐个处理 | 原生支持 |
| 合约数量 | 每种 NFT 一个合约 | 多种代币一个合约 |
| Gas 效率 | 较低 | 较高 |
| 生态支持 | 最广泛 | 快速增长 |
