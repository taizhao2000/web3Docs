# 链上事件监控、断线重连与主动安全防卫

> 在 Web3 中，智能合约一旦遭受黑客攻击，多维持一秒的“监控盲区”都可能导致池内的资金流失殆尽。建立一套毫秒级的、24 小时永不停机的链上事件监控告警系统，是项目上线后安全防卫的第一要务。本指南将剖析主流链上监控工具（OpenZeppelin Sentinel / Tenderly），并提供一个基于 **ethers.js v6** 编写的、具备**生产级自动断线重连（指数退避重试）与 Slack / Telegram Webhook 联动通知**的高可靠性监控引擎。

---

## 1. 为什么链上监控对 Web3 安全至关重要？

智能合约是没有主动执行和通知功能的。如果有大额资产非正常流出，或者特权管理员接口被他人冒用，项目团队在链下是无法得知这一情况的。

实时监控（Real-Time Event Monitoring）旨在：
- **感知黑客试探**：在黑客发送小额资金进行漏洞溢出试错时（零日漏洞攻击通常有试探阶段），抢先捕捉到报错事件，发出警报。
- **特权操作追踪**：多签（Multi-Sig）一旦提交了提权、改费率或转账申请，立刻广播给全团队审核，防止私钥泄露暗中作恶。
- **主动联动熔断**：一旦 Sentinel 拦截到攻击信号，自动触发链下防守 Keeper 广播抢跑交易，触发被测合约一键 `pause()`。

---

## 2. 工业界主流事件监控方案

在实际生产中，目前主要有两种主流的 SaaS 化链上安全监控平台：

### 2.1 OpenZeppelin Defender / Sentinel
- **定位**：一站式智能合约运维（SecOps）平台。
- **机制**：直接在云端配置 Sentinel，输入你的智能合约地址和 ABI。当检测到特定事件（如 `EmergencyPaused`）或者特定函数的调用（如 `withdraw` 的 `amount > 100k`）时，自动触发：
  - 飞书、Discord、Slack 机器人消息推送。
  - **触发 Autotask 自动化无服务器函数**：直接在链上广播一笔特定的防御交易。

### 2.2 Tenderly Alerting
- **定位**：智能合约调试、仿真与监控分析平台。
- **机制**：在发生 Revert 交易、Gas 费异常偏高或特定事件抛出时，生成高度可视化的 **Call Trace 错误调用链堆栈**，并实时通过 Webhook 回调通知你的后端监控系统，是安全研究员最常用的“听诊器”。

---

## 3. 生产级 ethers.js v6 断线重连事件监听引擎

### 3.1 为什么普通的 `contract.on` 无法直接在生产中使用？
很多教程会指导你写：
```javascript
const provider = new ethers.WebSocketProvider("wss://...");
contract.on("Transfer", (from, to, value) => { ... });
```
- **生产死穴**：在真实的云服务器（Node.js / PM2）中，由于 RPC 服务商（如 Alchemy / Infura）需要经常进行节点热备、或者因为网络抖动及 Socket 长时间无传输导致心跳包失败，**WebSocket 长连接会在 1 到 2 天之内必然发生断线（Disconnect）**。
- **后果**：一旦断开，普通的 `on` 监听会在链下**静默死掉**（不报错、不崩溃，只是再也无法捕获到新的链上事件），造成极其致命的安全空窗期。
- **防线**：生产环境必须编写一套能够**自动捕捉掉线（Websocket Error / Close）、执行指数退避延迟重连、并重新绑定订阅**的高可靠监听包装类。

---

### 3.2 生产级事件监控引擎代码实现

下面是一个完备的、具有高容错率的 Node.js/TypeScript 监控监听器，集成了 **WebSocket 异常自动检测、指数退避重连（Exponential Backoff Retry）、以及 Slack/Telegram Webhooks 发送器**。

```typescript
import { ethers } from "ethers";
import axios from "axios";

// 1. 配置项 (强烈建议从环境变量读取)
const WSS_PROVIDER_URL = process.env.WSS_PROVIDER_URL || "wss://eth-mainnet.g.alchemy.com/v2/your_key";
const CONTRACT_ADDRESS = "0x6B175474E89094C44Da98b954EedeAC495271d0F"; // 监控的目标合约 (例如 DAI)
const SLACK_WEBHOOK_URL = process.env.SLACK_WEBHOOK_URL || "https://hooks.slack.com/services/your/slack/webhook";

// 极简 ABI：用于提取事件
const MONITOR_ABI = [
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];

class SafeEventMonitor {
  private provider!: ethers.WebSocketProvider;
  private contract!: ethers.Contract;
  private reconnectDelay = 1000; // 初始重连延迟：1 秒
  private readonly maxReconnectDelay = 60000; // 最大延迟：60 秒

  constructor() {
    this.init();
  }

  /**
   * @notice 初始化并建立 WebSocket 连接
   */
  private init() {
    console.log(` [INIT] Connecting to WebSocket RPC: ${WSS_PROVIDER_URL}...`);
    
    // 1. 实例化 WebSocketProvider
    this.provider = new ethers.WebSocketProvider(WSS_PROVIDER_URL);
    this.contract = new ethers.Contract(CONTRACT_ADDRESS, MONITOR_ABI, this.provider);

    // 2. 绑定底层 Socket 关闭与错误事件监听
    this.bindSocketEvents();

    // 3. 开启链上事件订阅监听
    this.startListening();
  }

  /**
   * @notice 绑定底层 Socket 的生命周期事件
   */
  private bindSocketEvents() {
    // 获取底层 WebSocket 连接对象
    const websocket = this.provider.websocket;

    websocket.addEventListener("close", (event: any) => {
      console.warn(` [SOCKET CLOSED] Code: ${event.code}, Reason: ${event.reason || "No reason"}`);
      this.triggerReconnect();
    });

    websocket.addEventListener("error", (error: any) => {
      console.error(" [SOCKET ERROR] WebSocket error caught:", error.message || error);
      // 通常 error 后会自动触发 close，在此记录日志即可
    });
  }

  /**
   * @notice 核心订阅监听区域
   */
  private async startListening() {
    try {
      console.log(` [LISTENING] Successfully subscribed to events on contract: ${CONTRACT_ADDRESS}`);
      
      // 充值重试延迟（一旦成功连接，立即将指数重连延迟重置为 1秒）
      this.reconnectDelay = 1000;

      // 监听 Transfer 事件
      this.contract.on("Transfer", async (from: string, to: string, value: bigint, event: any) => {
        const parsedValue = ethers.formatEther(value);
        
        // 核心过滤：如果单笔转账大于 1,000,000 DAI，立刻拉响警报
        if (value >= ethers.parseEther("1000000")) {
          const alertMessage = `🚨 [ALERTER] Large Transfer Detected!\nContract: ${CONTRACT_ADDRESS}\nFrom: ${from}\nTo: ${to}\nAmount: ${parsedValue} DAI\nTx Hash: ${event.log.transactionHash}`;
          console.log(alertMessage);
          
          // 发送告警至 Slack
          await this.sendSlackAlert(alertMessage);
        }
      });

    } catch (err: any) {
      console.error(" [ERROR] Failed to set up contract.on listener:", err.message);
      this.triggerReconnect();
    }
  }

  /**
   * @notice 指数退避重连机制 (Exponential Backoff)
   */
  private triggerReconnect() {
    // 1. 彻底清除当前的订阅，释放连接
    try {
      this.contract.removeAllListeners();
      this.provider.destroy();
    } catch (e) {}

    console.log(` [RECONNECT] Scheduling reconnection in ${this.reconnectDelay / 1000} seconds...`);
    
    // 2. 延时触发
    setTimeout(() => {
      // 实施指数级延迟翻倍：1s -> 2s -> 4s -> 8s ... 最大 60s
      this.reconnectDelay = Math.min(this.reconnectDelay * 2, this.maxReconnectDelay);
      this.init();
    }, this.reconnectDelay);
  }

  /**
   * @notice 发送 Slack 飞书告警
   */
  private async sendSlackAlert(text: string) {
    if (SLACK_WEBHOOK_URL.includes("your/slack/webhook")) {
      return; // 未配置真实的 webhook 则静默跳过
    }
    try {
      await axios.post(SLACK_WEBHOOK_URL, { text });
    } catch (err: any) {
      console.error(" [ALERTER FAIL] Failed to send webhook alert:", err.message);
    }
  }
}

// 启动监听
if (require.main === module) {
  new SafeEventMonitor();
}
```

---

## 4. 主动安全防卫（Active Defense）集成路线

当你的事件监听器在生产环境捕获到高风险信号时，你可以根据协议特性触发自动化**主动防御（Active Defense）**：

1. **抢跑熔断 (Front-running Pausable)**：
   - 如果发现黑客正试图通过多次交互漏洞盗取资金（且该交易正在内存池排队中），你的监听程序应立即调用备份多签或 Guardian 私钥，以极高的 Gas 费广播一笔抢跑交易（`pause()`），在黑客交易打包前切断协议。
2. **多源校对（Multi-source Check）**：
   - 警报发出后，链下系统自动请求 Chainlink 价格、DEX 流动性价格进行三次校对。如果偏差超过阈值，证明价格发生脱锚操纵，立即自动触发暂停。
3. **资金隔离路由（Vault Isolation）**：
   - 实时监听大额出金。一旦满足预设的安全阈值警告，将后续的所有划款接口一律重定向至需要 24 小时延迟并由多签最终确认的延时保险箱（Time-locked Vault）中，将损失控制在最小范围。
