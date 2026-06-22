# MetaMask 集成完整指南

> MetaMask 是最流行的浏览器钱包扩展，DApp 通过 MetaMask 与用户交互。本文覆盖从钱包发现到生产级集成的完整流程。

---

## 1. MetaMask 是什么

MetaMask 是一个**浏览器钱包扩展**，充当用户与 DApp 之间的桥梁：

```
用户 ←→ MetaMask（私钥管理 + 签名） ←→ DApp 前端 ←→ 以太坊节点
```

### MetaMask 的角色

| 角色 | 说明 |
|------|------|
| 私钥管理 | 私钥永远不离开 MetaMask，DApp 无法直接访问 |
| 交易签名 | DApp 构造交易 → MetaMask 弹窗 → 用户确认 → 签名后广播 |
| 网络管理 | 用户可以切换链、添加自定义链 |
| 账户管理 | 用户可以有多个账户，DApp 获取当前选中的账户 |

---

## 2. EIP-6963：钱包发现（推荐）

### 2.1 为什么不用 `window.ethereum`

```javascript
// ❌ 旧方式：直接读取 window.ethereum
if (window.ethereum) {
  // 问题1：多钱包冲突（MetaMask + Coinbase Wallet + Rabby 同时安装时）
  // 问题2：注入时机不确定（页面加载时可能还没注入）
  // 问题3：无法区分是哪个钱包
}
```

### 2.2 EIP-6963 标准发现

EIP-6963 通过事件广播解决多钱包冲突问题：

```typescript
// utils/eip6963.ts
interface EIP6963ProviderInfo {
  uuid: string;
  name: string;
  icon: string;
  rdns: string;  // 反向域名标识（如 "io.metamask"）
}

interface EIP6963ProviderDetail {
  info: EIP6963ProviderInfo;
  provider: EIP1193Provider;  // EIP-1193 标准 Provider
}

interface EIP1193Provider {
  request: (args: { method: string; params?: unknown[] }) => Promise<unknown>;
  on: (event: string, listener: (...args: any[]) => void) => void;
  removeListener: (event: string, listener: (...args: any[]) => void) => void;
}

let providers: EIP6963ProviderDetail[] = [];

// 监听钱包注入事件
function announceProvider(providerDetail: EIP6963ProviderDetail) {
  if (!providers.some(p => p.info.uuid === providerDetail.info.uuid)) {
    providers.push(providerDetail);
  }
}

// EIP-6963 事件必须在页面加载时立即注册
window.addEventListener("eip6963:announceProvider", (event) => {
  announceProvider((event as CustomEvent).detail);
});

// 请求已注入的钱包
window.dispatchEvent(new Event("eip6963:requestProvider"));

// 获取所有钱包
export function getWallets(): EIP6963ProviderDetail[] {
  return providers;
}

// 获取特定钱包
export function getWallet(rdns: string): EIP6963ProviderDetail | undefined {
  return providers.find(p => p.info.rdns === rdns);
}

// 获取 MetaMask
export function getMetaMask(): EIP6963ProviderDetail | undefined {
  return providers.find(p => p.info.rdns === "io.metamask");
}
```

### 2.3 React Hook 使用

```typescript
// hooks/useWallets.ts
import { useState, useEffect } from "react";
import { getWallets, EIP6963ProviderDetail } from "../utils/eip6963";

export function useWallets() {
  const [wallets, setWallets] = useState<EIP6963ProviderDetail[]>([]);

  useEffect(() => {
    // 初始化
    const updateWallets = () => setWallets(getWallets());
    updateWallets();

    // 监听新钱包注入（可能延迟注入）
    window.addEventListener("eip6963:announceProvider", updateWallets);

    return () => {
      window.removeEventListener("eip6963:announceProvider", updateWallets);
    };
  }, []);

  return wallets;
}
```

```typescript
// 组件中使用
function WalletSelector() {
  const wallets = useWallets();

  return (
    <div>
      {wallets.map(wallet => (
        <button key={wallet.info.uuid} onClick={() => connect(wallet)}>
          <img src={wallet.info.icon} alt={wallet.info.name} />
          {wallet.info.name}
        </button>
      ))}
    </div>
  );
}
```

> **兼容性**：EIP-6963 于 2023 年推出，MetaMask、Coinbase Wallet、Rabby 等主流钱包均已支持。对于不支持 EIP-6963 的旧钱包，回退到 `window.ethereum`。

---

## 3. 连接与断开

### 3.1 连接钱包

```typescript
// 请求用户授权连接
async function connectWallet(provider: EIP1193Provider): Promise<string[]> {
  try {
    const accounts = await provider.request({
      method: "eth_requestAccounts",  // EIP-1102 标准
    }) as string[];
    return accounts;
  } catch (error: any) {
    if (error.code === 4001) {
      // 用户拒绝了连接请求
      console.log("用户拒绝了连接");
    } else if (error.code === -32002) {
      // 已经有连接请求在等待用户操作
      console.log("请在 MetaMask 中确认连接请求");
    } else {
      console.error("连接失败:", error);
    }
    return [];
  }
}

// 静默检查是否已连接（不弹窗）
async function checkConnection(provider: EIP1193Provider): Promise<string[]> {
  const accounts = await provider.request({
    method: "eth_accounts",  // EIP-1193 标准，不弹窗
  }) as string[];
  return accounts;
}
```

### 3.2 主动断开连接

```typescript
// MetaMask 没有提供断开连接的 API
// "断开"实际上是在 DApp 端清除状态
function disconnect() {
  // 清除本地存储的钱包信息
  localStorage.removeItem("walletProvider");
  // 重置状态
  setAccount("");
  setChainId(0);
  // 注意：MetaMask 仍然保持授权，下次连接不需要再次弹窗
}
```

> **MetaMask 的权限模型**：一旦用户授权了连接，权限会持久化。清除 DApp 状态不会清除 MetaMask 的授权。要真正撤销权限，用户需要在 MetaMask 中手动断开。

---

## 4. 账户与网络监听

### 4.1 EIP-1193 标准事件

```typescript
// 账户变更
provider.on("accountsChanged", (accounts: string[]) => {
  if (accounts.length === 0) {
    // 用户在 MetaMask 中断开了连接
    console.log("用户断开连接");
    handleDisconnect();
  } else {
    console.log("账户切换:", accounts[0]);
    handleAccountChange(accounts[0]);
  }
});

// 链变更
provider.on("chainChanged", (chainId: string) => {
  // chainId 是十六进制字符串，如 "0x1"（主网）
  console.log("链切换:", parseInt(chainId, 16));
  // EIP-1193 建议：链变更时刷新页面
  window.location.reload();
});

// 断开连接
provider.on("disconnect", (error: Error) => {
  console.log("断开连接:", error);
  handleDisconnect();
});

// ⚠️ 必须在组件卸载时移除监听器
provider.removeListener("accountsChanged", handler);
```

### 4.2 完整状态管理

```typescript
class MetaMaskManager {
  private provider: EIP1193Provider | null = null;
  private account: string = "";
  private chainId: number = 0;

  async init() {
    // 通过 EIP-6963 获取 MetaMask
    const metamask = getMetaMask();
    if (!metamask) {
      console.log("MetaMask 未安装");
      return false;
    }
    this.provider = metamask.provider;

    // 检查是否已连接
    const accounts = await this.checkConnection();
    if (accounts.length > 0) {
      this.account = accounts[0];
      this.chainId = await this.getChainId();
    }

    // 注册事件监听
    this.registerEvents();

    return true;
  }

  private registerEvents() {
    if (!this.provider) return;

    this.provider.on("accountsChanged", (accounts: string[]) => {
      if (accounts.length === 0) {
        this.account = "";
      } else {
        this.account = accounts[0];
      }
      this.notifyUpdate();
    });

    this.provider.on("chainChanged", (chainId: string) => {
      this.chainId = parseInt(chainId, 16);
      this.notifyUpdate();
      // 链变更时刷新页面
      window.location.reload();
    });
  }

  async connect() {
    if (!this.provider) throw new Error("MetaMask not initialized");
    const accounts = await this.provider.request({
      method: "eth_requestAccounts",
    }) as string[];
    this.account = accounts[0];
    this.chainId = await this.getChainId();
    this.notifyUpdate();
  }

  private async checkConnection(): Promise<string[]> {
    return await this.provider!.request({ method: "eth_accounts" }) as string[];
  }

  private async getChainId(): Promise<number> {
    const chainId = await this.provider!.request({ method: "eth_chainId" }) as string;
    return parseInt(chainId, 16);
  }

  private notifyUpdate() {
    window.dispatchEvent(new CustomEvent("metamask:update", {
      detail: { account: this.account, chainId: this.chainId },
    }));
  }
}
```

---

## 5. 网络管理

### 5.1 切换网络

```typescript
// 切换到指定链
async function switchChain(chainId: number): Promise<boolean> {
  try {
    await provider.request({
      method: "wallet_switchEthereumChain",
      params: [{ chainId: `0x${chainId.toString(16)}` }],
    });
    return true;
  } catch (error: any) {
    // error code 4902：链未添加到 MetaMask
    if (error.code === 4902) {
      return await addChain(chainId);
    }
    // error code 4001：用户拒绝
    if (error.code === 4001) {
      console.log("用户拒绝了切换链");
    }
    return false;
  }
}
```

### 5.2 添加自定义链

```typescript
// 添加自定义链（如本地测试链、L2 等）
async function addChain(chainConfig: {
  chainId: number;
  chainName: string;
  nativeCurrency: { name: string; symbol: string; decimals: number };
  rpcUrls: string[];
  blockExplorerUrls: string[];
}): Promise<boolean> {
  try {
    await provider.request({
      method: "wallet_addEthereumChain",
      params: [{
        chainId: `0x${chainConfig.chainId.toString(16)}`,
        chainName: chainConfig.chainName,
        nativeCurrency: chainConfig.nativeCurrency,
        rpcUrls: chainConfig.rpcUrls,
        blockExplorerUrls: chainConfig.blockExplorerUrls,
      }],
    });
    return true;
  } catch (error: any) {
    console.error("添加链失败:", error);
    return false;
  }
}

// 常用链配置
const CHAINS = {
  mainnet: {
    chainId: 1,
    chainName: "Ethereum Mainnet",
    nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
    rpcUrls: ["https://mainnet.infura.io/v3/YOUR_KEY"],
    blockExplorerUrls: ["https://etherscan.io"],
  },
  sepolia: {
    chainId: 11155111,
    chainName: "Sepolia Testnet",
    nativeCurrency: { name: "Sepolia Ether", symbol: "ETH", decimals: 18 },
    rpcUrls: ["https://sepolia.infura.io/v3/YOUR_KEY"],
    blockExplorerUrls: ["https://sepolia.etherscan.io"],
  },
  arbitrum: {
    chainId: 42161,
    chainName: "Arbitrum One",
    nativeCurrency: { name: "Ether", symbol: "ETH", decimals: 18 },
    rpcUrls: ["https://arb1.arbitrum.io/rpc"],
    blockExplorerUrls: ["https://arbiscan.io"],
  },
  local: {
    chainId: 31337,
    chainName: "Hardhat Local",
    nativeCurrency: { name: "ETH", symbol: "ETH", decimals: 18 },
    rpcUrls: ["http://127.0.0.1:8545"],
    blockExplorerUrls: [],
  },
};

// 使用
await switchChain(CHAINS.arbitrum.chainId);
```

### 5.3 确保在正确的链上

```typescript
const REQUIRED_CHAIN_ID = 1;  // 主网

async function ensureCorrectChain(): Promise<boolean> {
  const currentChainId = await provider.request({ method: "eth_chainId" });
  const currentId = parseInt(currentChainId as string, 16);

  if (currentId === REQUIRED_CHAIN_ID) {
    return true;
  }

  // 弹窗提示用户切换
  const confirmed = confirm(
    `请在 MetaMask 中切换到 ${CHAIN_NAMES[REQUIRED_CHAIN_ID]} 网络`
  );
  if (!confirmed) return false;

  return await switchChain(REQUIRED_CHAIN_ID);
}
```

---

## 6. 签名

### 6.1 个人消息签名

```typescript
// 签名一条人类可读的消息
async function signMessage(message: string, address: string): Promise<string> {
  const signature = await provider.request({
    method: "personal_sign",
    params: [
      ethers.hexlify(ethers.toUtf8Bytes(message)),  // 消息转 hex
      address,
    ],
  }) as string;
  return signature;
}

// 使用
const sig = await signMessage("登录 MyDApp", account);
console.log("签名:", sig);

// 验证签名
const recovered = ethers.verifyMessage("登录 MyDApp", sig);
console.log("验证地址:", recovered);
console.log("匹配:", recovered === account);
```

### 6.2 EIP-712 类型化数据签名

EIP-712 让用户可以看到结构化的签名内容，而不是一串十六进制：

```typescript
// EIP-712 类型化数据签名
const typedData = {
  domain: {
    name: "MyDApp",
    version: "1",
    chainId: 1,
    verifyingContract: "0xContract...",
  },
  types: {
    Permit: [
      { name: "owner", type: "address" },
      { name: "spender", type: "address" },
      { name: "value", type: "uint256" },
      { name: "nonce", type: "uint256" },
      { name: "deadline", type: "uint256" },
    ],
  },
  message: {
    owner: "0xOwner...",
    spender: "0xSpender...",
    value: "1000000000000000000",
    nonce: "0",
    deadline: "2000000000",
  },
};

async function signTypedData(address: string): Promise<string> {
  const signature = await provider.request({
    method: "eth_signTypedData_v4",
    params: [address, JSON.stringify(typedData)],
  }) as string;
  return signature;
}

// 用户在 MetaMask 中看到的是结构化信息：
// ┌─────────────────────────────────────┐
// │  签名请求                             │
// │  ─────────────────────────────────   │
// │  Domain: MyDApp v1                   │
// │  ─────────────────────────────────   │
// │  owner:    0xOwner...                │
// │  spender:  0xSpender...              │
// │  value:    1 ETH                     │
// │  nonce:    0                         │
// │  deadline: 2033-05-18                │
// └─────────────────────────────────────┘
```

---

## 7. 发送交易

### 7.1 基础 ETH 转账

```typescript
async function sendETH(from: string, to: string, amount: string) {
  const txHash = await provider.request({
    method: "eth_sendTransaction",
    params: [{
      from,
      to,
      value: "0x" + ethers.parseEther(amount).toString(16),  // hex
      // gas: "0x...",           // 可选
      // gasPrice: "0x...",      // 可选（Legacy）
      // maxFeePerGas: "0x...",  // EIP-1559
      // maxPriorityFeePerGas: "0x...",
    }],
  }) as string;

  console.log("TX hash:", txHash);
  return txHash;
}
```

### 7.2 合约调用

```typescript
// 使用 ethers.js 的 Contract（推荐）
async function transferToken(
  contractAddress: string,
  abi: any[],
  to: string,
  amount: bigint
) {
  const provider = new ethers.BrowserProvider(window.ethereum);
  const signer = await provider.getSigner();
  const contract = new ethers.Contract(contractAddress, abi, signer);
  const tx = await contract.transfer(to, amount);
  await tx.wait();
}

// 或直接构造 calldata
async function callContract(from: string, contractAddress: string, data: string) {
  const txHash = await provider.request({
    method: "eth_sendTransaction",
    params: [{
      from,
      to: contractAddress,
      data,  // ABI 编码的调用数据
    }],
  }) as string;
  return txHash;
}
```

---

## 8. 错误处理

### 8.1 MetaMask 错误码

| Code | 含义 | 处理方式 |
|------|------|---------|
| `4001` | 用户拒绝了请求 | 提示用户需要确认操作 |
| `-32603` | 内部错误 | 检查参数是否正确 |
| `-32002` | 请求已挂起（等待用户操作） | 提示用户在 MetaMask 中确认 |
| `4900` | 链断开 | 检查网络连接 |
| `4901` | 链未连接 | 检查 RPC URL |
| `4902` | 链未添加 | 调用 `wallet_addEthereumChain` |
| `-32601` | 方法不存在 | 检查方法名拼写 |

### 8.2 完整错误处理

```typescript
function handleMetaMaskError(error: any): string {
  if (!error) return "未知错误";

  switch (error.code) {
    case 4001:
      return "用户拒绝了请求";
    case -32603:
      return `内部错误: ${error.message}`;
    case -32002:
      return "请在 MetaMask 中确认挂起的请求";
    case 4902:
      return "当前网络未添加，请先添加网络";
    case -32601:
      return `方法不存在: ${error.message}`;
    default:
      return `错误 ${error.code}: ${error.message}`;
  }
}

// 使用
try {
  await connectWallet();
} catch (error) {
  const message = handleMetaMaskError(error);
  showToast(message, "error");
}
```

---

## 9. 完整 React Hook

```typescript
// hooks/useMetaMask.ts
import { useState, useEffect, useCallback } from "react";
import { ethers } from "ethers";

interface WalletState {
  account: string;
  chainId: number;
  isConnecting: boolean;
  error: string | null;
}

export function useMetaMask() {
  const [state, setState] = useState<WalletState>({
    account: "",
    chainId: 0,
    isConnecting: false,
    error: null,
  });

  // 检查 MetaMask 是否安装
  const isInstalled = useCallback(() => {
    return typeof window.ethereum !== "undefined";
  }, []);

  // 连接
  const connect = useCallback(async () => {
    if (!isInstalled()) {
      setState(prev => ({ ...prev, error: "请安装 MetaMask" }));
      return;
    }

    setState(prev => ({ ...prev, isConnecting: true, error: null }));

    try {
      const provider = new ethers.BrowserProvider(window.ethereum);
      const accounts = await provider.send("eth_requestAccounts", []);
      const network = await provider.getNetwork();

      setState({
        account: accounts[0],
        chainId: Number(network.chainId),
        isConnecting: false,
        error: null,
      });
    } catch (error: any) {
      let msg = "连接失败";
      if (error.code === 4001) msg = "用户拒绝了连接请求";
      else if (error.code === -32002) msg = "请在 MetaMask 中确认连接请求";
      setState(prev => ({ ...prev, isConnecting: false, error: msg }));
    }
  }, [isInstalled]);

  // 断开（清除状态）
  const disconnect = useCallback(() => {
    setState({
      account: "",
      chainId: 0,
      isConnecting: false,
      error: null,
    });
  }, []);

  // 自动检查是否已连接
  useEffect(() => {
    if (!isInstalled()) return;

    (async () => {
      const provider = new ethers.BrowserProvider(window.ethereum);
      const accounts = await provider.send("eth_accounts", []);
      if (accounts.length > 0) {
        const network = await provider.getNetwork();
        setState(prev => ({
          ...prev,
          account: accounts[0],
          chainId: Number(network.chainId),
        }));
      }
    })();
  }, [isInstalled]);

  // 监听账户和网络变更
  useEffect(() => {
    if (!isInstalled()) return;

    const handleAccountsChanged = (accounts: string[]) => {
      if (accounts.length === 0) {
        disconnect();
      } else {
        setState(prev => ({ ...prev, account: accounts[0] }));
      }
    };

    const handleChainChanged = (chainId: string) => {
      setState(prev => ({ ...prev, chainId: parseInt(chainId, 16) }));
      window.location.reload();
    };

    window.ethereum.on("accountsChanged", handleAccountsChanged);
    window.ethereum.on("chainChanged", handleChainChanged);

    return () => {
      window.ethereum.removeListener("accountsChanged", handleAccountsChanged);
      window.ethereum.removeListener("chainChanged", handleChainChanged);
    };
  }, [isInstalled, disconnect]);

  return { ...state, connect, disconnect, isInstalled };
}
```

---

## 10. 最佳实践

| 实践 | 说明 | 原因 |
|------|------|------|
| **用 EIP-6963 发现钱包** | 替代 `window.ethereum` | 多钱包不冲突 |
| **链变更时刷新页面** | `window.location.reload()` | 避免状态不一致 |
| **不自动请求连接** | 用户点击按钮时才请求 | 避免弹窗骚扰 |
| **优雅降级** | 未安装时引导安装 | 提升用户体验 |
| **错误码处理** | 4001/-32002 等 | 给用户正确提示 |
| **EIP-712 签名** | 替代 `personal_sign` | 用户可读的结构化数据 |
| **nonce 不前端管理** | 让 MetaMask 处理 nonce | 避免 nonce 冲突 |
| **Gas 估算** | 发送前估算 Gas | 避免交易失败 |

> **生产环境推荐**：使用 [wagmi](https://wagmi.sh) + [RainbowKit](https://rainbowkit.com)，它们已处理了所有边界情况，包括 EIP-6963 发现、多链切换、错误处理等。
