# EVM 极致 Gas 优化与省钱秘籍

> 在以太坊与 EVM 兼容公链中，Gas 费是用户交互最敏感的隐形门槛，也是衡量一个 Web3 项目技术含金量的绝对指标。智能合约底层的每一次 Storage（存储）读写、Memory（内存）开辟和 Bytecode（字节码）设计都对应着真金白银的 Gas 消耗。本指南将深度总结 EVM 核心的 Gas 优化黑科技，帮助团队在部署、调用以及高频循环中压榨每一滴 Gas 费。

---

## 1. EVM 存储层（Storage）极致压榨（最高性价比优化点）

在 EVM 状态机中，**Storage（持久化存储）读写是全链最昂贵的操作**。
- `SSTORE`（新写入未被初始化的 Slot）：需花费高达 **$20,000$ Gas**。
- `SLOAD`（冷读取从未被加载过的 Slot）：需花费 **$2,100$ Gas**。
- 相比之下，读取内存（`MLOAD`）和栈（`Stack`）操作通常只需要 **$3$ Gas**。因此，优化 Storage 是 Gas 优化的绝对核心。

### 1.1 存储槽紧凑打包（Storage Slot Packing）
EVM 每次分配存储空间是以一个 **256 位（32 字节）** 的槽（Slot）为单位的。如果变量的声明在代码行中是**连续的**，编译器会尝试将多个尺寸小于 32 字节的变量打包进同一个 Slot 中。

```
 ❌ 未优化变量排列（占用 3 个 Slot）：
 [ Slot 0 ] --> uint128 a; (16 字节)
 [ Slot 1 ] --> uint256 b; (32 字节) -- 阻断打包！
 [ Slot 2 ] --> uint128 c; (16 字节)

  极致紧凑打包优化（仅占用 2 个 Slot）：
 [ Slot 0 ] --> uint128 a; uint128 c; (16+16 = 32 字节，完美塞满 1 个 Slot)
 [ Slot 1 ] --> uint256 b;            (32 字节)
```

- **Gas 损耗对比**：未优化排列在写入时会触发 3 次 `SSTORE`（共约 $60,000$ Gas）；打包后只需触发 2 次（共约 $40,000$ Gas），直接省下 **$20,000$ Gas**。
- **打包限制**：此打包规则仅对**状态变量（State Variables）**和**结构体（Structs）**生效。在函数内的局部变量打包是没有意义的，因为局部变量在栈中存储。

### 1.2 `constant` 与 `immutable` 代替普通状态变量
如果一个变量的值在编译时或部署后就再也不会改变：
- **`constant`（常量）**：值在编译期写死（适合固定费率、版本号、固定地址）。
- **`immutable`（不可变量）**：值在部署构造函数 `constructor` 中传入写死。
- **Gas 减负原理**：这两者**完全不占用任何 Storage Slot**。编译器会把它们的值直接内联（Inline）到合约的运行时字节码（Runtime Bytecode）中。读取它们只需使用 `PUSH` 指令（花费 **$3$ Gas**），而不需要昂贵的 `SLOAD`（冷读取花费 **$2,100$ Gas**），**直接省去 99.9% 的读取开销**。

### 1.3 局部变量缓存 Storage（Caching Storage in Memory）
如果在函数中、特别是 `for` 循环里，需要多次读取同一个状态变量，必须在循环前先将其读入内存（Memory）的临时变量中。

```solidity
// ❌ 错误示范：在循环中直接高频读取状态变量
uint256 public globalLimit = 100;

function iterate(uint256[] calldata data) external {
    for (uint256 i = 0; i < data.length; i++) {
        // 每次循环都执行一次 SLOAD 读取 globalLimit，如果是冷读取消耗 2100 Gas！
        if (data[i] > globalLimit) { ... }
    }
}

//  正确示范：缓存到内存中
function iterateOptimized(uint256[] calldata data) external {
    uint256 limit = globalLimit; // 仅执行一次 SLOAD 缓存在内存中
    uint256 len = data.length;   // 缓存数组长度，防止每次循环都去拉取 calldata
    for (uint256 i = 0; i < len; ) {
        // 每次读取 limit 只需要 3 Gas (MLOAD)
        if (data[i] > limit) { ... }
        unchecked { ++i; } // 递增优化
    }
}
```

---

## 2. 内存（Memory）与输入数据（Calldata）优化

### 2.1 递增循环中使用 `unchecked { ++i; }`
在 Solidity 0.8.x 中，编译器会默认对所有的算术自增进行安全检查以防止溢出。但在 `for` 循环中，循环计数器 `i` 的上限通常是数组的长度（通常低于上百次），**绝对没有任何可能溢出至 `2**256 - 1` 的边界**。
- **Gas 节约**：在循环尾部，通过 `unchecked { ++i; }` 关闭该处的溢出校验，每次循环可省下 **$100 - 150$ Gas** 的算术冗余开销。
- **为什么使用 `++i` 而不是 `i++`**：`++i` 会在原地直接执行递增并返回结果；而 `i++` 需要在编译器后台暂存一份旧值的临时副本再返回，会多花费约 **$5$ Gas**。

### 2.2 只读参数一律使用 `calldata` 代替 `memory`
当你在外部调用（`external`）函数中需要传入数组、结构体、字符串等引用类型参数时：
- **`memory`**：EVM 会在执行函数时，强制将外部传入的数据**全部拷贝并重组写入当前的内存 MSTORE 中**。复制大数组非常消耗 Gas，且开销随数据长度呈二次方（$O(N^2)$）暴涨。
- **`calldata`**：EVM **不会复制数据**。它直接提供一个只读的指针，直接指向外部调用数据（Transaction Input Data）的原始位置。
- **优化原则**：只要函数内**不需要修改该输入参数**，必须声明为 `calldata`，**彻底省去内存复制的巨额开销**。

---

## 3. 部署层与编译层 Gas 减负（瘦身秘籍）

随着合约业务越来越复杂，以太坊对合约的最大字节码大小有严格限制（EIP-170 规定 **最大不能超过 $24,576$ 字节**）。如果合约写得过大，会直接面临“无法部署”的窘境。

### 3.1 自定义 Error（Custom Error）代替 Require 长字符串
早期我们习惯写 `require(balance >= amount, "Token: Insufficient balance for this transaction");`。
- **瘦身痛点**：每一个这样的错误提示字符串，都需要作为完整的文本存储在合约的 Deploy 字节码中。当发生 Revert 时，还需要将这段长文本转化为 Hex 编码返回，部署 Gas 和调用 Gas 双高。
- **自定义 Error 优势**：`revert InsufficientBalance();` 在底层被哈希并截断为 **$4$ 字节的选择器（Selector）**。它不携带任何文本，极大地瘦身了合约的 Bytecode 体积（可缩减 $10\% - 30\%$ 部署体积），且每次执行出错时可省下 **$100 - 300$ Gas**。

### 3.2 优化器（Optimizer Runs）的黄金权衡
在 `foundry.toml` 或者是 `hardhat.config.ts` 中，优化器（Optimizer）配置项中的 `runs` 控制着编译器的底层内联优化策略。

```toml
# foundry.toml 配置
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
optimizer = true
optimizer_runs = 10000 # 核心设置
```

- **`runs = 200`（低 runs 适合一次性部署，调用低频）**：
  - 编译器专注于**让部署合约的体积最小**。
  - **结果**：部署 Gas 极低；但后续交互执行时调用 Gas 稍高。适合仅部署一次、平时调用极低频的配置类合约。
- **`runs = 10000` / `20000`（高 runs 适合高频交互，如 DeFi 资金池）**：
  - 编译器专注于**让用户后续调用交互时的 Gas 最省**。为此不惜进行极限内联（Inline），会产生大体积的部署字节码。
  - **结果**：部署 Gas 极高；但后续用户进行充值、提款、交易时的**执行 Gas 降到冰点**。对于 DeFi 核心资金池项目，必须开满最大 runs 进行编译，把 Gas 福利留给用户。

---

## 4. Solidity 极致 Gas 优化检查清单（Cheatsheet）

| 优化项维度 | 优化操作要点 | 预计能省的 Gas |
| :--- | :--- | :--- |
| **存储层** | 连续声明小于 32 字节的变量进行 Storage Slot 紧凑打包 | $20,000$ Gas / 新 Slot 减少 |
| **存储层** | 只读不改的状态变量加装 `constant` 或 `immutable` 编译内联 | $2,000+$ Gas / 每次读取 |
| **存储层** | 循环内高频读取的状态变量，在循环前先读入 Memory 临时缓存 | $2,000 \times N$ Gas / 循环 |
| **算术层** | `for` 循环递增使用 `unchecked { ++i; }` | $100 - 150$ Gas / 每次迭代 |
| **算术层** | 用 `++i` 代替 `i++` 执行原地直接递增 | $5$ Gas / 每次递增 |
| **内存层** | 只读引用参数使用 `calldata` 代替 `memory` 避免内存拷贝 | 随数组大小呈 $O(N^2)$ 指数级省 Gas |
| **内存层** | 在 `for` 循环条件判断前，先缓存数组长度 `uint256 len = arr.length;` | $2,100$ (冷槽) / 每次迭代 |
| **编译层** | 自定义 Error `revert MyError()` 代替 `require` 长提示文本 | 缩减 $20\%$ 部署体积 & $200$ 调用 Gas |
| **编译层** | 频繁交互的资金池开启高 `optimizer_runs`（如 10,000+） | $50 - 500$ Gas / 每次高频函数调用 |
