# EVM 极致 Gas 优化

在以太坊与 EVM 中，Gas 费直接决定了用户的交互摩擦成本。高昂的交互成本会把大多数普通散户挡在门外。本模块系统整理了在开发中压榨每一滴 Gas 的黄金秘籍。

---

## 核心学习模块

> 📘 **[EVM 极致 Gas 优化与省钱秘籍](./Gas_Optimization_Guide.md)**
> 涵盖了以下核心高性价比优化点：
> 1. **EVM 存储（Storage）极致优化**：Storage Slot 打包排列（Storage Packing）原理，`constant` / `immutable` 编译内联指令，以及循环内将 Storage 局部变量缓存至内存的操作。
> 2. **内存（Memory）与输入数据（Calldata）自控**：在 for 循环中使用 `unchecked { ++i; }` 递增、`++i` 代替 `i++`，以及缓存数组长度、只读引用参数一律声明为 `calldata`。
> 3. **部署层与编译层减负**：自定义 Error `revert MyError()` 代替 `require` 长报错字符串，以及 EVM 优化器（`optimizer_runs`）在部署成本与交互成本之间的黄金权衡。

---

## 极致省钱秘籍卡

- **写 Storage packing 肌肉记忆**：在状态变量和 struct 定义时，必须让小于 32 字节（如 uint128, uint64, bool）的变量连续定义。
- **自定义 Error 标配化**：在 Solidity 0.8.x 阶段，彻底告别 require 错误字符串，全面转型自定义 Error，既能压缩字节码，更能降 interact 交互 Gas。
