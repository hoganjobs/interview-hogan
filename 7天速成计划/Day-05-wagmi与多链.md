# Day 5：wagmi + 多链支持

## 学习目标

- [ ] 掌握 wagmi 核心 Hooks
- [ ] 理解多链配置
- [ ] 了解 DEX 路由原理

---

## 学习资料

- [ ] `Interview-Front/dapp/wagmi.md`
- [ ] 真题：第 6、7 题
  - 第 6 题：如何在项目中新增一条链？
  - 第 7 题：DEX 路由是如何处理的？

---

## 学习内容

### 1. wagmi 核心 Hooks

| Hook | 说明 | 返回值 |
|------|------|--------|
| `useAccount` | 获取账户信息 | `{ address, chainId, isConnected }` |
| `useConnect` | 连接钱包 | `{ connect, connectors, error }` |
| `useDisconnect` | 断开连接 | `{ disconnect }` |
| `useReadContract` | 读合约 | `{ data, error, isLoading }` |
| `useWriteContract` | 写合约 | `{ data: hash, writeContract, error }` |
| `useWaitForTransactionReceipt` | 等待交易确认 | `{ isLoading, isSuccess, data }` |
| `useBalance` | 查询余额 | `{ data, error, isLoading }` |
| `useChainId` | 获取链 ID | `chainId` |
| `useSwitchChain` | 切换链 | `{ switchChain, error }` |

### 2. 多链配置结构

```typescript
interface Chain {
  id: number;              // 链 ID
  name: string;            // 链名称
  nativeCurrency: {        // 原生代币
    name: string;
    symbol: string;
    decimals: number;
  };
  rpcUrls: {
    default: { http: string[] };  // RPC 地址
  };
  blockExplorers: {        // 区块浏览器
    default: { name: string; url: string };
  };
}
```

### 3. DEX 路由原理

```
用户输入: 1 ETH → USDT
    ↓
聚合器查询:
    ├── Uniswap: 1 ETH → 2650 USDT
    ├── Curve:   1 ETH → 2630 USDT
    └── SushiSwap: 1 ETH → 2640 USDT
    ↓
选择最优路径: Uniswap
    ↓
前端展示:
    ├── 预期获得: 2650 USDT
    ├── Gas 费用: ~$5
    ├── 价格影响: 0.01%
    └── 路径: ETH → USDT (Uniswap)
```

---

## 必会代码

### 任务1：wagmi 账户连接

```javascript
import { useAccount, useConnect, useDisconnect } from 'wagmi';

function WalletConnect() {
    const { address, isConnected, chainId } = useAccount();
    const { connect, connectors, error } = useConnect();
    const { disconnect } = useDisconnect();

    if (isConnected) {
        return (
            <div>
                <p>已连接: {address}</p>
                <p>链 ID: {chainId}</p>
                <button onClick={() => disconnect()}>断开连接</button>
            </div>
        );
    }

    return (
        <div>
            {connectors.map(connector => (
                <button
                    key={connector.id}
                    onClick={() => connect({ connector })}
                >
                    连接 {connector.name}
                </button>
            ))}
            {error && <p>错误: {error.message}</p>}
        </div>
    );
}
```

### 任务2：读取合约数据

```javascript
import { useReadContract } from 'wagmi';

function TokenBalance({ address, tokenAddress }) {
    const { data: balance, error, isLoading } = useReadContract({
        address: tokenAddress,
        abi: tokenAbi,
        functionName: 'balanceOf',
        args: [address]
    });

    if (isLoading) return <div>加载中...</div>;
    if (error) return <div>错误: {error.message}</div>;

    return <div>余额: {balance?.toString()}</div>;
}
```

### 任务3：写入合约

```javascript
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi';

function TokenTransfer() {
    const { data: hash, writeContract, error, isPending } = useWriteContract();

    const { isLoading: confirming, isSuccess } = useWaitForTransactionReceipt({
        hash,
    });

    const handleTransfer = () => {
        writeContract({
            address: tokenAddress,
            abi: tokenAbi,
            functionName: 'transfer',
            args: [toAddress, amount]
        });
    };

    return (
        <div>
            <button
                onClick={handleTransfer}
                disabled={isPending || confirming}
            >
                {isPending || confirming ? '交易中...' : '发送'}
            </button>
            {isSuccess && <p>交易成功!</p>}
            {error && <p>错误: {error.message}</p>}
        </div>
    );
}
```

### 任务4：添加新链配置

```javascript
import { defineChain } from 'viem';

// 定义新链
const myChain = defineChain({
    id: 8453,
    name: 'Base',
    nativeCurrency: {
        name: 'Ether',
        symbol: 'ETH',
        decimals: 18
    },
    rpcUrls: {
        default: {
            http: ['https://mainnet.base.org']
        }
    },
    blockExplorers: {
        default: {
            name: 'BaseScan',
            url: 'https://basescan.org'
        }
    }
});

// 在 wagmi 配置中使用
import { createConfig, http } from 'wagmi';
import { injected, walletConnect } from 'wagmi/connectors';

export const config = createConfig({
    chains: [mainnet, polygon, myChain],  // 添加自定义链
    connectors: [
        injected(),
        walletConnect({ projectId: '...' })
    ],
    transports: {
        [mainnet.id]: http(),
        [polygon.id]: http(),
        [myChain.id]: http()  // 添加 RPC 传输
    }
});
```

### 任务5：切换链

```javascript
import { useSwitchChain } from 'wagmi';

function ChainSwitcher() {
    const { switchChain, error } = useSwitchChain();

    const handleSwitch = () => {
        switchChain({ chainId: 8453 });  // 切换到 Base
    };

    return (
        <button onClick={handleSwitch}>
            切换到 Base
        </button>
    );
}
```

### 任务6：DEX 路由查询示例

```javascript
// 调用 1inch API 获取最优路径
async function getBestRoute(amountIn, fromToken, toToken) {
    const response = await fetch(
        `https://api.1inch.dev/swap/v6.0/1/quote?` +
        `src=${fromToken}&dst=${toToken}&amount=${amountIn}`
    );
    const data = await response.json();

    return {
        expectedAmount: data.dstAmount,
        gasPrice: data.gasPrice,
        gas: data.estimateGas,
        route: data.routes  // 路由信息
    };
}

// 执行交易
async function executeSwap(route) {
    const tx = await writeContract({
        address: route.to,           // DEX 合约地址
        abi: routerAbi,
        functionName: 'swap',        // 交易方法
        args: route.calldata         // 编码后的交易数据
    });
}
```

---

## 真题解答

### Q1: 如何在项目中新增一条链？

**答**：
1. 在链配置中增加 `chainId`、RPC URL、`currency` 等参数
2. 在前端调用 wagmi/ethers 时指定新的链
3. 配置钱包连接逻辑，使其能切换或添加该链

### Q2: DEX 路由是如何处理的？

**答**：
1. 根据用户输入的 `amountIn` 或 `amountOut`，调用路由合约或聚合器
2. 聚合器返回最佳路径（包含多跳 swap）
3. 前端将最优路由展示给用户，并提供 Gas 成本、价格影响等信息

---

## 自测题

- [ ] 能对比 ethers.js 和 wagmi 的使用差异
- [ ] 知道如何在项目中新增一条链
- [ ] 能简单说明 DEX 路由的处理逻辑
- [ ] 能手写 wagmi 的基本 Hook 使用

---

## 进度检查

- [ ] 已阅读 wagmi.md
- [ ] 已手写所有代码示例
- [ ] 已完成自测题

---

## 明天预告

明天学习 **传统前端补漏**，复习 JavaScript 核心概念和 React Hooks/性能优化。
