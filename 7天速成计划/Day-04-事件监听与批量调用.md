# Day 4：事件监听 + 批量调用

## 学习目标

- [ ] 掌握链上事件监听原理
- [ ] 理解 ERC20/ERC721 Transfer 事件
- [ ] 会使用 Multicall 批量调用

---

## 学习资料

- [ ] 真题：第 5、8、9、10 题
  - 第 5 题：如何批量调用多个合约读取请求？
  - 第 8 题：ERC20 与 ERC721 的区别？
  - 第 9 题：如何判断代币是否被转走？
  - 第 10 题：监听交易事件的原理是什么？

---

## 学习内容

### 1. 事件监听原理

**区块链节点存储机制**：
```
交易执行 → 产生日志(logs) → 存储在链上 → 可被索引查询
```

**两种监听方式**：

| 方式 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| **WebSocket 订阅** | 实时推送 | 实时性好，延迟低 | 需要维护连接 |
| **eth_getLogs 轮询** | 定期查询 | 实现简单 | 有延迟，消耗资源 |

### 2. ERC20/721 Transfer 事件

```solidity
// ERC20 Transfer
event Transfer(address indexed from, address indexed to, uint256 value);

// ERC721 Transfer
event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
```

**区别**：
- ERC20: `value` 是金额（可分割）
- ERC721: `tokenId` 是唯一 ID（不可分割）

### 3. Multicall 原理

```
传统方式：              Multicall 方式：
┌─────┐                 ┌───────────┐
│ Call│ ─── RPC ───→    │ Multicall │ ─── RPC ───→ 节点
└─────┘                 │   合约    │
┌─────┐                 │           │
│ Call│ ─── RPC ───→    │  打包调用  │
└─────┘                 │           │
┌─────┐                 │           │
│ Call│ ─── RPC ───→    │           │
└─────┘                 └───────────┘

节省网络往返次数
```

---

## 必会代码

### 任务1：监听 Transfer 事件

```javascript
// 实时监听
contract.on('Transfer', (from, to, value) => {
    console.log(`Transfer ${value} from ${from} to ${to}`);
});

// 或使用过滤器
contract.on(contract.filters.Transfer(address1, address2), (from, to, value) => {
    // 只监听特定地址的转账
});

// 取消监听
contract.off('Transfer', handler);
```

### 任务2：查询历史事件

```javascript
// 查询指定区块范围的事件
const fromBlock = 12345678;
const toBlock = 'latest';

const events = await contract.queryFilter(
    'Transfer',
    fromBlock,
    toBlock
);

// 遍历事件
events.forEach(event => {
    console.log({
        from: event.args[0],
        to: event.args[1],
        value: event.args[2],
        blockNumber: event.blockNumber,
        txHash: event.transactionHash
    });
});
```

### 任务3：使用原生 eth_getLogs

```javascript
const logs = await provider.getLogs({
    address: tokenAddress,
    topics: [
        ethers.id('Transfer(address,address,uint256)'),  // 事件签名
        null,  // from (null 表示不过滤)
        userAddress  // to (只监听转给该用户的)
    ],
    fromBlock: 12345678,
    toBlock: 'latest'
});

// 解析日志
const iface = new ethers.Interface(abi);
logs.forEach(log => {
    const parsed = iface.parseLog(log);
    console.log(parsed.args);
});
```

### 任务4：批量调用（Multicall）

```javascript
// 使用 Multicall 合约
const multicallAbi = [
    'function aggregate(tuple(address target, bytes callData)[] calls) public returns (uint256 blockNumber, bytes[] returnData)'
];

const multicall = new ethers.Contract(multicallAddress, multicallAbi, provider);

// 准备多个调用
const calls = [
    {
        target: token1Address,
        callData: token1Interface.encodeFunctionData('balanceOf', [userAddress])
    },
    {
        target: token2Address,
        callData: token2Interface.encodeFunctionData('balanceOf', [userAddress])
    },
    {
        target: token3Address,
        callData: token3Interface.encodeFunctionData('balanceOf', [userAddress])
    }
];

// 执行批量调用
const [, returnData] = await multicall.aggregate(calls);

// 解析结果
const balance1 = token1Interface.decodeFunctionResult('balanceOf', returnData[0])[0];
const balance2 = token2Interface.decodeFunctionResult('balanceOf', returnData[1])[0];
const balance3 = token3Interface.decodeFunctionResult('balanceOf', returnData[2])[0];
```

### 任务5：批量调用（wagmi）

```javascript
import { readContracts } from 'wagmi';

const [balance1, balance2, balance3] = await readContracts({
    contracts: [
        {
            address: token1Address,
            abi: tokenAbi,
            functionName: 'balanceOf',
            args: [userAddress]
        },
        {
            address: token2Address,
            abi: tokenAbi,
            functionName: 'balanceOf',
            args: [userAddress]
        },
        {
            address: token3Address,
            abi: tokenAbi,
            functionName: 'balanceOf',
            args: [userAddress]
        }
    ]
});
```

### 任务6：判断代币是否转走（双保险）

```javascript
async function checkTokenTransfer(tokenContract, fromAddress) {
    // 方法1：监听事件
    let transferred = false;
    const handler = (from, to, value) => {
        if (from === fromAddress && to !== fromAddress) {
            transferred = true;
            console.log(`检测到转出: ${value}`);
        }
    };
    tokenContract.on('Transfer', handler);

    // 方法2：定期查询余额
    const initialBalance = await tokenContract.balanceOf(fromAddress);
    // ... 等待一段时间
    const currentBalance = await tokenContract.balanceOf(fromAddress);

    if (currentBalance < initialBalance) {
        console.log('余额减少，代币已被转走');
    }

    // 清理监听
    tokenContract.off('Transfer', handler);

    return transferred || currentBalance < initialBalance;
}
```

---

## 真题解答

### Q1: 如何批量调用多个合约读取请求？

**答**：
- 使用 wagmi 的 `readContracts` 方法，一次性传入多个合约配置
- 或调用 Multicall 合约，将多笔只读请求打包执行
- 返回结果逐一解构并在前端展示

### Q2: ERC20 与 ERC721 的区别？

**答**：
- **ERC20**：同质化代币，每个单位没有差别，`value` 是金额
- **ERC721**：非同质化代币，每个 `tokenId` 唯一，常用于 NFT

### Q3: 如何判断代币是否被转走？

**答**：
- **事件监听**：监听 `Transfer` 事件
- **链上查询**：调用 `balanceOf(address)` 查询余额
- **常见做法**：事件监听 + 定期查询双保险

### Q4: 监听交易事件的原理是什么？

**答**：
- 区块链节点在新区块生成时，将交易日志(logs)存储到链上
- SDK 提供订阅功能：订阅区块/合约事件，节点推送事件日志
- 实现方式：WebSocket 订阅（实时推送）或 `eth_getLogs` 轮询

---

## 自测题

- [ ] 能解释事件监听的两种实现方式
- [ ] 能手写 ERC20 Transfer 事件监听代码
- [ ] 知道如何使用 Multicall 优化性能
- [ ] 能说明 ERC20 和 ERC721 的区别

---

## 进度检查

- [ ] 已理解事件监听原理
- [ ] 已手写所有代码示例
- [ ] 已完成自测题

---

## 明天预告

明天学习 **wagmi + 多链支持**，掌握 wagmi 核心 Hooks、多链配置、DEX 路由原理。
