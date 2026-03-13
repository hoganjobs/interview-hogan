# Day 7：真题冲刺 + 模拟面试

## 学习目标

- [ ] 二刷所有 Web3 必考题
- [ ] 模拟面试场景
- [ ] 练习表达和思路

---

## 学习资料

- [ ] `Interview-Front/面试真题/前端(web3含量比较高).md`
- [ ] `Interview-Wallet/进阶/dappfrontend.md`（全部题目）
- [ ] 其他真题：交易所钱包、gmgn 等

---

## Web3 必考题（必须能口头回答）

### 基础概念

#### Q1: 一笔交易怎么确定需要多少 Gas？

**答**：
1. 使用 `eth_estimateGas` 或 `contract.estimateGas.method(args)` 进行估算
2. 节点会在本地模拟执行交易，返回一个预估值
3. 实际执行时仍可能因为链上状态不同而略有差异

**代码示例**：
```javascript
const gasEstimate = await contract.estimateGas.transfer(to, amount);
```

---

#### Q2: 模拟调用和正常调用有什么区别？

**答**：

| 模拟调用 (eth_call) | 正常调用 (sendTransaction) |
|---------------------|---------------------------|
| 本地执行，不修改链上状态 | 链上执行，修改状态 |
| 不消耗 Gas | 消耗 Gas |
| 不需要签名 | 需要用户签名 |
| 返回执行结果 | 返回交易哈希 |
| 用于测试和预览 | 用于实际执行 |

---

#### Q3: 发送交易之后，如何确定交易的状态？

**答**：
1. 通过交易哈希 `txHash` 查询
2. 调用 `provider.getTransactionReceipt(txHash)` 或 `tx.wait()`
3. 返回值中包含：是否成功、区块号、消耗的 Gas、事件日志等
4. 前端可通过轮询或事件监听更新 UI 状态

**代码示例**：
```javascript
const receipt = await tx.wait();
if (receipt.status === 1) {
    console.log('交易成功');
}
```

---

#### Q4: 交易哈希是怎么得到的？

**答**：
1. 在执行写操作时（如 `contract.transfer()`），钱包会返回交易对象
2. 交易对象中包含 `hash` 字段
3. 交易哈希是对交易内容进行 RLP 编码并经过 keccak256 哈希后的结果
4. 链上唯一标识该笔交易

---

### 合约交互

#### Q5: 如何批量调用多个合约读取请求？

**答**：
1. 使用 wagmi 的 `readContracts` 方法，一次性传入多个合约配置
2. 或调用 Multicall 合约，将多笔只读请求打包执行
3. 返回结果可逐一解构并在前端展示

**代码示例**：
```javascript
const [balance1, balance2] = await readContracts({
    contracts: [
        { address: token1, abi, functionName: 'balanceOf', args: [account] },
        { address: token2, abi, functionName: 'balanceOf', args: [account] }
    ]
});
```

---

#### Q6: 如何在项目中新增一条链？

**答**：
1. 在链配置中增加 `chainId`、RPC URL、`currency` 等参数
2. 在前端调用 wagmi/ethers 时指定新的链
3. 配置钱包连接逻辑，使其能切换或添加该链

**代码示例**：
```javascript
const newChain = {
    id: 8453,
    name: 'Base',
    nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://mainnet.base.org'],
    blockExplorers: [{ name: 'BaseScan', url: 'https://basescan.org' }]
};
```

---

#### Q7: DEX 路由是如何处理的？

**答**：
1. 根据用户输入的 `amountIn` 或 `amountOut`，调用路由合约或聚合器
2. 聚合器返回最佳路径（包含多跳 swap）
3. 前端将最优路由展示给用户，并提供 Gas 成本、价格影响等信息

---

### Token 标准

#### Q8: ERC20 与 ERC721 的区别？

**答**：
- **ERC20**：同质化代币，每个单位没有差别，`value` 是金额
- **ERC721**：非同质化代币，每个 `tokenId` 唯一，常用于 NFT

**Transfer 事件**：
```solidity
// ERC20
event Transfer(address indexed from, address indexed to, uint256 value);

// ERC721
event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
```

---

#### Q9: 如何判断代币是否被转走？

**答**：
- **事件监听**：监听 `Transfer` 事件
- **链上查询**：调用 `balanceOf(address)` 查询余额
- **常见做法**：事件监听 + 定期查询双保险

---

#### Q10: 监听交易事件的原理是什么？

**答**：
1. 区块链节点在新区块生成时，会将交易日志(logs)存储到链上
2. SDK 提供订阅功能：订阅区块/合约事件，节点推送事件日志
3. 实现方式：
   - WebSocket 订阅（实时推送）
   - JSON-RPC `eth_getLogs` 轮询

---

### 进阶

#### Q11: Solana 与 Ethereum 签名有什么区别？

**答**：

| Ethereum | Solana |
|----------|--------|
| ECDSA (secp256k1) | Ed25519 |
| 地址由公钥哈希得到 | 地址直接使用公钥 |
| 42 位 16 进制 | Base58 编码 |
| RLP 编码交易 | 序列化交易消息 |

---

#### Q12: 如何区分钱包适配器在多链中的封装？

**答**：
1. **底层**：针对不同链分别封装（EVM → ethers/wagmi，Solana → Solana Web3.js）
2. **中间层**：提供统一的 `connect`、`disconnect`、`signMessage` 接口
3. **顶层**：对外暴露统一的 API，屏蔽链差异

---

## 钱包必考题

### Q1: MetaMask 注入后 window.ethereum 包含哪些功能？

**答**：
- 基础属性：`isConnected()`、`selectedAddress`、`chainId`
- 账户方法：`eth_requestAccounts`、`eth_accounts`
- 网络方法：`wallet_switchEthereumChain`、`wallet_addEthereumChain`
- 交易方法：`eth_sendTransaction`、`personal_sign`、`eth_signTypedData_v4`
- 事件监听：`accountsChanged`、`chainChanged`、`connect`、`disconnect`

---

### Q2: 如何实现钱包连接功能？

**答**：
```javascript
async function connectWallet() {
    if (typeof window.ethereum === 'undefined') {
        alert('请安装 MetaMask!');
        return;
    }

    const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
    });

    return accounts[0];
}
```

---

### Q3: 如何监听钱包切换网络和账户变化？

**答**：
```javascript
window.ethereum.on('accountsChanged', (accounts) => {
    if (accounts.length === 0) {
        handleDisconnect();
    } else {
        handleAccountChange(accounts[0]);
    }
});

window.ethereum.on('chainChanged', (chainId) => {
    window.location.reload();
});
```

---

### Q4: 签名验证流程是如何工作的？

**答**：
1. 用户在前端发起签名请求
2. 钱包弹出签名确认框
3. 用户确认后，钱包用私钥对消息签名
4. 返回签名给前端
5. 前端可以用公钥验证签名是否有效

---

## 模拟面试建议

### 1. 面试流程模拟

```
自我介绍（2-3分钟）
    ↓
项目经验（10-15分钟）
    ↓
技术问答（20-30分钟）
    ↓
代码题（15-20分钟）
    ↓
反问环节（5-10分钟）
```

### 2. 回答技巧：STAR 法则

```
Situation（背景）：什么场景下遇到的问题
Task（任务）：需要完成什么
Action（行动）：你做了什么
Result（结果）：取得了什么成果
```

**示例**：
> S：在之前的 DApp 项目中
> T：需要优化大量合约调用的性能
> A：我使用 Multicall 将 50 个请求合并为 1 个
> R：页面加载时间从 5 秒降到 0.5 秒

### 3. 练习方式

1. **找人对练**：找朋友/同事模拟面试
2. **录音复盘**：录下来自己听，发现问题
3. **白板写代码**：练习在手写环境下写代码
4. **限时回答**：每道题 2 分钟内说完要点

---

## 今日任务清单

- [ ] 二刷 12 道 Web3 必考题
- [ ] 复习钱包连接相关代码
- [ ] 完成一次模拟面试（录音）
- [ ] 整理不熟悉的知识点

---

## 最后检查

- [ ] 已能手写 connectWallet 完整代码
- [ ] 已能手写合约读写代码
- [ ] 已能手写事件监听代码
- [ ] 已能手写批量调用代码
- [ ] 已能手写 Gas 估算代码
- [ ] 已能手写 wagmi 基本用法

---

## 恭喜！

7 天学习结束，你已经掌握了 DApp 前端面试的核心知识点。

**最后提醒**：
1. 保持手感，每天复习一道题
2. 关注行业动态，了解新技术
3. 多做项目，积累实战经验

祝你面试顺利，拿到心仪的 offer！🎉
