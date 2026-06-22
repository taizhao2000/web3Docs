# Web3.js 与现代前端栈指南

> Web3.js 是以太坊生态最早的 JavaScript 库（2015 年），现已逐步被 ethers.js 和 viem 取代。本文帮助理解旧项目，并指导迁移到现代前端栈。

---

## 1. Web3.js v4 概述

Web3.js v4（2022 年发布）是对 v1.x 的完全重写，用 TypeScript 编写：

```bash
npm install web3
```

### 基本用法

```typescript
import Web3 from "web3";

// 连接节点
const web3 = new Web3("https://mainnet.infura.io/v3/YOUR_KEY");

// 或连接 MetaMask
const web3 = new Web3(window.ethereum);
await window.ethereum.request({ method: "eth_requestAccounts" });

// 读取余额
const balance = await web3.eth.getBalance("0xd8dA...");
console.log(web3.utils.fromWei(balance, "ether"), "ETH");

// 合约交互
const contract = new web3.eth.Contract(abi, contractAddress);

// 读取
const name = await contract.methods.name().call();

// 写入
const tx = await contract.methods.transfer(to, amount).send({
  from: account,
  gas: 200000,
  gasPrice: await web3.eth.getGasPrice(),
});
```

### Web3.js vs ethers.js

| 特性 | Web3.js v4 | ethers.js v6 |
|------|-----------|-------------|
| 语言 | TypeScript | TypeScript |
| 包大小 | ~400KB | ~100KB |
| 大数类型 | `BigInt`（原生） | `bigint`（原生） |
| API 风格 | 面向对象 | 函数式 |
| 事件监听 | `contract.events.Transfer()` | `contract.on("Transfer", cb)` |
| 合约地址 | `contract.options.address` | `await contract.getAddress()` |
| 交易等待 | `tx` 直接返回 hash | `await tx.wait()` |
| Provider 类型 | `Web3.providers.HttpProvider` | `JsonRpcProvider` |
| 推荐度 | ⭐⭐ | ⭐⭐⭐⭐⭐ |
| 当前状态 | 维护模式 | 活跃开发 |

> **迁移建议**：Web3.js 已进入维护模式，新项目不应使用。现有项目应逐步迁移到 ethers.js 或 viem。

---

## 2. 现代前端栈：wagmi + viem

### 2.1 什么是 wagmi + viem

```
┌─────────────────────────────────────────────────┐
│                  你的 DApp 前端                    │
├─────────────────────────────────────────────────┤
│  wagmi (React Hooks)                             │
│    useAccount / useBalance / useContractRead     │
│    useContractWrite / useSendTransaction          │
├─────────────────────────────────────────────────┤
│  viem (底层库)                                    │
│    TypeScript-first / Tree-shakeable / Tiny      │
├─────────────────────────────────────────────────┤
│  ethers.js / Web3.js (旧方案)                     │
└─────────────────────────────────────────────────┘
```

| 库 | 角色 | 说明 |
|----|------|------|
| **viem** | 底层以太坊交互库 | ethers.js 的替代品，更小更快，TypeScript 原生 |
| **wagmi** | React Hooks 库 | 基于 viem，提供 React Hooks API |
| **RainbowKit / ConnectKit** | UI 组件库 | 连接钱包的 UI 组件 |

### 2.2 viem vs ethers.js

| 特性 | viem | ethers.js v6 |
|------|------|-------------|
| 包大小 | ~45KB | ~100KB |
| Tree-shaking | ✅ 按需导入 | ❌ 整包导入 |
| 类型安全 | **最强**（Literal 类型） | 强 |
| 性能 | **最快** | 快 |
| API 风格 | 函数式 | 函数式 |
| React 集成 | wagmi | 需手动 Hook |
| 推荐度 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

### 2.3 为什么选择 wagmi + viem

```
ethers.js：
  const provider = new BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();
  const contract = new Contract(address, abi, signer);
  const balance = await contract.balanceOf(userAddress);
  // 类型：any（无类型提示）

wagmi + viem：
  const { data: balance } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: "balanceOf",
    args: [userAddress],
  });
  // 类型：bigint（完整类型提示）
  // 自动缓存 / 自动刷新 / 错误重试
```

---

## 3. wagmi + viem 实战

### 3.1 安装

```bash
npm install wagmi viem @tanstack/react-query @rainbow-me/rainbowkit
```

### 3.2 配置

```typescript
// wagmi.config.ts
import { http, createConfig, WagmiConfig } from "wagmi";
import { mainnet, sepolia, polygon, arbitrum } from "wagmi/chains";
import { getDefaultWallets, RainbowKitProvider } from "@rainbow-me/rainbowkit";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

import "@rainbow-me/rainbowkit/styles.css";

// 1. 配置钱包连接器
const { connectors } = getDefaultWallets({
  appName: "MyDApp",
  projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID!,
  chains: [mainnet, polygon, arbitrum],
});

// 2. 创建 wagmi 配置
const config = createConfig({
  chains: [mainnet, polygon, arbitrum],
  connectors,
  transports: {
    [mainnet.id]: http("https://mainnet.infura.io/v3/YOUR_KEY"),
    [polygon.id]: http("https://polygon-rpc.com"),
    [arbitrum.id]: http("https://arb1.arbitrum.io/rpc"),
  },
});

const queryClient = new QueryClient();

// 3. 包裹 Provider
export function Web3Provider({ children }: { children: React.ReactNode }) {
  return (
    <WagmiConfig config={config}>
      <QueryClientProvider client={queryClient}>
        <RainbowKitProvider>
          {children}
        </RainbowKitProvider>
      </QueryClientProvider>
    </WagmiConfig>
  );
}
```

### 3.3 连接钱包

```typescript
import {
  ConnectButton,
  useAccount,
  useDisconnect,
} from "@rainbow-me/rainbowkit";
import { useAccount as useWagmiAccount } from "wagmi";

// RainbowKit 提供的连接按钮（一行搞定！）
function Header() {
  return <ConnectButton />;
}

// 获取账户信息
function AccountInfo() {
  const { address, chain, isConnected } = useAccount();

  if (!isConnected) return <p>未连接</p>;

  return (
    <div>
      <p>地址: {address}</p>
      <p>链: {chain?.name}</p>
    </div>
  );
}
```

### 3.4 读取合约数据

```typescript
import { useReadContract } from "wagmi";
import { erc20Abi } from "viem";

function TokenBalance({ tokenAddress, userAddress }: {
  tokenAddress: `0x${string}`;
  userAddress: `0x${string}`;
}) {
  const { data: balance, isLoading, isError } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: "balanceOf",
    args: [userAddress],
  });

  if (isLoading) return <p>加载中...</p>;
  if (isError) return <p>读取失败</p>;

  // balance 的类型是 bigint（自动推断！）
  return <p>余额: {balance?.toString()}</p>;
}

// 多次读取
function TokenInfo({ tokenAddress }: { tokenAddress: `0x${string}` }) {
  const { data: name } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: "name",
  });

  const { data: symbol } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: "symbol",
  });

  const { data: decimals } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: "decimals",
  });

  return (
    <div>
      <p>{name} ({symbol})</p>
      <p>Decimals: {decimals}</p>
    </div>
  );
}
```

### 3.5 写入合约

```typescript
import { useWriteContract, useWaitForTransactionReceipt } from "wagmi";
import { erc20Abi, parseUnits } from "viem";

function TransferButton({
  tokenAddress,
  to,
  amount,
  decimals,
}: {
  tokenAddress: `0x${string}`;
  to: `0x${string}`;
  amount: string;
  decimals: number;
}) {
  const { data: hash, writeContract, isPending } = useWriteContract();

  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  });

  const handleTransfer = () => {
    writeContract({
      address: tokenAddress,
      abi: erc20Abi,
      functionName: "transfer",
      args: [to, parseUnits(amount, decimals)],
    });
  };

  return (
    <div>
      <button
        onClick={handleTransfer}
        disabled={isPending || isConfirming}
      >
        {isPending ? "确认中..." : isConfirming ? "交易中..." : "转账"}
      </button>
      {isSuccess && <p>转账成功！</p>}
    </div>
  );
}
```

### 3.6 监听事件

```typescript
import { useWatchContractEvent } from "wagmi";
import { erc20Abi } from "viem";

function TransferEvents({ tokenAddress }: { tokenAddress: `0x${string}` }) {
  useWatchContractEvent({
    address: tokenAddress,
    abi: erc20Abi,
    eventName: "Transfer",
    onLogs(logs) {
      logs.forEach(log => {
        console.log(
          `Transfer: ${log.args.from} → ${log.args.to}: ${log.args.value}`
        );
      });
    },
  });

  return <div>监听 Transfer 事件中...</div>;
}
```

### 3.7 直接使用 viem

```typescript
import { createPublicClient, http, parseEther, formatEther } from "viem";
import { mainnet } from "viem/chains";

// 创建 Public Client（只读）
const client = createPublicClient({
  chain: mainnet,
  transport: http("https://mainnet.infura.io/v3/YOUR_KEY"),
});

// 读取余额
const balance = await client.getBalance({
  address: "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
});
console.log(formatEther(balance), "ETH");

// 读取合约
import { getContract } from "viem";

const contract = getContract({
  address: "0xA0b8...",
  abi: erc20Abi,
  client,
});

const name = await contract.read.name();
const balance = await contract.read.balanceOf(["0x..."]);

// 发送交易（需要 Wallet Client）
import { createWalletClient, custom } from "viem";

const walletClient = createWalletClient({
  chain: mainnet,
  transport: custom(window.ethereum),
});

const [account] = await walletClient.getAddresses();

const txHash = await walletClient.writeContract({
  address: "0xA0b8...",
  abi: erc20Abi,
  functionName: "transfer",
  args: ["0xRecipient...", parseUnits("100", 6)],
  account,
});
```

---

## 4. 技术栈选择决策

```
你的项目类型？
    │
    ├── React/Next.js DApp
    │   └── wagmi + viem + RainbowKit（最佳选择）
    │
    ├── Vue/Svelte DApp
    │   └── viem（无 React 依赖）+ 手动状态管理
    │
    ├── 后端脚本 / Bot
    │   └── ethers.js 或 viem（Node.js 环境）
    │
    ├── 维护旧 Web3.js 项目
    │   └── 保持 Web3.js 或迁移到 ethers.js
    │
    └── 维护旧 ethers.js v5 项目
        └── 迁移到 ethers.js v6 或直接升级到 wagmi + viem
```

| 场景 | 推荐 | 原因 |
|------|------|------|
| 新 React 项目 | **wagmi + viem + RainbowKit** | 最佳开发体验，完整类型安全 |
| 新非 React 项目 | **viem** | 轻量，Tree-shakeable |
| 维护 ethers.js v6 项目 | **保持 ethers.js v6** | 迁移成本 > 收益 |
| 维护 ethers.js v5 项目 | **迁移到 v6 或 viem** | v5 不再更新 |
| 维护 Web3.js 项目 | **迁移到 ethers.js 或 viem** | Web3.js 已进入维护模式 |
| 简单脚本 | **ethers.js v6** | 最简单直接 |
| 复杂 DApp | **wagmi + viem** | 自动缓存、错误重试、类型安全 |

---

## 5. 从 Web3.js 迁移到 ethers.js v6

### API 对照表

| 操作 | Web3.js v4 | ethers.js v6 |
|------|-----------|-------------|
| 连接节点 | `new Web3(rpcUrl)` | `new JsonRpcProvider(rpcUrl)` |
| 连接 MetaMask | `new Web3(window.ethereum)` | `new BrowserProvider(window.ethereum)` |
| 获取账户 | `web3.eth.getAccounts()` | `await provider.send("eth_requestAccounts", [])` |
| 读取余额 | `web3.eth.getBalance(addr)` | `provider.getBalance(addr)` |
| 格式化 ETH | `web3.utils.fromWei(x, "ether")` | `formatEther(x)` |
| 解析 ETH | `web3.utils.toWei(x, "ether")` | `parseEther(x)` |
| 创建合约 | `new web3.eth.Contract(abi, addr)` | `new Contract(addr, abi, provider)` |
| 读合约 | `contract.methods.x().call()` | `contract.x()` |
| 写合约 | `contract.methods.x().send({from})` | `contract.connect(signer).x()` |
| 事件监听 | `contract.events.Transfer()` | `contract.on("Transfer", cb)` |
| 签名消息 | `web3.eth.sign(msg, addr)` | `signer.signMessage(msg)` |
| 获取 Gas | `web3.eth.getGasPrice()` | `provider.getFeeData()` |
| 获取区块 | `web3.eth.getBlock(num)` | `provider.getBlock(num)` |
| 发送交易 | `web3.eth.sendTransaction(tx)` | `signer.sendTransaction(tx)` |

> **迁移工作量**：主要在于 API 调用方式的变化（面向对象 → 函数式）和大数类型的变化（Web3.js 的 `BigInt` vs ethers.js 的 `bigint`）。

---

## 快速参考卡

| 库 | 包大小 | TypeScript | React 集成 | 推荐场景 |
|----|--------|-----------|-----------|---------|
| **viem** | ~45KB | ✅ 最强 | wagmi | 新项目首选 |
| **ethers.js v6** | ~100KB | ✅ 强 | 手动 Hook | 旧项目维护、脚本 |
| **Web3.js v4** | ~400KB | ✅ | 手动 Hook | 旧项目维护 |
| **wagmi** | 依赖 viem | ✅ | **原生** | React DApp |
| **RainbowKit** | UI 库 | ✅ | 原生 | 钱包连接 UI |
