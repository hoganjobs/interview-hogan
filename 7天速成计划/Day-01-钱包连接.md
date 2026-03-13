# Day 1：钱包连接 + window.ethereum

## 学习目标

- [ ] 掌握 MetaMask 钱包连接完整流程
- [ ] 理解 window.ethereum 对象的所有核心方法
- [ ] 会写账户/网络变化监听代码

---

## 学习资料

- [ ] `Interview-Wallet/进阶/dapp前端.md` 第 2-6 题

---

## 学习内容

### 1. window.ethereum 核心方法

| 方法 | 说明 | 代码示例 |
|------|------|----------|
| `isConnected()` | 检查连接状态 | `window.ethereum.isConnected()` |
| `selectedAddress` | 当前地址 | `window.ethereum.selectedAddress` |
| `chainId` | 当前链 ID | `window.ethereum.chainId` |
| `eth_requestAccounts` | 请求连接 | `request({ method: 'eth_requestAccounts' })` |
| `eth_accounts` | 获取账户列表 | `request({ method: 'eth_accounts' })` |
| `wallet_switchEthereumChain` | 切换网络 | `request({ method: 'wallet_switchEthereumChain', params: [{ chainId }] })` |
| `wallet_addEthereumChain` | 添加网络 | `request({ method: 'wallet_addEthereumChain', params: [chainConfig] })` |
| `eth_sendTransaction` | 发送交易 | `request({ method: 'eth_sendTransaction', params: [tx] })` |
| `personal_sign` | 签名消息 | `request({ method: 'personal_sign', params: [msg, addr] })` |
| `eth_signTypedData_v4` | EIP-712 签名 | `request({ method: 'eth_signTypedData_v4', params: [addr, data] })` |

### 2. 事件监听

| 事件 | 说明 | 触发时机 |
|------|------|----------|
| `accountsChanged` | 账户变化 | 用户切换账户 |
| `chainChanged` | 链变化 | 用户切换网络 |
| `connect` | 连接成功 | 钱包连接 |
| `disconnect` | 断开连接 | 钱包断开 |

---

## 必会代码（能手写）

### 任务1：实现钱包连接功能

```javascript
async function connectWallet() {
    try {
        // 1. 检查是否安装了 MetaMask
        if (typeof window.ethereum === 'undefined') {
            alert('请安装 MetaMask!');
            return;
        }

        // 2. 请求连接钱包
        const accounts = await window.ethereum.request({
            method: 'eth_requestAccounts'
        });

        // 3. 返回钱包地址
        return accounts[0];

    } catch (error) {
        if (error.code === 4001) {
            alert('用户拒绝连接');
        } else {
            alert('连接出错');
        }
    }
}
```

### 任务2：监听账户和网络变化

```javascript
function setupWalletListeners() {
    // 1. 监听账户变化
    window.ethereum.on('accountsChanged', (accounts) => {
        if (accounts.length === 0) {
            // 用户断开了连接
            handleDisconnect();
        } else {
            // 用户切换了账户
            handleAccountChange(accounts[0]);
        }
    });

    // 2. 监听网络变化
    window.ethereum.on('chainChanged', (chainId) => {
        console.log('切换到链:', chainId);
        // MetaMask 推荐在链切换时刷新页面
        window.location.reload();
    });
}
```

### 任务3：处理断开连接

```javascript
function handleDisconnect() {
    // 清理状态
    localStorage.removeItem('walletConnected');
    // 更新 UI
    updateUI({
        isConnected: false,
        account: null,
        chainId: null
    });
}
```

### 任务4：发送交易

```javascript
async function sendTransaction(to, value) {
    const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
    });

    const txHash = await window.ethereum.request({
        method: 'eth_sendTransaction',
        params: [{
            from: accounts[0],
            to: to,
            value: value, // '0x...'
        }]
    });

    return txHash;
}
```

### 任务5：签名消息

```javascript
async function signMessage(message) {
    const accounts = await window.ethereum.request({
        method: 'eth_requestAccounts'
    });

    const signature = await window.ethereum.request({
        method: 'personal_sign',
        params: [message, accounts[0]]
    });

    return signature;
}
```

---

## 自测题

- [ ] 能手写 connectWallet 完整实现（含错误处理）
- [ ] 能解释 accountsChanged 和 chainChanged 的处理逻辑
- [ ] 知道错误码 4001 是什么（用户拒绝）
- [ ] 能说明为什么 chainChanged 时要刷新页面

---

## 进度检查

- [ ] 已阅读 dapp前端.md 第 2-6 题
- [ ] 已手写所有代码示例
- [ ] 已完成自测题

---

## 明天预告

明天学习 **ethers.js 核心操作**，理解 Provider、Signer、Contract 的区别。
