---
title: C# 依赖注入深度解析——从原理到生产实践
slug: csharp-dependency-injection-practice
description: >-
  系统讲解 C# 中的依赖注入：从 IoC 容器原理、Microsoft.Extensions.DependencyInjection 核心 API、服务生命周期管理，到高级场景（Keyed Services、Options 模式、工厂注入），覆盖常见陷阱与生产级最佳实践。
tags:
  - technical
added: "June 05 2026"
---

## 引言

依赖注入（Dependency Injection，DI）是 .NET 生态中最基础也最重要的设计模式之一。从 ASP.NET Core 的第一个版本开始，DI 就被内建到框架核心中，而不是像早期那样依赖第三方容器（如 Autofac、Ninject）。

但很多开发者对 DI 的理解停留在"在构造函数里传接口"的层面。一旦遇到生命周期错配、循环依赖、Scoped 服务注入 Singleton 等实际问题，就难以排查。

本文将从底层原理出发，系统讲解 C# 中依赖注入的完整知识体系，涵盖日常开发中的常见场景和进阶用法。

---

## 什么是依赖注入？

### 控制反转（IoC）与依赖注入

**控制反转（Inversion of Control）** 是一种设计原则，核心思想是：对象的依赖不由对象自己创建，而是由外部容器提供。

**依赖注入**是实现控制反转的具体手段。它包含三个角色：

- **消费者（Consumer）**：需要依赖的类
- **服务（Service）**：被依赖的接口或抽象
- **注入器（Injector）**：负责创建服务并注入给消费者的容器

### 没有 DI 的时代

```csharp
public class OrderService
{
    private readonly ILogger _logger;
    private readonly IRepository<Order> _repository;
    private readonly IEmailSender _emailSender;

    public OrderService()
    {
        // 问题 1：强耦合具体实现
        _logger = new FileLogger("logs/orders.log");
        _repository = new SqlOrderRepository("Server=localhost;...");
        _emailSender = new SmtpEmailSender("smtp.example.com", 587);
    }

    public async Task PlaceOrderAsync(Order order)
    {
        _logger.Info($"Placing order {order.Id}");
        await _repository.SaveAsync(order);
        await _emailSender.SendAsync(order.CustomerEmail, "Order confirmed");
    }
}
```

这段代码有四个致命问题：

| 问题 | 后果 |
|------|------|
| **强耦合** | 无法替换实现（比如测试时想换 InMemoryRepository） |
| **难以测试** | 每次实例化都会连数据库、发邮件 |
| **违反单一职责** | 构造函数承担了对象创建职责 |
| **生命周期混乱** | 每次 new 都创建新实例，无法共享 |

### 改造后的 DI 版本

```csharp
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly IRepository<Order> _repository;
    private readonly IEmailSender _emailSender;

    // 依赖由外部注入，不关心具体实现
    public OrderService(
        ILogger<OrderService> logger,
        IRepository<Order> repository,
        IEmailSender emailSender)
    {
        _logger = logger;
        _repository = repository;
        _emailSender = emailSender;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        _logger.LogInformation("Placing order {OrderId}", order.Id);
        await _repository.SaveAsync(order);
        await _emailSender.SendAsync(order.CustomerEmail, "Order confirmed");
    }
}
```

现在 `OrderService` 只关注业务逻辑，依赖的管理交给了容器。

---

## Microsoft.Extensions.DependencyInjection

### 核心概念

.NET 内建的 DI 容器由 `Microsoft.Extensions.DependencyInjection` 包提供，核心组件只有几个：

```
IServiceCollection  ── 服务注册表（集合）
IServiceProvider    ── 服务解析器（容器）
ServiceDescriptor   ── 单个服务的注册描述
```

### 注册与解析的基本流程

```csharp
// 1. 创建容器
var services = new ServiceCollection();

// 2. 注册服务
services.AddTransient<ILogger, ConsoleLogger>();
services.AddSingleton<IRepository<Order>, SqlOrderRepository>();
services.AddScoped<IOrderService, OrderService>();

// 3. 构建 ServiceProvider
using var serviceProvider = services.BuildServiceProvider();

// 4. 解析服务
var orderService = serviceProvider.GetRequiredService<IOrderService>();
```

在 ASP.NET Core 中，这个过程被框架封装了：

```csharp
var builder = WebApplication.CreateBuilder(args);

// 注册服务（builder.Services 就是 IServiceCollection）
builder.Services.AddScoped<IOrderService, OrderService>();

var app = builder.Build();

// 运行时由框架自动解析注入
```

---

## 服务生命周期（核心！）

这是 DI 中最容易出错的部分。.NET 提供三种生命周期：

### Transient（瞬时）

每次请求都创建新实例。

```csharp
services.AddTransient<INotificationService, NotificationService>();
```

**适用场景：** 轻量级、无状态的服务。

```
Resolve 1 → new NotificationService()  // 实例 A
Resolve 2 → new NotificationService()  // 实例 B（不同的！）
Resolve 3 → new NotificationService()  // 实例 C
```

### Scoped（作用域）

同一个作用域（Scope）内共享同一个实例。在 Web 应用中，**一个 HTTP 请求就是一个 Scope**。

```csharp
services.AddScoped<IUnitOfWork, UnitOfWork>();
```

**适用场景：** 需要在一次请求内保持状态的服务，比如数据库上下文、工作单元。

```
Scope 1:
  Resolve → UnitOfWork 实例 A
  Resolve → UnitOfWork 实例 A（同一个！）

Scope 2:
  Resolve → UnitOfWork 实例 B（新的 Scope，新的实例）
```

### Singleton（单例）

整个应用生命周期内只有一个实例，第一次请求时创建（或注册时指定实例）。

```csharp
services.AddSingleton<ICacheService, MemoryCacheService>();
```

**适用场景：** 缓存服务、配置读取器、连接池等全局共享的服务。

```
Resolve 1 → new MemoryCacheService()  // 实例 A
Resolve 2 → MemoryCacheService 实例 A  // 同一个！
Resolve 3 → MemoryCacheService 实例 A  // 还是同一个！
```

### 生命周期可视化对比

```
请求 1                    请求 2                    请求 3
  │                        │                        │
  ├─ Transient: A1         ├─ Transient: A2         ├─ Transient: A3
  ├─ Scoped:    B1         ├─ Scoped:    B2         ├─ Scoped:    B3
  │   (请求内共享)           │   (请求内共享)           │   (请求内共享)
  └─ Singleton: C           └─ Singleton: C           └─ Singleton: C
                            (全局唯一)                (全局唯一)
```

### ⚠️ 生命周期错配陷阱

**最危险的模式：Singleton 依赖 Scoped**

```csharp
// ❌ 错误！Singleton 服务依赖了 Scoped 服务
services.AddScoped<IUserContext, UserContext>();
services.AddSingleton<IReportGenerator, ReportGenerator>();

public class ReportGenerator : IReportGenerator
{
    private readonly IUserContext _userContext; // Scoped！

    public ReportGenerator(IUserContext userContext)
    {
        _userContext = userContext; // 这个实例会被"钉死"在第一次解析时的 Scope
    }
}
```

后果：`ReportGenerator` 在第一次解析时捕获了某个 Scope 的 `UserContext`，之后的所有请求都会使用这个"过期"的上下文。

.NET 在开发模式下会检测到这个问题并抛出 `InvalidOperationException`，但在生产模式（`ASPNETCORE_ENVIRONMENT=Production`）下默认不抛异常，导致难以排查的 Bug。

**解决方案：**

```csharp
// 方案 1：将 Singleton 降级为 Scoped
services.AddScoped<IReportGenerator, ReportGenerator>();

// 方案 2：注入 IServiceScopeFactory，按需创建 Scope
services.AddSingleton<IReportGenerator, ReportGenerator>();

public class ReportGenerator : IReportGenerator
{
    private readonly IServiceScopeFactory _scopeFactory;

    public ReportGenerator(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task GenerateReportAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var userContext = scope.ServiceProvider.GetRequiredService<IUserContext>();
        // 在正确的 Scope 内使用 Scoped 服务
    }
}
```

---

## 注册方式详解

### 基础注册

```csharp
// 接口 → 实现
services.AddTransient<IEmailSender, SmtpEmailSender>();

// 自身注册（没有接口）
services.AddSingleton<ConfigurationManager>();

// 注册已有实例
var cache = new MemoryCacheService();
services.AddSingleton<ICacheService>(cache);
```

### Factory 注册

当实例化需要逻辑时，使用工厂方法：

```csharp
services.AddSingleton<IRedisClient>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    var connectionString = config.GetConnectionString("Redis");
    return ConnectionMultiplexer.Connect(connectionString);
});
```

Factory 方法会在第一次解析时调用（对于 Transient 是每次解析时调用），可以访问其他已注册的服务。

### TryAdd 模式

```csharp
// 只有当服务未注册时才注册
services.TryAddTransient<ILogger, DefaultLogger>();

// 按类型替换（移除所有同类型的旧注册）
services.Replace(ServiceDescriptor.Transient<ILogger, BetterLogger>());
```

这在库开发中很有用——提供默认实现，同时允许用户覆盖。

### 泛型注册

```csharp
// 开放泛型——任何 T 都会匹配
services(typeof(IRepository<>), typeof(Repository<>));
services.AddTransient(typeof(IEventHandler<>), typeof(EventHandler<>));
```

---

## 构造函数注入

### 标准模式

```csharp
public class OrderProcessor
{
    private readonly IOrderRepository _repository;
    private readonly IEventPublisher _publisher;
    private readonly ILogger<OrderProcessor> _logger;

    public OrderProcessor(
        IOrderRepository repository,
        IEventPublisher publisher,
        ILogger<OrderProcessor> logger)
    {
        _repository = repository;
        _publisher = publisher;
        _logger = logger;
    }
}
```

### 可选依赖

如果某个依赖是可选的，可以注册默认实现或使用 `?` 标记可空：

```csharp
public class OrderProcessor
{
    private readonly IOptionalFeature _feature;

    // 注册时可以用 TryAdd 提供默认 null 实现
    public OrderProcessor(IOptionalFeature? feature = null)
    {
        _feature = feature ?? new NoOpOptionalFeature();
    }
}
```

更好的方式是用 `null` 注册：

```csharp
services.AddSingleton<IOptionalFeature>(sp => null);
```

### ⚠️ 避免 Service Locator 反模式

```csharp
// ❌ 反模式：Service Locator
public class BadService
{
    private readonly IServiceProvider _sp;

    public BadService(IServiceProvider sp)
    {
        _sp = sp;
    }

    public void DoWork()
    {
        // 隐藏依赖，难以发现和测试
        var repo = _sp.GetRequiredService<IRepository>();
    }
}

// ✅ 正确：显式构造函数注入
public class GoodService
{
    private readonly IRepository _repo;

    public GoodService(IRepository repo)
    {
        _repo = repo;
    }
}
```

Service Locator 的问题：
- **隐藏依赖**：从外部无法知道这个类需要什么
- **难以测试**：需要 mock 整个容器
- **延迟发现问题**：运行到那行代码时才知道缺了什么

---

## 高级场景

### Keyed Services（.NET 8+）

同一个接口有多个实现时，用 Key 区分：

```csharp
services.AddKeyedTransient<IPaymentProcessor, StripeProcessor>("stripe");
services.AddKeyedTransient<IPaymentProcessor, PayPalProcessor>("paypal");
services.AddKeyedTransient<IPaymentProcessor, AlipayProcessor>("alipay");

public class CheckoutService
{
    private readonly IKeyedServiceProvider _keyedSp;

    public CheckoutService(IKeyedServiceProvider keyedSp)
    {
        _keyedSp = keyedSp;
    }

    public async Task ProcessAsync(string paymentMethod, Payment payment)
    {
        // 按 Key 解析具体实现
        var processor = _keyedSp.GetRequiredKeyedService<IPaymentProcessor>(paymentMethod);
        await processor.ProcessAsync(payment);
    }
}
```

也可以使用 `[FromKeyedServices]` 属性注入：

```csharp
public class StripeCheckoutService
{
    private readonly IPaymentProcessor _processor;

    public StripeCheckoutService(
        [FromKeyedServices("stripe")] IPaymentProcessor processor)
    {
        _processor = processor;
    }
}
```

### Options 模式

配置信息的 DI 化管理：

```csharp
// 定义配置类
public class SmtpOptions
{
    public string Host { get; set; } = "localhost";
    public int Port { get; set; } = 25;
    public string Username { get; set; } = "";
    public string Password { get; set; } = "";
    public bool EnableSsl { get; set; } = true;
}

// 注册
builder.Services.Configure<SmtpOptions>(
    builder.Configuration.GetSection("Smtp"));

// 使用 IOptions<T>
public class SmtpEmailSender : IEmailSender
{
    private readonly SmtpOptions _options;

    public SmtpEmailSender(IOptions<SmtpOptions> options)
    {
        _options = options.Value;
    }
}
```

三种 Options 接口：

| 接口 | 行为 | 生命周期 |
|------|------|----------|
| `IOptions<T>` | 只读，启动时加载一次 | Singleton |
| `IOptionsSnapshot<T>` | 每次请求重新读取（支持热更新） | Scoped |
| `IOptionsMonitor<T>` | 实时监控变化，支持回调 | Singleton |

```csharp
// 配置变化时自动更新
public class DynamicConfigService
{
    public DynamicConfigService(IOptionsMonitor<SmtpOptions> monitor)
    {
        monitor.OnChange(newOptions =>
        {
            // 配置变了，可以做点什么
            Console.WriteLine($"Smtp Host changed to {newOptions.Host}");
        });
    }
}
```

### 工厂模式 + DI

对于需要在运行时根据参数创建对象的情况：

```csharp
// 定义工厂接口
public interface INotificationFactory
{
    INotification Create(NotificationType type);
}

// 实现
public class NotificationFactory : INotificationFactory
{
    private readonly IServiceProvider _sp;
    private readonly Dictionary<NotificationType, Type> _mapping;

    public NotificationFactory(IServiceProvider sp)
    {
        _sp = sp;
        _mapping = new Dictionary<NotificationType, Type>
        {
            [NotificationType.Email] = typeof(EmailNotification),
            [NotificationType.Sms] = typeof(SmsNotification),
            [NotificationType.Webhook] = typeof(WebhookNotification),
        };
    }

    public INotification Create(NotificationType type)
    {
        var implType = _mapping[type];
        return (INotification)ActivatorUtilities.CreateInstance(_sp, implType);
    }
}

// 注册
services.AddSingleton<INotificationFactory, NotificationFactory>();
services.AddTransient<EmailNotification>();
services.AddTransient<SmsNotification>();
services.AddTransient<WebhookNotification>();
```

`ActivatorUtilities.CreateInstance` 会从容器中解析构造函数参数，同时支持额外传入运行时参数。

### Decorator 模式（装饰器）

在不修改原有实现的情况下增强功能：

```csharp
// 原始实现
public class OrderRepository : IRepository<Order> { ... }

// 装饰器：添加缓存
public class CachingOrderRepository : IRepository<Order>
{
    private readonly IRepository<Order> _inner;
    private readonly IMemoryCache _cache;

    public CachingOrderRepository(IRepository<Order> inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order?> GetByIdAsync(int id)
    {
        if (_cache.TryGetValue($"order:{id}", out Order? order))
            return order;

        order = await _inner.GetByIdAsync(id);
        if (order != null)
            _cache.Set($"order:{id}", order, TimeSpan.FromMinutes(30));

        return order;
    }
}
```

注册时注意顺序——后注册的会被优先解析：

```csharp
services.AddScoped<IRepository<Order>, OrderRepository>();         // 先注册
services.Decorate<IRepository<Order>, CachingOrderRepository>();   // 装饰（需要 Scrutor 包）
```

或者手动注册：

```csharp
services.AddScoped<OrderRepository>();
services.AddScoped<IRepository<Order>>(sp =>
{
    var inner = sp.GetRequiredService<OrderRepository>();
    var cache = sp.GetRequiredService<IMemoryCache>();
    return new CachingOrderRepository(inner, cache);
});
```

---

## 循环依赖

### 什么是循环依赖

```csharp
public class ServiceA
{
    public ServiceA(ServiceB b) { }  // A 依赖 B
}

public class ServiceB
{
    public ServiceB(ServiceA a) { }  // B 依赖 A → 循环！
}
```

.NET 内建容器检测到循环依赖会抛出 `InvalidOperationException`。

### 解决方式

**方式 1：重构消除循环**

最常见的做法是提取第三个服务，让两者都依赖它：

```csharp
public class SharedDataService { }

public class ServiceA
{
    public ServiceA(SharedDataService shared) { }
}

public class ServiceB
{
    public ServiceB(SharedDataService shared) { }
}
```

**方式 2：使用 Lazy<T> 延迟解析**

```csharp
public class ServiceA
{
    private readonly Lazy<ServiceB> _b;

    public ServiceA(Lazy<ServiceB> b)
    {
        _b = b;  // 不会立即解析 B
    }

    public void DoWork()
    {
        var b = _b.Value;  // 用的时候才解析
    }
}
```

**方式 3：注入 IServiceProvider（不推荐，仅作应急）**

```csharp
public class ServiceA
{
    private readonly IServiceProvider _sp;

    public ServiceA(IServiceProvider sp)
    {
        _sp = sp;
    }

    public void DoWork()
    {
        var b = _sp.GetRequiredService<ServiceB>(); // 延迟解析
    }
}
```

---

## 常见第三方容器对比

虽然 .NET 内建容器已经能覆盖大部分场景，但有些高级功能需要第三方容器：

| 功能 | 内建容器 | Autofac | Scrutor |
|------|----------|---------|---------|
| 构造函数注入 | ✅ | ✅ | ✅ |
| 属性注入 | ❌ | ✅ | ❌ |
| 循环依赖检测 | ✅ | ✅ | - |
| 程序集扫描注册 | ❌ | ✅ | ✅ |
| Decorator | ❌ | ✅ | ✅ |
| Keyed Services | ✅ (.NET 8+) | ✅ | ❌ |
| Child Container | ❌ | ✅ | ❌ |
| 性能 | 极快 | 快 | - |

### 什么时候需要第三方容器？

- **程序集扫描/约定注册**：用 Scrutor（轻量级，基于内建容器）
- **复杂模块系统/子容器**：用 Autofac
- **只需要内建容器能做的**：不需要额外依赖

大多数项目用内建容器 + Scrutor 就够了。

---

## 生产级最佳实践

### 1. 构造函数只存引用，不做任何工作

```csharp
// ❌ 构造函数里有逻辑
public class BadService
{
    public BadService(IRepository repo)
    {
        _data = repo.GetAll().ToList();  // 同步 I/O！慢！
    }
}

// ✅ 只存引用
public class GoodService
{
    private readonly IRepository _repo;

    public GoodService(IRepository repo)
    {
        _repo = repo;
    }

    public async Task InitAsync()
    {
        _data = await _repo.GetAllAsync();
    }
}
```

### 2. 避免 Captive Dependency

Singleton 持有 Scoped/Transient 实例，导致被"囚禁"无法释放：

```csharp
// ❌ Singleton 持有了 Scoped 的 DbContext
public class ReportService
{
    private readonly AppDbContext _context; // Scoped！

    public ReportService(AppDbContext context) // 注册为 Singleton
    {
        _context = context; // DbContext 被囚禁在 Singleton 中
    }
}
```

### 3. 优先使用接口而非具体类

```csharp
// ❌
services.AddSingleton<SmtpEmailSender>();

// ✅
services.AddSingleton<IEmailSender, SmtpEmailSender>();
```

接口让替换和测试变得容易。

### 4. 用 `GetRequiredService<T>` 替代 `GetService<T>`

```csharp
// ❌ 返回 null，可能引发 NullReferenceException
var service = sp.GetService<IOrderService>();
service.Process(order); // 如果没注册就炸了

// ✅ 没注册直接抛异常，快速失败
var service = sp.GetRequiredService<IOrderService>();
service.Process(order);
```

### 5. 注册顺序很重要

```csharp
// 后面的注册会覆盖前面的
services.AddTransient<ILogger, ConsoleLogger>();
services.AddTransient<ILogger, FileLogger>();

// 最终解析的是 FileLogger
var logger = sp.GetRequiredService<ILogger>();
```

### 6. 合理使用 `ValidateOnBuild`

在开发环境启用验证，启动时就发现问题：

```csharp
services.AddOptions<ServiceProviderOptions>()
    .Configure(o => o.ValidateScopes = true);

var sp = services.BuildServiceProvider(
    new ServiceProviderOptions
    {
        ValidateScopes = true,      // 检测生命周期错配
        ValidateOnBuild = true      // 启动时验证所有服务能否解析
    });
```

---

## 常见错误速查

| 错误 | 原因 | 解决 |
|------|------|------|
| `Unable to resolve service for type 'X'` | 服务未注册 | 检查 `AddXxx` 调用 |
| `Cannot consume scoped service from singleton` | 生命周期错配 | 改为同生命周期或使用 `IServiceScopeFactory` |
| `Circular dependency detected` | 循环依赖 | 重构或改用 `Lazy<T>` |
| `ObjectDisposedException` on Scoped service | 在 Scope 外使用了 Scoped 服务 | 确保在正确的 Scope 内使用 |
| Singleton 里的数据"串了" | Singleton 不应持有请求级状态 | 将状态改为 Scoped 或 Transient |

---

## 总结

依赖注入不是一个"用了框架就行"的技术。理解生命周期、注册方式和常见陷阱，才能写出可测试、可维护、可扩展的代码。

核心要点回顾：

1. **构造函数注入是首选**，避免 Service Locator
2. **生命周期匹配**是 DI 中最关键的约束——Singleton 不能直接依赖 Scoped
3. **内建容器够用**，复杂场景再考虑第三方容器
4. **开发环境开启验证**，把问题拦截在启动阶段
5. **接口优于具体类**，为测试和替换留出空间

掌握这些，你就已经能应对 95% 的 DI 场景了。
