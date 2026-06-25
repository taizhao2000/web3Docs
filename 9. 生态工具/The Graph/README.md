# The Graph 链上数据去中心化索引

本模块详细探讨了如何解决以太坊 RPC `eth_getLogs` 在高频、高复杂度关系检索（如：查大额充值用户排名、最近历史流水、多条件筛选统计）时遍历太慢、费用昂贵且易超限中断的难题。

---

## 核心学习模块

我们在此专题中准备了极具工业水准的子图核心三件套实战模板：

> 📘 **[The Graph 链上数据去中心化索引服务指南](./The_Graph_Guide.md)**
> 深度解析以下 Subgraph 开发核心技术：
> 1. **去中心化解耦架构**：解析 Graph Node 捕获事件 -> AssemblyScript 脚本重组处理 -> PostgreSQL 高性能落地的底层链下索引流水线。
> 2. **`schema.graphql` 关系建模**：声明子图数据库中的 Entities，建立 User 与 Deposit 之间的外键绑定。
> 3. **`subgraph.yaml` 清单配置**：配置 Arbitrum 等网络、扫描起始块（`startBlock`），指定 Mapping 入口。
> 4. **`mapping.ts` AssemblyScript 代码**：提供完整的、可直接编译部署的事件接收处理和保存实体数据的逻辑脚本。
> 5. **GraphQL 极速复合条件查询**：演示前端如何使用单次查询，在毫秒内拉取累计充值大于 100 ETH 的前 5 位大户，并嵌套带出他们最近 3 笔流水。

---

## 优化箴言

- ** startBlock 同步优化**：必须配置 `startBlock` 为逻辑合约真实部署的区块高度，拒绝 0 块开始的无效漫长历史扫描。
- **防止实体 ID 碰撞**：流水的实体 ID 应该采用 `txHash + "-" + logIndex` 拼接，防范同一 transaction 内发生多笔同事件触发时状态数据被恶意覆盖。
