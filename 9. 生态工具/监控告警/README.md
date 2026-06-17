# 监控告警

## 概述

Web3 应用的监控告警体系，确保及时发现和响应异常情况。

## 监控维度

| 维度 | 监控项 | 告警条件 |
|------|--------|----------|
| 合约状态 | TVL 变化 | 大额异常流出 |
| 交易活动 | 交易量/频率 | 突增或骤降 |
| Gas | Gas 价格 | 超过阈值 |
| 节点 | 同步状态 | 落后超过 N 个区块 |
| 权限 | Owner 操作 | 任何权限变更 |
| 安全 | 异常调用 | 未知合约交互 |

## 告警通道

| 通道 | 适用场景 |
|------|----------|
| Slack/Discord Webhook | 团队即时通知 |
| Email | 重要事件 |
| Telegram Bot | 移动端推送 |
| PagerDuty | 7×24 值班 |
| 自定义 Webhook | 集成到内部系统 |

## 实现示例

```javascript
// 简单的监控服务
const monitor = async () => {
  const contract = new ethers.Contract(address, abi, provider);
  
  // 监控大额转账
  contract.on('Transfer', async (from, to, value) => {
    const amount = ethers.formatEther(value);
    if (parseFloat(amount) > THRESHOLD) {
      await sendAlert(`大额转账: ${amount} tokens from ${from} to ${to}`);
    }
  });
  
  // 监控 Owner 变更
  contract.on('OwnershipTransferred', async (previousOwner, newOwner) => {
    await sendCriticalAlert(`Owner 变更: ${previousOwner} -> ${newOwner}`);
  });
};
```

## 运维工具

| 工具 | 类型 | 说明 |
|------|------|------|
| Grafana + Prometheus | 指标监控 | 经典监控栈 |
| Tenderly | 链上监控 | 交易模拟和告警 |
| Forta | 安全监控 | 去中心化检测网络 |
| Sentinel | 自建监控 | OpenZeppelin 出品 |
