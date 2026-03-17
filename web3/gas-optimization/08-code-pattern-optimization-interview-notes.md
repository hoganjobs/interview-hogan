# Gas 代码模式优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（八）| 登链社区](https://learnblockchain.cn/article/22698)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## ⚠️ 重要提醒

> **始终进行基准测试！编译器不断改进，某些技巧可能已过时或适得其反。**
> 使用 `--via-ir` 编译器标志时，某些技巧可能降低效率。

---

## 一、条件与循环优化

### 1. 严格不等式 vs 非严格不等式 ⭐

```solidity
// 建议：使用严格不等式，但要测试
if (x > 0) { }   // ✅ 推荐
if (x >= 1) { }  // ❌ EVM 无 >= 操作码，编译为 !(x < 1)
```

### 2. 拆分 require 语句 ⭐⭐

```solidity
// ❌ 两个条件都会评估
require(x > 0 && y > 0);

// ✅ 短路：第一个失败立即回滚
require(x > 0);
require(y > 0);
```

### 3. 拆分 revert 语句 ⭐⭐

```solidity
// ❌ 布尔运算
if (x < 10 || x > 20) {
    revert BadValue();
}

// ✅ 拆分：更精确的错误信息 + 短路
if (x < 10) revert TooLow();
if (x > 20) revert TooHigh();
```

### 4. 命名返回值 ⭐⭐

```solidity
// ❌ 匿名返回
function myFunc(uint256 x, uint256 y) external pure returns (uint256) {
    return x * y;
}

// ✅ 命名返回（通常更高效）
function myFunc(uint256 x, uint256 y) external pure returns (uint256 z) {
    z = x * y;
}
```

### 5. 反转否定条件 ⭐

```solidity
// ❌ 多一个 ! 操作
if (!condition) {
    action1();
} else {
    action2();
}

// ✅ 避免否定
if (condition) {
    action2();
} else {
    action1();
}
```

### 6. ++i vs i++ ⭐⭐⭐

```solidity
// ❌ i++ 返回旧值，需要存储两个值
for (uint256 i = 0; i < n; i++) { }

// ✅ ++i 返回新值，只需存储一个值
for (uint256 i = 0; i < n; ++i) { }
```

### 7. 使用 unchecked 块 ⭐⭐⭐

```solidity
// ✅ 最优化的 for 循环
for (uint256 i; i < limit; ) {
    // 循环体
    unchecked {
        ++i;
    }
}
```

**适用场景**:
- 有自然上限的 for 循环
- 输入已限制在合理范围内
- 计数器递增

### 8. do-while 循环 ⭐⭐

```solidity
// ❌ for 循环
for (uint256 i; i < times;) {
    unchecked { ++i; }
}

// ✅ do-while 更省 gas（即使加空检查）
if (times == 0) return;
uint256 i;
do {
    unchecked { ++i; }
} while (i < times);
```

---

## 二、类型与变量优化

### 9. 避免不必要的小类型转换 ⭐⭐

```solidity
// ❌ uint8 会被 EVM 转换为 uint256
uint8 public num;
function incrementNum() public {
    num += 1;  // 每次都有转换成本
}

// ✅ 直接用 uint256
uint256 public num;
function incrementNum() public {
    num += 1;  // 无转换
}
```

**原则**: 除非需要打包，否则用 uint256

### 10. 短路布尔运算 ⭐⭐

```solidity
// || : 第一个为 true 就跳过第二个
// && : 第一个为 false 就跳过第二个

// ✅ 把更可能通过的条件放前面
require(msg.sender == owner || msg.sender == manager);

// ✅ 把更可能失败的条件放前面（对于 &&）
require(hasBalance && isAllowed);
```

### 11. 避免不必要的 public 变量 ⭐

```solidity
// ❌ public 会生成隐式 getter 函数
uint256 public num;  // 增加字节码

// ✅ 需要时才设为 public
uint256 internal num;
```

---

## 三、优化器与函数选择

### 12. 优化器 runs 参数选择 ⭐⭐

| runs 值 | 优化目标 | 适用场景 |
|--------|---------|---------|
| 小值 (1-10) | 部署成本 | 一次性合约 |
| 默认 (200) | 平衡 | 常规合约 |
| 大值 (1000+) | 执行成本 | 高频调用 |

### 13. 高频函数用最佳名称 ⭐⭐

```solidity
// EVM 用跳转表查找函数，选择器小的先检查
// selector = 0xa0712d68
function mint(uint256 amount) public { }

// selector = 0x000071c3 (更便宜！)
function mint_184E17(uint256 amount) public { }
```

**技巧**: 高频函数用前导零多的选择器

---

## 四、算术与位运算

### 14. 位移比乘除更省 gas ⭐⭐

```solidity
// ✅ 位移
uint256 x = y << 1;  // y * 2
uint256 z = y >> 2;  // y / 4

// ❌ 乘除（有溢出检查）
uint256 x = y * 2;
uint256 z = y / 4;
```

| 操作 | Gas |
|-----|-----|
| shl/shr | 5 gas |
| mul/div | 3 gas + 溢出检查 |

**注意**: 位移无溢出检查，需自行确保安全

### 15. 缓存 calldata 有时更便宜 ⭐

```solidity
// 有时缓存 arr[i] 到局部变量更便宜
// 需要测试两种方案
```

---

## 五、高级技巧

### 16. 无分支算法 ⭐

```solidity
// 消除 JUMP 操作码
// 前面的 max 示例就是无分支算法
```

### 17. 循环展开 ⭐

```solidity
// 一次处理两个元素，跳转次数减半
for (uint256 i; i < len; ) {
    process(arr[i]);
    process(arr[i + 1]);
    unchecked { i += 2; }
}
```

### 18. 内联只调用一次的内部函数 ⭐

```solidity
// ❌ 内部函数引入跳转标签
function helper() internal { ... }

// ✅ 如果只调用一次，直接内联
```

### 19. 大数组/字符串用哈希比较 ⭐

```solidity
// > 32 字节时，哈希比遍历更便宜
bytes32 hash1 = keccak256(arr1);
bytes32 hash2 = keccak256(arr2);
if (hash1 == hash2) { }
```

### 20. 预编译合约 ⭐

对于大数模乘或内存复制，考虑使用以太坊预编译合约

---

## 六、面试快速记忆

### 最优 for 循环模板

```solidity
for (uint256 i; i < limit; ) {
    // 循环体
    unchecked {
        ++i;
    }
}
```

### 常见优化检查清单

- [ ] ++i 代替 i++
- [ ] unchecked 递增
- [ ] 拆分 require/revert
- [ ] 命名返回值
- [ ] 位移代替 2 的幂乘除
- [ ] 短路布尔运算
- [ ] 测试对比！

---

*整理人: 小鹅斯基 🦆*
