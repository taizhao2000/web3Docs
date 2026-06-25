# AMM 核心原理与 DEX 开发深度指南

> 去中心化交易所（DEX）是整个 DeFi 生态的流动性枢纽。本指南深度解析恒定乘积自动化做市商（AMM）的数学模型、价格影响、滑点计算，以及 Uniswap V3 集中流动性（Concentrated Liquidity）的硬核几何数学推导。

---

## 1. 恒定乘积做市商 (Constant Product AMM)

以 Uniswap V2 为代表的经典 AMM，其核心基于极其简单的**恒定乘积公式**：

$$x \cdot y = k$$

- $x$: 池中代币 X 的储备量（Reserve）
- $y$: 池中代币 Y 的储备量（Reserve）
- $k$: 恒定乘积常数（在无流动性添加/移除时保持不变）

---

### 1.1 极简通俗解构：水果摊的“怪脾气”与价格自动调节

为了直观理解 $x \cdot y = k$，我们用一个通俗的“水果摊故事”来解构其背后的数学与金融直觉：

设想有一个只卖 **苹果（代币 X）** 和 **香蕉（代币 Y）** 的水果摊。摊主是一个叫“智能合约”的机器人，他脑子里没有“货币单位（元）”的概念，他唯一的铁律是：
> **“我不管苹果和香蕉各自值多少钱，只要有人来我这里以物易物，我摊位上的【苹果库存数量 $x$】乘以【香蕉库存数量 $y$】，乘积必须永远等于常数 $k$！”**

假设初始状态下：
* 摊位上有 **10 个苹果（$x = 10$）**。
* 摊位上有 **10 个香蕉（$y = 10$）**。
* 此时恒定乘积常数 **$k = 10 \times 10 = 100$**。两者的兑换比例为 $1:1$。

#### 交易发生：小红要买 5 个苹果
1. **买走后的苹果库存**：剩下 $10 - 5 = 5$ 个。
2. **铁律计算**：既然苹果只剩 $5$ 个，而乘积 $k$ 必须锁定在 $100$，那么香蕉的库存数量必须变成：
   $$y_{\text{new}} = \frac{k}{x_{\text{new}}} = \frac{100}{5} = 20 \text{ 个}$$
3. **结算应付香蕉**：为了让香蕉存量从 10 个增加到 20 个，小红必须给水果摊**放入 $20 - 10 = 10$ 个香蕉（$\Delta y = 10$）**。

#### 核心金融启示：
* **价格自动调节**：苹果的价格在交易前明明是 1 香蕉，但小红买 5 个时，平均每个苹果耗费了 2 个香蕉（实际成交均价 $P_{\text{exec}} = \frac{10}{5} = 2$）。随着苹果被买走（变少、稀缺），苹果的价格**自动暴涨**。
* **无需外部喂价**：合约（水果摊）完全通过**内部库存的变化**，自然而然地实现了一套最符合“供需关系”的自适应定价机制。

---

### 1.2 为什么常数 $k$ 是“恒定”的？（恒定的边界条件）

很多初学者最大的误区在于以为 $k$ 这个值在合约诞生后就永远一成不变。事实上，**“恒定”有着严格的边界限制**：

#### 场景 A：在仅有“交易兑换”发生时，$k$ 理论上保持恒定
当用户在池子里用 X 兑换 Y 时，这属于“以物易物”。没有资金注入或抽离，所以 $k$ 必须作为数学枷锁保持恒定，以此约束兑换比例。

* 💡 **生产级反直觉真相（手续费累加导致 $k$ 单向变大）**：
  在真实的 Uniswap 生产合约中，每笔交易会扣除 **$0.3\%$ 的交易手续费**。手续费在执行兑换前直接留存在池子中不参与兑换算术。因此，**在每笔交易结束后，由于手续费的复利留存，真实的新乘积 $k_{\text{new}}$ 实际上会比旧的 $k_{\text{old}}$ 稍微变大一点点**。这是流动性提供者（LP）收取手续费复利增值的底层原理。

#### 场景 B：当发生“添加/移除流动性”时，$k$ 会发生突变
当流动性提供者（LP）充值或提现时，$k$ 发生突变：
* **添加流动性**：LP 必须按照当前池子的比例同时存入 X 和 Y（如：同时放入 10 个苹果和 10 个香蕉）。此时，$x$ 变为 20，$y$ 变为 20。新的乘积骤增为 $k_{\text{new}} = 20 \times 20 = 400$。
* **池子变深（$k$ 变大）的重大金融意义**：
  当 $k$ 从 100 变成 400 时，我们再次模拟小红买 5 个苹果：
  * 苹果库存从 20 变为 15。
  * 要求的香蕉数：$400 / 15 = 26.67$。
  * 小红需付出的香蕉：$26.67 - 20 = 6.67$ 个（平均单价约 1.33）。
  * **结论**：**$k$ 越大，池子越深，大额交易产生的价格滑点（Price Impact）就越低，用户的交易摩擦成本就越小。**

---

### 1.3 渐近双曲线的几何魔力

在数学上，方程 $x \cdot y = k$ 的几何图像是直角坐标系第一象限内的一条**渐近双曲线**：

1. **资金永远买不空（Infinite Liquidity）**：
   看双曲线的边缘：当 $x$ 无限趋近于 $0$（意味着你想买光池子里所有的 X）时，曲线会无限向上延伸，此时要求的 $y$（你需要支付的 Y 数量）会**趋近于无穷大（$\infty$）**。因此，任何人都无法买空池子里的某一种代币，保证了去中心化交易所永远不会因为资产买空而发生交易挂起死锁。
2. **斜率即现货价格**：
   在双曲线上，某一个点 $(x_1, y_1)$ 的切线斜率（一阶导数）：
   $$\text{Price} = -\frac{dy}{dx} = \frac{y}{x}$$
   它恰好就是此时此刻 X 对 Y 的**瞬时现货价格**。用户每一次兑换代币，物理上就是**在把这个坐标点沿着双曲线向左或向右推动**。

---

### 1.4 无手续费下的交易公式（零滑点底噪）
假设用户输入 $\Delta x$ 个代币 X，想要换出 $\Delta y$ 个代币 Y，根据恒定乘积公式，池中储备量发生改变：

$$(x + \Delta x) \cdot (y - \Delta y) = k$$

将 $k = x \cdot y$ 代入：

$$(x + \Delta x) \cdot (y - \Delta y) = x \cdot y$$

通过简单的代数整理，推导得出**输出代币 $\Delta y$ 的计算公式**：

$$\Delta y = \frac{y \cdot \Delta x}{x + \Delta x}$$

---

### 1.5 带交易手续费（如 0.3%）的生产级公式
在 Uniswap V2 中，交易会收取 $0.3\%$（即 $\gamma = 0.997$）的手续费，该手续费在执行兑换前被直接扣除并留在池中。

扣除手续费后的有效输入量为 $\Delta x \cdot 0.997$。由此推导出的**生产级输出代币 $\Delta y$ 计算公式**（即 Uniswap V2 `getAmountOut` 接口实现）：

$$\Delta y = \frac{y \cdot \Delta x \cdot 997}{x \cdot 1000 + \Delta x \cdot 997}$$

#### 对应 Solidity 代码实现：
```solidity
function getAmountOut(
    uint256 amountIn,
    uint256 reserveIn,
    uint256 reserveOut
) internal pure returns (uint256 amountOut) {
    require(amountIn > 0, "DEX: INSUFFICIENT_INPUT_AMOUNT");
    require(reserveIn > 0 && reserveOut > 0, "DEX: INSUFFICIENT_LIQUIDITY");
    
    // 1. 扣除 0.3% 手续费（输入乘以 997）
    uint256 amountInWithFee = amountIn * 997;
    
    // 2. 分子计算 = y * (amountIn * 997)
    uint256 numerator = amountInWithFee * reserveOut;
    
    // 3. 分母计算 = x * 1000 + (amountIn * 997)
    uint256 denominator = reserveIn * 1000 + amountInWithFee;
    
    amountOut = numerator / denominator;
}
```

---

## 2. 价格影响 (Price Impact) 与滑点 (Slippage)

在 AMM 中，每一次兑换都会推动池内的储备量发生偏移，进而使得实际成交价偏离初始价格。

### 2.1 瞬时边际价格 vs 实际成交均价
- **边际价格 (Marginal Price)**：无限小额交易时的现货价 $P_0 = \frac{y}{x}$。
- **实际成交均价 (Execution Price)**：交易发生的均价 $P_{\text{exec}} = \frac{\Delta y}{\Delta x} = \frac{y}{x + \Delta x}$。
- **价格影响 (Price Impact)**：由于你的交易规模造成的池子现货价格位移：

$$\text{Price Impact} = \frac{P_0 - P_{\text{exec}}}{P_0} = \frac{\Delta x}{x + \Delta x}$$

> 💡 **核心启示**：当单笔交易规模 $\Delta x$ 占池子储备量 $x$ 的比例较大时，价格影响会急剧放大，交易者在买入时会买在更贵的价格。

### 2.2 滑点保护机制 (Slippage Protection)
为了防止用户在发送交易到最终打包期间，池子被其他交易改变或被[夹子机器人](../../4.%20后端服务/Mempool_and_MEV_Guide.md#3-mev最大可提取价值内幕夹子攻击)攻击导致买入代币严重缩水，合约中必须设定 **滑点保护** 参数 `amountOutMin`。

如果允许滑点为 $1\%$，在前端计算得到预期 $\Delta y$ 后：

$$\text{amountOutMin} = \Delta y \cdot (1 - 0.01) = \Delta y \cdot 0.99$$

#### 生产级滑点限制兑换函数代码：
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

interface IERC20 {
    function transferFrom(address from, address to, uint256 value) external returns (bool);
    function transfer(address to, uint256 value) external returns (bool);
}

contract SimpleDEX {
    address public tokenX;
    address public tokenY;
    uint256 public reserveX;
    uint256 public reserveY;

    event Swap(address indexed sender, uint256 amountIn, uint256 amountOut, bool xToY);

    // 带滑点保护的 Swap 函数
    function swap(
        uint256 amountIn,
        uint256 amountOutMin, // 滑点限制：低于此值直接回滚
        bool xToY
    ) external returns (uint256 amountOut) {
        require(amountIn > 0, "DEX: INVALID_AMOUNT");
        
        (uint256 resIn, uint256 resOut) = xToY ? (reserveX, reserveY) : (reserveY, reserveX);
        
        // 1. 计算理论输出量
        amountOut = getAmountOut(amountIn, resIn, resOut);
        
        // 2. 检查滑点下限
        require(amountOut >= amountOutMin, "DEX: SLIPPAGE_LIMIT_EXCEEDED");
        
        // 3. 转移代币与状态更新
        if (xToY) {
            IERC20(tokenX).transferFrom(msg.sender, address(this), amountIn);
            IERC20(tokenY).transfer(msg.sender, amountOut);
            reserveX += amountIn;
            reserveY -= amountOut;
        } else {
            IERC20(tokenY).transferFrom(msg.sender, address(this), amountIn);
            IERC20(tokenX).transfer(msg.sender, amountOut);
            reserveY += amountIn;
            reserveX -= amountOut;
        }
        
        emit Swap(msg.sender, amountIn, amountOut, xToY);
    }

    function getAmountOut(uint256 amountIn, uint256 rIn, uint256 rOut) public pure returns (uint256) {
        uint256 amountInWithFee = amountIn * 997;
        return (amountInWithFee * rOut) / (rIn * 1000 + amountInWithFee);
    }
}
```

---

## 3. Uniswap V3 集中流动性 (Concentrated Liquidity) 数学原理解析

Uniswap V2 的 $x \cdot y = k$ 虽然简单，但其流动性分布在价格 $[0, \infty)$ 整个区间。实际上，绝大多数交易集中在当前现货价格附近，这意味着分布在极高和极低价格的流动性永远不会被触及，造成了**极低的资本效率**。

Uniswap V3 允许流动性提供者（LP）在**指定的价格区间 $[P_a, P_b]$ 内提供流动性**。其数学原理基于“虚拟乘积曲线”的平移。

```
V2: [0 ─────────────────────────────────────── ∞]  (资金效率低)
V3:     [Pa ────────────■ Current Price ──── Pb]   (资金只在区间工作，效率极高)
```

### 3.1 虚拟准备金公式推导
要在有限价格区间内逼近 $x \cdot y = k$，Uniswap V3 引入了平移公式：

$$(x + x_v) \cdot (y + y_v) = L^2$$

- $L$: 流动性乘数（流动性深度 $L = \sqrt{k}$）
- $x_v$, $y_v$: 虚拟代币储备，使得当价格达到边界 $P_a$ 或 $P_b$ 时，真实的代币储备极少被彻底耗尽。

在任意价格 $P = \frac{y}{x}$ 下：

$$\sqrt{P} = \frac{y}{L} = \frac{L}{x}$$

对于指定的流动性区间 $[P_a, P_b]$，我们可以解出对应的虚拟储备偏移量：
- 当价格上涨到最高点 $P_b$ 时，池中所有的代币 Y 被全部兑换成了代币 X，此时真实代币 $y = 0$。
- 当价格下跌到最低点 $P_a$ 时，池中所有的代币 X 被全部兑换成了代币 Y，此时真实代币 $x = 0$。

代入边界条件解得：

$$x_v = \frac{L}{\sqrt{P_b}}, \quad y_v = L\sqrt{P_a}$$

因此，Uniswap V3 的**真实储备量与流动性深度的计算公式**为：

$$x = L \cdot \left( \frac{1}{\sqrt{P}} - \frac{1}{\sqrt{P_b}} \right)$$

$$y = L \cdot \left( \sqrt{P} - \sqrt{P_a} \right)$$

### 3.2 几何代数的核心：$\sqrt{P}$ 替代高精度浮点
为了消除 EVM 中极其昂贵的开方与浮点数运算，Uniswap V3 作出了一个天才设计：**在合约中不存储价格 $P$，也不存储代币数量，而是直接使用价格的平方根 $\sqrt{P}$**。

定义：

$$\Delta \sqrt{P} = \sqrt{P_{\text{new}}} - \sqrt{P_{\text{old}}}$$

当我们在同一个区间内兑换时，计算公式极其优雅，避免了除法：
- 存入代币 $y$ 所需的流动性：

$$\Delta y = L \cdot \Delta \sqrt{P}$$

- 存入代币 $x$ 所需的流动性：

$$\Delta x = L \cdot \Delta \left( \frac{1}{\sqrt{P}} \right)$$

---

## 4. Tick 机制与价格离散化

由于不同 LP 指定的区间五花八门，为了在链上高效合并这些流动性，V3 将价格区间离散化为一系列的 **Tick（价格刻度）**。

### 4.1 数学定义
以太坊上的 Tick 是一个一维整数轴 $i$，价格 $P$ 与 Tick 的指数关系定义为：

$$P(i) = 1.0001^i$$

- 每前进一步，价格变化约 $0.01\%$ (1 个基点)。
- **Tick 对应的 $\sqrt{P}$**：

$$\sqrt{P(i)} = 1.0001^{i/2}$$

```
Tick 轴: ───[Tick -100]──────[Tick 0]──────[Tick 100]───────→ (离散价格刻度)
```

在执行大额兑换时，如果价格穿过了某个 Tick，EVM 会像扫描线一样，触发该 Tick 上绑定的流动性激活或休眠，动态重新计算全网的流动性。

### 4.2 Uniswap V3 与 V2 开发核心选型
- **V2 适合开发**：长尾币种、流动性薄弱但交易频率低、需要低 Gas 开发成本的协议（V2 的 Swap Gas 开销极低，约 5-6 万 gas）。
- **V3 适合开发**：主流稳定币、主流大额蓝筹代币对（如 ETH/USDC，能利用集中流动性锁住 1000 倍的滑点深度，但每次跨越 Tick 消耗的 Gas 呈指数级上升）。
