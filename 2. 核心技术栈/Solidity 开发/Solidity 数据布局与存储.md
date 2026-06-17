# Solidity 数据布局与 EVM 存储

> 理解 EVM 的存储模型是写出 Gas 高效、存储安全的 Solidity 代码的前提。本文从"字节码层面"解释你写的每一行 Solidity 在链上到底发生了什么。

---

## 1. EVM 存储架构总览

EVM 有三类持久化程度不同的存储区域：

| 区域 | 位置 | 持久性 | 成本 | 大小 |
|------|------|--------|------|------|
| **Storage** | 区块链状态树 | 永久 | 极高（SSTORE: 20000 gas/新slot） | 2^256 个 32 字节 slot |
| **Memory** | 内存（RAM） | 单次调用 | 低（MSTORE: 3 gas） | 无限（受 Gas 限制） |
| **Calldata** | 调用输入 | 只读 | 最低 | 跟随交易大小 |

还有一个隐含的**Stack**（栈），最大深度 1024，用于 EVM 操作数的临时存储。

```
交易调用
    │
    ▼
┌──────────────────────────┐
│  Calldata（只读输入）      │
├──────────────────────────┤
│  Stack（操作数栈，≤1024）  │
├──────────────────────────┤
│  Memory（临时内存）        │
├──────────────────────────┤
│  Storage（永久状态）       │
└──────────────────────────┘
```

---

## 2. Storage 槽位（Slot）机制

### 2.1 基本规则

- Storage 是一个 `key → value` 的映射，key 和 value 都是 32 字节（256 位）
- 合约的状态变量按声明顺序从 slot 0 开始排列
- 每个变量占几个 slot 取决于其类型

```solidity
contract StorageLayout {
    uint256 public a;    // slot 0（32 字节 = 256 位，刚好一个 slot）
    uint256 public b;    // slot 1
    address public c;    // slot 2（address 只有 20 字节，但仍占一个 slot）
    bool public d;       // slot 3（bool 只有 1 字节，但仍占一个 slot）
    uint256 public e;    // slot 4
}
```

### 2.2 存储打包（Storage Packing）

小于 32 字节的相邻变量可以打包到同一个 slot：

```solidity
contract PackedStorage {
    // ❌ 浪费 Gas 的写法：每个变量独占一个 slot
    uint256 public a;     // slot 0
    uint8 public b;       // slot 1（浪费 31 字节）
    uint256 public c;     // slot 2
    uint8 public d;       // slot 3（浪费 31 字节）
    // 共 4 个 slot

    // ✅ 省 Gas 的写法：小类型打包
    uint256 public x;     // slot 4（必须独占）
    uint8 public y;        // slot 5 的第 1 字节
    uint8 public z;        // slot 5 的第 2 字节
    uint256 public w;     // slot 6（新 slot，因为前面已满或类型不兼容）
    // 共 3 个 slot
}
```

**打包规则**：
1. 从 slot 的低位开始填充
2. 如果当前变量能放进当前 slot 的剩余空间，就打包
3. 如果放不下，则从下一个 slot 开始
4. `uint256` 总是开启新 slot

> **Gas 影响**：SSTORE 一次写入一个 slot 花费 20000 gas（从零到非零），所以打包 2 个变量到 1 个 slot 可以省一次 SSTORE。但读取打包变量需要额外的位移操作（AND + SHR），在某些情况下可能反而更贵。

### 2.3 结构体的存储布局

```solidity
struct UserInfo {
    address wallet;    // 20 字节
    uint8 role;        // 1 字节  ┐
    bool isActive;     // 1 字节  ├─ 打包到同一个 slot
    uint8 level;       // 1 字节  ┘
    uint256 balance;   // 32 字节 → 必须新 slot
}

// 存储：
// slot N:   [wallet(20B) | role(1B) | isActive(1B) | level(1B) | 9B空白]
// slot N+1: [balance(32B)]
```

> **关键**：结构体中的 `uint256` 会触发新 slot，即使前面还有空间。所以结构体中应把大类型放后面、小类型放前面并紧凑排列。

---

## 3. Mapping 与动态数组的存储

Mapping 和动态数组不直接存储在 slot 中——slot 中存的是"占位符"，实际数据存储在由 `keccak256` 计算出的位置。

### 3.1 Mapping 的存储

```solidity
contract MappingStorage {
    mapping(address => uint256) public balances;     // slot 0
    mapping(uint256 => bool) public exists;          // slot 1
    mapping(address => mapping(uint256 => bool)) public doubleMap;  // slot 2
}
```

**单个 Mapping**：`balances[key]` 的存储位置 = `keccak256(abi.encode(key, slotNumber))`

```
balances[0xAAA...] → keccak256(abi.encode(0xAAA..., 0))
balances[0xBBB...] → keccak256(abi.encode(0xBBB..., 0))
```

**嵌套 Mapping**：逐层计算

```
doubleMap[0xAAA...][42]
→ 先算第一层：keccak256(abi.encode(0xAAA..., 2))  → 得到中间位置 P
→ 再算第二层：keccak256(abi.encode(42, P))
```

> **为什么 Mapping 不能遍历？** 因为存储位置是通过哈希函数计算的，不存在"所有 key"的列表。要从 key 找到 value 需要知道 key，没有逆向查找的可能。

### 3.2 动态数组的存储

```solidity
contract ArrayStorage {
    uint256[] public items;  // slot 0
}
```

- `slot 0` 存储的是数组**长度**（不是数据！）
- `items[i]` 的存储位置 = `keccak256(abi.encode(slotNumber)) + i`

```
items 的长度存在 slot 0
items[0] → keccak256(abi.encode(0)) + 0
items[1] → keccak256(abi.encode(0)) + 1
items[2] → keccak256(abi.encode(0)) + 2
...
```

> **数组的存储是连续的**，这意味着如果数组很大，它会在存储中占据一大段连续区域。而 Mapping 的每个元素位置是随机的（哈希分布）。

### 3.3 内联汇编验证

```solidity
contract StorageProbe {
    uint256 public value = 42;              // slot 0
    mapping(address => uint256) public balances; // slot 1
    uint256[] public items;                  // slot 2

    constructor() {
        items.push(100);
        items.push(200);
        balances[msg.sender] = 999;
    }

    // 用汇编读取任意 slot
    function readSlot(uint256 slot) public view returns (bytes32) {
        bytes32 result;
        assembly {
            result := sload(slot)
        }
        return result;
    }

    // 读取 balances[key]
    function readMapping(address key) public view returns (uint256) {
        uint256 slot = 1;
        bytes32 location = keccak256(abi.encode(key, slot));
        bytes32 result;
        assembly {
            result := sload(location)
        }
        return uint256(result);
    }

    // 读取 items[index]
    function readArray(uint256 index) public view returns (uint256) {
        uint256 slot = 2;
        bytes32 base = keccak256(abi.encode(slot));
        bytes32 result;
        assembly {
            result := sload(add(base, index))
        }
        return uint256(result);
    }
}
```

---

## 4. 三种数据位置的完整对比

### 4.1 规则表

| 变量类型 | 默认位置 | 可指定位置 | 说明 |
|---------|---------|-----------|------|
| 状态变量 | storage | ❌ 不可指定 | 永久存储 |
| 函数参数 | calldata (external) / memory (其他) | external 可显式指定 calldata | - |
| 局部变量（值类型） | memory | ❌ 不可指定 | 自动在栈上 |
| 局部变量（引用类型） | memory | ✅ 可显式 storage/memory | 不指定则 memory |
| 返回值 | memory | ✅ 可指定 | - |

### 4.2 赋值行为

```solidity
contract DataAssignment {
    uint256[] public storageArray;   // storage

    function demo() external {
        // 1. storage → memory：拷贝（创建独立副本）
        uint256[] memory memCopy = storageArray;
        memCopy[0] = 999;        // 不影响 storageArray

        // 2. memory → storage：拷贝（覆盖 storage）
        // storageArray = memCopy;  // 需要逐个赋值或用特殊方式

        // 3. storage → storage：引用（指向同一位置！）
        uint256[] storage ref = storageArray;
        ref[0] = 999;            // ✅ 修改了 storageArray[0]！
        ref.push(123);           // ✅ 给 storageArray 追加元素

        // 4. memory → memory：拷贝
        uint256[] memory memA = new uint256[](3);
        uint256[] memory memB = memA;   // 拷贝
        memB[0] = 999;           // 不影响 memA
    }
}
```

> **最危险的坑**：`storage` 引用。如果你不小心将局部变量声明为 storage 引用，修改它就是修改链上状态。反之，如果你需要修改链上状态但声明成了 memory，修改不会生效。

### 4.3 calldata 的特殊性

```solidity
function process(uint256[] calldata _data) external {
    // calldata 是只读的：
    // _data[0] = 99;  ❌ 编译错误

    // 如果需要修改，必须拷贝到 memory
    uint256[] memory mutableCopy = new uint256[](_data.length);
    for (uint256 i = 0; i < _data.length; i++) {
        mutableCopy[i] = _data[i] * 2;
    }
}
```

> **何时用 calldata**：external 函数中，如果参数不需要修改，一律用 `calldata`。它省去了从 calldata 拷贝到 memory 的开销，对大数组尤其明显。

---

## 5. Gas 优化与存储

### 5.1 存储操作的 Gas 成本

| 操作 | Gas | 说明 |
|------|-----|------|
| SLOAD（读已存在的 slot） | 2,100 | 从零读更贵：2,100 |
| SSTORE（写新值到零 slot） | 20,000 | 首次写入 |
| SSTORE（修改非零 → 非零） | 2,900 | 更新已有值 |
| SSTORE（清零，非零 → 零） | 2,900 + 4,800 退还 | 删除值有 Gas 退还 |
| SSTORE（同一交易内先写再清） | 2,900 + 4,800 退还 | 临时存储很便宜 |

### 5.2 实用优化技巧

**1. 用 constant/immutable 替代 storage**

```solidity
// ❌ 每次 SLOAD 花费 2100 gas
uint256 public maxSupply = 10000;

// ✅ 编译时内联，0 gas 读取
uint256 public constant MAX_SUPPLY = 10000;

// ✅ 部署时确定，极低 gas 读取
uint256 public immutable createdAt;
```

**2. 缓存 storage 到 memory**

```solidity
// ❌ 每次 arr.length 都是一次 SLOAD
for (uint256 i = 0; i < arr.length; i++) { ... }

// ✅ 缓存到 memory（省 ~2100 gas × 循环次数）
uint256 len = arr.length;
for (uint256 i = 0; i < len; i++) { ... }
```

**3. 批量操作减少 SSTORE**

```solidity
// ❌ 多次 SSTORE
function setValues(uint256 a, uint256 b, uint256 c) external {
    valueA = a;  // SSTORE × 3
    valueB = b;
    valueC = c;
}

// ✅ 打包到单个 slot（如果类型允许）
// uint128 a, uint128 b => 一个 slot
function setPacked(uint128 a, uint128 b) external {
    packed = (uint256(a) << 128) | uint256(b);  // SSTORE × 1
}
```

**4. 用 mapping 替代数组**

```solidity
// ❌ 数组查找 O(n)，受 Gas 限制
uint256[] public items;
function contains(uint256 target) external view returns (bool) {
    for (uint256 i = 0; i < items.length; i++) {
        if (items[i] == target) return true;  // O(n) 遍历
    }
    return false;
}

// ✅ Mapping 查找 O(1)，Gas 恒定
mapping(uint256 => bool) public itemExists;
function contains(uint256 target) external view returns (bool) {
    return itemExists[target];  // O(1) 一次 SLOAD
}
```

**5. 短路求值优化条件顺序**

```solidity
// ❌ 昂贵检查在前
require(complexCalculation(a) > 0 && a > 0);

// ✅ 便宜检查在前（短路后不执行昂贵计算）
require(a > 0 && complexCalculation(a) > 0);
```

### 5.3 unchecked 的适用场景

```solidity
// ✅ 循环变量不会溢出（i 最大 = arr.length < 2^256）
unchecked {
    for (uint256 i = 0; i < arr.length; i++) {
        // 省掉每次 i++ 的溢出检查
    }
}

// ✅ 已确认不会溢出的算术
function afterSub() external {
    require(a >= b);
    unchecked { a -= b; }  // require 已保证不溢出
}
```

---

## 6. 字节码层面的理解

### 6.1 合约编译流程

```
Solidity 源码 (.sol)
      │
      ▼  solc 编译器
┌─────────────┐
│ ABI         │ ← 函数签名与参数编码的描述（给外部调用者看）
│ Bytecode    │ ← 部署时提交到链上的机器码
│ Opcode      │ ← Bytecode 的人工可读形式
└─────────────┘
```

### 6.2 部署字节码 vs 运行时字节码

```
部署字节码 = 初始化代码 + 运行时字节码

初始化代码：执行 constructor，返回运行时字节码的地址
运行时字节码：合约的实际代码，存储在链上
```

```solidity
// 这段代码的 constructor 只在部署时运行
contract Example {
    address public immutable owner;

    constructor() {
        owner = msg.sender;  // 这在初始化代码中执行
    }

    function foo() public pure returns (uint256) {
        return 42;  // 这在运行时字节码中
    }
}
```

> **immutable 变量的秘密**：immutable 变量在 constructor 中赋值后，编译器会在运行时字节码中所有引用处替换为实际值。所以它不占 storage slot，但字节码会稍大。

### 6.3 常见操作码

| 操作码 | Gas | 作用 |
|--------|-----|------|
| `PUSH1 0x00` | 3 | 将 0 压入栈 |
| `SLOAD` | 2,100 | 从 storage 读取 32 字节 |
| `SSTORE` | 2,900~20,000 | 向 storage 写入 32 字节 |
| `MLOAD` | 3 | 从 memory 读取 32 字节 |
| `MSTORE` | 3 | 向 memory 写入 32 字节 |
| `CALL` | 700+ | 调用其他合约 |
| `DELEGATECALL` | 700+ | 委托调用（代理模式核心） |
| `CALLDATALOAD` | 3 | 从 calldata 读取 |
| `JUMP` | 8 | 跳转 |
| `RETURN` | 0 | 返回 |
| `REVERT` | 0 | 回滚 |

### 6.4 用 `objdump` 查看字节码

```bash
# 安装 solc
solc --bin --opcodes Example.sol

# 输出示例：
# PUSH1 0x80    ← 初始化内存
# PUSH1 0x40
# MSTORE
# CALLVALUE     ← 检查是否发送了 ETH
# DUP1
# ISZERO
# PUSH1 0xXX
# JUMPI
# ...
```

---

## 7. 跨合约调用的存储隔离

### 7.1 核心概念

每个合约有自己的 storage 空间（从 slot 0 开始）。合约 A 的 slot 0 和合约 B 的 slot 0 是完全不同的。

```
合约 A 的 Storage                合约 B 的 Storage
┌──────────┐                    ┌──────────┐
│ slot 0   │ ← A.balance       │ slot 0   │ ← B.owner
│ slot 1   │ ← A.owner         │ slot 1   │ ← B.count
│ slot 2   │ ← A.active        │ slot 2   │ ← B.data
└──────────┘                    └──────────┘
```

### 7.2 delegatecall 的存储陷阱

`delegatecall` 在调用者（代理）的 storage 中执行被调用者（逻辑）的代码——**按 slot 位置匹配，不按变量名匹配**：

```solidity
// 逻辑合约 V1
contract LogicV1 {
    uint256 public value;  // slot 0
    address public owner;  // slot 1
}

// 逻辑合约 V2（错误升级！）
contract LogicV2 {
    address public owner;  // slot 0 ← 读取的是代理合约 slot 0 的 value！
    uint256 public value;  // slot 1 ← 读取的是代理合约 slot 1 的 owner！
}

// 代理合约
contract Proxy {
    uint256 public value;       // slot 0（与逻辑合约 slot 0 对应）
    address public owner;       // slot 1（与逻辑合约 slot 1 对应）
    address public implementation;

    fallback() external payable {
        (bool s, bytes memory d) = implementation.delegatecall(msg.data);
        require(s);
        assembly { return(add(d, 32), mload(d)) }
    }
}
```

> **这就是为什么代理模式和存储布局铁律如此重要**：delegatecall 不看变量名，只看 slot 位置。乱改顺序 = 数据错位。

---

## 8. 存储布局检查工具

| 工具 | 用途 | 命令 |
|------|------|------|
| `forge inspect` | 查看合约存储布局 | `forge inspect Contract storage-layout` |
| `hardhat-storage-layout` | Hardhat 插件，自动生成布局 | `npx hardhat storage-layout` |
| `sol2uml` | 可视化 UML 图 | `sol2uml storage Contract` |
| OpenZeppelin Upgrades Plugin | 升级时自动检查布局兼容性 | `npx hardhat verify-storage-layout` |

```bash
# Foundry 示例输出
$ forge inspect SimpleToken storage-layout
| Name        | Type            | Slot | Offset | Bytes |
|-------------|-----------------|------|--------|-------|
| name        | string          | 0    | 0      | 32    |
| symbol      | string          | 1    | 0      | 32    |
| decimals    | uint8           | 2    | 0      | 1     |
| totalSupply | uint256         | 3    | 0      | 32    |
| balanceOf   | mapping(...)    | 4    | 0      | 32    |
| allowance   | mapping(...)    | 5    | 0      | 32    |
| owner       | address         | 6    | 0      | 20    |
```

---

## 快速参考卡

| 概念 | 关键记忆点 |
|------|-----------|
| Storage Slot | 2^256 个 32 字节槽位，从 0 顺序分配 |
| 存储打包 | 相邻小类型共享 slot，省 SSTORE Gas |
| Mapping 存储 | `keccak256(key, slot)` → 值位置，不可遍历 |
| 动态数组存储 | `slot` 存长度，`keccak256(slot) + i` 存元素 |
| calldata vs memory | calldata 只读省 Gas，memory 可修改 |
| storage 引用 | `Type storage ref = var` → 修改 ref = 修改链上 |
| SSTORE 成本 | 新写 20,000 / 更新 2,900 / 清零有退还 |
| delegatecall | 按.slot 位置匹配，不看变量名 |
| constant/immutable | 不占 slot，0 或极低 Gas 读取 |
| 升级铁律 | 只追加变量，不改顺序不改类型 |
