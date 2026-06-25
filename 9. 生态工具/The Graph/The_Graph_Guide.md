# The Graph 链上数据去中心化索引服务指南

> 在 Web3 全栈开发中，如何高效检索、筛选和展示复杂的链上历史数据，是区分初级开发者与架构师的试金石。以太坊底层的键值对存储账本并不支持传统数据库的复杂关系查询。**The Graph** 作为全球去中心化链上索引（Indexing）标准的底层协议，解决了这一核心痛点。本指南将详解传统 RPC 检索困局、The Graph 的子图架构，并提供一份生产级的 Subgraph（子图）核心三件套编写实战模板。

---

## 1. 传统以太坊链上检索的严重困局

在智能合约中，数据通常以 `mapping` 的哈希形式存储在 Slot 中（如：`mapping(address => UserInfo) public users`）。
- **局限性**：Solidity 支持根据特定的 `address` 精确查询这一个用户的 `UserInfo`。
- **检索困局**：然而，如果你要查询类似以下的**复合条件关系数据**：
  1. “在过去 24 小时内，全网存入资金大于 $10,000$ 美元的所有用户有哪些？”
  2. “某个特定用户历史上发生的最近 5 笔交易记录及时间戳是什么？”
  3. “某个 NFT 交易市场中，交易量排名前 5 的稀有度属性分布是怎样的？”

- **传统 JSON-RPC 的笨拙手段**：开发者不得不使用 `eth_getLogs`。前端或后台脚本需要向节点请求，从上百万个历史区块中**极其缓慢且昂贵地进行拉取并执行逐块循环遍历过滤**。这会导致极慢的网页加载速度、由于单次拉取日志超限被 RPC 平台中断拦截，耗费巨额的 RPC 计费 CU。

---

## 2. The Graph 的高并发、去中心化解耦架构

The Graph 通过将**链上高频事件监听**与**链下高性能 PostgreSQL（关系型数据库）缓存**完美解耦，彻底终结了 RPC 检索的慢和贵：

```
 [1. 触发事件] ──> 链上合约抛出 `event Deposited(address user, uint256 amount)`
                        │
                        ▼
 [2. 图节点监控] ─> Graph Node 实时扫描区块，拦截到该事件
                        │
                        ▼
 [3. 映射逻辑] ──> 调用 Mapping 脚本（AssemblyScript），重组并处理数据
                        │
                        ▼
 [4. 写入数据库] ─> 自动写入图节点底层的关系型 PostgreSQL 数据库中
                        │
                        ▼
 [5. 毫秒级查询] <─ 前端 DApp 利用超轻量的 GraphQL 复合条件直接发起毫秒级毫秒筛选查询
```

---

## 3. 生产级 Subgraph（子图）核心三件套编写

要将你的智能合约接入 The Graph 的索引系统，你需要定义并部署一个名为 **Subgraph（子图）** 的配置。一个完整的子图由以下三个核心文件组成：

### 3.1 核心一：`schema.graphql` (实体关系数据模型)
使用 GraphQL 的模式定义语言（SDL），声明你需要在链下进行复合查询的数据表（Entities）。

```graphql
# schema.graphql
# 定义用户实体（包含其历史存款的关联信息）
type User @entity {
  id: ID! # 用户的钱包地址（必须作为唯一主键 ID）
  totalDeposits: BigInt! # 累计存款额
  depositCount: Int! # 存款次数
  deposits: [Deposit!]! @derivedFrom(field: "user") # 关联：由 Deposit 实体通过 user 字段反向派生，无需物理建表
}

# 每一笔具体存款的细节实体
type Deposit @entity {
  id: ID! # 交易 Hash + 交易 logIndex 的拼接（确保唯一）
  user: User! # 关联外部 User 实体
  amount: BigInt! # 本次存款的具体金额
  timestamp: BigInt! # 存款发生的时间戳
  blockNumber: BigInt! # 发生时的块高度
}
```

---

### 3.2 核心二：`subgraph.yaml` (子图配置清单)
告诉 Graph Node 需要在什么网络（如 `mainnet`、`arbitrum`）、监听哪个合约地址、在哪个块高度开始扫描（`startBlock`，防范从 0 块开始扫描浪费几天时间）、以及绑定哪些事件回调：

```yaml
specVersion: 0.0.5
schema:
  file: ./schema.graphql
dataSources:
  - kind: ethereum
    name: NestingStore # 数据源名称
    network: arbitrum-one # 监听的网络 (如 Arbitrum One)
    source:
      address: "0x87870B2Da85947F119478e12d460741e30b83C74" # 智能合约在主网部署的地址
      abi: NestingStore # 指向 NestingStore 的 JSON ABI 声明
      startBlock: 19000000 # 核心性能点：从合约部署的起始块开始扫描，免除无谓的历史块扫描时间
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - User
        - Deposit
      abis:
        - name: NestingStore
          file: ./abis/NestingStore.json
      eventHandlers:
        - event: Deposited(indexed address,uint256) # 监听的事件特征
          handler: handleDeposited # AssemblyScript 中的事件接收回调函数
      file: ./src/mapping.ts # 映射文件具体代码位置
```

---

### 3.3 核心三：`mapping.ts` (AssemblyScript 映射转换代码)
采用 AssemblyScript（TypeScript 的严格子集，被编译为 WebAssembly 执行）编写。它负责在 Graph Node 抓取到事件时，执行账本的计算、实体的实例化和落地保存：

```typescript
// mapping.ts
import { BigInt } from "@graphprotocol/graph-ts";
import { Deposited } from "../generated/NestingStore/NestingStore";
import { User, Deposit } from "../generated/schema";

/**
 * @notice 当 Graph Node 捕获到 Deposited 事件时，自动执行此回调
 */
export function handleDeposited(event: Deposited): void {
  let userId = event.params.user.toHexString();
  
  // 1. 读取或初始化 User 实体 (若数据库中尚无此人，则新建)
  let user = User.load(userId);
  if (user == null) {
    user = new User(userId);
    user.totalDeposits = BigInt.fromI32(0);
    user.depositCount = 0;
  }

  // 2. 累加用户的内部属性
  user.totalDeposits = user.totalDeposits.plus(event.params.amount);
  user.depositCount = user.depositCount + 1;
  user.save(); // 将状态保存并落地写入 PostgreSQL 数据库

  // 3. 创建并记录本次独立的 Deposit 细分流水
  // 使用 txHash + logIndex 作为 Deposit 的唯一 ID
  let depositId = event.transaction.hash.toHexString() + "-" + event.logIndex.toString();
  let deposit = new Deposit(depositId);
  
  deposit.user = userId; // 建立对 User 的实体外键绑定
  deposit.amount = event.params.amount;
  deposit.timestamp = event.block.timestamp;
  deposit.blockNumber = event.block.number;
  deposit.save(); // 落地保存
}
```

---

## 4. GraphQL 极速复合条件查询实战

当 Subgraph 部署完毕并完成同步后，前端或后端不需要通过 RPC，只需直接通过 GraphQL API 发送单次查询，即可在**数毫秒之内拉取极度复杂的复合条件数据**。

### 🧪 查询范例：查询全网累计充值最大的前 5 位土豪，并拉取他们历史上最近的 3 次大额存款流水
```graphql
query {
  # (a) 查询 Users 表：按总充值降序排序，取前 5 条，筛选累计充值大于 100 ETH (100 * 1e18 wei) 的用户
  users(
    first: 5,
    orderBy: totalDeposits,
    orderDirection: desc,
    where: { totalDeposits_gt: "100000000000000000000" }
  ) {
    id
    totalDeposits
    depositCount
    
    # (b) 链式查询：自动拉取这些用户下挂的最近 3 笔存款流水，按时间戳降序
    deposits(
      first: 3,
      orderBy: timestamp,
      orderDirection: desc
    ) {
      id
      amount
      timestamp
      blockNumber
    }
  }
}
```

---

## 5. Subgraph 性能与安全性最佳实践

1. **设置合理的 `startBlock`**：
   - ❌ 绝对不要将 `startBlock` 留空（默认从 0 块开始）。如果不设置，Graph Node 会从以太坊创始块（10多年前）慢速同步到最新块，耗费几天时间，且极易产生网络超时报错。
   -  必须设置为该智能合约真实在主网部署的那一个高度。
2. **防范实体 ID 重合碰撞**：
   - 对于非单一账户一笔的流水数据（如 Deposit），不能直接用 `txHash` 或者是 `userAddress` 作为 Entity ID（因为同一个用户会有多次充值，或者同一个 tx 内可能包含多次同事件抛出，这会导致状态被恶意覆盖）。
   -  工业级标准：采用 **`txHash + "-" + logIndex`** 拼接作为唯一流水 ID。
3. **保持 AssemblyScript 极其轻量**：
   - 不要试图在 `mapping.ts` 中写大量的浮点数运算或复杂的遍历去计算极其沉重的业务（比如高深的 APR 指数）。
   -  映射只负责**落地存储与基本累加**。沉重的复杂业务和图表渲染，应由前端在拉取到干净的关系型数据后自行在本地进行二次计算。
