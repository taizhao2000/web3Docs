# 5.1 DEX 开发（Decentralized Exchange）

去中心化交易所（DEX）是整个 DeFi 世界的流动性基石。通过部署在区块链上的智能合约，DEX 允许任何人在零准入、非托管且无中心化第三方的环境下执行点对点的资产兑换。

本专题深度解析了 AMM 的数学算法、价格滑点控制，以及原生代币与 ERC-20 标准之间的桥梁与交易流程。

---

## 🧭 知识卡片与导航

本子树包含以下两篇高价值技术指南，请点击链接直接阅读：

| 技术指南文档 | 核心内容简介 | 核心技术关键词 |
| :--- | :--- | :--- |
| 📖 **[AMM & DEX 核心原理指南](./AMM_and_DEX_Guide.md)** | 深入推导恒定乘积 $x \cdot y = k$ 数学与几何特性，详解价格影响、滑点计算公式与合约防夹子代码。 | 恒定乘积公式、价格影响、滑点保护、Uniswap V2 / V3、边际价格。 |
| 📖 **[原生代币、ERC-20 与 WETH 封装机制](./Native_vs_ERC20_and_WETH.md)** | 回答“为什么 ERC-20 能够被自动交易？”、“原生代币与 ERC-20 物理差异是什么？”、“WETH 如何在 1:1 双向锚定中充当桥梁？”，以及无第三方参与的链上 USDT 购买 ETH 的原子数据流拆解。 | ERC-20 统一接口、原生 ETH 转账、WETH 寄存机制、1:1 物理锚定、Uniswap 路由原子数据流。 |

---

## 🛠️ DEX 交易核心工作流速览

在以太坊等区块链网络上，一次非托管的、不通过中心化交易所（第三方）的原生资产兑换，涉及以下两个经典流程：

### 1. 经典代币兑换（ERC-20 To ERC-20）
```solidity
// 用户需要先调用 ERC-20 合约的 approve，将额度授予 DEX 路由合约。
// 随后，路由合约在 swapExactTokensForTokens 中使用 transferFrom 扣减 Token A，并 transfer 派发 Token B。
function swapExactTokensForTokens(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

### 2. 包含原生资产的兑换（ERC-20 To ETH）
```solidity
// 用户输入 USDT 兑换 ETH。
// 路由合约内部流程：
// 1. 调用 USDT.transferFrom()，将用户的 USDT 存入 USDT-WETH 的流动性池。
// 2. 兑换出 WETH 并转给路由合约。
// 3. 路由合约调用 WETH.withdraw() 销毁 WETH 提取原生 ETH。
// 4. 路由合约将原生 ETH 转入用户钱包。
function swapExactTokensForETH(
    uint amountIn,
    uint amountOutMin,
    address[] calldata path,
    address to,
    uint deadline
) external returns (uint[] memory amounts);
```

整个过程是**原子性（Atomic）**的，即 100% 成功，或者在发生异常、滑点过大时 100% 自动撤销回滚，不产生任何资金滞留和托管风险。
