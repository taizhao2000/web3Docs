# ethers.js v6 完整指南

> ethers.js 是以太坊生态最流行的 JavaScript 交互库。本文基于 **v6.x**（2024 年最新稳定版），完整覆盖从基础到生产级使用。如果你在维护 v5 项目，文末有 v5→v6 迁移对照表。

---

## 1. ethers.js 是什么

ethers.js 是一个**轻量级、安全的以太坊交互库**，提供：

| 功能 | 说明 |
|------|------|
| Provider | 读取链上数据（余额、区块、合约调用） |
| Signer | 签名交易（需要私钥或钱包连接） |
| Contract | 合约交互（调用方法、监听事件） |
| Utils | 格式化、编码、哈希、ABI 编解码 |
| Wallet | 私钥签名者（后端/脚本用） |

### v5 vs v6 关键变化

| 变化点 | v5 | v6 |
|--------|----|----|
| 包导入 | `ethers` 整体导入 | `ethers` 模块化导入 |
| Provider | `Web3Provider(window.ethereum)` | `BrowserProvider(window.ethereum)` |
| 获取 Signer | `provider.getSigner()` | `provider.getSigner()` (异步) |
| 合约地址 | `contract.address` | `await contract.getAddress()` |
| 事件监听 | `contract.on("Transfer", cb)` | `contract.on("Transfer", cb)` (ABI 必须有事件定义) |
| 格式化 | `ethers.utils.formatEther(x)` | `ethers.formatEther(x)` |
| BN 类型 | `BigNumber` | `bigint`（原生） |
| IPC Provider | ✅ | ❌ 移除 |

---

## 2. 安装与导入

```bash
# npm
npm install ethers

# pnpm
pnpm add ethers

# yarn
yarn add ethers
```

```typescript
// TypeScript / ES Modules
import { ethers, JsonRpcProvider, BrowserProvider, Wallet, Contract, formatEther, parseEther } from "ethers";

// Node.js CommonJS
const { ethers, JsonRpcProvider, Wallet, Contract } = require("ethers");
```

---

## 3. Provider 体系

Provider 是**只读**的链上数据访问入口——不需要私钥。

### 3.1 Provider 类型

| Provider | 用途 | 示例 |
|----------|------|------|
| `BrowserProvider` | 浏览器中连接 MetaMask | `new BrowserProvider(window.ethereum)` |
| `JsonRpcProvider` | 连接 RPC 节点 | `new JsonRpcProvider("https://...")` |
| `InfuraProvider` | Infura 专用 | `new InfuraProvider("mainnet", apiKey)` |
| `AlchemyProvider` | Alchemy 专用 | `newAlchemyProvider("mainnet", apiKey)` |
| `FallbackProvider` | 多节点容错 | 多个 Provider 组合 |
| `InfuraWebSocketProvider` | WebSocket 实时 | 实时监听事件 |

### 3.2 基础用法

```typescript
import { JsonRpcProvider, formatEther } from "ethers";

// 连接节点
const provider = new JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_KEY");

// 读取最新区块号
const blockNumber = await provider.getBlockNumber();
console.log("Latest block:", blockNumber);

// 读取 ETH 余额
const balance = await provider.getBalance("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045");
console.log("Balance:", formatEther(balance), "ETH");
// 输出: Balance: 1000.0 ETH

// 读取交易
const tx = await provider.getTransaction("0x...");
console.log("From:", tx.from);
console.log("To:", tx.to);
console.log("Value:", formatEther(tx.value), "ETH");

// 读取区块
const block = await provider.getBlock(blockNumber);
console.log("Timestamp:", new Date(block.timestamp * 1000));

// 获取 Gas 价格
const feeData = await provider.getFeeData();
console.log("Gas price:", formatEther(feeData.gasPrice!), "ETH");
console.log("Max fee:", formatEther(feeData.maxFeePerGas!), "ETH");
```

### 3.3 浏览器中连接 MetaMask

```typescript
import { BrowserProvider } from "ethers";

// 检查 MetaMask 是否安装
if (window.ethereum) {
  // 创建 Provider
  const provider = new BrowserProvider(window.ethereum);

  // 请求用户授权连接
  await provider.send("eth_requestAccounts", []);

  // 获取 Signer（可以签名交易）
  const signer = await provider.getSigner();

  // 获取当前账户
  const address = await signer.getAddress();
  console.log("Connected:", address);

  // 获取网络信息
  const network = await provider.getNetwork();
  console.log("Chain ID:", network.chainId.toString());
  console.log("Network name:", network.name);
} else {
  console.log("请安装 MetaMask");
}
```

### 3.4 FallbackProvider（多节点容错）

```typescript
import { FallbackProvider, JsonRpcProvider } from "ethers";

const providers = [
  new JsonRpcProvider("https://mainnet.infura.io/v3/KEY1"),
  new JsonRpcProvider("https://mainnet.g.alchemy.com/v2/KEY2"),
  new JsonRpcProvider("https://eth.llamarpc.com"),
];

// 如果一个节点失败，自动切换到下一个
const fallbackProvider = new FallbackProvider(providers);

// 使用方式与普通 Provider 相同
const balance = await fallbackProvider.getBalance("0x...");
```

---

## 4. Signer 体系

Signer 是**可写**的链上交互入口——需要私钥或钱包授权。

### 4.1 Wallet（私钥签名者）

用于后端脚本、测试环境：

```typescript
import { Wallet, JsonRpcProvider, parseEther } from "ethers";

// 从私钥创建 Wallet
const privateKey = "0x...";  // ⚠️ 绝不要硬编码到前端！
const wallet = new Wallet(privateKey);

// 连接到 Provider
const provider = new JsonRpcProvider("https://sepolia.infura.io/v3/YOUR_KEY");
const connectedWallet = wallet.connect(provider);

// 发送 ETH
const tx = await connectedWallet.sendTransaction({
  to: "0xRecipient...",
  value: parseEther("0.1"),
});
console.log("TX hash:", tx.hash);

// 等待确认
const receipt = await tx.wait();
console.log("Confirmed in block:", receipt!.blockNumber);

// 从助记词创建
const mnemonicWallet = Wallet.fromPhrase("test test test test test test test test test test test junk");
console.log("Address:", mnemonicWallet.address);
```

### 4.2 浏览器 Signer（MetaMask）

```typescript
import { BrowserProvider } from "ethers";

const provider = new BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

// 发送 ETH
const tx = await signer.sendTransaction({
  to: "0xRecipient...",
  value: parseEther("0.1"),
});
await tx.wait();
```

### 4.3 签名消息

```typescript
// 签名原始消息
const signature = await signer.signMessage("Hello Ethereum");
console.log("Signature:", signature);

// 验证签名
const recoveredAddress = ethers.verifyMessage("Hello Ethereum", signature);
console.log("Recovered:", recoveredAddress);
// 应该等于 signer 的地址

// EIP-712 类型化数据签名
const typedData = {
  types: {
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" },
    ],
  },
  domain: {
    name: "MyToken",
    version: "1",
    chainId: 1,
    verifyingContract: "0xToken...",
  },
  message: {
    owner: "0xOwner...",
    spender: "0xSpender...",
    value: 1000n,
    nonce: 0n,
    deadline: 2000000000n,
  },
};

const sig = await signer.signTypedData(
  typedData.domain,
  typedData.types,
  typedData.message
);
```

---

## 5. Contract 交互

### 5.1 创建合约实例

```typescript
import { Contract, BrowserProvider, JsonRpcProvider } from "ethers";

// ABI 定义
const ERC20_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function decimals() view returns (uint8)",
  "function totalSupply() view returns (uint256)",
  "function balanceOf(address) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "function approve(address spender, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 value)",
  "event Approval(address indexed owner, address indexed spender, uint256 value)",
];

const tokenAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"; // USDC

// 只读合约（Provider）
const provider = new JsonRpcProvider("https://mainnet.infura.io/v3/YOUR_KEY");
const readOnlyContract = new Contract(tokenAddress, ERC20_ABI, provider);

// 可写合约（Signer）
const browserProvider = new BrowserProvider(window.ethereum);
const signer = await browserProvider.getSigner();
const writableContract = new Contract(tokenAddress, ERC20_ABI, signer);
```

### 5.2 读取数据（view / pure）

```typescript
// 调用 view 函数
const name = await readOnlyContract.name();
console.log("Name:", name);  // "USD Coin"

const symbol = await readOnlyContract.symbol();
console.log("Symbol:", symbol);  // "USDC"

const decimals = await readOnlyContract.decimals();
console.log("Decimals:", decimals);  // 6

const totalSupply = await readOnlyContract.totalSupply();
console.log("Total supply:", ethers.formatUnits(totalSupply, decimals));
// "1000000000.0"

// 读取余额
const balance = await readOnlyContract.balanceOf("0xd8dA...");
console.log("Balance:", ethers.formatUnits(balance, 6));
```

### 5.3 写入数据（交易）

```typescript
// 调用写函数
const tx = await writableContract.transfer(
  "0xRecipient...",
  ethers.parseUnits("100", 6)  // 100 USDC
);
console.log("TX hash:", tx.hash);

// 等待交易确认
const receipt = await tx.wait();
console.log("Status:", receipt!.status);  // 1 = 成功, 0 = 失败
console.log("Block:", receipt!.blockNumber);
console.log("Gas used:", receipt!.gasUsed.toString());
```

### 5.4 估算 Gas

```typescript
// 估算交易 Gas
const gasEstimate = await writableContract.transfer.estimateGas(
  "0xRecipient...",
  ethers.parseUnits("100", 6)
);
console.log("Estimated gas:", gasEstimate.toString());

// 估算 Gas 费用
const feeData = await provider.getFeeData();
const gasCost = gasEstimate * feeData.gasPrice!;
console.log("Gas cost:", ethers.formatEther(gasCost), "ETH");
```

### 5.5 事件监听

```typescript
// 监听 Transfer 事件
const filter = writableContract.filters.Transfer;

writableContract.on("Transfer", (from, to, amount, event) => {
  console.log(`Transfer: ${from} → ${to}: ${ethers.formatUnits(amount, 6)}`);
  console.log("Block:", event.log.blockNumber);
});

// 过滤特定地址的 Transfer
const fromFilter = writableContract.filters.Transfer("0xFrom...");
const toFilter = writableContract.filters.Transfer(null, "0xTo...");
const bothFilter = writableContract.filters.Transfer("0xFrom...", "0xTo...");

// 查询历史事件
const events = await writableContract.queryFilter(fromFilter, -10000);  // 最近 10000 区块
for (const event of events) {
  console.log(event.args);
}

// 只监听一次
writableContract.once("Transfer", (from, to, amount) => {
  console.log("First transfer:", from, to, amount);
});

// 移除监听
writableContract.removeAllListeners("Transfer");
```

### 5.6 使用 Human-Readable ABI

ethers.js 支持用字符串数组代替 JSON ABI：

```typescript
// ✅ 简洁的人类可读 ABI
const abi = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function balanceOf(address owner) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "event Transfer(address indexed from, address indexed to, uint256 value)",
];

// 等价于 JSON ABI
const jsonAbi = [
  {
    "inputs": [],
    "name": "name",
    "outputs": [{ "type": "string" }],
    "stateMutability": "view",
    "type": "function"
  },
  // ... 省略
];
```

---

## 6. 交易管理

### 6.1 发送交易

```typescript
import { parseEther, formatEther } from "ethers";

// 方式1：sendTransaction
const tx = await signer.sendTransaction({
  to: "0xRecipient...",
  value: parseEther("0.1"),
  // data: "0x...",        // 可选：合约调用数据
  // gasLimit: 21000,      // 可选：Gas 限制
  // maxFeePerGas: parseUnits("20", "gwei"),  // EIP-1559
  // maxPriorityFeePerGas: parseUnits("1", "gwei"),
});

// 方式2：send（等价）
const tx2 = await signer.send({
  to: "0xRecipient...",
  value: parseEther("0.1"),
});

// 等待确认
const receipt = await tx.wait();
console.log("Confirmed:", receipt!.status === 1);
console.log("Block:", receipt!.blockNumber);
console.log("Gas used:", receipt!.gasUsed.toString());
console.log("Effective gas price:", formatEther(receipt!.gasPrice!), "ETH");
console.log("Total fee:", formatEther(receipt!.gasUsed * receipt!.gasPrice!), "ETH");
```

### 6.2 交易状态轮询

```typescript
// 等待 N 个区块确认
const receipt = await tx.wait(3);  // 等 3 个区块确认

// 手动轮询
const txHash = "0x...";
let receipt = null;
while (!receipt) {
  receipt = await provider.getTransactionReceipt(txHash);
  if (!receipt) {
    console.log("Waiting...");
    await new Promise(r => setTimeout(r, 3000));  // 3 秒轮询
  }
}
console.log("Confirmed!");
```

### 6.3 替换交易

```typescript
// 发送一笔交易
const tx = await signer.sendTransaction({
  to: "0xRecipient...",
  value: parseEther("0.1"),
  nonce: await signer.getNonce(),
});

// 如果交易卡住了，用更高的 Gas 替换
const replacementTx = await signer.sendTransaction({
  to: "0xRecipient...",
  value: parseEther("0.1"),
  nonce: tx.nonce,              // 相同 nonce
  maxFeePerGas: parseUnits("50", "gwei"),  // 更高 Gas
});
```

---

## 7. 格式化与单位转换

```typescript
import { formatEther, parseEther, formatUnits, parseUnits, formatAddress } from "ethers";

// ETH ↔ Wei
parseEther("1.0");          // 1000000000000000000n (bigint)
formatEther(1_000_000_000_000_000_000n);  // "1.0"

// 任意精度单位
parseUnits("100", 6);       // 100000000n (USDC 6 decimals)
formatUnits(100000000n, 6); // "100.0"

parseUnits("50", "gwei");   // 50000000000n
formatUnits(50000000000n, "gwei");  // "50.0"

// 地址校验与简写
ethers.isAddress("0xd8dA...");  // true
ethers.getAddress("0xd8da...");  // 校验和地址
// 简写地址（自定义函数）
function shortAddr(addr: string) {
  return `${addr.slice(0, 6)}...${addr.slice(-4)}`;
}
```

---

## 8. React 集成模式

### 8.1 基础 Hook

```typescript
// hooks/useWallet.ts
import { useState, useEffect, useCallback } from "react";
import { BrowserProvider, formatEther } from "ethers";

export function useWallet() {
  const [provider, setProvider] = useState<BrowserProvider | null>(null);
  const [signer, setSigner] = useState<ethers.JsonRpcSigner | null>(null);
  const [address, setAddress] = useState<string>("");
  const [chainId, setChainId] = useState<number>(0);
  const [balance, setBalance] = useState<string>("0");

  const connect = useCallback(async () => {
    if (!window.ethereum) {
      alert("请安装 MetaMask");
      return;
    }
    const browserProvider = new BrowserProvider(window.ethereum);
    await browserProvider.send("eth_requestAccounts", []);
    const signer = await browserProvider.getSigner();
    const address = await signer.getAddress();
    const network = await browserProvider.getNetwork();
    const balance = await browserProvider.getBalance(address);

    setProvider(browserProvider);
    setSigner(signer);
    setAddress(address);
    setChainId(Number(network.chainId));
    setBalance(formatEther(balance));
  }, []);

  // 监听账户和网络变更
  useEffect(() => {
    if (!window.ethereum) return;

    const handleAccountsChanged = (accounts: string[]) => {
      if (accounts.length === 0) {
        // 用户断开连接
        setSigner(null);
        setAddress("");
        setBalance("0");
      } else {
        setAddress(accounts[0]);
        connect();  // 重新获取数据
      }
    };

    const handleChainChanged = (chainId: string) => {
      setChainId(parseInt(chainId, 16));
      window.location.reload();
    };

    window.ethereum.on("accountsChanged", handleAccountsChanged);
    window.ethereum.on("chainChanged", handleChainChanged);

    return () => {
      window.ethereum.removeListener("accountsChanged", handleAccountsChanged);
      window.ethereum.removeListener("chainChanged", handleChainChanged);
    };
  }, [connect]);

  return { provider, signer, address, chainId, balance, connect };
}
```

### 8.2 合约交互 Hook

```typescript
// hooks/useERC20.ts
import { useState, useEffect, useCallback } from "react";
import { Contract, BrowserProvider, formatUnits } from "ethers";

const ERC20_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function decimals() view returns (uint8)",
  "function balanceOf(address) view returns (uint256)",
  "function transfer(address to, uint256) returns (bool)",
];

export function useERC20(tokenAddress: string, userAddress: string) {
  const [name, setName] = useState("");
  const [symbol, setSymbol] = useState("");
  const [decimals, setDecimals] = useState(18);
  const [balance, setBalance] = useState("0");

  useEffect(() => {
    async function loadData() {
      if (!window.ethereum || !tokenAddress || !userAddress) return;
      const provider = new BrowserProvider(window.ethereum);
      const contract = new Contract(tokenAddress, ERC20_ABI, provider);
      const [name, symbol, decimals, balance] = await Promise.all([
        contract.name(),
        contract.symbol(),
        contract.decimals(),
        contract.balanceOf(userAddress),
      ]);
      setName(name);
      setSymbol(symbol);
      setDecimals(decimals);
      setBalance(formatUnits(balance, decimals));
    }
    loadData();
  }, [tokenAddress, userAddress]);

  const transfer = useCallback(async (to: string, amount: string) => {
    if (!window.ethereum) return;
    const provider = new BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const contract = new Contract(tokenAddress, ERC20_ABI, signer);
    const tx = await contract.transfer(to, parseUnits(amount, decimals));
    await tx.wait();
    // 刷新余额
    const newBalance = await contract.balanceOf(userAddress);
    setBalance(formatUnits(newBalance, decimals));
  }, [tokenAddress, userAddress, decimals]);

  return { name, symbol, decimals, balance, transfer };
}
```

> **生产环境推荐**：使用 [wagmi](https://wagmi.sh) + [viem](https://viem.sh) 替代手动 Hook。wagmi 提供了更完善的 React Hooks（自动缓存、自动刷新、错误重试等）。

---

## 9. 常用工具函数速查

```typescript
// 编码与解码
ethers.encodeBase58(bytes);
ethers.decodeBase58(string);
ethers.encodeRlp(object);
ethers.decodeRlp(hexString);

// 哈希
ethers.id("hello");              // keccak256("hello")
ethers.sha256(data);
ethers.ripemd160(data);

// ABI 编码
ethers.AbiCoder.defaultAbiCoder().encode(
  ["uint256", "address"],
  [100n, "0x1234..."]
);

// 地址
ethers.isAddress(addr);
ethers.getAddress(addr);         // 校验和地址
ethers.zeroPadValue(data, 32);

// 数值
ethers.toQuantity(42);           // "0x2a"

// 单位转换
ethers.parseEther("1.0");
ethers.formatEther(1_000_000_000_000_000_000n);
ethers.parseUnits("100", 6);
ethers.formatUnits(100000000n, 6);

// 时间
ethers.toUtf8String(bytes);
ethers.toUtf8Bytes("hello");

// 签名
ethers.verifyMessage(message, signature);
ethers.verifyTypedData(domain, types, message, signature);
```

---

## 10. v5 → v6 迁移对照表

| 操作 | ethers v5 | ethers v6 |
|------|-----------|-----------|
| Provider（浏览器） | `new ethers.providers.Web3Provider(window.ethereum)` | `new ethers.BrowserProvider(window.ethereum)` |
| Provider（RPC） | `new ethers.providers.JsonRpcProvider(url)` | `new ethers.JsonRpcProvider(url)` |
| 获取 Signer | `provider.getSigner()` | `await provider.getSigner()` |
| 合约地址 | `contract.address` | `await contract.getAddress()` |
| 格式化 ETH | `ethers.utils.formatEther(x)` | `ethers.formatEther(x)` |
| 解析 ETH | `ethers.utils.parseEther(x)` | `ethers.parseEther(x)` |
| 格式化单位 | `ethers.utils.formatUnits(x, d)` | `ethers.formatUnits(x, d)` |
| 大数类型 | `BigNumber` | `bigint`（原生） |
| 大数运算 | `BigNumber.from(x).add(y)` | `x + y`（bigint 原生） |
| 地址校验 | `ethers.utils.isAddress(x)` | `ethers.isAddress(x)` |
| 合约接口 | `ethers.utils.Interface(abi)` | `ethers.Interface(abi)` |
| Wallet 创建 | `new ethers.Wallet(pk, provider)` | `new ethers.Wallet(pk, provider)` |
| 助记词 | `ethers.Wallet.fromMnemonic(m)` | `ethers.Wallet.fromPhrase(m)` |
| WebSocket | `ethers.providers.WebSocketProvider` | `ethers.WebSocketProvider` |

> **迁移建议**：v6 不向后兼容 v5，需要逐行迁移。建议先在测试分支迁移，逐步替换。主要工作量在于 `BigNumber` → `bigint` 的类型转换。

---

## 快速参考卡

| 概念 | 关键记忆点 |
|------|-----------|
| Provider | 只读，不需要私钥，用于查询链上数据 |
| Signer | 可写，需要私钥或钱包授权，用于发送交易 |
| Contract | `new Contract(address, abi, providerOrSigner)` |
| 读取 | `await contract.methodName(args)` |
| 写入 | `const tx = await contract.methodName(args); await tx.wait()` |
| 事件监听 | `contract.on("Event", callback)` |
| 事件过滤 | `contract.filters.Event(from, to)` |
| 格式化 | `formatEther` / `formatUnits` / `parseEther` / `parseUnits` |
| v6 大数 | 使用原生 `bigint`，不是 `BigNumber` |
| BrowserProvider | `new BrowserProvider(window.ethereum)` |
| 等待确认 | `await tx.wait(confirmationCount)` |
