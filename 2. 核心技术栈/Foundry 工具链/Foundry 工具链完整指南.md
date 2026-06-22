# Foundry 工具链完整指南

> Foundry 是以太坊开发工具的未来——用 Rust 编写，速度比 JavaScript 工具链快 10 倍以上，原生支持 Solidity 测试和 Fuzz 测试。

---

## 1. Foundry 是什么

Foundry 是一个**极速以太坊应用开发工具链**，由 Paradigm 于 2021 年推出，用 Rust 编写，包含四个核心工具：

| 工具 | 作用 | 类比 |
|------|------|------|
| **Forge** | 测试、编译、部署 | Hardhat（全部功能） |
| **Cast** | 链上交互 CLI | ethers.js CLI 版 |
| **Anvil** | 本地测试节点 | Ganache / Hardhat Node |
| **Chisel** | Solidity REPL | Node.js REPL |

### 1.1 为什么选择 Foundry

```
Hardhat 测试速度：           Foundry 测试速度：
  50 个测试 → 15 秒           50 个测试 → 0.3 秒
  500 个测试 → 150 秒         500 个测试 → 3 秒
  Fuzz 测试 → ❌ 不支持       Fuzz 测试 → ✅ 原生支持
```

| 维度 | Hardhat | Foundry |
|------|---------|---------|
| 编译速度 | 中等（solc native） | **极快（Rust + 并行编译）** |
| 测试速度 | 慢（JS → RPC → 链） | **极快（直接在 EVM 中执行）** |
| 测试语言 | JavaScript/TypeScript | **Solidity（原生）** |
| Fuzz 测试 | 需插件 | **✅ 原生支持** |
| 不变式测试 | 需插件 | **✅ 原生支持** |
| 依赖管理 | npm | **Git submodules** |
| 安装大小 | ~200MB (node_modules) | **~50MB（单二进制文件）** |

---

## 2. 安装

### 2.1 安装 Foundryup

```bash
# Linux / macOS
curl -L https://foundry.paradigm.xyz | bash

# 重新加载 shell
source ~/.bashrc  # 或 ~/.zshrc

# 安装 Foundry
foundryup

# 验证安装
forge --version   # forge 0.2.0
cast --version    # cast 0.2.0
anvil --version   # anvil 0.2.0
chisel --version  # chisel 0.2.0
```

### 2.2 更新 Foundry

```bash
# 更新到最新版
foundryup

# 更新到特定版本
foundryup --version nightly-...  # 指定 nightly
```

### 2.3 Windows 安装

```bash
# 方式1：WSL2（推荐）
# 在 WSL2 中按 Linux 方式安装

# 方式2：直接安装
# 下载 Release: https://github.com/foundry-rs/foundry/releases
```

---

## 3. 项目初始化

### 3.1 创建新项目

```bash
# 创建新项目
forge init my-dapp
cd my-dapp

# 项目结构
# my-dapp/
# ├── src/                    # 合约源码
# │   └── Counter.sol         # 示例合约
# ├── test/                   # 测试文件
# │   └── Counter.t.sol       # 示例测试
# ├── script/                 # 部署脚本
# │   └── Counter.s.sol       # 示例脚本
# ├── lib/                    # 依赖库（git submodules）
# │   └── forge-std/          # Foundry 标准库
# └── foundry.toml            # 配置文件
```

### 3.2 从现有目录初始化

```bash
# 在已有目录中初始化
cd existing-project
forge init --no-commit   # 不自动创建 git commit
```

### 3.3 使用模板

```bash
# 使用 OpenZeppelin 模板
forge init --template oz

# 使用 Solmate 模板
forge init --template solmate
```

---

## 4. foundry.toml 配置详解

```toml
# foundry.toml
[profile.default]
# ═══════════════════════════════════════════════════
# 路径配置
# ═══════════════════════════════════════════════════
src = "src"
out = "out"
test = "test"
script = "script"
libs = ["lib"]                     # 依赖库目录

# ═══════════════════════════════════════════════════
# 编译器配置
# ═══════════════════════════════════════════════════
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 200
evm_version = "paris"
via_ir = false                     # 启用 IR 优化管线（大合约可能需要）

# ═══════════════════════════════════════════════════
# 测试配置
# ═══════════════════════════════════════════════════
fuzz = { runs = 256 }              # Fuzz 测试运行次数
invariant = { runs = 256, depth = 15 }  # 不变式测试配置
verbosity = 2                      # 日志级别（0-5）

# ═══════════════════════════════════════════════════
# RPC 端点
# ═══════════════════════════════════════════════════
[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
arbitrum = "${ARBITRUM_RPC_URL}"
local = "http://127.0.0.1:8545"

# ═══════════════════════════════════════════════════
# Etherscan 验证
# ═══════════════════════════════════════════════════
[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }
arbitrum = { key = "${ARBISCAN_API_KEY}" }

# ═══════════════════════════════════════════════════
# 与 Hardhat 共存配置
# ═══════════════════════════════════════════════════
[profile.default.src]
# 如果合约在 Hardhat 的 contracts/ 目录
# src = "contracts"

# ═══════════════════════════════════════════════════
# CI 配置
# ═══════════════════════════════════════════════════
[profile.ci]
fuzz = { runs = 10000 }            # CI 中更多 fuzz 运行
invariant = { runs = 1000, depth = 50 }
```

---

## 5. Forge：编译、测试、部署

### 5.1 编译

```bash
# 编译所有合约
forge build
# 或
forge compile

# 强制重新编译
forge build --force

# 清除缓存
forge clean

# 查看编译信息
forge inspect Counter abi           # 查看 ABI
forge inspect Counter bytecode      # 查看字节码
forge inspect Counter storage-layout  # 查看存储布局
```

### 5.2 依赖管理

```bash
# 安装依赖（通过 Git submodules）
forge install foundry-rs/forge-std              # 标准库（init 时自动安装）
forge install OpenZeppelin/openzeppelin-contracts  # OpenZeppelin
forge install transmissions11/solmate            # Solmate
forge install Uniswap/v3-core                    # Uniswap V3

# 安装指定版本
forge install OpenZeppelin/openzeppelin-contracts@v5.0.0

# 更新依赖
forge update

# 移除依赖
forge remove openzeppelin-contracts
```

在 Solidity 中使用依赖：

```solidity
// OpenZeppelin
import "@openzeppelin-contracts/contracts/token/ERC20/ERC20.sol";

// Solmate
import "@solmate/src/tokens/ERC20.sol";

// Forge 标准库
import "forge-std/Test.sol";
```

> **注意**：Foundry 使用 Git submodules 管理依赖，依赖安装在 `lib/` 目录。import 路径取决于 `foundry.toml` 中的 `remappings` 配置（通常自动生成）。

```bash
# 查看自动生成的 remappings
forge remappings
# @openzeppelin-contracts/=lib/openzeppelin-contracts/
# @solmate/=lib/solmate/src/
# forge-std/=lib/forge-std/src/
```

```toml
# 也可以在 foundry.toml 中手动指定
remappings = [
  "@openzeppelin-contracts/=lib/openzeppelin-contracts/",
  "@solmate/=lib/solmate/src/",
]
```

### 5.3 测试——Forge 的核心

Forge 的测试用 Solidity 编写，直接在 EVM 中执行，速度极快。

#### 基本测试结构

```solidity
// test/Counter.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;
    address public owner = address(0x1);
    address public alice = address(0x2);

    // setUp 在每个测试前执行
    function setUp() public {
        // vm.prank 模拟下一次调用的 msg.sender
        vm.prank(owner);
        counter = new Counter();

        // 也可以先部署再用 prank
        // counter = new Counter();
    }

    // 以 test 开头的函数才会被执行
    function test_Increment() public {
        counter.increment();
        assertEq(counter.count(), 1);
    }

    function test_SetNumber() public {
        vm.prank(owner);
        counter.setNumber(42);
        assertEq(counter.count(), 42);
    }

    // 测试 revert
    function test_RevertWhen_NotOwner() public {
        vm.prank(alice);  // alice 不是 owner
        vm.expectRevert("Not owner");
        counter.setNumber(100);
    }

    // 测试事件
    function test_EmitEvent() public {
        vm.expectEmit(true, true, false, false);
        emit CountSet(owner, 42);
        vm.prank(owner);
        counter.setNumber(42);
    }

    event CountSet(address indexed setter, uint256 newCount);
}
```

#### Fuzz 测试

Fuzz 测试是 Foundry 的杀手级特性——自动生成随机输入，发现边界情况：

```solidity
contract CounterFuzzTest is Test {
    Counter public counter;

    function setUp() public {
        counter = new Counter();
    }

    // 参数名以随机值注入
    function testFuzz_SetNumber(uint256 x) public {
        vm.prank(counter.owner());
        counter.setNumber(x);
        assertEq(counter.count(), x);
    }

    // 多参数 fuzz
    function testFuzz_AddAndSubtract(uint256 a, uint256 b) public {
        vm.assume(a + b >= a);  // 排除溢出情况
        counter.setNumber(a);
        counter.increment();  // a + 1
        // ...
    }

    // 限制 fuzz 范围
    function testFuzz_SmallNumbers(uint8 x) public view {
        // uint8 范围: 0-255
        vm.assume(x > 0);
        vm.assume(x < 100);
        // x 的范围是 1-99
    }
}
```

> **`vm.assume()` vs `require()`**：`vm.assume()` 会丢弃不符合条件的输入并重新生成，不影响测试结果。`require()` 会导致测试失败。Fuzz 测试中应该用 `vm.assume()`。

#### 不变式测试（Invariant Testing）

不变式测试持续调用合约函数，验证系统的"不变式"始终成立：

```solidity
// test/invariants/TokenInvariant.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../../../src/SimpleToken.sol";
import "../../../src/TokenSale.sol";
import {Handler} from "./Handler.sol";

contract TokenInvariantTest is Test {
    SimpleToken public token;
    TokenSale public sale;
    Handler public handler;

    function setUp() public {
        token = new SimpleToken(1000000);
        sale = new TokenSale(100, address(this), address(token));
        handler = new Handler(token, sale);

        // 告诉 Foundry 调用 handler 中的函数
        targetContract(address(handler));
    }

    // 不变式：Token 的总供应量始终等于所有余额之和
    function invariant_TotalSupplyEqualsBalances() public view {
        uint256 total = token.totalSupply();
        uint256 sum = token.balanceOf(address(this))
            + token.balanceOf(address(sale))
            + token.balanceOf(address(handler));

        assertEq(total, sum);
    }

    // 不变式：TokenSale 永远不会拥有超过总供应量的 Token
    function invariant_SaleCannotExceedTotalSupply() public view {
        assertLe(token.balanceOf(address(sale)), token.totalSupply());
    }
}

// Handler：定义不变式测试可以调用的函数
contract Handler is Test {
    SimpleToken public token;
    TokenSale public sale;

    constructor(SimpleToken _token, TokenSale _sale) {
        token = _token;
        sale = _sale;
    }

    function buyTokens(uint256 amount) public {
        amount = bound(amount, 1, 1000);  // 限制范围
        // ... 模拟购买
    }

    function sellTokens(uint256 amount) public {
        amount = bound(amount, 1, 100);
        // ... 模拟出售
    }
}
```

#### 运行测试

```bash
# 运行所有测试
forge test

# 详细输出
forge test -vv

# 显示执行追踪
forge test -vvv

# 显示完整的执行追踪和 setup 追踪
forge test -vvvv

# 显示调试追踪（包括每个操作码）
forge test -vvvvv

# 运行匹配的测试
forge test --match-test test_Increment
forge test --match-contract CounterTest
forge test --match-path test/Counter.t.sol

# 只运行 fuzz 测试
forge test --match-test testFuzz

# 运行不变式测试
forge test --match-test invariant

# Gas 报告
forge test --gas-report

# 指定 fuzz 运行次数
forge test --fuzz-runs 1000

# 使用 CI 配置
forge test --profile ci
```

---

## 6. Forge Script：部署

### 6.1 编写部署脚本

```solidity
// script/DeploySimpleToken.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/SimpleToken.sol";

contract DeploySimpleToken is Script {
    function run() external returns (SimpleToken) {
        // 从环境变量读取参数
        uint256 initialSupply = vm.envUint("INITIAL_SUPPLY");

        // 开始广播——之后的操作会被记录为交易
        vm.startBroadcast();

        // 部署合约
        SimpleToken token = new SimpleToken(initialSupply);

        vm.stopBroadcast();

        return token;
    }
}
```

### 6.2 带依赖的部署

```solidity
// script/DeployTokenSale.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/SimpleToken.sol";
import "../src/TokenSale.sol";

contract DeployTokenSale is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        uint256 initialSupply = 1000000;
        uint256 rate = 100;

        vm.startBroadcast(deployerPrivateKey);

        // 1. 部署 Token
        SimpleToken token = new SimpleToken(initialSupply);
        console.log("Token deployed:", address(token));

        // 2. 部署 TokenSale
        TokenSale sale = new TokenSale(rate, msg.sender, address(token));
        console.log("TokenSale deployed:", address(sale));

        // 3. 转移 Token
        token.transfer(address(sale), 500000);
        console.log("Transferred 500000 tokens to TokenSale");

        vm.stopBroadcast();
    }
}
```

### 6.3 执行部署

```bash
# 模拟部署（dry-run，不发交易）
forge script script/DeploySimpleToken.s.sol --rpc-url $SEPOLIA_RPC_URL

# 实际部署
forge script script/DeploySimpleToken.s.sol \
  --rpc-url $SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast

# 部署并验证
forge script script/DeploySimpleToken.s.sol \
  --rpc-url $SEPOLIA_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify

# 部署到本地 Anvil
forge script script/DeploySimpleToken.s.sol \
  --rpc-url http://127.0.0.1:8545 \
  --private-key <ANVIL_PRIVATE_KEY> \
  --broadcast

# 使用 ledger 硬件钱包部署
forge script script/DeploySimpleToken.s.sol \
  --rpc-url $SEPOLIA_RPC_URL \
  --ledger \
  --broadcast
```

### 6.4 部署记录

```
broadcast/
└── DeploySimpleToken.s.sol/
    └── 11155111/              # 链 ID
        ├── run-latest.json    # 最新部署记录
        └── run-12345.json     # 历史部署记录
```

---

## 7. Cast：链上交互

Cast 是 Foundry 最强大的链上交互工具——几乎可以做 ethers.js 能做的一切。

### 7.1 读取数据（cast call）

```bash
# 读取 ERC-20 余额
cast call 0xA0b8... "balanceOf(address)(uint256)" 0x1234... --rpc-url $MAINNET_RPC_URL

# 读取 ERC-20 名称
cast call 0xA0b8... "name()(string)" --rpc-url $MAINNET_RPC_URL

# 读取 ERC-20 总供应量
cast call 0xA0b8... "totalSupply()(uint256)" --rpc-url $MAINNET_RPC_URL

# 获取 ETH 余额
cast balance 0x1234... --rpc-url $MAINNET_RPC_URL

# 获取 nonce
cast nonce 0x1234... --rpc-url $MAINNET_RPC_URL
```

### 7.2 发送交易（cast send）

```bash
# 转账 ETH
cast send 0x1234... --value 1ether --private-key $PRIVATE_KEY --rpc-url $SEPOLIA_RPC_URL

# 调用合约写入方法
cast send 0xA0b8... "transfer(address,uint256)" 0x5678... 100 --private-key $PRIVATE_KEY --rpc-url $SEPOLIA_RPC_URL

# Approve
cast send 0xA0b8... "approve(address,uint256)" 0xSpender 1000 --private-key $PRIVATE_KEY --rpc-url $SEPOLIA_RPC_URL
```

### 7.3 区块链信息

```bash
# 最新区块号
cast block-number --rpc-url $MAINNET_RPC_URL

# 最新区块信息
cast block latest --rpc-url $MAINNET_RPC_URL

# 指定区块
cast block 19000000 --rpc-url $MAINNET_RPC_URL

# 交易详情
cast tx 0xabcd... --rpc-url $MAINNET_RPC_URL

# 交易收据
cast receipt 0xabcd... --rpc-url $MAINNET_RPC_URL

# Gas 价格
cast gas-price --rpc-url $MAINNET_RPC_URL

# 估算 Gas
cast estimate 0xA0b8... "transfer(address,uint256)" 0x5678... 100 --rpc-url $SEPOLIA_RPC_URL
```

### 7.4 实用工具

```bash
# 地址转换
cast wallet address --private-key $PRIVATE_KEY

# 签名消息
cast wallet sign "Hello World" --private-key $PRIVATE_KEY

# 验证签名
cast wallet verify --address 0x1234... --message "Hello World" --signature 0xabcd...

# ABI 编码/解码
cast abi-encode "f(uint256,address)" 1 0x1234...
cast abi-decode "f(uint256,address)" 0x...

# keccak256 哈希
cast keccak "Hello World"

# 单位转换
cast to-wei 1 ether       # 1000000000000000000
cast from-wei 1000000000000000000 ether  # 1

# 十六进制转换
cast to-hex 42            # 0x2a
cast to-dec 0x2a          # 42

# 选择器计算
cast sig "transfer(address,uint256)"  # 0xa9059cbb

# 存储 slot 读取
cast storage 0xContract 0 --rpc-url $MAINNET_RPC_URL    # slot 0
cast storage 0xContract 1 --rpc-url $MAINNET_RPC_URL    # slot 1
```

---

## 8. Anvil：本地测试节点

### 8.1 启动 Anvil

```bash
# 默认启动（10 个账户，每个 10000 ETH）
anvil

# 指定端口
anvil --port 8545

# 指定账户数和余额
anvil --accounts 20 --balance 10000

# Fork 主网
anvil --fork-url https://mainnet.infura.io/v3/YOUR_KEY

# Fork 指定区块
anvil --fork-url https://mainnet.infura.io/v3/YOUR_KEY --fork-block-number 19000000

# 使用固定助记词（确定性地址）
anvil --mnemonic "test test test test test test test test test test test junk"

# 输出示例：
# Available Accounts
# ==================
# (0) 0xf39F... (10000 ETH)
# (1) 0x7099... (10000 ETH)
# ...
#
// Private Keys
// ==================
// (0) 0xac09...
// (1) 0x59c6...
// ...
```

### 8.2 Anvil vs Ganache vs Hardhat Node

| 特性 | Anvil | Ganache | Hardhat Node |
|------|-------|---------|-------------|
| 速度 | **极快（Rust）** | 慢 | 中等 |
| 主网 Fork | ✅ `--fork-url` | ✅ `--fork` | ✅ `--fork` |
| 自动挖矿 | ✅ 默认 | ✅ | ✅ |
| 间隔挖矿 | ✅ `--block-time` | ✅ | ✅ |
| 确定性地址 | ✅ `--mnemonic` | ✅ `--deterministic` | ✅ |
| Impersonate | ✅ `anvil_impersonateAccount` | ❌ | ✅ |
| 日志追踪 | ✅ | ❌ | ✅ |
| 持久化 | ⚠️ `--dump-state` | ✅ | ❌ |

---

## 9. Chisel：Solidity REPL

```bash
# 启动 Chisel
chisel

# 在 REPL 中写 Solidity
➜ uint256 x = 42;
➜ x + 8
Type: uint256
├ Hex: 0x32
└ Decimal: 50

➜ address(0).balance
Type: uint256
└ Decimal: 0

➜ keccak256("hello")
Type: bytes32
└ Hex: 0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a7a36deac8

# 内置命令
➜ :help     # 帮助
➜ :clear    # 清除所有变量
➜ :exec     # 执行当前会话为完整合约
➜ :quit     # 退出
```

---

## 10. 常用命令速查

| 命令 | 作用 |
|------|------|
| `forge init` | 初始化新项目 |
| `forge build` / `forge b` | 编译合约 |
| `forge test` / `forge t` | 运行测试 |
| `forge test -vvvv` | 详细测试输出 |
| `forge test --gas-report` | Gas 报告 |
| `forge test --match-test test_X` | 运行匹配的测试 |
| `forge script <path> --broadcast` | 部署合约 |
| `forge verify-contract <addr> <name>` | Etherscan 验证 |
| `forge install <repo>` | 安装依赖 |
| `forge update` | 更新依赖 |
| `forge inspect <contract> <field>` | 查看合约信息 |
| `forge flatten <file>` | 展平合约 |
| `forge snapshot` | Gas 快照（对比变化） |
| `forge coverage` | 代码覆盖率 |
| `forge doc` | 生成 NatSpec 文档 |
| `cast call <addr> <sig> <args>` | 读取合约 |
| `cast send <addr> <sig> <args>` | 发送交易 |
| `cast balance <addr>` | 查询 ETH 余额 |
| `cast block <number>` | 查询区块信息 |
| `cast tx <hash>` | 查询交易信息 |
| `cast wallet address` | 获取地址 |
| `anvil` | 启动本地节点 |
| `chisel` | Solidity REPL |

---

## 11. Cheatcode 速查（vm. 操作）

Cheatcode 是 Foundry 测试的核心——通过 `vm` 预编译合约操控 EVM 状态：

| Cheatcode | 作用 | 示例 |
|-----------|------|------|
| `vm.prank(addr)` | 下一次调用的 msg.sender | `vm.prank(alice); token.transfer(...)` |
| `vm.startPrank(addr)` | 持续设定 msg.sender | `vm.startPrank(alice); ... vm.stopPrank()` |
| `vm.deal(addr, amt)` | 设置地址余额 | `vm.deal(alice, 10 ether)` |
| `vm.expectRevert()` | 期待下一次调用 revert | `vm.expectRevert(); failingFunc()` |
| `vm.expectEmit()` | 期待事件 | `vm.expectEmit(); emit Event(...); func()` |
| `vm.warp(ts)` | 设置 block.timestamp | `vm.warp(2000000000)` |
| `vm.skip(time)` | 快进时间 | `vm.skip(30 days)` |
| `vm.roll(blockNum)` | 设置 block.number | `vm.roll(1000)` |
| `vm.expectRevert(msg)` | 期待特定 revert 消息 | `vm.expectRevert("Not owner")` |
| `vm.expectCustomError()` | 期待自定义错误 | `vm.expectRevert(abi.encodeWithSelector(CustomError.selector, ...))` |
| `vm.assume(cond)` | Fuzz 过滤条件 | `vm.assume(x > 0)` |
| `vm.bound(val, min, max)` | 限制 fuzz 范围 | `uint256 x = vm.bound(raw, 1, 100)` |
| `vm.snapshot()` | 保存状态快照 | `uint256 snap = vm.snapshot()` |
| `vm.revertTo(snap)` | 恢复状态快照 | `vm.revertTo(snap)` |
| `vm.mockCall(addr, ret)` | Mock 合约调用 | `vm.mockCall(addr, abi.encode(...), ret)` |
| `vm.expectCall(addr, data)` | 期待内部调用 | `vm.expectCall(addr, abi.encodeWithSelector(...))` |
| `vm.setStorageAt(addr, slot, val)` | 直接修改存储 | `vm.setStorageAt(addr, 0, bytes32(uint256(42)))` |
| `vm.label(addr, name)` | 给地址加标签（调试用） | `vm.label(alice, "Alice")` |
| `vm.getDeployedCode(addr)` | 获取已部署字节码 | `bytes memory code = vm.getDeployedCode(addr)` |

---

## 12. 与 Hardhat 共存

Foundry 和 Hardhat 可以在同一项目中使用——Foundry 做合约测试，Hardhat 做部署和前端集成：

```bash
# 在 Hardhat 项目中安装 Foundry
forge init --no-commit  # 不覆盖已有文件
```

```toml
# foundry.toml（共存配置）
[profile.default]
src = "contracts"          # 使用 Hardhat 的合约目录
out = "out"
test = "test/foundry"      # Foundry 测试放在独立目录
script = "script"
libs = ["node_modules", "lib"]

remappings = [
  "@openzeppelin/=node_modules/@openzeppelin/",
  "forge-std/=lib/forge-std/src/",
]

cache_path = "cache/foundry"  # 避免与 Hardhat 冲突
```

```
项目根目录/
├── contracts/              # 共享合约
├── test/
│   ├── foundry/            # ← Foundry Solidity 测试（fuzz/invariant）
│   │   ├── Token.t.sol
│   │   └── invariant/
│   └── hardhat/            # ← Hardhat JS/TS 测试（集成测试）
│       └── Token.test.ts
├── scripts/                # Hardhat 部署脚本
├── script/                 # Foundry 部署脚本
├── hardhat.config.ts       # Hardhat 配置
├── foundry.toml            # Foundry 配置
└── package.json
```

> **推荐分工**：用 Foundry 做合约层面的单元测试和 fuzz 测试，用 Hardhat 做端到端集成测试和部署管理。
