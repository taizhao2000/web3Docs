# 事件监控

## 概述

链上事件监控是 DApp 运维的核心能力，用于实时跟踪合约状态变化和交易活动。

## 监控方式

### 1. 事件过滤（ethers.js）

```javascript
const contract = new ethers.Contract(address, abi, provider);

// 监听 Transfer 事件
contract.on('Transfer', (from, to, value, event) => {
  console.log(`Transfer: ${from} -> ${to}, value: ${value}`);
  console.log(`Tx: ${event.transactionHash}`);
});

// 过滤特定条件的事件
const filter = contract.filters.Transfer(fromAddress);
contract.on(filter, (from, to, value, event) => {
  // 只处理特定 from 地址的 Transfer
});
```

### 2. WebSocket 实时监控

```javascript
const wsProvider = new ethers.WebSocketProvider(
  'wss://mainnet.infura.io/ws/v3/YOUR_KEY'
);

// 监听新区块
wsProvider.on('block', (blockNumber) => {
  console.log('New block:', blockNumber);
});

// 监听 Pending 交易
wsProvider.on('pending', (txHash) => {
  console.log('Pending tx:', txHash);
});
```

### 3. The Graph 订阅

```graphql
subscription {
  transfers(orderBy: blockTimestamp, orderDirection: desc, first: 10) {
    id
    from
    to
    value
  }
}
```

## 监控架构

```
链上事件 → WebSocket/Filter → 消息队列 → 处理服务 → 数据库/通知
```

## 工具推荐

| 工具 | 用途 |
|------|------|
| Tenderly | 交易监控和模拟 |
| Forta | 威胁检测机器人 |
| OpenZeppelin Defender | 自动化运维 |
| Alchemy Notify | Webhook 通知 |
