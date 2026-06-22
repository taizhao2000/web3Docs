# Hardhat 开发环境完整指南

> Hardhat 是目前 JavaScript/TypeScript 生态中最成熟的以太坊开发框架。本文覆盖从安装到生产部署的完整工作流。

---

## 1. Hardhat 是什么

Hardhat 是一个**专业的以太坊开发环境**，用 TypeScript 编写，提供：

| 功能 | 说明 | 对比 Truffle |
|------|------|-------------|
| 编译 | 自动检测变更、增量编译 | ✅ 更快，支持多版本 solc |
| 测试 | Mocha + Chai + ethers.js | ✅ 原生 TypeScript，console.log 调试 |
| 部署 | 脚本化部署 | ✅ 更灵活，无编号约束 |
| 本地链 | 内置 Hardhat Network | ✅ 无需启动 Ganache |
| 调试 | Solidity 堆栈追踪 | ✅ 错误信息精确到源码行 |
| 主网 Fork | 内置支持 | ✅ 无需额外工具 |

---

## 2. 安装与项目初始化

### 2.1 环境要求

- Node.js >= 18.0.0（推荐 20 LTS）
- npm >= 9.0.0 或 pnpm

### 2.2 创建项目

```bash
# 创建项目目录
mkdir my-dapp && cd my-dapp

# 初始化 npm 项目
npm init -y

# 安装 Hardhat
npm install --save-dev hardhat

# 初始化 Hardhat 项目
npx hardhat init
# 选择 "Create a TypeScript project"（推荐）
# 或 "Create a JavaScript project"
```

### 2.3 TypeScript 项目结构

```
my-dapp/
├── contracts/                    # Solidity 合约源码
│   └── Lock.sol                 # 示例合约
├── scripts/                     # 部署脚本
│   └── deploy.ts
├── test/                        # 测试文件
│   └── Lock.test.ts
├── ignition/                    # Hardhat Ignition 部署模块
│   └── modules/
│       └── LockModule.ts
├── hardhat.config.ts             # 核心配置文件
├── tsconfig.json                # TypeScript 配置
├── package.json
└── .env                         # 环境变量（私钥、API Key）
```

> **Hardhat Ignition** 是 Hardhat 3.x 新引入的声明式部署系统，替代传统脚本式部署。本文同时覆盖两种方式。

### 2.4 安装常用依赖

```bash
# 工具包（包含 ethers.js、chai、hardhat-network-helpers 等）
npm install --save-dev @nomicfoundation/hardhat-toolbox

# 环境变量管理
npm install --save-dev dotenv

# OpenZeppelin 合约库
npm install @openzeppelin/contracts

# 代理升级支持（如需可升级合约）
npm install --save-dev @openzeppelin/hardhat-upgrades
```

---

## 3. hardhat.config.ts 配置详解

这是 Hardhat 项目的核心——理解配置就理解了整个工作流。

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from "hardhat/config";
import "@nomicfoundation/hardhat-toolbox";
import "@openzeppelin/hardhat-upgrades";
import "dotenv/config";

const config: HardhatUserConfig = {
  // ═══════════════════════════════════════════════════
  // Solidity 编译器配置
  // ═══════════════════════════════════════════════════
  solidity: {
    version: "0.8.24",

    // 多版本编译（兼容旧合约）
    compilers: [
      { version: "0.8.24" },
      { version: "0.6.12" },
      { version: "0.4.24" },
    ],

    // 覆盖特定合约的编译器版本
    overrides: {
      "contracts/legacy/LegacyToken.sol": {
        version: "0.6.12",
        settings: {
          optimizer: { enabled: true, runs: 200 },
        },
      },
    },

    settings: {
      optimizer: {
        enabled: true,
        runs: 200,        // 优化运行次数
        // runs 高 → 运行时省 Gas（如 DEX 合约）
        // runs 低 → 部署时省 Gas（如一次性部署的合约）
      },
      evmVersion: "paris",   // EVM 目标版本
      viaIR: false,           // 启用 IR 优化管线（大合约可能需要）
    },
  },

  // ═══════════════════════════════════════════════════
  // 网络配置
  // ═══════════════════════════════════════════════════
  networks: {
    // 内置开发网络（默认，无需配置）
    hardhat: {
      // 主网 Fork（在本地模拟主网状态）
      forking: {
        url: `https://mainnet.infura.io/v3/${process.env.INFURA_KEY}`,
        enabled: false,     // 默认关闭，按需开启
      },
      chainId: 31337,
      // 模拟账户余额
      accounts: {
        accountsBalance: "10000000000000000000000", // 10000 ETH
      },
    },

    // 本地节点（npx hardhat node）
    localhost: {
      url: "http://127.0.0.1:8545",
    },

    // Sepolia 测试网
    sepolia: {
      url: `https://sepolia.infura.io/v3/${process.env.INFURA_KEY}`,
      accounts: [process.env.PRIVATE_KEY!],  // 直接传私钥
      chainId: 11155111,
      gas: "auto",
      gasPrice: "auto",
      gasMultiplier: 1.1,    // Gas 乘数（防止低估）
    },

    // 主网
    mainnet: {
      url: `https://mainnet.infura.io/v3/${process.env.INFURA_KEY}`,
      accounts: [process.env.PRIVATE_KEY!],
      chainId: 1,
      gas: 6000000,
      gasPrice: 30000000000,  // 30 gwei
    },

    // Arbitrum One（L2 示例）
    arbitrum: {
      url: `https://arb-mainnet.g.alchemy.com/v2/${process.env.ALCHEMY_KEY}`,
      accounts: [process.env.PRIVATE_KEY!],
      chainId: 42161,
    },
  },

  // ═══════════════════════════════════════════════════
  // Etherscan 验证配置
  // ═══════════════════════════════════════════════════
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY!,
      sepolia: process.env.ETHERSCAN_API_KEY!,
      arbitrumOne: process.env.ARBISCAN_API_KEY!,
    },
  },

  // ═══════════════════════════════════════════════════
  // Gas 报告配置
  // ═══════════════════════════════════════════════════
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_KEY,
    gasPrice: 20,  // gwei
  },

  // ═══════════════════════════════════════════════════
  // 代码覆盖率
  // ═══════════════════════════════════════════════════
  coverage: {
    include: ["contracts/**/*.sol"],
    exclude: ["contracts/mocks/**", "contracts/test/**"],
  },

  // ═══════════════════════════════════════════════════
  // 路径配置（可选自定义）
  // ═══════════════════════════════════════════════════
  paths: {
    sources: "./contracts",
    tests: "./test",
    cache: "./cache",
    artifacts: "./artifacts",
  },
};

export default config;
```

### .env 文件

```bash
# .env（⚠️ 绝对不要提交到 Git！）
PRIVATE_KEY=0xabc123...
INFURA_KEY=your_infura_key
ALCHEMY_KEY=your_alchemy_key
ETHERSCAN_API_KEY=your_etherscan_key
ARBISCAN_API_KEY=your_arbiscan_key
COINMARKETCAP_KEY=your_coinmarketcap_key
REPORT_GAS=true
```

```bash
# .gitignore 必须包含
.env
cache/
artifacts/
typechain-types/
coverage/
```

---

## 4. 合约编译

### 4.1 编译命令

```bash
# 编译所有合约
npx hardhat compile

# 强制重新编译
npx hardhat compile --force

# 编译输出：
# Compiled 3 Solidity files successfully
```

### 4.2 编译产物

```
artifacts/
├── contracts/
│   ├── Lock.sol/
│   │   ├── Lock.json              # 完整 artifact（ABI + 字节码 + 部署信息）
│   │   └── Lock.dbg.json          # 调试信息
│   └── ...
├── build-info/                    # 编译器输入输出详细信息
└── ...

# 同时生成 TypeScript 类型
typechain-types/
├── Lock.ts                        # 合约类型定义
├── factories/
│   └── Lock__factory.ts           # 合约工厂类型
└── ...
```

> **TypeChain**：Hardhat 自动为合约生成 TypeScript 类型，这意味着你在调用合约方法时有完整的类型提示和编译时检查。

### 4.3 增量编译

Hardhat 默认**增量编译**——只重新编译有变更的合约。缓存存储在 `cache/` 目录中。

```bash
# 清除缓存（解决编译异常时使用）
npx hardhat clean
```

### 4.4 编译大小检查

```bash
# 安装
npm install --save-dev hardhat-contract-sizer

# 在 hardhat.config.ts 中引入
# import "hardhat-contract-sizer";

# 运行
npx hardhat size-contracts
# ┌──────────────────┬────────────┬─────────────┐
# │ Contract         │ Size (B)   │ Change (B)  │
# ├──────────────────┼────────────┼─────────────┤
# │ Lock             │ 1242       │ +1242       │
# └──────────────────┴────────────┴─────────────┘
```

> **24KB 限制**：EVM 限制合约部署字节码不超过 24576 字节（EIP-170）。超过此限制的合约无法部署。

---

## 5. Hardhat Network（内置开发链）

Hardhat Network 是内置的以太坊网络模拟器——**无需启动 Ganache**，测试时自动启动。

### 5.1 自动模式 vs 节点模式

```bash
# 自动模式：运行测试或脚本时自动启动，结束后销毁
npx hardhat test                  # 自动启动/关闭
npx hardhat run scripts/deploy.ts # 自动启动/关闭

# 节点模式：手动启动，持久运行
npx hardhat node
# Started HTTP and WebSocket JSON-RPC server at http://127.0.0.1:8545

# 另一个终端连接
npx hardhat console --network localhost
```

### 5.2 主网 Fork

```bash
# 启动时 Fork 主网
npx hardhat node --fork https://mainnet.infura.io/v3/YOUR_KEY

# Fork 指定区块
npx hardhat node --fork https://mainnet.infura.io/v3/YOUR_KEY --fork-block-number 19000000
```

```typescript
// 测试中使用主网 Fork
describe("USDC interaction", function () {
  it("should interact with mainnet USDC", async function () {
    // Fork 模式下，可以使用真实合约地址
    const usdc = await ethers.getContractAt("IERC20", "0xA0b8...");
    const balance = await usdc.balanceOf("0x...");
    console.log("USDC balance:", ethers.formatUnits(balance, 6));
  });
});
```

### 5.3 Impersonate（模拟任意地址）

```typescript
import { time, impersonateAccount, stopImpersonatingAccount } from "@nomicfoundation/hardhat-network-helpers";

describe("Impersonate", function () {
  it("should act as Vitalik", async function () {
    // 模拟 Vitalik 的地址
    const vitalik = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045";

    await impersonateAccount(vitalik);
    const vitalikSigner = await ethers.getSigner(vitalik);

    // 现在 vitalikSigner 可以发送交易
    const usdc = await ethers.getContractAt("IERC20", "0xA0b8...");
    await usdc.connect(vitalikSigner).transfer(someAddress, amount);

    await stopImpersonatingAccount(vitalik);
  });
});
```

### 5.4 时间与区块操作

```typescript
import { time, mine } from "@nomicfoundation/hardhat-network-helpers";

describe("Time manipulation", function () {
  it("should advance time", async function () {
    // 快进 30 天
    const thirtyDays = 30 * 24 * 60 * 60;
    await time.increase(thirtyDays);

    // 快进到指定时间戳
    const futureTimestamp = 2000000000;
    await time.increaseTo(futureTimestamp);

    // 挖一个新区块
    await mine();

    // 挖多个区块
    await mine(5);
  });
});
```

---

## 6. 部署脚本

### 6.1 传统脚本式部署

```typescript
// scripts/deploy.ts
import { ethers } from "hardhat";

async function main() {
  const [deployer] = await ethers.getSigners();
  console.log("Deploying with:", deployer.address);
  console.log("Balance:", ethers.formatEther(await ethers.provider.getBalance(deployer.address)));

  // 部署合约
  const SimpleToken = await ethers.getContractFactory("SimpleToken");
  const token = await SimpleToken.deploy(1000000);
  await token.waitForDeployment();

  console.log("SimpleToken deployed to:", await token.getAddress());
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```

```bash
# 部署到本地
npx hardhat run scripts/deploy.ts

# 部署到 Sepolia
npx hardhat run scripts/deploy.ts --network sepolia

# 部署到主网
npx hardhat run scripts/deploy.ts --network mainnet
```

### 6.2 带依赖的部署

```typescript
// scripts/deploy-with-deps.ts
import { ethers } from "hardhat";

async function main() {
  // 1. 部署 Token
  const SimpleToken = await ethers.getContractFactory("SimpleToken");
  const token = await SimpleToken.deploy(1000000);
  await token.waitForDeployment();
  const tokenAddress = await token.getAddress();
  console.log("Token:", tokenAddress);

  // 2. 部署 TokenSale（依赖 Token 地址）
  const TokenSale = await ethers.getContractFactory("TokenSale");
  const rate = 100;
  const [deployer] = await ethers.getSigners();
  const sale = await TokenSale.deploy(rate, deployer.address, tokenAddress);
  await sale.waitForDeployment();
  const saleAddress = await sale.getAddress();
  console.log("TokenSale:", saleAddress);

  // 3. 转移 Token 给 TokenSale
  await token.transfer(saleAddress, 500000);
  console.log("Transferred 500000 tokens to TokenSale");

  // 4. 验证部署结果
  const saleTokenBalance = await token.balanceOf(saleAddress);
  console.log("TokenSale balance:", ethers.formatUnits(saleTokenBalance, 18));
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1));
  });
```

### 6.3 Hardhat Ignition（声明式部署）

Hardhat Ignition 是新一代部署系统——用声明式模块描述部署，自动处理依赖和重试。

```typescript
// ignition/modules/TokenSaleModule.ts
import { buildModule } from "@nomicfoundation/hardhat-ignition/modules";

export default buildModule("TokenSaleModule", (m) => {
  // 声明部署 Token
  const token = m.contract("SimpleToken", [1000000]);

  // 声明部署 TokenSale（自动等待 Token 部署完成）
  const sale = m.contract("TokenSale", [100, m.getAccount(0), token]);

  // 声明 Token 转账
  m.call(token, "transfer", [sale, 500000]);

  return { token, sale };
});
```

```bash
# 部署 Ignition 模块
npx hardhat ignition deploy ignition/modules/TokenSaleModule.ts --network sepolia

# 部署状态自动保存在 ignition/deployments/ 目录
# 如果中途失败，重试时自动跳过已完成步骤
```

---

## 7. 测试框架

### 7.1 基本测试结构

```typescript
// test/SimpleToken.test.ts
import { expect } from "chai";
import { ethers } from "hardhat";
import { SimpleToken } from "../typechain-types";

describe("SimpleToken", function () {
  let token: SimpleToken;
  let owner: any, alice: any, bob: any;

  beforeEach(async function () {
    // 获取签名者
    [owner, alice, bob] = await ethers.getSigners();

    // 部署合约
    const SimpleTokenFactory = await ethers.getContractFactory("SimpleToken");
    token = await SimpleTokenFactory.deploy(1000000);
    await token.waitForDeployment();
  });

  describe("Deployment", function () {
    it("should set correct name", async function () {
      expect(await token.name()).to.equal("SimpleToken");
    });

    it("should set correct symbol", async function () {
      expect(await token.symbol()).to.equal("ST");
    });

    it("should mint initial supply to deployer", async function () {
      const balance = await token.balanceOf(owner.address);
      expect(balance).to.equal(ethers.parseUnits("1000000", 18));
    });
  });

  describe("Transfer", function () {
    it("should transfer tokens", async function () {
      await token.transfer(alice.address, 100);
      expect(await token.balanceOf(alice.address)).to.equal(100);
    });

    it("should emit Transfer event", async function () {
      await expect(token.transfer(alice.address, 100))
        .to.emit(token, "Transfer")
        .withArgs(owner.address, alice.address, 100);
    });

    it("should revert on insufficient balance", async function () {
      await expect(
        token.connect(alice).transfer(bob.address, 999999)
      ).to.be.revertedWithCustomError(token, "ERC20InsufficientBalance");
    });
  });

  describe("Approve", function () {
    it("should approve and transferFrom", async function () {
      await token.approve(alice.address, 200);
      expect(await token.allowance(owner.address, alice.address)).to.equal(200);

      await token.connect(alice).transferFrom(owner.address, bob.address, 100);
      expect(await token.balanceOf(bob.address)).to.equal(100);
      expect(await token.allowance(owner.address, alice.address)).to.equal(100);
    });

    it("should emit Approval event", async function () {
      await expect(token.approve(alice.address, 200))
        .to.emit(token, "Approval")
        .withArgs(owner.address, alice.address, 200);
    });
  });
});
```

### 7.2 常用测试模式

**Revert 测试：**

```typescript
// 自定义错误
await expect(token.transfer(alice.address, 999999))
  .to.be.revertedWithCustomError(token, "ERC20InsufficientBalance");

// 字符串错误（旧版 require）
await expect(someContract.someFunction())
  .to.be.revertedWith("Not owner");

// 任意 revert
await expect(someContract.failingFunction())
  .to.be.reverted;
```

**事件测试：**

```typescript
// 检查事件
await expect(token.transfer(alice.address, 100))
  .to.emit(token, "Transfer")
  .withArgs(owner.address, alice.address, 100);

// 检查多个事件
await expect(tx)
  .to.emit(token, "Transfer")
  .and.to.emit(token, "Approval");
```

**Gas 消耗测试：**

```typescript
it("should report gas cost", async function () {
  const tx = await token.transfer(alice.address, 100);
  const receipt = await tx.wait();
  console.log(`Transfer gas used: ${receipt!.gasUsed.toString()}`);
});
```

**时间相关测试：**

```typescript
import { time } from "@nomicfoundation/hardhat-network-helpers";

describe("Vesting", function () {
  it("should release tokens after cliff", async function () {
    const vesting = await Vesting.deploy(...);

    // 快进 1 年
    await time.increase(365 * 24 * 60 * 60);

    // 现在可以释放
    await expect(vesting.release()).to.not.be.reverted;
  });
});
```

### 7.3 运行测试

```bash
# 运行所有测试
npx hardhat test

# 运行特定测试文件
npx hardhat test test/SimpleToken.test.ts

# 详细输出
npx hardhat test --verbose

# 只运行匹配的测试
npx hardhat test --grep "should transfer"

# Gas 报告
REPORT_GAS=true npx hardhat test

# 代码覆盖率
npx hardhat coverage
```

---

## 8. console.log 调试

Hardhat 独有的功能——在 Solidity 中使用 `console.log`：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "hardhat/console.sol";

contract Debugging {
    function complexCalculation(uint256 x) public pure returns (uint256) {
        console.log("Input x:", x);

        uint256 step1 = x * 2;
        console.log("Step 1 (x * 2):", step1);

        uint256 step2 = step1 + 10;
        console.log("Step 2 (step1 + 10):", step2);

        return step2;
    }
}
```

> **注意**：`console.log` 只在开发网络中有效。部署到真实网络时，`console.sol` 是空操作（no-op），不消耗额外 Gas。

支持的类型：
```solidity
console.log(string);
console.log(uint256);
console.log(int256);
console.log(address);
console.log(bool);
console.log(string, uint256);  // 格式化输出
console.log("Balance of %s is %d", addr, balance);  // 占位符
```

---

## 9. Etherscan 验证

```bash
# 验证已部署的合约
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> "Constructor arg 1" "Constructor arg 2"

# 示例
npx hardhat verify --network sepolia 0x1234... 1000000

# 验证代理合约
npx hardhat verify --contract contracts/LogicV1.sol:LogicV1 --network sepolia 0x5678...
```

```typescript
// 部署脚本中自动验证
async function main() {
  const token = await SimpleToken.deploy(1000000);
  await token.waitForDeployment();

  // 等待几个区块确认后验证
  await token.deploymentTransaction()!.wait(5);

  await hre.run("verify:verify", {
    address: await token.getAddress(),
    constructorArguments: [1000000],
  });
}
```

---

## 10. 常用命令速查

| 命令 | 作用 |
|------|------|
| `npx hardhat compile` | 编译合约 |
| `npx hardhat clean` | 清除缓存和编译产物 |
| `npx hardhat test` | 运行测试 |
| `npx hardhat test --grep "pattern"` | 运行匹配的测试 |
| `npx hardhat coverage` | 代码覆盖率 |
| `npx hardhat run scripts/deploy.ts` | 运行部署脚本 |
| `npx hardhat run scripts/deploy.ts --network sepolia` | 部署到指定网络 |
| `npx hardhat node` | 启动本地节点 |
| `npx hardhat console --network localhost` | 交互控制台 |
| `npx hardhat verify --network sepolia <ADDR> <ARGS>` | Etherscan 验证 |
| `npx hardhat ignition deploy <MODULE> --network sepolia` | Ignition 部署 |
| `npx hardhat size-contracts` | 查看合约大小 |
| `REPORT_GAS=true npx hardhat test` | Gas 报告 |
| `npx hardhat flatten contracts/Token.sol` | 展平合约（用于验证） |

---

## 11. 与 Foundry 共存

Hardhat 和 Foundry 可以在同一项目中使用，互补优势：

```
项目根目录/
├── contracts/          # 共享合约源码
├── test/
│   ├── foundry/        # Foundry Solidity 测试（fuzz/invariant）
│   │   ├── Token.t.sol
│   │   └── Invariant.t.sol
│   └── hardhat/        # Hardhat JS/TS 测试（集成测试）
│       └── Token.test.ts
├── script/             # Foundry 部署脚本
│   └── Deploy.s.sol
├── scripts/            # Hardhat 部署脚本
│   └── deploy.ts
├── hardhat.config.ts   # Hardhat 配置
└── foundry.toml        # Foundry 配置
```

```toml
# foundry.toml（与 Hardhat 共存配置）
[profile.default]
src = "contracts"
out = "out"
test = "test/foundry"
script = "script"
libs = ["node_modules", "lib"]

# 读取 Hardhat 的编译缓存
cache_path = "cache/foundry"

# 避免与 Hardhat artifacts 冲突
force = true
```
