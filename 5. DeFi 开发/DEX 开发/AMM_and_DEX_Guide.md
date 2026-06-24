# AMM 核心原理与 DEX 开发深度指南

> 去中心化交易所（DEX）是整个 DeFi 生态的流动性枢纽。本指南深度解析恒定乘积自动化做市商（AMM）的数学模型、价格影响、滑点计算，以及 Uniswap V3 集中流动性（Concentrated Liquidity）的硬核几何数学推导。

---

## 1. 恒定乘积做市商 (Constant Product AMM)

以 Uniswap V2 为代表的经典 AMM，其核心基于极其简单的**恒定乘积公式**：

$$x \cdot y = k$$

- $x$: 池中代币 X 的储备量（Reserve）
- $y$: 池中代币 Y 的储备量（Reserve）
- $k$: 恒定乘积常数（在无流动性添加/移除时保持不变）

### 1.1 无手续费下的交易公式（零滑点底噪）
假设用户输入 $\Delta x$ 个代币 X，想要换出 $\Delta y$ 个代币 Y，根据恒定乘积公式，池中储备量发生改变：

$$(x + \Delta x) \cdot (y - \Delta y) = k$$

将 $k = x \cdot y$ 代入：

$$(x + \Delta x) \cdot (y - \Delta y) = x \cdot y$$

通过简单的代数整理，推导得出**输出代币 $\Delta y$ 的计算公式**：

$$\Delta y = \frac{y \cdot \Delta x}{x + \Delta x}$$

### 1.2 带交易手续费（如 0.3%）的生产级公式
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
- $x_v$, $y_v$: 虚拟代币储备，使得当价格达到边界 $P_a$ 或 $P_b$ 时，真实的代币储备恰好耗尽。

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
