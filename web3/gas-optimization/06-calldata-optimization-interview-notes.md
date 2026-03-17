# Gas Calldata 优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（六）| 登链社区](https://learnblockchain.cn/article/22696)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 一、Calldata Gas 成本

| 字节类型 | Gas 成本 |
|---------|---------|
| 零字节 (0x00) | **4 gas** |
| 非零字节 | **16 gas** |

> **关键**: 零字节比非零字节便宜 4 倍！

---

## 二、优化技巧

### 1. 使用虚荣地址（前导零地址）⭐⭐

```solidity
// ✅ OpenSea Seaport 合约地址
// 0x00000000000000ADc04C56Bf30aC9d3c0aAF14dC
// 前面有很多零，作为参数传递时省 gas
```

**原理**: 地址作为函数参数时，零字节多的地址 calldata 成本更低

**注意**:
- 直接调用该地址不省 gas
- 仅当地址作为参数传递时有效
- 智能合约用 CREATE2 生成虚荣地址是安全的（无私钥风险）

### 2. 避免 calldata 中使用有符号整数 ⭐

```solidity
// ❌ 小负数在 calldata 中大部分是非零
// -1 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
int8 x = -1;  // 昂贵

// ✅ 使用无符号整数
uint8 x = 255;  // 便宜
```

**原理**: Solidity 用二进制补码表示有符号数，小负数大部分位为 1（非零字节）

### 3. Calldata 比 Memory 更便宜 ⭐⭐

```solidity
// ✅ 使用 calldata（只读数据）
function getData(bytes calldata data) public pure returns (bytes memory) {
    return data;
}

// ❌ 使用 memory（需要复制）
function getData(bytes memory data) public pure returns (bytes memory) {
    return data;
}
```

**原则**: 只在需要修改数据时使用 memory，否则用 calldata

---

## 三、Cancun 升级后的变化

### EIP-4844 对 L2 的影响

| 升级前 | 升级后 (Blob 交易) |
|-------|------------------|
| 零字节 4 gas，非零 16 gas | Blob 不区分零/非零字节 |
| Calldata 优化重要 | **Calldata 优化重要性降低** |
| - | Blob 费用降低 80-90% |

**L2 新优化优先级**:
- ✅ 高优先级：存储优化、计算优化、架构优化
- 🔻 低优先级：Calldata 零字节、虚荣地址

**仍需优化的场景**:
- L1 主网合约
- L1↔L2 跨链消息
- 极大数据量操作

---

## 四、面试常见问题

### Q1: 为什么零字节在 calldata 中更便宜？

**回答**: EVM 对零字节收取 4 gas，非零字节 16 gas。这是设计决策，零字节更容易压缩。

### Q2: 什么时候用 calldata vs memory？

**回答**:
- 只读数据 → calldata（省 gas）
- 需要修改 → memory

### Q3: Cancun 升级对 calldata 优化有什么影响？

**回答**:
- L2 使用 Blob 交易，不再区分零/非零字节
- Calldata 优化在 L2 上重要性降低
- L1 主网仍需关注 calldata 优化

---

*整理人: 小鹅斯基 🦆*
