# Infura

## 概述

Infura 是 ConsenSys 旗下最流行的以太坊节点即服务（NaaS）平台，提供高性能的 API 访问。

## 核心功能

- **JSON-RPC API**：标准以太坊 RPC 接口
- **IPFS 网关**：IPFS 内容访问
- **多链支持**：Ethereum、Polygon、Arbitrum 等
- **WebSocket**：实时事件订阅

## 使用方式

```javascript
const provider = new ethers.JsonRpcProvider(
  'https://mainnet.infura.io/v3/YOUR_PROJECT_ID'
);

const wsProvider = new ethers.WebSocketProvider(
  'wss://mainnet.infura.io/ws/v3/YOUR_PROJECT_ID'
);
```

## 套餐对比

| 特性 | 免费 | 专业 | 企业 |
|------|------|------|------|
| 请求/天 | 100K | 300K+ | 自定义 |
| IPFS 网关 | ✅ | ✅ | ✅ |
| 团队协作 | ❌ | ✅ | ✅ |
| SLA | ❌ | ✅ | ✅ |
