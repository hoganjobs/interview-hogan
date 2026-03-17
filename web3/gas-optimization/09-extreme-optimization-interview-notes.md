# Gas 极端优化技巧（竞赛用）- 面试笔记

> **出处**: [Gas 优化手册（九）| 登链社区](https://learnblockchain.cn/article/22699)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## ⚠️ 警告

> **这些技巧仅适用于 Gas 优化竞赛！在生产环境中极不推荐，或应极度谨慎使用。**

---

## 极端优化技巧

### 1. 用 gasprice() 或 msg.value 传递信息

**原理**: 传递参数至少 128 gas，但设置 gasprice/msg.value 免费

**问题**:
- msg.value 需要真实 ETH
- gasprice 太低交易无法进行

**结论**: 仅限竞赛，生产不可行

### 2. 操纵环境变量

操纵 `coinbase()` 或 `block.number` 作为侧信道修改合约行为

**问题**: 生产环境不可行

### 3. 用 gasleft() 分支决策

```solidity
// 随着执行 gas 消耗，可用于分支决策
if (gasleft() > threshold) {
    // 执行 A
} else {
    // 执行 B
}
```

**原理**: gasleft() 减量免费

### 4. send() 不检查成功与否

```solidity
// ❌ 危险：忽略返回值
addr.send(1 ether);  // 不检查返回值

// vs transfer 会自动 revert
addr.transfer(1 ether);
```

**问题**: 极其糟糕的实践，编译器不阻止

### 5. 所有函数设为 payable

```solidity
// 竞赛中可能省少量 gas
// 避免检查 msg.value 是否为非零
function dangerous() public payable {
    // 允许意外的 ETH 转入
}
```

**问题**: 允许意外状态变化

**合理使用**: 构造函数和管理员函数可以设为 payable

### 6. 外部库跳转优化

将跳转目标作为 calldata 参数，函数选择器减少到 1 字节，避免跳转表

**问题**: 非常不安全！

### 7. 附加字节码创建子程序

将计算密集型算法（如哈希函数）用原始字节码编写，附加到合约末尾

**案例**: Tornado Cash 将 MiMC 哈希函数直接写成字节码

---

## 面试要点

### Q: 这些技巧为什么不能用于生产？

**回答**:
- 安全风险极高
- 可读性差，难以审计
- 依赖特定环境假设
- 可能引入意外行为

### Q: 有哪些是可以在生产中使用的？

**回答**:
- 构造函数设为 payable（合理）
- 管理员函数设为 payable（部署者知道自己在做什么）

---

*整理人: 小鹅斯基 🦆*
