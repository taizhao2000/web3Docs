# WalletConnect v2 完整指南

> WalletConnect 是连接 DApp 和移动钱包的开放协议。v2 于 2023 年发布，完全重构了架构，支持多链、多会话和推送通知。

---

## 1. WalletConnect 是什么

WalletConnect 解决了一个核心问题：**移动端用户如何与 DApp 交互**。

```
桌面浏览器 DApp ←→ WalletConnect Relay ←→ 手机钱包 App
    │                                        │
    │  展示二维码                              │  扫码连接
    │  发送签名请求                            │  确认签名
    │  接收签名结果                            │  返回签名
```

### v1 vs v2 核心差异

| 特性 | v1（已弃用） | v2（当前） |
|------|-------------|-----------|
| 架构 | 桥接服务器转发 | Relay 中继 + 客户端加密 |
| 多链 | ❌ 单链 | ✅ 多链多会话 |
| 会话 | 1:1 | 1:N（一个 DApp 连多个钱包） |
| 加密 | 弱 | 端到端加密 |
| 推送 | ❌ | ✅ 推送通知 |
| 持久化 | ❌ | ✅ 会话持久化 |
| 多账户 | ❌ | ✅ 多账户 |

---

## 2. 架构

```
┌──────────────┐     Waku     ┌──────────────┐     Waku     ┌──────────────┐
│   DApp       │ ←─────────→ │    Relay     │ ←─────────→ │   Wallet     │
│ (App)        │   加密消息   │  (Middleware) │   加密消息   │  (App)       │
└──────────────┘             └──────────────┘             └──────────────┘
      │                             │                             │
      │ 1. 生成 pairing URI         │                             │
      │ 2. 显示二维码                │                             │
      │                             │ 3. 钱包扫描二维码            │
      │                             │ 4. 建立 pairing              │
      │                             │ 5. 协商会话参数              │
      │                             │ 6. 用户确认                  │
      │ 7. 收到会话建立              │                             │
      │ 8. 发送签名请求               │                             │
      │                             │ 9. 转发签名请求              │
      │                             │ 10. 用户确认签名             │
      │ 11. 收到签名结果              │                             │
```

### 核心概念

| 概念 | 说明 |
|------|------|
| **Pairing** | 配对关系，通过 URI/二维码建立 |
| **Session** | 会话，在 pairing 基础上协商出具体的链和权限 |
| **Relay** | 中继服务器，转发加密消息（不看到内容） |
| **Namespace** | 命名空间，定义支持的链和方法（如 eip155:1） |
| **Proposal** | 会话提案，DApp 提出需求（链、方法、事件） |
| **Settlement** | 会话确认，钱包接受或拒绝提案 |

---

## 3. DApp 端集成

### 3.1 安装

```bash
# 核心 SDK
npm install @walletconnect/modal @walletconnect/ethers

# 或使用 SignClient + Modal（更灵活）
npm install @walletconnect/sign-client @walletconnect/modal

# 与 ethers.js 集成
npm install ethers @walletconnect/ethers
```

### 3.2 初始化 SignClient

```typescript
// lib/walletconnect.ts
import SignClient from "@walletconnect/sign-client";
import { WalletConnectModal } from "@walletconnect/modal";

let signClient: SignClient | null = null;

export async function getSignClient(): Promise<SignClient> {
  if (signClient) return signClient;

  signClient = await SignClient.init({
    projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID!,  // 在 cloud.walletconnect.com 注册
    relayUrl: "wss://relay.walletconnect.com",
    metadata: {
      name: "MyDApp",
      description: "My awesome DApp",
      url: "https://mydapp.com",
      icons: ["https://mydapp.com/logo.png"],
    },
  });

  // 监听会话事件
  signClient.on("session_event", (event) => {
    console.log("Session event:", event);
    // 账户或链变更
  });

  signClient.on("session_update", ({ topic, params }) => {
    console.log("Session updated:", topic, params);
    // 钱包更新了会话（如切换账户）
  });

  signClient.on("session_delete", ({ topic }) => {
    console.log("Session deleted:", topic);
    // 钱包断开连接
  });

  signClient.on("session_expire", ({ topic }) => {
    console.log("Session expired:", topic);
    // 会话过期
  });

  return signClient;
}
```

### 3.3 连接钱包

```typescript
import { getSignClient } from "../lib/walletconnect";
import { WalletConnectModal } from "@walletconnect/modal";

export async function connectWalletConnect() {
  const client = await getSignClient();

  // 创建配对 URI
  const { uri, approval } = await client.connect({
    requiredNamespaces: {
      // EIP-155（以太坊系链）
      eip155: {
        chains: [
          "eip155:1",       // Ethereum Mainnet
          "eip155:137",     // Polygon
          "eip155:42161",   // Arbitrum
        ],
        methods: [
          "eth_sendTransaction",
          "eth_signTransaction",
          "eth_sign",
          "personal_sign",
          "eth_signTypedData_v4",
        ],
        events: ["accountsChanged", "chainChanged"],
      },
    },
    optionalNamespaces: {
      eip155: {
        chains: ["eip155:11155111"],  // Sepolia（可选）
        methods: ["eth_sendTransaction"],
        events: ["accountsChanged"],
      },
    },
  });

  // 显示二维码模态框
  if (uri) {
    const modal = new WalletConnectModal({
      projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID!,
      chains: [1, 137, 42161],
    });

    modal.openModal({ uri });

    // 等待用户扫码连接
    try {
      const session = await approval();
      modal.closeModal();
      console.log("Connected:", session);

      // 获取账户
      const accounts = getAccountsFromSession(session);
      return accounts;
    } catch (error) {
      modal.closeModal();
      throw error;
    }
  }
}

// 从会话中提取账户
function getAccountsFromSession(session: any): string[] {
  const accounts: string[] = [];
  for (const [namespace, data] of Object.entries(session.namespaces)) {
    if (namespace === "eip155") {
      for (const account of (data as any).accounts) {
        // 格式：eip155:1:0xABC...
        const [chainId, address] = account.split(":").slice(1);
        accounts.push(address);
      }
    }
  }
  return accounts;
}
```

### 3.4 发送交易

```typescript
export async function sendTransaction(
  client: SignClient,
  topic: string,
  from: string,
  to: string,
  value: string,        // hex
  data: string = "0x"  // hex
): Promise<string> {
  const request = {
    topic,
    request: {
      method: "eth_sendTransaction",
      params: [{
        from,
        to,
        value,
        data,
      }],
    },
    chainId: "eip155:1",  // 目标链
  };

  // 发送请求到钱包（钱包会弹窗确认）
  const result = await client.request(request);
  return result as string;  // 交易哈希
}

// 使用
const txHash = await sendTransaction(
  client,
  session.topic,
  "0xSender...",
  "0xRecipient...",
  "0xde0b6b3a7640000",  // 1 ETH
  "0x"
);
console.log("TX hash:", txHash);
```

### 3.5 签名消息

```typescript
// personal_sign
export async function signMessage(
  client: SignClient,
  topic: string,
  address: string,
  message: string
): Promise<string> {
  const result = await client.request({
    topic,
    request: {
      method: "personal_sign",
      params: [
        ethers.hexlify(ethers.toUtf8Bytes(message)),
        address,
      ],
    },
    chainId: "eip155:1",
  });
  return result as string;
}

// eth_signTypedData_v4
export async function signTypedData(
  client: SignClient,
  topic: string,
  address: string,
  typedData: object
): Promise<string> {
  const result = await client.request({
    topic,
    request: {
      method: "eth_signTypedData_v4",
      params: [address, JSON.stringify(typedData)],
    },
    chainId: "eip155:1",
  });
  return result as string;
}
```

### 3.6 断开会话

```typescript
export async function disconnect(client: SignClient, topic: string) {
  await client.disconnect({
    topic,
    reason: {
      code: 6000,           // 用户主动断开
      message: "User disconnected",
    },
  });
  // 清除本地状态
}
```

---

## 4. React 集成

### 4.1 Hook

```typescript
// hooks/useWalletConnect.ts
import { useState, useEffect, useCallback } from "react";
import SignClient from "@walletconnect/sign-client";
import { getSignClient } from "../lib/walletconnect";

export function useWalletConnect() {
  const [client, setClient] = useState<SignClient | null>(null);
  const [session, setSession] = useState<any>(null);
  const [address, setAddress] = useState<string>("");
  const [chainId, setChainId] = useState<number>(0);
  const [isConnecting, setIsConnecting] = useState(false);

  // 初始化
  useEffect(() => {
    (async () => {
      const c = await getSignClient();
      setClient(c);

      // 恢复已有会话
      if (c.session.keys.length > 0) {
        const lastKey = c.session.keys[c.session.keys.length - 1];
        const lastSession = c.session.get(lastKey);
        setSession(lastSession);
        const accounts = getAccountsFromSession(lastSession);
        if (accounts.length > 0) {
          setAddress(accounts[0]);
          setChainId(1);
        }
      }
    })();
  }, []);

  const connect = useCallback(async () => {
    if (!client) return;
    setIsConnecting(true);
    try {
      // ... connectWalletConnect 逻辑
      // setSession, setAddress, setChainId
    } finally {
      setIsConnecting(false);
    }
  }, [client]);

  const disconnect = useCallback(async () => {
    if (!client || !session) return;
    await client.disconnect({
      topic: session.topic,
      reason: { code: 6000, message: "User disconnected" },
    });
    setSession(null);
    setAddress("");
    setChainId(0);
  }, [client, session]);

  return { client, session, address, chainId, isConnecting, connect, disconnect };
}
```

---

## 5. 与 wagmi 集成（推荐）

wagmi v2 内置了 WalletConnect 支持，**不需要手动管理会话**：

```typescript
// app/providers.tsx
import { WagmiProvider, createConfig } from "wagmi";
import { mainnet, sepolia, polygon, arbitrum } from "wagmi/chains";
import { walletConnect } from "wagmi/connectors";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

const config = createConfig({
  chains: [mainnet, sepolia, polygon, arbitrum],
  connectors: [
    walletConnect({
      projectId: process.env.NEXT_PUBLIC_WC_PROJECT_ID!,
      // 可选：显示的二维码模态框
      showQrModal: true,
      // 可选：支持的链
      optionalChains: [mainnet, polygon, arbitrum],
      // 可选：dApp 元数据
      metadata: {
        name: "MyDApp",
        description: "My awesome DApp",
        url: "https://mydapp.com",
        icons: ["https://mydapp.com/logo.png"],
      },
    }),
  ],
});

const queryClient = new QueryClient();

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        {children}
      </QueryClientProvider>
    </WagmiProvider>
  );
}
```

```typescript
// 组件中使用
import { useConnect, useAccount, useDisconnect } from "wagmi";

function ConnectButton() {
  const { connect, connectors } = useConnect();
  const { address, isConnected } = useAccount();
  const { disconnect } = useDisconnect();

  if (isConnected) {
    return (
      <div>
        <p>已连接: {address}</p>
        <button onClick={() => disconnect()}>断开</button>
      </div>
    );
  }

  return (
    <div>
      {connectors.map(connector => (
        <button
          key={connector.id}
          onClick={() => connect({ connector })}
        >
          {connector.name}
        </button>
      ))}
    </div>
  );
}
```

> **优势**：wagmi 自动处理了会话恢复、多链切换、账户变更监听等复杂逻辑。

---

## 6. 获取 Project ID

WalletConnect Relay 服务需要 Project ID（免费）：

1. 访问 [cloud.walletconnect.com](https://cloud.walletconnect.com)
2. 注册账号
3. 创建新项目
4. 获取 Project ID
5. 配置到环境变量

```bash
# .env.local
NEXT_PUBLIC_WC_PROJECT_ID=your_project_id_here
```

---

## 7. 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 二维码不显示 | Modal 未正确初始化 | 检查 projectId 配置 |
| 扫码后无响应 | Relay 服务器不可达 | 检查网络连接 |
| 连接后无法发送交易 | 会话未包含目标链 | 检查 requiredNamespaces 配置 |
| 会话丢失 | 页面刷新后未恢复 | 检查 SignClient 初始化时是否恢复会话 |
| 签名失败 | 方法未在 methods 中声明 | 在 connect 时声明所需方法 |
| 移动端不弹窗 | 推送通知未配置 | 注册推送服务或使用 deep link |
