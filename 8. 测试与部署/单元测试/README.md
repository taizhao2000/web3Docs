# 单元测试

## 概述

单元测试是智能合约开发的基础保障，确保每个函数按预期工作。

## Hardhat 测试示例

```javascript
const { expect } = require('chai');
const { ethers } = require('hardhat');

describe('Token', function () {
  let Token, token, owner, addr1, addr2;
  
  beforeEach(async function () {
    Token = await ethers.getContractFactory('Token');
    token = await Token.deploy('MyToken', 'MTK', 1000000);
    [owner, addr1, addr2] = await ethers.getSigners();
  });
  
  describe('Deployment', function () {
    it('应设置正确的名称和符号', async function () {
      expect(await token.name()).to.equal('MyToken');
      expect(await token.symbol()).to.equal('MTK');
    });
    
    it('应将总供应量分配给部署者', async function () {
      expect(await token.balanceOf(owner.address)).to.equal(1000000);
    });
  });
  
  describe('Transfers', function () {
    it('应正确转账', async function () {
      await token.transfer(addr1.address, 100);
      expect(await token.balanceOf(addr1.address)).to.equal(100);
    });
    
    it('余额不足时应回滚', async function () {
      await expect(
        token.connect(addr1).transfer(addr2.address, 100)
      ).to.be.revertedWith('Insufficient balance');
    });
  });
});
```

## Foundry 测试示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "../src/Token.sol";

contract TokenTest is Test {
    Token token;
    address owner = address(this);
    address alice = makeAddr("alice");
    
    function setUp() public {
        token = new Token("MyToken", "MTK", 1000000);
    }
    
    function testDeployment() public {
        assertEq(token.name(), "MyToken");
        assertEq(token.balanceOf(owner), 1000000);
    }
    
    function testTransfer() public {
        token.transfer(alice, 100);
        assertEq(token.balanceOf(alice), 100);
    }
    
    function testRevertInsufficientBalance() public {
        vm.prank(alice);
        vm.expectRevert("Insufficient balance");
        token.transfer(owner, 100);
    }
}
```

## 测试覆盖率目标

| 指标 | 最低 | 推荐 |
|------|------|------|
| 行覆盖率 | 80% | 95%+ |
| 分支覆盖率 | 70% | 90%+ |
| 函数覆盖率 | 90% | 100% |
