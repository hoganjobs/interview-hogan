# Gas 跨合约调用优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（四）| 登链社区](https://learnblockchain.cn/article/22694)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 一、跨合约调用成本

每次跨合约调用都有**固定的 Gas 开销**，这是 EVM 的固有限制。

> **面试要点**: 跨合约调用不可避免地消耗额外 Gas，需要权衡架构和性能

---

## 二、6 条跨合约调用优化技巧

### 1. 使用转账 Hook 处理代币 ⭐⭐⭐

#### 传统授权流程（低效）

```
msg.sender → approve(A, tokenB)
msg.sender → 调用合约A
合约A → 调用 tokenB.transferFrom()
tokenB → 执行转账
tokenB → 调用 A.onTokenReceived()
```

**问题**: 多次跨合约调用，Gas 成本高

#### 优化后：转账 Hook（高效）

```
msg.sender → 调用 tokenB.transferAndCall()
tokenB → 执行转账
tokenB → 调用 A.tokenReceived()  // Hook
```

**好处**: 减少一次跨合约调用

#### 支持转账 Hook 的代币标准

| 标准 | Hook 方法 | 说明 |
|-----|----------|------|
| ERC1155 | `onERC1155Received` | 所有代币都支持 |
| ERC721 | `onERC721Received` | `safeTransfer` / `safeMint` |
| ERC1363 | `onTransferReceived` | `transferAndCall` |
| ERC777 | `tokensReceived` | 已弃用，建议用 ERC1363 |

```solidity
// 使用 data 字段传递参数
tokenB.transferAndCall(address(contractA), amount, abi.encode(param1, param2));

// 合约 A 解析参数
function onTransferReceived(address, address, uint256, bytes calldata data) 
    external returns (bytes4) {
    (uint256 param1, uint256 param2) = abi.decode(data, (uint256, uint256));
    // 处理逻辑
}
```

### 2. 纯 ETH 转账用 receive ⭐⭐

```solidity
// ✅ 使用 receive 钩子
contract AddLiquidity {
    receive() external payable {
        IWETH(weth).deposit{msg.value}();
        AAVE.deposit(weth, msg.value, msg.sender, REFERRAL_CODE);
    }
}

// fallback 可接收字节数据，用 abi.decode 解析
fallback() external payable {
    (address target, uint256 amount) = abi.decode(msg.data, (address, uint256));
}
```

**好处**: 跳过函数调用开销，直接通过转账触发

### 3. 预热存储槽 (ERC-2930) ⭐⭐

**访问列表交易**: 交易开始前声明将访问的地址和存储槽

```javascript
// ethers.js 使用访问列表
const tx = await contract.myFunction({
    accessList: [
        {
            address: "0x...",  // 目标合约地址
            storageKeys: [
                "0x...",  // 存储槽 1
                "0x...",  // 存储槽 2
            ]
        }
    ]
});
```

**适用场景**:
- 能准确预测访问目标
- 多次或深层状态访问的复杂交易
- 冷访问频繁的场景

**注意**:
- access list 本身增加 calldata 成本
- 对 proxy/clone 的 delegatecall 通常无显著收益
- **不应默认使用**，需要测试对比

### 4. 缓存外部调用结果 ⭐⭐

```solidity
// ❌ 多次调用 Chainlink
function calculate() external view returns (uint256) {
    uint256 price1 = oracle.getPrice();  // 外部调用
    uint256 price2 = oracle.getPrice();  // 重复调用
    return price1 + price2;
}

// ✅ 缓存到内存
function calculate() external view returns (uint256) {
    uint256 price = oracle.getPrice();  // 只调用一次
    return price * 2;
}
```

**原理**: 避免重复的昂贵外部调用

### 5. 实现 Multicall 批量处理 ⭐⭐⭐

```solidity
// Multicall 模式
contract Multicall {
    function multicall(bytes[] calldata data) external returns (bytes[] memory) {
        bytes[] memory results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            (bool success, bytes memory result) = address(this).delegatecall(data[i]);
            require(success, "call failed");
            results[i] = result;
        }
        return results;
    }
}
```

**好处**:
- 多个操作合并为一次交易
- 节省交易基础成本 (21,000 gas)
- 原子性执行

**案例**: Uniswap Router、Compound Bulker

### 6. 单体合约避免跨合约调用 ⭐

**权衡考量**:

| 方案 | 优点 | 缺点 |
|-----|------|------|
| 单体合约 | 无跨合约调用开销 | 难维护、难审计、可能超 24KB |
| 模块化合约 | 架构清晰、安全隔离 | 跨合约调用 Gas 开销 |

> **原则**: 极端性能敏感场景考虑单体，但架构合理性和安全性通常更重要

---

## 三、DApp 前端相关要点

### 3.1 前端实现 Multicall

```javascript
// 使用 multicall 合并多个调用
const calls = [
    contract.interface.encodeFunctionData("approve", [spender, amount]),
    contract.interface.encodeFunctionData("transfer", [recipient, amount]),
];

await contract.multicall(calls);
```

### 3.2 转账 Hook 与前端交互

```javascript
// ERC721 safeTransferFrom
await nftContract.safeTransferFrom(
    fromAddress,
    toContractAddress,
    tokenId,
    abi.encode(param)  // data 字段传递参数
);
```

### 3.3 访问列表优化

```javascript
// 对于复杂交易，考虑预热
const accessList = await provider.buildAccessList(txRequest);
await contract.myFunction({ accessList });
```

---

## 四、面试常见问题

### Q1: 如何减少跨合约调用的 Gas 成本？

**回答**:
1. 使用转账 Hook（ERC1363、ERC1155）减少调用次数
2. 缓存外部调用结果，避免重复调用
3. 使用 Multicall 批量处理多个操作
4. 极端场景考虑单体合约设计

### Q2: 什么是 Multicall？有什么好处？

**回答**:
- 将多个合约调用合并为一次交易
- 节省交易基础成本 (21,000 gas)
- 保证原子性执行
- 案例：Uniswap Router

### Q3: ERC-2930 访问列表有什么用？

**回答**:
- 交易前声明将访问的地址和存储槽
- 预热冷访问，减少 Gas
- 适用于复杂交易、多次状态访问
- 需要测试对比，不应默认使用

---

## 五、快速记忆

```
跨合约调用有成本，
转账 Hook 省一步，
receive 收 ETH 快，
缓存结果别重复，
multicall 批量做，
单体合约极端用。
```

---

*整理人: 小鹅斯基 🦆*
*用于: DApp 前端开发面试准备*
