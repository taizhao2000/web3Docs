# 常见漏洞类型与黑客攻防

在 Solidity 开发和审计中，一些经典的编程逻辑偏差（Bug）常常会被黑客放大为毁灭协议的漏洞。本模块通过“脆弱、攻击、修复”的极高实战代码标准，带你深度拆解漏洞的发生机制。

---

## 核心学习模块

> 📘 **[Solidity 常见漏洞类型深度剖析指南](./Common_Vulnerabilities_Guide.md)**
> 详细涵盖了以下核心板块：
> 1. **经典与跨合约只读重入（Reentrancy）**：解析从单合约转账回调重入，到最新 Curve 式“不修改数据、仅只读重入操纵第三方价格”的高阶漏洞。
> 2. **预言机操纵漏洞（Oracle Manipulation）**：探讨依靠瞬时 DEX 池价格导致的闪电贷巨款砸盘清空国库的漏洞原理。
> 3. **`tx.origin` 管理员越权钓鱼**：演示中间钓鱼合约的回调接收如何秒过 `tx.origin == owner` 的假检验。
> 4. **ECDSA 签名重放（Signature Replay）**：说明无 Nonce、无 ChainId 及无合约地址绑定时的签名二次使用漏洞及高安全修复方案。
> 5. **算术溢出与 `unchecked` 滥用**：详解 Solidity 0.8.x 算术默认检查机制，以及在 `unchecked` 块内混入用户输入引起算术溢出爆发的灾难。

---

## 漏洞防卫卡片

- **防重入首选**：先改账本，后调外部（Checks-Effects-Interactions）。
- **防预言机操纵首选**：丢弃 DEX 瞬时现货价，强制采用 Chainlink 或者是 Uniswap V3 TWAP 时间加权平均价。
- **防钓鱼首选**：彻底封杀 `tx.origin`，统一采用 `msg.sender` 锁定直接调用源。
