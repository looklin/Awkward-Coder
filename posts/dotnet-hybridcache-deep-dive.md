---
title: .NET HybridCache 深度解析 — L1+L2 两级缓存、惊群防护与标签失效
slug: dotnet-hybridcache-deep-dive
description: >-
  HybridCache 是 .NET 9 引入的新一代缓存库，融合内存缓存的高速与分布式缓存的共享能力，内置缓存惊群防护和标签式失效机制。本文从设计理念到实战配置，全方位解读 HybridCache 的使用与原理。
tags:
  - technical
added: "June 11 2026"
---

# .NET HybridCache 深度解析 — L1+L2 两级缓存、惊群防护与标签失效

> 缓存是性能优化的"银弹"，但传统方案各有痛点：`IMemoryCache` 快但不共享，`IDistributedCache` 共享但慢，自己手写两级缓存又容易踩坑。.NET 9 推出的 **HybridCache** 一次性解决了这些问题。本文从架构到实战，带你全面掌握这个新利器。

---

## 一、为什么需要 HybridCache？

在 HybridCache 出现之前，.NET 开发者面临着两难选择：

| 方案 | 速度 | 数据一致性 | 部署 | 代码量 |
|------|:---:|:---------:|:----:|:-----:|
| `IMemoryCache` | ⚡ 极快 | 每节点独立 | 单节点 | 少 |
| `IDistributedCache`（Redis） | 🐢 网络IO | 全局一致 | 多节点 | 多（序列化/反序列化） |
| 手写 L1 + L2 | 快 | 可配置 | 多节点 | **非常多** |

手写两级缓存需要自己处理：序列化/反序列化、缓存旁路（Cache-Aside）模式、惊群防护、两级过期策略……代码量巨大且容易出错。

**HybridCache 的核心价值：** 把 L1（内存）和 L2（分布式）整合成一个 API，开发者只需关心"取数据"，剩下的事交给框架。

```yaml
┌─────────────────────────────────────────────────┐
│                    Application                    │
│                                                   │
│  var result = await cache.GetOrCreateAsync(...)   │
│                                                   │
├─────────────────────────────────────────────────┤
│                   HybridCache                     │
│                                                   │
│  ┌──────────┐    ┌──────────────┐                 │
│  │  L1 Cache │    │  L2 Cache   │                  │
│  │ (IMemory) │◄──►│ (Redis/... )│                  │
│  │ 极快 / 本地 │    │  共享 / 持久  │                  │
│  └──────────┘    └──────────────┘                 │
└─────────────────────────────────────────────────┘
```

---

## 二、快速上手

### 安装 NuGet 包

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
```

从 .NET 9 Preview.4 开始可用，.NET 10 中已稳定。最新版本 v10.6.0。

### 基础配置

```csharp
// Program.cs
builder.Services.AddHybridCache();
```

就这么简单。默认使用进程内 `IMemoryCache` 作为 L1，不配置 L2 时退化为纯内存缓存。

### 配合 Redis 作为 L2

```csharp
// 先安装 Redis 包
// dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis

builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp";
});

builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryAbsoluteExpiration = TimeSpan.FromMinutes(30);
});
```

HybridCache 自动检测并利用已注册的 `IDistributedCache` 作为 L2 层。

### 核心 API：GetOrCreateAsync

```csharp
public class ProductService
{
    private readonly HybridCache _cache;

    public ProductService(HybridCache cache)
    {
        _cache = cache;
    }

    public async Task<ProductDto?> GetProductAsync(int id, CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async cancel =>
            {
                // 只有缓存未命中时执行（且仅一次）
                var product = await dbContext.Products
                    .FirstOrDefaultAsync(p => p.Id == id, cancel);

                return product?.ToDto();
            },
            cancellationToken: ct
        );
    }
}
```

相比传统 `IDistributedCache`，不需要手动序列化、不需要 `Get + Set` 两步操作、不需要处理缓存穿透。

---

## 三、深入特性

### 3.1 L1 / L2 双级过期

HybridCache 可以为每个缓存条目独立设置 L1 和 L2 的过期时间：

```csharp
var entryOptions = new HybridCacheEntryOptions
{
    // L2 分布式缓存过期：数据真正清理的时间
    Expiration = TimeSpan.FromMinutes(30),
    // L1 本地缓存过期：内存中保留较短时间，加速读取
    // 不设置时继承 Expiration 的值
    LocalCacheExpiration = TimeSpan.FromSeconds(30)
};

await cache.GetOrCreateAsync(key, factory, entryOptions, cancellationToken);
```

**巧妙的设计：** L1 过期时间通常远短于 L2。这意味着内存中的数据频繁刷新，但分布式缓存仍能扛住大部分读取压力。L1 过期后会从 L2 读取（比从数据库快），L2 过期后才真正回源数据库。

```
时间线：
────────┬──────────┬──────────┬──────────►
        ↑           ↑           ↑
    数据写入    L1 过期      L2 过期
                 │            │
            从 L2 读取    回源数据库
            (还是快的)    (慢的)
```

### 3.2 缓存惊群防护（Stampede Protection）

这是 HybridCache 最亮眼的内置特性。

**什么是缓存惊群？** 当某个热门缓存项过期后，大量并发请求同时发现缓存未命中，同时回源数据库——轻则数据库被打垮，重则全站雪崩。

传统方案需要在业务代码中加锁：
```csharp
// 老方案：手动加锁，代码又臭又长
private readonly SemaphoreSlim _lock = new(1, 1);

public async Task<Data> GetDataAsync(string key)
{
    if (_memoryCache.TryGetValue(key, out var data))
        return (Data)data;

    await _lock.WaitAsync();
    try
    {
        // 双重检查锁
        if (_memoryCache.TryGetValue(key, out data))
            return data;

        data = await _distributedCache.GetAsync(key);
        if (data == null)
        {
            data = await LoadFromDbAsync();
            await _distributedCache.SetAsync(key, data);
        }
        _memoryCache.Set(key, data);
        return (Data)data;
    }
    finally
    {
        _lock.Release();
    }
}
```

**HybridCache 一行代码解决：**
```csharp
return await cache.GetOrCreateAsync(key, factory, cancellationToken: ct);
```

它的底层确保：**对于同一个 key，任意时刻只有一个 factory 调用在执行**，其余并发请求自动排队等待该结果。`CancellationToken` 是所有并发调用者的合并令牌——只要任一调用者取消，整个操作取消。

### 3.3 标签式失效（Tag-based Invalidation）

传统缓存只能按 key 逐个失效。HybridCache 支持给缓存项打标签，然后按标签批量失效。

**场景：** 用户修改了个人资料，需要使该用户所有相关的缓存（用户信息、最近订单、推荐列表等）一起失效。

```csharp
// 写入时打标签
await cache.GetOrCreateAsync(
    $"user:{userId}",
    factory,
    tags: ["users", $"tenant:{tenantId}", $"user:{userId}"],
    cancellationToken: ct
);

await cache.GetOrCreateAsync(
    $"user:{userId}:orders",
    factory,
    tags: ["users", "orders", $"user:{userId}"],
    cancellationToken: ct
);

// 需要时按标签批量失效
await cache.RemoveByTagAsync($"user:{userId}", ct);
// ↑ 上面两个缓存项同时失效
```

标签失效是异步的（在网络延迟场景下表现为最终一致性），适合不需要立即强一致的场景。

### 3.4 全局配置项

```csharp
builder.Services.AddHybridCache(options =>
{
    // 全局默认过期时间
    options.DefaultEntryAbsoluteExpiration = TimeSpan.FromMinutes(30);

    // L1 最大缓存条目数（避免内存膨胀）
    options.MaximumLocalCacheEntries = 5000;

    // 单条缓存最大尺寸（字节），超过则不缓存到 L1
    options.LocalCacheEntrySizeLimit = 1 * 1024 * 1024; // 1MB

    // 是否启用压缩（默认关闭）
    options.EnableCompression = true;
});
```

---

## 四、实战：EF Core + Minimal API 集成

下面是一个完整的真实场景：商品列表接口，用 HybridCache + Redis + EF Core。

```csharp
// Program.cs
using Microsoft.Extensions.Caching.Hybrid;

var builder = WebApplication.CreateBuilder(args);

// 1. Redis L2
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
});

// 2. HybridCache
builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryAbsoluteExpiration = TimeSpan.FromMinutes(10);
});

// 3. EF Core + 其他
builder.Services.AddDbContext<AppDbContext>(/* ... */);
builder.Services.AddScoped<IProductRepository, ProductRepository>();

var app = builder.Build();

// 4. 接口
app.MapGet("/api/products/{category}", async (
    string category,
    IProductRepository repo,
    HybridCache cache,
    CancellationToken ct) =>
{
    var products = await cache.GetOrCreateAsync(
        $"products:category:{category}",
        async cancel =>
        {
            var result = await repo.GetByCategoryAsync(category, cancel);
            return result; // HybridCache 自动处理序列化
        },
        new HybridCacheEntryOptions
        {
            Expiration = TimeSpan.FromMinutes(5),
            LocalCacheExpiration = TimeSpan.FromSeconds(30)
        },
        tags: [$"category:{category}", "products"],
        cancellationToken: ct
    );

    return Results.Ok(products);
});

// 5. 商品更新时失效缓存
app.MapPut("/api/products/{id}", async (
    int id,
    UpdateProductRequest request,
    IProductRepository repo,
    HybridCache cache,
    CancellationToken ct) =>
{
    await repo.UpdateAsync(id, request, ct);
    // 按标签失效，下次请求重新加载
    await cache.RemoveByTagAsync("products", ct);

    return Results.NoContent();
});

app.Run();
```

---

## 五、从 IDistributedCache 迁移

如果你项目中大量使用了 `IDistributedCache`，迁移路径非常平滑：

| 旧模式 | 新模式 |
|--------|--------|
| `IDistributedCache _cache` 注入 | `HybridCache _cache` 注入 |
| 手动 `Serialize/Deserialize` | 自动处理 |
| `GetAsync + SetAsync` 两步 | `GetOrCreateAsync` 一步 |
| 手动加锁防惊群 | 内置惊群防护 |
| 逐 key 删除 | 按标签批量失效 |
| 仅 L2 分布式缓存 | L1 内存 + L2 分布式 |

**迁移示例：**

```csharp
// 旧代码
public async Task<UserDto?> GetUserAsync(int id)
{
    var key = $"user:{id}";
    var cached = await _distCache.GetStringAsync(key);
    if (cached != null)
        return JsonSerializer.Deserialize<UserDto>(cached);

    var user = await _db.Users.FindAsync(id);
    if (user == null) return null;

    var dto = user.ToDto();
    await _distCache.SetStringAsync(key,
        JsonSerializer.Serialize(dto),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
        });
    return dto;
}

// 新代码
public async Task<UserDto?> GetUserAsync(int id, CancellationToken ct)
{
    return await _hybridCache.GetOrCreateAsync(
        $"user:{id}",
        async cancel =>
        {
            var user = await _db.Users.FindAsync(new object[] { id }, cancel);
            return user?.ToDto();
        },
        cancellationToken: ct
    );
}
```

代码量减少 60% 以上，同时获得了 L1 内存加速和惊群防护。

---

## 六、高级话题

### 6.1 自定义序列化

默认使用 `System.Text.Json`，可以替换：

```csharp
builder.Services.AddHybridCache()
    .AddSerializerFactory<MyCustomSerializerFactory>();

public class MyCustomSerializerFactory : IHybridCacheSerializerFactory
{
    public bool TryCreateSerializer<T>(out IHybridCacheSerializer<T> serializer)
    {
        // 返回 true 表示支持
        serializer = new MyCustomSerializer<T>();
        return true;
    }
}
```

### 6.2 性能预期

根据社区基准测试（BenchmarkDotNet）：
- 纯 L1 命中：< 1μs（纳秒级）
- L2（Redis）命中：~1-5ms
- 回源数据库：10-100ms+
- 相比手写 `IDistributedCache` + `IMemoryCache`，HybridCache 的开销几乎可以忽略

### 6.3 注意事项

1. **不要过度使用标签**：标签存储在内存中，大量标签会增加内存开销
2. **CancellationToken 是合并的**：任何一个调用者取消都会导致整个操作取消
3. **L1 不能存太大对象**：合理设置 `LocalCacheEntrySizeLimit` 避免内存压力
4. **标签删除是最终一致**：在网络延迟场景下，其他节点可能短暂读到旧数据
5. **不适合高频写入场景**：缓存设计仍遵循"读多写少"的基本原则

---

## 七、总结

| 特性 | HybridCache | IMemoryCache | IDistributedCache |
|------|:-----------:|:------------:|:-----------------:|
| 速度快 | ✅ L1 | ✅ | ❌ 网络IO |
| 跨节点共享 | ✅ L2 | ❌ | ✅ |
| 惊群防护 | ✅ 内置 | ❌ | ❌ |
| 序列化自动 | ✅ | ✅ 非必须 | ❌ |
| 标签失效 | ✅ | ❌ | ❌ |
| API 简洁 | ✅ | ✅ | ❌ Get/Set/Serialize |
| 支持 Redis 等 | ✅ | ❌ | ✅ |

HybridCache 不是要替代现有的缓存方案，而是**统一了缓存编程模型**。它让简单的场景保持简单，复杂的场景变得可控。如果你正在 .NET 9+ 上构建新项目，或者对现有项目做性能优化，它应该成为你的首选缓存抽象。

你对 HybridCache 有什么看法？欢迎在评论区讨论。🚀
