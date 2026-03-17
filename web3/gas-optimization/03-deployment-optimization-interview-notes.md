# Gas 部署优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（三）| 登链社区](https://learnblockchain.cn/article/22693)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 一、部署成本计算

### 部署成本公式

```
部署成本 = 基础成本 + 代码存储成本 + 初始化成本
```

| 组成部分 | Gas 成本 |
|---------|---------|
| 基础成本 (CREATE/CREATE2) | 32,000 gas |
| 代码存储 | **200 gas / 字节** |
| 构造函数执行 | 视代码而定 |

### 实际计算示例

```
合约字节码: 10,000 字节

- 基础: 32,000 gas
- 代码存储: 10,000 × 200 = 2,000,000 gas
- 构造函数: ~500,000 gas
总计: ~2,532,000 gas

实际成本 (50 gwei):
2,532,000 × 50 gwei = 0.1266 ETH ≈ $250
```

> **面试要点**: 代码存储是部署成本的大头，每字节 200 gas

---

## 二、合约大小限制

### EIP-170 限制

**最大合约大小**: 24,576 字节 (24 KB)

### 超出限制的解决方案

| 方案 | 说明 |
|-----|------|
| **拆分合约** | 首选方案，分离功能到多个小合约 |
| **Diamond Pattern** | 钻石模式，功能拆分到多个 Facet |
| **Proxy Pattern** | 代理模式，逻辑合约可更换 |
| **Library** | 使用库函数 |

### 拆分合约的思考问题

1. 哪些函数应该在一起？
2. 哪些函数不需要读取全部状态？
3. 能否把存储和函数分开？

---

## 三、部署优化技巧

### 1. 优化器设置 ⭐⭐⭐

#### runs 参数

| runs 值 | 优化目标 | 适用场景 |
|--------|---------|---------|
| 1 | 部署成本最低 | 一次性合约、部署后很少调用 |
| 200 (默认) | 运行时平衡 | 常规合约 |
| 更高值 | 运行时成本最低 | 高频调用的函数 |

#### viaIR 编译选项

```toml
# foundry.toml
[profile.default]
solc_version = "0.8.24"
optimizer = true
optimizer_runs = 1
via_ir = true  # 启用 IR 优化
```

**IR 优化好处**:
- 更激进的代码优化，生成更紧凑字节码
- 某些情况减少 10-20% 合约大小
- 对接近 24KB 限制的合约特别有用

**注意事项**:
- 编译时间增加 2-5 倍
- 某些边缘情况 gas 消耗可能不同
- 需要测试对比

### 2. 预测合约地址 ⭐⭐

通过 nonce 预先计算合约地址，避免存储变量：

```solidity
import {LibRLP} from "solady/src/utils/LibRLP.sol";

contract BurnerDeployer {
    using LibRLP for address;

    function deploy() public returns(StorageContract storageContract, address writer) {
        // 预测 nonce=2 时的地址
        StorageContract storageContractComputed =
            StorageContract(address(this).computeAddress(2));

        // 先部署 Writer (nonce=1)
        writer = address(new Writer(storageContractComputed));

        // 再部署 StorageContract (nonce=2)
        storageContract = new StorageContract(writer);

        require(storageContract == storageContractComputed, "address mismatch");
    }
}
```

**好处**: 节省 2k+ gas，不需要 setter 函数

### 3. 构造函数设为 payable ⭐⭐

```solidity
// ✅ 节省 200 gas
contract B {
    constructor() payable {}
}

// ❌ 隐式检查 msg.value == 0
contract A {
    constructor() {}
}
```

**原理**: 非 payable 函数会插入隐式 `require(msg.value == 0)`

### 4. 优化 IPFS 哈希 ⭐

- Solidity 编译器附加 51 字节元数据
- 每字节 200 gas → 共 10,200+ gas
- 方案 1: `--no-cbor-metadata` 禁用（影响验证）
- 方案 2: 优化代码注释使哈希包含更多零

### 5. selfdestruct 一次性合约 ⭐

```solidity
contract ConstructorSelfDestruct {
    constructor() payable {
        _initialize();
        // ✅ 构造函数内自毁（同一交易）
        selfdestruct(payable(tx.origin));
    }
}
```

⚠️ **Cancun 升级 (EIP-6780) 后的变化**:

| 场景 | SELFDESTRUCT 效果 |
|-----|------------------|
| 同一交易内创建并销毁 | ✅ 完整功能（删除代码、清除存储） |
| 已存在合约调用 | ❌ 仅转移 ETH，不删除代码/存储 |

### 6. Modifier vs Internal Function ⭐⭐

| 特性 | Modifier | Internal Function |
|-----|----------|------------------|
| 部署成本 | 高（代码重复） | 低 ✅ |
| 运行时成本 | 低 ✅ | 高 |
| 灵活性 | 只能在函数头/尾 | 可在任意位置 |

```solidity
// Modifier: 部署 195,435 gas，调用 28,367 gas
modifier onlyOwner() {
    require(msg.sender == owner);
    _;
}

// Internal: 部署 159,309 gas，调用 28,391 gas
function onlyOwner() internal view {
    require(msg.sender == owner);
}
```

**权衡**: 高频调用用 Modifier，部署成本敏感用 Internal

### 7. 克隆代理 (EIP-1167) ⭐⭐

部署多个相似合约时使用最小代理：

```solidity
// EIP-1167 最小代理标准
// 字节码中存储实现合约地址，以代理方式交互
```

**权衡**:
- 部署成本大幅降低
- 运行时多几十到一两百 gas（代理跳转）

**案例**: Gnosis Safe 使用克隆降低部署成本

### 8. 管理员函数设为 payable ⭐

管理员函数设为 payable：
- 跳过 msg.value 检查
- 合约更小，部署成本更低

### 9. 自定义错误 vs require ⭐⭐⭐

```solidity
// ✅ 自定义错误：只存储 4 字节
error InvalidAmount();
if (_amount > 10 ether) revert InvalidAmount();

// ❌ require 字符串：至少 64 字节
require(_amount <= 10 ether, "Error: Pass in a valid amount");
```

**原理**: 自定义错误只存储错误签名的前 4 字节

### 10. 复用 CREATE2 工厂 ⭐

需要确定性地址时，复用预先部署的工厂合约

---

## 四、DApp 前端相关要点

### 4.1 部署成本估算

```javascript
// ethers.js 估算部署成本
const factory = new ethers.ContractFactory(abi, bytecode, signer);
const deployTx = factory.getDeployTransaction(args);
const estimatedGas = await provider.estimateGas(deployTx);
const feeData = await provider.getFeeData();

const estimatedCost = estimatedGas * feeData.gasPrice;
console.log(`部署预估: ${ethers.formatEther(estimatedCost)} ETH`);
```

### 4.2 前端展示建议

| 场景 | 建议 |
|-----|------|
| 合约部署 | 显示预估 Gas 和 ETH/USD 成本 |
| 批量部署 | 考虑使用工厂合约 + 克隆代理 |
| 高频交互 | 提示用户选择 Gas 优化时机 |

---

## 五、面试常见问题

### Q1: 合约大小限制是多少？超出怎么办？

**回答**:
- EIP-170 限制 24 KB
- 解决方案：拆分合约、代理模式、钻石模式、使用库

### Q2: 优化器 runs 参数怎么选？

**回答**:
- runs=1: 优化部署成本，适合一次性合约
- runs=200(默认): 平衡部署和运行
- 高 runs 值: 优化高频调用

### Q3: 自定义错误为什么比 require 省 Gas？

**回答**:
- 自定义错误只存储 4 字节签名
- require 字符串至少 64 字节
- revert 时内存存储更少

### Q4: 什么是克隆代理？

**回答**:
- EIP-1167 最小代理标准
- 多个代理指向同一实现合约
- 大幅降低部署成本，运行时略有增加

---

## 六、快速记忆

```
部署成本三部分，
基础三万二固定，
代码两百每字节，
构造函数看情况。

优化手段要记牢，
拆分合约是首选，
runs 设一省部署，
IR 优化减体积，
克隆代理批量用，
自定义错误省空间。
```

---

*整理人: 小鹅斯基 🦆*
*用于: DApp 前端开发面试准备*
