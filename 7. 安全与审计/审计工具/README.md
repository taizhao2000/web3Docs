# 审计工具

## 静态分析工具

| 工具 | 语言 | 特点 |
|------|------|------|
| Slither | Python | 速度快，检测模式丰富 |
| Securify2 | - | 基于模式匹配 |
| Solhint | Node.js | Lint 规则检查 |
| Mythril | Python | 符号执行分析 |

## 动态分析工具

| 工具 | 说明 |
|------|------|
| Echidna | 属性测试（模糊测试） |
| Harvey | 自动化模糊测试 |
| Medusa | Go 语言模糊测试框架 |

## 形式化验证

| 工具 | 说明 |
|------|------|
| Certora Prover | 基于CVL规约语言的形式化验证 |
| Halmos | Foundry 集成的符号执行工具 |
| KEVM | K 框架的 EVM 形式化验证 |

## 使用示例

### Slither

```bash
# 安装
pip3 install slither-analyzer

# 运行分析
slither .

# 检测特定漏洞
slither . --detect reentrancy-eth
```

### Echidna

```solidity
// 属性测试示例
function echidna_test_balance() public view returns (bool) {
    return token.balanceOf(address(this)) >= 0;
}
```

```bash
echidna-test contracts/MyContract.sol --contract MyContract
```

## 专业审计公司

- Trail of Bits
- OpenZeppelin
- ConsenSys Diligence
- Certik
- Quantstamp
