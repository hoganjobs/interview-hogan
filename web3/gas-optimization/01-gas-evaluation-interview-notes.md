# Gas 评估方法 - 面试笔记

> **出处**: [Gas 优化手册（Gas 评估篇）| 登链社区](https://learnblockchain.cn/article/22706)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 一、为什么需要 Gas 评估？⭐

Gas 评估是优化工作的基础：

| 目的 | 说明 |
|------|------|
| **建立基线** | 了解当前代码的 gas 消耗情况 |
| **识别瓶颈** | 找出最消耗 gas 的操作 |
| **验证优化效果** | 对比优化前后的差异 |
| **防止性能退化** | 在 CI/CD 中监控 gas 消耗变化 |

> **面试要点**: Gas 评估不是一次性工作，应该贯穿整个开发周期

---

## 二、Foundry Gas Report ⭐⭐

### 2.1 基础用法

```bash
forge test --gas-report
```

输出示例：
```
| src/MyToken.sol:MyToken contract |                |       |        |       |         |
|----------------------------------|----------------|-------|--------|-------|---------|
| Deployment Cost                  | Deployment Size|       |        |       |         |
| 500000                           | 2500           |       |        |       |         |
| Function Name                    | min            | avg   | median | max   | # calls |
| transfer                         | 51343          | 51343 | 51343  | 51343 | 1       |
| mint                             | 48000          | 48000 | 48000  | 48000 | 2       |
```

### 2.2 配置 Gas Report

```toml
# foundry.toml
[profile.default]
gas_reports = ["*"]  # 报告所有合约
# gas_reports = ["MyToken", "MyNFT"]  # 只报告特定合约
gas_reports_ignore = ["MockContract", "TestHelpers"]  # 排除某些合约
```

### 2.3 优势

✅ **自动化** - 无需修改合约代码
✅ **统计全面** - 显示 min/avg/median/max 值
✅ **易于对比** - 直观展示所有函数的 gas 消耗

---

## 三、Gas Snapshots ⭐⭐

### 3.1 创建 Snapshot

```bash
forge snapshot
forge snapshot --snap <FILE_NAME>  # 自定义快照文件名
```

生成 `.gas-snapshot` 文件：
```
MyTokenTest:testApproveGas() (gas: 46123)
MyTokenTest:testTransferGas() (gas: 51343)
```

### 3.2 对比 Snapshot

```bash
forge snapshot --diff
```

输出示例：
```
MyTokenTest:testTransferGas() (gas: 51243 (↓ -100 | -0.19%))
MyTokenTest:testApproveGas() (gas: 46123 (no change))
MyTokenTest:testMintGas() (gas: 48150 (↑ +150 | +0.31%))
```

### 3.3 检查 Snapshot 差异

```bash
forge snapshot --check  # 如果 gas 消耗增加超过阈值则失败
```

### 3.4 最佳实践 ⭐

✅ **提交到版本控制** - 将 `.gas-snapshot` 文件提交到 Git
✅ **在 PR 中审查** - 关注 gas 消耗的变化
✅ **设置告警阈值** - 显著的 gas 增加应该引起注意
✅ **为关键函数编写专门测试** - 确保核心功能的 gas 效率

---

## 四、gasleft() 内部测量 ⭐⭐⭐

### 4.1 基本用法

```solidity
function measureGas() public {
    uint256 gasBefore = gasleft();
    
    // 要测量的操作
    uint256 result = expensiveOperation();
    
    uint256 gasAfter = gasleft();
    uint256 gasUsed = gasBefore - gasAfter;
    
    emit GasConsumed("expensiveOperation", gasUsed);
}
```

### 4.2 在测试中对比实现

```solidity
function testCompareImplementations() public {
    // 测量实现 A
    uint256 gasBeforeA = gasleft();
    uint256 resultA = implementationA();
    uint256 gasUsedA = gasBeforeA - gasleft();
    
    // 测量实现 B
    uint256 gasBeforeB = gasleft();
    uint256 resultB = implementationB();
    uint256 gasUsedB = gasBeforeB - gasleft();
    
    // 验证结果相同
    assertEq(resultA, resultB);
    
    // 断言优化版本更省 gas
    assertLt(gasUsedB, gasUsedA, "Implementation B should use less gas");
}
```

### 4.3 注意事项 ⚠️

| 问题 | 说明 |
|------|------|
| **测量开销** | `gasleft()` 本身消耗约 2 gas |
| **编译器优化** | 某些情况下编译器可能会优化掉未使用的代码 |
| **外部调用影响** | 跨合约调用时，gas 的计算可能更复杂 |
| **Context 依赖** | 测量结果可能受调用上下文影响（如存储槽的冷/热状态） |

---

## 五、三种方法对比 ⭐⭐⭐

| 方法 | 优点 | 适用场景 |
|------|------|----------|
| **Forge Gas Report** | 自动化、全面 | 整体性能评估 |
| **Gas Snapshots** | 追踪变化、CI 集成 | 防止性能退化 |
| **gasleft()** | 精确、灵活 | 细粒度测量 |

---

## 六、推荐做法 ⭐

1. **建立基线** - 在优化前先测量当前性能
2. **使用多种工具** - 结合 gas report、snapshot 和 gasleft()
3. **版本控制** - 提交 `.gas-snapshot` 文件
4. **关注关键路径** - 重点优化高频调用的函数
5. **验证优化效果** - 优化后必须再次测量

---

## 七、面试常见问题

### Q1: 如何在 CI/CD 中监控 gas 消耗？

使用 `forge snapshot --check`，如果 gas 消耗增加超过阈值则构建失败。

### Q2: gasleft() 的局限性是什么？

- 本身有约 2 gas 的开销
- 编译器可能优化掉未使用代码
- 跨合约调用计算复杂
- 受调用上下文影响（冷/热存储）

### Q3: 为什么要把 .gas-snapshot 提交到 Git？

可以在 PR 中清晰看到 gas 消耗的变化，防止意外的性能退化。

---

## 八、代码示例：完整评估流程

```solidity
// 待优化的合约
contract TokenV1 {
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        balances[msg.sender] = balances[msg.sender] - amount;
        balances[to] = balances[to] + amount;
    }
}

// 优化后的合约
contract TokenV2 {
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        unchecked {
            balances[msg.sender] -= amount;
            balances[to] += amount;
        }
    }
}
```

```bash
# 完整评估流程
forge test --gas-report --match-contract GasComparisonTest
forge snapshot --match-contract GasComparisonTest
forge test --match-test testGasComparison -vv
```

---

*上一篇: [Gas 优化基础知识](./00-gas-basics-interview-notes.md)*
*下一篇: [存储优化](./02-storage-optimization-interview-notes.md)*
