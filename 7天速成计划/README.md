# DApp 前端面试 7 天速成计划

> 目标：快速备战 Web3/DApp 前端面试，掌握高频考点

---

## 时间分配

```
Web3/DApp：70%（约 5 小时）
传统前端：30%（约 2 小时）
总计：每天 7 小时
```

---

## 7 天概览

| Day | 主题 | 核心目标 | 资料位置 |
|-----|------|----------|----------|
| [Day 1](./Day-01-钱包连接.md) | 钱包连接 + window.ethereum | 掌握 MetaMask 连接流程，能手写连接代码 | [dapp前端.md](../Interview-Wallet/进阶/dapp前端.md) |
| [Day 2](./Day-02-ethers核心操作.md) | ethers.js 核心操作 | Provider/Signer/Contract，合约读写 | [ether.js.md](../Interview-Front/dapp/ether.js.md) |
| [Day 3](./Day-03-Gas与交易流程.md) | Gas + 交易流程 | Gas 估算、交易哈希、模拟调用 | 真题 1/2/4 题 |
| [Day 4](./Day-04-事件监听与批量调用.md) | 事件监听 + 批量调用 | Transfer 事件、Multicall | 真题 5/8/9/10 题 |
| [Day 5](./Day-05-wagmi与多链.md) | wagmi + 多链支持 | wagmi Hooks、多链配置、DEX 路由 | [wagmi.md](../Interview-Front/dapp/wagmi.md) |
| [Day 6](./Day-06-前端基础补漏.md) | 传统前端补漏 | JS 闭包/原型、React Hooks/性能优化 | JS/React PDF |
| [Day 7](./Day-07-真题冲刺.md) | 真题刷题 + 模拟面试 | 12 道必考题口头演练 | 全部真题 |

---

## 学习策略

1. **按顺序学习**：Day 1-7 有依赖关系，建议按顺序
2. **先看真题**：每天开始前，先看对应真题，知道考什么
3. **必须手写**：光看不练，面试必挂
4. **完成自测**：每天学完做自测题

---

## 进度跟踪

- [ ] [Day 1: 钱包连接](./Day-01-钱包连接.md)
- [ ] [Day 2: ethers.js 核心操作](./Day-02-ethers核心操作.md)
- [ ] [Day 3: Gas 与交易流程](./Day-03-Gas与交易流程.md)
- [ ] [Day 4: 事件监听与批量调用](./Day-04-事件监听与批量调用.md)
- [ ] [Day 5: wagmi 与多链](./Day-05-wagmi与多链.md)
- [ ] [Day 6: 前端基础补漏](./Day-06-前端基础补漏.md)
- [ ] [Day 7: 真题冲刺](./Day-07-真题冲刺.md)

---

## 必背知识点速查

### P0 必会（不会就挂）

```javascript
// 1. 钱包连接
const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });

// 2. 事件监听
window.ethereum.on('accountsChanged', handler);
window.ethereum.on('chainChanged', handler);

// 3. 估算 Gas
const gas = await contract.estimateGas.transfer(to, amount);

// 4. 发送交易
const tx = await contract.transfer(to, amount);
const receipt = await tx.wait();

// 5. 监听事件
contract.on('Transfer', (from, to, value) => { /* ... */ });

// 6. ERC20 Transfer 事件
// event Transfer(address indexed from, address indexed to, uint256 value);

// 7. 读取余额
const balance = await token.balanceOf(address);
```

### P1 加分项

- [ ] Multicall 批量调用
- [ ] DEX 路由原理（聚合器、多跳）
- [ ] EIP-712 类型化数据签名
- [ ] RainbowKit 配置和使用
- [ ] React 性能优化（bailout、eagerState）

---

## 资料优先级

| 优先级 | 资料 | 位置 | 说明 |
|--------|------|------|------|
| ⭐⭐⭐ | dapp前端.md | Interview-Wallet/进阶/ | 直接考，代码题最多 |
| ⭐⭐⭐ | 前端(web3含量比较高).md | Interview-Front/面试真题/ | 12 道高频题 |
| ⭐⭐ | ether.js.md | Interview-Front/dapp/ | 技术栈必问 |
| ⭐⭐ | wagmi.md | Interview-Front/dapp/ | 技术栈必问 |
| ⭐ | JavaScript题目.pdf | Interview-Front/前端面试题/ | 基础补漏 |
| ⭐ | React生态题目.pdf | Interview-Front/前端面试题/ | 基础补漏 |

---

## 学习提醒

1. **先看真题，再看资料** — 带着问题学效率最高
2. **必须手写代码** — 光看不练，面试必挂
3. **练习表达** — 能说清楚比会写更重要
4. **关注错误处理** — try-catch、用户拒绝、网络错误等
5. **注意用户体验** — 加载状态、交易确认提示等

---

## 每日检查清单

### 学习前
- [ ] 确认今天要学的内容
- [ ] 准备好代码编辑器

### 学习中
- [ ] 按清单逐项完成
- [ ] 手写所有代码示例
- [ ] 记录不理解的点

### 学习后
- [ ] 复习今天的内容
- [ ] 完成自测题
- [ ] 准备明天的学习

---

加油！7 天后，你一定会有质的飞跃！
