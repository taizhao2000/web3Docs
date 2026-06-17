# MetaMask 集成

## 概述

MetaMask 是最流行的浏览器钱包扩展，DApp 通过 MetaMask 与用户交互。

## 连接流程

```javascript
// 检测 MetaMask
if (typeof window.ethereum !== 'undefined') {
  // 请求用户授权
  const accounts = await window.ethereum.request({
    method: 'eth_requestAccounts'
  });
  console.log('已连接账户:', accounts[0]);
}

// 监听账户变更
window.ethereum.on('accountsChanged', (accounts) => {
  console.log('账户切换:', accounts[0]);
});

// 监听链变更
window.ethereum.on('chainChanged', (chainId) => {
  console.log('链切换:', chainId);
  window.location.reload(); // 推荐刷新页面
});
```

## 最佳实践

- 使用 EIP-6963 发现钱包提供者（替代 `window.ethereum`）
- 使用 EIP-1193 标准事件监听
- 处理网络切换和添加自定义链
- 优雅降级：未安装 MetaMask 时提供引导
