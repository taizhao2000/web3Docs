# 自建节点

自行部署和维护区块链节点，获取完整的控制权、更低的延迟、和免受任何速率限制的归档 Trace 查询。

## 详细指南

请参阅以下硬核自建运维指南：
- **[自建以太坊节点深度运维指南](../Self_Hosted_Node_Guide.md)**：使用 Docker Compose 同时启动执行层（Geth）和共识层（Lighthouse）双引擎客户端，配置 Engine API、JWT 验证，并配合 Prometheus + Grafana 实现对节点健康红线的监控。

## 常用双引擎搭配方案

| 执行层客户端（EL） | 共识层客户端（CL） | 特点 |
| :--- | :--- | :--- |
| **Geth (Go-Ethereum)** | **Lighthouse (Rust)** | 运行最广泛，性能和内存控制最稳健（**推荐**） |
| **Geth (Go-Ethereum)** | **Prysm (Go)** | 生态最活跃的共识层，UI 指标完善 |
| **Nethermind (C#)** | **Lighthouse (Rust)** | Nethermind 支持在线状态裁剪（Online Pruning），免手动维护 |

## 相关章节
- [NaaS 节点服务综合指南](../Node_Services_Guide.md) — 节点和归档节点的原理介绍
- [Mempool 监控、Gas 预测与 MEV 实战指南](../Mempool_and_MEV_Guide.md) — 内存池工作原理
