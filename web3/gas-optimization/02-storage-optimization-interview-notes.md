# Gas 存储优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（二）| 登链社区](https://learnblockchain.cn/article/22692)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## 一、核心存储优化原则

### 存储操作 Gas 成本速查

| 操作类型 | Gas 成本 |
|---------|---------|
| 零 → 非零写入 | **22,100 gas** (20,000 写入 + 2,100 冷访问) |
| 非零 → 非零写入 | **5,000 gas** |
| 非零 → 零写入 | 5,000 gas (有退款) |
| SLOAD (冷访问) | 2,100 gas |
| SLOAD (热访问) | 100 gas |
| TSTORE/TLOAD | **100 gas** (瞬时存储) |

> **关键洞察**: 从零到非零的写入是最昂贵的操作！

---

## 二、15 条存储优化技巧

### 1. 避免零到非零写入 ⭐⭐⭐

- 初始化存储变量是最昂贵的操作之一
- OpenZeppelin 重入保护使用 1 和 2，而不是 0 和 1
- 避免让 ERC20 余额变为零，始终保留一小笔金额

### 2. 缓存存储变量 ⭐⭐⭐

```solidity
// ❌ 低效：读取两次
function increment() public {
    require(number < 10);
    number = number + 1;  // 第二次读取
}

// ✅ 高效：只读取一次
function increment() public {
    uint256 _number = number;  // 缓存到内存
    require(_number < 10);
    number = _number + 1;
}
```

> **原理**: Solidity 不会自动缓存存储读取

### 3. 变量打包到同一槽位 ⭐⭐⭐

**存储槽大小**: 32 字节 (256 bits)

| 方案 | 槽位数量 | 效率 |
|-----|---------|-----|
| 手动打包 (位移) | 1 个 | 最高 ✅ |
| EVM 自动打包 | 1 个 | 略低 |
| 不打包 (uint256) | 2 个 | 最低 ❌ |

```solidity
// ✅ 高效：两个 uint80 打包到 uint160
uint160 public packedVariables;
function packVariables(uint80 x, uint80 y) external {
    packedVariables = uint160(x) << 80 | uint160(y);
}

// ❌ 低效：使用两个 uint256
uint256 public var1;  // 占用整个槽位
uint256 public var2;  // 占用整个槽位
```

### 4. 结构体成员打包 ⭐⭐

```solidity
// ❌ 未优化：3 个槽位
struct unpackedStruct {
    uint64 time;      // 槽位 1 (只用 64 bits)
    uint256 money;    // 槽位 2 (完整 256 bits)
    address person;   // 槽位 3 (只用 160 bits)
}

// ✅ 优化后：2 个槽位
struct packedStruct {
    uint64 time;      // 槽位 1 (64 bits)
    address person;   // 槽位 1 (160 bits) - 与 time 共享！
    uint256 money;    // 槽位 2 (256 bits)
}
```

> **技巧**: 把小于 32 字节的变量放在一起，让大变量单独占槽

### 5. 保持字符串 < 32 字节 ⭐⭐

| 字符串长度 | 存储方式 | Gas 成本 |
|-----------|---------|---------|
| < 32 字节 | 直接存在槽位中 | 低 ✅ |
| ≥ 32 字节 | 长度在原槽位，数据在 keccak(槽位) | 高 ❌ |

### 6. 使用 constant 和 immutable ⭐⭐⭐

```solidity
// ✅ 嵌入字节码，不占用存储
contract Constants {
    uint256 constant MAX_UINT256 = 0xff...ff;  // 编译时嵌入
    uint immutable maxBalance;                  // 部署时嵌入

    function get_max_value() external pure returns (uint256) {
        return MAX_UINT256;
    }
}

// ❌ 每次读取都需要 SLOAD
contract NoConstants {
    uint256 MAX_UINT256 = 0xff...ff;  // 存储在槽位中
}
```

### 7. 瞬时存储 (Transient Storage) ⭐⭐⭐ 新特性

**Cancun 升级 (2024年3月)** 引入的新型存储：

| 特性 | 说明 |
|-----|------|
| 操作码 | TSTORE / TLOAD |
| Gas 成本 | **100 gas** (vs SSTORE 22,100) |
| 生命周期 | 仅在单个交易内有效 |
| 节省 | **99.5%** |

```solidity
// ❌ 传统方案：~25,000 gas
contract TraditionalReentrancyGuard {
    uint256 private _status;
    modifier nonReentrant() {
        require(_status != 2);
        _status = 2;  // SSTORE: 22,100 gas
        _;
        _status = 1;  // SSTORE: 2,900 gas
    }
}

// ✅ 瞬时存储：仅 200 gas (Solidity 0.8.24+)
contract TransientReentrancyGuard {
    bool private transient _locked;
    modifier nonReentrant() {
        require(!_locked);
        _locked = true;   // TSTORE: 100 gas
        _;
        _locked = false;  // TSTORE: 100 gas
    }
}
```

**适用场景**:
- 重入锁 (Reentrancy Guard)
- 闪电贷状态管理
- 批量操作的中间状态
- 跨合约调用的瞬时标记

### 8. 用 Mapping 代替 Array ⭐⭐

```solidity
// ❌ 数组：get(0) = 4,860 gas
contract Array {
    uint256[] a;
    function get(uint256 index) external view returns(uint256) {
        return a[index];  // Solidity 添加边界检查
    }
}

// ✅ 映射：get(0) = 2,758 gas (省 2,102 gas)
contract Mapping {
    mapping(uint256 => uint256) a;
    function get(uint256 index) external view returns(uint256) {
        return a[index];  // 无边界检查
    }
}
```

> **原因**: 数组读取时会检查 index < length，映射没有这个检查

### 9. 使用 unsafeAccess (OpenZeppelin)

如果必须用数组，可以用 `unsafeAccess` 跳过长度检查（需确保索引安全）

### 10. 布尔位图 ⭐⭐

一个存储槽 = 256 位，可以存储 **256 个布尔标志**

适用场景：空投领取状态、NFT mint 状态

### 11. SSTORE2 / SSTORE3 ⭐

存储大量数据时的替代方案：

| 方案 | 原理 | 适用场景 |
|-----|------|---------|
| SSTORE | 存储槽存储 | 常规数据 |
| SSTORE2 | 部署合约，数据存为字节码 | 写少读多，指针 > 14 字节 |
| SSTORE3 | 结合 SSTORE + CREATE2 | 写少读多，指针 < 14 字节 |

**成本对比**:
- SSTORE: ~690 gas/字节
- 合约字节码: ~200 gas/字节

### 12. 存储指针 vs 内存 ⭐⭐

```solidity
// ❌ 复制整个结构体到内存
function returnLastSeenSecondsAgo(uint256 _id) public view returns (uint256) {
    User memory _user = users[_id];  // 复制所有字段
    return block.timestamp - _user.lastSeen;
}

// ✅ 使用存储指针，只读取需要的字段
function returnLastSeenSecondsAgoOptimized(uint256 _id) public view returns (uint256) {
    User storage _user = users[_id];  // 只存指针
    return block.timestamp - _user.lastSeen;  // 只读取 lastSeen
}
```

**节省**: ~5,000 gas

### 13. 避免 ERC20 余额归零

频繁清空账户会导致大量零→非零写入

### 14. 倒数计数

```solidity
// ✅ 从 n 倒数到 0
for (uint i = n; i > 0; i--) {
    // 变量归零时有退款
}
```

### 15. 时间戳和区块号用小类型

- `uint48` 时间戳可用数百万年
- 区块号约 12 秒递增一次

---

## 三、DApp 前端相关要点

### 3.1 与前端交互的优化点

| 场景 | 建议 |
|-----|------|
| 读取合约状态 | 使用 `view` 函数，不消耗 gas |
| 批量操作 | 考虑合约端支持批量处理 |
| 状态查询 | 使用事件日志 (Event) 而非存储变量 |
| 前端缓存 | 缓存不变的合约数据 (constant/immutable) |

### 3.2 面试回答思路

**Q: 前端如何帮助用户节省 Gas？**

1. 合并交易 - 批量操作减少交易次数
2. 预估 Gas - 显示预估费用让用户知情
3. 选择时机 - 网络拥堵时提示用户等待
4. 缓存数据 - 减少不必要的链上查询

**Q: 你了解 Transient Storage 吗？**

- Cancun 升级引入 (2024年3月)
- 仅 100 gas vs SSTORE 22,100 gas
- 数据仅在单个交易内有效
- 适合重入锁、闪电贷等场景
- DApp 前端可能需要考虑兼容性

---

## 四、面试常见问题

### Q1: 为什么零到非零写入这么贵？

**回答**: 需要从空白状态初始化存储，涉及：
- 20,000 gas 的存储写入费用
- 2,100 gas 的冷访问费用
- 总计 22,100 gas

### Q2: 如何优化结构体存储？

**回答**:
1. 把小于 32 字节的变量放在一起
2. 让 uint256 等大类型单独占槽
3. 合理排序成员顺序

### Q3: constant 和 immutable 的区别？

| 特性 | constant | immutable |
|-----|----------|-----------|
| 赋值时机 | 编译时 | 部署时 |
| 修改 | 不可 | 不可 |
| 适用 | 已知常量 | 构造函数参数 |

---

## 五、快速记忆口诀

```
存储优化记心间，
零写最贵两万二，
缓存变量只读一次，
打包槽位省空间，
常量不变用 const，
瞬时存储 Cancun 新，
映射代替数组查，
位图存储布尔值。
```

---

*整理人: 小鹅斯基 🦆*
*用于: DApp 前端开发面试准备*
