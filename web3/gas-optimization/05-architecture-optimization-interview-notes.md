# Gas 架构优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（五）| 登链社区](https://learnblockchain.cn/article/22695)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 一、批量处理优化

### 1. Multicall / Multidelegatecall ⭐⭐⭐

允许用户在一个交易中调用多个函数，保留 `msg.sender` 和 `msg.value`。

```solidity
// Uniswap 实现
function multicall(bytes[] calldata data) public payable returns (bytes[] memory results) {
    results = new bytes[](data.length);
    for (uint256 i = 0; i < data.length; i++) {
        (bool success, bytes memory result) = address(this).delegatecall(data[i]);
        
        if (!success) {
            if (result.length < 68) revert();
            assembly {
                result := add(result, 0x04)
            }
            revert(abi.decode(result, (string)));
        }
        results[i] = result;
    }
}
```

**⚠️ 注意**: `msg.value` 是持久的，可能导致安全问题

---

## 二、白名单/Airdrop 优化方案

### 2. Merkle 树 ⭐⭐⭐

**传统方式问题**:
```solidity
// ❌ 10,000 个地址 ≈ 200,000,000 gas
mapping(address => bool) public whitelist;
```

**Merkle 树方案**:
```solidity
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract MerkleAirdrop {
    bytes32 public immutable merkleRoot;  // 仅存储 32 字节根哈希
    mapping(address => bool) public claimed;

    function claim(uint256 amount, bytes32[] calldata merkleProof) external {
        require(!claimed[msg.sender], "Already claimed");
        
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount));
        require(MerkleProof.verify(merkleProof, merkleRoot, leaf), "Invalid proof");
        
        claimed[msg.sender] = true;
        // 执行 airdrop...
    }
}
```

### 3. ECDSA 签名方案 ⭐⭐⭐

```solidity
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract SignatureAirdrop {
    address public immutable signer;
    mapping(address => bool) public claimed;

    function claim(uint256 amount, bytes calldata signature) external {
        require(!claimed[msg.sender], "Already claimed");
        
        bytes32 messageHash = keccak256(abi.encodePacked(msg.sender, amount));
        bytes32 ethSignedMessageHash = messageHash.toEthSignedMessageHash();
        
        address recoveredSigner = ethSignedMessageHash.recover(signature);
        require(recoveredSigner == signer, "Invalid signature");
        
        claimed[msg.sender] = true;
    }
}
```

### Merkle 树 vs ECDSA 签名对比

| 指标 | Merkle 树 | ECDSA 签名 |
|-----|----------|-----------|
| 部署成本 | ~20,000 gas | ~20,000 gas |
| Claim 成本 | ~50,000 gas | ~35,000 gas ✅ |
| Calldata | ~14×32 字节 (10k用户) | 65 字节 (固定) ✅ |
| 去中心化 | 高 ✅ | 依赖签名者 |
| 动态白名单 | 需重新部署 | 随时签发 ✅ |
| 私钥风险 | 无 | 有泄露风险 |

### 选择建议

**选 Merkle 树**:
- 去中心化要求高
- 百万级用户
- 需要公开透明
- 无需信任服务器

**选 ECDSA 签名**:
- 用户体验优先 (calldata 小)
- 动态白名单
- Gas 成本敏感
- 10k 以下用户

---

## 三、代币标准选择

### 4. ERC20Permit ⭐⭐⭐

一笔交易完成授权 + 转账：

```solidity
// 传统方式：两笔交易
token.approve(spender, amount);  // 交易 1
token.transferFrom(owner, spender, amount);  // 交易 2

// ERC20Permit：一笔交易
token.permit(owner, spender, amount, deadline, v, r, s);
token.transferFrom(owner, spender, amount);
```

**好处**: 用户不需要单独支付 approve 的 Gas

### 5. ERC1155 vs ERC721 ⭐⭐

| 特性 | ERC721 | ERC1155 |
|-----|--------|---------|
| 单合约多代币 | ❌ 每个代币单独合约 | ✅ 一个合约管理所有 |
| 余额追踪 | balanceOf 很少用 | 按 id 跟踪余额 ✅ |
| 批量转账 | 不支持 | 支持 ✅ |
| Gas 效率 | 较低 | 较高 ✅ |

### 6. ERC1155/ERC6909 代替多个 ERC20 ⭐

- 一个合约管理多种代币
- **缺点**: 与部分 DeFi 原语不兼容
- 不需要回调可用 ERC6909

---

## 四、升级模式选择

### 7. UUPS vs 透明代理 ⭐⭐

| 模式 | Gas 成本 | 原理 |
|-----|---------|------|
| 透明代理 | 高 | 每次交易都读取管理员地址 |
| UUPS ✅ | 低 | 升级逻辑在实现合约中 |

> **原理**: 透明代理每次交易都要检查调用者是否为管理员，额外存储读取

---

## 五、替代库推荐

### 8. Solady vs OpenZeppelin ⭐⭐

| 功能 | OpenZeppelin | Solady | 节省 |
|-----|-------------|--------|-----|
| ERC20 Transfer | 677 gas | 546 gas | 19% |
| ERC721 Mint | 2,800 gas | 2,200 gas | 21% |
| MulDivDown | 674 gas | 504 gas | 25% |
| Sqrt | 1,146 gas | 683 gas | 40% |

**Solady 特点**:
- 大量使用汇编优化
- 经过审计和实战验证
- Gas 效率极高

**选择建议**:
- 常规项目：OpenZeppelin（安全、文档完善）
- 高频调用：Solady（极致优化）

---

## 六、其他架构优化

### 9. 迁移到 Layer2 ⭐⭐

| 场景 | 建议 |
|-----|------|
| 高吞吐、低价值交易 | 考虑 L2 (Optimism, Arbitrum, Base) |
| 游戏/高频交互 | L2 或侧链 (Polygon, BNB Chain) |

### 10. 状态通道 ⭐

- 最古老的可扩展性方案
- 链下交易，链上结算
- 适合特定应用场景

### 11. 投票委托 (ERC20 Votes) ⭐

- 代币持有者委托代表投票
- 减少投票交易数量

### 12. 可迭代映射 ⭐

- 数组维护成本高（插入/删除涉及大量移动）
- 用链表实现可迭代映射，O(1) 操作

---

## 七、DApp 前端相关要点

### 7.1 前端实现 Merkle 证明

```javascript
import { MerkleTree } from 'merkletreejs';
import keccak256 from 'keccak256';

// 构建树
const leaves = addresses.map(addr => keccak256(addr));
const tree = new MerkleTree(leaves, keccak256, { sortPairs: true });
const root = tree.getRoot();

// 生成证明
const proof = tree.getHexProof(keccak256(userAddress));

// 调用合约
await contract.claim(amount, proof);
```

### 7.2 前端实现 ECDSA 签名验证

```javascript
// 后端签名
const messageHash = ethers.solidityPackedKeccak256(
    ['address', 'uint256'],
    [userAddress, amount]
);
const signature = await signer.signMessage(ethers.getBytes(messageHash));

// 前端调用
await contract.claim(amount, signature);
```

### 7.3 ERC20Permit 集成

```javascript
const domain = {
    name: tokenName,
    version: '1',
    chainId: chainId,
    verifyingContract: tokenAddress,
};

const types = {
    Permit: [
        { name: 'owner', type: 'address' },
        { name: 'spender', type: 'address' },
        { name: 'value', type: 'uint256' },
        { name: 'nonce', type: 'uint256' },
        { name: 'deadline', type: 'uint256' },
    ],
};

const signature = await signer.signTypedData(domain, types, value);
const { v, r, s } = ethers.Signature.from(signature);

// 一笔交易完成 permit + transferFrom
await contract.claimWithPermit(amount, deadline, v, r, s);
```

---

## 八、面试常见问题

### Q1: Airdrop/白名单场景如何选择 Merkle 树还是 ECDSA 签名？

**回答要点**:
- Gas: ECDSA 更省 (35k vs 50k)
- Calldata: ECDSA 固定 65 字节
- 去中心化: Merkle 更好
- 动态性: ECDSA 更灵活
- 用户规模: 百万级用 Merkle，中小规模用签名

### Q2: 什么是 ERC20Permit？有什么好处？

**回答**:
- 用户签名授权，无需单独发 approve 交易
- 授权 + 转账合并为一笔交易
- 用户不需要持有 ETH 支付 approve Gas

### Q3: 为什么 UUPS 比透明代理省 Gas？

**回答**:
- 透明代理每次交易都要读取管理员地址判断
- UUPS 把升级逻辑放在实现合约，省去这个检查

### Q4: 什么时候考虑用 Solady？

**回答**:
- 高频调用的合约
- Gas 优化是首要目标
- 团队有 Solidity 和汇编经验

---

## 九、快速记忆

```
架构优化看场景，
Merkle 省存储签名省调用，
Permit 合并 approve，
1155 代替 721，
UUPS 比透明省，
Solady 汇编强，
L2 降成本好帮手。
```

---

*整理人: 小鹅斯基 🦆*
*用于: DApp 前端开发面试准备*
