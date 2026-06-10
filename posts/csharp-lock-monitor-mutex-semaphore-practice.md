---
title: C# 中 lock、Monitor、Mutex、Semaphore 的区别与实战——多线程同步机制全面解析
slug: csharp-lock-monitor-mutex-semaphore-practice
description: >-
  深入对比 C# 中四种常用的线程同步原语：lock、Monitor、Mutex 和 Semaphore，从底层原理、性能差异到实际场景，帮你选对工具。
tags:
  - technical
added: "June 10 2026"
---

## 引言

多线程开发中，"谁先谁后"是一个永恒的问题。多个线程同时访问共享资源，不加控制就会出现数据竞争、状态不一致、甚至程序崩溃。

.NET 提供了多种同步机制：`lock`、`Monitor`、`Mutex`、`Semaphore`……它们看起来都能"锁"，但底层原理、适用场景和性能差异天差地别。

> 选错同步工具，轻则性能瓶颈，重则死锁崩溃。

本文将从底层实现到实战场景，把这四个同步原语讲透。

---

## 一、lock——最基础的语法糖

### 本质

`lock` 不是 CLR 原生关键字，它是 `Monitor.Enter / Monitor.Exit` 的 **C# 语法糖**。编译器会把它翻译成带有 `try/finally` 保护的 Monitor 调用。

```csharp
// 你写的
lock (obj)
{
    // 临界区
}

// 编译器实际生成的
bool acquired = false;
try
{
    Monitor.Enter(obj, ref acquired);
    // 临界区
}
finally
{
    if (acquired) Monitor.Exit(obj);
}
```

这个 `try/finally` 保证了即使临界区抛出异常，锁也会被正确释放。这是 `lock` 最重要的价值。

### 适用场景

- **进程内线程同步**（最常见）
- 保护简单的共享资源访问
- 需要轻量级、高性能的锁

### 示例：线程安全的计数器

```csharp
public class Counter
{
    private readonly object _lockObj = new();
    private int _count;

    public void Increment()
    {
        lock (_lockObj)
        {
            _count++;
        }
    }

    public int GetCount()
    {
        lock (_lockObj)
        {
            return _count;
        }
    }
}
```

### ⚠️ 注意事项

1. **永远不要 lock 值类型**——值类型装箱后每次产生新对象，锁不住
2. **不要 lock `this`**——外部代码也可能 lock 同一个对象，容易死锁
3. **不要 lock `typeof(Type)`**——类型对象是全局共享的，跨 AppDomain 都有效
4. **推荐使用 `private readonly object _lockObj = new();`** 作为锁对象

### 性能特征

`lock` 基于 **SpinLock + 内核对象** 的混合模式：

1. 线程尝试获取锁时，先在用户态自旋一小段时间（避免内核切换开销）
2. 自旋失败后，才挂起线程进入内核态等待

这种混合策略让它在竞争不激烈时性能极好。

---

## 二、Monitor——lock 的完全体

### 比 lock 多了什么

`Monitor` 是 `lock` 的底层实现，提供了 `lock` 无法实现的三个能力：

1. **`TryEnter`**——带超时的尝试获取，获取失败不会无限阻塞
2. **`Wait / Pulse / PulseAll`**——线程间条件等待和唤醒
3. **精细控制**——手动 Enter/Exit，适合复杂场景

### 生产者-消费者模式（经典 Wait/Pulse）

```csharp
public class ProducerConsumerQueue<T>
{
    private readonly Queue<T> _queue = new();
    private readonly object _lock = new();
    private bool _completed;

    public void Enqueue(T item)
    {
        lock (_lock)
        {
            _queue.Enqueue(item);
            Monitor.Pulse(_lock);  // 唤醒一个等待的消费者
        }
    }

    public T Dequeue()
    {
        lock (_lock)
        {
            while (_queue.Count == 0 && !_completed)
            {
                Monitor.Wait(_lock);  // 释放锁并等待，被 Pulse 唤醒后重新获取锁
            }

            if (_queue.Count > 0)
            {
                return _queue.Dequeue();
            }

            throw new InvalidOperationException("Queue is completed.");
        }
    }

    public void Complete()
    {
        lock (_lock)
        {
            _completed = true;
            Monitor.PulseAll(_lock);  // 唤醒所有等待的消费者
        }
    }
}
```

这个模式的关键点：

- `Monitor.Wait` 会**释放锁**并让线程进入等待状态
- 被 `Pulse` 唤醒后，线程会**自动重新获取锁**
- `while` 循环是必须的——被唤醒不代表条件一定满足（可能有多个消费者竞争）

### TryEnter 超时控制

```csharp
bool lockTaken = false;
try
{
    Monitor.TryEnter(_lock, TimeSpan.FromSeconds(2), ref lockTaken);
    if (lockTaken)
    {
        // 获取锁成功，执行业务
    }
    else
    {
        // 超时，记录日志或走降级逻辑
        _logger.LogWarning("Failed to acquire lock within 2 seconds");
    }
}
finally
{
    if (lockTaken) Monitor.Exit(_lock);
}
```

这个在实际项目中非常实用：当某个外部资源响应缓慢时，你不想让线程无限期阻塞，TryEnter 给了你优雅降级的机会。

### 性能特征

和 `lock` 一样——用户态自旋 + 内核态等待的混合模式。两者在本质上是同一个东西，`lock` 只是语法糖。

---

## 三、Mutex——跨进程的重量级锁

### 核心特性

`Mutex`（互斥体）和 `Monitor` 最大的区别：

| 维度 | Monitor/lock | Mutex |
|------|-------------|-------|
| 跨进程 | ❌ 不行 | ✅ 可以 |
| 跨 AppDomain | ❌ 不行 | ✅ 可以 |
| 系统级命名 | ❌ 不行 | ✅ 支持具名 Mutex |
| 性能 | 快（混合模式） | 慢（纯内核对象） |
| 异常安全 | try/finally 保证 | 需要手动 ReleaseMutex |

### 场景一：确保程序只运行一个实例

```csharp
class Program
{
    private static Mutex? _mutex;

    static void Main(string[] args)
    {
        bool createdNew;
        _mutex = new Mutex(true, "MyApp_SingleInstance_Mutex", out createdNew);

        if (!createdNew)
        {
            Console.WriteLine("程序已在运行中！");
            return;
        }

        try
        {
            // 正常启动程序
            RunApplication();
        }
        finally
        {
            _mutex.ReleaseMutex();
        }
    }
}
```

具名 Mutex（带名字的那个构造函数）是跨进程可见的。如果另一个进程创建了同名的 Mutex，`createdNew` 就会返回 `false`。

### 场景二：跨进程资源保护

```csharp
public class SharedFileWriter
{
    private readonly Mutex _mutex;

    public SharedFileWriter()
    {
        // 全局命名的 Mutex，所有进程都能看见
        _mutex = new Mutex(false, "Global\\SharedFileWriter_Lock");
    }

    public void Write(string content)
    {
        _mutex.WaitOne();  // 等待获取锁
        try
        {
            File.AppendAllText("shared.log", content);
        }
        finally
        {
            _mutex.ReleaseMutex();
        }
    }
}
```

### ⚠️ 重要警告

1. **Mutex 性能比 lock 慢 10-50 倍**——因为它总是涉及内核态切换，没有自旋优化
2. **必须手动 ReleaseMutex**——没有语法糖保护，异常时容易泄露
3. **不要在同进程中递归获取**——Mutex 支持递归（同一线程可以多次 WaitOne），但这通常是设计问题的信号
4. **全局命名空间**——`Global\` 前缀让 Mutex 在所有用户会话间共享，`Local\`（默认）仅在当前会话内

### 性能对比实测

```csharp
// 百万次锁获取对比（仅供参考，不同机器有差异）
// lock/Monitor: ~30ms
// Mutex:        ~1500ms（约 50 倍差距）
```

---

## 四、Semaphore——控制并发数量

### 核心概念

`lock`、`Monitor`、`Mutex` 都是"一次只允许一个线程进入"——它们的容量是 1。

`Semaphore`（信号量）的核心思想是：**允许 N 个线程同时进入**。

把它想象成一个停车场：

- 容量 = 车位数量
- 每进来一辆车，空位减一
- 每开走一辆车，空位加一
- 没车位了，后来者只能在外面等

### 构造函数

```csharp
Semaphore(int initialCount, int maximumCount)
SemaphoreSlim(int initialCount, int maximumCount)
```

- `initialCount`：初始可用许可数
- `maximumCount`：最大许可数

### Semaphore vs SemaphoreSlim

.NET 提供了两个版本：

| 维度 | Semaphore | SemaphoreSlim |
|------|-----------|---------------|
| 跨进程 | ✅ | ❌ |
| 性能 | 慢（内核对象） | 快（混合模式，类似 Monitor） |
| 异步等待 | ❌ | ✅（WaitAsync） |
| 推荐场景 | 跨进程限流 | 进程内限流 |

**99% 的场景都应该用 `SemaphoreSlim`**。

### 场景一：连接池限流

```csharp
public class DbConnectionPool
{
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DbConnection> _pool;

    public DbConnectionPool(int maxConnections)
    {
        _semaphore = new SemaphoreSlim(maxConnections, maxConnections);
        _pool = new Queue<DbConnection>();

        for (int i = 0; i < maxConnections; i++)
        {
            _pool.Enqueue(new DbConnection());
        }
    }

    public async Task<DbConnection> GetConnectionAsync()
    {
        await _semaphore.WaitAsync();  // 异步等待，不阻塞线程
        return _pool.Dequeue();
    }

    public async Task ReturnConnectionAsync(DbConnection conn)
    {
        _pool.Enqueue(conn);
        _semaphore.Release();  // 释放一个许可
    }
}
```

这里用 `WaitAsync` 是关键——在 ASP.NET Core 等异步优先的环境中，阻塞线程是昂贵的。`SemaphoreSlim.WaitAsync` 不会占用线程池线程。

### 场景二：并发请求限流

```csharp
public class ApiClient
{
    private readonly HttpClient _http;
    private readonly SemaphoreSlim _semaphore;

    public ApiClient(int maxConcurrentRequests)
    {
        _http = new HttpClient();
        _semaphore = new SemaphoreSlim(maxConcurrentRequests, maxConcurrentRequests);
    }

    public async Task<string> FetchAsync(string url)
    {
        await _semaphore.WaitAsync();
        try
        {
            var response = await _http.GetAsync(url);
            return await response.Content.ReadAsStringAsync();
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### 场景三：任务节流（生产者-消费者变形）

```csharp
public class TaskThrottle<T>
{
    private readonly SemaphoreSlim _semaphore;
    private readonly Func<T, Task> _handler;

    public TaskThrottle(int maxDegreeOfParallelism, Func<T, Task> handler)
    {
        _semaphore = new SemaphoreSlim(maxDegreeOfParallelism, maxDegreeOfParallelism);
        _handler = handler;
    }

    public async Task SubmitAsync(T item)
    {
        await _semaphore.WaitAsync();

        // 用 _ = 触发不等待，但 handler 内部完成后释放许可
        _ = ProcessAsync(item);
    }

    private async Task ProcessAsync(T item)
    {
        try
        {
            await _handler(item);
        }
        finally
        {
            _semaphore.Release();
        }
    }
}
```

### ⚠️ 注意事项

1. **Release 次数不能超过 maximumCount**——会抛 `SemaphoreFullException`
2. **SemaphoreSlim 不支持跨进程**——它纯粹是进程内的
3. **异步场景优先用 SemaphoreSlim**——只有它有 `WaitAsync`
4. **不要忘记 Release**——和 Mutex 一样，异常时需要 finally 保护

---

## 五、横向对比总结

### 选型决策树

```
需要跨进程同步？
├─ 是 → 只用一次？ → Mutex
│            └─ 允许多个？ → Semaphore（具名）
│
└─ 否 → 允许多个线程同时进入？
         ├─ 是 → 需要异步等待？ → SemaphoreSlim
         │                   └─ 不需要 → SemaphoreSlim
         │
         └─ 否 → 需要 Wait/Pulse？ → Monitor
                            └─ 不需要 → lock
```

### 速查表

| 原语 | 作用域 | 容量 | 性能 | 异步支持 | 推荐场景 |
|------|--------|------|------|----------|----------|
| `lock` | 进程内 | 1 | ⭐⭐⭐⭐⭐ | ❌ | 最基础的线程安全 |
| `Monitor` | 进程内 | 1 | ⭐⭐⭐⭐⭐ | ❌ | 条件等待、超时控制 |
| `Mutex` | 跨进程 | 1 | ⭐⭐ | ❌ | 单实例、跨进程互斥 |
| `Semaphore` | 跨进程 | N | ⭐⭐ | ❌ | 跨进程限流 |
| `SemaphoreSlim` | 进程内 | N | ⭐⭐⭐⭐ | ✅ | 连接池、并发限流 |

### 性能排序（从快到慢）

```
lock ≈ Monitor >> SemaphoreSlim >> Mutex ≈ Semaphore
```

### 常见组合模式

实际项目中，这些原语经常组合使用：

```csharp
// Monitor + 条件变量：经典的 Wait/Pulse 模式
lock (_lock) {
    while (!condition) Monitor.Wait(_lock);
    // 执行业务
    Monitor.Pulse(_lock);
}

// SemaphoreSlim + lock：限流 + 精细保护
await _semaphore.WaitAsync();
try
{
    lock (_fineLock) {
        // 临界区
    }
}
finally
{
    _semaphore.Release();
}
```

---

## 六、实战踩坑指南

### 坑一：lock 住了异步方法

```csharp
// ❌ 错误——lock 不能包含 await
lock (_lock)
{
    var result = await _service.GetDataAsync();  // 编译报错！
    _cache[key] = result;
}

// ✅ 正确——用 SemaphoreSlim
await _semaphore.WaitAsync();
try
{
    var result = await _service.GetDataAsync();
    _cache[key] = result;
}
finally
{
    _semaphore.Release();
}
```

`lock` 语句块内不能有 `await`，因为 await 可能在不同线程上恢复，导致 Exit 在错误的线程上调用。

### 坑二：锁粒度太粗

```csharp
// ❌ 把耗时的 I/O 操作也锁住了
lock (_lock)
{
    var data = _cache.GetOrAdd(key);  // 快
    var result = await _externalApi.CallAsync(data);  // 慢！别锁这个
    _cache[key] = result;
}

// ✅ 只锁共享资源访问
lock (_lock)
{
    data = _cache.GetOrAdd(key);
}
var result = await _externalApi.CallAsync(data);  // 锁外执行
lock (_lock)
{
    _cache[key] = result;
}
```

### 坑三：嵌套锁导致死锁

```csharp
// ❌ 线程 A：lock(A) → lock(B)
//    线程 B：lock(B) → lock(A)
//    → 死锁！

// ✅ 始终按固定顺序获取锁
// 约定：总是先 lock A，再 lock B
```

### 坑四：Mutex 忘记释放

```csharp
// ❌ 异常时 Mutex 不会释放
_mutex.WaitOne();
DoWork();           // 如果这里抛异常...
_mutex.ReleaseMutex();  // 永远执行不到

// ✅ 必须用 try/finally
_mutex.WaitOne();
try
{
    DoWork();
}
finally
{
    _mutex.ReleaseMutex();
}
```

---

## 结语

四个同步原语，本质上是两个维度的组合：

- **作用域**：进程内 vs 跨进程
- **容量**：1（互斥） vs N（限流）

选型原则很简单：

1. 能用 `lock` 就用 `lock`——最简单、最安全
2. 需要 Wait/Pulse 或超时 → `Monitor`
3. 需要跨进程 → `Mutex`
4. 需要控制并发数 → `SemaphoreSlim`（进程内）或 `Semaphore`（跨进程）
5. 异步代码中需要锁 → `SemaphoreSlim` + `WaitAsync`

同步机制的正确使用，是写出健壮多线程程序的基石。选对工具，少走弯路。

---

> **延伸阅读：**
> - [C# 中 Parallel.For 与普通 For 的区别](/posts/csharp-parallel-for-vs-for)
> - [.NET 中的 async/await 原理](/posts/async-await-vs-wait-result-getawaiter-getresult)
> - [TickerQ——.NET 后台任务调度器](/posts/tickerq-dotnet-background-task-scheduler)
