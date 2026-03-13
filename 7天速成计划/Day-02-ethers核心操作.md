# Day 2：ethers.js 核心操作

## 学习目标

- [ ] 理解 Provider、Signer、Contract 的区别
- [ ] 掌握合约读写操作
- [ ] 会处理交易状态确认

---

## 学习资料

- [ ] `Interview-Front/dapp/ether.js.md`
- [ ] 真题：第 3 题「发送交易之后，如何确定交易的状态？」

---

## 学习内容

### 1. ethers.js 核心概念

| 概念 | 说明 | 用途 |
|------|------|------|
| **Provider** | 连接区块链节点的抽象 | 只读操作，查询链上数据 |
| **Signer** | 签名交易的账户 | 写操作，发送交易 |
| **Contract** | 合约交互接口 | 调用合约方法 |

**区别**：
- Provider 只读，不需要私钥
- Signer 可写，需要私钥签名
- Contract 可以用 Provider 或 Signer 创建，取决于是否需要签名

### 2. 合约调用方式

| 方式 | 说明 | 是否上链 | 是否消耗 Gas |
|------|------|----------|-------------|
| `contract.method()` | 读取/写入 | 自动判断 | 读取不消耗，写入消耗 |
| `callStatic.method()` | 模拟调用 | 否 | 否 |
| `estimateGas.method()` | 估算 Gas | 否 | 否 |

---

## 必会代码

### 任务1：创建 Provider 和 Signer

```javascript
// 创建 Provider（只读）
const provider = new ethers.BrowserProvider(window.ethereum);

// 创建 Signer（可写，需要签名）
const signer = await provider.getSigner();
// 或指定账户
const signer = await provider.getSigner(address);

// 获取地址
const address = await signer.getAddress();
```

### 任务2：创建合约实例

```javascript
// 只读合约（用于查询）
const contractRead = new ethers.Contract(
    address,
    abi,
    provider  // 用 Provider
);

// 可写合约（用于交易）
const contractWrite = new ethers.Contract(
    address,
    abi,
    signer   // 用 Signer
);
```

### 任务3：读取数据

```javascript
// 读取余额
const balance = await contract.balanceOf(address);

// 读取多个数据
const [name, symbol, decimals] = await Promise.all([
    contract.name(),
    contract.symbol(),
    contract.decimals()
]);
```

### 任务4：发送交易

```javascript
// 发送交易
const tx = await contract.transfer(to, amount);

// 等待交易确认
const receipt = await tx.wait();

// 检查交易状态
if (receipt.status === 1) {
    console.log('交易成功');
} else {
    console.log('交易失败');
}
```

### 任务5：处理交易状态

```javascript
// 获取交易收据
async function getTransactionReceipt(txHash) {
    const receipt = await provider.getTransactionReceipt(txHash);

    if (!receipt) {
        console.log('交易尚未确认');
        return null;
    }

    console.log('区块号:', receipt.blockNumber);
    console.log('Gas 使用:', receipt.gasUsed);
    console.log('状态:', receipt.status === 1 ? '成功' : '失败');
    console.log('日志:', receipt.logs);

    return receipt;
}

// 轮询等待确认
async function waitForConfirmation(txHash) {
    while (true) {
        const receipt = await provider.getTransactionReceipt(txHash);
        if (receipt) {
            return receipt;
        }
        await new Promise(resolve => setTimeout(resolve, 1000));
    }
}
```

### 任务6：估算 Gas

```javascript
// 估算 Gas
const gasEstimate = await contract.estimateGas.transfer(to, amount);

// 获取当前 Gas Price
const feeData = await provider.getFeeData();
const gasPrice = feeData.gasPrice;

// 设置 Gas 参数发送交易
const tx = await contract.transfer(to, amount, {
    gasLimit: gasEstimate * 2n,  // 留余量
    gasPrice: gasPrice
});
```

---

## 常见问题

### Q1: Provider 和 Signer 的区别？

**答**：
- Provider：只读，用于查询链上数据，不需要私钥
- Signer：可写，用于签名发送交易，需要私钥

### Q2: 如何判断交易是否成功？

**答**：
```javascript
const receipt = await tx.wait();
const isSuccess = receipt.status === 1;  // 1=成功, 0=失败
```

### Q3: tx.wait() 返回什么？

**答**：返回 TransactionReceipt 对象，包含：
- `status`：交易状态（1=成功，0=失败）
- `blockNumber`：所在区块号
- `gasUsed`：实际消耗的 Gas
- `logs`：事件日志数组
- `contractAddress`：新合约地址（如果是合约创建交易）

---

## 自测题

- [ ] 能解释 Provider 和 Signer 的区别
- [ ] 知道 tx.wait() 返回什么
- [ ] 能手写完整的合约调用流程
- [ ] 能说明如何判断交易是否成功

---

## 进度检查

- [ ] 已阅读 ether.js.md
- [ ] 已手写所有代码示例
- [ ] 已完成自测题

---

## 明天预告

明天学习 **Gas + 交易流程**，理解 Gas 估算原理、交易哈希生成、模拟调用等。
