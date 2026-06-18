# Truffle 迁移指南

> Truffle 已被 Consensys 于 2023 年正式弃用。如果你在维护一个 Truffle 项目，本文提供完整的迁移到 Hardhat 或 Foundry 的路径。

---

## 1. 为什么要迁移

### 1.1 Truffle 弃用时间线

```
2016  Truffle 发布，成为以太坊开发事实标准
2019  Hardhat 发布，开始蚕食 Truffle 市场份额
2021  Foundry 发布，以 Rust 性能和原生 fuzz 测试颠覆市场
2023  Consensys 宣布弃用 Truffle 和 Ganache
2024+ Truffle 不再接收安全更新，所有新项目应使用 Hardhat/Foundry
```

### 1.2 迁移的核心收益

| 维度 | Truffle | Hardhat | Foundry |
|------|---------|---------|---------|
| 编译速度 | 慢（solcjs） | 中等（solc native） | **快（Rust + solc native）** |
| 测试速度 | 慢（JS → RPC → 链） | 中等（JS → Hardhat Network） | **快（直接在 EVM 中）** |
| 测试语言 | JS + Solidity | **JS/TS + Solidity** | **Solidity（原生）** |
| Fuzz 测试 | ❌ 不支持 | ⚠️ 需插件 | **✅ 原生支持** |
| TypeScript | ⚠️ 需额外配置 | ✅ 原生支持 | ✅ 原生支持 |
| 插件生态 | 停滞 | **活跃** | 活跃 |
| 控制台 | Truffle Console | **Hardhat Console** | Cast CLI |
| 本地链 | Ganache | **Hardhat Network** | Anvil |
| 部署脚本 | Migrations（编号顺序） | **Scripts（灵活控制）** | Forge Script |

### 1.3 迁移还是不迁移

```
你的项目是否需要迁移？
    │
    ├── 新项目 → 直接用 Hardhat 或 Foundry，不用 Truffle
    │
    ├── 小型项目，已部署，不再迭代 → 可以不迁移，但不要用新 Truffle 功能
    │
    ├── 中型项目，仍在迭代 → 推荐迁移到 Hardhat（迁移成本最低）
    │
    └── 大型项目，需要高性能测试 → 迁移到 Foundry（长期收益最大）
```

---

## 2. 迁移到 Hardhat

Hardhat 是迁移成本最低的选择——同为 JavaScript 生态，概念映射直接。

### 2.1 项目结构映射

```
Truffle                          Hardhat
───────                          ───────
contracts/                  →    contracts/
migrations/                 →    scripts/               ← 概念变化
test/                       →    test/
build/contracts/            →    artifacts/             ← 编译产物位置
truffle-config.js           →    hardhat.config.js      ← 配置文件
```

### 2.2 配置文件转换

**Truffle 配置：**

```javascript
// truffle-config.js
const HDWalletProvider = require("@truffle/hdwallet-provider");
require("dotenv").config();

module.exports = {
  networks: {
    development: {
      host: "127.0.0.1",
      port: 8545,
      network_id: "*",
    },
    goerli: {
      provider: () =>
        new HDWalletProvider({
          privateKeys: [process.env.PRIVATE_KEY],
          providerOrUrl: `https://goerli.infura.io/v3/${process.env.INFURA_KEY}`,
        }),
      network_id: 5,
      gas: 8000000,
    },
  },
  compilers: {
    solc: {
      version: "0.8.19",
      settings: {
        optimizer: { enabled: true, runs: 200 },
      },
    },
  },
};
```

**Hardhat 等效配置：**

```javascript
// hardhat.config.js
require("@nomicfoundation/hardhat-toolbox");
require("dotenv").config();

/** @type import('hardhat/config').HardhatUserConfig */
module.exports = {
  solidity: {
    version: "0.8.19",
    settings: {
      optimizer: { enabled: true, runs: 200 },
    },
  },
  networks: {
    // Hardhat 内置开发网络，无需配置 Ganache
    hardhat: {},

    // 连接外部 Ganache（迁移过渡期）
    ganache: {
      url: "http://127.0.0.1:8545",
      chainId: 1337,
    },

    goerli: {
      url: `https://goerli.infura.io/v3/${process.env.INFURA_KEY}`,
      accounts: [process.env.PRIVATE_KEY],   // 直接传私钥数组
      gas: 8000000,
    },

    mainnet: {
      url: `https://mainnet.infura.io/v3/${process.env.INFURA_KEY}`,
      accounts: [process.env.PRIVATE_KEY],
    },
  },
};
```

**关键变化：**

| 配置项 | Truffle | Hardhat |
|--------|---------|---------|
| 网络提供者 | `HDWalletProvider`（复杂） | `accounts: [privateKey]`（简单） |
| 开发网络 | 需启动 Ganache | 内置 Hardhat Network（自动启动） |
| solc 版本 | `compilers.solc.version` | `solidity.version` |
| 多版本 solc | ❌ 不支持 | ✅ 支持多版本混合编译 |

### 2.3 部署脚本转换

**Truffle Migration：**

```javascript
// migrations/2_deploy_contracts.js
const SimpleToken = artifacts.require("SimpleToken");

module.exports = async function (deployer, network, accounts) {
  await deployer.deploy(SimpleToken, 1000000);
  const token = await SimpleToken.deployed();
  console.log("Token deployed:", token.address);
};
```

**Hardhat Script：**

```javascript
// scripts/deploy.js
const { ethers } = require("hardhat");

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying with:", deployer.address);

  const SimpleToken = await ethers.getContractFactory("SimpleToken");
  const token = await SimpleToken.deploy(1000000);
  await token.waitForDeployment();     // 等待部署完成

  console.log("Token deployed:", await token.getAddress());
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

**关键区别：**

| 概念 | Truffle | Hardhat |
|------|---------|---------|
| 合约引用 | `artifacts.require("Name")` | `ethers.getContractFactory("Name")` |
| 部署方式 | `deployer.deploy(Contract, args)` | `Contract.deploy(args)` |
| 获取地址 | `Contract.address` | `await contract.getAddress()` |
| 等待部署 | 自动等待 | `await contract.waitForDeployment()` |
| 执行顺序 | 文件名编号（1_, 2_, 3_） | 手动控制（命令行参数） |
| 合约实例 | `Contract.deployed()` | `ethers.getContractAt("Name", addr)` |

### 2.4 测试转换

**Truffle 测试：**

```javascript
// test/SimpleToken.test.js (Truffle)
const { assert } = require("chai");
const SimpleToken = artifacts.require("SimpleToken");

contract("SimpleToken", (accounts) => {
  const [owner, alice] = accounts;
  let token;

  beforeEach(async () => {
    token = await SimpleToken.new(1000000);
  });

  it("should transfer", async () => {
    await token.transfer(alice, 100);
    const balance = await token.balanceOf(alice);
    assert.equal(balance.toString(), "100");
  });
});
```

**Hardhat 测试（ethers.js v6）：**

```javascript
// test/SimpleToken.test.js (Hardhat)
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleToken", function () {
  let token;
  let owner, alice;

  beforeEach(async () => {
    // Hardhat 获取签名者（替代 accounts）
    [owner, alice] = await ethers.getSigners();

    // 部署合约（替代 artifacts.require + .new()）
    const SimpleToken = await ethers.getContractFactory("SimpleToken");
    token = await SimpleToken.deploy(1000000);
    await token.waitForDeployment();
  });

  it("should transfer", async () => {
    // 使用 ethers.js API（替代 Truffle Contract API）
    await token.transfer(alice.address, 100);
    const balance = await token.balanceOf(alice.address);
    expect(balance.toString()).to.equal("100");
  });

  it("should emit Transfer event", async () => {
    // Hardhat 事件测试更简洁
    await expect(token.transfer(alice.address, 100))
      .to.emit(token, "Transfer")
      .withArgs(owner.address, alice.address, 100);
  });
});
```

**核心 API 映射表：**

| 操作 | Truffle | Hardhat (ethers.js) |
|------|---------|---------------------|
| 获取账户 | `accounts` 参数 | `await ethers.getSigners()` |
| 部署合约 | `Contract.new(args)` | `Factory.deploy(args)` |
| 已部署合约 | `Contract.deployed()` | `ethers.getContractAt(name, addr)` |
| 调用方法 | `token.transfer(to, amt)` | `token.transfer(to, amt)` |
| 指定调用者 | `token.transfer(to, amt, {from: alice})` | `token.connect(alice).transfer(to, amt)` |
| 获取余额 | `token.balanceOf(addr)` | `token.balanceOf(addr)` |
| 读取地址 | `token.address` | `await token.getAddress()` |
| BN 处理 | `balance.toString()` | `balance.toString()` |
| 事件测试 | `result.logs.find(...)` | `.to.emit(contract, "Event")` |
| 期待失败 | try-catch | `await expect(...).to.be.revertedWith(...)` |

### 2.5 前端代码转换

**Truffle 前端：**

```javascript
// Truffle 前端
import contract from "@truffle/contract";
import SimpleTokenArtifact from "./build/contracts/SimpleToken.json";

const SimpleToken = contract(SimpleTokenArtifact);
SimpleToken.setProvider(window.ethereum);
const token = await SimpleToken.deployed();
const balance = await token.balanceOf(account);
```

**Hardhat 前端（ethers.js v6）：**

```javascript
// Hardhat 前端
import { ethers } from "ethers";
import SimpleTokenArtifact from "./artifacts/contracts/SimpleToken.sol/SimpleToken.json";

const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();
const token = new ethers.Contract(
  deployedAddress,
  SimpleTokenArtifact.abi,
  signer
);
const balance = await token.balanceOf(account);
```

> **关键变化**：Truffle 的 artifact JSON 在 `build/contracts/`，Hardhat 的 artifact 在 `artifacts/contracts/`。前端只需要 ABI，可以直接从 artifact 中提取。

---

## 3. 迁移到 Foundry

Foundry 是更激进的迁移——从 JavaScript 生态完全切换到 Rust + Solidity 生态。

### 3.1 项目结构映射

```
Truffle                          Foundry
───────                          ───────
contracts/                  →    src/
migrations/                 →    script/
test/                       →    test/
truffle-config.js           →    foundry.toml
package.json                →    （不需要，用 forge）
```

### 3.2 配置文件转换

**Truffle → Foundry（foundry.toml）：**

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
test = "test"
script = "script"

solc_version = "0.8.19"
optimizer = true
optimizer_runs = 200
evm_version = "paris"

# 对应 Truffle 的 networks 配置
[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
goerli = "${GOERLI_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"

[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
goerli = { key = "${ETHERSCAN_API_KEY}" }
```

### 3.3 部署脚本转换

**Truffle Migration → Foundry Script：**

```solidity
// script/DeploySimpleToken.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";
import "../src/SimpleToken.sol";

contract DeploySimpleToken is Script {
    function run() external returns (SimpleToken) {
        // 开始广播（记录交易）
        vm.startBroadcast();

        // 部署合约
        SimpleToken token = new SimpleToken(1000000);

        vm.stopBroadcast();

        console.log("Token deployed:", address(token));
        return token;
    }
}
```

```bash
# 执行部署脚本
# 本地
forge script script/DeploySimpleToken.s.sol --broadcast

# 测试网
forge script script/DeploySimpleToken.s.sol \
  --rpc-url $GOERLI_RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --verify
```

### 3.4 测试转换

**Truffle JS 测试 → Foundry Solidity 测试：**

```solidity
// test/SimpleToken.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "../src/SimpleToken.sol";

contract SimpleTokenTest is Test {
    SimpleToken token;
    address owner = address(this);
    address alice = address(0x1);

    function setUp() public {
        token = new SimpleToken(1000000);
    }

    function testTransfer() public {
        token.transfer(alice, 100);
        assertEq(token.balanceOf(alice), 100);
    }

    function testInsufficientBalance() public {
        // Foundry 的 expectRevert 替代 try-catch
        vm.expectRevert();
        token.transfer(alice, 999999999999);
    }

    // ✅ Fuzz 测试——Truffle 完全不支持
    function testFuzzTransfer(uint256 amount) public {
        vm.assume(amount <= token.balanceOf(owner));
        token.transfer(alice, amount);
        assertEq(token.balanceOf(alice), amount);
    }

    // ✅ 不变式测试
    function invariantTotalSupplyConserved() public {
        assertEq(
            token.balanceOf(owner) + token.balanceOf(alice),
            token.totalSupply()
        );
    }
}
```

**API 映射表：**

| 操作 | Truffle | Foundry |
|------|---------|---------|
| 部署合约 | `Contract.new(args)` | `new Contract(args)` |
| 获取地址 | `token.address` | `address(token)` |
| 期待 revert | `try { } catch (e) { }` | `vm.expectRevert()` |
| 设置余额 | ❌ | `vm.deal(addr, amt)` / `deal(addr, amt)` |
| 切换调用者 | `{from: alice}` | `vm.prank(alice)` / `vm.startPrank(alice)` |
| 快进时间 | ❌ | `vm.warp(timestamp)` / `skip(time)` |
| 快进区块 | ❌ | `vm.roll(blockNum)` / `roll(blockNum)` |
| Mock 合约 | 部署 mock | `vm.mockCall(addr, ret)` |
| Fuzz 测试 | ❌ | `function testFuzz_X(uint256 input)` |
| 获取事件 | `result.logs` | `vm.expectEmit()` |

---

## 4. 逐步迁移实战流程

### 4.1 第一阶段：双工具并行（低风险）

```bash
# 1. 安装 Hardhat（不删除 Truffle）
cd my-truffle-project
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# 2. 创建 hardhat.config.js
cat > hardhat.config.js << 'EOF'
require("@nomicfoundation/hardhat-toolbox");
module.exports = {
  solidity: "0.8.19",
  networks: {
    hardhat: {},
    ganache: {
      url: "http://127.0.0.1:8545",
      chainId: 1337,
    },
  },
};
EOF

# 3. 用 Hardhat 编译（验证合约能编译通过）
npx hardhat compile

# 4. 旧 Truffle 测试仍可用，新测试用 Hardhat
# 逐步将 test/ 中的测试迁移到 Hardhat 格式
```

### 4.2 第二阶段：迁移测试

```bash
# 1. 逐个迁移测试文件
# 优先迁移核心合约的测试

# 2. 对照 API 映射表转换：
#    - artifacts.require → ethers.getContractFactory
#    - accounts → ethers.getSigners
#    - contract → describe
#    - assert → expect

# 3. 验证迁移后的测试通过
npx hardhat test

# 4. 所有测试通过后删除旧 Truffle 测试
rm test/SimpleToken.test.js  # 旧 Truffle 测试
```

### 4.3 第三阶段：迁移部署脚本

```bash
# 1. 创建 scripts/ 目录
mkdir -p scripts

# 2. 为每个 migration 创建对应的 deploy script
# migrations/2_deploy_contracts.js → scripts/deploy.js

# 3. 验证部署
npx hardhat run scripts/deploy.js

# 4. 测试网部署验证
npx hardhat run scripts/deploy.js --network goerli
```

### 4.4 第四阶段：清理

```bash
# 1. 删除 Truffle 特有文件
rm -rf migrations/ build/ truffle-config.js
rm -rf test/*_test.sol      # Solidity 测试（Truffle 格式）

# 2. 卸载 Truffle 依赖
npm uninstall truffle @truffle/hdwallet-provider @truffle/contract

# 3. 更新 package.json 脚本
# package.json
{
  "scripts": {
    "compile": "hardhat compile",
    "test": "hardhat test",
    "deploy": "hardhat run scripts/deploy.js",
    "deploy:goerli": "hardhat run scripts/deploy.js --network goerli"
  }
}

# 4. 更新 README 和文档
```

---

## 5. 常见迁移坑点

### 5.1 合约路径问题

```javascript
// Truffle：合约名直接引用，不关心路径
const Token = artifacts.require("SimpleToken");

// Hardhat：需要正确配置路径
// 默认在 contracts/ 目录下查找
// 如果有子目录，hardhat.config.js 中配置：
module.exports = {
  paths: {
    sources: "./contracts",
  },
};
```

### 5.2 OpenZeppelin 导入路径

```solidity
// Truffle 和 Hardhat 都使用 npm 包，路径一致
// 但 Truffle 有时需要在 truffle-config.js 中额外配置：

// Truffle 可能需要：
// contracts_directory: "./contracts",
// 但通常 @openzeppelin/contracts 自动被 Truffle 发现

// Hardhat 自动处理 npm 包中的合约
// 无需额外配置
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
```

### 5.3 编号变量与地址

```javascript
// Truffle 返回 BN 对象
const balance = await token.balanceOf(addr);
balance.toString();   // "1000000000000000000000000"
balance.toNumber();   // ⚠️ 大数会丢失精度！

// Hardhat（ethers.js v6）返回 BigInt
const balance = await token.balanceOf(addr);
balance.toString();   // "1000000000000000000000000"
// 没有 toNumber()，必须用 formatUnits 转换
ethers.formatUnits(balance, 18);  // "1000000.0"
```

### 5.4 事件监听

```javascript
// Truffle：事件在 logs 数组中
const result = await token.transfer(alice, 100);
const event = result.logs.find(l => l.event === "Transfer");
const { from, to, value } = event.args;

// Hardhat + ethers.js v6：使用 queryFilter 或 expect
const filter = token.filters.Transfer;
const events = await token.queryFilter(filter);
// 或在测试中：
await expect(token.transfer(alice, 100))
  .to.emit(token, "Transfer")
  .withArgs(owner.address, alice.address, 100);
```

### 5.5 多重 solc 版本

```javascript
// Truffle：只能用一个版本
compilers: { solc: { version: "0.8.19" } }

// Hardhat：支持多版本混合
solidity: {
  compilers: [{ version: "0.8.19" }],
  overrides: {
    "contracts/old/LegacyContract.sol": {
      version: "0.6.12",   // 旧合约用旧编译器
      settings: { optimizer: { enabled: true, runs: 200 } },
    },
  },
}
```

### 5.6 Ganache → Hardhat Network 差异

| 特性 | Ganache | Hardhat Network |
|------|---------|-----------------|
| 启动方式 | 独立进程 | 内置（自动启动） |
| 区块时间 | 即时 | 默认即时（可设为自动挖矿） |
| 模拟主网 Fork | `--fork` 参数 | `npx hardhat node --fork` |
| `console.log` | ❌ | ✅（Hardhat 独有） |
| 时间操纵 | API 调用 | `time.increase()` 或 `evm_increaseTime` |
| 快照 | API | `evm_snapshot` / `evm_revert` |

---

## 6. 迁移检查清单

- [ ] **合约编译**：`npx hardhat compile` 通过，无错误
- [ ] **测试覆盖**：所有 Truffle 测试已迁移到 Hardhat 测试，全部通过
- [ ] **部署脚本**：每个 migration 都有对应的 deploy script
- [ ] **测试网部署**：在测试网上成功部署，合约地址验证通过
- [ ] **Etherscan 验证**：合约在 Etherscan 上验证通过
- [ ] **前端集成**：前端使用新的 ABI 文件路径，连接正常
- [ ] **CI/CD 更新**：GitHub Actions 或其他 CI 脚本更新为 Hardhat 命令
- [ ] **文档更新**：README 中安装和运行命令更新
- [ ] **清理 Truffle**：删除 truffle-config.js、migrations/、build/ 目录
- [ ] **卸载依赖**：`npm uninstall truffle @truffle/hdwallet-provider`

---

## 7. 工具链选择决策树

```
你的团队情况？
    │
    ├── 纯 JS/TS 团队，不熟悉 Rust
    │   └── Hardhat（学习成本最低，生态最成熟）
    │
    ├── 有 Rust 经验，或愿意学习
    │   └── Foundry（性能最高，fuzz 测试原生支持）
    │
    ├── 混合团队，各有偏好
    │   └── Hardhat + Foundry（可以共存！）
    │       ├── Foundry 做合约测试（fuzz + invariant）
    │       └── Hardhat 做部署脚本和前端集成
    │
    └── 极简主义，只想学一个工具
        └── Foundry（一个工具覆盖编译+测试+部署+交互）
```

> **当前社区趋势**：新项目中 Foundry 占比已超过 Hardhat，尤其在 DeFi 领域。但 Hardhat 仍是 JavaScript 生态的首选。两者可以共存于同一个项目。
