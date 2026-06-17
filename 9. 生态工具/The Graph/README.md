# The Graph

## 概述

The Graph 是去中心化的链上数据索引协议，让 DApp 可以高效查询区块链数据。

## 核心概念

- **Subgraph**：定义如何索引数据的配置
- **Schema**：GraphQL 数据模型定义
- **Mapping**：将链上事件映射为数据的处理函数
- **Query**：GraphQL 查询接口

## Subgraph 开发流程

### 1. 定义 Schema（schema.graphql）

```graphql
type Transfer @entity {
  id: Bytes!
  from: Bytes!
  to: Bytes!
  value: BigInt!
  blockNumber: BigInt!
  blockTimestamp: BigInt!
  transactionHash: Bytes!
}
```

### 2. 配置数据源（subgraph.yaml）

```yaml
dataSources:
  - kind: ethereum
    name: Token
    network: mainnet
    source:
      address: "0x..."
      abi: Token
    mapping:
      kind: ethereum/events
      apiVersion: 0.0.7
      language: wasm/assemblyscript
      entities:
        - Transfer
      abis:
        - name: Token
          file: ./abis/Token.json
      eventHandlers:
        - event: Transfer(indexed address,indexed address,uint256)
          handler: handleTransfer
      file: ./src/mapping.ts
```

### 3. 编写映射（mapping.ts）

```typescript
import { Transfer as TransferEvent } from '../generated/Token/Token';
import { Transfer } from '../generated/schema';

export function handleTransfer(event: TransferEvent): void {
  let entity = new Transfer(
    event.transaction.hash.concatI32(event.logIndex.toI32())
  );
  entity.from = event.params.from;
  entity.to = event.params.to;
  entity.value = event.params.value;
  entity.blockNumber = event.block.number;
  entity.blockTimestamp = event.block.timestamp;
  entity.transactionHash = event.transaction.hash;
  entity.save();
}
```

### 4. 查询数据

```graphql
{
  transfers(where: { from: "0x..." }, orderBy: blockTimestamp, orderDirection: desc) {
    id
    to
    value
    blockTimestamp
  }
}
```
