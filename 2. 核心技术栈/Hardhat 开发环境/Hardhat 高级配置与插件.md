# Hardhat 高级配置与插件

> 掌握基础后，这些高级功能和插件将显著提升开发效率和代码质量。

---

## 1. 插件生态详解

### 1.1 @nomicfoundation/hardhat-toolbox

一站式工具包，包含以下插件：

| 包含插件 | 功能 |
|---------|------|
| `@nomicfoundation/hardhat-ethers` | ethers.js v6 集成 |
| `@nomicfoundation/hardhat-chai-matchers` | Chai 匹配器（revertedWith、emit 等） |
| `@nomicfoundation/hardhat-network-helpers` | 网络操作（时间、挖矿、impersonate） |
| `@nomicfoundation/hardhat-ignition` | 声明式部署 |
| `@nomicfoundation/hardhat-ignition-modules` | Ignition 模块 |
| `hardhat-gas-reporter` | Gas 消耗报告 |
| `solidity-coverage` | 代码覆盖率 |

```typescript
// hardhat.config.ts
import "@nomicfoundation/hardhat-toolbox";
// 一行搞定！
```

### 1.2 hardhat-gas-reporter

```bash
npm install --save-dev hardhat-gas-reporter
```

```typescript
// hardhat.config.ts
import "hardhat-gas-reporter";

const config: HardhatUserConfig = {
  gasReporter: {
    enabled: process.env.REPORT_GAS === "true",
    currency: "USD",
    coinmarketcap: process.env.COINMARKETCAP_KEY,
    gasPrice: 20,                    // gwei
    outputFile: "gas-report.txt",    // 输出到文件
    noColors: true,                  // 无颜色（适合 CI）
    showTimeSpent: true,             // 显示执行时间
    showMethodSig: true,             // 显示完整方法签名
  },
};
```

输出示例：
```
·----------------------------------------|----------------------------|-------------|----------------------------·
|  Solc version: 0.8.24                  ·  Optimizer enabled: true   ·  Runs: 200  ·  Block limit: 30000000 gas  │
·········································|····························|·············|·····························
|  Methods                                                                                                          │
·········································|····························|·············|·····························
|  Contract                             ·  Method                   ·  Min        ·  Max                       │
·········································|····························|·············|·····························
|  SimpleToken                          ·  transfer                 ·  51286     ·  65286                    │
|  SimpleToken                          ·  approve                  ·  30824     ·  47990                    │
·········································|····························|·············|·····························
|  Deployments                                       ·                                               │
·········································|····························|·············|·····························
|  SimpleToken                                       ·  1180953                                      │
·----------------------------------------|----------------------------|-------------|----------------------------·
```

### 1.3 solidity-coverage

```bash
npm install --save-dev solidity-coverage
```

```bash
# 运行覆盖率检查
npx hardhat coverage

# 输出：
# ------------|----------|----------|----------|----------|
# File        |  % Stmts | % Branch |  % Funcs |   % Lines |
# ------------|----------|----------|----------|----------|
#  contracts/ |          |          |          |          |
#   Token.sol |     87.5 |    66.67 |      100 |    87.88 |
# ------------|----------|----------|----------|----------|
# All files   |     87.5 |    66.67 |      100 |    87.88 |
# ------------|----------|----------|----------|----------|
```

### 1.4 @openzeppelin/hardhat-upgrades

部署可升级合约的必备插件——自动检查存储布局兼容性：

```bash
npm install --save-dev @openzeppelin/hardhat-upgrades @openzeppelin/contracts-upgradeable
```

```typescript
// hardhat.config.ts
import "@openzeppelin/hardhat-upgrades";

// scripts/deploy-upgradeable.ts
import { ethers, upgrades } from "hardhat";

async function main() {
  // 部署可升级代理
  const Box = await ethers.getContractFactory("Box");
  const box = await upgrades.deployProxy(Box, [42], {
    kind: "uups",    // 或 "transparent"
  });
  await box.waitForDeployment();
  console.log("Box (proxy):", await box.getAddress());

  // 升级合约
  const BoxV2 = await ethers.getContractFactory("BoxV2");
  const boxV2 = await upgrades.upgradeProxy(await box.getAddress(), BoxV2);
  console.log("Box upgraded!");
}
```

```typescript
// 插件自动检查存储布局兼容性
// 如果升级时修改了变量顺序，会报错：
// Error: New storage layout is incompatible
//   - `newField` was inserted before `owner`
//   - This changes the storage slot of `owner`
```

### 1.5 @nomicfoundation/hardhat-verify

Etherscan 和 Sourcify 验证：

```typescript
// hardhat.config.ts
import "@nomicfoundation/hardhat-verify";

const config: HardhatUserConfig = {
  etherscan: {
    apiKey: {
      mainnet: process.env.ETHERSCAN_API_KEY!,
      sepolia: process.env.ETHERSCAN_API_KEY!,
      arbitrumOne: process.env.ARBISCAN_API_KEY!,
      optimisticEthereum: process.env.OPTIMISTIC_API_KEY!,
      polygon: process.env.POLYGONSCAN_API_KEY!,
    },
  },
};
```

---

## 2. 自定义任务（Task）

Hardhat 的任务系统让你可以扩展 CLI 命令：

```typescript
// hardhat.config.ts
import { task } from "hardhat/config";

// 简单任务
task("accounts", "Prints the list of accounts", async (taskArgs, hre) => {
  const accounts = await hre.ethers.getSigners();
  for (const account of accounts) {
    console.log(account.address);
  }
});

// 带参数的任务
task("balance", "Prints an account's balance")
  .addParam("account", "The account's address")
  .setAction(async (taskArgs, hre) => {
    const balance = await hre.ethers.provider.getBalance(taskArgs.account);
    console.log(hre.ethers.formatEther(balance), "ETH");
  });

// 带可选参数的任务
task("mint", "Mint tokens to an address")
  .addParam("to", "Recipient address")
  .addOptionalParam("amount", "Amount to mint", "1000")
  .setAction(async (taskArgs, hre) => {
    const [deployer] = await hre.ethers.getSigners();
    const token = await hre.ethers.getContractAt("SimpleToken", "0x...");
    const tx = await token.mint(taskArgs.to, taskArgs.amount);
    await tx.wait();
    console.log(`Minted ${taskArgs.amount} tokens to ${taskArgs.to}`);
  });
```

```bash
# 使用自定义任务
npx hardhat accounts
npx hardhat balance --account 0x1234...
npx hardhat mint --to 0x5678... --amount 5000
```

---

## 3. Hardhat Network 高级功能

### 3.1 自动挖矿 vs 即时挖矿

```typescript
// hardhat.config.ts
networks: {
  hardhat: {
    mining: {
      // 自动挖矿（默认）：每笔交易自动出块
      auto: true,

      // 间隔挖矿：每 N 毫秒出一个块
      // auto: false,
      // interval: 5000,  // 5 秒一个块

      // 手动挖矿：通过 API 触发
      // auto: false,
    },
  },
}
```

### 3.2 日志与追踪

```typescript
networks: {
  hardhat: {
    logging: {
      // 打印 Hardhat Network 内部日志
      enabled: false,
    },
  },
}
```

### 3.3 设置账户余额

```typescript
networks: {
  hardhat: {
    accounts: {
      // 指定账户数量和余额
      count: 20,
      accountsBalance: "10000000000000000000000",  // 10000 ETH per account
    },
  },
}
```

### 3.4 hardhat_setCode 和 hardhat_setNonce

```typescript
// 在测试中修改链上状态（仅 Hardhat Network）
await network.provider.send("hardhat_setCode", [
  contractAddress,
  "0x608060...",  // 新的字节码
]);

await network.provider.send("hardhat_setNonce", [
  address,
  "0x42",  // 新的 nonce
]);

await network.provider.send("hardhat_setBalance", [
  address,
  "0xDE0B6B3A7640000",  // 1 ETH in hex
]);
```

---

## 4. hardhat-deploy 部署管理

`hardhat-deploy` 是社区广泛使用的部署管理插件，提供比原生脚本更完善的部署记录和参数化：

```bash
npm install --save-dev hardhat-deploy hardhat-deploy-ethers
```

### 4.1 部署脚本

```typescript
// deploy/01_deploy_token.ts
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { DeployFunction } from "hardhat-deploy/types";

const deployToken: DeployFunction = async function (hre: HardhatRuntimeEnvironment) {
  const { deployer } = await hre.getNamedAccounts();
  const { deploy } = hre.deployments;

  await deploy("SimpleToken", {
    from: deployer,
    args: [1000000],              // constructor 参数
    log: true,                    // 打印部署日志
    waitConfirmations: 5,         // 等待确认数
    proxy: {                       // 代理升级配置（可选）
      proxyContract: "UUPSProxy",
      owner: deployer,
    },
  });
};

export default deployToken;
deployToken.tags = ["Token"];      // 标签（用于选择性部署）
deployToken.dependencies = [];     // 依赖（先部署依赖合约）
```

### 4.2 部署记录

`hardhat-deploy` 自动在 `deployments/` 目录保存部署记录：

```
deployments/
├── sepolia/
│   ├── SimpleToken.json       # ABI + 地址 + 字节码 + 参数
│   ├── TokenSale.json
│   └── .chainId               # 链 ID 验证
└── mainnet/
    └── ...
```

```bash
# 部署
npx hardhat deploy --network sepolia

# 只部署带 "Token" 标签的合约
npx hardhat deploy --network sepolia --tags Token

# 导出部署信息
npx hardhat export --network sepolia ./frontend/src/contracts/
```

---

## 5. CI/CD 集成

### 5.1 GitHub Actions 示例

```yaml
# .github/workflows/test.yml
name: Hardhat CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Compile contracts
        run: npx hardhat compile

      - name: Run tests
        run: npx hardhat test

      - name: Run coverage
        run: npx hardhat coverage

      - name: Check contract sizes
        run: npx hardhat size-contracts

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/lcov.info
```

### 5.2 部署工作流

```yaml
# .github/workflows/deploy-sepolia.yml
name: Deploy to Sepolia

on:
  workflow_dispatch:  # 手动触发

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Compile contracts
        run: npx hardhat compile

      - name: Deploy to Sepolia
        run: npx hardhat run scripts/deploy.ts --network sepolia
        env:
          PRIVATE_KEY: ${{ secrets.PRIVATE_KEY }}
          INFURA_KEY: ${{ secrets.INFURA_KEY }}

      - name: Verify on Etherscan
        run: npx hardhat verify --network sepolia ${{ steps.deploy.outputs.address }}
        env:
          ETHERSCAN_API_KEY: ${{ secrets.ETHERSCAN_API_KEY }}
```

---

## 6. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| `HH100: Network account doesn't have enough funds` | 账户余额不足 | 给部署地址充 ETH |
| `InvalidInputError: unsupported engine` | solc 版本不支持 | 检查 solidity.version 配置 |
| `Compiler version not supported` | 编译器版本太新/旧 | 使用 compilers 数组指定多版本 |
| `Nonce too high` | 本地 nonce 与链上不同步 | 重置本地账户 nonce |
| `Transaction reverted without reason` | 合约 require 失败但无消息 | 用 `console.log` 逐行排查 |
| `Contract code size exceeds 24576 bytes` | 合约超过 24KB 限制 | 拆分合约或开启 viaIR 优化 |
| `TypeError: Cannot read properties of undefined` | ABI 不匹配 | 重新编译，检查 artifact 版本 |

### 调试技巧

```typescript
// 1. 使用 console.log
import "hardhat/console.sol";
console.log("Debug: value =", value);

// 2. 使用 Hardhat 的堆栈追踪
// Hardhat 自动在 revert 时显示 Solidity 源码行号

// 3. 使用 debug_traceTransaction
npx hardhat run --trace scripts/debug.ts

// 4. 打印交易详情
const tx = await someContract.someFunction();
const receipt = await tx.wait();
console.log("Gas used:", receipt.gasUsed.toString());
console.log("Events:", receipt.logs);
```
