# Gas 汇编优化技巧 - 面试笔记

> **出处**: [Gas 优化手册（七）| 登链社区](https://learnblockchain.cn/article/22697)
> **整理时间**: 2026-03-17
> **用途**: DApp 前端开发面试准备

---

## ⚠️ 重要提醒

> **不要假设汇编一定更高效！始终测试非汇编版本进行对比。**

---

## 一、常用汇编优化技巧

### 1. 汇编回滚带错误消息 ⭐⭐

```solidity
// ❌ Solidity require: 24,042 gas
contract SolidityRevert {
    function restrictedAction(uint256 num) external {
        require(msg.sender == owner, "caller is not owner");
        specialNumber = num;
    }
}

// ✅ 汇编 revert: 23,734 gas (省 300+ gas)
contract AssemblyRevert {
    function restrictedAction(uint256 num) external {
        assembly {
            if sub(caller(), sload(owner.slot)) {
                mstore(0x00, 0x20)
                mstore(0x20, 0x13)  // 消息长度
                mstore(0x40, 0x63616c6c6572206973206e6f74206f776e657200...)
                revert(0x00, 0x60)
            }
        }
        specialNumber = num;
    }
}
```

### 2. 汇编重用内存数据 ⭐⭐

```solidity
// ❌ Solidity: 30,570 gas
contract Sol {
    function set(address addr, uint256 num) external {
        Callme(addr).setNum(num);
    }
}

// ✅ 汇编: 30,350 gas (省 220 gas)
contract Assembly {
    function set(address addr, uint256 num) external {
        assembly {
            mstore(0x00, hex"cd16ecbf")  // 函数选择器
            mstore(0x04, num)
            
            // 检查地址是否有代码
            if iszero(extcodesize(addr)) {
                revert(0x00, 0x00)
            }
            
            let success := call(gas(), addr, 0x00, 0x00, 0x24, 0x00, 0x00)
            if iszero(success) {
                revert(0x00, 0x00)
            }
        }
    }
}
```

### 3. 数学运算优化 (max/min) ⭐⭐

```solidity
// ❌ 三元运算符
function max(uint256 x, uint256 y) public pure returns (uint256 z) {
    z = x > y ? x : y;  // 包含条件跳转，昂贵
}

// ✅ 汇编无分支 (来自 Solady)
function max(uint256 x, uint256 y) public pure returns (uint256 z) {
    assembly {
        z := xor(x, mul(xor(x, y), gt(y, x)))
    }
}
```

### 4. 相等性比较 ⭐

```solidity
// ✅ 有时比 EQ 更高效
if sub(caller, sload(owner.slot)) {
    revert(0x00, 0x00)
}

// XOR 也可以，但注意位翻转问题
```

### 5. 地址零检查 ⭐⭐

```solidity
// ❌ Solidity: 较高
function check(address _caller) public pure returns (bool) {
    require(_caller != address(0x00), "Zero address");
    return true;
}

// ✅ 汇编: 省约 90 gas
function checkOptimized(address _caller) public pure returns (bool) {
    assembly {
        if iszero(_caller) {
            mstore(0x00, 0x20)
            mstore(0x20, 0x0c)
            mstore(0x40, 0x5a65726f204164647265737300...)
            revert(0x00, 0x60)
        }
    }
    return true;
}
```

### 6. selfbalance vs address(this).balance ⭐

```solidity
// 有时 selfbalance() 更高效
assembly {
    let bal := selfbalance()
}

// 但编译器可能已优化，需测试
```

---

## 二、内存布局与优化

### Solidity 内存布局

| 偏移 | 用途 | 大小 |
|-----|------|------|
| 0x00-0x40 | 临时空间 | 64 字节 |
| 0x40-0x60 | 自由内存指针 | 32 字节 |
| 0x60-0x80 | 零槽 | 32 字节 |

### 7. 汇编处理小数据 (≤96 字节) ⭐⭐

```solidity
// ❌ Solidity: 22,790 gas
function returnBlockData() external {
    emit BlockData(block.timestamp, block.number, block.gaslimit);
}

// ✅ 汇编: 26,145 gas (省 ~2000 gas)
function returnBlockData() external {
    assembly {
        mstore(0x00, timestamp())
        mstore(0x20, number())
        mstore(0x40, gaslimit())
        log1(0x00, 0x60, 0x9ae98f19...)  // event hash
    }
}
```

### 8. 多外部调用重用内存 ⭐⭐

```solidity
// ❌ Solidity: 7,262 gas
// ✅ 汇编重用内存: 5,281 gas (省 ~2000 gas)
```

**技巧**: 使用临时空间存储参数，第二次调用重用同一内存

### 9. 创建合约重用内存 ⭐

```solidity
// ✅ 汇编创建合约: 省 ~1000 gas
assembly {
    let called1 := create(0x00, add(0x20, creationCode), mload(creationCode))
    let called2 := create(0x00, add(0x20, creationCode), mload(creationCode))
}
```

---

## 三、其他技巧

### 10. 奇偶数检查 ⭐

```solidity
// ❌ 模运算
if (x % 2 == 0) { }  // 昂贵

// ✅ 位运算
if (x & uint256(1) == 0) { }  // 便宜
```

---

## 四、面试要点

### Q1: 为什么汇编能省 Gas？

**回答**:
- 直接操作内存，避免 Solidity 编译器的额外检查
- 使用更少的操作码
- 避免条件跳转（无分支算法）

### Q2: 使用汇编有什么风险？

**回答**:
- 可读性差，难维护
- 容易引入安全漏洞
- 编译器优化可能已处理某些场景
- 需要深入理解 EVM 内存布局

### Q3: 什么时候应该用汇编？

**回答**:
- Gas 极端敏感的热点代码
- 简单操作（如检查、比较）
- 团队有汇编经验
- 始终测试对比非汇编版本

---

*整理人: 小鹅斯基 🦆*
