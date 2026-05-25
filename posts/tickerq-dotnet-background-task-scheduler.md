---
title: TickerQ — 基于 Source Generator 的 .NET 高性能后台任务调度器
slug: tickerq-dotnet-background-task-scheduler
description: >-
  深入解析 TickerQ 的核心技术要点：零反射、Source Generator、Cron 表达式、EF Core 持久化和实时仪表盘，以及在实际项目中的实践应用。
tags:
  - technical
added: "May 25 2026"
---

## 引言

在 .NET 生态中，后台任务调度是一个绕不开的话题。无论是定时数据同步、周期性报表生成，还是延迟消息处理，我们都需要一个可靠的调度方案。过去，`BackgroundService`、`Hangfire`、`Quartz.NET` 是主流选择，但它们各有取舍——要么配置繁琐，要么反射开销大，要么对 EF Core 集成不友好。

**TickerQ** 是一个新兴的 .NET 后台任务调度库，它的核心设计理念很明确：**快、零反射、现代 API**。本文将从技术要点和实践用途两个维度来拆解它。

> 项目地址：[github.com/Arcenox-co/TickerQ](https://github.com/Arcenox-co/TickerQ)

---

## 核心技术要点

### 1. Source Generator：零反射的任务发现

TickerQ 最突出的技术亮点是使用了 **C# Source Generator** 来替代传统的运行时反射。

传统的任务调度框架（如 Hangfire）通常在运行时通过反射扫描程序集，查找带有特定 Attribute 的方法来注册任务。这种方式有两个问题：

- **启动慢**：反射扫描程序集是 O(n) 操作，项目越大越慢
- **运行时开销**：每次调用任务方法都要通过 `MethodInfo.Invoke`，有 boxing 和 JIT 成本

TickerQ 的做法是在**编译时**通过 Source Generator 扫描代码，自动生成任务注册代码：

```csharp
// 你只需要写这样的代码
[TickerQ.Task("send-welcome-email", "*/5 * * * *")]
public class WelcomeEmailTask : ITask
{
    public async Task ExecuteAsync(CancellationToken ct)
    {
        // 发送邮件逻辑
    }
}
```

编译时，Source Generator 会自动生成类似这样的代码（伪代码）：

```csharp
// 自动生成的注册代码
partial class GeneratedTickerQRegistrations
{
    public static void RegisterAll(ITaskRegistry registry)
    {
        registry.Add<WelcomeEmailTask>("send-welcome-email", "*/5 * * * *");
    }
}
```

这意味着：**任务注册在编译时就已完成，运行时零反射、零额外开销。**

### 2. Cron 表达式 + 时间驱动执行

TickerQ 支持两种调度模式：

- **Cron 表达式**：`0 */2 * * *`（每两小时执行一次）
- **时间间隔**：`TimeSpan.FromMinutes(30)`（每 30 分钟执行一次）

底层使用了一个高效的 **优先级队列（Priority Queue）**，按下次执行时间排序。每次 Tick 时，只检查队首任务是否到期，避免全量扫描。

```csharp
builder.Services.AddTickerQ(options =>
{
    // Cron 方式
    options.AddCronTask<ReportGenerationTask>("0 2 * * *");  // 每天凌晨 2 点

    // 间隔方式
    options.AddIntervalTask<HealthCheckTask>(TimeSpan.FromMinutes(5));
});
```

### 3. EF Core 持久化

TickerQ 将任务状态（执行记录、失败次数、下次执行时间等）持久化到数据库，通过 EF Core 实现：

```csharp
builder.Services.AddTickerQ()
    .AddEntityFrameworkPersistence(options =>
        options.UseSqlServer(connectionString));
```

这带来了几个关键优势：

| 特性 | 说明 |
|------|------|
| **任务恢复** | 服务重启后自动从数据库恢复未执行的任务 |
| **分布式支持** | 多实例部署时通过数据库协调，避免重复执行 |
| **执行历史** | 保留任务执行记录，便于排查问题 |
| **失败重试** | 记录失败次数，按策略自动重试 |

### 4. 重试策略与限流

TickerQ 内置了灵活的重试机制：

```csharp
builder.Services.AddTickerQ(options =>
{
    options.AddCronTask<DataSyncTask>("0 * * * *", config =>
    {
        config
            .WithRetryPolicy(RetryPolicy.ExponentialBackoff(
                maxRetries: 3,
                initialDelay: TimeSpan.FromSeconds(5)))
            .WithRateLimit(maxExecutionsPerMinute: 10);
    });
});
```

支持的策略包括：
- **固定间隔重试**（Fixed Delay）
- **指数退避**（Exponential Backoff）
- **速率限制**（Rate Limiting）

### 5. 实时仪表盘

TickerQ 自带一个 Web Dashboard，可以实时查看：

- 所有已注册任务列表
- 任务执行状态（成功/失败/运行中）
- 手动触发或取消任务
- 执行历史和错误日志

```csharp
// 启用 Dashboard
builder.Services.AddTickerQDashboard();

app.MapTickerQDashboard("/tickerq");
```

---

## 实践用途

### 场景一：定时数据同步

```csharp
[TickerQ.Task("sync-user-data", "0 */4 * * *")]
public class UserDataSyncTask : ITask
{
    private readonly IExternalApiService _api;
    private readonly AppDbContext _db;

    public UserDataSyncTask(IExternalApiService api, AppDbContext db)
    {
        _api = api;
        _db = db;
    }

    public async Task ExecuteAsync(CancellationToken ct)
    {
        var users = await _api.FetchNewUsersAsync(ct);
        await _db.Users.AddRangeAsync(users, ct);
        await _db.SaveChangesAsync(ct);
    }
}
```

每 4 小时自动同步一次用户数据，EF Core 持久化确保服务重启后不会丢失调度状态。

### 场景二：延迟任务（邮件/通知）

```csharp
// 在业务代码中调度一个延迟任务
await _tickerQ.EnqueueAsync<SendReminderEmailTask>(
    args: new { UserId = user.Id },
    delay: TimeSpan.FromHours(24));
```

用户注册 24 小时后自动发送欢迎邮件，即使服务中途重启也会按时执行。

### 场景三：健康检查与告警

```csharp
[TickerQ.Task("system-health-check", "*/10 * * * *")]
public class HealthCheckTask : ITask
{
    public async Task ExecuteAsync(CancellationToken ct)
    {
        var metrics = await CollectMetricsAsync(ct);
        if (metrics.CpuUsage > 90)
        {
            await _alertService.NotifyAsync("CPU usage critical!", ct);
        }
    }
}
```

每 10 分钟执行一次系统健康检查，异常时自动告警。

---

## 与传统方案对比

| 特性 | TickerQ | Hangfire | Quartz.NET | BackgroundService |
|------|---------|----------|------------|-------------------|
| 反射开销 | ✅ 零（Source Generator） | ❌ 有 | ❌ 有 | ✅ 无 |
| Cron 支持 | ✅ | ✅ | ✅ | ❌ 需手动实现 |
| 持久化 | ✅ EF Core | ✅ 多存储 | ✅ ADO.NET | ❌ 需自行实现 |
| Dashboard | ✅ 内置 | ✅ 内置 | ❌ 无 | ❌ 无 |
| 分布式 | ✅ 数据库协调 | ✅ | ✅ | ❌ 需自行实现 |
| 学习成本 | 🟢 低 | 🟡 中 | 🔴 高 | 🟢 低 |

---

## 总结

TickerQ 代表了 .NET 后台任务调度的新方向——**用编译时代码生成替代运行时反射**，在保持 API 简洁的同时获得了更好的性能。

它的核心优势：

1. **零反射**：Source Generator 在编译时完成所有任务注册
2. **开箱即用**：Cron + 间隔调度 + 持久化 + 仪表盘，一套搞定
3. **现代 API**：原生支持 DI、CancellationToken、异步执行
4. **EF Core 集成**：与 .NET 生态的 ORM 方案无缝衔接

如果你正在为 .NET 项目选择后台任务调度方案，TickerQ 值得纳入候选。

---

> 参考资料：
> - [TickerQ GitHub](https://github.com/Arcenox-co/TickerQ)
> - [TickerQ 官方文档](https://tickerq.arcenox.com)
> - [TickerQ on NuGet](https://www.nuget.org/packages/TickerQ)
