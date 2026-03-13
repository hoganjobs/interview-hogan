# Day 6：传统前端补漏

## 学习目标

- [ ] 复习 JavaScript 核心概念
- [ ] 复习 React Hooks 和性能优化
- [ ] 查漏补缺，巩固基础

---

## 学习资料

- [ ] `Interview-Front/前端面试题/3.Javascript题目.pdf`
- [ ] `Interview-Front/前端面试题/6.React 生态题目.pdf`

---

## 学习内容

### JavaScript 核心概念（P0 必会）

| 概念 | 说明 | 面试频度 |
|------|------|----------|
| **闭包** | 函数+词法环境 | ⭐⭐⭐ |
| **原型链** | 继承机制 | ⭐⭐⭐ |
| **this 指向** | 运行时绑定 | ⭐⭐⭐ |
| **Promise/async-await** | 异步编程 | ⭐⭐⭐ |
| **Event Loop** | 执行机制 | ⭐⭐ |
| **深浅拷贝** | 数据复制 | ⭐⭐ |
| **防抖节流** | 性能优化 | ⭐⭐ |

### React 核心概念（P0 必会）

| 概念 | 说明 | 面试频度 |
|------|------|----------|
| **useState 原理** | 状态管理 | ⭐⭐⭐ |
| **useEffect 原理** | 副作用处理 | ⭐⭐⭐ |
| **useCallback** | 函数缓存 | ⭐⭐ |
| **useMemo** | 值缓存 | ⭐⭐ |
| **虚拟 DOM** | 性能优化基础 | ⭐⭐⭐ |
| **diff 算法** | 更新优化 | ⭐⭐ |
| **React.memo** | 组件缓存 | ⭐⭐ |

---

## 必会知识点

### JavaScript

#### 1. 闭包

```javascript
// 经典闭包
function createCounter() {
    let count = 0;
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count
    };
}

// 应用场景
// 1. 数据私有化
// 2. 函数柯里化
// 3. 模块模式
```

#### 2. this 指向

```javascript
// 规则：谁调用指向谁
const obj = {
    name: 'test',
    fn() {
        console.log(this.name);  // this = obj
    },
    fn2: () => {
        console.log(this.name);  // this = 外部作用域
    }
};

// 箭头函数没有自己的 this
// call/apply/bind 可以改变 this
```

#### 3. Event Loop

```
┌─────────────────────┐
│   Stack (调用栈)      │  同步代码
├─────────────────────┤
│ Microtask Queue     │  Promise.then/queueMicrotask
│ (微任务队列)         │  优先级高，每轮都清空
├─────────────────────┤
│ Macrotask Queue     │  setTimeout/setInterval
│ (宏任务队列)         │  优先级低，每轮取一个
└─────────────────────┘

执行顺序：同步 → 微任务 → 宏任务
```

### React

#### 1. Hooks 原理

```javascript
// useState 简化实现
let state = [];
let index = 0;

function useState(initialValue) {
    const currentIndex = index;
    state[currentIndex] = state[currentIndex] ?? initialValue;

    const setState = (newValue) => {
        state[currentIndex] = newValue;
        render();  // 触发重新渲染
    };

    index++;
    return [state[currentIndex], setState];
}
```

#### 2. useCallback vs useMemo

```javascript
// useCallback: 缓存函数
const memoizedFn = useCallback(() => {
    doSomething(a, b);
}, [a, b]);

// useMemo: 缓存值
const memoizedValue = useMemo(() => {
    return computeExpensive(a, b);
}, [a, b]);

// 使用场景
// useCallback: 传给子组件的函数
// useMemo: 计算成本高的值
```

#### 3. React.memo 性能优化

```javascript
// React.memo: 浅比较 props
const MyComponent = React.memo(({ name, age }) => {
    return <div>{name}: {age}</div>;
});

// 自定义比较
const MyComponent = React.memo(({ user }) => {
    return <div>{user.name}</div>;
}, (prevProps, nextProps) => {
    // 返回 true 表示跳过渲染
    return prevProps.user.id === nextProps.user.id;
});
```

---

## 常见面试题

### JavaScript

#### Q1: 什么是闭包？有什么应用场景？

**答**：
- 闭包 = 函数 + 函数声明时的词法环境
- 特点：内部函数可以访问外部函数的变量
- 应用：数据私有化、函数柯里化、模块模式

#### Q2: Event Loop 执行顺序？

**答**：
```
同步代码 → 微任务队列 → 宏任务队列 → 循环
```

#### Q3: 深拷贝和浅拷贝的区别？

**答**：
- 浅拷贝：只复制一层，嵌套对象仍是引用
- 深拷贝：递归复制所有层级
- 实现：`JSON.parse(JSON.stringify())` 或 `structuredClone`

### React

#### Q1: useEffect 的依赖数组为空时是什么意思？

**答**：
- 只在组件挂载时执行一次
- 相当于 `componentDidMount`

#### Q2: 为什么 React 需要虚拟 DOM？

**答**：
- 批量更新，减少真实 DOM 操作
- diff 算法找出最小变化
- 跨平台（React Native 等）

#### Q3: useCallback 和 useMemo 的区别？

**答**：
- `useCallback` 缓存函数，返回函数
- `useMemo` 缓存计算结果，返回值
- `useCallback(fn, deps)` ≈ `useMemo(() => fn, deps)`

---

## 自测题

### JavaScript
- [ ] 能解释闭包的实际应用场景
- [ ] 知道 this 的绑定规则
- [ ] 能说明 Event Loop 执行顺序
- [ ] 知道深浅拷贝的区别

### React
- [ ] 能说明 useState 的原理
- [ ] 知道 useEffect 的使用场景
- [ ] 能区分 useCallback 和 useMemo
- [ ] 知道 React 性能优化方式

---

## 进度检查

- [ ] 已复习 JavaScript 核心概念
- [ ] 已复习 React 核心概念
- [ ] 已完成自测题

---

## 明天预告

明天是最后一天，**真题冲刺 + 模拟面试**，二刷所有真题，练习表达和思路。
