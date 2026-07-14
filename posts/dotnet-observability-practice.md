---
title: .NET 可观测性实战指南 — OpenTelemetry 集成方案与实施
slug: dotnet-observability-practice
description: >-
  可观测性是现代分布式系统的基石。本文系统讲解 .NET 平台下基于 OpenTelemetry 的可观测性集成方案，涵盖日志、指标、链路追踪三大信号，结合 OpenTelemetry Collector、Prometheus、Grafana、Jaeger 等生态工具，提供从理论到落地的完整实施指南。
tags:
  - technical
added: "July 14 2026"
---

# .NET 可观测性实战指南 — OpenTelemetry 集成方案与实施

> 没有可观测性的分布式系统就像在黑暗中驾驶飞机——故障发生时你只知道"坏了"，但不知道"哪坏了"、"为什么坏"。本文手把手带你将 .NET 应用接入 OpenTelemetry 生态，构建真正的可观测性能力。

---

## 一、可观测性三大支柱

可观测性（Observability）不是一个工具，而是一种**通过外部输出推断系统内部状态的能力**。它建立在三大信号之上：

| 信号 | 含义 | .NET 核心类型 |
|------|------|---------------|
| **Logs（日志）** | 离散的事件记录，描述"发生了什么" | `ILogger<T>` |
| **Metrics（指标）** | 聚合的数值序列，描述"趋势和速率" | `Meter` / `Counter` |
| **Traces（链路追踪）** | 请求在分布式系统中的完整路径，描述"链路和耗时" | `ActivitySource` / `Activity` |

> 🔑 **关键认知**：三大信号不是独立的，它们互相关联——一个 trace 可以关联多条 log，一个 metric 可以从多条 trace 聚合而来。OpenTelemetry 通过 **Context Propagation** 将它们打通。

---

## 二、技术栈选型

本文采用以下技术栈：

| 组件 | 选型 | 说明 |
|------|------|------|
| 数据采集 | **OpenTelemetry .NET SDK** | 官方 SDK，生成三大信号 |
| 数据管道 | **OpenTelemetry Collector** | 轻量级代理，接收、处理、转发遥测数据 |
| 时序数据库 | **Prometheus** | 存储指标（Metrics） |
| 分布式追踪 | **Jaeger** | 存储和可视化链路追踪（Traces） |
| 日志存储 | **Loki** | 日志聚合系统（替代 Elasticsearch） |
| 可视化 | **Grafana** | 统一仪表盘 |

架构示意：

```
.NET App ──OTLP──► OpenTelemetry Collector ──┬──► Prometheus ──► Grafana
                                              ├──► Jaeger
                                              └──► Loki
```

---

## 三、.NET 项目集成 — 基础配置

### 3.1 安装 NuGet 包

```bash
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.Runtime

# 导出器（按需选择）
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol  # OTLP 导出
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore  # Prometheus 拉取
dotnet add package OpenTelemetry.Exporter.Console                # 调试用
```

### 3.2 配置 OpenTelemetry — 一步到位

在 `Program.cs` 中注册 OpenTelemetry：

```csharp
using OpenTelemetry.Logs;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;

var builder = WebApplication.CreateBuilder(args);

// 1. 定义 Resource（标识服务身份）
var resourceBuilder = ResourceBuilder.CreateDefault()
    .AddService("OrderService", serviceVersion: "1.0.0")
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = builder.Environment.EnvironmentName
    });

// 2. 配置日志
builder.Logging.ClearProviders();
builder.Logging.AddOpenTelemetry(options =>
{
    options.SetResourceBuilder(resourceBuilder);
    options.IncludeScopes = true;  // 关联 scope 上下文
    options.ParseStateValues = true;
    options.IncludeFormattedMessage = true;

    // 导出到 OTLP Collector
    options.AddOtlpExporter(otlp =>
    {
        otlp.Endpoint = new Uri("http://otel-collector:4317");
    });
});

// 3. 配置指标
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()       // HTTP 请求指标
            .AddHttpClientInstrumentation()       // HttpClient 指标
            .AddRuntimeInstrumentation()          // .NET 运行时指标（GC、CPU、内存）
            .AddMeter("Microsoft.AspNetCore.Hosting")
            .AddMeter("Microsoft.AspNetCore.Server.Kestrel")
            .AddMeter("MyApp.CustomMetrics")      // 自定义指标
            .AddOtlpExporter(otlp =>
            {
                otlp.Endpoint = new Uri("http://otel-collector:4317");
            })
            // 也可以直接暴露 Prometheus 端点（不经过 Collector）
            .AddPrometheusExporter();
    })
    // 4. 配置链路追踪
    .WithTracing(tracing =>
    {
        tracing.SetResourceBuilder(resourceBuilder)
            .AddAspNetCoreInstrumentation()       // 自动追踪入站 HTTP 请求
            .AddHttpClientInstrumentation()       // 自动追踪出站 HTTP 请求
            .AddEntityFrameworkCoreInstrumentation()  // 自动追踪数据库调用
            .AddRedisInstrumentation()            // 自动追踪 Redis 调用（需要 StackExchange.Redis）
            .AddSource("MyApp.CustomTraces")      // 自定义 ActivitySource
            .SetSampler(new AlwaysOnSampler())     // 采样策略（生产环境建议调整）
            .AddOtlpExporter(otlp =>
            {
                otlp.Endpoint = new Uri("http://otel-collector:4317");
            });
    });

var app = builder.Build();
```

### 3.3 关键配置说明

| 配置项 | 说明 |
|--------|------|
| `Resource` | 标识遥测数据的来源服务，跨系统关联的基础 |
| `Sampler` | 采样策略，生产环境推荐 `TraceIdRatioBasedSampler(0.1)` 控制成本 |
| `Instrumentation` | 自动注入探针，零侵入捕获框架级遥测 |
| `Exporter` | 输出通道，OTLP 是 OpenTelemetry 原生协议 |

---

## 四、自定义埋点 — 业务级可观测性

自动采集只能覆盖基础设施层面，**真正的可观测性需要与业务逻辑绑定**。

### 4.1 自定义指标（Metrics）

定义业务指标：

```csharp
// 创建 Meter 实例（建议作为单例/DI 注入）
public static class DiagnosticNames
{
    public static readonly Meter MyMeter = new("MyApp.CustomMetrics", "1.0.0");

    // 计数器：订单创建数
    public static readonly Counter<long> OrderCreatedCounter =
        MyMeter.CreateCounter<long>("orders.created.count", description: "Number of orders created");

    // 直方图：订单处理耗时（ms）
    public static readonly Histogram<double> OrderProcessingDuration =
        MyMeter.CreateHistogram<double>("orders.processing.duration", "ms", "Order processing duration");

    // 可观测仪表盘：当前待处理订单数
    private static long _pendingOrders;
    public static readonly ObservableGauge<long> PendingOrdersGauge =
        MyMeter.CreateObservableGauge("orders.pending", () => new Measurement<long>(_pendingOrders));
}
```

在业务代码中使用：

```csharp
public class OrderService
{
    public async Task<Order> CreateOrderAsync(CreateOrderRequest request)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            // 业务逻辑...
            var order = await SaveOrderAsync(request);

            // 记录指标
            DiagnosticNames.OrderCreatedCounter.Add(1, new KeyValuePair<string, object?>("region", request.Region));
            DiagnosticNames.PendingOrdersGauge.Observe(() => GetPendingCount());

            return order;
        }
        finally
        {
            stopwatch.Stop();
            DiagnosticNames.OrderProcessingDuration.Record(
                stopwatch.ElapsedMilliseconds,
                new KeyValuePair<string, object?>("order_type", request.Type));
        }
    }
}
```

### 4.2 自定义链路追踪（Traces）

创建自定义 Span，跟踪业务操作链路：

```csharp
public class PaymentService
{
    private static readonly ActivitySource ActivitySource = new("MyApp.CustomTraces", "1.0.0");

    public async Task<PaymentResult> ProcessPaymentAsync(Order order)
    {
        // 创建子 Span
        using var activity = ActivitySource.StartActivity("Payment.Process", ActivityKind.Internal);

        if (activity is null) return await DoPayment(order);

        // 给 Span 打标签
        activity.SetTag("order.id", order.Id);
        activity.SetTag("payment.amount", order.TotalAmount);
        activity.SetTag("payment.method", order.PaymentMethod);

        try
        {
            var result = await CallPaymentGatewayAsync(order);

            if (result.Success)
                activity.SetStatus(ActivityStatusCode.Ok);
            else
                activity.SetStatus(ActivityStatusCode.Error, result.ErrorMessage);

            // 记录事件
            activity.AddEvent(new ActivityEvent("PaymentCallCompleted",
                tags: new ActivityTagsCollection
                {
                    ["payment.status"] = result.Status,
                    ["payment.gateway"] = order.PaymentMethod
                }));

            return result;
        }
        catch (Exception ex)
        {
            activity.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity.RecordException(ex);
            throw;
        }
    }
}
```

### 4.3 丰富日志上下文

```csharp
// 使用 LoggerMessage 模式（高性能结构化日志）
public static class OrderLogger
{
    private static readonly Action<ILogger, string, decimal, Exception?> OrderCreated =
        LoggerMessage.Define<string, decimal>(
            LogLevel.Information,
            EventId(1001, "OrderCreated"),
            "Order {OrderId} created with total {TotalAmount:C}");

    public static void OrderCreatedLog(this ILogger logger, string orderId, decimal totalAmount)
    {
        OrderCreated(logger, orderId, totalAmount, null);
    }
}
```

---

## 五、OpenTelemetry Collector 部署

直接向 Jaeger/Prometheus 发送数据是反模式。引入 Collector 作为统一网关：

### 5.1 Docker Compose 配置

```yaml
version: "3.8"

services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Collector 自身指标

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    ports:
      - "16686:16686"  # UI
      - "4317:4317"    # OTLP 接收
      - "4318:4318"

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
```

### 5.2 Collector 配置

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024
  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
  attributes:
    actions:
      - key: environment
        value: production
        action: upsert
  filter:
    error_mode: ignore
    metrics:
      metric:
        - 'IsMatch(name, "http.server.duration")'  # 按需过滤指标
  # 采样处理器（降低 trace 存储成本）
  probabilistic_sampler:
    sampling_percentage: 10  # 10% 采样

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    resource_to_telemetry_conversion:
      enabled: true
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true
  otlphttp/loki:
    endpoint: "http://loki:3100/otlp"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, probabilistic_sampler]
      exporters: [otlp/jaeger]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch, filter]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [otlphttp/loki]
```

---

## 六、高级实践

### 6.1 自定义指标聚合 — 更合理的分布记录

不是所有耗时都适合用平均值。使用**直方图（Histogram）** 自定义分桶：

```csharp
// 注册时自定义桶边界
metrics.AddMeter("MyApp.CustomMetrics")
    .AddView("orders.processing.duration", new ExplicitBucketHistogramConfiguration
    {
        Boundaries = [5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
    });

// HTTP 请求耗时也自定义桶
metrics.AddView("http.server.request.duration", new ExplicitBucketHistogramConfiguration
{
    Boundaries = [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
});
```

### 6.2 上下文传播 — 打通日志 × 链路

让日志自动携带 TraceId：

```csharp
// 通过 OpenTelemetry LoggerProvider 自动注入 TraceId
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeFormattedMessage = true;
    options.IncludeScopes = true;

    // 自定义日志处理器，自动附加 traceId
    options.AddProcessor(new TraceLogProcessor());
});

public class TraceLogProcessor : BaseProcessor<LogRecord>
{
    public override void OnEnd(LogRecord data)
    {
        var ctx = System.Diagnostics.Activity.Current?.Context;
        if (ctx.HasValue)
        {
            data.Attributes = data.Attributes?.Append(new KeyValuePair<string, object?>(
                "trace.id", ctx.Value.TraceId.ToHexString())).ToArray();
            data.Attributes = data.Attributes?.Append(new KeyValuePair<string, object?>(
                "span.id", ctx.Value.SpanId.ToHexString())).ToArray();
        }
    }
}
```

### 6.3 健康检查与可观测性联动

```csharp
// 自定义健康检查，暴露为可观测指标
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database")
    .AddCheck<RedisHealthCheck>("redis")
    .AddCheck<PaymentGatewayHealthCheck>("payment_gateway");

// 将健康检查结果暴露为 Prometheus 指标
app.MapPrometheusScrapingEndpoint();
app.MapHealthChecks("/healthz");
app.MapHealthChecks("/readyz");
```

### 6.4 生产环境 — 性能与成本考量

| 关注点 | 建议 |
|--------|------|
| **采样** | 生产环境不要全量采样。使用 `TraceIdRatioBasedSampler(0.1)` + 头部采样 |
| **指标分桶** | 合理设置 Histogram 桶边界，减少无关维度的基数 |
| **日志级别** | 生产环境使用 `Warning` 及以上级别，Debug/Trace 通过动态开关控制 |
| **批处理** | Collector 端配置 `batch` 处理器，合并发送减少 IO |
| **内存限制** | Collector 配置 `memory_limiter`，防止 OOM |
| **告警** | 基于 Prometheus 指标配置 AlertManager，不依赖全量日志 |

```csharp
// 动态日志级别（无需重启）
// 集成 OpenTelemetry 的动态可观测性
builder.Logging.AddOpenTelemetry(options =>
{
    options.IncludeScopes = true;
    options.ParseStateValues = true;
    options.IncludeFormattedMessage = true;

    // OpenTelemetry 标准日志处理器
    options.AddOtlpExporter();

    // 从配置读取日志级别（支持热更新）
}).Services.ConfigureOpenTelemetryLoggerProvider((sp, providerOptions) =>
{
    // 可以通过 IOptionsMonitor 动态修改
});
```

---

## 七、Grafana 仪表盘配置

### 7.1 Grafana 数据源配置

```yaml
# grafana/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true

  - name: Jaeger
    type: jaeger
    url: http://jaeger:16686

  - name: Loki
    type: loki
    url: http://loki:3100
```

### 7.2 核心监控面板推荐

- **RED 方法面板**：Rate（请求速率）、Errors（错误率）、Duration（延迟分布）
- **USE 方法面板**：Utilization（CPU/内存利用率）、Saturation（饱和队列长度）、Errors
- **运行时面板**：GC 暂停耗时、线程池状态、JIT 编译耗时
- **业务面板**：订单量、支付成功率、各 API 端点延迟

> 📊 RED 和 USE 方法是 Google SRE 推荐的监控方法论，简洁而高效。

---

## 八、完整项目结构

```
src/
├── Program.cs                       # OpenTelemetry 配置
├── Observability/
│   ├── DiagnosticNames.cs           # Meter / Counter 定义
│   ├── ActivitySources.cs           # ActivitySource 定义
│   └── TraceLogProcessor.cs         # 日志链路关联处理器
├── Services/
│   ├── OrderService.cs              # 业务服务（含自定义埋点）
│   └── PaymentService.cs            # 支付服务（含自定义 Span）
deploy/
├── docker-compose.yml               # Collector + Prometheus + Jaeger + Loki + Grafana
├── otel-collector-config.yaml       # Collector 管道配置
├── prometheus.yml
└── grafana/
    └── datasources/
        ├── prometheus.yaml
        ├── jaeger.yaml
        └── loki.yaml
```

---

## 九、诊断与排错

### 9.1 常见问题

| 问题 | 排查方向 |
|------|----------|
| 仪表盘没有数据 | 检查 Collector 端口（4317/4318）是否可访问；telnet 测试 |
| Trace 不完整 | 确认所有调用链的服务都启用了 OpenTelemetry，且使用相同的 TraceId 传播格式 |
| 内存飙升 | 检查是否开启了全量采样；限制 Collector `memory_limiter` |
| 日志不携带 TraceId | `IncludeScopes=true`、`ParseStateValues=true` 必须设置 |

### 9.2 调试技巧

```bash
# 查看应用端 OTLP 导出是否成功（启用 ConsoleExporter）
# 设置环境变量
OTEL_DOTNET_AUTO_TRACES_CONSOLE_EXPORTER_ENABLED=true
OTEL_DOTNET_AUTO_METRICS_CONSOLE_EXPORTER_ENABLED=true

# Collector 自身健康度
curl http://localhost:8888/metrics

# 验证 OTLP gRPC 连接
grpcurl -plaintext localhost:4317 list
```

---

## 十、总结

本文从零到一构建了 .NET 应用的完整可观测性体系：

1. **三大信号**：日志（Logs）、指标（Metrics）、链路追踪（Traces）全部覆盖
2. **自动化 + 自定义**：框架级自动探针 + 业务级手动埋点，灵活兼顾
3. **统一管道**：OpenTelemetry Collector 作为数据中枢，解耦应用与后端存储
4. **生态闭环**：Prometheus（指标）→ Jaeger（链路）→ Loki（日志）→ Grafana（可视化）

> 💡 **最佳实践建议**：从 Metrics 开始，快速获得系统 SLO 可见性；逐步引入 Traces 排查复杂跨服务问题；最后用 Logs 补充细粒度排错能力。不要试图一步到位。

可观测性不是一次性工程，而是持续演进的能力架构——开始比完美更重要。

---

*本文所有代码示例基于 .NET 9 + OpenTelemetry 1.11+*
