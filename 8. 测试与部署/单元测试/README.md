# 单元测试 (Unit Testing)

单元测试是智能合约测试金字塔中运行速度最快、覆盖面最广的基石。本模块通过全面的 Solidity 和 TypeScript 真实代码对比，带你掌握现代主流合约单元测试的完整技术。

---

## 核心学习模块

我们在此专题中准备了高保真的实战指南与双框架合约示例：

> 📘 **[单元测试深度解析与 Foundry/Hardhat 双框架实战](./Unit_Testing_Guide.md)**
> 深入探索和掌握以下核心单元测试技术：
> 1. **Foundry 原生 Solidity 测试**：讲解 `setUp` 初始化，剖析 `vm.prank` 越权模拟、`vm.deal` 余额篡改、`vm.warp` 时间欺骗以及 `vm.expectRevert` / `vm.expectEmit` 报错与事件精确匹配。
> 2. **Foundry 单元测试实战**：提供 `NestingStore.sol` 被测合约及其完备的 `NestingStore.t.sol` 测试脚本。
> 3. **Hardhat TypeScript 测试**：讲解如何通过 **`loadFixture`（部署状态快照复用）** 将 Mocha 运行效率提升 10x，提供 TypeScript 的高保真测试。

---

## 框架抉择导读

- **Solidity 密集测试**：首选 **Foundry**，无需异步等待与类型转换，可获得极致的 Rust 底层执行速度和 call trace 调用追踪报错日志。
- **全栈与 SDK 联动测试**：如果你的测试需要高频与外部 TS 脚本（如 DApp 前端 SDK、Ethers.js）发生联动交互，采用 **Hardhat Mocha+Chai** 则是极好的选择。
