# 借贷协议与清算机制

去中心化借贷协议是 DeFi 生态中的信用枢纽。本模块带你深入去中心化借贷的核心记账逻辑、利率模型以及应对黑客攻击的安全要点。

---

## 核心学习模块

在这个专题中，我们编写了极具技术含金量的深度指南：

> 📘 **[借贷协议与清算机制深度指南](./Lending_and_Liquidation_Guide.md)**
> 本指南详细涵盖了以下核心板块：
> 1. **超额抵押与健康因子（Health Factor）模型**：详细解构抵押因子（LTV）、清算阈值的概念，并给出借贷池评估账户安全性的健康因子 $H_f$ 严格计算公式。
> 2. **Kink 拐点双率利率模型**：探讨如何通过资金利用率 $U$ 来动态调整借贷利率，防止挤兑，并提供一个生产级的 `JumpRateModel` Solidity 计算合约。
> 3. **Compound cToken 汇率与计息机制**：讲解 cToken 作为存款凭证如何通过 $e = \frac{\text{Cash} + \text{Borrows} - \text{Reserves}}{\text{Total Supply}}$ 实现本息复利增值，以及 Accrue Interest 的链上高频累加。
> 4. **清算机器人套利实战**：解析 Close Factor（最大单次清算比例）以及清算惩罚带来的套利数学，提供一个能够直接由链下触发、用于套利的 SimpleLiquidationBot 合约。

---

## 常见名词速览

- **Over-collateralization**：超额抵押，链上缺乏信用体系时的唯一防违约手段。
- **Utilization Rate ($U$)**：资金利用率，反映池子借贷紧张度，通常维持在 $80\%$ 上下为最佳状态。
- **Health Factor ($H_f$)**：小于 1 时即触发可清算状态，此时任何外部套利机器人均可时代扣债务并折扣价卷走资产。
