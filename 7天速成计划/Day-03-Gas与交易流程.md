# Day 3：Gas + 交易流程

## 学习目标

- [ ] 理解 Gas 估算原理
- [ ] 掌握交易生命周期
- [ ] 理解模拟调用 vs 正常调用

---

## 学习资料

- [ ] 真题：第 1、2、4 题
  - 第 1 题：一笔交易怎么确定需要多少 Gas？
  - 第 2 题：模拟调用和正常调用有什么区别？
  - 第 4 题：交易哈希是怎么得到的？

---

## 学习内容

### 1. Gas 相关概念

| 概念 | 说明 |
|------|------|
| **Gas Limit** | 交易愿意支付的最大 Gas 量 |
| **Gas Used** | 实际消耗的 Gas 量 |
| **Gas Price** | 单位 Gas 的价格（Gwei） |
| **Base Fee** | 网络基础费用（EIP-1559） |
| **Priority Fee** | 优先费用（给矿工的小费） |

### 2. Gas 估算流程

```
1. 本地模拟执行交易
   └── 节点在本地执行，不修改链上状态

2. 计算执行所需的 Gas
   └── 根据操作码计算

3. 返回预估值
   └── 可能与实际略有差异（链上状态可能变化）
```

### 3. 交易哈希生成

```
交易数据
   ↓
RLP 编码
   ↓
keccak256 哈希
   ↓
交易哈希（32 字节）
```

### 4. 模拟调用 vs 正常调用

| 对比项 | 模拟调用 | 正常调用 |
|--------|----------|----------|
| 方法 | `eth_call` / `callStatic` | `eth_sendTransaction` |
| 执行位置 | 本地节点 | 链上 |
| 修改状态 | 否 | 是 |
| 消耗 Gas | 否 | 是 |
| 需要签名 | 否 | 是 |
| 返回值 | 执行结果 | 交易哈希 |
| 用途 | 测试、预览 | 实际执行 |

---

## 必会代码

### 任务1：估算 Gas

```javascript
// 使用 ethers.js
const gasEstimate = await contract.estimateGas.transfer(to, amount);

// 使用原生 API
const gasEstimate = await window.ethereum.request({
    method: 'eth_estimateGas',
    params: [{
        from: account,
        to: contractAddress,
        data: contractInterface.encodeFunctionData('transfer', [to, amount])
    }]
});
```

### 任务2：设置 Gas 参数发送交易

```javascript
const gasEstimate = await contract.estimateGas.transfer(to, amount);
const feeData = await provider.getFeeData();

const tx = await contract.transfer(to, amount, {
    gasLimit: gasEstimate * 12n / 10n,  // 增加 20% 余量
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas
});
```

### 任务3：模拟调用

```javascript
// ethers.js callStatic
try {
    const result = await contract.callStatic.transfer(to, amount);
    console.log('模拟调用成功，返回:', result);
} catch (error) {
    console.log('模拟调用失败:', error.message);
}

// 原生 API
const result = await window.ethereum.request({
    method: 'eth_call',
    params: [{
        to: contractAddress,
        data: contractInterface.encodeFunctionData('transfer', [to, amount])
    }, 'latest']
});
```

### 任务4：获取交易收据

```javascript
// 等待交易确认
const receipt = await tx.wait();

// 收据包含的信息
console.log('状态:', receipt.status);        // 1=成功, 0=失败
console.log('区块号:', receipt.blockNumber);
console.log('Gas 使用:', receipt.gasUsed);
console.log('Gas 费用:', receipt.gasPrice * receipt.gasUsed);
console.log('日志:', receipt.logs);
```

### 任务5：处理 Gas 不足

```javascript
try {
    const tx = await contract.transfer(to, amount);
} catch (error) {
    if (error.message.includes('insufficient funds')) {
        console.log('余额不足');
    } else if (error.message.includes('gas required exceeds')) {
        console.log('Gas 不足，尝试增加 Gas Limit');
        const gasEstimate = await contract.estimateGas.transfer(to, amount);
        const tx = await contract.transfer(to, amount, {
            gasLimit: gasEstimate * 2n
        });
    }
}
```

---

## 真题解答

### Q1: 一笔交易怎么确定需要多少 Gas？

**答**：
1. 使用 `eth_estimateGas` 或 `contract.estimateGas.method(args)` 进行估算
2. 节点在本地模拟执行交易，返回预估值
3. 实际执行时可能因链上状态不同而略有差异

### Q2: 模拟调用和正常调用有什么区别？

**答**：

| 模拟调用 (eth_call) | 正常调用 (sendTransaction) |
|---------------------|---------------------------|
| 本地执行，不修改链上状态 | 链上执行，修改状态 |
| 不消耗 Gas | 消耗 Gas |
| 不需要签名 | 需要用户签名 |
| 返回执行结果 | 返回交易哈希 |
| 用于测试和预览 | 用于实际执行 |

### Q3: 交易哈希是怎么得到的？

**答**：
- 交易哈希是对交易内容进行 RLP 编码并经过 keccak256 哈希后的结果
- 链上唯一标识该笔交易
- 在执行写操作时，钱包返回的交易对象中包含 `hash` 字段

---

## 自测题

- [ ] 能解释模拟调用和正常调用的区别
- [ ] 知道交易哈希是如何生成的
- [ ] 能说明为什么实际 Gas 可能与估算不同
- [ ] 能手写 Gas 估算和交易发送代码

---

## 进度检查

- [ ] 已理解 Gas 估算原理
- [ ] 已手写所有代码示例
- [ ] 已完成自测题

---

## 明天预告

明天学习 **事件监听 + 批量调用**，理解链上事件监听原理、ERC20/721 Transfer 事件、Multicall 批量调用等。
