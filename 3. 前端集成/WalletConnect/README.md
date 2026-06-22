# WalletConnect

WalletConnect 是连接 DApp 和移动钱包的开放协议，通过二维码实现跨设备连接。

## 文档索引

| 文档 | 内容 |
|------|------|
| [WalletConnect v2 完整指南](./WalletConnect%20v2%20完整指南.md) | v2 架构（Relay/Pairing/Session）、DApp 端集成（SignClient 初始化/连接/签名/断开）、React Hook、与 wagmi 集成 |

## 版本状态

| 版本 | 状态 | 说明 |
|------|------|------|
| v1 | ❌ 已弃用 | 基于桥接的通信 |
| v2 | ✅ 当前版本 | 多链、多会话、端到端加密 |

## 快速开始

```bash
# 获取 Project ID（免费）
# 访问 https://cloud.walletconnect.com 注册

npm install @walletconnect/sign-client @walletconnect/modal
```

## 相关章节

- [MetaMask 集成](../MetaMask%20集成/) — 浏览器端钱包接入
- [Web3.js 与现代前端栈](../Web3.js/) — wagmi + RainbowKit 内置 WalletConnect 支持
