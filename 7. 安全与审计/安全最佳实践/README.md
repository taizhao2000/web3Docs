# 安全最佳实践

## 编码规范

### 检查-生效-交互（Checks-Effects-Interactions）

```solidity
function transfer(address to, uint256 amount) external {
    // 1. 检查
    require(balances[msg.sender] >= amount, "Insufficient balance");
    require(to != address(0), "Invalid recipient");
    
    // 2. 生效（更新状态）
    balances[msg.sender] -= amount;
    balances[to] += amount;
    
    // 3. 交互（外部调用）
    emit Transfer(msg.sender, to, amount);
}
```

### 使用 OpenZeppelin 合约

| 组件 | 用途 |
|------|------|
| Ownable | 所有权管理 |
| Pausable | 紧急暂停 |
| ReentrancyGuard | 重入保护 |
| AccessControl | 角色权限 |
| SafeERC20 | 安全 ERC20 操作 |

## 部署前检查清单

- [ ] 所有外部调用是否安全
- [ ] 访问控制是否完善
- [ ] 是否有重入风险
- [ ] 整数运算是否安全
- [ ] 预言机是否可靠
- [ ] 是否有紧急暂停机制
- [ ] 权限是否最小化
- [ ] 是否经过代码审计
- [ ] 测试覆盖率是否足够
- [ ] 是否有 Bug Bounty 计划

## 审计流程

1. **内部审查**：团队代码评审
2. **自动化扫描**：Slither、Mythril
3. **外部审计**：专业安全公司审计
4. **形式化验证**：数学证明关键逻辑
5. **Bug Bounty**：社区安全研究
6. **主网监控**：部署后持续监控
