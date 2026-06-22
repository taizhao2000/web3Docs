# MetaMask 集成

MetaMask 是最流行的浏览器钱包扩展，DApp 通过 MetaMask 与用户交互。

## 文档索引

| 文档 | 内容 |
|------|------|
| [MetaMask 集成完整指南](./MetaMask%20集成完整指南.md) | EIP-6963 钱包发现、EIP-1193 标准、连接/断开/账户切换/网络切换、添加自定义链、签名（personal_sign/EIP-712）、交易发送、错误码处理、完整 React Hook |

## 快速开始

```typescript
// 检测并连接 MetaMask
if (window.ethereum) {
  const accounts = await window.ethereum.request({
    method: "eth_requestAccounts"
  });
  console.log("已连接:", accounts[0]);
}
```

## 相关章节

- [ethers.js v6](../ethers.js/) — BrowserProvider 封装了 MetaMask 的 window.ethereum
- [WalletConnect v2](../WalletConnect/) — 移动端钱包连接方案
