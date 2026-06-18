# Truffle 框架完整指南

> Truffle 是以太坊生态最早的成熟开发框架（2016 年由 Consensys 推出），曾长期占据统治地位。虽然已被 Consensys 于 2023 年正式弃用，但大量历史项目和教程仍在使用 Truffle。本文旨在帮助你**理解旧项目**、**维护遗留代码**，并为迁移到 Hardhat/Foundry 做准备。

---

## 1. Truffle 是什么

### 1.1 定位

Truffle 是一个**一站式以太坊开发框架**，提供：

- 合约编译（Solidity → 字节码 + ABI）
- 部署管理（migrations 迁移脚本）
- 测试框架（JavaScript + Solidity 双语言）
- 本地区块链（Ganache）
- 合约交互控制台
- 前端集成（Drizzle React 库）

### 1.2 核心组件

| 组件 | 作用 | 现状 |
|------|------|------|
| **Truffle** | 主体框架（编译、部署、测试） | ❌ 已弃用（2023） |
| **Ganache** | 本地区块链模拟器 | ⚠️ 仍可用，但推荐用 Hardhat Network / Anvil |
| **Drizzle** | React 前端数据同步 | ❌ 已弃用，推荐 wagmi + viem |
| **@truffle/hdwallet-provider** | HD Wallet 签名提供者 | ⚠️ 仍可用，独立维护 |

### 1.3 为什么还要学 Truffle

```
你接手了一个 2021 年的 DeFi 项目
    │
    ├── 合约用 Solidity 0.6.x 编写
    ├── 部署脚本用 Truffle migrations
    ├── 测试用 Truffle + Mocha
    ├── 前端用 Drizzle React
    └── 本地调试用 Ganache
    │
    ▼
你必须理解 Truffle 才能维护和迁移这个项目
```

---

## 2. 安装与项目初始化

### 2.1 安装

```bash
# 全局安装 Truffle CLI
npm install -g truffle

# 验证安装
truffle version
# Truffle v5.11.5 (core: 5.11.5)
# Ganache v7.9.1
# Solidity v0.8.20

# 安装 Ganache（本地区块链）
npm install -g ganache
```

> **Node.js 版本要求**：Truffle 5.x 需要 Node.js >= 14.0.0，推荐 16 或 18。

### 2.2 创建项目

```bash
# 创建空项目
mkdir my-dapp && cd my-dapp
truffle init

# 或使用 unbox 模板（类似 create-react-app）
truffle unbox react          # Truffle + React
truffle unbox metacoin        # 示例项目
truffle unbox drizzle         # Truffle + Drizzle React
```

### 2.3 项目结构

```
my-dapp/
├── contracts/              # Solidity 合约源码
│   ├── Migrations.sol      # 迁移管理合约（自动生成）
│   └── MyContract.sol      # 你的合约
├── migrations/             # 部署脚本
│   ├── 1_initial_migration.js   # 部署 Migrations.sol（自动生成）
│   └── 2_deploy_contracts.js    # 部署你的合约
├── test/                   # 测试文件
│   ├── mycontract.test.js       # JavaScript 测试
│   └── MyContract.test.sol      # Solidity 测试
├── build/                  # 编译输出（自动生成）
│   └── contracts/          # 编译后的 JSON（含 ABI + 字节码）
├── truffle-config.js       # 配置文件（网络、编译器等）
└── package.json
```

---

## 3. truffle-config.js 配置详解

这是 Truffle 项目的核心配置文件，理解它就理解了项目的网络和编译器设置。

```javascript
// truffle-config.js
module.exports = {
  // ═══════════════════════════════════════════════════
  // 网络配置
  // ═══════════════════════════════════════════════════
  networks: {
    // 开发网络（连接 Ganache）
    development: {
      host: "127.0.0.1",
      port: 7545,           // Ganache GUI 默认端口
      network_id: "*",      // 匹配任何网络 ID
    },

    // 命令行启动的 Ganache
    ganache_cli: {
      host: "127.0.0.1",
      port: 8545,           // Ganache CLI 默认端口
      network_id: 1337,
    },

    // Goerli 测试网（通过 Infura）
    goerli: {
      provider: () => new HDWalletProvider({
        privateKeys: [process.env.PRIVATE_KEY],
        providerOrUrl: `https://goerli.infura.io/v3/${process.env.INFURA_KEY}`,
        chainId: 5,
      }),
      network_id: 5,
      gas: 8000000,
      gasPrice: 20000000000,  // 20 gwei
      confirmations: 2,       // 等待 2 个区块确认
      timeoutBlocks: 200,
    },

    // 主网
    mainnet: {
      provider: () => new HDWalletProvider({
        mnemonic: process.env.MNEMONIC,
        providerOrUrl: `https://mainnet.infura.io/v3/${process.env.INFURA_KEY}`,
      }),
      network_id: 1,
      gas: 6000000,
      gasPrice: 30000000000,  // 30 gwei
      confirmations: 3,
    },

    // 本地 Hardhat 网络（兼容模式）
    hardhat: {
      host: "127.0.0.1",
      port: 8545,
      network_id: 31337,
    },
  },

  // ═══════════════════════════════════════════════════
  // 编译器配置
  // ═══════════════════════════════════════════════════
  compilers: {
    solc: {
      version: "0.8.19",            // 编译器版本
      docker: false,                // 是否使用 Docker 版 solc
      parser: "solcjs",             // 解析器：solcjs 或 native
      settings: {
        optimizer: {
          enabled: true,
          runs: 200,                // 优化运行次数
        },
        evmVersion: "paris",        // EVM 目标版本
      },
    },
  },

  // ═══════════════════════════════════════════════════
  // 合约目录配置
  // ═══════════════════════════════════════════════════
  contracts_directory: "./contracts",
  contracts_build_directory: "./build/contracts",
  migrations_directory: "./migrations",
  test_directory: "./test",

  // ═══════════════════════════════════════════════════
  // 插件
  // ═══════════════════════════════════════════════════
  plugins: [
    "truffle-plugin-verify",        // Etherscan 验证插件
    "truffle-contract-size",        // 合约大小检查
  ],

  // Etherscan API 配置（用于验证）
  api_keys: {
    etherscan: process.env.ETHERSCAN_API_KEY,
  },
};
```

### 关键配置说明

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `network_id: "*"` | 匹配任何网络（仅开发用） | - |
| `gas` | 部署交易的 Gas 上限 | 区块限制 |
| `gasPrice` | Gas 价格（wei） | 链上建议 |
| `confirmations` | 等待区块确认数 | 0 |
| `from` | 部署者地址（不指定则用 provider 第一个） | - |
| `optimizer.runs` | 优化次数（高=省运行Gas，低=省部署Gas） | 200 |

---

## 4. 合约编写与编译

### 4.1 合约编写

```solidity
// contracts/SimpleToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract SimpleToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("SimpleToken", "ST") {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }
}
```

### 4.2 安装依赖

```bash
# Truffle 使用 @openzeppelin/contracts 包
npm install @openzeppelin/contracts

# 配置文件中需要添加 contracts_directory
# truffle-config.js 已自动包含 node_modules/@openzeppelin/contracts
```

### 4.3 编译

```bash
# 编译所有合约
truffle compile

# 强制重新编译（不使用缓存）
truffle compile --all

# 输出示例：
# Compiling your contracts...
# ===========================
# > Compiling ./contracts/SimpleToken.sol
# > Compiling @openzeppelin/contracts/token/ERC20/ERC20.sol
# > Artifacts written to ./build/contracts
# > Compiled successfully using:
#    - solc: 0.8.19
```

### 4.4 编译产物

编译后每个合约生成一个 JSON 文件在 `build/contracts/` 目录：

```json
// build/contracts/SimpleToken.json
{
  "contractName": "SimpleToken",
  "abi": [...],                    // ABI 接口定义
  "bytecode": "0x608060...",      // 部署字节码
  "deployedBytecode": "0x6080...",// 运行时字节码
  "sourcePath": "contracts/SimpleToken.sol",
  "compiler": { "version": "0.8.19" },
  "networks": {                   // 各网络部署地址
    "5": {
      "events": {},
      "links": {},
      "address": "0x1234...",
      "transactionHash": "0xabcd..."
    }
  }
}
```

> **关键**：Truffle 的编译产物是完整 JSON（含 ABI + 字节码 + 部署地址），这与 Hardhat 不同（Hardhat 分成 artifacts 和 deployments）。

---

## 5. Migrations（迁移脚本）

Migration 是 Truffle 的部署系统——按编号顺序执行，记录部署状态。

### 5.1 Migrations.sol（自动生成）

```solidity
// contracts/Migrations.sol（Truffle 自动生成）
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Migrations {
    address public owner;
    uint256 public last_completed_migration;

    constructor() {
        owner = msg.sender;
    }

    modifier restricted() {
        require(msg.sender == owner, "Not authorized");
        _;
    }

    function setCompleted(uint256 completed) public restricted {
        last_completed_migration = completed;
    }
}
```

> 这个合约的作用是**记录最后一个执行的迁移脚本编号**，防止重复执行。

### 5.2 编写迁移脚本

```javascript
// migrations/1_initial_migration.js
const Migrations = artifacts.require("Migrations");

module.exports = function (deployer) {
  deployer.deploy(Migrations);
};
```

```javascript
// migrations/2_deploy_contracts.js
const SimpleToken = artifacts.require("SimpleToken");

module.exports = function (deployer, network, accounts) {
  // 参数：deployer（部署器）, network（网络名）, accounts（账户列表）

  console.log(`Deploying to network: ${network}`);
  console.log(`Deployer: ${accounts[0]}`);

  // 部署合约，传入 constructor 参数
  deployer.deploy(SimpleToken, 1000000);  // initialSupply = 1,000,000
};
```

### 5.3 高级迁移：带依赖的部署

```javascript
// migrations/3_deploy_token_sale.js
const SimpleToken = artifacts.require("SimpleToken");
const TokenSale = artifacts.require("TokenSale");

module.exports = async function (deployer, network, accounts) {
  // 先部署 Token
  await deployer.deploy(SimpleToken, 1000000);
  const token = await SimpleToken.deployed();

  // 再部署 TokenSale（需要 Token 地址作为参数）
  const rate = 100;  // 1 ETH = 100 Token
  const wallet = accounts[0];
  await deployer.deploy(TokenSale, rate, wallet, token.address);

  const tokenSale = await TokenSale.deployed();

  // 将 Token 的所有权转移给 TokenSale
  await token.transfer(tokenSale.address, 500000);

  console.log("Token:", token.address);
  console.log("TokenSale:", tokenSale.address);
};
```

### 5.4 按网络条件部署

```javascript
// migrations/4_deploy_contracts.js
const MyContract = artifacts.require("MyContract");

module.exports = async function (deployer, network, accounts) {
  if (network === "development") {
    // 开发网络：部署 mock 合约
    const MockOracle = artifacts.require("MockOracle");
    await deployer.deploy(MockOracle);
    const oracle = await MockOracle.deployed();
    await deployer.deploy(MyContract, oracle.address);
  } else if (network.startsWith("goerli")) {
    // 测试网：使用真实 Oracle 地址
    const oracleAddress = "0x...";
    await deployer.deploy(MyContract, oracleAddress);
  } else if (network === "mainnet") {
    // 主网：使用主网 Oracle 地址，额外检查
    const oracleAddress = "0x...";
    console.log("⚠️ 部署到主网！");
    console.log("确认信息：");
    console.log(`  Oracle: ${oracleAddress}`);
    console.log(`  Deployer: ${accounts[0]}`);
    // 可以在这里添加确认逻辑
    await deployer.deploy(MyContract, oracleAddress);
  }
};
```

### 5.5 部署命令

```bash
# 部署到开发网络（Ganache）
truffle migrate

# 部署到指定网络
truffle migrate --network goerli

# 从特定编号重新执行迁移
truffle migrate --f 2            # 从第 2 个脚本开始
truffle migrate --to 3            # 执行到第 3 个脚本

# 重置迁移状态（重新部署所有合约）
truffle migrate --reset

# 跳过 dry-run（直接部署，不模拟）
truffle migrate --skip-dry-run
```

---

## 6. 测试

Truffle 支持 JavaScript 和 Solidity 两种测试语言。

### 6.1 JavaScript 测试（Mocha + Chai）

```javascript
// test/SimpleToken.test.js
const { assert } = require("chai");
const SimpleToken = artifacts.require("SimpleToken");

contract("SimpleToken", (accounts) => {
  const [owner, alice, bob] = accounts;
  let token;

  beforeEach(async () => {
    // 每个测试前部署新合约
    token = await SimpleToken.new(1000000);
  });

  describe("Deployment", () => {
    it("should deploy with correct name", async () => {
      const name = await token.name();
      assert.equal(name, "SimpleToken");
    });

    it("should deploy with correct symbol", async () => {
      const symbol = await token.symbol();
      assert.equal(symbol, "ST");
    });

    it("should mint initial supply to deployer", async () => {
      const balance = await token.balanceOf(owner);
      // 注意：Truffle 返回 BN，需要用 toString 或比较
      assert.equal(balance.toString(), "1000000000000000000000000");
      // 1000000 * 10^18
    });
  });

  describe("Transfer", () => {
    it("should transfer tokens between accounts", async () => {
      await token.transfer(alice, 100);
      const aliceBalance = await token.balanceOf(alice);
      assert.equal(aliceBalance.toString(), "100");
    });

    it("should fail when sender has insufficient balance", async () => {
      // Truffle 用 try-catch 或 expectRevert
      try {
        await token.transfer(bob, 100, { from: alice });
        assert.fail("Should have thrown");
      } catch (error) {
        assert.include(error.message, "ERC20");
      }
    });

    it("should emit Transfer event", async () => {
      const result = await token.transfer(alice, 100);
      // Truffle 返回的 result 包含 logs
      const transferEvent = result.logs.find(
        (log) => log.event === "Transfer"
      );
      assert.exists(transferEvent);
      assert.equal(transferEvent.args.from, owner);
      assert.equal(transferEvent.args.to, alice);
      assert.equal(transferEvent.args.value.toString(), "100");
    });
  });

  describe("Allowance", () => {
    it("should approve and transferFrom", async () => {
      // owner 授权 alice
      await token.approve(alice, 200);
      const allowance = await token.allowance(owner, alice);
      assert.equal(allowance.toString(), "200");

      // alice 从 owner 转给 bob
      await token.transferFrom(owner, bob, 100, { from: alice });
      const bobBalance = await token.balanceOf(bob);
      assert.equal(bobBalance.toString(), "100");
    });
  });
});
```

### 6.2 Solidity 测试

```solidity
// test/SimpleToken_test.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "truffle/Assert.sol";
import "truffle/DeployedAddresses.sol";
import "../contracts/SimpleToken.sol";

contract TestSimpleToken {
    SimpleToken token = SimpleToken(DeployedAddresses.SimpleToken());

    function testInitialBalance() public {
        uint256 expected = 1000000 * 10 ** 18;
        Assert.equal(
            token.balanceOf(address(this)),
            expected,
            "Owner should have initial supply"
        );
    }

    function testTransfer() public {
        address recipient = address(0x1234);
        token.transfer(recipient, 100);
        Assert.equal(
            token.balanceOf(recipient),
            100,
            "Recipient should have 100 tokens"
        );
    }
}
```

> Solidity 测试的限制：无法使用 `beforeEach`（每次重新部署），只能测试已部署的合约。JavaScript 测试更灵活，推荐使用 JS 测试。

### 6.3 使用 OpenZeppelin Test Helpers

```javascript
// test/advanced.test.js
const { expectEvent, expectRevert, BN } = require("@openzeppelin/test-helpers");
const SimpleToken = artifacts.require("SimpleToken");

contract("SimpleToken", ([owner, alice]) => {
  let token;

  beforeEach(async () => {
    token = await SimpleToken.new(1000000);
  });

  it("should revert on insufficient balance", async () => {
    await expectRevert(
      token.transfer(alice, 999999999999),
      "ERC20InsufficientBalance"
    );
  });

  it("should emit Transfer event", async () => {
    const receipt = await token.transfer(alice, 100);
    expectEvent(receipt, "Transfer", {
      from: owner,
      to: alice,
      value: new BN("100"),
    });
  });
});
```

### 6.4 运行测试

```bash
# 运行所有测试
truffle test

# 指定网络
truffle test --network goerli

# 运行特定测试文件
truffle test test/SimpleToken.test.js

# 使用 ganache 作为测试网络
truffle test --network ganache_cli

# 输出示例：
# Using network 'development'.
# Compiling your contracts...
# ===========================
# > Compiling ./test/SimpleToken_test.sol
#
# Contract: SimpleToken
#   Deployment
//     ✓ should deploy with correct name (42ms)
//     ✓ should deploy with correct symbol (38ms)
//     ✓ should mint initial supply to deployer (51ms)
//   Transfer
//     ✓ should transfer tokens between accounts (65ms)
//     ✓ should fail when sender has insufficient balance (23ms)
//     ✓ should emit Transfer event (44ms)
//   Allowance
//     ✓ should approve and transferFrom (72ms)
//
// 7 passing (2s)
```

---

## 7. Ganache 本地区块链

### 7.1 启动 Ganache

```bash
# 方式1：命令行启动
ganache
# 启动后输出 10 个测试账户，每个 100 ETH

# 指定端口和 Gas 上限
ganache --port 8545 --gasLimit 12000000 --deterministic

# 方式2：Ganache GUI（可视化界面）
# 下载：https://trufflesuite.com/ganache/
# 点击 "Quickstart" 即可

# --deterministic：使用固定的助记词和账户
# 这样每次启动的账户地址相同，方便调试
```

### 7.2 Ganache 的特性

| 特性 | 说明 |
|------|------|
| 即时交易 | 无需等待区块时间 |
| 固定 Gas 价格 | 默认 1 gwei |
| 可视化（GUI 版） | 查看交易、合约、区块、事件 |
| 可挖矿控制 | 可手动挖矿或自动挖矿 |
| Fork 主网 | `ganache --fork https://mainnet.infura.io/v3/...` |
| 模拟时间 | 可控制 `block.timestamp` |

```bash
# Fork 主网（在本地模拟主网状态）
ganache --fork https://mainnet.infura.io/v3/YOUR_KEY --fork-block-number 18000000

# 这样你可以在本地操作主网上的真实合约
# 例如：在本地 fork 的 Uniswap 上测试交易
```

### 7.3 在 Truffle 中使用 Ganache

```javascript
// truffle-config.js
module.exports = {
  networks: {
    development: {
      host: "127.0.0.1",
      port: 7545,     // Ganache GUI
      network_id: "*",
    },

    // 如果用命令行 ganache
    ganache_cli: {
      host: "127.0.0.1",
      port: 8545,
      network_id: 1337,
    },
  },
};
```

---

## 8. 部署到测试网和主网

### 8.1 配置 HDWalletProvider

```bash
# 安装 HDWalletProvider
npm install @truffle/hdwallet-provider

# 安装 dotenv（管理环境变量）
npm install dotenv
```

```javascript
// truffle-config.js
require("dotenv").config();
const HDWalletProvider = require("@truffle/hdwallet-provider");

module.exports = {
  networks: {
    // 私钥方式
    goerli: {
      provider: () =>
        new HDWalletProvider({
          privateKeys: [process.env.PRIVATE_KEY],
          providerOrUrl: `https://goerli.infura.io/v3/${process.env.INFURA_KEY}`,
          chainId: 5,
        }),
      network_id: 5,
      gas: 8000000,
      gasPrice: 20000000000,  // 20 gwei
    },

    // 助记词方式（不推荐，安全性低）
    mainnet: {
      provider: () =>
        new HDWalletProvider({
          mnemonic: process.env.MNEMONIC,
          providerOrUrl: `https://mainnet.infura.io/v3/${process.env.INFURA_KEY}`,
        }),
      network_id: 1,
      gas: 6000000,
      gasPrice: 30000000000,  // 30 gwei
    },

    // Sepolia 测试网
    sepolia: {
      provider: () =>
        new HDWalletProvider({
          privateKeys: [process.env.PRIVATE_KEY],
          providerOrUrl: `https://sepolia.infura.io/v3/${process.env.INFURA_KEY}`,
          chainId: 11155111,
        }),
      network_id: 11155111,
      gas: 8000000,
    },
  },
};
```

```bash
# .env 文件
PRIVATE_KEY=0xabc123...
INFURA_KEY=your_infura_key
ETHERSCAN_API_KEY=your_etherscan_key
```

### 8.2 部署流程

```bash
# 1. 启动 Ganache（开发用）
ganache

# 2. 编译合约
truffle compile

# 3. 部署到本地
truffle migrate

# 4. 部署到 Goerli 测试网
truffle migrate --network goerli

# 5. 部署到主网（确认无误后）
truffle migrate --network mainnet --skip-dry-run
```

### 8.3 在 Etherscan 上验证合约

```bash
# 安装验证插件
npm install truffle-plugin-verify

# 在 truffle-config.js 中添加插件
# plugins: ["truffle-plugin-verify"]

# 验证合约
truffle run verify SimpleToken --network goerli

# 批量验证
truffle run verify --network goerli
```

---

## 9. Truffle Console 与合约交互

### 9.1 交互控制台

```bash
# 启动控制台（连接开发网络）
truffle console

# 连接指定网络
truffle console --network goerli

# 在控制台中：
truffle(development)> const token = await SimpleToken.deployed()
truffle(development)> const name = await token.name()
truffle(development)> console.log(name)
# SimpleToken

truffle(development)> const balance = await token.balanceOf(accounts[0])
truffle(development)> console.log(balance.toString())
# 1000000000000000000000000

truffle(development)> await token.transfer(accounts[1], 1000)
truffle(development)> const bal1 = await token.balanceOf(accounts[1])
truffle(development)> console.log(bal1.toString())
# 1000
```

### 9.2 常用控制台命令

```javascript
// 查看所有可用合约
truffle(development)> SimpleToken

// 查看已部署合约地址
truffle(development)> SimpleToken.address

// 查看所有账户
truffle(development)> accounts

// 查看网络 ID
truffle(development)> network

// 查看网络信息
truffle(development)> web3.eth.net.getId()
truffle(development)> web3.eth.getBlock("latest")

// 直接发送交易
truffle(development)> web3.eth.sendTransaction({from: accounts[0], to: accounts[1], value: web3.utils.toWei("1", "ether")})
```

---

## 10. 前端集成

### 10.1 在 JavaScript 中使用 Truffle Artifact

```javascript
// frontend/src/contract.js
import Web3 from "web3";
import SimpleToken from "./build/contracts/SimpleToken.json";

// 获取合约实例
export async function getTokenContract() {
  const web3 = new Web3(window.ethereum);
  await window.ethereum.enable();

  const networkId = await web3.eth.net.getId();
  const deployedNetwork = SimpleToken.networks[networkId];

  if (!deployedNetwork) {
    throw new Error("Contract not deployed on this network");
  }

  const contract = new web3.eth.Contract(
    SimpleToken.abi,
    deployedNetwork.address
  );

  return contract;
}

// 调用合约方法
export async function getTokenBalance(address) {
  const token = await getTokenContract();
  const balance = await token.methods.balanceOf(address).call();
  return balance;
}

// 发送交易
export async function transferToken(to, amount, from) {
  const token = await getTokenContract();
  const result = await token.methods.transfer(to, amount).send({ from });
  return result;
}
```

### 10.2 使用 @truffle/contract（更简洁的 API）

```javascript
// 使用 Truffle 自带的 contract 抽象
import contract from "@truffle/contract";
import SimpleTokenArtifact from "./build/contracts/SimpleToken.json";

const SimpleToken = contract(SimpleTokenArtifact);

// 设置 provider
SimpleToken.setProvider(window.ethereum);

// 使用方式更接近 Truffle Console
async function interact() {
  const token = await SimpleToken.deployed();
  const balance = await token.balanceOf(accounts[0]);
  console.log(balance.toString());

  // 发送交易
  const result = await token.transfer(accounts[1], 100, { from: accounts[0] });
  console.log(result.tx);  // 交易哈希
  console.log(result.logs); // 事件日志
}
```

### 10.3 Drizzle（已弃用，仅了解）

```javascript
// Drizzle 是 Truffle 配套的 React 数据同步库
// 已弃用，现代项目请用 wagmi + viem

import { Drizzle } from "@drizzle/store";
import SimpleToken from "./contracts/SimpleToken.json";

const drizzleOptions = {
  contracts: [SimpleToken],
};

const drizzle = new Drizzle(drizzleOptions);

// 在 React 组件中使用
function Balance({ drizzle, drizzleState }) {
  const [balance, setBalance] = useState("0");

  useEffect(() => {
    const contract = drizzle.contracts.SimpleToken;
    const dataKey = contract.methods.balanceOf.cacheCall(
      drizzleState.accounts[0]
    );
    setBalance(contract.balanceOf[dataKey]);
  }, []);

  return <div>Balance: {balance}</div>;
}
```

---

## 11. 常用命令速查

| 命令 | 作用 |
|------|------|
| `truffle init` | 初始化新项目 |
| `truffle unbox <template>` | 使用模板创建项目 |
| `truffle compile` | 编译合约 |
| `truffle compile --all` | 强制重新编译 |
| `truffle migrate` | 部署合约（开发网络） |
| `truffle migrate --network <name>` | 部署到指定网络 |
| `truffle migrate --reset` | 重新部署所有合约 |
| `truffle test` | 运行所有测试 |
| `truffle test <file>` | 运行指定测试文件 |
| `truffle console` | 启动交互控制台 |
| `truffle console --network <name>` | 连接指定网络的控制台 |
| `truffle run verify <contract>` | 在 Etherscan 验证合约 |
| `truffle run contract-size` | 查看合约字节码大小 |
| `truffle watch` | 监听文件变化并自动编译 |
| `truffle debug <tx-hash>` | 调试交易 |
| `truffle flattener <file>` | 展平合约（用于验证） |

---

## 12. Truffle 的历史地位与局限性

### 12.1 Truffle 的贡献

Truffle 在 2016-2021 年间是以太坊开发的事实标准，它首创了：
- **Migration 部署系统**（按编号顺序执行部署脚本）
- **Artifact JSON 格式**（ABI + 字节码 + 部署信息一体化）
- **Ganache 本地链**（可视化开发环境）
- **双语言测试**（JS + Solidity）

### 12.2 为什么被弃用

| 问题 | 说明 |
|------|------|
| **速度慢** | 编译用 solcjs（比 native solc 慢 10x+），Foundry 用 Rust |
| **配置繁琐** | truffle-config.js 复杂，网络配置需手动写 HDWalletProvider |
| **测试慢** | JS 测试需通过 web3.js 与链通信，Foundry 直接在 EVM 中测试 |
| **生态停滞** | 插件少、更新慢，Hardhat 生态活跃得多 |
| **TypeScript 支持差** | 原生 JS，TS 支持需额外配置 |
| **不支持 Forge 风格的 fuzz 测试** | 无法做属性测试 |

### 12.3 迁移路线

```
Truffle (已弃用)
    │
    ├─→ Hardhat（JavaScript/TypeScript 生态，渐进式迁移）
    │
    └─→ Foundry（Rust 工具链，最高性能，原生 fuzz 测试）
```

> 详细的迁移指南参见 [Truffle 迁移指南](./Truffle%20迁移指南.md)。
