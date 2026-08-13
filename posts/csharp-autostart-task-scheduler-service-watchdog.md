---
title: C# 采集程序三种开机自启动方案：任务计划、服务、看门狗怎么选
slug: csharp-autostart-task-scheduler-service-watchdog
description: >-
  工业采集程序要 7x24 小时稳定运行，开机自启动是第一步。
  任务计划、Windows 服务、看门狗三种方案各有适用场景，
  本文从实现原理、代码示例、优缺点到选型决策逐一拆解。
tags:
  - technical
added: "August 13 2026"
---

## 引言

做采集程序（数据采集、PLC 通讯、串口/网口设备对接、传感器轮询）的人都知道一个痛点：**程序不能挂，挂了没人知道，重启了又怕起不来**。

采集程序通常跑在工控机、边缘网关或者车间服务器上，特点很鲜明：

- **7×24 小时运行**，没有"下班"概念
- **开机就要跑**，最好比操作员到岗还早
- **崩溃要自愈**，不能等人工去点
- **环境复杂**，可能没有显示器、没有键盘、甚至没有域控

开机自启动就是第一道防线。常见的做法有三类：

1. **任务计划（Task Scheduler）**——Windows 自带的任务调度
2. **Windows 服务（Service）**——系统级后台服务
3. **看门狗（Watchdog）**——外部进程盯梢 + 自动拉起

这三者不是非此即彼的关系，选错了轻则天天手动重启，重则数据断采没人发现。本文会从实现原理、代码示例、优缺点、适用场景四个维度拆开讲，最后给出一套选型决策思路。

## 方案一：任务计划（Task Scheduler）

### 原理

Windows 任务计划程序是系统自带的调度器，可以在**用户登录时**、**系统启动时**、**特定时间点**触发程序运行。它本身不监控程序死活——任务跑完就结束，进程被杀了它也不管。

### 实现方式

**方式 A：命令行创建**

```bash
# 系统启动时运行（需要管理员权限）
schtasks /Create /TN "Collector" /TR "C:\app\Collector.exe" /SC ONSTART /RU SYSTEM /RL HIGHEST

# 用户登录时运行
schtasks /Create /TN "Collector" /SC ONLOGON /TR "C:\app\Collector.exe"
```

**方式 B：PowerShell**

```powershell
$action  = New-ScheduledTaskAction -Execute "C:\app\Collector.exe"
$trigger = New-ScheduledTaskTrigger -AtStartup
$settings = New-ScheduledTaskSettingsSet -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1)
Register-ScheduledTask -TaskName "Collector" -Action $action -Trigger $trigger `
    -Settings $settings -User "SYSTEM" -RunLevel Highest
```

注意 `-RestartCount 3 -RestartInterval 1` —— 这是任务计划里**唯一的"自愈"能力**：任务失败后最多自动重试 3 次，每次间隔 1 分钟。但只对"非零退出码"有效，进程被杀、程序假死（卡住不退）它都管不了。

**方式 C：C# 代码创建（安装程序里顺手做）**

```csharp
using Microsoft.Win32.TaskScheduler;

using (var ts = new TaskService())
{
    var task = ts.NewTask();
    task.RegistrationInfo.Description = "数据采集程序开机自启";
    task.Principal.RunLevel = TaskRunLevel.Highest;
    task.Principal.UserId = "SYSTEM";

    task.Triggers.Add(new BootTrigger()); // 系统启动时触发

    task.Actions.Add(new ExecAction(@"C:\app\Collector.exe", "", @"C:\app"));

    // 失败重试 3 次
    task.Settings.RestartCount = 3;
    task.Settings.RestartInterval = TimeSpan.FromMinutes(1);
    task.Settings.ExecutionTimeLimit = TimeSpan.Zero; // 不限运行时长

    ts.RootFolder.RegisterTaskDefinition("Collector", task);
}
```

需要 NuGet 包 `TaskScheduler`（`Microsoft.Win32.TaskScheduler`）。

### 优点

- **零改造**：任何现成 exe 都能挂，不用改一行代码
- **系统自带**：不用装任何依赖，工控机上开箱即用
- **有 UI**：`taskschd.msc` 图形化查看运行状态、上次运行时间、退出码，排查方便
- **权限可控**：可以指定以 SYSTEM 还是指定用户运行

### 缺点

- **不做存活监控**：进程崩溃/假死无人问津，最多靠 `RestartCount` 兜底
- **"按需运行"语义**：它认为任务执行完就结束了，对"常驻进程"的理解是拧巴的
- **启动时序不可控**：`AtStartup` 触发时可能网络/服务还没就绪，采集程序一启动就连不上 PLC
- **环境依赖**：某些精简版 Windows（如部分工控定制镜像）可能阉割了任务计划服务

### 适用场景

- 采集程序本身就是"跑一次就退出"的批处理模式（如定时导出数据）
- 现场没有 IT 支持，运维人员只会点开任务计划程序看状态
- 快速部署的临时方案，不追求高可用

## 方案二：Windows 服务（Service）

### 原理

Windows 服务由 SCM（服务控制管理器）托管，**系统启动时由 SCM 按依赖顺序拉起**，不依赖用户登录。服务崩了，SCM 可以按配置自动重启；服务还可以设置"与其他服务有依存关系"，实现启动顺序控制。

### 实现方式

现代 .NET 首选 **Worker Service + `WindowsService` 托管**：

```csharp
// 1. 项目文件
// <PackageReference Include="Microsoft.Extensions.Hosting.WindowsServices" Version="8.0.0" />

// 2. Program.cs
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddHostedService<CollectorWorker>();

// 关键：启用 Windows 服务生命周期
builder.UseWindowsService(options =>
{
    options.ServiceName = "DataCollectorService";
});

var host = builder.Build();
host.Run();
```

采集 Worker 本体：

```csharp
public class CollectorWorker : BackgroundService
{
    private readonly ILogger<CollectorWorker> _logger;

    public CollectorWorker(ILogger<CollectorWorker> logger) => _logger = logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("采集服务启动");

        // 等网络/依赖就绪
        await WaitForDependenciesAsync(stoppingToken);

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await CollectOnceAsync(stoppingToken); // 采集一轮
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "采集循环异常，继续下一轮");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task CollectOnceAsync(CancellationToken ct)
    {
        // 读 PLC、串口、传感器……具体业务
        await Task.Delay(100, ct);
    }
}
```

**安装为系统服务**（管理员权限）：

```bash
sc.exe create DataCollectorService binPath= "C:\app\Collector.exe" start= auto
sc.exe failure DataCollectorService reset= 86400 actions= restart/5000/restart/10000/restart/30000
sc.exe start DataCollectorService
```

`sc failure` 那行是重点：**进程非正常退出时，服务在 5 秒/10 秒/30 秒后依次自动重启**，失败计数 24 小时（86400 秒）后重置。这是 SCM 自带的"一级看门狗"。

想做得更完善，可以配合 `ServiceController` 在安装脚本里做服务恢复配置，或用 `Microsoft.Extensions.Hosting` 自带的 `Host.CreateApplicationBuilder` + `UseWindowsService` 一键注册（`dotnet publish` 后用 `sc` 安装即可）。

### 优点

- **真正的开机自启**：SCM 保证启动顺序，可声明对网络、数据库服务的依赖
- **无会话依赖**：不需要任何用户登录，适合无人值守工控机
- **自带故障恢复**：`sc failure` 配置重启策略，崩溃后自动拉起
- **权限模型清晰**：以 LocalSystem / NetworkService 运行，与交互桌面隔离
- **生态成熟**：日志（EventLog、文件）、健康检查、`dotnet` 生态一整套

### 缺点

- **开发成本略高**：要按 Host/BackgroundService 模型改造（其实现在很轻）
- **假死检测缺失**：进程活着但线程卡死（死锁、无响应），SCM 认为服务"运行正常"，照样不重启
- **调试麻烦**：不能直接 F5 跑，要附加调试或先在控制台模式下运行
- **部署有门槛**：需要管理员权限安装/卸载，现场运维要会 `sc` 或 `services.msc`

### 适用场景

- 正式交付的采集程序，要求"开机即跑、崩溃自动重启"
- 需要按依赖顺序启动（先网络、先数据库、先网关服务）
- 有固定服务器/工控机，一次性部署长期运行

## 方案三：看门狗（Watchdog）

### 原理

看门狗本质是**"外部哨兵"**：一个独立的小进程（或硬件），定期检查主程序是否还活着，死了就拉起来，活着的判定可以是：

- 进程是否存在
- 心跳文件/心跳端口是否还在更新
- 业务级心跳（比如最近 N 秒内是否成功采集到数据）

看门狗和主程序**必须分开部署、分开进程**——否则主程序挂了，看门狗跟着一起挂，就失去了意义。

### 实现方式

**方式 A：极简版——心跳文件 + 重启逻辑**

主程序每隔 10 秒写一次心跳：

```csharp
// 主程序里
var heartbeatFile = @"C:\app\data\heartbeat.txt";
while (!ct.IsCancellationRequested)
{
    File.WriteAllText(heartbeatFile, DateTime.Now.ToString("O"));
    await Task.Delay(TimeSpan.FromSeconds(10), ct);
}
```

看门狗（独立小 exe，自身注册为任务计划或服务）：

```csharp
// 看门狗：每分钟检查一次心跳
while (true)
{
    var stale = DateTime.UtcNow -
        File.GetLastWriteTimeUtc(heartbeatFile);

    if (stale > TimeSpan.FromSeconds(30)) // 心跳超时 = 假死
    {
        KillProcess("Collector");          // 先杀
        StartProcess(@"C:\app\Collector.exe"); // 再拉起
        Log("看门狗重启了采集程序");
    }

    Thread.Sleep(TimeSpan.FromMinutes(1));
}
```

**方式 B：业务级健康检查（最可靠）**

心跳文件只是"进程活着"，不代表"业务正常"。更好的做法是主程序把**最近一次成功采集的时间**写进状态文件/数据库：

```csharp
// 主程序每轮采集成功后
statusWriter.Update(lastSuccessAt: DateTime.Now, lastError: null);
```

看门狗检查：

```csharp
var lastSuccess = statusReader.GetLastSuccessAt();
if (DateTime.Now - lastSuccess > TimeSpan.FromMinutes(5))
{
    // 5 分钟没采到数据 = 业务假死，重启
    RestartCollector();
}
```

**方式 C：硬件看门狗**

部分工控机主板/扩展卡带硬件看门狗（如研华、凌华工控机的 WDT 定时器）。软件定期"喂狗"，如果系统级卡死（连看门狗进程都跑不动了），硬件直接强制重启整机。这是软件方案兜不住的最后一层，适合极端无人值守场景。

### 优点

- **能检测假死**：业务级心跳可以区分"进程活着但没干活"
- **自愈能力强**：不限重启次数、可自定义退避策略（指数退避防抖）
- **与主程序解耦**：主程序代码侵入小，甚至能监控多个程序
- **可扩展**：看门狗同时承担"挂了发短信/邮件/微信告警"的职责

### 缺点

- **多一个进程要维护**：看门狗自己也要自启、也要防挂（需要任务计划/服务托管）
- **心跳机制要自己设计**：心跳频率、超时阈值、重启退避策略都要调
- **误杀风险**：阈值太激进，主程序只是慢了一拍就被杀掉重启，反而丢数据
- **启动竞争**：看门狗和主程序同时开机，看门狗要先于主程序就绪，时序要处理

### 适用场景

- 采集程序偶发假死（死锁、驱动卡住、第三方库 hang）是主要故障模式
- 现场无人值守、无法接受"宕机到人发现"的窗口
- 有多个采集程序需要统一监管，看门狗做成"监管中心"

## 三方案对比

| 维度 | 任务计划 | Windows 服务 | 看门狗 |
|------|---------|-------------|--------|
| 开机自启 | ✅ 支持（AtStartup/OnLogon） | ✅ 最标准，SCM 托管 | ❌ 自身需靠前两者托管 |
| 崩溃重启 | ⚠️ 仅非零退出码，最多重试 N 次 | ✅ `sc failure` 自动重启 | ✅ 完全自控 |
| 假死检测 | ❌ 无 | ❌ 无（SCM 只看进程） | ✅ 心跳/业务级检测 |
| 启动顺序控制 | ❌ 无 | ✅ 服务依赖声明 | ⚠️ 需自己协调 |
| 开发改造量 | 零改造 | 中（Host 模型改造） | 低（加心跳代码） |
| 部署复杂度 | 低 | 中（需管理员安装） | 中（多一个进程） |
| 运维可观测性 | 中（任务计划 UI） | 高（services.msc + 事件日志） | 看实现 |
| 典型定位 | 快速/临时方案 | 正式交付标准方案 | 保命兜底方案 |

## 怎么选：一套决策思路

不要直接问"哪个最好"，先回答三个问题：

**问题 1：程序会不会"假死"？**

- 程序简单、串行逻辑、基本不会卡 → 服务就够
- 涉及第三方驱动、PLC 通讯库、老 DllImport 调用，存在 hang 风险 → 必须加看门狗

**问题 2：现场有没有 IT 运维？**

- 有运维、有域控、有统一部署工具 → 服务优先，运维用 services.msc 就能管理
- 无人值守、只有一台裸奔工控机 → 服务 + 看门狗组合，重启和告警都自动化

**问题 3：能接受多大的"宕机发现延迟"？**

- 接受分钟级 → 服务 + `sc failure` 就够
- 要求秒级自愈 + 业务级健康 → 业务心跳看门狗

**推荐组合拳（生产环境最稳）：**

```
Windows 服务（承载主程序，处理开机自启 + 崩溃重启）
    +
看门狗（独立进程，做业务心跳检测 + 假死恢复 + 告警）
    +
任务计划（只用来托管看门狗自身，AtStartup 启动）
```

这样每一层各司其职：

- 服务层解决"开机起没起来、崩了重不重试"
- 看门狗解决"活了但没干活"
- 任务计划解决"看门狗自己怎么活"

如果项目刚起步、还在验证阶段，先用任务计划跑起来；进入正式部署阶段再迁移到服务；等出现过一次"进程活着但不采集"的诡异故障后，你自然会心甘情愿加上看门狗——**每个做采集的人都会遇到这一天**。

## 踩坑清单

最后给几个实战中容易踩的坑：

1. **服务里别弹 MessageBox**：服务跑在 Session 0，UI 不可见，弹窗 = 永久挂起。要提示走日志或通知渠道。
2. **心跳文件别写在 Program Files 里**：权限问题会导致心跳写失败，看门狗误判。放数据目录或注册表。
3. **重启要防抖**：看门狗连续拉起 10 次都失败，说明环境有问题，应停止重试并发告警，而不是无限重启循环。
4. **采集前先等依赖**：开机后网络、PLC 网关未必就绪，启动时先重试连接（指数退避），别一启动就连不上就退出。
5. **记录"最后一次成功"**：任何方案都建议主程序持久化最近成功采集时间，这是所有健康判断的事实基础。
6. **测试要模拟真故障**：杀进程、拔网线、停 PLC，逐个演练看门狗/服务的恢复行为，别只在正常路径上测。

## 总结

- **任务计划**：零改造、最快上线，适合临时方案和批处理式采集，但基本没有自愈能力
- **Windows 服务**：正式交付的标准答案，SCM 托管开机自启 + 崩溃重启，缺点是管不了假死
- **看门狗**：解决假死和业务级健康问题的兜底方案，但自身也需要被托管

生产环境的最优解不是三选一，而是**服务 + 看门狗组合**：服务负责"活着"，看门狗负责"干着活"。把这三层理解透了，你的采集程序就能做到"开机自己跑、挂了自动起、假死有人管"，这才是 7×24 采集该有的样子。
