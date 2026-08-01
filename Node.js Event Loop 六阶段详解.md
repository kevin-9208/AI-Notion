# Node.js Event Loop 六阶段详解

Node.js 的 Event Loop 和浏览器完全是两套设计哲学——浏览器只有"宏任务/微任务"两级简单模型，而 Node 是基于 **libuv** 实现的，有明确的**阶段（Phase）**划分，每个阶段维护自己的 FIFO 队列。这是理解 Node 异步模型的核心。

## 一、整体结构总览

```
   ┌───────────────────────────┐
┌─>│           timers           │  ← setTimeout / setInterval 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks      │  ← 系统级回调（如 TCP 错误 ECONNREFUSED）
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare        │  ← Node内部使用，开发者基本用不到
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐      ┌───────────────┐
│  │           poll             │<─────┤ 新的I/O事件到达 │
│  └─────────────┬─────────────┘      └───────────────┘
│  ┌─────────────┴─────────────┐
│  │           check            │  ← setImmediate 回调
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks       │  ← socket.on('close', ...)
   └───────────────────────────┘
```

每次进入下一阶段前，都会**清空当前阶段队列里的所有回调**（有数量上限保护，避免饿死其他阶段），然后再检查微任务队列。

## 二、逐阶段拆解

### 1. timers 阶段
执行 `setTimeout` 和 `setInterval` 注册的回调。

**关键点**：这个阶段只是检查"有没有定时器时间到了"，并不是精确的"到点立即执行"——Node 只保证**不早于**设定时间执行，实际执行时间取决于当前 Event Loop 走到哪个阶段、以及前面阶段任务是否耗时。这也是为什么 `setTimeout(fn, 0)` 实际测出来可能延迟到几毫秒甚至更久。

### 2. pending callbacks 阶段
处理上一轮循环中被推迟的系统级 I/O 回调，典型场景比如 TCP 连接出错时的回调（如 `ECONNREFUSED`）。这个阶段日常开发基本不会直接接触。

### 3. idle, prepare 阶段
纯 Node 内部使用，用户代码不会用到，跳过即可，面试提一句知道就行。

### 4. poll 阶段 —— 核心阶段，最复杂

这是整个 Event Loop 里**最关键**的阶段，做两件事：

1. **计算应该阻塞多久去等待 I/O 事件**（如果没有 timer 在排队）
2. **处理 poll 队列里的事件**：文件读写完成、网络请求响应回来等 I/O 回调都堆在这里执行

poll 阶段的具体行为分支：

- **如果 poll 队列非空**：同步执行队列里的所有回调，直到队列清空或者达到系统限制
- **如果 poll 队列为空**：
  - 如果有 `setImmediate` 回调在等待 → **立即结束 poll 阶段**，进入 check 阶段
  - 如果没有 `setImmediate`，但有 timer 在排队 → 检查最近的 timer 时间是否到了，**如果到了就回到 timers 阶段**
  - 如果都没有 → **阻塞等待新的 I/O 事件到来**（这就是 Node 进程"空闲挂起不退出"的原理）

### 5. check 阶段
专门执行 `setImmediate` 回调。

`setImmediate` 被设计成"**poll 阶段结束后立刻执行**"，这也是它和 `setTimeout(fn, 0)` 语义上的核心区别。

### 6. close callbacks 阶段
处理关闭类事件，比如 `socket.on('close', ...)`、`process.on('exit', ...)` 这类清理性质的回调。

## 三、面试高频题：setTimeout vs setImmediate 谁先执行？

```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

**答案：不确定**，两次运行结果可能不一样。

原因：这段代码是在**主模块顶层**运行的，此时 Event Loop 还没真正进入 timers 阶段，`setTimeout(fn, 0)` 实际会被内部拉到 1ms（Node 的最小定时器精度），所以两者谁先触发取决于**代码执行到这里时，系统时间是否已经超过 1ms 这个阈值**——存在竞态，是不确定的。

但如果放到 **I/O 回调内部**，结果就是**确定的**：

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});
```

**输出永远是：`immediate` 先于 `timeout`**

原因：`fs.readFile` 的回调是在 **poll 阶段**执行的，执行完后，poll 队列空了，此时按前面讲的分支逻辑——**先检查有没有 setImmediate**，有的话立刻进入 check 阶段执行它，然后才会**回到** timers 阶段处理 setTimeout。这是理解这道题的关键：不是"谁注册的快就先执行"，而是**当前所处的阶段决定了下一步去哪**。

## 四、process.nextTick —— 游离在六阶段之外的"插队王"

这是 Node 特有、浏览器没有的概念，很多人会跟微任务搞混，需要明确区分：

- `process.nextTick` 的回调**不属于任何一个 Phase**，它有自己独立的队列
- **优先级最高**：每当一个操作完成（不管是同步代码执行完，还是某个阶段的某个回调执行完），在**进入下一阶段之前**，会先把 `nextTickQueue` **清空**
- 甚至比 Promise 微任务还要早——准确顺序是：

```
process.nextTick 队列  >  Promise微任务队列  >  进入下一个 Phase
```

验证：

```javascript
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
```

**输出：`nextTick` 先于 `promise`**

**注意坑点**：如果在 `nextTick` 回调里递归调用 `process.nextTick`，会一直阻塞 Event Loop 进入下一阶段（因为规则是"清空"而不是"取一个"），这是 Node 早期版本一个真实的性能陷阱，官方文档专门提醒过度使用 `nextTick` 可能"饿死" I/O。

## 五、每个阶段之间都会检查微任务队列

这是最容易被漏讲的一点：**并不是整个 Event Loop 跑完一圈才处理微任务**，而是**每执行完一个回调**（不管是在哪个阶段），都会检查并清空一次 `nextTick队列 + Promise微任务队列`。

也就是说完整的执行粒度是：

```
执行一个宏任务回调
   ↓
清空 nextTick 队列
   ↓
清空 Promise 微任务队列
   ↓
执行同一阶段下一个回调（如果还有）
   ↓
... 直到该阶段队列清空，进入下一阶段
```

这也是为什么下面这段代码会交替打印：

```javascript
setTimeout(() => {
  console.log('timeout 1');
  process.nextTick(() => console.log('nextTick 1'));
}, 0);

setTimeout(() => {
  console.log('timeout 2');
  process.nextTick(() => console.log('nextTick 2'));
}, 0);
```

输出：`timeout 1 → nextTick 1 → timeout 2 → nextTick 2`（而不是两个 timeout 挨着打完再统一处理 nextTick）。

## 六、浏览器 Event Loop vs Node Event Loop 对比表（面试总结题常考）

| 维度 | 浏览器 | Node.js |
|---|---|---|
| 底层实现 | 各浏览器自行实现（V8+Blink等） | libuv |
| 任务分级 | 宏任务 / 微任务 两级 | 六个明确 Phase，每个有独立队列 |
| 微任务时机 | 每个宏任务执行完清空一次 | 每个回调执行完就清空一次（粒度更细） |
| 特殊高优先级任务 | 无 | `process.nextTick`（比Promise微任务还早） |
| 渲染 | 有专属的渲染时机（Layout/Paint） | 无渲染概念（后端环境） |
| setImmediate | 不存在 | check 阶段专属API |

## 七、一道综合面试真题（把所有知识点串起来）

```javascript
console.log('start');

setTimeout(() => console.log('timeout'), 0);

setImmediate(() => console.log('immediate'));

process.nextTick(() => console.log('nextTick'));

Promise.resolve().then(() => console.log('promise'));

console.log('end');
```

**输出：`start → end → nextTick → promise → timeout → immediate`**（timeout 和 immediate 顺序在顶层代码中不确定，这里按常见情况举例）

解析要点：
1. 同步代码先跑完：`start`、`end`
2. 同步代码执行完毕后，清空 `nextTick` 队列：`nextTick`
3. 再清空 Promise 微任务队列：`promise`
4. 进入 timers 阶段，`setTimeout` 到期执行：`timeout`
5. 走到 poll 阶段（此时无 I/O，队列空），发现有 `setImmediate` 排队，进入 check 阶段：`immediate`

---
