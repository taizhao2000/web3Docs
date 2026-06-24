# 自动化审计工具实战与代码检查清单

> 在 Web3 世界中，仅靠人工肉眼审计是远远不够的。引入成熟的静态分析、符号执行和模糊测试等自动化安全检测工具，能够帮助团队在开发与编译阶段抢先拦截 90% 的已知漏洞。本指南将深度探讨以太坊主流审计工具（Slither、Mythril）的实战运行、Foundry 的 Fuzz/Invariant 自动化边界测试方法，并提供一份涵盖 30+ 核心漏洞的智能合约安全终极自查清单。

---

## 1. 静态分析工具：Slither 实战

**Slither** 是由 Trail of Bits 开发的一款高性能 Solidity 静态分析框架。它底层通过将 Solidity 代码解析为抽象语法树（AST），并转化为名为 **SlithIR** 的单静态赋值（SSA）中间语言，从而实现对合约控制流图（CFG）和数据依赖性的多重扫描。

### 1.1 核心优势与检出能力
Slither 可以在几秒钟内跑完数百个内置的检测器（Detectors），能够快速发现：
- Checks-Effects-Interactions (CEI) 规则违背（重入风险）
- 未加 `initializer` 锁的代理升级漏洞
- 状态变量写权限缺失与影子变量
- 废弃的 Solidity 语法或全局变量错误使用（如 `tx.origin`）

### 1.2 生产级运行实战
在 Hardhat 或 Foundry 项目根目录下运行极简命令：

```bash
# 1. 扫描整个项目
slither .

# 2. 仅扫描特定合约，排除第三方依赖以减少噪音
slither contracts/MyContract.sol --exclude-dependencies

# 3. 自动生成项目的控制流图 (CFG)
slither contracts/MyContract.sol --print cfg
```

---

## 2. 符号执行工具：Mythril 实战

**Mythril** 是由 ConsenSys 开发的、针对 EVM 字节码的安全分析工具。它通过 **符号执行（Symbolic Execution）**、**污点分析（Taint Analysis）** 以及 **控制流分析**，不需要任何真实的输入，即可自动化建立并求解状态机分支约束，从而寻找能够导致代码发生崩溃或权限越权的路径。

### 2.1 适用场景
Mythril 极其适合检测底层的复杂死锁、权限绕过、整数溢出、未捕获的转账异常，特别是能检测人工由于逻辑盲区很难想到的“多步骤交叉重入和套扣攻击”。

### 2.2 实战命令
```bash
# 1. 直接分析已编译的 Solidity 文件 (分析最大深度设置为 22)
myth analyze contracts/VulnerableStore.sol --max-depth 22

# 2. 分析特定的已上链合约地址
myth analyze --address 0x1234567890123456789012345678901234567890 --rpc infura-mainnet
```

---

## 3. 模糊测试（Fuzzing）与不变式测试（Invariant Testing）实战

传统单元测试中，我们习惯输入硬编码的测试边界（如 `testTransfer(10 ether)`）。但这无法拦截黑客通过输入边缘极端数据（如 `2**256 - 1`）引起的算术坍塌。为此，必须在开发中深度集成 **Foundry 模糊/不变式测试**。

### 3.1 什么是模糊测试 (Fuzz Testing)？
Foundry 默认支持参数化 Fuzz 测试。只要在测试函数中声明输入参数，Foundry 就会在每次测试时**随机生成并注入 256 组甚至上千组不同的边缘数值**，帮助开发者撞击出未被 `require` 防御住的溢出或精度逻辑漏洞。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../../MyToken.sol";

contract FuzzTest is Test {
    MyToken token;

    function setUp() public {
        token = new MyToken();
    }

    /**
     * @notice 参数化 Fuzz：测试任何随机数额的铸币逻辑
     * @dev Foundry 自动在底层对 amount 注入随机的 256 组数据测试边界
     */
    function testFuzzMint(uint256 amount) public {
        // 使用 vm.assume 过滤掉非法的极大输入（边界截断防护）
        vm.assume(amount > 0 && amount < 100_000_000 * 1e18);
        
        token.mint(address(0x123), amount);
        assertEq(token.balanceOf(address(0x123)), amount);
    }
}
```

### 3.2 高阶状态不变式测试 (Invariant Testing) & Handler 模式
**不变式（Invariants）** 是指整个合约系统在**任何时候、经历任何多次的、随机的交叉交易（存款、借款、取款、清算）后，必须永远保持为真（True）的数学真理**。

- **不变式案例**：在借贷池中，`池子当前现金 Cash + 正在计息的总债务 Borrows` 永远应当大于或等于 `存款总凭证 cToken 的 Underlying 换算值`。
- **Handler 模式**：由于随机调用极易产生无效交易，Foundry 推荐编写 `Handler` 协议哨兵合约，用于过滤掉无谓的异常，将随机操作严格限制在合法的“存款、取款”调用范围内，使 Invariant 模糊扫描的覆盖深度和检出效率翻倍。

---

## 4. 终极智能合约安全审计检查清单 (Audit Checklist)

下面是全网最完备的智能合约安全自查清单，供团队在提交外部专业审计公司前，进行 100% 的内部通关核对：

### 📋 分类自查红线表

#### I. 初始化与合约部署安全
- [ ] **[ ] 是否禁用了逻辑合约初始化？** 逻辑合约的 `constructor` 必须显式声明 `_disableInitializers()`，防止黑客对逻辑合约进行 initializer 注入从而进行 `selfdestruct` 销毁。
- [ ] **[ ] 升级合约是否包含 `constructor` 变量赋值？** 升级/代理合约的所有初值必须写在 `initialize()` 函数中，不能在状态变量声明时直接赋值（例如：`uint256 x = 100;` 跨 Proxy 会失效）。
- [ ] **[ ] 构造函数中是否有空地址（zero address）校验？** 部署传入的所有核心地址和多签钱包地址必须进行 `require(addr != address(0))` 拦截。

#### II. 权限与访问控制安全
- [ ] **[ ] 所有敏感修改函数是否都有正确的 `onlyOwner` 或权限修饰器？** 确保所有改变费率、修改配置、提取平台分成的方法没有被遗忘挂载权限修饰器。
- [ ] **[ ] 是否禁用了 `tx.origin` 进行所有权鉴权？** 必须用且仅用 `msg.sender` 防范外部钓鱼合约在 `fallback` 里代为调用。
- [ ] **[ ] 角色控制（AccessControl）是否有两阶段所有权移交？** 避免直接 `transferOwnership` 到一个打错字母的死地址。应采用两阶段所有权移交（第一阶段：`proposeOwner`；第二阶段：由新 Owner 本人调用 `claimOwnership` 确认认领）。

#### III. 数学计算与精度下溢
- [ ] **[ ] 整数除法是否先乘后除？** 在计算费率或价格时，为防止精度被截断清零，必须在除法前先进行乘法扩大（例如：`(amount * feeBps) / 10000`）。
- [ ] **[ ] `unchecked` 块内是否混入了用户可输入的变量？** 确保 unchecked 只用于自增计数器等确定绝对不会发生溢出的封闭循环环境。
- [ ] **[ ] 汇率或价格的分母是否可能为 `0`？** 在执行 `getUtilizationRate` 或 cToken 兑换时，必须防范 `Total Supply == 0` 或 `totalSupply` 被外部故意赎回归零导致全合约除零死锁。

#### IV. 资产转账与重入安全
- [ ] **[ ] 是否绝对遵循了 Checks-Effects-Interactions (CEI) 规范？** 在向外部转账前，确保所有的 user balance 清零及状态变更动作在代码行上先执行完毕。
- [ ] **[ ] 关键函数是否全局加装了 `nonReentrant` 保护锁？** 所有提取、赎回、清算方法，一律加挂重入拦截修饰器。
- [ ] **[ ] 转账是否校验了返回值？** 使用 ERC20 代币转账时，应强制使用 OpenZeppelin 的 `SafeERC20` 库（如 `safeTransfer` / `safeTransferFrom`），防范类似 USDT 等没有正常返回 Boolean 值代币导致交易意外挂死。
- [ ] **[ ] Native ETH 转账是否防范了 Gas 限制？** 避免使用 `transfer()` 和 `send()`（它们只携带固定的 2300 Gas），应统一使用 `call{value: ...}("")`。

#### V. 外部依赖与预言机安全
- [ ] **[ ] 是否避免了使用 AMM 池的即时 Spot Price（现货价格）进行喂价？** 绝对不能使用 DEX 的瞬时价格。必须接入 Chainlink 去中心化多源喂价。
- [ ] **[ ] Chainlink 喂价是否校验了“是否停机/脏数据”？** 读取 Chainlink 价格数据时，必须检查 `updatedAt`（更新时间戳间隔，防停机价格不更新）和 `answer > 0`。
- [ ] **[ ] 重大操作是否加入了滑点保护（Slippage Protection）？** 存款和交易购买时，必须允许用户指定最差可接受的 `minAmountOut` 或 `maxAmountIn`，拒绝由夹子机器人（Sandwich/Front-running）带来的价格滑损。

#### VI. 逻辑边界与极端状态
- [ ] **[ ] 循环变量上限是否有限制？** 合约中 `for` 循环遍历的数组长度绝对不能无限制增长（不能让用户任意 deposit 从而使数组无限膨胀），否则会因为遍历消耗的 Gas 超过单区块 Gas Limit 导致提现/领奖被彻底锁死。
- [ ] **[ ] 敏感计算是否存在时间戳依赖？** 避免将 `block.timestamp` 用于生成彩票等强随机数（容易被矿工在区块生成时提前试错操纵价格），应一律接入 **Chainlink VRF**。
