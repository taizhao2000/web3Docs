# Infura 节点服务

Infura 是 ConsenSys 旗下最流行的以太坊节点即服务（NaaS）平台，提供高性能的标准以太坊 API 访问。

## 详细指南

请参阅核心指南了解更多细节：
- **[NaaS 节点服务综合指南](../Node_Services_Guide.md)**：涵盖 Infura 的连接配置、全节点与归档节点原理，以及如何使用 `FallbackProvider` 实现 Infura 和 Alchemy 之间的热备。

## 快速配置示例

```typescript
import { JsonRpcProvider } from "ethers";

const INFURA_ID = "YOUR_INFURA_PROJECT_ID";

// 连接主网
const provider = new JsonRpcProvider(`https://mainnet.infura.io/v3/${INFURA_ID}`);

// 连接 Sepolia 测试网
const sepoliaProvider = new JsonRpcProvider(`https://sepolia.infura.io/v3/${INFURA_ID}`);
```

## 相关章节
- [Alchemy 节点服务](../Alchemy/) — 支持更丰富增强型 API 的节点服务商
- [自建节点](../自建节点/) — 当请求量超出服务商上限时的自部署方案
