---
title: Apache PLC4X .NET 工业通信实践——用 PLCOnNet 从 C# 统一访问工业 PLC
slug: plc4x-dotnet-plconnet-practice
description: >-
  深入介绍 Apache PLC4X 在 .NET 生态的落地实现 PLCOnNet，涵盖架构设计、支持的协议（S7、Modbus、ADS、BACnet 等）、核心 API 使用（读/写/订阅）、CLI 工具与项目配置，以及实际工业场景中的最佳实践。
tags:
  - technical
added: "June 30 2026"
---

## 引言

工业自动化领域的通信协议林立：西门子的 S7、施耐德的 Modbus、倍福的 ADS、BACnet、KNX、OPC-UA……每种协议都有自己的一套 SDK 和编程模型。如果你需要在同一个应用里对接多种品牌的 PLC，代码会迅速变成一团乱麻。

**Apache PLC4X** 就是为了解决这个问题而生的。它定义了一套统一的抽象 API，屏蔽底层协议差异，让开发者用同一套接口读写任意品牌的 PLC。原本它计划支持 Java、Go、C、Python、C# 五种语言，但官方在 2022 年左右放弃了 .NET（C#）的实现。

好在社区没有放弃——[MASES Group](https://masesgroup.com) 的 **PLCOnNet** 项目接过了这个接力棒。它不是"重写一个 .NET 版"，而是通过 JCOBridge 技术桥接 Java 版的 PLC4J，在 .NET 中直接复用全部 Java 类库。这意味着**Java 版有什么功能，.NET 版就有什么功能**，零延迟跟进上游迭代。

本文是 PLC4X 在 .NET 生态的完整实战指南。

---

## 一、PLC4X 是什么？

### 1.1 核心思想

PLC4X 的核心很简单：**用统一的 API 访问不同厂商的 PLC**。

想象一下，你有一个工厂，生产线同时用着西门子 S7-1200、施耐德 Modbus 设备、倍福 CX 系列。传统做法是引入三个 SDK，分别初始化、连接、读写，错误处理逻辑也不同。

PLC4X 的做法是把这一切抽象成两个概念：

- **连接字符串（Connection String）** —— 像数据库连接字符串一样指定协议和目标地址
- **驱动（Driver）** —— 每种协议一个驱动，内部解析连接字符串、处理握手协议、转换数据类型

你只需要写一次读写逻辑，换一个连接字符串就能切换 PLC 品牌。

### 1.2 支持的协议

PLC4X 目前支持的主流协议列表（Java 和 Go 版都已实现，.NET 版通过 PLCOnNet 完全可用）：

| 协议 | 全称 | 适用设备 | 连接字符串示例 |
|------|------|---------|-------------|
| S7 | Step7 | 西门子 S7 系列 PLC | `s7://10.10.64.20` |
| Modbus | Modbus TCP/UDP/RTU/ASCII | 施耐德、ABB 等 | `modbus:tcp://192.168.1.100:502` |
| ADS/AMS | Automation Device Specification | 倍福 Beckhoff CX/BC 系列 | `ads://192.168.1.100:851` |
| BACnet/IP | 楼宇自动化协议 | 霍尼韦尔、江森自控 | `bacnet://192.168.1.100` |
| KNXnet/IP | 智能楼宇协议 | ABB、施耐德 KNX 网关 | `knxnetip://192.168.1.100` |
| OPC-UA | OPC Unified Architecture | 通用工业通信 | `opcua:tcp://192.168.1.100:4840` |
| Open-Protocol | Torque-Tools | 阿特拉斯·科普柯拧紧工具 | `open-protocol://192.168.1.100` |

此外还支持 DF1（Allen-Bradley）、EtherNet/IP 等，完整的驱动列表随版本持续增加。

---

## 二、PLCOnNet：.NET 世界的 PLC4X

### 2.1 背景

PLC4X 官方仓库的 README 写得很直白：

> C# (.Net) (not ready for usage - abandoned)

官方只保留了 Java 和 Go 两个目标语言。对 .NET 开发者来说，这意味着要么用 Java 版开个 sidecar 进程做 HTTP 桥接，要么用 gRPC 转发——都不是优雅的方案。

**PLCOnNet** 走了一条不同的路：它使用 JCOBridge 在 .NET 进程内直接加载 JVM，通过 JNI 调用 PLC4J 的 Java 类。对开发者而言，你写的每一行 C# 代码背后跑的仍然是官方 Java 驱动，但感觉上就是原生 .NET API。

### 2.2 架构

```
┌────────────────────────────────────────────────┐
│  Your .NET Application (C#/VB.NET/PowerShell)   │
├────────────────────────────────────────────────┤
│              PLCOnNet (.NET Assembly)           │
│      PlcDriverManager / PlcConnection / ...     │
├────────────────────────────────────────────────┤
│                JCOBridge (JNI Bridge)           │
├────────────────────────────────────────────────┤
│           PLC4J (Java) — Official Driver        │
│   S7 Driver | Modbus Driver | ADS Driver | ...  │
├────────────────────────────────────────────────┤
│               Java Virtual Machine (JVM)        │
└────────────────────────────────────────────────┘
```

CLR 和 JVM 运行在同一个进程中，但通过 JNI 做安全隔离——没有代码注入，不共享内存，两个运行时各自管理自己的内存和线程。既保留了 Java 驱动的成熟度，又让你能继续使用 C# 生态的工具链。

### 2.3 安装与配置

**第一步：确保安装了 JRE/JDK**

PLCOnNet 需要 JVM 运行环境。推荐 Java 11+。可以设置环境变量或在运行时通过命令行指定：

```sh
# 设置 JVM 路径（可选，JCOBridge 会自动查找）
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
```

**第二步：通过 NuGet 安装**

```sh
dotnet add package MASES.PLCOnNet
```

或者安装 CLI 工具：

```sh
dotnet tool install --global MASES.PLCOnNetCLI
```

安装项目模板（可选）：

```sh
dotnet new install MASES.PLCOnNet.Templates
```

**第三步：创建一个控制台项目**

```sh
dotnet new console -n PlcReader
cd PlcReader
dotnet add package MASES.PLCOnNet
```

---

## 三、核心 API 实战

### 3.1 连接 PLC

所有操作的入口是 `PlcDriverManager`，类似于 ADO.NET 中的 `DbProviderFactory`。传入连接字符串即可获取连接：

```csharp
using org.apache.plc4x.java;
using org.apache.plc4x.java.api;
using org.apache.plc4x.java.api.authentication;
using org.apache.plc4x.java.api.connection;
using org.apache.plc4x.java.api.messages;
using org.apache.plc4x.java.api.model;
using org.apache.plc4x.java.api.types;
using java.util.concurrent;

const string connectionString = "s7://10.10.64.20";

using var plcConnection = PlcDriverManager.Default
    .ConnectionManager
    .GetConnection(connectionString);

Console.WriteLine($"Connected: {plcConnection.IsConnected}");
```

连接字符串格式为 `协议://主机[:端口]`，各协议默认端口不同（S7 默认 102，Modbus TCP 默认 502，ADS 默认 851）。

### 3.2 读取数据

读操作是最常见的需求。PLC4X 使用**标签地址（Tag Address）** 来定位变量，每个协议有自己的地址语法。S7 的地址格式为：

| 格式 | 说明 | 示例 |
|------|------|------|
| `%I{byte}.{bit}:BOOL` | 输入位 | `%I0.2:BOOL` |
| `%Q{byte}.{bit}:BOOL` | 输出位 | `%Q0.4:BOOL` |
| `%I{byte}:BYTE` | 输入字节 | `%I2:BYTE` |
| `%Q{byte}:BYTE` | 输出字节 | `%Q0:BYTE` |
| `%DB{db}.{byte}:INT` | DB 块整数 | `%DB.DB1.4:INT` |
| `%DB{db}.{byte}:REAL` | DB 块浮点数 | `%DB.DB1.0:REAL` |
| `%DB{db}.{byte}:STRING[length]` | DB 块字符串 | `%DB.DB10.0:STRING[20]` |

完整读操作：

```csharp
using var plcConnection = PlcDriverManager.Default
    .ConnectionManager
    .GetConnection(connectionString);

if (!plcConnection.Metadata.IsReadSupported())
{
    Console.WriteLine("This connection doesn't support reading.");
    return;
}

// 构建读请求
var builder = plcConnection.ReadRequestBuilder();
builder.AddTagAddress("temperature", "%DB.DB1.0:REAL");
builder.AddTagAddress("pressure", "%DB.DB1.4:REAL");
builder.AddTagAddress("status", "%DB.DB1.8:INT");
builder.AddTagAddress("running", "%Q0.4:BOOL");

var readRequest = builder.Build() as PlcReadRequest;

// 执行（同步等待结果，超时 5 秒）
var cf = readRequest.Execute<PlcReadResponse>();
var response = cf.Get(5000, TimeUnit.MILLISECONDS);

// 解析结果
foreach (Java.Lang.String tagName in response.TagNames)
{
    if (response.GetResponseCode(tagName) == PlcResponseCode.OK)
    {
        int numValues = response.GetNumberOfValues(tagName);
        if (numValues == 1)
        {
            Console.WriteLine($"{tagName}: {response.GetObject(tagName)}");
        }
        else
        {
            Console.WriteLine($"{tagName} ({numValues} values):");
            for (int i = 0; i < numValues; i++)
            {
                Console.WriteLine($"  [{i}] {response.GetObject(tagName, i)}");
            }
        }
    }
    else
    {
        Console.WriteLine($"ERROR [{tagName}]: {response.GetResponseCode(tagName).Name()}");
    }
}
```

### 3.3 写入数据

写操作的套路和读完全对称，只是在 AddTagAddress 时多传一个 value 参数：

```csharp
using var plcConnection = PlcDriverManager.Default
    .ConnectionManager
    .GetConnection(connectionString);

if (!plcConnection.Metadata.IsWriteSupported())
{
    Console.WriteLine("This connection doesn't support writing.");
    return;
}

var builder = plcConnection.WriteRequestBuilder();
builder.AddTagAddress("setpoint", "%DB.DB10.0:REAL", 75.5f);
builder.AddTagAddress("start_cmd", "%Q0.0:BOOL", true);
builder.AddTagAddress("mode", "%DB.DB10.4:INT", 3);

var writeRequest = builder.Build() as PlcWriteRequest;
var cf = writeRequest.Execute<PlcWriteResponse>();
var response = cf.Get(5000, TimeUnit.MILLISECONDS);

foreach (Java.Lang.String tagName in response.TagNames)
{
    if (response.GetResponseCode(tagName) == PlcResponseCode.OK)
    {
        Console.WriteLine($"{tagName}: updated successfully");
    }
    else
    {
        Console.WriteLine($"ERROR [{tagName}]: {response.GetResponseCode(tagName).Name()}");
    }
}
```

数组写入也支持：

```csharp
builder.AddTagAddress("array_val", "%DB.DB20.0:INT[3]", 10, 20, 30);
```

### 3.4 订阅数据变化

工业场景常需要**实时监控**PLC 变量变化。PLC4X 支持三种订阅模式：

- **Change of State** — 值发生变化时触发
- **Cyclic** — 按固定周期轮询
- **Event** — PLC 侧主动上报的报警/事件

```csharp
using System.Collections.Concurrent;

ConcurrentDictionary<int, PlcConsumerRegistration> _registrationMap = new();

using var plcConnection = PlcDriverManager.Default
    .ConnectionManager
    .GetConnection(connectionString);

if (!plcConnection.Metadata.IsSubscribeSupported())
{
    Console.WriteLine("Subscriptions not supported.");
    return;
}

// 创建订阅请求
var builder = plcConnection.SubscriptionRequestBuilder();
builder.AddChangeOfStateTagAddress("running", "%Q0.4:BOOL");
builder.AddCyclicTagAddress("temperature", "%DB.DB1.0:REAL", 
    java.time.Duration.OfMillis(1000));  // 每秒轮询一次
builder.AddEventTagAddress("alarm", "%DB.DB100.0:BOOL");

var subscriptionRequest = builder.Build() as PlcSubscriptionRequest;
var cf = subscriptionRequest.Execute<PlcSubscriptionResponse>();
var response = cf.Get(5000, TimeUnit.MILLISECONDS);

// 注册事件处理器
var plcEvent = new Consumer<PlcSubscriptionEvent>
{
    OnAccept = e =>
    {
        foreach (Java.Lang.String tagName in e.TagNames)
        {
            Console.WriteLine($"[EVENT] {tagName} = {e.GetPlcValue(tagName)}");
        }
    }
};

foreach (PlcSubscriptionHandle handle in response.SubscriptionHandles)
{
    var reg = handle.Register(plcEvent);
    _registrationMap.GetOrAdd(reg.ConsumerId, _ => reg);
}

Console.WriteLine("Subscribed. Press Enter to stop...");
Console.ReadLine();

// 清理
foreach (var kv in _registrationMap)
{
    kv.Value.Unregister();
}
```

### 3.5 处理不同类型的 PLC

切换协议只需改**连接字符串**，读写代码**一字不改**：

```csharp
// 西门子 S7
string cString = "s7://10.10.64.20";
// 需要记住——S7 的标签地址是 %I/%Q/%DB 格式

// 施耐德 Modbus
string cString = "modbus:tcp://192.168.1.100:502";
// Modbus 的标签地址是 holding-register:400001 之类的格式

// 倍福 ADS
string cString = "ads://192.168.1.100:851";
// ADS 的标签地址是 MAIN.temperature 之类的符号名

// OPC UA
string cString = "opcua:tcp://192.168.1.100:4840";
// OPC UA 的标签地址是 ns=2;s=Machine.Temperature 之类的节点路径
```

这就是 PLC4X 的威力——一套代码适配全场。

---

## 四、CLI 工具与调试

### 4.1 交互式 REPL

PLCOnNet CLI 提供了一个交互式 shell，可用于快速调试和验证连接：

```sh
dotnet tool install --global MASES.PLCOnNetCLI
plconnet -i
```

进入 REPL 后可以直接输入 C# 表达式：

```csharp
> using org.apache.plc4x.java;
> var conn = PlcDriverManager.Default.ConnectionManager.GetConnection("s7://10.10.64.20");
> conn.IsConnected
True
>
```

这在排查连接问题时非常有用，比反复编译运行快得多。

### 4.2 Docker 镜像

PLCOnNet 也提供了 Docker 镜像，免去 JVM 配置：

```sh
docker run ghcr.io/masesgroup/plconnet -i
```

---

## 五、最佳实践

### 5.1 连接管理

**不要每次读写都创建新连接。** 建立 TCP 连接（尤其是 S7 的 ISO-on-TCP 握手）开销很大，应该复用连接：

```csharp
// 一个进程维护一个连接池（简单场景单例即可）
public class PlcConnectionPool : IDisposable
{
    private readonly PlcConnection _connection;
    private readonly SemaphoreSlim _semaphore = new(1, 1);

    public PlcConnectionPool(string connectionString)
    {
        _connection = PlcDriverManager.Default
            .ConnectionManager
            .GetConnection(connectionString);
    }

    public async Task<T> ReadAsync<T>(Func<PlcConnection, T> operation)
    {
        await _semaphore.WaitAsync();
        try
        {
            return operation(_connection);
        }
        finally
        {
            _semaphore.Release();
        }
    }

    public void Dispose() => _connection?.Dispose();
}
```

### 5.2 超时处理

网络环境复杂的工业现场，一定要设超时：

```csharp
var cf = request.Execute<TResponse>();
var response = cf.Get(5000, TimeUnit.MILLISECONDS);  // 5 秒超时
```

### 5.3 错误重试

PLC 通信偶尔会出现瞬态失败（网络抖动、PLC 繁忙），建议加上重试逻辑：

```csharp
public T RetryRead<T>(Func<T> readFunc, int maxRetries = 3)
{
    for (int i = 0; i < maxRetries; i++)
    {
        try
        {
            return readFunc();
        }
        catch (Exception ex) when (i < maxRetries - 1)
        {
            Console.WriteLine($"Read failed (attempt {i + 1}/{maxRetries}): {ex.Message}");
            Thread.Sleep(1000 * (i + 1));  // 指数退避
        }
    }
    throw new Exception($"Read failed after {maxRetries} attempts.");
}
```

### 5.4 JVM 内存配置

如果你的应用同时管理大量 PLC 连接，注意 JVM 堆内存可能成为瓶颈。PLCOnNet 支持通过 `JCOBridge` 配置参数传递 JVM 启动参数：

```sh
plconnet -i --JVMOptions "-Xmx512m"
```

在 .NET 代码中也可以设置：

```csharp
// 在创建任何 PLC4X 对象之前设置
Environment.SetEnvironmentVariable("JCOBRIDGE_JVMOPS", "-Xmx512m");
```

### 5.5 延迟加载驱动

首次调用 `GetConnection` 时会加载对应的 Java 驱动类，这可能需要几百毫秒。建议在应用启动时做一次探查连接（可以连个虚拟目标），把驱动预热好。

---

## 六、限制与注意事项

1. **需要 JRE/JDK**：PLCOnNet 通过 JNI 桥接 Java 运行时，因此运行环境必须预装 JRE 8 或 11+。这和纯粹的原生 .NET 库（如 S7.NetPlus 或 nModbus）不同，部署时要多一个依赖。

2. **启动速度**：JVM 冷启动需要 1~3 秒，如果你做的是"连一次就退出的 CLI 工具"会感觉到延迟。但对于长时间运行的守护进程（如数据采集服务），这不是问题。

3. **进程内存**：一个进程中同时运行 CLR 和 JVM，内存占用会比纯 .NET 方案高一些（大约多 100~200 MB 的基线和 JVM 堆）。对于工业上位机来说这通常不是问题，但边缘计算设备上需要注意。

4. **社区 vs 官方**：PLCOnNet 是第三方的社区项目，不是 Apache PLC4X 官方的 .NET 实现。但它已在生产环境中使用，持续跟进上游版本发布（目前基于 PLC4X 0.13.x）。

---

## 七、完整项目示例

以下是一个完整的 .NET 控制台程序，每隔 1 秒读取西门子 S7-1200 的若干变量并输出：

```csharp
using org.apache.plc4x.java;
using org.apache.plc4x.java.api;
using org.apache.plc4x.java.api.messages;
using org.apache.plc4x.java.api.types;
using java.util.concurrent;

const string connectionString = "s7://192.168.1.10";
const int intervalMs = 1000;

using var connection = PlcDriverManager.Default
    .ConnectionManager
    .GetConnection(connectionString);

Console.WriteLine($"Connected: {connection.IsConnected}");

var builder = connection.ReadRequestBuilder();
builder.AddTagAddress("temp", "%DB.DB1.0:REAL");
builder.AddTagAddress("pressure", "%DB.DB1.4:REAL");
builder.AddTagAddress("flow", "%DB.DB1.8:REAL");
builder.AddTagAddress("status", "%DB.DB1.12:INT");

var readRequest = builder.Build() as PlcReadRequest;

var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;
    cts.Cancel();
};

Console.WriteLine("Reading PLC data every 1s. Press Ctrl+C to stop.\n");

while (!cts.Token.IsCancellationRequested)
{
    try
    {
        var cf = readRequest.Execute<PlcReadResponse>();
        var response = cf.Get(5000, TimeUnit.MILLISECONDS);

        Console.WriteLine($"[{DateTime.Now:HH:mm:ss}] -------------");
        foreach (Java.Lang.String tagName in response.TagNames)
        {
            if (response.GetResponseCode(tagName) == PlcResponseCode.OK)
            {
                Console.WriteLine($"  {tagName}: {response.GetObject(tagName)}");
            }
            else
            {
                Console.WriteLine($"  {tagName}: ERROR ({response.GetResponseCode(tagName).Name()})");
            }
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Read failed: {ex.Message}");
    }

    try
    {
        await Task.Delay(intervalMs, cts.Token);
    }
    catch (OperationCanceledException) { break; }
}

Console.WriteLine("\nStopped.");
```

运行结果：

```
Connected: True
[14:32:01] -------------
  temp: 25.3
  pressure: 1.02
  flow: 34.5
  status: 1
[14:32:02] -------------
  temp: 25.4
  pressure: 1.02
  flow: 34.6
  status: 1
...
```

---

## 八、总结

PLC4X 的核心理念——**统一协议抽象**——做得很优雅。它在物联网和工业 4.0 场景中天然适配：上位机采集平台、边缘数据网关、MES 对接中间件、SCADA 系统，只要有协议异构的需求就能用到它。

对 .NET 开发者来说，PLCOnNet 是目前在 .NET 生态中使用 PLC4X 的唯一可行方案。虽然它不是"原生"的 .NET 实现，但通过 JCOBridge 桥接的方式让 .NET 和 Java 在同一个进程中协作，即插即用地继承了 Java 版的所有驱动能力和社区迭代，省去了重复开发和验证的工作。

如果你正在搭建一个需要对接多品牌 PLC 的 .NET 系统，不妨试试这个组合。

---

## 参考资源

- [PLCOnNet GitHub 仓库](https://github.com/masesgroup/PLCOnNet)
- [PLCOnNet NuGet](https://www.nuget.org/packages/MASES.PLCOnNet)
- [Apache PLC4X 官网](https://plc4x.apache.org)
- [JCOBridge](https://www.jcobridge.com)
