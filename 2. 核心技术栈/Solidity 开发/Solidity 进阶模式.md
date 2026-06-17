# Solidity 进阶模式

> 掌握语法基础后，你需要理解这些模式才能写出生产级合约。本文聚焦"为什么这样写"而非"语法是什么"。

---

## 1. 继承与多态

### 1.1 单继承

```solidity
contract Animal {
    string public species;

    constructor(string memory _species) {
        species = _species;
    }

    function speak() public pure virtual returns (string memory) {
        return "...";
    }
}

contract Dog is Animal {
    constructor() Animal("Canine") {}

    function speak() public pure override returns (string memory) {
        return "Woof!";
    }
}
```

### 1.2 多重继承与 C3 线性化

Solidity 的多重继承采用 **C3 线性化**（Python 同款算法）确定调用顺序——从**最基类到最派生类**。

```solidity
contract A {
    function foo() public pure virtual returns (string memory) {
        return "A";
    }
}

contract B is A {
    function foo() public pure override returns (string memory) {
        return "B";
    }
}

contract C is A {
    function foo() public pure override returns (string memory) {
        return "C";
    }
}

// ⚠️ 必须按"最基类 → 最派生类"的顺序声明
contract D is B, C {   // 线性化：A → B → C → D
    function foo() public pure override(B, C) returns (string memory) {
        // super.foo() 调用的是 C（不是 B！因为 C 在 B 之后）
        return string.concat(super.foo(), " D");  // "C D"
    }
}

// 另一个顺序
contract E is C, B {   // 线性化：A → C → B → E
    function foo() public pure override(C, B) returns (string memory) {
        return string.concat(super.foo(), " E");  // "B E"（注意不是 C！）
    }
}
```

> **关键理解**：`super` 不是指"父类"，而是指"C3 线性化中的下一个合约"。顺序不同，`super` 指向的目标就不同。

### 1.3 构造函数的继承

```solidity
contract Ownable {
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }
}

// 方式1：在声明中传递
contract MyToken is Ownable(msg.sender) {
    // ...
}

// 方式2：在自己的 constructor 中传递
contract MyToken is Ownable {
    constructor() Ownable(msg.sender) {
        // ...
    }
}

// ⚠️ 如果父合约有无参构造函数，不需要显式传递
contract SimpleOwnable is Ownable {
    constructor() Ownable(msg.sender) {}
}
```

### 1.4 钻石继承问题

```solidity
//        Animal
//       /      \
//    Pet       Guard
//       \      /
//       GuardDog

contract Animal {
    function name() public pure virtual returns (string memory) {
        return "Animal";
    }
}

contract Pet is Animal {
    function name() public pure override returns (string memory) {
        return "Pet";
    }
}

contract Guard is Animal {
    function name() public pure override returns (string memory) {
        return "Guard";
    }
}

// ✅ 必须重写，解决歧义
contract GuardDog is Pet, Guard {
    function name() public pure override(Pet, Guard) returns (string memory) {
        return "GuardDog";
    }
}
```

---

## 2. 接口与抽象合约的选择

| 特性 | 接口 (interface) | 抽象合约 (abstract) |
|------|-----------------|-------------------|
| 有实现的函数 | ❌ | ✅ |
| 构造函数 | ❌ | ✅ |
| 状态变量 | ❌ | ✅ |
| 事件 | ✅ | ✅ |
| 函数可见性 | 只能 external | 任意 |
| 多重继承 | ✅ | ✅ |
| 典型用途 | 标准定义（ERC-20等） | 共享逻辑+部分实现 |

### 实战模式：接口定义标准 + 抽象合约提供骨架

```solidity
// 第1层：接口——定义标准
interface IStaking {
    function stake(uint256 amount) external;
    function unstake(uint256 amount) external;
    function earnReward(address account) external view returns (uint256);
    event Staked(address indexed account, uint256 amount);
    event Unstaked(address indexed account, uint256 amount);
}

// 第2层：抽象合约——提供通用逻辑骨架
abstract contract StakingBase is IStaking {
    mapping(address => uint256) public stakedBalance;
    uint256 public totalStaked;

    modifier hasStake() {
        require(stakedBalance[msg.sender] > 0, "No stake");
        _;
    }

    function stake(uint256 amount) external virtual override;
    function unstake(uint256 amount) external virtual override;
}

// 第3层：具体合约——填入具体逻辑
contract EthStaking is StakingBase {
    function stake(uint256 amount) external override {
        require(amount > 0, "Cannot stake 0");
        stakedBalance[msg.sender] += amount;
        totalStaked += amount;
        emit Staked(msg.sender, amount);
    }

    function unstake(uint256 amount) external override hasStake {
        require(stakedBalance[msg.sender] >= amount, "Insufficient stake");
        stakedBalance[msg.sender] -= amount;
        totalStaked -= amount;
        emit Unstaked(msg.sender, amount);
    }

    function earnReward(address account) external pure override returns (uint256) {
        // 简化示例
        return 0;
    }
}
```

---

## 3. 库（Library）

库是 Solidity 的"工具箱"——可复用的函数集合，但不能有状态变量。

### 3.1 内联库 vs 部署库

```solidity
// 内联库：所有函数都是 internal，编译时内联到调用合约
library Math {
    function max(uint256 a, uint256 b) internal pure returns (uint256) {
        return a >= b ? a : b;
    }

    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
}

// 使用
contract Calculator {
    using Math for uint256;

    function test() public pure returns (uint256) {
        uint256 a = 10;
        return a.max(20);  // 20（像方法调用一样使用）
    }
}
```

```solidity
// 部署库：包含 public/external 函数，需要单独部署
library StringUtils {
    function toUpper(string memory s) public pure returns (string memory) {
        bytes memory b = bytes(s);
        for (uint256 i = 0; i < b.length; i++) {
            if (b[i] >= 0x61 && b[i] <= 0x7A) {
                b[i] = bytes1(uint8(b[i]) - 32);
            }
        }
        return string(b);
    }
}

// 使用部署库时，调用通过 delegatecall
contract MyContract {
    using StringUtils for string;

    function shout(string memory s) public pure returns (string memory) {
        return s.toUpper();
    }
}
```

### 3.2 经典案例：SafeMath

在 0.8.x 之前，所有算术运算都需要手动检查溢出：

```solidity
// 0.7.x 时代的写法
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        return a - b;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }
}

// 0.8.x 以后：不再需要！内置溢出检查
// 除非你用 unchecked {} 显式跳过
```

### 3.3 库的限制

- 不能有状态变量
- 不能继承或被继承
- 不能接收 ETH
- `internal` 函数内联（不占部署成本），`public` 函数需要单独部署

---

## 4. using...for 与函数式调用

```solidity
// using A for B：让类型 B 可以用 A 中定义的函数
using SafeMath for uint256;
using Address for address;

// 两种调用方式等价
uint256 result = a.add(b);         // 函数式调用
uint256 result = SafeMath.add(a, b); // 直接调用

// 实战：扩展现有类型
library ArrayUtils {
    function remove(uint256[] storage arr, uint256 index) internal {
        // 用最后一个元素覆盖要删除的元素，然后 pop
        // 这比 delete arr[index] 好——不会留下空洞
        require(index < arr.length, "Index out of bounds");
        arr[index] = arr[arr.length - 1];
        arr.pop();
    }

    function sum(uint256[] memory arr) internal pure returns (uint256) {
        uint256 total;
        for (uint256 i = 0; i < arr.length; i++) {
            total += arr[i];
        }
        return total;
    }
}

contract MyContract {
    using ArrayUtils for uint256[];

    uint256[] public numbers;

    function removeAt(uint256 index) public {
        numbers.remove(index);  // 像原生方法一样调用
    }
}
```

---

## 5. 合约工厂与注册表模式

### 5.1 合约工厂

当需要动态创建多个同类型合约时（如每个用户一个钱包、每个项目一个众筹合约）：

```solidity
// 产品合约
contract Campaign {
    address public creator;
    uint256 public goal;
    uint256 public deadline;
    uint256 public raised;
    bool public claimed;
    mapping(address => uint256) public contributions;

    constructor(uint256 _goal, uint256 _duration) {
        creator = msg.sender;
        goal = _goal;
        deadline = block.timestamp + _duration;
    }

    function contribute() external payable {
        require(block.timestamp < deadline, "Ended");
        contributions[msg.sender] += msg.value;
        raised += msg.value;
    }

    function claim() external {
        require(msg.sender == creator, "Not creator");
        require(raised >= goal, "Goal not reached");
        require(!claimed, "Already claimed");
        claimed = true;
        payable(creator).transfer(raised);
    }
}

// 工厂合约
contract CampaignFactory {
    Campaign[] public campaigns;

    event CampaignCreated(address indexed campaign, address indexed creator, uint256 goal);

    function createCampaign(uint256 _goal, uint256 _duration) external {
        Campaign newCampaign = new Campaign(_goal, _duration);
        campaigns.push(newCampaign);
        emit CampaignCreated(address(newCampaign), msg.sender, _goal);
    }

    function getCampaignCount() external view returns (uint256) {
        return campaigns.length;
    }
}
```

### 5.2 注册表模式

需要"从地址到合约实例"的映射时：

```solidity
contract TokenRegistry {
    struct TokenInfo {
        address tokenAddress;
        string name;
        string symbol;
        bool active;
    }

    mapping(address => TokenInfo) public tokens;
    address[] public tokenList;

    event TokenRegistered(address indexed token, string name);

    function register(address _token, string memory _name, string memory _symbol) external {
        require(tokens[_token].tokenAddress == address(0), "Already registered");

        tokens[_token] = TokenInfo({
            tokenAddress: _token,
            name: _name,
            symbol: _symbol,
            active: true
        });

        tokenList.push(_token);
        emit TokenRegistered(_token, _name);
    }

    function getActiveTokens() external view returns (address[] memory) {
        // 先计算活跃数量
        uint256 count;
        for (uint256 i = 0; i < tokenList.length; i++) {
            if (tokens[tokenList[i]].active) count++;
        }

        // 再填充数组
        address[] memory active = new address[](count);
        uint256 j;
        for (uint256 i = 0; i < tokenList.length; i++) {
            if (tokens[tokenList[i]].active) {
                active[j++] = tokenList[i];
            }
        }
        return active;
    }
}
```

---

## 6. 状态机模式

很多合约有明确的生命周期阶段，状态机模式让状态转换显式化、安全化：

```solidity
contract Escrow {
    enum State { Created, Funded, Delivered, Confirmed, Refunded, Cancelled }

    State public state = State.Created;
    address public buyer;
    address public seller;
    uint256 public amount;

    error InvalidTransition(State from, State to);

    modifier transition(State _expected) {
        if (state != _expected) revert InvalidTransition(state, _expected);
        _;
    }

    constructor(address _seller) payable {
        buyer = msg.sender;
        seller = _seller;
        amount = msg.value;
        state = State.Funded;  // 创建时即打款
    }

    function deliver() external transition(State.Funded) {
        require(msg.sender == seller, "Only seller");
        state = State.Delivered;
    }

    function confirm() external transition(State.Delivered) {
        require(msg.sender == buyer, "Only buyer");
        state = State.Confirmed;
        payable(seller).transfer(amount);
    }

    function refund() external transition(State.Delivered) {
        require(msg.sender == seller, "Only seller");
        state = State.Refunded;
        payable(buyer).transfer(amount);
    }

    function cancel() external transition(State.Created) {
        state = State.Cancelled;
        payable(buyer).transfer(amount);
    }
}
```

---

## 7. 可升级合约模式

> 关于部署与升级成本的详细讨论，参见 [智能合约部署与升级成本](../../8.%20测试与部署/部署流程/智能合约部署与升级成本.md)

### 7.1 代理模式的核心原理

```
用户 → 代理合约（存数据） ──delegatecall──→ 逻辑合约（存代码）
```

`delegatecall` 的特殊性：在调用者（代理）的上下文中执行被调用者（逻辑）的代码。

```solidity
// 极简代理示例
contract SimpleProxy {
    address public implementation;  // slot 0
    address public admin;           // slot 1

    constructor(address _impl) {
        implementation = _impl;
        admin = msg.sender;
    }

    // 核心：delegatecall 到逻辑合约
    fallback() external payable {
        (bool success, bytes memory data) = implementation.delegatecall(msg.data);
        require(success, "Delegatecall failed");
        assembly {
            return(add(data, 32), mload(data))
        }
    }

    receive() external payable {}

    function upgrade(address _newImpl) external {
        require(msg.sender == admin, "Only admin");
        implementation = _newImpl;
    }
}
```

### 7.2 存储布局冲突——升级的铁律

```solidity
// ❌ 错误：升级时在中间插入变量
contract LogicV1 {
    uint256 public value;      // slot 0
    address public owner;      // slot 1
}

contract LogicV2_Bad {
    uint256 public value;      // slot 0
    uint256 public newField;   // slot 1 ← 抢了原来 owner 的位置！
    address public owner;      // slot 2
}

// ✅ 正确：只在末尾追加
contract LogicV2_Good {
    uint256 public value;      // slot 0
    address public owner;      // slot 1
    uint256 public newField;   // slot 2 ← 追加到末尾
}
```

**升级存储布局铁律**：
1. **不能**修改已有变量的类型
2. **不能**删除已有变量
3. **不能**在已有变量之间插入新变量
4. **只能**在末尾追加新变量
5. **不能**改变继承顺序

---

## 8. 代理模式对比

| 特性 | Transparent | UUPS | Diamond | Beacon |
|------|-----------|------|---------|--------|
| 升级执行者 | Admin（独立地址） | 合约自身 | 多个 facet | Beacon 合约 |
| 代理合约复杂度 | 高 | 低 | 最高 | 中 |
| 逻辑合约复杂度 | 低 | 中 | 高 | 低 |
| Gas（每次调用） | 高（需检查调用者） | 低 | 中 | 中 |
| 多逻辑合约 | ❌ | ❌ | ✅ | ✅（共享升级） |
| 推荐场景 | 简单项目 | 大多数项目 | 复杂模块化 | 同逻辑多实例 |

### Transparent Proxy

```solidity
// 代理合约根据调用者身份决定行为：
// - Admin 调用 → 执行代理自身的管理函数
// - 其他调用 → delegatecall 到逻辑合约

contract TransparentProxy {
    address public implementation;
    address public admin;

    fallback() external payable {
        if (msg.sender == admin) {
            // admin 只能调代理管理函数（upgrade 等）
            return;
        }
        // 非 admin 用户 → delegatecall 到逻辑合约
        (bool success, bytes memory data) = implementation.delegatecall(msg.data);
        require(success);
        assembly { return(add(data, 32), mload(data)) }
    }
}
```

### UUPS (Universal Upgradeable Proxy Standard)

```solidity
// UUPS 把升级逻辑放在逻辑合约中，而非代理中
// 优势：代理更简单，Gas 更低
// 劣势：如果逻辑合约忘了写 upgradeTo → 永远无法升级！

contract UUPSLogicV1 {
    address public implementation;  // slot 0（代理存储，由代理合约使用）
    address public admin;           // slot 1

    function upgradeTo(address _newImpl) external {
        require(msg.sender == admin, "Only admin");
        // 直接修改代理的 implementation slot
        assembly {
            sstore(0, _newImpl)  // slot 0 = implementation
        }
    }
}
```

---

## 9. ERC-20 完整实现

将以上所有模式综合运用的经典案例：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

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

error ERC20InsufficientBalance(address sender, uint256 balance, uint256 needed);
error ERC20InvalidSender(address sender);
error ERC20InvalidReceiver(address receiver);
error ERC20InsufficientAllowance(address spender, uint256 allowance, uint256 needed);

contract SimpleToken is IERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    address public immutable owner;

    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }

    constructor(string memory _name, string memory _symbol, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        owner = msg.sender;

        balanceOf[msg.sender] = _initialSupply;
        totalSupply = _initialSupply;
        emit Transfer(address(0), msg.sender, _initialSupply);
    }

    function transfer(address to, uint256 amount) public returns (bool) {
        return _transfer(msg.sender, to, amount);
    }

    function approve(address spender, uint256 amount) public returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public returns (bool) {
        uint256 currentAllowance = allowance[from][msg.sender];
        if (currentAllowance != type(uint256).max) {
            if (currentAllowance < amount)
                revert ERC20InsufficientAllowance(msg.sender, currentAllowance, amount);
            unchecked {
                allowance[from][msg.sender] = currentAllowance - amount;
            }
        }
        return _transfer(from, to, amount);
    }

    function _transfer(address from, address to, uint256 amount) internal returns (bool) {
        if (from == address(0)) revert ERC20InvalidSender(address(0));
        if (to == address(0)) revert ERC20InvalidReceiver(address(0));

        uint256 fromBalance = balanceOf[from];
        if (fromBalance < amount)
            revert ERC20InsufficientBalance(from, fromBalance, amount);

        unchecked {
            balanceOf[from] = fromBalance - amount;
            balanceOf[to] += amount;
        }

        emit Transfer(from, to, amount);
        return true;
    }

    // 铸造新代币（仅所有者）
    function mint(address to, uint256 amount) external onlyOwner {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
}
```

---

## 10. 设计原则总结

| 原则 | 说明 | 反面案例 |
|------|------|---------|
| **最小权限** | 每个函数只给必要的权限 | 所有人都能调 admin 函数 |
| **最小状态** | 只存链上必须的数据，链下存元数据 | 把图片 URI 上链 |
| **Fail-safe** | 默认安全状态，出错时回滚 | 出错时自动转账 |
| **检查-生效-交互** | 先检查条件，再改状态，最后外部调用 | 先外部调用再改状态（重入攻击） |
| **存储布局不可变** | 升级只能追加变量，不能改顺序 | 升级时在中间插入变量 |
| **事件驱动** | 重要操作必须 emit event | 转账没有事件，前端无法监听 |
| **Gas 意识** | 循环受 Gas 限制，映射不可遍历 | 遍历 10000 元素的数组 |

### 检查-生效-交互（CEI）模式

这是防重入攻击最根本的模式：

```solidity
// ❌ 错误：交互在生效之前 → 重入漏洞
function withdrawBad() external {
    uint256 amount = balances[msg.sender];
    (bool success, ) = msg.sender.call{value: amount}("");  // 交互：外部调用
    require(success);
    balances[msg.sender] = 0;   // 生效：修改状态（太晚了！攻击者可以重入）
}

// ✅ 正确：检查 → 生效 → 交互
function withdrawGood() external {
    uint256 amount = balances[msg.sender];   // 检查
    require(amount > 0, "No balance");

    balances[msg.sender] = 0;                // 生效：先改状态
    emit Withdrawn(msg.sender, amount);

    (bool success, ) = msg.sender.call{value: amount}("");  // 交互：最后外部调用
    require(success, "Transfer failed");
}
```
