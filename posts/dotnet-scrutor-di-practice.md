---
title: .NET 依赖注入利器 Scrutor —— 程序集扫描、装饰器与约定注册
slug: dotnet-scrutor-di-practice
description: >-
  深入讲解 Scrutor 在 .NET 依赖注入中的三大核心能力：程序集扫描自动注册、装饰器模式、约定注册。从传统手写注册的痛点切入，通过完整示例展示如何让 DI 代码从冗长变得优雅。
tags:
  - technical
added: "June 09 2026"
---

## 引言

在 ASP.NET Core 项目中，随着业务模块不断增多，`Program.cs` 或 `Startup.cs` 里的服务注册代码往往会膨胀成这样：

```csharp
builder.Services.AddSingleton<IOrderService, OrderService>();
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<ICustomerService, CustomerService>();
builder.Services.AddScoped<IPaymentService, PaymentService>();
builder.Services.AddTransient<INotificationService, NotificationService>();
builder.Services.AddTransient<ISmsProvider, AliyunSmsProvider>();
builder.Services.AddTransient<IEmailProvider, SendGridEmailProvider>();
// ... 还有 50 行
```

每新增一个服务就要手动加一行。一旦漏注册，运行时才报 `InvalidOperationException: Unable to resolve service`。这种"体力活"在大型项目中简直是折磨。

**Scrutor** 就是为了解决这个问题而生的。它是一个开源的轻量级 NuGet 包，对 `Microsoft.Extensions.DependencyInjection` 进行了优雅的扩展，提供了两大核心能力：**程序集扫描（Assembly Scanning）** 和 **装饰器（Decoration）**。

本文会带你从零开始，系统地掌握 Scrutor 的用法，理解它为什么能让 DI 代码从冗长变得优雅。

---

## Scrutor 是什么？

Scrutor（读作 "structure"）由 Kristian Hellang 创建，是一个专门为 .NET 内建 DI 容器设计的扩展库。它的核心定位非常明确：

> **不是替代 Autofac、Lamar 等第三方容器，而是让内建 DI 容器更强大。**

安装方式很简单：

```bash
dotnet add package Scrutor
```

它提供的所有功能都通过 `IServiceCollection` 的扩展方法暴露出来，无缝集成到 ASP.NET Core 的启动流程中。

---

## 痛点一：手动注册太繁琐

### 传统方式的三大问题

1. **冗长且重复**：每新增一个服务就加一行，几十上百行注册代码充斥着 `Program.cs`
2. **容易遗漏**：写了实现类但忘了注册，编译能通过，运行时才崩溃
3. **维护困难**：修改生命周期或移除服务时，要在一大坨注册代码中手动查找

### 用 Scrutor 程序集扫描来解决

Scrutor 的 `Scan` 方法允许你按约定自动发现和注册服务：

```csharp
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses()
    .UsingRegistrationStrategy(RegistrationStrategy.Skip)
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

这段代码的含义是：

| 步骤 | 含义 |
|------|------|
| `FromAssemblyOf<Program>()` | 扫描 `Program` 所在的程序集 |
| `AddClasses()` | 获取所有非抽象、非泛型的公开类 |
| `AsImplementedInterfaces()` | 注册为该类实现的所有接口 |
| `WithScopedLifetime()` | 统一使用 Scoped 生命周期 |

就这么四行，替代了前面几十行手写注册。新增服务时，只要实现了对应的接口，就会自动被扫描到。

---

## Scrutor 程序集扫描详解

### 1. 指定扫描范围

```csharp
// 扫描单个程序集
.FromAssemblyOf<Program>()

// 扫描多个指定程序集
.FromAssemblies(typeof(Program), typeof(SharedModule.Service))

// 扫描入口程序集引用的所有程序集
.FromAssemblyDependencies(typeof(Program))

// 扫描当前 AppDomain 所有程序集
.FromApplicationDependencies()

// 扫描包含特定特性的程序集（自定义标记）
.FromAssemblyOf<IScannable>()
```

### 2. 过滤要注册的类

```csharp
// 注册所有公开非抽象非泛型类（默认行为）
.AddClasses()

// 只注册实现了特定接口的类
.AddClasses(classes => classes.AssignableTo<IOrderProcessor>())

// 只注册带特定 Attribute 的类
.AddClasses(classes => classes.Where(t =>
    t.GetCustomAttributes<ServiceAttribute>().Any()))

// 排除某些命名空间的类
.AddClasses(classes => classes.Where(t =>
    !t.Namespace!.StartsWith("Legacy")))
```

### 3. 注册策略

当扫描到的服务已经手动注册过时，需要决定如何处理冲突：

```csharp
.UsingRegistrationStrategy(RegistrationStrategy.Skip)       // 跳过已注册的
.UsingRegistrationStrategy(RegistrationStrategy.Replace)   // 替换已有的
.UsingRegistrationStrategy(RegistrationStrategy.Append)    // 追加（同一接口多个实现）
.UsingRegistrationStrategy(RegistrationStrategy.Throw)     // 抛出异常
```

### 4. 映射规则

```csharp
// 注册为该类实现的所有接口
.AsImplementedInterfaces()

// 注册为类本身（自注册）
.AsSelf()

// 同时注册为自身和所有接口
.AsSelfWithInterfaces()

// 注册为特定的基类或接口
.As<IOrderProcessor>()

// 注册为所有接口和基类
.AsImplementedInterfaces()
.AsSelf()
```

### 5. 生命周期控制

```csharp
// 统一生命周期
.WithScopedLifetime()
.WithTransientLifetime()
.WithSingletonLifetime()

// 根据接口命名约定自动判断生命周期（推荐做法）
.WithLifetimeSelector(serviceType =>
{
    if (serviceType.Name.EndsWith("Singleton"))
        return ServiceLifetime.Singleton;
    if (serviceType.Name.EndsWith("Transient"))
        return ServiceLifetime.Transient;
    return ServiceLifetime.Scoped;
})
```

---

## 实战：按约定批量注册

这是最实用的模式 —— 通过命名约定自动判断生命周期：

```csharp
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses(classes => classes.Where(t =>
        t.Namespace!.StartsWith("MyApp.Services")))

    // 命名以 "Singleton" 结尾的 → Singleton
    .UsingRegistrationStrategy(RegistrationStrategy.Skip)
    .AsImplementedInterfaces()
    .WithSingletonLifetime()

    // 继续扫描 → Transient
    .AddClasses(classes => classes.Where(t =>
        t.Namespace!.StartsWith("MyApp.Services")))
    .AsImplementedInterfaces()
    .WithTransientLifetime()

    // 继续扫描 → Scoped
    .AddClasses(classes => classes.Where(t =>
        t.Namespace!.StartsWith("MyApp.Services")))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

更优雅的方式是创建一个自定义的 `ILifetimeSelector`：

```csharp
public class ConventionLifetimeSelector : ILifetimeSelector
{
    public ServiceLifetime SelectLifetime(Type serviceType, Type implementationType)
    {
        var name = implementationType.Name;
        if (name.EndsWith("Singleton")) return ServiceLifetime.Singleton;
        if (name.EndsWith("Transient")) return ServiceLifetime.Transient;
        return ServiceLifetime.Scoped; // 默认 Scoped
    }
}

// 使用
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses()
    .AsImplementedInterfaces()
    .WithLifetimeSelector(new ConventionLifetimeSelector()));
```

这样你的服务类只需要遵循命名约定即可：

```csharp
public class OrderService : IOrderService { }            // → Scoped
public class CacheManagerSingleton : ICacheManager { }   // → Singleton
public class GuidGeneratorTransient : IGuidGenerator { } // → Transient
```

---

## 痛点二：装饰器模式难以实现

### 什么是装饰器模式？

装饰器模式允许你在不修改原有类的情况下，给它添加额外行为。在 DI 场景中，典型用例包括：

- **日志记录**：在执行前后记录日志
- **性能监控**：记录方法执行时间
- **重试逻辑**：自动重试失败的操作
- **缓存**：在真实调用前先查缓存
- **权限校验**：在执行前检查权限

### 传统实现的问题

在没有 Scrutor 的情况下，你需要手动创建装饰器类并手动注册：

```csharp
// 手工写装饰器
public class LoggingOrderService : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger<LoggingOrderService> _logger;

    public LoggingOrderService(IOrderService inner, ILogger<LoggingOrderService> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);
        var result = await _inner.CreateOrderAsync(request);
        _logger.LogInformation("Order created with ID {OrderId}", result.Id);
        return result;
    }
}

// 还要手动注册
builder.Services.AddScoped<IOrderService, OrderService>();
// 然后替换成装饰后的版本……这很容易出错
```

### 用 Scrutor 一行搞定

Scrutor 提供了 `Decorate` 扩展方法，让装饰器注册变得极其简单：

```csharp
// 先注册原始服务
builder.Services.AddScoped<IOrderService, OrderService>();

// 用一行代码添加装饰器
builder.Services.Decorate<IOrderService, LoggingOrderService>();
```

当 DI 容器解析 `IOrderService` 时，返回的实际上是 `LoggingOrderService`，而 `LoggingOrderService` 内部持有原始的 `OrderService` 实例。

---

## 装饰器进阶用法

### 1. 多层装饰（装饰器链）

Scrutor 支持链式装饰，会按照注册顺序从外到内包裹：

```csharp
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.Decorate<IOrderService, LoggingOrderService>();
builder.Services.Decorate<IOrderService, CachingOrderService>();
builder.Services.Decorate<IOrderService, RetryOrderService>();
```

解析顺序：`RetryOrderService` → `CachingOrderService` → `LoggingOrderService` → `OrderService`

这意味着：重试在最外层 → 先查缓存 → 记录日志 → 实际执行。

### 2. 使用泛型装饰器

泛型装饰器可以应用于所有实现了某个接口的服务：

```csharp
// 定义泛型装饰器
public class LoggingDecorator<T> : IQueryHandler<T> where T : IQuery
{
    private readonly IQueryHandler<T> _inner;
    private readonly ILogger<LoggingDecorator<T>> _logger;

    public LoggingDecorator(IQueryHandler<T> inner, ILogger<LoggingDecorator<T>> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<TResult> HandleAsync(T query)
    {
        var sw = Stopwatch.StartNew();
        _logger.LogInformation("Handling query {QueryType}", typeof(T).Name);
        var result = await _inner.HandleAsync(query);
        _logger.LogInformation("Query {QueryType} completed in {Elapsed}ms",
            typeof(T).Name, sw.ElapsedMilliseconds);
        return result;
    }
}

// 注册
builder.Services.Decorate(typeof(IQueryHandler<>), typeof(LoggingDecorator<>));
```

### 3. 使用工厂方法创建装饰器

```csharp
builder.Services.Decorate<IOrderService>((inner, provider) =>
{
    var logger = provider.GetRequiredService<ILogger<OrderService>>();
    var config = provider.GetRequiredService<IConfiguration>();

    return new ConditionalLoggingOrderService(inner, logger, config);
});
```

### 4. 条件装饰

Scrutor 5.0+ 支持条件装饰：

```csharp
builder.Services.Decorate<IOrderService, LoggingOrderService>(
    (serviceType, provider) => provider.GetRequiredService<IConfiguration>()
        .GetValue<bool>("EnableLogging"));
```

---

## 完整实战示例

### 场景：电商订单系统

假设我们有一个电商系统，需要为订单服务添加日志、缓存和重试。

#### 项目结构

```
MyShop/
├── Services/
│   ├── OrderService.cs          → IOrderService
│   ├── ProductService.cs        → IProductService
│   └── PaymentService.cs        → IPaymentService
├── Decorators/
│   ├── LoggingDecorator.cs      → 通用日志装饰器
│   ├── CachingDecorator.cs      → 缓存装饰器
│   └── RetryDecorator.cs        → 重试装饰器
└── Program.cs
```

#### 服务实现

```csharp
// 接口
public interface IOrderService
{
    Task<Order> CreateOrderAsync(CreateOrderRequest request);
    Task<Order?> GetOrderAsync(int orderId);
}

// 实现
public class OrderService : IOrderService
{
    private readonly ApplicationDbContext _db;

    public OrderService(ApplicationDbContext db)
    {
        _db = db;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // 业务逻辑...
        var order = new Order { CustomerId = request.CustomerId, /* ... */ };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync();
        return order;
    }

    public async Task<Order?> GetOrderAsync(int orderId)
    {
        return await _db.Orders.FindAsync(orderId);
    }
}
```

#### 装饰器

```csharp
// 日志装饰器
public class LoggingDecorator<T> : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger<LoggingDecorator<T>> _logger;

    public LoggingDecorator(IOrderService inner, ILogger<LoggingDecorator<T>> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        _logger.LogInformation("Creating order for customer {CustomerId}", request.CustomerId);
        var result = await _inner.CreateOrderAsync(request);
        _logger.LogInformation("Order created successfully: {OrderId}", result.Id);
        return result;
    }

    public async Task<Order?> GetOrderAsync(int orderId)
    {
        _logger.LogInformation("Fetching order {OrderId}", orderId);
        return await _inner.GetOrderAsync(orderId);
    }
}

// 缓存装饰器
public class CachingOrderService : IOrderService
{
    private readonly IOrderService _inner;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachingOrderService> _logger;

    public CachingOrderService(IOrderService inner, IMemoryCache cache, ILogger<CachingOrderService> logger)
    {
        _inner = inner;
        _cache = cache;
        _logger = logger;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        // 创建订单不走缓存
        return await _inner.CreateOrderAsync(request);
    }

    public async Task<Order?> GetOrderAsync(int orderId)
    {
        var cacheKey = $"order:{orderId}";
        if (_cache.TryGetValue(cacheKey, out Order? cached))
        {
            _logger.LogInformation("Cache hit for order {OrderId}", orderId);
            return cached;
        }

        var order = await _inner.GetOrderAsync(orderId);
        if (order != null)
        {
            _cache.Set(cacheKey, order, TimeSpan.FromMinutes(10));
        }
        return order;
    }
}

// 重试装饰器
public class RetryOrderService : IOrderService
{
    private readonly IOrderService _inner;
    private readonly ILogger<RetryOrderService> _logger;
    private readonly int _maxRetries;

    public RetryOrderService(IOrderService inner, ILogger<RetryOrderService> logger)
    {
        _inner = inner;
        _logger = logger;
        _maxRetries = 3;
    }

    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        for (int i = 0; i <= _maxRetries; i++)
        {
            try
            {
                return await _inner.CreateOrderAsync(request);
            }
            catch (Exception ex) when (i < _maxRetries)
            {
                _logger.LogWarning(ex, "Retry {Attempt}/{Max} for CreateOrder", i + 1, _maxRetries);
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i))); // 指数退避
            }
        }
        throw new InvalidOperationException("CreateOrder failed after retries");
    }

    public async Task<Order?> GetOrderAsync(int orderId)
    {
        return await _inner.GetOrderAsync(orderId);
    }
}
```

#### Program.cs 注册

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. 程序集扫描自动注册所有服务
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses(classes => classes
        .Where(t => t.Namespace!.StartsWith("MyShop.Services")))
    .AsImplementedInterfaces()
    .WithScopedLifetime());

// 2. 添加装饰器（注意顺序：最后注册的最外层）
builder.Services.Decorate<IOrderService, LoggingDecorator<LoggingDecorator<IOrderService>>>();
builder.Services.Decorate<IOrderService, CachingOrderService>();
builder.Services.Decorate<IOrderService, RetryOrderService>();

var app = builder.Build();
app.Run();
```

解析 `IOrderService` 时的实际调用链：

```
RetryOrderService          ← 最外层：重试
  └── CachingOrderService    ← 第二层：缓存
        └── LoggingDecorator   ← 第三层：日志
              └── OrderService   ← 核心业务
```

---

## Scrutor vs 第三方 DI 容器

| 特性 | Scrutor | Autofac | Lamar |
|------|---------|---------|-------|
| 程序集扫描 | ✅ 原生支持 | ✅ 支持 | ✅ 支持 |
| 装饰器 | ✅ 简洁 API | ⚠️ 需要手动配置 | ✅ 支持 |
| 与内建 DI 兼容 | ✅ 完全兼容 | ⚠️ 需要替换容器 | ⚠️ 需要替换容器 |
| 学习成本 | 低 | 中高 | 中 |
| 性能 | 与内建 DI 一致 | 略有差异 | 快 |
| 适合场景 | 中小项目、团队统一 | 大型复杂项目 | 需要高性能的场景 |

Scrutor 的最大优势是：**不需要替换容器**。你仍然使用 `Microsoft.Extensions.DependencyInjection`，只是多了几个扩展方法。团队成员不需要学习新的 DI 语法。

---

## Scrutor 的局限性

Scrutor 不是银弹，以下场景可能需要考虑切换到 Autofac 或 Lamar：

1. **属性注入（Property Injection）**：Scrutor 和内建 DI 都不支持
2. **子容器（Child Containers）**：内建 DI 不支持，Autofac 支持
3. **复杂的多条件注册**：当扫描规则非常复杂时，手写可能更清晰
4. **AOP 拦截**：Scrutor 没有 AOP 能力，需要配合 Castle DynamicProxy 等

---

## 最佳实践总结

1. **扫描范围尽量精确**：用 `FromAssemblyOf<T>` 指定具体程序集，避免扫描不必要的类型
2. **配合命名约定**：通过类名或接口名约定生命周期，减少配置
3. **装饰器顺序很重要**：先注册的在内层，后注册的在外层，按调用链反向思考
4. **装饰器只添加行为，不修改业务逻辑**：这是装饰器模式的核心原则
5. **不要用 Scrutor 做复杂条件注册**：如果规则太复杂，手写 `AddScoped` 反而更清晰
6. **生产环境建议开启注册验证**：启动时检查关键服务是否都被正确注册

---

## 总结

Scrutor 通过两个核心能力 —— **程序集扫描**和**装饰器** —— 解决了 .NET 内建 DI 容器中最常见的两个痛点：

- **程序集扫描**让你告别手写几十行 `AddScoped`，新增服务零配置
- **装饰器**让你用一行代码为服务添加日志、缓存、重试等横切关注点

它不替代第三方 DI 容器，而是让内建 DI 容器变得"够用"。对于大多数 .NET 项目来说，Scrutor + 内建 DI 已经足够了。

如果你的项目中还在用几十行手写注册，不妨试试 Scrutor，你的 `Program.cs` 会感谢你。

---

## 参考资料

- [Scrutor GitHub](https://github.com/khellang/Scrutor)
- [Adding decorated classes to the ASP.NET Core DI container using Scrutor — Andrew Lock](https://andrewlock.net/adding-decorated-classes-to-the-asp.net-core-di-container-using-scrutor/)
- [Introduction to Scrutor Library in .NET — Code Maze](https://code-maze.com/dotnet-dependency-injection-with-scrutor/)
- [Scrutor in .NET - Auto-Register Dependencies — CodeWithMukesh](https://codewithmukesh.com/blog/scrutor-dotnet-auto-register-dependencies/)
