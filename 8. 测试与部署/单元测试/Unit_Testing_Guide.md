# 单元测试深度解析与 Foundry/Hardhat 双框架实战

> 在智能合约开发中，没有被测试覆盖的代码等同于“存在重大潜在漏洞的代码”。因为合约部署后不可篡改，单元测试是保障合约逻辑在极端输入、角色越权和异常边界下依然保持完备自洽的唯一防线。本指南将针对现代黄金标准 **Foundry（Solidity 原生测试）** 以及经典主流 **Hardhat（TS/JS 异步测试）** 双框架，详解核心测试断言、高级欺骗（Cheatcodes）环境操控，并提供高保真的实战测试代码模板。

---

## 1. 测试金字塔中的“单元测试”
单元测试（Unit Testing）要求对智能合约中的**最小可测试单元（如单个纯函数、单一角色的逻辑流）**进行完全隔离测试。
- **单元测试的要求**：运行速度极快（数毫秒级）、不依赖真实的公链网络、能够模拟出任意极端的块高度、时间和账户余额。

---

## 2. 现代黄金标准：Foundry Solidity 原生测试

**Foundry** 是目前全球最受欢迎的以太坊开发测试工具。由于其测试脚本采用 **纯 Solidity** 编写，免去了 JS/TS 繁琐的异步操作（`async/await`）与类型转换，且运行效率高出 Hardhat 数十倍。

### 2.1 基础测试框架结构
在 Foundry 中，所有的测试合约均应继承 `forge-std/Test.sol`。其测试流包含三大核心步骤：
1. **`setUp()`**：初始化函数。测试执行前，Forge 虚拟机会在后台优先执行此函数，用于部署被测合约、分配初始流动性。
2. **`test...()`**：正常的测试用例（以 `test` 关键字开头）。
3. **`testFail...()`**：断言发生失败的测试用例（测试执行中只要发生 Revert，该测试即判定为通过）。

### 2.2 核心 Cheatcodes（虚拟机控制欺骗）
Foundry 提供了极强大的 `vm` 欺骗机制（Cheatcodes），允许开发者直接对 EVM 虚拟机的底层状态进行强行修改：

- **`vm.prank(address)` / `vm.startPrank(address)`**：改变下一个调用的 `msg.sender`。主要用于测试多角色（如非管理员）的权限越权边界。
- **`vm.deal(address, uint256)`**：强行注入指定地址的 Native ETH 余额。
- **`vm.warp(uint256)`**：强行篡改当前的区块时间戳（`block.timestamp`），常用于测试 Staking 到期领奖或 Timelock 时间限制。
- **`vm.roll(uint256)`**：强行篡改当前的区块高度（`block.number`）。
- **`vm.expectRevert(bytes)`**：断言紧随其后的下一条交易**必定发生 Revert**（支持原生自定义 Error 的断言）。
- **`vm.expectEmit(bool, bool, bool, bool)`**：断言紧随其后的交易中必定触发指定的事件。四个 Boolean 分别对应 Indexed 主题参数 1、2、3 及 Data 数据。

---

### 2.3 Foundry 生产级单元测试实战

#### 被测合约 (NestingStore.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract NestingStore {
    address public owner;
    mapping(address => uint256) public deposits;

    error NotOwner();
    error InsufficientBalance();

    event Deposited(address indexed user, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner();
        _;
    }

    function deposit() external payable {
        deposits[msg.sender] += msg.value;
        emit Deposited(msg.sender, msg.value);
    }

    function withdraw(uint256 _amount) external {
        if (deposits[msg.sender] < _amount) revert InsufficientBalance();
        deposits[msg.sender] -= _amount;
        
        (bool success, ) = msg.sender.call{value: _amount}("");
        require(success, "ETH transfer failed");

        emit Withdrawn(msg.sender, _amount);
    }

    // 仅限管理员清退全部资金
    function emergencySweep() external onlyOwner {
        payable(owner).transfer(address(this).balance);
    }
}
```

#### 🧪 Foundry 测试脚本 (NestingStore.t.sol)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "./NestingStore.sol";

contract NestingStoreTest is Test {
    NestingStore public store;
    
    // 模拟测试账户
    address public alice = address(0x1111);
    address public bob = address(0x2222);

    /**
     * @notice 初始化部署
     */
    function setUp() public {
        store = new NestingStore(); // 部署被测合约，此时 owner 默认为 NestingStoreTest
    }

    /**
     * @notice 测试 1：基本的存款逻辑与余额记录
     */
    function testDeposit() public {
        // (a) 利用 deal 注入 Alice 10 ETH 余额
        vm.deal(alice, 10 ether);

        // (b) 模拟 alice 发送交易
        vm.prank(alice);
        store.deposit{value: 5 ether}();

        // (c) 验证余额账本
        assertEq(store.deposits(alice), 5 ether);
        assertEq(address(store).balance, 5 ether);
    }

    /**
     * @notice 测试 2：事件抛出检测
     */
    function testDepositEventEmit() public {
        vm.deal(alice, 10 ether);

        // 告知虚拟机我们要精确校验 Deposited 事件，校验 indexed 主题 1 (即 alice)，不校验主题 2 和 3，校验 Data（即 5 ETH）
        vm.expectEmit(true, false, false, true);
        emit NestingStore.Deposited(alice, 5 ether); // 写入期望触发的事件特征

        // 执行真实动作
        vm.prank(alice);
        store.deposit{value: 5 ether}();
    }

    /**
     * @notice 测试 3：取款溢出拦截与自定义错误断言
     */
    function testWithdrawRevertInsufficient() public {
        vm.deal(alice, 1 ether);

        vm.prank(alice);
        store.deposit{value: 1 ether}();

        // 期望下一条交易抛出自定义错误 InsufficientBalance()
        vm.expectRevert(NestingStore.InsufficientBalance.selector);
        
        vm.prank(alice);
        store.withdraw(2 ether); // alice 试图取款 2 ETH，应被拦截
    }

    /**
     * @notice 测试 4：非 owner 越权清退防御测试
     */
    function testEmergencySweepFailNonOwner() public {
        // 期望下一条交易抛出 NotOwner() 报错
        vm.expectRevert(NestingStore.NotOwner.selector);
        
        // 伪装非 owner 角色 bob
        vm.prank(bob);
        store.emergencySweep();
    }
}
```

---

## 3. 经典主流：Hardhat TypeScript + Chai 测试

尽管 Foundry 日渐崛起，但在许多成熟的既有项目中，**Hardhat** 依旧是统治级的主流。它基于 JavaScript 生态的 Mocha 测试框架与 Chai 匹配验证器。

### 3.1 核心加速黑科技：`loadFixture`
- **传统痛点**：如果每一个测试用例（`it`）在执行前都调用一次 deploy 重新部署部署合约，会导致极大的时间开销。
- **快照方案**：Hardhat 提供了 **`loadFixture`**。当首次执行部署后，Hardhat 节点会强制在后台截取当前区块链状态的**快照（Snapshot）**。当后续 `it` 运行时，合约会瞬间将状态一键重置到快照点，**免除了多次重复编译和部署的代码，使测试速度提升 10x**。

### 3.2 TS 生产级单元测试实战

#### 🧪 Hardhat 测试脚本 (NestingStore.test.ts)
```typescript
import { expect } from "chai";
import { ethers } from "hardhat";
import { loadFixture } from "@nomicfoundation/hardhat-toolbox/network-helpers";

describe("NestingStore Hardhat 单元测试", function () {
  // 定义 Fixture（部署快照）
  async function deployStoreFixture() {
    const [owner, alice, bob] = await ethers.getSigners();

    const NestingStoreFactory = await ethers.getContractFactory("NestingStore");
    const store = await NestingStoreFactory.deploy(); // 部署

    return { store, owner, alice, bob };
  }

  it("应该允许用户正常存款并抛出 Deposited 事件", async function () {
    const { store, alice } = await loadFixture(deployStoreFixture);

    const depositAmount = ethers.parseEther("1.5");

    // Alice 充值，并链式校验事件 Deposited
    await expect(
      store.connect(alice).deposit({ value: depositAmount })
    )
      .to.emit(store, "Deposited")
      .withArgs(alice.address, depositAmount);

    // 校验账本
    expect(await store.deposits(alice.address)).to.equal(depositAmount);
  });

  it("余额不足取款时应该抛出 InsufficientBalance 自定义错误", async function () {
    const { store, alice } = await loadFixture(deployStoreFixture);

    // 存入 1 ETH
    await store.connect(alice).deposit({ value: ethers.parseEther("1.0") });

    // 取款 2 ETH，期望抛出自定义错误
    await expect(
      store.connect(alice).withdraw(ethers.parseEther("2.0"))
    ).to.be.revertedWithCustomError(store, "InsufficientBalance");
  });

  it("非 Owner 清退资金应该拒绝越权", async function () {
    const { store, bob } = await loadFixture(deployStoreFixture);

    // bob 试图越权扫币，应抛出 NotOwner 错误
    await expect(
      store.connect(bob).emergencySweep()
    ).to.be.revertedWithCustomError(store, "NotOwner");
  });
});
```

---

## 4. Foundry 与 Hardhat 在单元测试维度的抉择

当架构一个新的 Web3 项目时，可以根据下表选择单元测试主力框架：

| 选型维度 | Foundry 🧪 (推荐) | Hardhat 👷 |
| :--- | :--- | :--- |
| **测试编写语言** | 纯 Solidity（0 门槛上手） | TypeScript / JavaScript |
| **运行执行速度** | **极快**（Rust 底层编译，无需 JS 虚拟机交互） | 较慢（依赖 node.js 加载 Ethers.js） |
| **测试报错提示** | 极强（包含全套 call trace 报错行追踪） | 较强（需依赖 `console.log` 与 Hardhat 网络报错） |
| **虚拟机欺骗机制**| **丰富**（包含 50+ 个底层的 `vm.` cheatcode） | 较少（需要通过 `helpers` 插件或 RPC 发送 raw 调动） |
| **依赖生态管理** | 使用 Git submodules 或者是 npm 进行引入 | 使用 npm / pnpm 软件包安装引入 |
| **首选适用场景** | 高频迭代、数学密集型 DeFi 协议、10k NFT | 大型企业级全栈 DApp、包含大量前端 SDK 交互测试的项目 |
