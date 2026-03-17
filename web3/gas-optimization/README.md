# Gas 优化手册 - 面试笔记索引

> **系列出处**: [Gas 优化手册 | 登链社区](https://learnblockchain.cn/column/article-series/gas-optimization)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 📚 系列概述

本系列整理自登链社区的 Gas 优化手册，涵盖 Solidity 智能合约 gas 优化的完整知识体系，适合面试前快速复习。

**共 11 篇文章**，按学习顺序排列：

---

## 📖 文章目录

| 序号 | 标题 | 原文链接 | 面试笔记 |
|:----:|------|----------|----------|
| 01 | Gas 优化基础知识 | [原文](https://learnblockchain.cn/article/22691) | [笔记](./00-gas-basics-interview-notes.md) |
| 02 | Gas 评估方法 | [原文](https://learnblockchain.cn/article/22706) | [笔记](./01-gas-evaluation-interview-notes.md) |
| 03 | 存储优化 | [原文](https://learnblockchain.cn/article/22692) | [笔记](./02-storage-optimization-interview-notes.md) |
| 04 | 部署优化 | [原文](https://learnblockchain.cn/article/22693) | [笔记](./03-deployment-optimization-interview-notes.md) |
| 05 | 跨合约调用优化 | [原文](https://learnblockchain.cn/article/22694) | [笔记](./04-cross-contract-optimization-interview-notes.md) |
| 06 | 架构层面优化 | [原文](https://learnblockchain.cn/article/22695) | [笔记](./05-architecture-optimization-interview-notes.md) |
| 07 | Calldata 优化 | [原文](https://learnblockchain.cn/article/22696) | [笔记](./06-calldata-optimization-interview-notes.md) |
| 08 | 内联汇编优化 | [原文](https://learnblockchain.cn/article/22697) | [笔记](./07-assembly-optimization-interview-notes.md) |
| 09 | 代码模式优化 | [原文](https://learnblockchain.cn/article/22698) | [笔记](./08-code-pattern-optimization-interview-notes.md) |
| 10 | 极端优化技巧 | [原文](https://learnblockchain.cn/article/22699) | [笔记](./09-extreme-optimization-interview-notes.md) |
| 11 | 已弃用的优化 | [原文](https://learnblockchain.cn/article/22700) | [笔记](./10-deprecated-optimization-interview-notes.md) |

---

## 🎯 面试重点速查

### 高频考点 ⭐⭐⭐

1. **EIP-1559 费用机制** - Base Fee + Priority Fee
2. **存储优化** - 变量打包、slot 优化、冷热存储
3. **gasleft() 测量** - 注意事项和局限性
4. **循环优化** - unchecked、++i vs i++

### 中频考点 ⭐⭐

1. **Gas Snapshots** - CI/CD 集成
2. **跨合约调用** - delegatecall vs call
3. **Calldata vs Memory** - 选择策略
4. **部署优化** - 合约大小控制

### 了解即可 ⭐

1. 内联汇编优化
2. 极端优化技巧
3. 已弃用的优化方法

---

## 📋 快速复习清单

- [ ] Gas 基础概念和 EIP-1559
- [ ] Gas 评估三种方法（Report/Snapshot/gasleft）
- [ ] 存储优化（变量打包、slot）
- [ ] 循环优化技巧
- [ ] 跨合约调用成本
- [ ] Calldata vs Memory
- [ ] 常见面试题练习

---

## 🔗 相关资源

- [登链社区](https://learnblockchain.cn/)
- [Foundry 官方文档](https://book.getfoundry.sh/)
- [Solidity 官方文档](https://docs.soliditylang.org/)

---

*整理 by 小鹅斯基 🦆*
