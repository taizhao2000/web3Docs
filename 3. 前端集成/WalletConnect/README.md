# WalletConnect

## 概述

WalletConnect 是一种开放协议，用于 DApp 和移动钱包之间的安全连接。

## 核心特性

- **二维码连接**：DApp 展示二维码，钱包扫码连接
- **多链支持**：支持多条区块链
- **推送通知**：交易状态推送
- **会话管理**：安全的会话生命周期

## 集成方式

```javascript
import WalletConnect from '@walletconnect/client';
import QRCodeModal from '@walletconnect/qrcode-modal';

// 创建连接
const connector = new WalletConnect({
  bridge: 'https://bridge.walletconnect.org',
});

// 生成二维码
if (!connector.connected) {
  await connector.createSession();
  const uri = connector.uri;
  QRCodeModal.open(uri);
}

// 监听连接
connector.on('connect', (error, payload) => {
  const { accounts, chainId } = payload.params[0];
});

// 发送交易
const txHash = await connector.sendTransaction({
  from: address,
  to: recipient,
  value: '0x...',
  data: '0x...',
});
```

## 版本演进

| 版本 | 状态 | 说明 |
|------|------|------|
| v1 | 已弃用 | 基于桥接的通信 |
| v2 | 当前版本 | 多链、多会话支持 |
