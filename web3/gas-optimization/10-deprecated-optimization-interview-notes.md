# Gas 过时优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（十）| 登链社区](https://learnblockchain.cn/article/22700)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 概述

> **由于 Solidity 编译器改进、EVM 升级或生态变化，某些优化技巧已过时或重要性大幅降低。**

---

## 一、编译器改进导致的过时技巧

### 1. external vs public ⛔

| 说法 | 现状 |
|-----|------|
| external 比 public 省 gas | ~Solidity 0.8.x 后效果不明显 |

**结论**: 为清晰起见仍可使用 external，但 gas 影响可忽略

### 2. != 0 vs > 0 ⛔

| 说法 | 现状 |
|-----|------|
| != 0 比 > 0 更便宜 | Solidity 0.8.12+ 不再成立 |

**结论**: 编译器已自动优化

### 3. 编译器自动优化示例

```solidity
// 0.8.20+: PUSH0 优化
function getZero() public pure returns (uint256) {
    return 0;  // 自动使用 PUSH0 操作码
}

// 0.8.22+: 部分循环优化
// 编译器可能识别安全循环并优化

// 2 的幂乘除法
uint256 x = y * 2;  // 可能优化为 y << 1
uint256 z = y / 4;  // 可能优化为 y >> 2
```

**建议**: 仍建议显式使用 `unchecked { ++i }` 和位移

---

## 二、EVM 升级导致的过时技巧

### 4. 跨交易 SELFDESTRUCT ⛔⛔⛔

**Cancun 升级 (EIP-6780) 后的变化**:

| 场景 | 旧行为 | 新行为 |
|-----|-------|-------|
| 同一交易内创建并销毁 | 删除代码、清除存储、退 gas | ✅ 保留完整功能 |
| 已存在合约调用 SELFDESTRUCT | 删除代码、清除存储、退 gas | ❌ 仅转移 ETH |

**受影响的模式**:
```solidity
// ❌ 不再有效：跨交易销毁
contract OldPattern {
    function cleanup() external {
        selfdestruct(payable(msg.sender));
        // 代码和存储不会被删除！
    }
}

// ❌ 不再有效：CREATE2 + SELFDESTRUCT 重新部署
// 无法在同一地址重新部署
```

**替代方案**:
- 代理模式（UUPS / 透明代理）
- 最小克隆（EIP-1167）
- 状态标志停用合约

### 5. L2 上极致优化 Calldata 零字节 ⛔

**EIP-4844 的影响**:

| 升级前 | 升级后 |
|-------|-------|
| 零字节 4 gas，非零 16 gas | Blob 不区分零/非零 |
| Calldata 优化重要 | **重要性大幅降低** |

**L2 新优先级**:
- ✅ 高优先级：存储优化、计算优化、架构优化
- 🔻 低优先级：Calldata 零字节、虚荣地址、函数名压缩

**仍需优化的场景**: L1 主网、L1↔L2 跨链消息

### 6. SSTORE2 用于临时数据 ⛔

**对比**:

| 方案 | 成本 | 持久性 |
|-----|------|-------|
| SSTORE2 | ~200 gas/字节 | 永久 |
| 临时存储 (EIP-1153) | 100 gas | 交易结束后清零 |

```solidity
// ❌ 旧方案：SSTORE2 存临时数据
address pointer = SSTORE2.write(abi.encode(items));

// ✅ 新方案：临时存储
mapping(uint256 => bytes) private transient tempData;
tempData[i] = items[i];  // TSTORE: 100 gas
// 交易结束自动清零
```

**何时仍用 SSTORE2**:
- 需要永久保存的大量数据（> 24 KB）
- 需要跨交易访问的数据
- 链上 NFT 元数据存储

---

## 三、总结

### 过时技巧分类

| 原因 | 过时技巧 |
|-----|---------|
| 编译器改进 | external vs public、!= 0 vs > 0、某些循环/位移优化 |
| EVM 升级 | 跨交易 SELFDESTRUCT、L2 calldata 零字节、SSTORE2 临时数据 |

### 建议

- ✅ 使用 Solidity 0.8.24+ 和 Cancun EVM
- ✅ 定期审查优化策略是否仍有效
- ✅ 始终基准测试
- ✅ 关注以太坊升级和编译器更新
- ✅ 使用现代工具：Solady、临时存储、Blob 交易

---

## 四、面试要点

### Q1: SELFDESTRUCT 在 Cancun 后有什么变化？

**回答**:
- 同一交易内创建并销毁：保留完整功能
- 已存在合约调用：仅转移 ETH，不删除代码/存储
- 不再退还 gas

### Q2: 为什么 L2 上 calldata 零字节优化不再重要？

**回答**:
- EIP-4844 引入 Blob 交易
- Blob 不区分零/非零字节
- 费用降低 80-90%

### Q3: 临时存储 (Transient Storage) 有什么优势？

**回答**:
- 仅 100 gas (vs SSTORE 22,100)
- 交易结束后自动清零
- 适合重入锁、闪电贷、批量操作中间状态

---

*整理人: 小鹅斯基 🦆*
