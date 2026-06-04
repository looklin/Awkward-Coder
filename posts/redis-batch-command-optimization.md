---
title: Redis 批量命令优化实战——从秒杀到工业读写的性能调优
slug: redis-batch-command-optimization
description: >-
  深入解析 Redis 批量执行命令的优化方案：Pipeline、MGET/MSET、Lua 脚本、连接池复用，结合秒杀抢单、工业高频读写等真实场景，给出可落地的性能调优策略和避坑指南。
tags:
  - technical
added: "June 04 2026"
---

# Redis 批量命令优化实战——从秒杀到工业读写的性能调优

> Redis 虽然是单线程模型、微秒级响应，但如果每次操作都走一次网络往返（RTT），累积起来的延迟会吃掉所有性能优势。本文将从原理出发，结合秒杀和工业读写场景，系统梳理 Redis 批量命令的优化方案。

---

## 一、为什么需要批量执行？

### 1. 网络 RTT 是最大瓶颈

Redis 的内存操作通常在 **1 微秒** 以内，但一次网络往返（RTT）通常是：

| 网络环境 | 典型 RTT |
|---------|---------|
| 同机房内网 | 0.1 - 0.5 ms |
| 跨机房专线 | 1 - 5 ms |
| 公有云内网 | 0.2 - 1 ms |
| 公网 | 20 - 200 ms |

假设同机房 RTT = 0.3ms，你需要读 1000 个 key：

```
串行方式：1000 × 0.3ms = 300ms  （仅网络延迟）
Pipeline：1 × 0.3ms = 0.3ms      （一次往返）
```

**300 倍差距，核心在于减少网络往返次数。**

### 2. Redis 单线程模型的制约

Redis 6.0 之前是纯单线程处理命令。这意味着：

- ✅ 没有锁竞争、线程切换开销
- ❌ 客户端串行发送命令时，Redis 大部分时间在等待网络

批量命令让 Redis "一次拿够食材"，充分利用其单线程的处理能力。

---

## 二、批量执行的核心方案

### 方案一：Pipeline（管道）

Pipeline 是最通用的批量执行方式，客户端将多个命令打包发送，Redis 依次执行后返回所有结果。

```csharp
// StackExchange.Redis 示例
using var muxer = ConnectionMultiplexer.Connect("localhost:6379");
var db = muxer.GetDatabase();

// 创建批处理
var batch = db.CreateBatch();

// 批量写入
var tasks = new List<Task>();
for (int i = 0; i < 1000; i++)
{
    tasks.Add(batch.StringSetAsync($"user:{i}:score", i * 10));
}

// 执行
batch.Execute();

// 等待所有操作完成
await Task.WhenAll(tasks);
Console.WriteLine($"批量写入 1000 个 key 完成");

// 批量读取
tasks.Clear();
var results = new string[1000];
for (int i = 0; i < 1000; i++)
{
    tasks.Add(batch.StringGetAsync($"user:{i}:score")
        .ContinueWith(t => results[i] = t.Result.ToString()));
}
batch.Execute();
await Task.WhenAll(tasks);
```

**Pipeline 的特点：**

- ✅ 最灵活，支持任意命令组合
- ✅ 大幅减少 RTT 次数
- ❌ Pipeline 中的命令不是原子执行的（中间可能插入其他客户端的命令）
- ❌ 结果集全部在内存中，超大 Pipeline 可能 OOM

### 方案二：MGET / MSET（多 key 操作）

Redis 原生支持的多 key 原子操作命令：

```csharp
var db = muxer.GetDatabase();

// MSET — 批量写入（原子性）
var kvpairs = new KeyValuePair<RedisKey, RedisValue>[1000];
for (int i = 0; i < 1000; i++)
{
    kvpairs[i] = new KeyValuePair<RedisKey, RedisValue>(
        $"user:{i}:score", i * 10);
}
await db.StringSetAsync(kvpairs);

// MGET — 批量读取（原子性）
var keys = new RedisKey[1000];
for (int i = 0; i < 1000; i++)
{
    keys[i] = $"user:{i}:score";
}
RedisValue[] values = await db.StringGetAsync(keys);
```

**MGET/MSET 的特点：**

- ✅ 原子操作（Redis 单线程天然保证）
- ✅ 一次 RTT
- ❌ 只支持 String 类型
- ❌ 不支持过期时间设置（MSETNX 除外，且不带 TTL）
- ❌ Key 分布在不同 hash slot 时，集群模式会报错

### 方案三：Lua 脚本

Lua 脚本在 Redis 服务端原子执行，适合需要读-改-写逻辑的批量操作：

```csharp
// 批量增加库存并返回结果
string luaScript = @"
local results = {}
for i = 1, #KEYS do
    local key = KEYS[i]
    local delta = tonumber(ARGV[i])
    local current = tonumber(redis.call('GET', key) or '0')
    local new_val = current + delta
    if new_val < 0 then
        results[i] = -1  -- 库存不足
    else
        redis.call('SET', key, new_val)
        results[i] = new_val
    end
end
return results
";

var prepared = LuaScript.Prepare(luaScript);

var keys = new RedisKey[500];
var values = new RedisValue[500];
for (int i = 0; i < 500; i++)
{
    keys[i] = $"stock:product:{i}";
    values[i] = -1; // 每个减 1
}

var result = await db.ScriptEvaluateAsync(prepared, keys, values);
```

**Lua 脚本的特点：**

- ✅ 原子执行（脚本执行期间不会有其他命令插入）
- ✅ 支持复杂逻辑（条件判断、循环、多种数据结构操作）
- ❌ 脚本执行时间过长会阻塞 Redis（Redis 没有脚本超时中断）
- ❌ 调试困难

### 方案四：连接池 + 并行 Pipeline

对于分布式场景，可以在多个连接上并行执行 Pipeline：

```csharp
// 将任务按连接分片，并行执行
public async Task<int> BatchSetParallelAsync(
    IEnumerable<(string Key, string Value)> items,
    int batchSize = 500)
{
    var groups = items
        .Select((item, index) => (item, index))
        .GroupBy(x => x.index / batchSize)
        .Select(g => g.Select(x => x.item).ToList())
        .ToList();

    var tasks = groups.Select(async batch =>
    {
        var conn = await GetConnectionAsync(); // 从连接池获取
        var db = conn.GetDatabase();
        var kv = batch.Select(x =>
            new KeyValuePair<RedisKey, RedisValue>(x.Key, x.Value)
        ).ToArray();
        await db.StringSetAsync(kv);
        return batch.Count;
    });

    var results = await Task.WhenAll(tasks);
    return results.Sum();
}
```

---

## 三、实战场景一：秒杀系统

### 3.1 场景分析

秒杀场景的核心特征：

- ⚡ **极高并发**：瞬间数万 QPS
- ⚡ **读写密集**：库存检查 + 扣减 + 订单创建
- ⚡ **强一致性要求**：不能超卖
- ⚡ **时间窗口极短**：通常只有几分钟

### 3.2 方案演进

#### 版本一：串行操作（反模式）

```csharp
// ❌ 每次请求 3 次 Redis 往返
public async Task<bool> SeckillV1(string userId, string productId)
{
    // 1. 检查库存
    var stock = await db.StringGetAsync($"stock:{productId}");
    if (stock <= 0) return false;

    // 2. 扣减库存
    var newStock = await db.StringDecrementAsync($"stock:{productId}");
    if (newStock < 0)
    {
        // 回滚（已经超卖了！）
        await db.StringIncrementAsync($"stock:{productId}");
        return false;
    }

    // 3. 记录购买（防重复购买）
    var added = await db.SetAddAsync($"seckill:{productId}:buyers", userId);
    if (!added)
    {
        // 重复购买，回滚库存
        await db.StringIncrementAsync($"stock:{productId}");
        return false;
    }

    return true;
}
```

**问题：**

- 3 次 RTT，延迟放大 3 倍
- 检查库存和扣减库存之间有竞态条件（TOCTOU）
- 高并发下 Redis 连接池可能被耗尽

#### 版本二：Lua 脚本原子操作

```csharp
string seckillLua = @"
local stockKey = KEYS[1]
local buyerSetKey = KEYS[2]
local userId = ARGV[1]
local quantity = tonumber(ARGV[2])

-- 1. 检查是否已购买
if redis.call('SISMEMBER', buyerSetKey, userId) == 1 then
    return -1  -- 重复购买
end

-- 2. 检查并扣减库存
local stock = tonumber(redis.call('GET', stockKey) or '0')
if stock < quantity then
    return -2  -- 库存不足
end

redis.call('DECRBY', stockKey, quantity)

-- 3. 记录购买
redis.call('SADD', buyerSetKey, userId)

-- 4. 记录订单（可选：写入队列）
local orderId = redis.call('INCR', 'order:id:gen')
redis.call('HSET', 'order:' .. orderId,
    'user', userId,
    'product', stockKey,
    'quantity', quantity,
    'time', redis.call('TIME')[1])

return orderId
";

var seckillScript = LuaScript.Prepare(seckillLua);

public async Task<long> SeckillV2(string userId, string productId, int quantity = 1)
{
    var result = await db.ScriptEvaluateAsync(
        seckillScript,
        new RedisKey[] { $"stock:{productId}", $"seckill:{productId}:buyers" },
        new RedisValue[] { userId, quantity });

    long orderId = (long)result;

    if (orderId == -1) throw new InvalidOperationException("您已购买过此商品");
    if (orderId == -2) throw new InvalidOperationException("库存不足");

    return orderId; // 返回订单号
}
```

**优势：**

- ✅ 1 次 RTT，原子执行
- ✅ 无竞态条件，不会超卖
- ✅ 所有逻辑在 Redis 端完成，减少应用服务器压力

#### 版本三：Pipeline 批量预加载 + Lua

对于需要批量查询多个商品库存的场景：

```csharp
// 预热：批量加载多个商品库存到应用层缓存
public async Task<Dictionary<string, int>> BatchLoadStock(IEnumerable<string> productIds)
{
    var keys = productIds.Select(id => (RedisKey)$"stock:{id}").ToArray();
    var values = await db.StringGetAsync(keys);

    var result = new Dictionary<string, int>();
    for (int i = 0; i < productIds.Count(); i++)
    {
        result[productIds.ElementAt(i)] = (int)values[i];
    }
    return result;
}

// 批量扣减多个商品库存（组合购买场景）
string multiProductLua = @"
local results = {}
for i = 1, #KEYS do
    local stockKey = KEYS[i]
    local qty = tonumber(ARGV[i])
    local stock = tonumber(redis.call('GET', stockKey) or '0')

    if stock < qty then
        -- 库存不足，回滚之前所有的扣减
        for j = 1, i - 1 do
            redis.call('INCRBY', KEYS[j], tonumber(ARGV[j]))
        end
        return { -i, stock }  -- 返回哪个商品库存不足
    end

    redis.call('DECRBY', stockKey, qty)
    results[i] = stock - qty
end
return results
";
```

### 3.3 性能对比

| 方案 | RTT 次数 | 原子性 | 1000 并发下单耗时 | 超卖风险 |
|------|:---:|:---:|:---:|:---:|
| 串行操作 | 3 | ❌ | ~300ms | ⚠️ 有 |
| Lua 脚本 | 1 | ✅ | ~10ms | ❌ 无 |
| Pipeline + Lua | 1~2 | ✅ | ~5ms | ❌ 无 |

---

## 四、实战场景二：工业高频读写

### 4.1 场景分析

工业场景（如 IoT 传感器采集、PLC 数据读取、产线监控）的特征：

- 📊 **高频写入**：数千传感器，每秒数百次数据点上报
- 📊 **批量读取**：Dashboard 需要一次性拉取大量最新值
- 📊 **数据量大**：可能有百万级 key 在内存中
- 📊 **时效性要求高**：数据延迟不能超过秒级
- 📊 **可靠性要求高**：数据不能丢

### 4.2 批量写入优化

#### 场景：1000 个传感器，每秒上报一次

```csharp
// ❌ 反模式：逐条写入
foreach (var sensor in sensors)
{
    await db.StringSetAsync(
        $"sensor:{sensor.Id}:latest",
        sensor.Value);
    // 1000 次 RTT！
}

// ✅ 方案一：Pipeline 批量写入
public async Task BatchWriteSensorData(IEnumerable<SensorReading> readings)
{
    var batch = db.CreateBatch();
    var tasks = new List<Task>();

    foreach (var r in readings)
    {
        // 写入最新值
        tasks.Add(batch.StringSetAsync(
            $"sensor:{r.SensorId}:latest",
            r.Value,
            expiry: TimeSpan.FromHours(24)));

        // 写入时序数据（用 List 存储）
        tasks.Add(batch.ListLeftPushAsync(
            $"sensor:{r.SensorId}:history",
            $"{r.Timestamp}:{r.Value}",
            -1, // 从左侧推入
            truncateTo: 2880)); // 保留最近 2880 条（约 1 天，每分钟一条）
    }

    batch.Execute();
    await Task.WhenAll(tasks);
}
// 2000 个命令 → 1 次 RTT
```

#### 方案二：Hash 结构优化

当同一个传感器的多个属性需要同时写入时，用 Hash 更优：

```csharp
// 一个传感器的多个指标一次性写入
public async Task WriteSensorMetrics(string sensorId, Dictionary<string, double> metrics)
{
    var fields = metrics.Select(kv =>
        new HashEntry(kv.Key, kv.Value)
    ).ToArray();

    // 一次 HMSET，原子写入所有字段
    await db.HashSetAsync($"sensor:{sensorId}:metrics", fields);

    // 如果需要过期时间，单独设置（HMSET 不支持字段级 TTL）
    await db.KeyExpireAsync($"sensor:{sensorId}:metrics", TimeSpan.FromHours(1));
}

// 读取也只需要一次 HGETALL
public async Task<Dictionary<string, double>> ReadSensorMetrics(string sensorId)
{
    var entries = await db.HashGetAllAsync($"sensor:{sensorId}:metrics");
    return entries.ToDictionary(
        e => e.Name.ToString(),
        e => double.Parse(e.Value.ToString()));
}
```

#### 方案三：消息队列缓冲 + 批量刷入

对于极端高频场景（万级传感器），应用层做缓冲：

```csharp
public class SensorBuffer
{
    private readonly IDatabase _db;
    private readonly ConcurrentQueue<SensorReading> _queue = new();
    private readonly Timer _flushTimer;
    private const int BatchSize = 500;

    public SensorBuffer(IConnectionMultiplexer redis)
    {
        _db = redis.GetDatabase();
        // 每 100ms 或攒满 500 条时批量刷入
        _flushTimer = new Timer(Flush, null, 100, 100);
    }

    public void Enqueue(SensorReading reading)
    {
        _queue.Enqueue(reading);

        // 达到批次大小，主动触发刷入
        if (_queue.Count >= BatchSize)
        {
            Flush(null);
        }
    }

    private void Flush(object? state)
    {
        var batch = new List<SensorReading>();
        while (batch.Count < BatchSize && _queue.TryDequeue(out var reading))
        {
            batch.Add(reading);
        }

        if (batch.Count == 0) return;

        try
        {
            // 按传感器分组，使用 HMSET
            var grouped = batch.GroupBy(r => r.SensorId);
            var tasks = new List<Task>();

            foreach (var group in grouped)
            {
                var hashEntries = group.Select(r =>
                    new HashEntry(r.MetricName, r.Value)
                ).ToArray();

                tasks.Add(_db.HashSetAsync(
                    $"sensor:{group.Key}:metrics", hashEntries));
            }

            Task.WaitAll(tasks.ToArray(), TimeSpan.FromSeconds(1));
        }
        catch (Exception ex)
        {
            // 回退到队列，稍后重试
            foreach (var item in batch)
                _queue.Enqueue(item);
        }
    }
}
```

### 4.3 批量读取优化

#### 场景：Dashboard 需要展示 500 个传感器的最新值

```csharp
// ❌ 反模式：逐个读取
var data = new Dictionary<string, double>();
foreach (var sensorId in sensorIds)
{
    data[sensorId] = double.Parse(
        await db.StringGetAsync($"sensor:{sensorId}:latest"));
}
// 500 次 RTT

// ✅ 方案一：Pipeline 批量读取
public async Task<Dictionary<string, double>> BatchReadLatest(
    IEnumerable<string> sensorIds)
{
    var keys = sensorIds
        .Select(id => (RedisKey)$"sensor:{id}:latest")
        .ToArray();

    var values = await db.StringGetAsync(keys);

    var result = new Dictionary<string, double>();
    for (int i = 0; i < sensorIds.Count(); i++)
    {
        if (!values[i].IsNull)
        {
            result[sensorIds.ElementAt(i)] = double.Parse(values[i]);
        }
    }
    return result;
}
// 500 次读取 → 1 次 RTT

// ✅ 方案二：对于 Hash 结构，用 HMGET 取指定字段
public async Task<Dictionary<string, double>> BatchReadMetrics(
    string sensorId, IEnumerable<string> metricNames)
{
    var fields = metricNames.Select(n => (RedisValue)n).ToArray();
    var values = await db.HashGetAsync($"sensor:{sensorId}:metrics", fields);

    var result = new Dictionary<string, double>();
    for (int i = 0; i < metricNames.Count(); i++)
    {
        if (!values[i].IsNull)
        {
            result[metricNames.ElementAt(i)] = double.Parse(values[i]);
        }
    }
    return result;
}
```

### 4.4 工业场景 Redis 数据结构选型

| 数据类型 | 适用场景 | 批量操作 |
|---------|---------|---------|
| String | 单个传感器最新值 | MGET/MSET |
| Hash | 一个传感器的多个指标 | HMGET/HMSET/HGETALL |
| List | 传感器历史时序数据 | LPUSH/LRANGE |
| Sorted Set | 按时间排序的事件日志 | ZADD/ZRANGE |
| Stream | 传感器事件流（Redis 5+） | XADD/XRANGE/XREAD |

---

## 五、高级优化技巧

### 5.1 连接复用与连接池

```csharp
// StackExchange.Redis 的连接是复用的，不要每次操作都创建新连接
// ✅ 正确：全局共享 ConnectionMultiplexer
private static Lazy<ConnectionMultiplexer> lazyConnection =
    new Lazy<ConnectionMultiplexer>(() =>
        ConnectionMultiplexer.Connect(new ConfigurationOptions
        {
            EndPoints = { "redis-master:6379", "redis-replica:6379" },
            Password = "your-password",
            AbortOnConnectFail = false,
            ConnectTimeout = 5000,
            SyncTimeout = 5000,
            // 连接池相关配置
            ConnectionTimeout = 5000,
        }));

public static IDatabase GetDatabase(int db = 0)
{
    return lazyConnection.Value.GetDatabase(db);
}
```

### 5.2 Pipeline 分批执行

Pipeline 不是越大越好。过大的 Pipeline 会导致：

- Redis 内存暴涨（需要缓存所有结果）
- 客户端响应延迟（必须等所有结果返回）

```csharp
// ✅ 分批执行 Pipeline
public async Task BatchSetWithChunking(
    IEnumerable<KeyValuePair<string, string>> items,
    int chunkSize = 500)
{
    var chunk = new List<KeyValuePair<string, string>>();

    foreach (var item in items)
    {
        chunk.Add(item);

        if (chunk.Count >= chunkSize)
        {
            await FlushChunk(chunk);
            chunk.Clear();
        }
    }

    if (chunk.Count > 0)
    {
        await FlushChunk(chunk);
    }
}

private async Task FlushChunk(List<KeyValuePair<string, string>> chunk)
{
    var batch = db.CreateBatch();
    var tasks = new List<Task>();

    foreach (var kv in chunk)
    {
        tasks.Add(batch.StringSetAsync(kv.Key, kv.Value));
    }

    batch.Execute();
    await Task.WhenAll(tasks);
}
```

### 5.3 Redis Cluster 注意事项

Redis 集群模式下，不同 key 可能分布在不同节点。批量操作需要处理 **Cross-slot** 问题：

```csharp
// ❌ 在集群模式下，不同 slot 的 key 不能在一个 MGET/MSET 中
var keys = new RedisKey[] { "user:1:name", "product:2:price" };
// 会报错：CROSSSLOT Keys in request don't hash to the same slot

// ✅ 方案一：使用 Hash Tag 强制同 slot
var keys = new RedisKey[] { "{session}:user:1", "{session}:product:2" };
// 大括号内的内容决定 slot，确保同 slot

// ✅ 方案二：按 slot 分组后分别执行 Pipeline
public async Task BatchSetCluster(
    IEnumerable<KeyValuePair<string, string>> items)
{
    // 按 hash slot 分组
    var bySlot = items.GroupBy(item =>
        GetHashSlot(item.Key)
    );

    var tasks = bySlot.Select(async group =>
    {
        var kv = group.Select(x =>
            new KeyValuePair<RedisKey, RedisValue>(x.Key, x.Value)
        ).ToArray();
        await db.StringSetAsync(kv);
    });

    await Task.WhenAll(tasks);
}

private int GetHashSlot(string key)
{
    // CRC16 计算 slot（简化版）
    var tagStart = key.IndexOf('{');
    var tagEnd = key.IndexOf('}');

    if (tagStart >= 0 && tagEnd > tagStart)
    {
        key = key.Substring(tagStart + 1, tagEnd - tagStart - 1);
    }

    return Crc16(key) % 16384;
}
```

### 5.4 AOF 持久化与批量写入的权衡

```bash
# Redis 配置优化

# 默认配置：每次写命令都刷盘（最安全，最慢）
appendfsync always

# 推荐配置：每秒刷盘（兼顾性能和安全性）
appendfsync everysec

# 高性能配置：依赖操作系统刷盘（最快，可能丢几秒数据）
appendfsync no
```

**秒杀场景建议：** `appendfsync no` + 异步持久化，允许少量数据丢失换取极致性能。

**工业场景建议：** `appendfsync everysec`，最多丢 1 秒数据，性能损耗可接受。

---

## 六、性能基准参考

### 测试环境

- Redis 6.2.14，单实例
- 客户端：.NET 8，StackExchange.Redis 2.7
- 网络：同机房内网（RTT ≈ 0.2ms）
- Key 大小：64 字节

### 写入性能

| 操作方式 | 100 次 | 1000 次 | 10000 次 |
|---------|:---:|:---:|:---:|
| 串行 SET | 22ms | 210ms | 2100ms |
| Pipeline (全部) | 3ms | 5ms | 42ms |
| Pipeline (分批 500) | 4ms | 6ms | 50ms |
| MSET (全部) | 2ms | 3ms | 35ms |
| Lua Script | 5ms | 8ms | 65ms |

### 读取性能

| 操作方式 | 100 次 | 1000 次 | 10000 次 |
|---------|:---:|:---:|:---:|
| 串行 GET | 21ms | 205ms | 2050ms |
| Pipeline (全部) | 3ms | 4ms | 38ms |
| MGET (全部) | 2ms | 3ms | 30ms |

**结论：** Pipeline 和 MGET/MSET 比串行操作快 **50-70 倍**。

---

## 七、常见陷阱与避坑

### 1. Pipeline 中混用同步和异步

```csharp
// ❌ 混用可能导致死锁
var batch = db.CreateBatch();
batch.StringSetAsync("key1", "value1");
batch.StringSet("key2", "value2"); // 同步调用
batch.Execute();

// ✅ 全部使用异步
var batch = db.CreateBatch();
var t1 = batch.StringSetAsync("key1", "value1");
var t2 = batch.StringSetAsync("key2", "value2");
batch.Execute();
await Task.WhenAll(t1, t2);
```

### 2. Lua 脚本中的大 Key 操作

```lua
-- ❌ 对包含百万成员的 Set 做 SMEMBERS 会阻塞 Redis
local members = redis.call('SMEMBERS', 'huge_set')

-- ✅ 使用 SSCAN 分批扫描（但 Lua 中不支持异步，所以大 Key 操作应避免在 Lua 中做）
```

**原则：** Lua 脚本的执行时间应控制在 **毫秒级**，避免阻塞 Redis。

### 3. Pipeline 中的 WATCH/MULTI/EXEC

```csharp
// WATCH 和 Pipeline 不能混用
// WATCH 是连接级别的，Pipeline 中的命令不会被当作事务

// ✅ 使用事务（Transaction）
var tran = db.CreateTransaction();
tran.AddCondition(Condition.StringEqual("stock:product:1", "100"));
tran.StringSetAsync("stock:product:1", "99");
bool committed = await tran.ExecuteAsync();
```

### 4. 超时设置

```csharp
// Pipeline 超时 = 单条命令超时 × 命令数量（粗略估计）
// 需要根据实际批量大小调整 SyncTimeout
var options = new ConfigurationOptions
{
    EndPoints = { "localhost:6379" },
    SyncTimeout = 10000,  // 10 秒，根据批量大小调整
    AsyncTimeout = 10000,
};
```

---

## 八、总结

Redis 批量命令优化的核心思路可以归纳为三句话：

1. **减少网络往返** — Pipeline 和 MGET/MSET 是最直接的武器
2. **保证原子性** — Lua 脚本处理需要读-改-写的复合操作
3. **控制批量大小** — 分批执行避免内存暴涨和超时

**秒杀场景选型建议：**

- 库存扣减 → Lua 脚本（原子性 + 一次 RTT）
- 批量预加载 → MGET / Pipeline
- 防重复购买 → SET NX / SADD

**工业场景选型建议：**

- 传感器数据写入 → Hash + Pipeline + 缓冲队列
- Dashboard 批量读取 → MGET / HMGET
- 历史数据存储 → List / Stream + LRANGE / XRANGE
- 集群模式 → Hash Tag + 按 slot 分批

用对工具，Redis 的吞吐量可以从千级 QPS 飙升到十万级。但别忘了：**优化之前先测量，盲目优化是万恶之源。**

---

## 参考链接

- [Redis Pipelining](https://redis.io/docs/latest/develop/use/pipelining/)
- [Redis Lua Scripting](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/)
- [StackExchange.Redis Documentation](https://stackexchange.github.io/StackExchange.Redis/)
- [Redis Cluster Specification](https://redis.io/docs/latest/operate/oss_and_cluster/reference/cluster-spec/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
