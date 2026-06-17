# Solidity 语法基础

> 面向有编程基础、初次接触 Solidity 的开发者。本文不解释"什么是变量"，而是讲清楚 Solidity 与其他语言的关键差异。

---

## 1. 源文件结构

每个 `.sol` 文件至少包含两个部分：

```solidity
// SPDX-License-Identifier: MIT        ← 必须在第一行，声明开源协议
pragma solidity ^0.8.19;               ← 编译器版本声明

import "./OtherContract.sol";           ← 导入其他文件

contract MyContract {                   ← 合约定义
    // ...
}
```

### 版本声明规则

| 语法 | 含义 |
|------|------|
| `pragma solidity ^0.8.19;` | 兼容 0.8.19 到 <0.9.0 |
| `pragma solidity >=0.8.0 <0.9.0;` | 明确范围 |
| `pragma solidity 0.8.19;` | 精确锁定一个版本 |

> **为什么版本声明很重要？** Solidity 还在快速迭代，0.8.x 引入了内置溢出检查（之前需要用 SafeMath），0.8.4+ 引入了自定义错误。不同版本的代码可能无法互相编译。

### import 的几种写法

```solidity
import "./A.sol";                          // 导入 A.sol 中所有符号
import {Token} from "./Token.sol";          // 只导入 Token
import {Token as TK} from "./Token.sol";    // 重命名导入
import * as Tokens from "./Token.sol";      // 命名空间导入
```

---

## 2. 值类型（Value Types）

值类型赋值时**拷贝值**，这是理解 Gas 消耗的基础。

### 2.1 布尔型

```solidity
bool public isActive = true;
bool public hasPermission = false;
```

### 2.2 整型

```solidity
uint256 public amount = 100;      // 无符号，0 到 2^256-1
int256  public balance = -50;     // 有符号，-2^255 到 2^255-1

// 速记：uint 等价于 uint256，int 等价于 int256
uint public defaultUint;          // uint256

// 也可以用 uint8, uint16, ..., uint248（步长为8）
// 但注意：使用更小的类型不一定省 Gas（EVM 按 32 字节操作）
uint8 public smallNumber = 255;   // 最大 255
```

> **0.8.x 溢出保护**：整型运算溢出会自动 revert，不需要 SafeMath。

```solidity
uint8 public overflow = 255;
// overflow + 1 → revert!（0.8.x 自动检测）
// 0.7.x 则会静默溢出回 0，这是历史上无数漏洞的根源
```

### 2.3 地址型

```solidity
address public owner = 0x1234...;           // 20 字节地址
address payable public recipient = payable(0x1234...);  // 可接收 ETH

// 关键区别：只有 payable 地址可以调用 .transfer() 和 .send()
recipient.transfer(1 ether);     // ✅ 发送 1 ETH
// owner.transfer(1 ether);      // ❌ 编译错误：非 payable 地址
```

| 方法 | Gas | 失败行为 | 推荐度 |
|------|-----|---------|--------|
| `.transfer(amt)` | 2300（固定） | revert | ⚠️ 不推荐（gas 不够回调） |
| `.send(amt)` | 2300（固定） | 返回 false | ❌ 不推荐（静默失败） |
| `.call{value: amt}("")` | 转发所有剩余 gas | 返回 (bool, bytes) | ✅ 推荐 |

```solidity
// ✅ 推荐的 ETH 转账方式
(bool success, ) = recipient.call{value: 1 ether}("");
require(success, "Transfer failed");
```

### 2.4 定长字节数组

```solidity
bytes1 public b1 = 0x12;        // 1 字节
bytes32 public b32 = 0x1234...;  // 32 字节（常用：keccak256 哈希的输出）

// bytes1 ~ bytes32，步长为1
// 注意：bytes32 ≠ string！bytes32 是右填充的固定长度
```

### 2.5 枚举

```solidity
enum OrderStatus { Created, Paid, Shipped, Delivered, Cancelled }

OrderStatus public status = OrderStatus.Created;

// 枚举底层是 uint8，可以从/向 uint 转换
uint8 public statusValue = uint8(status);  // 0
```

---

## 3. 引用类型（Reference Types）

引用类型赋值时**拷贝引用**（除非显式拷贝），这是 Solidity 最容易出 bug 的地方。

### 3.1 字符串

```solidity
string public name = "Hello Solidity";

// ⚠️ 字符串在 Solidity 中非常受限：
// - 不支持索引访问：name[0] ❌
// - 不支持长度：name.length ❌
// - 不支持拼接：name + "!" ❌
// - 比较需用 keccak256：keccak256(bytes(name)) == keccak256(bytes("test"))

// 需要操作字符串时，转换为 bytes
bytes memory nameBytes = bytes(name);
uint256 len = nameBytes.length;  // ✅ 获取长度
```

### 3.2 数组

```solidity
// 固定长度数组
uint256[5] public fixedArray = [1, 2, 3, 4, 5];

// 动态数组
uint256[] public dynamicArray;

function arrayOps() public {
    dynamicArray.push(10);           // 追加元素
    dynamicArray.push(20);
    uint256 last = dynamicArray[dynamicArray.length - 1];  // 访问
    dynamicArray.pop();               // 删除末尾元素

    // ⚠️ 删除中间元素：delete 只是置零，不改变长度！
    delete dynamicArray[0];           // dynamicArray[0] = 0，长度不变
}

// 内存中的数组
function memoryArray() public pure returns (uint256[] memory) {
    uint256[] memory arr = new uint256[](3);  // 必须指定长度
    arr[0] = 1;
    arr[1] = 2;
    arr[2] = 3;
    return arr;
}
```

### 3.3 映射（Mapping）

映射是 Solidity 最独特的数据结构，理解它对后续学习存储布局至关重要。

```solidity
// 声明
mapping(address => uint256) public balances;
mapping(address => mapping(uint256 => bool)) public approvals;  // 嵌套映射

function mappingOps() public {
    // 写入
    balances[msg.sender] = 100;

    // 读取（不存在的 key 返回默认值 0）
    uint256 bal = balances[address(0)];  // 0，不是 error

    // ⚠️ 映射的关键限制：
    // 1. 无法遍历所有 key
    // 2. 无法获取 key 的数量
    // 3. 无法删除一个 key（只能置为默认值）
    delete balances[msg.sender];  // balances[msg.sender] = 0
}
```

> **映射为什么不能遍历？** 映射在存储中只存储 value，不存储 key。key 通过 `keccak256(key)` 计算存储位置，所以不存在"所有 key 的列表"。如果需要遍历，需要额外维护一个数组。

### 3.4 结构体

```solidity
struct User {
    address wallet;
    uint256 balance;
    bool isActive;
    string name;
}

// 声明方式
mapping(address => User) public users;
User[] public userList;

function createUser(string memory _name) public {
    // 方式1：位置参数
    users[msg.sender] = User({
        wallet: msg.sender,
        balance: 0,
        isActive: true,
        name: _name
    });

    // 方式2：有序参数（不推荐，容易出错）
    // users[msg.sender] = User(msg.sender, 0, true, _name);

    userList.push(users[msg.sender]);
}

// ⚠️ 结构体的一个坑：不能直接修改映射中结构体的字段值
// users[msg.sender].balance += 10;  // ❌ 编译错误！

// 正确做法：先存到 storage 引用，再修改
function addBalance(uint256 _amount) public {
    User storage user = users[msg.sender];  // storage 引用
    user.balance += _amount;                // ✅
}
```

---

## 4. 变量与作用域

### 4.1 三种数据位置

这是 Solidity 独有的概念，直接影响 Gas 消耗和程序行为。

| 位置 | 存储在哪 | 持久性 | Gas | 使用场景 |
|------|---------|--------|-----|---------|
| **storage** | 区块链上 | 永久 | 极高 | 状态变量 |
| **memory** | 内存 | 函数调用期间 | 低 | 函数参数、临时变量 |
| **calldata** | 调用数据 | 函数调用期间 | 最低 | external 函数参数 |

```solidity
contract DataLocation {
    uint256[] public storageArray;  // storage：区块链上永久存储

    function demo(uint256[] calldata _input) external {
        // _input 在 calldata 中，不可修改，Gas 最低

        uint256[] memory memCopy = new uint256[](_input.length);
        for (uint256 i = 0; i < _input.length; i++) {
            memCopy[i] = _input[i] * 2;  // memory 可修改
        }

        // 写入 storage（最贵）
        for (uint256 i = 0; i < memCopy.length; i++) {
            storageArray.push(memCopy[i]);
        }
    }
}
```

> **经验法则**：能用 `calldata` 就不用 `memory`（省拷贝 Gas），能少写 `storage` 就少写（省存储 Gas）。

### 4.2 状态变量 vs 局部变量

```solidity
contract Scope {
    uint256 public count;              // 状态变量：storage，永久
    string public name = "default";   // 状态变量：storage

    function increment() public {
        uint256 temp = count + 1;      // 局部变量：memory
        count = temp;                  // 写入 storage

        // ⚠️ 循环变量必须是局部变量
        for (uint256 i = 0; i < 10; i++) {
            // i 是 memory 变量
        }
    }
}
```

### 4.3 常量与不可变量

```solidity
contract Constants {
    // constant：编译时确定，不占用 storage slot
    uint256 public constant MAX_SUPPLY = 10000;
    address public constant DEAD = address(0xdead);

    // immutable：部署时确定（constructor 中赋值），之后不可修改
    address public immutable OWNER;
    uint256 public immutable CREATED_AT;

    constructor() {
        OWNER = msg.sender;
        CREATED_AT = block.timestamp;
    }
}
```

| 特性 | constant | immutable |
|------|----------|-----------|
| 赋值时机 | 编译时 | 部署时（constructor） |
| 可用类型 | 值类型 + 字符串 | 值类型 |
| Gas（读取） | 0（内联到字节码） | 极低 |
| 存储 slot | 不占用 | 不占用 |

---

## 5. 运算符与表达式

### 5.1 算术运算

```solidity
uint256 a = 10;
uint256 b = 3;

a + b;    // 13
a - b;    // 7
a * b;    // 30
a / b;    // 3（整数除法，向下取整）
a % b;    // 1
a ** b;   // 1000（指数运算，⚠️ Gas 较高）

// ⚠️ 0.8.x 溢出保护
uint8 c = 255;
// c + 1;  // revert！不是回绕到 0

// 如果确实需要回绕行为（不推荐）：
unchecked { c + 1; }  // 0，跳过溢出检查省 Gas
```

### 5.2 类型转换

```solidity
// 小 → 大：安全，隐式转换
uint8 small = 100;
uint256 big = small;  // ✅

// 大 → 小：不安全，必须显式转换，可能截断
uint256 big = 300;
uint8 small = uint8(big);  // 44！300 % 256 = 44

// address 相关
address addr = address(uint160(someUint));
uint160 addrInt = uint160(addr);

// ⚠️ int ↔ uint 转换
int256 signed = -1;
uint256 unsigned = uint256(signed);  // 2^256 - 1（不是 1！）
```

---

## 6. 控制流

```solidity
// if-else
if (balance > 100) {
    // ...
} else if (balance > 50) {
    // ...
} else {
    // ...
}

// ⚠️ 没有 switch 语句

// for 循环
for (uint256 i = 0; i < array.length; i++) {
    // ⚠️ 循环变量 i 不能是 uint8（可能溢出）
    // ⚠️ 循环次数受 Gas 限制，无法无限循环
}

// while 循环（少用，Gas 风险更高）
uint256 j = 0;
while (j < 10) {
    j++;
}

// do-while（少用）
uint256 k = 0;
do {
    k++;
} while (k < 10);

// 三元运算符（常用！比 if-else 省 Gas）
string memory label = isActive ? "active" : "inactive";
```

> **关键限制**：所有循环都受 Gas 限制。如果数组有 10000 个元素，遍历它可能超出区块 Gas 上限。这就是为什么 Solidity 中经常用映射替代数组查询。

---

## 7. 函数

函数是 Solidity 的核心——理解函数修饰符，就理解了合约的交互模型。

### 7.1 完整的函数声明

```solidity
function transfer(address to, uint256 amount)
    public           // 可见性（必须指定）
    virtual          // 可重写性（可选）
    returns (bool)   // 返回值
{
    // 函数体
}
```

### 7.2 可见性修饰符

| 修饰符 | 合约内部 | 继承合约 | 外部调用 |
|--------|---------|---------|---------|
| `public` | ✅ | ✅ | ✅ |
| `external` | ❌ | ❌ | ✅ |
| `internal` | ✅ | ✅ | ❌ |
| `private` | ✅ | ❌ | ❌ |

```solidity
contract Visibility {
    uint256 public publicVar = 1;      // public 状态变量自动生成 getter
    uint256 private privateVar = 2;     // 只有本合约可访问
    uint256 internal internalVar = 3;   // 本合约 + 子合约可访问

    function publicFunc() public pure returns (string memory) {
        return "anyone can call";
    }

    function externalFunc() external pure returns (string memory) {
        return "only external calls";
    }

    function internalFunc() internal pure returns (string memory) {
        return "only this + children";
    }

    function privateFunc() private pure returns (string memory) {
        return "only this contract";
    }
}
```

> **public vs external 的 Gas 差异**：对于引用类型参数（数组、字符串），`external` 使用 `calldata`（Gas 更低），`public` 使用 `memory`（需要拷贝）。如果函数只被外部调用，优先用 `external`。

### 7.3 状态修饰符

| 修饰符 | 可读状态 | 可写状态 | Gas | 说明 |
|--------|---------|---------|-----|------|
| `view` | ✅ | ❌ | 不消耗 Gas（外部调用时） | 只读 |
| `pure` | ❌ | ❌ | 不消耗 Gas（外部调用时） | 纯计算 |
| 无修饰 | ✅ | ✅ | 消耗 Gas | 可修改状态 |

```solidity
contract StateModifiers {
    uint256 public count = 0;

    // view：读取状态但不修改
    function getCount() public view returns (uint256) {
        return count;  // 读取 storage
    }

    // pure：既不读取也不修改状态
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;  // 只用参数计算
    }

    // 无修饰：修改状态
    function increment() public {
        count += 1;  // 写入 storage，消耗 Gas
    }
}
```

> **关键区别**：`view` 和 `pure` 函数在**外部调用**时不消耗 Gas（节点本地执行），但在**内部调用**（从另一个交易中调用）时仍然消耗 Gas。

### 7.4 特殊函数

```solidity
// 1. constructor：部署时执行一次
constructor(address _token) {
    owner = msg.sender;
    token = IERC20(_token);
}

// 2. receive：接收纯 ETH 转账（无 calldata）
receive() external payable {
    emit Received(msg.sender, msg.value);
}

// 3. fallback：调用不存在的函数时触发，或接收带 calldata 的 ETH
fallback() external payable {
    emit FallbackCalled(msg.data);
}

// ⚠️ receive 与 fallback 的选择逻辑：
// 发送 ETH 时，如果有 receive → 调 receive
// 没有 receive → 调 fallback
// 两个都没有 → revert
```

```solidity
// 实际例子：合约接收 ETH 的完整处理
contract EthReceiver {
    event Deposit(address indexed from, uint256 amount, bytes data);

    receive() external payable {
        emit Deposit(msg.sender, msg.value, "");
    }

    fallback() external payable {
        emit Deposit(msg.sender, msg.value, msg.data);
    }

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

### 7.5 函数重载与重写

```solidity
// 重载：同名函数，不同参数
contract Overload {
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }

    function add(uint256 a, uint256 b, uint256 c) public pure returns (uint256) {
        return a + b + c;
    }
}

// 重写：子合约覆盖父合约函数
contract Parent {
    function greet() public pure virtual returns (string memory) {
        return "Hello from Parent";
    }
}

contract Child is Parent {
    function greet() public pure override returns (string memory) {
        return "Hello from Child";
    }
}
```

---

## 8. 事件（Events）

事件是合约与外部世界通信的唯一桥梁——它们写入交易日志（tx log），不在状态中存储。

```solidity
contract Events {
    // 声明：indexed 参数最多 3 个（用于过滤）
    event Transfer(address indexed from, address indexed to, uint256 amount);
    event Approval(address indexed owner, address indexed spender, uint256 amount);

    function transfer(address to, uint256 amount) public {
        // ...转账逻辑...
        emit Transfer(msg.sender, to, amount);
    }
}
```

| 特性 | indexed 参数 | 非 indexed 参数 |
|------|-------------|----------------|
| 可过滤 | ✅ | ❌ |
| 数量限制 | 最多 3 个 | 无限制 |
| 存储方式 | 作为 topic（32字节） | 作为 data（ABI 编码） |
| Gas 成本 | 较低（375 gas/topic） | 较高（按字节计费） |

```javascript
// 前端监听事件
contract.on("Transfer", (from, to, amount) => {
    console.log(`${from} → ${to}: ${amount}`);
});

// 过滤 indexed 参数
const filter = contract.filters.Transfer(fromAddress, null);
```

---

## 9. 错误处理

### 9.1 三种方式

```solidity
// ❌ 0.8.x 之前：字符串错误（Gas 高，错误信息上链）
require(balance >= amount, "Insufficient balance");

// ✅ 0.8.4+：自定义错误（Gas 低，错误信息不上链）
error InsufficientBalance(uint256 available, uint256 required);

function withdraw(uint256 amount) public {
    if (balance < amount)
        revert InsufficientBalance(balance, amount);
}
```

| 方式 | Gas | 可携带数据 | 推荐度 |
|------|-----|-----------|--------|
| `require("message")` | 高（字符串上链） | ❌ | ⚠️ 旧代码常见 |
| `revert CustomError()` | 低 | ✅ | ✅ 推荐 |
| `assert()` | 高 | ❌ | ❌ 仅用于内部不变量检查 |

### 9.2 自定义错误

```solidity
error Unauthorized(address caller);
error InsufficientBalance(uint256 available, uint256 required);
error TransferFailed(address to, uint256 amount);

contract Token {
    mapping(address => uint256) public balances;

    function withdraw(uint256 amount) public {
        if (msg.sender != owner)
            revert Unauthorized(msg.sender);

        if (balances[msg.sender] < amount)
            revert InsufficientBalance(balances[msg.sender], amount);

        balances[msg.sender] -= amount;
        (bool success, ) = msg.sender.call{value: amount}("");
        if (!success)
            revert TransferFailed(msg.sender, amount);
    }
}
```

> **为什么自定义错误省 Gas？** 字符串需要按字节存储在交易收据中（至少 375 + 8*字节数 gas），而自定义错误只用一个 4 字节选择器标识错误类型，参数 ABI 编码。

---

## 10. 全局变量与单位

### 10.1 常用全局变量

```solidity
// 消息相关
msg.sender    // address：调用者地址
msg.value     // uint256：附带的 ETH 数量（wei）
msg.data      // bytes：完整的 calldata
msg.sig       // bytes4：函数选择器（前4字节）

// 交易相关
tx.origin     // address：交易发起者（⚠️ 不建议用于认证）
tx.gasprice   // uint256：Gas 价格

// 区块相关
block.number      // uint256：当前区块号
block.timestamp   // uint256：当前区块时间戳（Unix秒）
block.chainid     // uint256：链 ID
block.difficulty  // uint256：难度（⚠️ PoS 后为 prevrandao）
block.gaslimit    // uint256：区块 Gas 上限
block.coinbase    // address payable：矿工/验证者地址

// ⚠️ 安全警告：
// - 不要用 block.timestamp 做随机数来源（矿工可操纵）
// - 不要用 tx.origin 做权限检查（钓鱼攻击可绕过）
```

### 10.2 单位

```solidity
// ETH 单位
uint256 oneWei = 1 wei;
uint256 oneGwei = 1 gwei;     // = 1e9 wei
uint256 oneEther = 1 ether;   // = 1e18 wei

// 时间单位
uint256 oneSecond = 1 seconds;
uint256 oneMinute = 1 minutes;  // = 60 seconds
uint256 oneHour = 1 hours;      // = 3600 seconds
uint256 oneDay = 1 days;        // = 86400 seconds
uint256 oneWeek = 1 weeks;     // = 604800 seconds

// ⚠️ 时间单位只是字面量常数，不随闰秒等变化
// 1 years 已在 0.7.0 移除（因为闰年问题）
```

---

## 11. 接口与抽象合约

### 11.1 接口（Interface）

```solidity
// 接口：完全的"合同"——只定义函数签名，不定义实现
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

接口的规则：
- 所有函数必须是 `external`
- 不能有构造函数
- 不能有状态变量
- 可以有事件
- 继承接口的合约**必须实现所有函数**

### 11.2 抽象合约（Abstract Contract）

```solidity
// 抽象合约：可以有实现的函数，也可以有没有实现的函数
abstract contract Ownable {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    // 没有实现的函数——子合约必须实现
    function withdraw(uint256 amount) external virtual;
}

contract MyContract is Ownable {
    mapping(address => uint256) public balances;

    function withdraw(uint256 amount) external override onlyOwner {
        // 具体实现
        payable(owner).transfer(amount);
    }
}
```

---

## 12. 修饰器（Modifier）

修饰器是 Solidity 的权限控制机制，`_` 是函数体插入点。

```solidity
contract Modifiers {
    address public owner;
    bool public paused;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;   // ← 函数体插入这里
    }

    modifier whenNotPaused() {
        require(!paused, "Contract is paused");
        _;
    }

    modifier costs(uint256 price) {
        require(msg.value >= price, "Insufficient payment");
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    // 多个修饰器按顺序执行
    function buy()
        external
        payable
        whenNotPaused
        costs(1 ether)
    {
        // 先检查 whenNotPaused → 再检查 costs → 然后执行函数体
    }

    // 修饰器也可以在函数执行后运行
    modifier lockUntilDone() {
        require(!locked, "Reentrancy");
        locked = true;
        _;              // 函数体先执行
        locked = false;  // 函数体执行后再解锁
    }

    bool private locked;
}
```

---

## 快速参考卡

| 概念 | 关键记忆点 |
|------|-----------|
| 值类型 | 赋值拷贝值，uint=int256，0.8.x 自动溢出检查 |
| 引用类型 | 赋值拷贝引用，mapping 不可遍历，struct 需 storage 引用修改 |
| 数据位置 | calldata < memory < storage（Gas 从低到高） |
| 函数可见性 | public > external > internal > private |
| 函数状态 | pure < view < 无修饰（Gas 从低到高） |
| 错误处理 | 用自定义错误，不用 require("string") |
| 事件 | indexed 最多 3 个，是合约与外部通信的唯一方式 |
| 修饰器 | `_` 是插入点，可前置检查也可后置清理 |
| ETH 转账 | 用 `.call{value: }("")`，不用 `.transfer()` |
| 全局变量 | `msg.sender` ✅ `tx.origin` ❌ `block.timestamp` ⚠️ |
