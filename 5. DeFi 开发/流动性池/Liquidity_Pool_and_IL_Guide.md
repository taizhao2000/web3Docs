# 流动性池、LP Token 与无常损失数学指南

> 流动性池是去中心化金融的世界级底层积木。本指南深度解析流动性份额代币（LP Token）的铸造与销毁数学原理、无常损失（Impermanent Loss）的公式推导、以及生产级 Staking 挖矿合约的分配算法。

---

## 1. LP Token（流动性凭证）铸造与销毁数学原理

当用户向 AMM（例如 Uniswap V2）中存入代币 X 和 Y 时，池子会向用户按比例发行（Mint）代表其份额的 **LP Token（流动性提供者代币）**。LP Token 遵循 ERC20 标准，可以转移、抵押甚至再质押。

### 1.1 首次添加流动性（开天辟地）
当池子为空时，需要初始化池子的流动性。为了防止早期流动性提供者通过操纵极小的代币存入量来卡死池子份额：

$$S_{\text{minted}} = \sqrt{x_{\text{deposited}} \cdot y_{\text{deposited}}} - \text{MINIMUM\_LIQUIDITY}$$

- $\text{MINIMUM\_LIQUIDITY}$：默认是 $10^3$（即 $1000$ wei），这部分份额会被永远转入 `address(0)` 锁死。这样可以防止任何人将池子的价值归零，从而导致除零错误，同时也避免了“通胀攻击”的攻击向量。

### 1.2 后续按比例添加流动性
当池子已经有存量流动性时，再次添加流动性必须**严格按照池子现有的代币比例**存入。发行的 LP Token 数量由用户注入的资产占比最低的一方决定（木桶原理）：

$$S_{\text{minted}} = \min \left( \frac{\Delta x}{x} \cdot S_{\text{total}}, \quad \frac{\Delta y}{y} \cdot S_{\text{total}} \right)$$

- $S_{\text{total}}$: 当前已经发行的 LP Token 总供应量。
- $\Delta x$, $\Delta y$: 用户投入的 X 和 Y 代币数量。
- $x$, $y$: 注入前池子中 X 和 Y 的储备量。

### 1.3 移除流动性（销毁）
当 LP 想要撤资时，将 LP Token 退还给合约进行销毁（Burn），合约根据用户持有的份额占总供应量的比例退还其代币：

$$\Delta x = \frac{S_{\text{burned}}}{S_{\text{total}}} \cdot x$$

$$\Delta y = \frac{S_{\text{burned}}}{S_{\text{total}}} \cdot y$$

---

## 2. 无常损失 (Impermanent Loss) 的严格数学推导

无常损失是每个 LP 的“阿喀琉斯之踵”。它是指：**由于市场价格发生偏离，LP 将代币存入池子中提供的流动性，与简单将相同代币持有在手中（HODL）相比所产生的机会成本损失**。

```
                   无常损失比率 vs 价格变化倍数
  0% ──[1.0x]─────────────────────────────────
 -5%             [1.5x] (-2.0%)
-10% 
-15%                           [2.0x] (-5.7%)
-20%                                         [3.0x] (-13.4%)
-25%                                                       [5.0x] (-25.5%)
```

### 2.1 经典推导公式
设初始时池子处于均衡价格 $P = \frac{y}{x}$，用户提供了一定流动性。
在经历了价格波动后，新价格变为 $P' = k_{\text{price}} \cdot P$，其中 $k_{\text{price}}$ 是价格变化系数（如价格变为原来的 2 倍，则 $k_{\text{price}} = 2$）。

在无手续费复利前提下，通过 $x \cdot y = k$ 及几何套利关系，可严格推导出 **无常损失公式**：

$$IL(k_{\text{price}}) = \frac{\text{资产在 LP 池内的总价值}}{\text{仅持有在手中（HODL）的总价值}} - 1$$

$$IL(k_{\text{price}}) = \frac{2\sqrt{k_{\text{price}}}}{1 + k_{\text{price}}} - 1$$

### 2.2 价格变化倍数与无常损失对照表

| 价格变化倍数 ($k_{\text{price}}$) | 无常损失 ($IL$) | 解释 |
|:---|:---|:---|
| **1.0x** | **0.00%** | 无价格偏离，无损失 |
| **1.25x** | **-0.60%** | 价格上涨 25% |
| **1.50x** | **-2.00%** | 价格上涨 50% |
| **2.00x** | **-5.72%** | 价格翻倍，产生约 5.7% 机会亏损 |
| **3.00x** | **-13.40%** | 价格上涨至 3 倍 |
| **4.00x** | **-20.00%** | 价格上涨至 4 倍 |
| **5.00x** | **-25.50%** | 价格上涨至 5 倍 |
| **0.50x** | **-5.72%** | 价格腰斩，同样产生 5.7% 的损失 |

> 💡 **专家避坑提示**：
> 1. 无常损失是**单向向下**的，无论价格是上涨还是下跌，LP 相比 HODL 都是产生额外损失（因为池子在用你的币被迫向相反方向逆市操作）。
> 2. 只有当价格完美回落到初始存入价格时，无常损失才会消失为 0。这也解释了为什么 LP 最适合在**价格剧烈震荡但中枢维持不变**的稳定币或蓝筹代币对中提供。

---

## 3. 生产级流动性挖矿 (Staking Rewards) 算法剖析

当用户获得 LP Token 后，为了激励用户不撤资，协议通常会推出 Staking 挖矿池，根据时间向质押 LP 的用户发放流动性挖矿代币奖励。

### 3.1 传统循环发放的 Gas 噩梦
如果在每过一秒或有人质押/撤资时，合约都需要用一个 `for` 循环遍历所有质押的用户并更新其收益，当用户数超过数百人时，Gas 消耗会超出区块上限，导致合约**被彻底锁死**。

### 3.2 黄金标准：Synthetix Staking 算法（$O(1)$ 时间复杂度）
Synthetix 首创了一种**分摊累加算法**，无论有多少用户质押，每一次存入、提现、提取收益的 Gas 复杂度均为恒定的 $O(1)$。

#### 核心数学原理：
我们将一段时间内的总奖励率设为 $R$（每秒发放多少奖励代币），在任意时刻 $t$：

$$\text{Reward Per Token} (r_t) = r_{t-1} + \frac{R \cdot (t - t_{\text{last}})}{L_{\text{total\_staked}}}$$

其中：
- $L_{\text{total\_staked}}$: 当前在 Staking 池中被所有用户质押的 LP Token 总量。
- $r_t$: 累计到当前时间点，每单位 LP 代币应得的奖励总额累计值。

对于任何特定的用户 $i$，其在时间点 $t$ 的**已赚取奖励（Earned）**：

$$\text{Earned}_i = \text{Staked}_i \cdot \left( r_t - r_{\text{user\_paid\_index}, i} \right) + \text{Rewards\_Accrued}_i$$

- $r_{\text{user\_paid\_index}, i}$：记录用户上次结算时，已经享受过的累计 $r$ 历史值。
- 当用户质押、提现或加仓时，合约强制先调用 `updateReward` 刷新该用户的历史小账本，将当前利润锁入 `Rewards_Accrued`，并将其 $r_{\text{user\_paid\_index}}$ 抬高到当前的 $r_t$。

#### 生产级 Staking 极简合约实现：
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC20 {
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function transfer(address to, uint256 value) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract StakingPool {
    IERC20 public stakingToken; // 质押代币（如 LP Token）
    IERC20 public rewardToken;  // 奖励代币

    uint256 public rewardRate; // 每秒发放的奖励数量
    uint256 public lastUpdateTime;
    uint256 public rewardPerTokenStored;

    mapping(address => uint256) public userRewardPerTokenPaid;
    mapping(address => uint256) public rewards; // 用户累计锁定的奖励

    uint256 private _totalSupply; // 质押总额
    mapping(address => uint256) private _balances; // 用户质押余额

    modifier updateReward(address account) {
        rewardPerTokenStored = rewardPerToken();
        lastUpdateTime = block.timestamp;
        
        if (account != address(0)) {
            rewards[account] = earned(account);
            userRewardPerTokenPaid[account] = rewardPerTokenStored;
        }
        _;
    }

    constructor(address _stakingToken, address _rewardToken, uint256 _rewardRate) {
        stakingToken = IERC20(_stakingToken);
        rewardToken = IERC20(_rewardToken);
        rewardRate = _rewardRate;
        lastUpdateTime = block.timestamp;
    }

    // 1. 获取累积到现在的每 token 收益
    function rewardPerToken() public view returns (uint256) {
        if (_totalSupply == 0) {
            return rewardPerTokenStored;
        }
        return rewardPerTokenStored + (rewardRate * (block.timestamp - lastUpdateTime) * 1e18) / _totalSupply;
    }

    // 2. 计算用户累积已赚取尚未领取的奖励
    function earned(address account) public view returns (uint256) {
        return (_balances[account] * (rewardPerToken() - userRewardPerTokenPaid[account])) / 1e18 + rewards[account];
    }

    // 3. 质押 (O(1))
    function stake(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot stake 0");
        _totalSupply += amount;
        _balances[msg.sender] += amount;
        stakingToken.transferFrom(msg.sender, address(this), amount);
    }

    // 4. 提现 (O(1))
    function withdraw(uint256 amount) external updateReward(msg.sender) {
        require(amount > 0, "Cannot withdraw 0");
        _totalSupply -= amount;
        _balances[msg.sender] -= amount;
        stakingToken.transfer(msg.sender, amount);
    }

    // 5. 领奖 (O(1))
    function getReward() external updateReward(msg.sender) {
        uint256 reward = rewards[msg.sender];
        if (reward > 0) {
            rewards[msg.sender] = 0;
            rewardToken.transfer(msg.sender, reward);
        }
    }
}
```
