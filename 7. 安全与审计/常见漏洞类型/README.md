# 常见漏洞类型

## 1. 重入攻击（Reentrancy）

最经典的智能合约漏洞，The DAO 攻击就是利用此漏洞。

### 漏洞代码

```solidity
// ❌ 易受重入攻击
function withdraw() external {
    uint256 balance = balances[msg.sender];
    (bool success, ) = msg.sender.call{value: balance}(""); // 先转账
    require(success);
    balances[msg.sender] = 0; // 后更新状态
}
```

### 修复方案

```solidity
// ✅ 检查-生效-交互模式
function withdraw() external {
    uint256 balance = balances[msg.sender];
    balances[msg.sender] = 0; // 先更新状态
    (bool success, ) = msg.sender.call{value: balance}(""); // 后转账
    require(success);
}

// ✅ 使用 ReentrancyGuard
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Safe is ReentrancyGuard {
    function withdraw() external nonReentrant {
        // ...
    }
}
```

## 2. 整数溢出/下溢

Solidity 0.8.0 之前版本需要使用 SafeMath。

## 3. 访问控制缺陷

缺少 onlyOwner 等修饰符，导致任何人都可以调用特权函数。

## 4. 前端运行（Front-running）

攻击者通过 Gas 价格操纵交易顺序。

## 5. 预言机操纵

依赖单一 DEX 价格作为预言机，可被闪电贷攻击操纵。

## 漏洞分类

| 类型 | 严重性 | 示例 |
|------|--------|------|
| 重入攻击 | 🔴 严重 | The DAO |
| 整数溢出 | 🔴 严重 | BEC Token |
| 访问控制 | 🔴 严重 | Parity Wallet |
| 预言机操纵 | 🟡 高 | bZx 攻击 |
| 前端运行 | 🟡 中 | Uniswap 交易 |
| 拒绝服务 | 🟠 中 | GovernMental |
