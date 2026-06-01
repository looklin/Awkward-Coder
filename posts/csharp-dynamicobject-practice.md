---
title: C# DynamicObject 深度解析——动态类型的实践与优势
slug: csharp-dynamicobject-practice
description: >-
  深入理解 C# DynamicObject 的实现机制，从 TryGetMember、TryInvokeMember 到底层 DLR 绑定流程，涵盖 JSON 解析、链式构建器、Mock 框架等真实场景，对比 ExpandoObject 与手写 DynamicObject 的适用边界。
tags:
  - technical
added: "June 01 2026"
---

## 引言

C# 是一门静态类型语言——编译期确定类型、方法签名、成员访问，编译器负责帮你发现错误。这带来极高的安全性和 IDE 体验，但有些场景下也会让你觉得笨重。

比如对接一个结构不定的外部 API 返回体，或者想写一个"点哪里就有哪里"的链式配置构建器。这时候如果被迫声明一大堆 `class` 和 `interface`，代码会变得冗长而脆弱。

**`DynamicObject`** 就是 C# 在这两个世界之间搭的桥——保留静态类型的编译期安全，同时在需要时打开一扇动态调用的门。

本文将从底层原理出发，拆解 `DynamicObject` 的工作机制，覆盖多个真实场景的实践，并给出与同类方案的对比与选型建议。

---

## 什么是 DynamicObject？

`System.Dynamic.DynamicObject` 是 .NET Framework 4.0 引入的基类（.NET Core / .NET 5+ 同样可用）。它实现了 `IDynamicMetaObjectProvider` 接口，允许你在**运行时**动态解析成员访问、方法调用、索引操作等。

```csharp
public class MyDynamic : DynamicObject
{
    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        // 当代码写 obj.SomeProperty 时，这个方法是入口
        result = null;
        return false; // 返回 false 表示"我没有这个成员"
    }
}

dynamic d = new MyDynamic();
var x = d.AnythingYouWant; // 编译通过，运行时由 TryGetMember 决定行为
```

关键点：**`dynamic` 是语言关键字，`DynamicObject` 是基类**。两者配合使用——`dynamic` 告诉编译器"延迟绑定"，`DynamicObject` 提供运行时行为。

---

## 底层原理：DLR 绑定流程

理解 `DynamicObject` 之前，需要先了解 **DLR（Dynamic Language Runtime）** 的调用链路。

### 编译期 → 运行期

```csharp
dynamic d = GetSomeObject();
Console.WriteLine(d.Name);
```

这段代码的编译器生成过程：

1. **编译期**：编译器看到 `dynamic`，不生成普通 IL 调用，而是生成 `CallSite` 相关代码，把 `Name` 的绑定推迟到运行时
2. **运行期第一次调用**：
   - 检查 `d` 是否实现 `IDynamicMetaObjectProvider`
   - 如果是 → 调用 `GetMetaObject()` → 拿到 `DynamicMetaObject`
   - DLR 调用 `DynamicMetaObject.BindGetMember()` → 内部转发到 `DynamicObject.TryGetMember()`
   - 返回结果，并**缓存绑定结果**
3. **后续调用**：命中缓存，直接执行，不再走绑定流程

### 为什么性能没那么差？

很多人一听到"动态"就担心性能。实际上 DLR 有一层 **CallSite 缓存**：

- 同一调用点（同一个 `d.Name`）首次绑定后会被缓存
- 后续调用走缓存路径，开销接近普通虚方法调用
- 只有**首次**或**类型变化**时才走完整绑定流程

所以 `dynamic` 不适合在高频循环里使用，但常规业务场景的性能损耗完全可以接受。

---

## 核心重载方法一览

`DynamicObject` 提供了十多个 `TryXxx` 虚方法，覆盖几乎所有动态操作：

| 方法 | 触发场景 | 示例 |
|------|----------|------|
| `TryGetMember` | 获取属性 | `d.Name` |
| `TrySetMember` | 设置属性 | `d.Name = "test"` |
| `TryInvokeMember` | 调用方法 | `d.SayHello("world")` |
| `TryGetIndex` | 索引器 | `d["key"]` |
| `TrySetIndex` | 索引器赋值 | `d["key"] = "value"` |
| `TryConvert` | 类型转换 | `(string)d` |
| `TryUnaryOperation` | 一元运算 | `!d`, `-d` |
| `TryBinaryOperation` | 二元运算 | `d + 1` |
| `TryInvoke` | 把对象当函数调用 | `d("arg")` |
| `TryCreateInstance` | new 动态对象 | `d(1, 2)` (少见) |
| `TryDeleteMember` | 删除成员 | `d.Name = null` (特殊场景) |

每个方法的签名都包含一个 `Binder` 对象，携带调用的元数据（成员名、参数数量、转换类型等），以及 `out` 参数用来返回结果。

**返回值规则**：返回 `true` 表示操作成功处理；返回 `false` 表示运行时抛出 `RuntimeBinderException`。

---

## 场景一：万能 JSON 解析器

对接第三方 API 时，返回的 JSON 结构可能经常变化。与其每次改 DTO 类，不如用 `DynamicObject` 做一层动态包装。

### 实现

```csharp
public class JsonDynamicObject : DynamicObject
{
    private readonly Dictionary<string, object> _properties = new();

    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        if (_properties.TryGetValue(binder.Name, out var value))
        {
            // 如果值是 JObject/JArray，继续包装成 DynamicObject
            result = WrapIfJson(value);
            return true;
        }
        result = null;
        return false;
    }

    public override bool TrySetMember(SetMemberBinder binder, object value)
    {
        _properties[binder.Name] = value;
        return true;
    }

    public override bool TryGetIndex(GetIndexBinder binder, object[] indexes, out object result)
    {
        var key = indexes[0].ToString();
        if (_properties.TryGetValue(key, out var value))
        {
            result = WrapIfJson(value);
            return true;
        }
        result = null;
        return false;
    }

    public override bool TryConvert(ConvertBinder binder, out object result)
    {
        if (binder.Type == typeof(Dictionary<string, object>))
        {
            result = _properties;
            return true;
        }
        if (binder.Type == typeof(string))
        {
            result = JsonSerializer.Serialize(_properties);
            return true;
        }
        result = null;
        return false;
    }

    private static object WrapIfJson(object value)
    {
        // 递归包装嵌套 JSON 对象
        if (value is JsonElement je && je.ValueKind == JsonValueKind.Object)
        {
            var dyn = new JsonDynamicObject();
            foreach (var prop in je.EnumerateObject())
            {
                dyn._properties[prop.Name] = prop.Value;
            }
            return dyn;
        }
        return ConvertJsonElement(value);
    }

    private static object ConvertJsonElement(object value)
    {
        return value switch
        {
            JsonElement je when je.ValueKind == JsonValueKind.String => je.GetString(),
            JsonElement je when je.ValueKind == JsonValueKind.Number => je.GetDouble(),
            JsonElement je when je.ValueKind == JsonValueKind.True => true,
            JsonElement je when je.ValueKind == JsonValueKind.False => false,
            JsonElement je when je.ValueKind == JsonValueKind.Null => null,
            _ => value
        };
    }
}
```

### 使用

```csharp
// 假设从 API 拿到 JSON 字符串
var json = """
{
    "user": {
        "name": "Alice",
        "age": 30,
        "address": {
            "city": "Shanghai",
            "zip": "200000"
        }
    },
    "orders": [
        { "id": 1, "product": "Laptop" },
        { "id": 2, "product": "Phone" }
    ]
}
""";

using var doc = JsonDocument.Parse(json);
var root = new JsonDynamicObject();
foreach (var prop in doc.RootElement.EnumerateObject())
{
    root._properties[prop.Name] = prop.Value;
}

dynamic data = root;

// 点号直接访问，不需要提前定义类
Console.WriteLine(data.user.name);        // Alice
Console.WriteLine(data.user.address.city); // Shanghai
Console.WriteLine(data.orders[0].product); // Laptop

// 也可以转回字典或 JSON 字符串
string backToJson = (string)data;
```

**优势**：API 返回体变了不需要改代码，调试时直接看 JSON 结构就行。

---

## 场景二：链式配置构建器

想写一个读起来像自然语言的配置 DSL？`DynamicObject` 可以让它非常优雅。

### 实现

```csharp
public class FluentBuilder : DynamicObject
{
    private readonly List<string> _steps = new();
    private readonly Dictionary<string, object> _values = new();

    public override bool TryInvokeMember(InvokeMemberBinder binder, object[] args, out object result)
    {
        var methodName = binder.Name;
        _steps.Add($"{methodName}({string.Join(", ", args.Select(a => a?.ToString() ?? "null"))})");

        // 如果有返回值类型的参数，记录下来
        if (args.Length == 1)
        {
            _values[methodName] = args[0];
        }

        // 返回自身，支持链式调用
        result = this;
        return true;
    }

    public override bool TryGetMember(GetMemberBinder binder, out object result)
    {
        if (_values.TryGetValue(binder.Name, out var value))
        {
            result = value;
            return true;
        }
        result = this;
        return true;
    }

    public override bool TryConvert(ConvertBinder binder, out object result)
    {
        if (binder.Type == typeof(string))
        {
            result = string.Join(" → ", _steps);
            return true;
        }
        result = null;
        return false;
    }

    public IReadOnlyDictionary<string, object> GetValues() => _values;
    public IReadOnlyList<string> GetSteps() => _steps;
}
```

### 使用

```csharp
dynamic build = new FluentBuilder();

build.CreateDatabase("MyApp")
     .AddTable("Users")
     .WithColumn("Id").OfType("int").AsPrimaryKey()
     .WithColumn("Name").OfType("nvarchar(100)").NotNull()
     .CreateIndex("IX_Users_Name").OnColumns("Name");

Console.WriteLine((string)build);
// 输出: CreateDatabase(MyApp) → AddTable(Users) → WithColumn(Id)
//        → OfType(int) → AsPrimaryKey() → WithColumn(Name)
//        → OfType(nvarchar(100)) → NotNull() → CreateIndex(IX_Users_Name)
//        → OnColumns(Name)

var values = build.GetValues();
Console.WriteLine(values["CreateDatabase"]); // MyApp
```

这种模式在 **数据库迁移脚本、测试数据构建器、规则引擎 DSL** 中非常实用。

---

## 场景三：动态 Mock 对象

写单元测试时，有时需要一个"什么方法都能调、记录调用历史"的 Mock 对象。用 `DynamicObject` 可以手写一个轻量版，不依赖 Moq 等框架。

```csharp
public class SimpleMock : DynamicObject
{
    public class CallRecord
    {
        public string MethodName { get; init; }
        public object[] Args { get; init; }
        public DateTime Timestamp { get; init; } = DateTime.Now;
    }

    private readonly List<CallRecord> _calls = new();
    private readonly Dictionary<string, object> _returns = new();

    public IReadOnlyList<CallRecord> Calls => _calls;

    // 预设某个方法的返回值
    public void Returns(string methodName, object returnValue)
    {
        _returns[methodName] = returnValue;
    }

    public override bool TryInvokeMember(InvokeMemberBinder binder, object[] args, out object result)
    {
        _calls.Add(new CallRecord
        {
            MethodName = binder.Name,
            Args = args
        });

        _returns.TryGetValue(binder.Name, out result);
        return true; // 即使没有预设返回值也成功（返回 null）
    }

    public void VerifyCalled(string methodName, int expectedTimes = 1)
    {
        var actual = _calls.Count(c => c.MethodName == methodName);
        if (actual != expectedTimes)
            throw new Exception(
                $"Expected {methodName} to be called {expectedTimes} time(s), but was {actual}");
    }
}
```

### 使用

```csharp
dynamic mockService = new SimpleMock();
mockService.Returns("GetUser", new { Name = "Alice", Role = "Admin" });

// 业务代码调用
var user = mockService.GetUser(42);
mockService.LogAction("login");
mockService.SendNotification("Welcome!");

// 断言
mockService.VerifyCalled("GetUser", 1);
mockService.VerifyCalled("SendNotification", 1);

Console.WriteLine($"Total calls: {mockService.Calls.Count}"); // 3
foreach (var call in mockService.Calls)
{
    Console.WriteLine($"  {call.MethodName}({string.Join(", ", call.Args)})");
}
```

这就是 Moq、NSubstitute 等框架的"极简内核"——当然生产级框架还多了表达式树解析、泛型支持、线程安全等特性，但原理相通。

---

## 场景四：动态代理（AOP 风格）

利用 `TryInvokeMember` 可以在方法调用前后插入横切逻辑。

```csharp
public class LoggingProxy : DynamicObject
{
    private readonly object _target;

    public LoggingProxy(object target) => _target = target;

    public override bool TryInvokeMember(InvokeMemberBinder binder, object[] args, out object result)
    {
        var method = _target.GetType().GetMethod(
            binder.Name,
            BindingFlags.Public | BindingFlags.Instance,
            null,
            args.Select(a => a?.GetType() ?? typeof(object)).ToArray(),
            null);

        if (method == null)
        {
            result = null;
            return false;
        }

        var sw = System.Diagnostics.Stopwatch.StartNew();
        result = method.Invoke(_target, args);
        sw.Stop();

        Console.WriteLine($"[Proxy] {binder.Name} executed in {sw.ElapsedMilliseconds}ms");
        return true;
    }
}
```

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public int Multiply(int a, int b)
    {
        Thread.Sleep(50);
        return a * b;
    }
}

var calc = new Calculator();
dynamic proxy = new LoggingProxy(calc);

proxy.Add(3, 4);       // [Proxy] Add executed in 0ms
proxy.Multiply(5, 6);  // [Proxy] Multiply executed in 50ms
```

这比手写装饰器模式简洁得多——一个代理类就能覆盖目标对象的所有方法。

---

## DynamicObject vs ExpandoObject vs 匿名类型

这三个是 C# 动态编程的"三驾马车"，适用场景不同：

| 维度 | `DynamicObject` | `ExpandoObject` | 匿名类型 (`new { ... }`) |
|------|----------------|-----------------|--------------------------|
| 行为控制 | ✅ 完全自定义 | ❌ 只能增删属性 | ❌ 只读，编译期固定 |
| 方法调用 | ✅ 可拦截 | ❌ 不支持 | ❌ 不支持 |
| 类型转换 | ✅ 可自定义 | ❌ 不支持 | ❌ 不支持 |
| 使用复杂度 | 高（要重写方法） | 低（开箱即用） | 极低 |
| 适用场景 | 框架开发、DSL、代理 | 临时数据结构、配置传递 | LINQ 投影、局部数据 |

### ExpandoObject 快速上手

```csharp
dynamic user = new ExpandoObject();
user.Name = "Alice";
user.Age = 30;
user.SayHi = new Action(() => Console.WriteLine($"Hi, I'm {user.Name}"));

user.SayHi(); // Hi, I'm Alice

// ExpandoObject 本质就是一个 Dictionary<string, object>
var dict = (IDictionary<string, object>)user;
dict.Remove("Age");
```

**经验法则**：

- 只需要"运行时添加属性" → 用 `ExpandoObject`
- 需要控制"点号/方法调用/类型转换"的行为 → 用 `DynamicObject`
- 编译期已知结构，只是不想声明类 → 匿名类型就够了

---

## 生产实践中的注意事项

### 1. 性能考量

`dynamic` 调用比直接调用慢 **10-100 倍**（首次绑定更慢），但 DLR 缓存机制让后续调用接近虚方法调用速度。

```
直接方法调用:  ~1 ns
dynamic 调用:  ~10-50 ns (缓存命中后)
dynamic 调用:  ~500-2000 ns (首次绑定)
反射调用:      ~100-500 ns
```

**不要在 hot path 里用 `dynamic`**，但一般业务层（API 层、配置层）完全没问题。

### 2. 没有编译期检查

```csharp
dynamic d = new JsonDynamicObject();
var x = d.TypooName; // 编译通过，运行时才发现拼写错误
```

应对策略：

- 在 `TryGetMember` 中加日志或默认值
- 对关键路径写单元测试覆盖
- 重要数据最终还是要映射回强类型 DTO

### 3. IDE 支持有限

`dynamic` 变量没有智能提示、没有重构支持、编译期看不到成员列表。调试时可以看运行时对象结构，但体验远不如强类型。

### 4. `null` 传播陷阱

```csharp
dynamic d = GetOrNull();
d.SomeMethod(); // 如果 d 是 null，抛 RuntimeBinderException 而非 NullReferenceException
```

`dynamic` 对 `null` 的处理和普通引用类型不同——它会走 DLR 绑定流程然后失败。加 null 检查是好习惯。

### 5. 泛型与 dynamic 的交互

```csharp
public void Process<T>(T input) { }

dynamic d = GetData();
Process(d);  // 运行时解析 T 的具体类型
Process<int>(d); // 编译期强制 T=int，运行期可能转换失败
```

理解这个差异可以避免"看起来类型对但运行时报错"的坑。

---

## 总结

`DynamicObject` 是 C# 生态中一个被低估的工具。它不是要取代静态类型——而是**在静态类型不够用的地方，提供一条优雅的逃生通道**。

**最适合用它解决的问题**：

- 结构不定的外部数据（JSON、API 响应）
- DSL / 链式构建器
- 测试 Mock / 动态代理
- 配置对象的灵活传递

**选型决策树**：

```
需要运行时动态行为？
├── 只需要动态属性？  → ExpandoObject
├── 需要控制方法调用/类型转换/索引？ → DynamicObject
└── 编译期结构已知？ → 强类型 class 或匿名类型
```

C# 的强大之处在于它同时拥抱了两种范式——静态类型让你写不出错，动态类型让你在需要时足够灵活。理解 `DynamicObject`，就是掌握了这座桥的钥匙。

---

> 本文所有代码示例均可在 .NET 6+ 环境直接运行。如有问题或想深入某个场景，欢迎在评论区交流。
