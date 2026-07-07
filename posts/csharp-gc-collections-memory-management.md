---
title: C# GC 是如何管理集合内存的？
slug: csharp-gc-collections-memory-management
description: >-
  深入分析 .NET GC 对 List、Dictionary 等常用集合的内存分配、晋升、压缩与回收机制。涵盖内部实现原理、LOH 对大集合的影响、以及如何写出 GC-friendly 的集合代码。
tags:
  - technical
added: "July 07 2026"
---

# C# GC 是如何管理集合内存的？

> C# 程序员每天和 `List<T>`、`Dictionary<,>` 打交道，但很少有人停下来问：**这些集合的内存到底是怎么被 GC 管理的？** 本文从 GC 的视角切入，带你理解集合的"一生"——从分配、扩容、晋升到回收，以及如何写出 GC-friendly 的集合代码。

---

## 一、从 GC 的视角看集合

### 集合在 GC 眼中是什么？

无论你用的是 `List<T>`、`Dictionary<,>` 还是 `HashSet<T>`，在 GC 眼里它们只有两种构成：

1. **集合对象本身**——一个引用类型实例，在托管堆上占据一块内存
2. **内部缓冲区（后备数组）**——一个 `T[]` 数组，也是引用类型

```csharp
// List<T> 的核心内部结构（简化版）
public class List<T> {
    internal T[] _items;  // 实际存储数据的数组
    internal int _size;   // 当前元素数量
    internal int _version;
    private object _syncRoot;
}
```

当你写 `new List<int>()` 时，GC 分配了 List 对象本身（约 32-40 字节）和一个默认容量（通常是 4）的内部数组。**这两个对象在 GC 堆上，都受 GC 管理。**

### 值类型 vs 引用类型的集合，GC 行为截然不同

这是第一个关键认知分歧：

| 集合类型 | `List<int>` | `List<string>` |
|---------|------------|---------------|
| 内部数组 | `int[]`（值类型数组） | `string[]`（引用类型数组） |
| 元素位置 | 元素直接存在数组中 | 数组存的是引用，对象在堆上别处 |
| GC 扫描代价 | 一次扫描整个连续块 | 需要追踪每个引用，访问更多对象 |

```csharp
var intList = new List<int> { 1, 2, 3 };
// GC 视角：只需扫描 List 对象 + int[] 连续内存块
// int[] 内没有引用需要追踪，GC 代价很低

var stringList = new List<string> { "a", "b", "c" };
// GC 视角：需要扫描 List 对象 + string[]（每个元素都是引用）
// 还要追踪 "a"、"b"、"c" 三个 string 对象
```

对于包含引用类型的集合，GC **标记阶段**的工作量随元素数量线性增长，因为每个引用都需要被追踪。这就是为什么**结构体集合**（能保持值类型语义）在某些场景下更 GC-friendly。

---

## 二、集合扩容——GC 最"忙"的时刻

### 扩容时发生了什么？

以 `List<T>` 为例，当添加元素超出当前容量时：

```csharp
// List<T>.Add 的内部逻辑（简化）
public void Add(T item) {
    if (_size == _items.Length) {
        EnsureCapacity(_size + 1);  // 触发扩容
    }
    _items[_size++] = item;
    _version++;
}

private void EnsureCapacity(int min) {
    if (_items.Length < min) {
        int newCapacity = _items.Length == 0 ? 4 : _items.Length * 2;
        T[] newArray = new T[newCapacity];  // 新数组分配
        Array.Copy(_items, newArray, _size); // 元素拷贝
        _items = newArray;  // 旧数组变成垃圾
    }
}
```

从 GC 角度看，每次扩容涉及：

1. **分配**一个新数组（更大的 `T[]`）
2. **拷贝**元素到新数组
3. 旧数组变成**GC 垃圾**，等待回收

这意味着：**如果你预知集合大小但没设置初始容量，每一次扩容都在给 GC 施加额外压力。**

```csharp
// 坏例子：默认容量 4，不断扩容
var list = new List<byte[]>();
for (int i = 0; i < 10000; i++)
    list.Add(new byte[256]);

// 好例子：预置容量
var list = new List<byte[]>(10000);
for (int i = 0; i < 10000; i++)
    list.Add(new byte[256]);
```

不预设容量的极端情况：需要扩容 `⌈log₂(n/4)⌉` 次（约 12 次到 10000），每次产生一个垃圾数组，最终产生约 `n` 大小的额外垃圾总和。

### Dictionary 的扩容更复杂

`Dictionary<,>` 除了后备数组（`Entry<TKey,TValue>[]`），还维护了 buckets 数组用于哈希寻址：

```csharp
// Dictionary<TKey,TValue> 内部结构（简化）
private struct Entry {
    public int hashCode;    // 哈希码
    public int next;        // 链表指针（处理冲突）
    public TKey key;
    public TValue value;
}

private int[] _buckets;  // 哈希桶
private Entry[] _entries; // 实际数据
```

扩容时 GC 要：

1. 分配新的 `_buckets` 数组
2. 分配新的 `_entries` 数组
3. 拷贝所有 Entry 并重新计算 bucket 映射
4. 旧数组变成垃圾

`Dictionary` 的扩容代价远高于 `List`——不仅分配两倍内存，GC 还要处理两次数组回收。

---

## 三、分代回收与集合的生命周期

### Gen0 → Gen1 → Gen2 的晋升

.NET GC 使用分代回收，集合对象在生命周期中会经历晋升：

```csharp
void ProcessData() {
    var tempList = new List<int>(100);  // Gen0 分配
    // ... 使用 tempList ...
}  // 方法结束，tempList 变成 Gen0 垃圾

// 下次 Gen0 回收 → 立即回收
```

但如果集合被提升字段或闭包引用，会经历晋升：

```csharp
private List<Order> _orders = new();  // 类级别

void LoadData() {
    for (int i = 0; i < 1000; i++) {
        var tmp = new List<int>(10);  // Gen0 分配
        // ...
    }
}
```

| 发生场景 | GC 代 | 集合行为 |
|---------|-------|---------|
| 局部变量，短生命周期 | Gen0 | 随方法结束变成垃圾，Gen0 回收很快 |
| 类字段，中等生命周期 | Gen0 → Gen1 | 第一次幸存后晋升到 Gen1 |
| 静态字段 / 缓存 | Gen0 → Gen1 → Gen2 | 长期幸存，最终到 Gen2。Gen2 回收很少触发 |
| 大集合（>85KB） | LOH | **不压缩**，直接分配在大对象堆 |

### Gen2 中的"常驻"集合

存在字段或静态变量中的集合，晋升到 Gen2 后：

```csharp
static class AppCache {
    // 这个 Dictionary 会很快到达 Gen2，且很少被回收
    public static readonly Dictionary<string, object> Cache = new();
}
```

**问题在于：** Gen2 GC 触发频率很低（通常数十秒到数分钟一次），但如果 Gen2 占用了大量内存，它会迫使系统触发更多 Gen2 回收，造成明显的 STW（Stop-The-World）暂停。

### Gen2 对象的"固定"问题

当集合内部的数组被 fixed（如在 P/Invoke 中传递给非托管代码），GC 无法移动它：

```csharp
byte[] buffer = new byte[4096];
fixed (byte* p = buffer) {
    // 此期间 buffer 无法被 GC 压缩
    NativeCall(p);
}
```

固定对象的产生会导致**内存碎片**——GC 无法压缩数组周围的内存，降低了内存利用率。

---

## 四、大对象堆（LOH）与大集合

### 超过 85KB 的数组

所有大小超过 85KB（85000 字节）的对象都分配在 LOH：

```csharp
// 这个数组 > 85KB，分配在 LOH
byte[] bigBuffer = new byte[85001];  // LOH 分配

// 集合内部数组超过 85KB 时
var list = new List<MyStruct>();    // 当内部数组超过 85KB → LOH
```

**LOH 的关键特性：不压缩。** 这意味着：

- LOH 上没有迁移/复制开销
- 但会产生碎片——大对象的间隙很难被后续分配复用
- 扩容时更糟糕：旧的大数组 + 新的大数组 = LOH 上两个垃圾

### 对大集合的 LOH 影响

```csharp
var list = new List<byte[]>();
for (int i = 0; i < 10000; i++)
    list.Add(new byte[300]);  // 每个 300 字节 < 85KB，仍在 Gen0
```

只要数组大小小于 85KB，它就不会进入 LOH。但：

```csharp
var strings = new List<string>(200_000);
// 内部数组：200_000 * 8 字节（64位引用）= 1.6MB → LOH 分配
```

仅列表的后备数组就已达 1.6MB，分配在 LOH 上。如果这个列表频繁扩容，LOH 上会出现多个 1.6M+ 的垃圾数组，造成碎片。

**变通方案：** 如果集合大小可预知，用 `Capacity` 一步到位：

```csharp
// 提前分配 LOH 上正好大小的数组，避免多次扩容
var list = new List<string>(200_000);  // 一次 LOH 分配，而后不再扩容
```

.NET 5+ 引入了 `GCSettings.LargeObjectHeapCompactionMode` 允许对 LOH 进行压缩，但这是全堆操作，代价很高：

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();
```

---

## 五、GC 友好的集合实践

### 5.1 使用 ArrayPool 避免分配

`System.Buffers.ArrayPool<T>` 是缓解集合 GC 压力的最有效手段：

```csharp
// 不使用 ArrayPool：每次分配新数组，压力大
byte[] buffer = new byte[1024];
ProcessData(buffer);
// buffer 变成垃圾

// 使用 ArrayPool：复用数组
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
try {
    ProcessData(buffer);
} finally {
    ArrayPool<byte>.Shared.Return(buffer);
    // 数组回到池中，下次复用
    // GC 完全不需要处理它
}
```

**内部原理：** `ArrayPool` 内部维护了多组分代数组（类似 GC 的分代思想）：

- **桶（buckets）**：按 2 的幂划分大小
- 每个桶包含多个数组，供不同线程借用
- 返还时不清空（默认），减少下次使用时的 CPU 开销
- 线程局部缓存 + 共享池的双层结构

对于集合场景，`List<T>` 不能直接利用 ArrayPool，但可以自己封装：

```csharp
public ref struct PooledList<T> {
    private T[] _buffer;
    private int _count;

    public PooledList(int capacity) {
        _buffer = ArrayPool<T>.Shared.Rent(capacity);
        _count = 0;
    }

    public void Add(T item) {
        if (_count >= _buffer.Length) Expand();
        _buffer[_count++] = item;
    }

    public void Dispose() {
        ArrayPool<T>.Shared.Return(_buffer);
    }

    private void Expand() {
        T[] newBuffer = ArrayPool<T>.Shared.Rent(_buffer.Length * 2);
        Array.Copy(_buffer, newBuffer, _count);
        ArrayPool<T>.Shared.Return(_buffer);
        _buffer = newBuffer;
    }
}
```

### 5.2 用集合池减少对象分配

`Microsoft.Toolkit.Collections.ObjectPool`（或自己实现简单的池）：

```csharp
public class ListPool<T> {
    private readonly ConcurrentBag<List<T>> _pool = new();

    public List<T> Rent() {
        return _pool.TryTake(out var list) ? list : new List<T>();
    }

    public void Return(List<T> list) {
        list.Clear();
        _pool.Add(list);
    }
}

// 使用
var pool = new ListPool<int>();
var list = pool.Rent();
// ... 使用 ...
pool.Return(list);  // 对象不产生 GC 压力
```

### 5.3 集合的选择影响 GC

| 场景 | 推荐集合 | 对 GC 的影响 |
|------|---------|-------------|
| 只追加、随机访问 | `List<T>` | 单一数组，GC 友好 |
| 高频增删头部 | `LinkedList<T>` | 节点分散在堆上，GC 扫描需要追踪每个节点引用 |
| 高频插入/删除中间 | `LinkedList<T>` | 同上，每个节点是独立对象 |
| 交替 插入/删除 | `SortedList<K,V>` | 内部两个数组，GC 代价较低 |
| 快速查找 | `Dictionary<,>` | 两个数组 + Entry 结构，有额外开销 |
| 快速 HashSet | `HashSet<T>` | 类似 Dictionary |

**`LinkedList<T>` 的 GC 代价：**

```csharp
var linkedList = new LinkedList<int>();
for (int i = 0; i < 10000; i++)
    linkedList.AddLast(i);
// 产生了 10000 个 LinkedListNode<int> 对象（每个约 40 字节）
// Gen0 GC 要追踪 10000 个对象，外加 LinkedList 本身

var list = new List<int>(10000);
for (int i = 0; i < 10000; i++)
    list.Add(i);
// 只有一个 int[] + List 对象
// Gen0 GC 只需扫描 2 个对象（数组内无引用要追踪）
```

### 5.4 Span<T> 与 Memory<T>：零分配的子集操作

.NET 的 `Span<T>` 可以在不分配新数组的前提下操作内存片段：

```csharp
// ❌ 分配新数组：产生 GC 压力
byte[] fullData = GetLargeData();
byte[] firstChunk = fullData[..1000];  // 新数组分配
ProcessChunk(firstChunk);

// ✅ 零分配：只是"指向"原始数组的一段
byte[] fullData = GetLargeData();
Span<byte> chunk = fullData.AsSpan(0, 1000);  // 零分配
ProcessChunk(chunk);  // 接受 ReadOnlySpan<byte>
```

对于集合操作，`CollectionsMarshal` 提供了直接操作 `List<T>` 后备数组的能力：

```csharp
using System.Runtime.InteropServices;

var list = new List<int> { 1, 2, 3 };
Span<int> span = CollectionsMarshal.AsSpan(list);
// 直接操作后备数组，零拷贝
span[0] = 10;  // 等价于 list[0] = 10
```

当需要子集、合并等操作时，优先用 Span 而非创建新集合。

### 5.5 关注 struct 在集合中的表现

值类型在集合中的行为至关重要：

```csharp
struct Point {
    public int X, Y;
}

// 集合包含值类型
var points = new List<Point>(1000);
// points._items 是 Point[]，值紧凑排列
// GC 扫描：只需要看 List 对象 + Point[] 这个连续块
// Point[] 中不含引用 → GC 扫描很快

class PointClass {
    public int X, Y;
}

// 集合包含引用类型
var points2 = new List<PointClass>(1000);
// points2._items 是 PointClass[]，每个元素是一个引用
// GC 扫描：需要看 List 对象 + PointClass[] + 1000 个 PointClass 实例
// 扫描代价高出几个数量级
```

**但要注意：** 如果 struct 包含引用类型字段，情况就不同了：

```csharp
struct Person {
    public string Name;  // 引用类型字段
    public int Age;
}

var people = new List<Person>(1000);
// GC 仍然需要扫描每个 Person 的 Name 字段（是引用）
// 但 Person 本身作为一个值类型，在 GC 扫描中仍然比类高效
// 因为不需要为每个 Person 追踪独立的对象头
```

### 5.6 避免集合在 LOH 上频繁扩容

```csharp
// ❌ LOH 每次扩容都产生一个大垃圾
static List<long> Data = new List<long>();
void AddBatch(long[] values) {
    Data.AddRange(values);
}

// 第一次 AddRange → 内部数组 4（SOH）
// 随着数据增多 → 扩容到 8, 16, 32, ... 
// 当 > 85KB 时 → LOH
// 之后每次扩容都在 LOH 上留下垃圾

// ✅ 预分配足够的容量
static List<long> Data = new List<long>(100_000);  // 一次到位
```

### 5.7 使用 Immutable Collections 的场景

在并发场景下，不可变集合通过结构共享减少 GC 压力：

```csharp
var builder = ImmutableArray.CreateBuilder<int>();
builder.AddRange(data);
ImmutableArray<int> arr = builder.ToImmutable();

// 不可变集合 = 成员线程安全
// 多线程共享同一个引用，不会产生 GC 压力
// 修改操作返回新实例，但内部可以共享未变更部分
```

---

## 六、一个综合案例

假设有一个高频调用的处理方法，每次处理一批订单数据：

```csharp
// ❌ 高压写法：每次分配大量临时集合
class OrderProcessor {
    public void ProcessBatch(List<Order> orders) {
        var filtered = new List<Order>(orders.Count);
        foreach (var order in orders) {
            if (order.Amount > 100) filtered.Add(order);
        }

        var ids = new List<int>(filtered.Count);
        foreach (var order in filtered) {
            ids.Add(order.Id);
        }

        SendToApi(ids);
    }
}
// 每次调用分配 3 个 List（含内部数组），高频调用下 GC 压力巨大
```

```csharp
// ✅ 低压力写法：复用 + 零分配
class OrderProcessor {
    // 复用集合
    private readonly List<Order> _filtered = new();
    private readonly List<int> _ids = new();

    public void ProcessBatch(List<Order> orders) {
        _filtered.Clear();
        _ids.Clear();

        foreach (var order in orders) {
            if (order.Amount > 100) _filtered.Add(order);
        }

        foreach (var order in _filtered) {
            _ids.Add(order.Id);
        }

        SendToApi(_ids);
    }
}
// 复用已有集合，只在首次调用时产生 GC 分配
// 后续调用零分配
```

```csharp
// ✅✅ 极致优化：用 ArrayPool + Span 零分配
class OrderProcessor {
    private int[] _idsPool = ArrayPool<int>.Shared.Rent(1024);
    private Order[] _filteredPool = ArrayPool<Order>.Shared.Rent(1024);

    public void ProcessBatch(Span<Order> orders) {
        int filterCount = 0;
        foreach (ref var order in orders) {
            if (order.Amount > 100) {
                _filteredPool[filterCount++] = order;
            }
        }

        int idCount = 0;
        var filteredSpan = _filteredPool.AsSpan(0, filterCount);
        foreach (ref var order in filteredSpan) {
            _idsPool[idCount++] = order.Id;
        }

        SendToApi(_idsPool.AsSpan(0, idCount));
    }

    public void Dispose() {
        ArrayPool<int>.Shared.Return(_idsPool);
        ArrayPool<Order>.Shared.Return(_filteredPool);
    }
}
// 完全零分配
```

---

## 七、总结

| 关键认知 | 对 GC 的影响 |
|---------|-------------|
| 集合内部是数组 | 扩容分配新数组 → 旧数组变垃圾，增加 GC 压力 |
| 分代晋升 | 集合的 GC 代决定了回收频率。Gen2 对象回收代价高 |
| LOH 不压缩 | 大集合在 LOH 上容易碎片化，预分配容量避免多次扩容 |
| 值类型 vs 引用类型 | 值类型集合在 GC 扫描中代价低得多，因为数组不包含引用 |
| 扩容次数 | 扩容次数 = GC 新分配次数。预设 `Capacity` 是最简单的优化 |
| 数组池 | `ArrayPool` 可以让数组完全不经过 GC，适合高频临时场景 |
| Span/Memory | 零分配的子集操作替代集合拷贝 |

**一句话总结：** 集合的 GC 性能，核心在于控制**分配次数**和**数组大小**。预设容量、避免扩容、复用数组、优先使用值类型集合——做到这几点，你的集合代码就对 GC 非常友好了。

理解 GC 如何管理集合，不只是为了"优化"，更是为了写出可预测、低延迟的 .NET 应用。在频繁触发 Gen2 GC 或 LOH 碎片导致内存暴涨时，这些知识会是你最有力的 debug 工具。

---

> **参考阅读：**
> - [.NET GC 内部机制文档](https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/)
> - [ArrayPool<T> 官方文档](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1)
> - [Pro .NET Memory Management](https://prodotnetmemory.com/) — Konrad Kokosa
> - [Writing High-Performance .NET Code](https://www.writinghighperf.net/) — Ben Watson
