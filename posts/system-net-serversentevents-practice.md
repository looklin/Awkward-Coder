---
title: System.Net.ServerSentEvents 实践指南——从 .NET 9 到 .NET 10 的原生 SSE 支持
slug: system-net-serversentevents-practice
description: >-
  全面解析 System.Net.ServerSentEvents 命名空间——从 .NET 9 的客户端解析器到 .NET 10 的服务端原生支持，涵盖核心 API、真实场景代码实践、与 WebSocket/SignalR 的对比，以及生产环境调优建议。
tags:
  - technical
added: "Jun 03 2026"
---

## 引言

在 .NET 生态中，实时通信长期由 **SignalR** 和 **WebSocket** 主导。但它们都有一个共同点——**太重**。

如果你只需要**服务器向客户端单向推送数据**（实时通知、AI 流式响应、股票行情、构建日志），SignalR 的 Hub 协议和 WebSocket 的双工握手都是过度设计。

更轻的方案叫 **SSE（Server-Sent Events）**——基于标准 HTTP 的单向事件流协议。浏览器原生支持 `EventSource` API，协议简单到几行代码就能实现。

然而，.NET 对 SSE 的原生支持一直缺位。开发者要么手写 `text/event-stream` 的拼接和解析，要么用第三方库。直到 **`.NET 9` 引入客户端解析器**、**`.NET 10` 引入服务端原生 API**，SSE 才终于成为 .NET 的**一等公民**。

本文将带你全面了解 `System.Net.ServerSentEvents` 的设计、API、实践场景，以及落地时的注意事项。

---

## 什么是 SSE？

SSE（Server-Sent Events）是一个 W3C 标准（WHATWG 规范），定义了一种**服务器向客户端单向推送事件**的协议。

核心特点：

- **基于 HTTP**：不需要额外的协议升级（不像 WebSocket 的 101 Switching Protocols）
- **单向通信**：服务器 → 客户端（如果需要双向，用 WebSocket）
- **自动重连**：客户端内置重连机制（浏览器 `EventSource` 自动处理）
- **事件 ID 机制**：断线后可从上次位置恢复
- **文本协议**：格式简单，`Content-Type: text/event-stream`

SSE 的协议格式：

```
event: message
id: 42
data: {"type":"order","status":"shipped"}

event: notification
data: {"title":"新订单","body":"用户A下单了"}

```

每条消息以空行分隔，支持 `event`（事件类型）、`id`（消息 ID）、`data`（数据负载）、`retry`（重连间隔）四个字段。

---

## .NET SSE 支持演进

| 版本 | 支持状态 |
|------|---------|
| .NET 8 及更早 | ❌ 无原生支持，需手写或第三方库 |
| .NET 9 | ✅ 客户端：`SseParser` 解析器（`System.Net.ServerSentEvents` NuGet 包） |
| .NET 10 | ✅ 服务端：`TypedResults.ServerSentEvents()` + `SseItem<T>` 强类型支持 |

### .NET 9：客户端先行

.NET 9 首先补齐了 **SSE 客户端消费能力**。新增的 `System.Net.ServerSentEvents` NuGet 包提供 `SseParser`，可以直接从 `Stream` 中解析 SSE 事件流：

```bash
dotnet add package System.Net.ServerSentEvents
```

### .NET 10：服务端闭环

.NET 10 将 SSE 提升到框架级别，新增：

- `TypedResults.ServerSentEvents()` — 最小 API 的一等返回类型
- `SseFormat` 相关 API — 服务端事件格式化
- 与 `IAsyncEnumerable<T>` 的深度集成

从此，.NET 的 SSE 故事**服务端和客户端都齐全了**。

---

## 核心 API 解析

### 客户端（.NET 9+）：SseParser

`SseParser` 是客户端的核心类型，职责很纯粹：**从流中解析 SSE 事件**。

```csharp
using System.Net.ServerSentEvents;

// 1. 发起 HTTP 请求
using var http = new HttpClient();
using var response = await http.GetAsync(
    "https://api.example.com/events",
    HttpCompletionOption.ResponseHeadersRead
);
response.EnsureSuccessStatusCode();

// 2. 创建解析器
using var stream = await response.Content.ReadAsStreamAsync();
var parser = SseParser.Create(stream);

// 3. 枚举事件（异步流）
await foreach (SseItem<string> item in parser.EnumerateAsync())
{
    Console.WriteLine($"事件类型: {item.EventType}");
    Console.WriteLine($"数据: {item.Data}");
}
```

**关键细节**：

- `HttpCompletionOption.ResponseHeadersRead` — **必须用这个**，否则 HttpClient 会等待整个响应体读完才开始处理，SSE 长连接会直接挂掉
- `SseParser.Create(stream)` — 接受任何 `Stream`，不限于 HTTP 响应流
- `EnumerateAsync()` — 返回 `IAsyncEnumerable<SseItem<T>>`，天然适配 `await foreach`

### 强类型解析

`SseParser.Create<T>()` 的泛型参数支持自定义反序列化：

```csharp
// 用 System.Text.Json 解析为强类型对象
var parser = SseParser.Create<OrderEvent>(stream);

await foreach (SseItem<OrderEvent> item in parser.EnumerateAsync())
{
    // item.Data 已经是 OrderEvent 类型
    Console.WriteLine($"订单 {item.Data.OrderId} 状态变更为 {item.Data.Status}");
}

public record OrderEvent(string OrderId, string Status, DateTime Timestamp);
```

### 按事件类型分发

SSE 支持多种事件类型，`EnumerateAsync` 可以按类型过滤：

```csharp
var parser = SseParser.Create(stream);

await foreach (SseItem<string> item in parser.EnumerateAsync("notification"))
{
    // 只处理 event: notification 类型的事件
    HandleNotification(item.Data);
}
```

### 服务端（.NET 10+）：TypedResults.ServerSentEvents()

.NET 10 的 Minimal API 可以直接返回 SSE 流：

```csharp
app.MapGet("/api/notifications", async (HttpContext ctx) =>
{
    return TypedResults.ServerSentEvents(async writer =>
    {
        // 发送事件
        await writer.WriteEventAsync("notification", "新消息来了！");
        
        // 也可以发送结构化数据
        var data = new { Title = "系统通知", Body = "构建完成" };
        await writer.WriteEventAsync("build", JsonSerializer.Serialize(data));
        
        // 心跳（防止代理超时）
        await writer.WriteCommentAsync("keep-alive");
    });
});
```

配合 `IAsyncEnumerable<T>` 使用更优雅：

```csharp
app.MapGet("/api/orders/stream", (OrderService service) =>
    TypedResults.ServerSentEvents<OrderEvent>(
        source: service.SubscribeToOrdersAsync(),
        formatEvent: (order) => new SseItem<OrderEvent>(
            order,
            order.EventType // "created" / "updated" / "shipped"
        )
    ));

// 服务层返回 IAsyncEnumerable<OrderEvent>
public async IAsyncEnumerable<OrderEvent> SubscribeToOrdersAsync()
{
    await foreach (var order in _orderChannel.Reader.ReadAllAsync())
    {
        yield return new OrderEvent(order.Id, order.Status, "created");
    }
}
```

---

## 实践场景

### 场景一：AI 大模型流式响应

这是 SSE 最火的应用场景。ChatGPT、Claude、文心一言等大模型都通过 SSE 逐字输出。

**服务端（.NET 10）**：

```csharp
app.MapPost("/api/chat", async (ChatRequest request) =>
{
    return TypedResults.ServerSentEvents(async writer =>
    {
        await using var chatClient = new ChatClient("gpt-4");
        
        await foreach (var chunk in chatClient.CompleteStreamingAsync(request.Message))
        {
            var payload = new { Content = chunk.Text, Index = chunk.Index };
            await writer.WriteEventAsync(
                "delta",
                JsonSerializer.Serialize(payload)
            );
        }
        
        // 输出完毕信号
        await writer.WriteEventAsync("done", "{\"finished\":true}");
    });
});
```

**客户端（.NET 9）**：

```csharp
using var http = new HttpClient();
using var response = await http.PostAsJsonAsync(
    "http://localhost:5000/api/chat",
    new { Message = "写一首关于春天的诗" },
    HttpCompletionOption.ResponseHeadersRead
);

using var stream = await response.Content.ReadAsStreamAsync();
var parser = SseParser.Create<ChatDelta>(stream);

Console.Write("AI 回复: ");
await foreach (var item in parser.EnumerateAsync())
{
    if (item.EventType == "delta")
    {
        Console.Write(item.Data.Content); // 逐字输出
    }
    else if (item.EventType == "done")
    {
        Console.WriteLine("\n\n✅ 生成完毕");
        break;
    }
}

public record ChatDelta(string Content, int Index);
```

效果：用户在终端看到文字逐字蹦出来，就像 ChatGPT 的打字机效果。

### 场景二：实时通知推送

**服务端**：

```csharp
app.MapGet("/api/notifications/{userId}", async (string userId, NotificationHub hub) =>
{
    return TypedResults.ServerSentEvents(async writer =>
    {
        await foreach (var notification in hub.SubscribeAsync(userId))
        {
            await writer.WriteEventAsync(
                notification.Type, // "order" / "message" / "alert"
                JsonSerializer.Serialize(notification)
            );
        }
    });
});

// NotificationHub 用 Channel 管理订阅
public class NotificationHub
{
    private readonly ConcurrentDictionary<string, Channel<Notification>> _subscribers = new();

    public async IAsyncEnumerable<Notification> SubscribeAsync(string userId)
    {
        var channel = Channel.CreateUnbounded<Notification>();
        _subscribers[userId] = channel;

        await foreach (var item in channel.Reader.ReadAllAsync())
        {
            yield return item;
        }
    }

    public async Task PushAsync(string userId, Notification notification)
    {
        if (_subscribers.TryGetValue(userId, out var channel))
        {
            await channel.Writer.WriteAsync(notification);
        }
    }
}
```

**前端（浏览器）**：

```javascript
const eventSource = new EventSource(`/api/notifications/${userId}`);

eventSource.addEventListener('order', (e) => {
    const data = JSON.parse(e.data);
    showToast(`新订单: ${data.orderId}`);
});

eventSource.addEventListener('message', (e) => {
    const data = JSON.parse(e.data);
    showNotification(data.title, data.body);
});

eventSource.addEventListener('alert', (e) => {
    const data = JSON.parse(e.data);
    showAlert(data.severity, data.message);
});
```

零依赖，浏览器原生支持，不需要任何 JS 库。

### 场景三：构建/部署日志实时查看

CI/CD 场景的经典需求：用户在网页上看构建日志，像 Jenkins/GitHub Actions 那样逐行滚动。

**服务端**：

```csharp
app.MapGet("/api/builds/{buildId}/logs", async (string buildId, BuildService service) =>
{
    return TypedResults.ServerSentEvents(async writer =>
    {
        await foreach (var logLine in service.TailBuildLogsAsync(buildId))
        {
            // 用不同的 event 类型区分日志级别
            var eventType = logLine.Level switch
            {
                LogLevel.Error => "error",
                LogLevel.Warning => "warning",
                _ => "log"
            };

            await writer.WriteEventAsync(eventType, logLine.Message);
        }

        // 构建完成
        await writer.WriteEventAsync("complete", "Build finished successfully");
    });
});
```

**前端**：

```javascript
const es = new EventSource(`/api/builds/${buildId}/logs`);

es.addEventListener('log', (e) => appendLine(e.data, 'info'));
es.addEventListener('warning', (e) => appendLine(e.data, 'warn'));
es.addEventListener('error', (e) => appendLine(e.data, 'error'));
es.addEventListener('complete', (e) => {
    appendLine(e.data, 'success');
    es.close();
});

function appendLine(text, level) {
    const div = document.createElement('div');
    div.className = `log-line log-${level}`;
    div.textContent = text;
    logContainer.appendChild(div);
    logContainer.scrollTop = logContainer.scrollHeight; // 自动滚动
}
```

### 场景四：服务端间 SSE 通信（.NET 9 客户端）

SSE 不限于浏览器。微服务之间也可以用 SSE 推送事件流：

```csharp
// 服务 A：消费服务 B 的事件流
public class OrderEventConsumer : BackgroundService
{
    private readonly HttpClient _http;
    private readonly ILogger<OrderEventConsumer> _logger;

    public OrderEventConsumer(HttpClient http, ILogger<OrderEventConsumer> logger)
    {
        _http = http;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                using var response = await _http.GetAsync(
                    "http://order-service/api/events/stream",
                    HttpCompletionOption.ResponseHeadersRead,
                    stoppingToken
                );
                response.EnsureSuccessStatusCode();

                using var stream = await response.Content.ReadAsStreamAsync(stoppingToken);
                var parser = SseParser.Create<OrderEvent>(stream);

                await foreach (var item in parser.EnumerateAsync().WithCancellation(stoppingToken))
                {
                    _logger.LogInformation(
                        "收到订单事件: {OrderId} - {Status}",
                        item.Data.OrderId,
                        item.Data.Status
                    );

                    // 处理业务逻辑...
                    await ProcessOrderEvent(item.Data);
                }
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "SSE 连接异常，5 秒后重试");
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }
    }

    private Task ProcessOrderEvent(OrderEvent evt) => Task.CompletedTask;
}

public record OrderEvent(string OrderId, string Status);
```

这个模式特别适合：**服务 B 是事件生产者，服务 A 是消费者，不需要消息队列的持久化保证，只需实时通知**。

---

## SSE vs WebSocket vs SignalR

这是最常被问到的问题。决策表如下：

| 维度 | SSE | WebSocket | SignalR |
|------|-----|-----------|---------|
| **通信方向** | 单向（服务端→客户端） | 双向 | 双向 |
| **协议** | HTTP（标准） | WebSocket（ws/wss） | 抽象（支持 WebSocket/SSE/Long Polling） |
| **浏览器支持** | 原生 `EventSource` | 原生 `WebSocket` | 需要 `@microsoft/signalr` JS 库 |
| **自动重连** | ✅ 内置 | ❌ 需手动实现 | ✅ 内置 |
| **穿透代理/防火墙** | ✅ 普通 HTTP 端口 | ⚠️ 可能被拦截 | ✅ 降级策略 |
| **实现复杂度** | 极低 | 中 | 中高 |
| **性能开销** | 低（HTTP 头部 + 文本） | 极低（帧协议） | 中（Hub 协议开销） |
| **跨语言客户端** | ✅ 任何 HTTP 客户端 | ✅ 任何 WebSocket 客户端 | ⚠️ 需 SignalR 客户端库 |
| **.NET 原生支持** | ✅ .NET 9/10 | ✅ | ✅ |

**一句话决策**：

- **只需要服务端推送到客户端** → SSE ✅（最轻量）
- **需要客户端也能发消息给服务端** → WebSocket
- **需要 .NET 生态内全栈方案（自动传输降级、Hub 分组、强类型调用）** → SignalR
- **微服务间轻量事件流** → SSE（比 gRPC Streaming 更简单）

---

## 生产环境踩坑指南

### 1. 反向代理超时

Nginx、HAProxy 等反向代理默认有请求超时（通常 60 秒）。SSE 是长连接，会被代理当作超时断开。

**Nginx 配置**：

```nginx
location /api/events/stream {
    proxy_pass http://backend;
    proxy_buffering off;          # 关闭缓冲，实时推送
    proxy_cache off;              # 关闭缓存
    proxy_read_timeout 3600s;     # 读超时 1 小时
    proxy_connect_timeout 10s;
    
    # SSE 必需的头
    proxy_set_header Connection '';
    proxy_http_version 1.1;
    chunked_transfer_encoding off;
}
```

### 2. 客户端心跳保活

如果服务端长时间没有事件发送，客户端或中间代理可能认为连接已死。定期发送心跳：

```csharp
await foreach (var item in source.WithCancellation(stoppingToken))
{
    await writer.WriteEventAsync(item.EventType, item.Data);
}

// 或者用定时心跳
_ = Task.Run(async () =>
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await writer.WriteCommentAsync("ping"); // :ping\n\n
        await Task.Delay(TimeSpan.FromSeconds(15), stoppingToken);
    }
}, stoppingToken);
```

### 3. 断线重连与事件 ID

SSE 内置断线重连机制。如果服务端发送事件时带了 `id` 字段，浏览器重连时会自动发送 `Last-Event-ID` 头：

```csharp
// 服务端发送带 ID 的事件
await writer.WriteEventAsync(
    eventType: "update",
    data: JsonSerializer.Serialize(payload),
    id: eventSequence.ToString() // 递增的事件 ID
);

// 服务端读取 Last-Event-ID，从断点恢复
string? lastEventId = Request.Headers["Last-Event-ID"];
long lastId = long.TryParse(lastEventId, out var id) ? id : 0;
```

**注意**：.NET 10 的 `SseItem` 构造函数支持传入 ID：
```csharp
new SseItem<T>(data, eventType, id)
```

### 4. 连接数限制

浏览器对同一域名的并发连接数有限制（通常 6 个）。如果页面同时开了多个 `EventSource`，可能会阻塞其他请求。

**解决方案**：
- 合并事件流（一个 SSE 连接，用 `event` 类型区分业务）
- 使用不同子域名（`events.example.com` 有独立的连接池）

### 5. 内存泄漏防范

SSE 连接持有 `IAsyncEnumerable` 和底层 Channel，如果客户端断开但服务端没有感知，Channel 会无限积压。

```csharp
// ✅ 正确：使用 CancellationToken 监听断开
app.MapGet("/stream", async (HttpContext ctx) =>
{
    return TypedResults.ServerSentEvents(async writer =>
    {
        var token = ctx.RequestAborted; // 客户端断开时触发
        
        await foreach (var item in source.WithCancellation(token))
        {
            await writer.WriteEventAsync(item.Type, item.Data);
        }
    });
});
```

### 6. 大数据量传输

SSE 协议是文本的，不适合传输大量二进制数据。如果需要传输图片或文件：

- 传 URL 引用，让客户端另行下载
- 转 Base64 嵌入 `data` 字段（体积膨胀 ~33%）
- 改用 WebSocket 或 HTTP chunked 传输

---

## 什么时候该用 System.Net.ServerSentEvents？

### ✅ 适合

- **AI 流式输出**：LLM 逐 token 返回，天然的 SSE 场景
- **实时通知/消息推送**：订单状态、系统告警、聊天消息
- **构建/部署日志实时查看**：CI/CD pipeline 日志流
- **实时监控仪表盘**：CPU/内存/业务指标推送
- **股票/加密货币行情**：高频数据流推送
- **微服务间轻量事件流**：不需要消息队列持久化的场景
- **浏览器实时 UI 更新**：不需要客户端发回的单向推送

### ❌ 不适合

- **需要客户端发消息给服务端** → WebSocket
- **需要消息持久化和回放** → Kafka / RabbitMQ
- **需要 Exactly-Once 投递保证** → 消息队列
- **大量二进制数据传输** → WebSocket / HTTP chunked
- **需要 .NET 强类型 Hub 调用** → SignalR

---

## 总结

`System.Net.ServerSentEvents` 补齐了 .NET 在轻量级实时通信上的最后一块拼图。

回顾整个演进：

```
.NET 8 及之前 → 手写 SSE（拼接字符串，手动解析）
      ↓
.NET 9       → 客户端 SseParser（从 Stream 解析事件流）
      ↓
.NET 10      → 服务端 TypedResults.ServerSentEvents()（一等 API 支持）
      ↓
现在         → 完整的双端原生 SSE 支持 🎉
```

SSE 不是什么新技术，但在 .NET 生态中，它终于不再是一个"二等公民"。如果你只需要**单向实时推送**，SSE 是比 WebSocket 和 SignalR 更简单、更轻量的选择。

记住这个决策路径：

```
需要实时通信？
├── 双向通信？
│   ├── 是 → 需要 .NET Hub 生态？
│   │   ├── 是 → SignalR ✅
│   │   └── 否 → WebSocket ✅
│   └── 否（只需服务端推送）
│       ├── 需要消息持久化？
│       │   ├── 是 → Kafka / RabbitMQ
│       │   └── 否 → SSE ✅（System.Net.ServerSentEvents）
```

在合适的场景下，SSE 的简洁本身就是最大的优势。

---

## 参考资源

- [System.Net.ServerSentEvents NuGet 包](https://www.nuget.org/packages/System.Net.ServerSentEvents)
- [SseParser 文档（Microsoft Learn）](https://learn.microsoft.com/en-us/dotnet/api/system.net.serversentevents.sseparser)
- [Server-Sent Events 规范（WHATWG）](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [.NET 10 SSE 示例项目](https://github.com/fkucukkara/server-sent-events)
- [MDN EventSource API](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)
