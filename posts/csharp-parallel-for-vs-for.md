---
title: C# 中 Parallel.For 与普通 For 的区别——从原理到实践
slug: csharp-parallel-for-vs-for
description: >-
  深入对比 C# 中 Parallel.For 与普通 for 循环的本质差异，涵盖执行模型、线程安全、性能瓶颈和实际使用场景。
tags:
  - technical
added: "May 29 2026"
---

## 引言

在处理大批量数据时，很多 .NET 开发者会面临一个选择：**用普通的 `for` 循环，还是用 `Parallel.For`？**

表面上看，它们做的事一样——遍历一组数据，对每个元素执行某个操作。但底层执行模型完全不同，选错了不仅不会加速，反而可能让性能雪崩。

本文将从执行模型、性能特征、线程安全和适用场景四个维度，把这个问题讲透。

---

## 一、普通 for 循环——单线程顺序执行

### 基本概念

```csharp
for (int i = 0; i < items.Length; i++)
{
    Process(items[i]);
}
```

这是最朴素的循环方式。它的核心特征是：

- **单线程执行**：所有迭代都在调用线程上依次运行
- **严格顺序**：第 i 次迭代一定在第 i+1 次之前完成
- **确定性**：每次运行，结果完全一致

### 执行示意

```
Thread-1:  [item0] → [item1] → [item2] → [item3] → ... → [itemN]
           └──────────────── 依次执行，全部串行 ────────────────┘
```

### 什么时候用它？

- 数据量小，操作轻量
- 迭代之间有**依赖关系**（后一项依赖前一项的结果）
- 需要**严格顺序**保证
- 操作本身涉及非线程安全的共享状态

---

## 二、Parallel.For——多线程并行执行

### 基本概念

```csharp
Parallel.For(0, items.Length, i =>
{
    Process(items[i]);
});
```

`Parallel.For` 是 TPL（Task Parallel Library）提供的并行循环 API。它的核心特征是：

- **多线程执行**：TPL 会将迭代划分成多个分区，分配到线程池的不同线程上
- **无序执行**：迭代之间没有固定的执行顺序
- **自动调度**：TPL 会根据 CPU 核心数、当前负载自动决定并行度

### 执行示意

```
Thread-1:  [item0] → [item1] → [item4] → [item5]
Thread-2:  [item2] → [item3] → [item6] → [item7]
Thread-3:  [item8] → [item9] → [itemA] → [itemB]
           └───── 同时进行，互不等待 ─────┘
```

### 底层机制

`Parallel.For` 的并行调度经历了几个阶段：

1. **分区（Partitioning）**：将迭代范围分成多个 chunk，分配给不同线程
2. **工作窃取（Work Stealing）**：空闲的线程可以从繁忙线程"偷"任务，实现负载均衡
3. **线程池复用**：使用 `ThreadPool` 中的线程，不会为每次调用创建新线程

---

## 三、核心区别对比

| 维度 | 普通 for | Parallel.For |
|------|----------|-------------|
| 线程数 | 1（调用线程） | 多个（线程池） |
| 执行顺序 | 严格顺序 | 无序 |
| 异常处理 | 同步抛出 | 聚合为 `AggregateException` |
| 返回值 | 无特殊返回 | 返回 `ParallelLoopResult` |
| 线程安全 | 天然安全 | 需要开发者保证 |
| 开销 | 几乎为零 | 分区、调度、同步有额外开销 |

---

## 四、性能对比——不是所有场景都快

很多人直觉认为"并行一定更快"，这是最大的误区。

### 场景一：CPU 密集型、大批量——Parallel.For 明显更快

```csharp
// 计算 1000 万个数的平方根
int n = 10_000_000;

// 普通 for：约 80ms（取决于 CPU）
var results = new double[n];
for (int i = 0; i < n; i++)
    results[i] = Math.Sqrt(i);

// Parallel.For：约 15ms（8 核机器）
Parallel.For(0, n, i =>
    results[i] = Math.Sqrt(i));
```

**原因**：计算任务被分摊到多个核心，总时间 ≈ 单核时间 / 核心数（理想情况）。

### 场景二：小数据量——Parallel.For 反而更慢

```csharp
// 只有 100 个元素，每个操作极轻
int n = 100;

// 普通 for：< 1μs
for (int i = 0; i < n; i++)
    results[i] = i * 2;

// Parallel.For：可能 > 50μs
Parallel.For(0, n, i =>
    results[i] = i * 2);
```

**原因**：并行调度的开销（分区、线程切换、同步）远大于任务本身。这就是所谓的"并行税"。

### 场景三：I/O 密集型——Parallel.For 可能帮倒忙

```csharp
// 每个迭代都要读文件或访问网络
Parallel.For(0, urls.Length, i =>
{
    var content = File.ReadAllText(urls[i]); // 或 HttpClient 请求
    Process(content);
});
```

**问题**：
- 线程池线程被 I/O 阻塞，无法执行其他任务
- 大量并行 I/O 请求可能导致目标服务过载
- 更好的选择是 `async/await` + `SemaphoreSlim` 控制并发度

---

## 五、线程安全——Parallel.For 的最大坑

普通 for 循环不需要考虑线程安全，但 Parallel.For 必须考虑。

### ❌ 错误示例：共享状态竞争

```csharp
int sum = 0;
Parallel.For(0, 1_000_000, i =>
{
    sum += i;  // 多线程同时读写 sum → 数据竞争！
});
// 结果不确定，大概率错误
```

### ✅ 正确做法 1：使用线程安全的累加

```csharp
long sum = 0;
Parallel.For(0, 1_000_000,
    () => 0L,                          // 线程本地初始化
    (i, state, local) => local + i,    // 线程本地累加
    local => Interlocked.Add(ref sum, local) // 合并结果
);
```

### ✅ 正确做法 2：使用 PLINQ（更简洁）

```csharp
long sum = Enumerable.Range(0, 1_000_000)
    .AsParallel()
    .Sum(i => (long)i);
```

### ✅ 正确做法 3：使用线程安全集合

```csharp
var results = new ConcurrentBag<string>();
Parallel.For(0, items.Length, i =>
{
    results.Add(Process(items[i]));
});
```

---

## 六、高级特性

### 1. 控制并行度

```csharp
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = 4  // 最多 4 个线程
};
Parallel.For(0, n, options, i => Process(i));
```

**适用场景**：
- 不想占满所有 CPU 核心
- 任务涉及 I/O，需要限制并发度
- 避免线程池饥饿

### 2. 提前终止

```csharp
Parallel.For(0, items.Length, (i, state) =>
{
    if (items[i] == target)
    {
        state.Stop();  // 终止循环
        return;
    }
    Process(items[i]);
});
```

### 3. 异常聚合

```csharp
try
{
    Parallel.For(0, n, i =>
    {
        if (items[i] < 0)
            throw new ArgumentException($"Invalid value at {i}");
    });
}
catch (AggregateException ex)
{
    // 所有迭代抛出的异常被聚合在这里
    foreach (var inner in ex.InnerExceptions)
        Console.WriteLine(inner.Message);
}
```

---

## 七、决策树——什么时候用哪个？

```
需要遍历数据
  │
  ├─ 数据量 < 1000？ → 用普通 for
  │
  ├─ 迭代之间有依赖？ → 用普通 for
  │
  ├─ 操作是 I/O 密集型？ → 用 async/await + SemaphoreSlim
  │
  ├─ 操作是 CPU 密集型，且数据量大？
  │     │
  │     ├─ 每个迭代独立，无共享状态？ → 用 Parallel.For ✅
  │     │
  │     └─ 有共享状态？ → 用 Parallel.For + 线程安全机制
  │
  └─ 需要 LINQ 风格的并行查询？ → 用 PLINQ (.AsParallel())
```

---

## 总结

| | 普通 for | Parallel.For |
|--|----------|-------------|
| 优势 | 简单、安全、无开销 | 多核加速，适合大批量 CPU 密集型任务 |
| 劣势 | 单核跑不满 | 线程安全要自己管，小数据量反而更慢 |
| 本质 | 顺序执行 | 任务并行 |

**一句话总结**：`Parallel.For` 不是"更快的 for"，而是"把 for 拆成多份同时跑"。用对了是利器，用错了是负担。关键判断标准只有一个——**每个迭代是否独立且计算量足够大**。
