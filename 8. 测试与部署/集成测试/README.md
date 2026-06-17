# 集成测试

## 概述

集成测试验证多个合约之间的交互是否正确，模拟真实的链上场景。

## 测试场景

| 场景 | 说明 |
|------|------|
| 合约间调用 | A 合约调用 B 合约 |
| 代理模式 | 代理合约与逻辑合约交互 |
| 跨协议交互 | 与 Uniswap/Aave 等协议交互 |
| 升级流程 | 合约升级后状态一致性 |
| 多用户并发 | 模拟多用户操作 |

## Fork 测试

使用主网 Fork 进行真实环境测试：

```javascript
// hardhat.config.js
module.exports = {
  networks: {
    hardhat: {
      forking: {
        url: 'https://mainnet.infura.io/v3/YOUR_KEY',
        blockNumber: 17000000, // 固定区块号确保可重复
      },
    },
  },
};
```

```javascript
// 与真实协议交互
it('应通过 Uniswap 交换代币', async function () {
  const uniswapRouter = await ethers.getContractAt('IUniswapV2Router02', ROUTER_ADDRESS);
  await token.approve(ROUTER_ADDRESS, amount);
  await uniswapRouter.swapExactTokensForTokens(...);
});
```

## Foundry Fork 测试

```solidity
// foundry.toml
[profile.default]
fork_block_number = 17000000

// 测试文件
function testForkInteraction() public {
    // 使用真实链上合约
    IUniswapV2Router02 router = IUniswapV2Router02(UNISWAP_ROUTER);
    // ...
}
```
