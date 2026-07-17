---
title: 基于 System.Threading.Channels 的工业 PLC 高并发采集背压限流方案
slug: system-threading-channels-plc-backpressure
description: >-
  深入探讨如何利用 .NET 的 System.Threading.Channels 构建工业 PLC 高并发数据采集场景下的背压限流方案，解决生产者-消费者速度不匹配、内存爆炸和服务雪崩等典型问题。
tags:
  - technical
added: "July 17 2026"
---

## 引言

工业物联网（IIoT）场景下，PLC（可编程逻辑控制器）数据采集是一个绕不开的核心环节。一条产线动辄几十上百个 PLC 点位，采集频率可能从 100ms 到 5s 不等，叠加后的并发请求量轻松达到每秒数万甚至数十万级别。

在这种高并发、高吞吐场景下，一个典型的架构问题是：**采集端（生产者）产生数据的速度远超消费端（存储、计算、报警处理等）的处理能力**。一旦处理出现瓶颈，内存飙升、服务雪崩、数据丢失接踵而至。

本文提出一种基于 .NET `System.Threading.Channels` 的背压限流方案，在不引入额外中间件依赖的前提下，用纯托管代码优雅地解决这个痛点。

---

## 一、高并发 PLC 采集的典型痛点

### 1.1 场景描述

假设一条产线有 200 个 PLC，每个 PLC 每 500ms 采集一次，每次采集产生 10 个 tag 值：

```
200 PLC × 120 次/分钟 × 10 tag = 240,000 tag/min ≈ 4,000 tag/s
```

如果采集程序需要对这些数据进行持久化、阈值判断、实时计算等操作，消费端常常跟不上。

### 1.2 传统方案的缺陷

| 方案 | 问题 |
|------|------|
| 同步阻塞写数据库 | 采集线程被 IO 阻塞，拖慢采集节拍，导致采集窗口偏差 |
| `ConcurrentQueue<T>` + 消费者线程 | 无界增长，消费者跟不上时内存爆炸 |
| `BlockingCollection<T>` | 界限有限，但 API 老旧，缺乏异步支持 |
| 每数据直接 `Task.Run` | 线程池饥饿，上下文切换开销巨大 |
| 外部消息队列（RabbitMQ/Kafka） | 架构复杂度高，非所有场景都需要引入中间件 |

核心矛盾就一句话：**生产者不能等，消费者跟不上，内存不能无限涨。**

---

## 二、System.Threading.Channels 简介

`System.Threading.Channels` 是 .NET Core 3.0 起引入的一个高性能异步生产者-消费者管道库。它本质上是线程安全的、支持异步的生产者-消费者队列，但设计理念远超普通队列。

### 2.1 核心概念

- **Channel<T>**：一个可读可写的管道
- **ChannelWriter<T>**：生产者写入端
- **ChannelReader<T>**：消费者读取端
- **BoundedChannel**：有容量上限的通道
- **UnboundedChannel**：无容量上限的通道

### 2.2 为什么选它？

```csharp
// 创建有界通道，容量 4096
var channel = Channel.CreateBounded<TagData>(new BoundedChannelOptions(4096)
{
    FullMode = BoundedChannelFullMode.Wait,      // 满了就让生产者等待
    SingleWriter = false,                         // 多生产者
    SingleReader = false                          // 多消费者
});
```

对比 `BlockingCollection`，Channels 的显著优势：
- **完全异步 API**：`WriteAsync` / `ReadAsync`，不阻塞线程
- **丰富的满队列策略**：Wait / DropWrite / DropNewest / DropOldest
- **背压原生支持**：满时让生产者自动慢下来
- **高性能**：内部基于无锁数据结构，GC 友好

---

## 三、背压限流方案设计

### 3.1 整体架构

```
┌─────────────────────────┐
│   PLC 采集任务/定时器    │  ← 生产者 × N
│   ChannelWriter.WriteAsync
└─────────┬───────────────┘
          │
          ▼
┌─────────────────────────┐
│  BoundedChannel<TagData> │  ← 有界通道，容量可配置
│    FullMode = Wait       │    背压在这里发生
└─────────┬───────────────┘
          │
          ▼
┌─────────────────────────┐
│  消费 Worker × M         │  ← 多消费者并行处理
│   ChannelReader.ReadAsync │     批量写入/计算
└─────────────────────────┘
```

### 3.2 关键设计点

1. **BoundedChannel** + **FullMode.Wait**：当通道满时，`WriteAsync` 会自然等待，不消耗 CPU，不增长内存
2. **可配置容量**：容量 = 允许的瞬时积压数据量，决定内存上界
3. **多消费者并行**：消费端用多个 `Task` 从 `ChannelReader` 拉取数据
4. **背压自动传导**：消费变慢 → 通道积压 → 通道满 → 生产者等待 → 采集节拍自然减慢

### 3.3 优雅降级 vs 数据丢失

如果业务要求宁可丢数据也不能延迟采集（比如高速振动信号），可以采用 `FullMode = BoundedChannelFullMode.DropOldest`：

```
容量满了 → 丢弃最旧的数据 → 保证最新数据的实时性
```

这样就变成了一个有损的滑动窗口——用数据精度换实时性。

---

## 四、完整代码实现

### 4.1 定义数据模型

```csharp
public readonly record struct TagData(
    string PlcId,
    string TagName,
    object Value,
    DateTime Timestamp
);
```

用 `readonly record struct` 减少 GC 压力——高吞吐场景下，引用类型的分配开销不可忽视。

### 4.2 采集管理器

```csharp
public sealed class PlcCollector : IAsyncDisposable
{
    private readonly Channel<TagData> _channel;
    private readonly List<Task> _workers;
    private readonly CancellationTokenSource _cts;
    private readonly int _channelCapacity;

    public PlcCollector(int channelCapacity = 8192, int consumerCount = 4)
    {
        _channelCapacity = channelCapacity;
        _cts = new CancellationTokenSource();

        _channel = Channel.CreateBounded<TagData>(
            new BoundedChannelOptions(channelCapacity)
            {
                FullMode = BoundedChannelFullMode.Wait,
                SingleWriter = false,
                SingleReader = false,
                AllowSynchronousContinuations = false,
            });

        _workers = Enumerable.Range(0, consumerCount)
            .Select(_ => Task.Run(() => ConsumeLoopAsync(_cts.Token)))
            .ToList();
    }
```

### 4.3 生产者——PLC 采集

```csharp
    /// <summary>
    /// 采集一个 PLC 的点位数据并写入通道。
    /// 如果通道满了，WriteAsync 会自然等待——这就是背压。
    /// </summary>
    public async ValueTask CollectAsync(
        string plcId,
        IReadOnlyList<(string TagName, object Value)> tags,
        CancellationToken ct = default)
    {
        var now = DateTime.UtcNow;

        foreach (var (tagName, value) in tags)
        {
            var data = new TagData(plcId, tagName, value, now);

            // ⚠️ 这里才是核心：满了就等，不丢数据，不爆内存
            await _channel.Writer.WriteAsync(data, ct);
        }
    }
```

### 4.4 消费者——批量处理

```csharp
    private async Task ConsumeLoopAsync(CancellationToken ct)
    {
        // 批量缓冲区：累积到一定数量再写入，减少 IO 次数
        var buffer = new List<TagData>(capacity: 512);

        await foreach (var data in _channel.Reader.ReadAllAsync(ct))
        {
            buffer.Add(data);

            // 达到批次大小或超时，执行一次批量写入
            if (buffer.Count >= 512)
            {
                await BatchPersistAsync(buffer, ct);
                buffer.Clear();
            }
        }

        // 退出前刷空剩余数据
        if (buffer.Count > 0)
            await BatchPersistAsync(buffer, ct);
    }

    private ValueTask BatchPersistAsync(
        List<TagData> batch, CancellationToken ct)
    {
        // 实际实现：批量写入 InfluxDB / SQL Server / 时序库
        // 这里假设有批量写入能力
        return InfluxDbWriter.WriteBatchAsync(batch, ct);
    }
```

### 4.5 资源清理

```csharp
    public async ValueTask DisposeAsync()
    {
        _channel.Writer.TryComplete();
        _cts.Cancel();

        await Task.WhenAll(_workers);
        _cts.Dispose();
    }
}
```

---

## 五、进阶优化

### 5.1 批量通道（Chunk Mode）

如果数据体积小但量大，可以引入二次批次，进一步减少消费者频繁读取的开销：

```csharp
var batchChannel = Channel.CreateBounded<TagData[]>(
    new BoundedChannelOptions(128)
    {
        FullMode = BoundedChannelFullMode.Wait
    });

// 攒批生产者：从采集通道读取，攒够一批再推给消费通道
_ = BatchProducerAsync(_channel.Reader, batchChannel.Writer, ct);
```

### 5.2 动态调节容量

基于当前消费延迟动态调整通道容量或消费者数量：

```csharp
// 从消费者角度监控积压情况
if (_channel.Reader.Count > _channelCapacity * 0.8)
    AddConsumer();   // 增加一个消费者 Task

if (_channel.Reader.Count < _channelCapacity * 0.2 && _workers.Count > _minWorkers)
    RemoveConsumer(); // 减少一个消费者 Task
```

> **注意**：`ChannelReader.Count` 仅在 `SingleReader` 模式且 `AllowSynchronousContinuations = false` 时可用，非此模式要用近似度量。

### 5.3 监控与可观测性

```csharp
public ChannelHealth GetHealth()
{
    return new ChannelHealth
    {
        Capacity = _channelCapacity,
        CurrentCount = _channel.Reader.Count,
        Utilization = (double)_channel.Reader.Count / _channelCapacity,
        ConsumerCount = _workers.Count,
        CompletedWrites = _completedWrites,
        CompletedReads = _completedReads
    };
}
```

接入 Prometheus / OpenTelemetry 后，这就是一个现成的背压告警指标。

### 5.4 采集超时保护

如果业务上不允许生产者无限等待，可以用 `WriteAsync` 配合超时：

```csharp
using var timeoutCts = CancellationTokenSource.CreateLinkedTokenSource(
    ct, _cts.Token);

try
{
    timeoutCts.CancelAfter(TimeSpan.FromMilliseconds(500));
    await _channel.Writer.WriteAsync(data, timeoutCts.Token);
}
catch (OperationCanceledException) when (!ct.IsCancellationRequested)
{
    // 采集超时 → 记录告警、丢弃该帧或纳入补偿队列
    _logger.Warn("采集超时，丢弃数据: {PlcId}/{Tag}", plcId, tagName);
}
```

---

## 六、方案对比

| 特性 | BlockingCollection | ConcurrentQueue + Manual | System.Threading.Channels |
|------|-------------------|------------------------|--------------------------|
| 异步支持 | ❌ 同步阻塞 | ❌ 需要自己封装 | ✅ WriteAsync/ReadAsync |
| 背压 | ✅ 有界 + Bounded | ❌ 需要自己实现 | ✅ 原生设计 |
| 满队列策略 | ✅ 三种 | ❌ | ✅ 四种 |
| 多生产者/消费者 | ✅ | ✅ | ✅ |
| GC 友好 | ❌ 包装器 | ⚠️ 中等 | ✅ 优化设计 |
| 批量读取 | ❌ | ✅ 自己实现 | ✅ ReadAllAsync + 批量 |
| 取消支持 | ✅ CancellationToken | ✅ | ✅ |
| 内存占用 | 有界可控 | 无界危险 | 有界可控 |
| 代码量 | 较少 | 大量 | 极少 |

---

## 七、适用场景与注意事项

### ✅ 适合场景

- 边缘网关 / 采集服务器上的 PLC 数据汇聚
- 产线 OPC UA / Modbus TCP 数据高并发采集
- 高频振动 / 温度数据采集（需要背压保护）
- 对轻量化要求高、不想引入消息队列的场景

### ⚠️ 注意事项

1. **Channel 不是持久化存储**——进程崩溃数据就丢了。需要持久化去重的话，结合本地 WAL（Write-Ahead Log）或 SQLite 缓冲
2. **SingleReader/SingleWriter 性能提示**：如果明确只有一个生产者/消费者，设为 `true` 可获得额外优化
3. **`Wait` ≠ 无延迟**：背压意味着生产者会自然变慢，这是设计意图，不是 Bug
4. **消费者异常处理**：消费循环必须 try-catch，异常时重建消费者 Task，否则通道会无人消费导致死锁

---

## 八、总结

`System.Threading.Channels` 为工业 PLC 高并发采集提供了一个**零外部依赖、托管代码、线程安全**的背压限流方案。

它的价值不在于"快"——实际上它多了一层内存缓冲——而在于**可控**。通过 `BoundedChannel`，你可以清晰地回答三个问题：

- **内存上限是多少？** → 通道容量 × 单条数据大小
- **生产跟不上了怎么办？** → WriteAsync 自然等待，不丢数据
- **消费跟不上了怎么办？** → 同上的背压，或 Drop 保最新

在一个连边缘网关都要控制硬件成本的工业场景里，多一个 RabbitMQ 就是多一个运维炸弹。而 `System.Threading.Channels` 作为 .NET 运行时的一部分，开箱即用，是轻量化采集架构中值得优先考虑的选择。

---

*本文的完整示例代码可在 [GitHub](https://github.com/Awkward-Coder) 查看。*
