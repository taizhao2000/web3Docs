# 自建以太坊节点深度运维指南

> 自建节点是去中心化项目走向生产级的终极方案，它提供 100% 的隐私、免受第三方速率限制、支持归档 trace API。本指南提供生产级双引擎（执行层 + 共识层）部署方案、数据修剪、安全加固及完整监控配置。

---

## 1. 节点双引擎架构（The Merge 后的以太坊）

自以太坊“合并”（The Merge）过渡到 Proof of Stake (PoS) 后，运行一个以太坊节点**必须同时部署两个客户端**：
1. **执行层客户端 (Execution Layer, EL)**：负责执行交易、管理状态树（EVM）、广播交易。常用客户端：`Geth`、`Nethermind`、`Besu`。
2. **共识层客户端 (Consensus Layer, CL)**：负责验证区块、处理 PoS 共识、同步信标链状态。常用客户端：`Lighthouse`、`Prysm`、`Teku`。

两类客户端通过安全通道 **Engine API** (运行在 8551 端口，使用 JWT 秘钥验证) 进行本地通信。

```
     DApp / 客户端
           │ (RPC / 8545)
           ▼
┌──────────────────────────┐
│     执行层 (Geth)        │
│    (Execution Layer)     │
└──────────────────────────┘
           ▲
           │ Engine API (JWT / 8551)
           ▼
┌──────────────────────────┐
│   共识层 (Lighthouse)    │
│    (Consensus Layer)     │
└──────────────────────────┘
           ▲
           │ P2P (TCP/UDP 9000)
           ▼
       以太坊信标链
```

---

## 2. 生产级部署实战 (Geth + Lighthouse + Docker Compose)

下面提供一套成熟的、基于 Docker Compose 的主网/测试网全节点部署配置。

### 2.1 准备 JWT 秘钥
在部署前，必须生成一个共享的 JWT 秘钥，供两个客户端握手：

```bash
mkdir -p /srv/ethereum/jwt
openssl rand -hex 32 | sudo tee /srv/ethereum/jwt/jwt.hex
sudo chmod 600 /srv/ethereum/jwt/jwt.hex
```

### 2.2 Docker Compose 配置文件

```yaml
# /srv/ethereum/docker-compose.yml
version: '3.8'

services:
  # ----------------------------------------------------
  # 执行层: Geth (Go-Ethereum)
  # ----------------------------------------------------
  execution:
    image: ethereum/client-go:v1.13.14
    container_name: geth-execution
    restart: unless-stopped
    ports:
      - "30303:30303"      # P2P 发现端口 (必须向公网开放)
      - "30303:30303/udp"
      - "8545:8545"        # HTTP RPC 端口 (建议反代，勿向公网开放)
      - "8546:8546"        # WebSocket 端口 (建议反代)
    volumes:
      - /srv/ethereum/geth-data:/root/.ethereum
      - /srv/ethereum/jwt:/jwt
    command:
      - --mainnet          # 运行在主网 (如需测试网改为 --sepolia)
      - --syncmode=snap    # 快速同步模式 (Snap Sync)
      - --http
      - --http.addr=0.0.0.0
      - --http.vhosts=*
      - --http.api=eth,net,web3,txpool
      - --ws
      - --ws.addr=0.0.0.0
      - --ws.origins=*
      - --ws.api=eth,net,web3,txpool
      - --authrpc.addr=0.0.0.0
      - --authrpc.port=8551
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/jwt/jwt.hex # 握手JWT
      - --metrics
      - --metrics.addr=0.0.0.0
      - --metrics.port=6060

  # ----------------------------------------------------
  # 共识层: Lighthouse (Rust)
  # ----------------------------------------------------
  consensus:
    image: sigp/lighthouse:v5.0.0
    container_name: lighthouse-consensus
    restart: unless-stopped
    depends_on:
      - execution
    ports:
      - "9000:9000"        # P2P 共识层端口 (必须向公网开放)
      - "9000:9000/udp"
      - "5054:5054"        # 指标监控端口
    volumes:
      - /srv/ethereum/lighthouse-data:/root/.lighthouse
      - /srv/ethereum/jwt:/jwt
    command:
      - lighthouse
      - beacon_node
      - --network=mainnet
      - --execution-endpoint=http://execution:8551 # 连接 EL
      - --execution-jwt=/jwt/jwt.hex
      - --checkpoint-sync-url=https://beaconstate.info # 极速检查点同步
      - --http
      - --http-address=0.0.0.0
      - --http-port=5052
      - --metrics
      - --metrics-address=0.0.0.0
      - --metrics-port=5054
```

> 💡 **极其重要：Checkpoint Sync（检查点同步）**：
> 共识层（信标链）如果要从创世区块开始同步需要数天时间。通过传入 `--checkpoint-sync-url=https://beaconstate.info`，共识层可以直接下载最近的受信任快照，在 **3 分钟内**完成共识层同步并开始驱动执行层同步。

### 2.3 同步控制与监控启动

```bash
cd /srv/ethereum
docker compose up -d

# 查看日志
docker compose logs -f execution
```

---

## 3. 硬件开销与网络拓扑

### 3.1 生产硬件最低推荐指标
运行以太坊主网全节点（Full Node）硬件要求不妥协，尤其是 **SSD IOPS**：

| 硬件 | 最低配置 | 生产推荐配置 | 原因 |
|:---|:---|:---|:---|
| **CPU** | 4 核 / 2.9GHz+ | 8 核 / 16 线程 | 交易验证需要强大的并行计算 |
| **内存** | 16 GB | 32 GB | 防止因为 Geth 缓存过大触发 OOM 被系统强杀 |
| **磁盘** | 2 TB **SSD** | **2 TB+ NVMe SSD** | **绝对不能使用机械硬盘 (HDD)**，IOPS 不够会导致区块写入永远跟不上网络同步速度。必须是 NVMe |
| **网络** | 20 Mbps 独享 | 100 Mbps+ 独享 | P2P 节点需要频繁与外部交换区块数据，月度流量达 2TB-4TB |

---

## 4. 节点日常运维：磁盘修剪 (Pruning) 详解

Geth 全节点运行 6-9 个月后，由于积累了大量历史状态树垃圾数据，磁盘占用会逼近 1.2TB+。此时需要进行 **Offline Pruning（离线修剪）**，将磁盘空间压缩回 ~500GB。

### 4.1 Geth 离线修剪步骤
离线修剪需要**暂停节点写入**，运维人员应安排在业务低峰期执行：

```bash
# 1. 暂停节点容器
cd /srv/ethereum
docker compose stop execution

# 2. 运行 Geth 内置修剪命令 (注意容器路径对应 geth-data)
docker compose run --rm execution snapshot prune-state

# 3. 漫长的等待（通常需要 2-4 小时，具体取决于 SSD 性能）
# 期间会显示 Pruning progress...
# 4. 修剪完成后重新启动节点
docker compose start execution
```
*提示：随着 Nethermind 的普及，它支持 "Online Pruning（在线自动修剪）"，如果是企业级长期免维护环境，建议考虑更换执行层为 Nethermind。*

---

## 5. 节点安全加固与 RPC 防护

将裸节点的 8545 (HTTP)/8546 (WS) 直接暴露给公网是极其危险的。黑客可以利用未加锁的 RPC 接口发起拒绝服务攻击，或调用控制台敏感接口。

### 5.1 Nginx 反向代理与 Basic Auth 认证配置

在节点服务器上部署 Nginx，提供加密传输和密码防盗：

```nginx
# /etc/nginx/sites-available/ethereum.conf
server {
    listen 443 ssl;
    server_name rpc.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/rpc.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rpc.yourdomain.com/privkey.pem;

    location / {
        # 开启 Basic Auth 账户密码登录
        auth_basic "Ethereum Secure RPC Portal";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://127.0.0.1:8545;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

```bash
# 生成加密登录密码
sudo apt-get install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd rpc_admin_user
# 输入密码后重启 Nginx
sudo systemctl restart nginx
```

---

## 6. 监控体系：Prometheus + Grafana 大盘配置

生产节点必须配置看板。Geth 提供了 Prometheus 格式指标（Port 6060），Lighthouse（Port 5054）。

### 6.1 核心监控指标 (Metrics) 告警红线

| 监控指标 | Prometheus 指标名 | 异常判定阈值 | 处理方案 |
|:---|:---|:---|:---|
| **同步状态** | `eth_syncing` | `1` | 节点正在追赶状态，处于离线状态 |
| **区块高度差距** | `chain_head_block - eth_block_number` | `> 5` | 节点同步出现卡顿，需要检查共识层 |
| **已连接 P2P 节点数** | `p2p_peers` | `< 8` | 连上的对等节点太少，检查 30303 端口防火墙 |
| **磁盘剩余空间** | `node_disk_written_bytes` | `< 10%` | 磁盘空间不足，触发 Offline Pruning 报警 |
| **系统内存** | `node_memory_Active_bytes` | `> 90%` | 内存溢出风险，需要调大 Geth 的 `--cache` 参数限制 |

### 6.2 导入现成的大盘 Dashboard
不需要从零开始画图，在 Grafana 中可以直接输入以下 Dashboard ID 导入专业级看盘：
- **Geth 官方 Dashboard ID**：`13877`
- **Lighthouse 官方 Dashboard ID**：`12046`
- **Node Exporter (服务器硬件指标) ID**：`1860`
