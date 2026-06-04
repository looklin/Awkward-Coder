---
title: C# LINQ 全面指南——从入门到高性能实践
slug: csharp-linq-practice
description: >-
  全面介绍 C# LINQ 的核心概念、操作符分类、实战场景、常见陷阱与性能优化技巧。涵盖 Enumerable 与 Queryable 的区别、延迟执行机制、内存分配优化以及 EF Core 中的 LINQ 注意事项。
tags:
  - technical
added: "June 04 2026"
---

# C# LINQ 全面指南——从入门到高性能实践

> LINQ（Language Integrated Query）是 C# 最具标志性的特性之一。它将查询能力直接融入语言，让数据操作变得声明式、可读且组合灵活。但用不好也会带来性能陷阱。本文将从原理到实践，带你真正掌握 LINQ。

---

## 一、什么是 LINQ？

LINQ 是 C# 3.0（.NET Framework 3.5）引入的查询语法，提供了一套统一的数据操作 API。它的核心思想是：**用相同的方式查询不同来源的数据**。

```csharp
// 查询内存集合
var adults = people.Where(p => p.Age >= 18)
                   .OrderBy(p => p.Name)
                   .Select(p => p.Name);

// 查询数据库（EF Core）
var dbAdults = context.People.Where(p => p.Age >= 18)
                             .OrderBy(p => p.Name)
                             .Select(p => p.Name)
                             .ToList();

// 查询 XML
var elements = doc.Descendants("Person")
                  .Where(e => (int)e.Element("Age") >= 18);
```

### 两种 LINQ 提供程序

| 类型 | 命名空间 | 操作的数据源 | 执行位置 |
|------|----------|-------------|---------|
| **LINQ to Objects** | `System.Linq.Enumerable` | 内存集合 (`IEnumerable<T>`) | 进程内 |
| **LINQ to Entities / SQL** | `System.Linq.Queryable` | 外部数据源（数据库等） | 远程（翻译为 SQL） |

---

## 二、两种查询语法

### 1. 方法语法（Method Syntax）

```csharp
var result = numbers.Where(n => n > 0)
                    .Select(n => n * 2)
                    .OrderByDescending(n => n)
                    .Take(5)
                    .ToList();
```

### 2. 查询表达式语法（Query Syntax）

```csharp
var result = (from n in numbers
              where n > 0
              orderby n descending
              select n * 2)
             .Take(5)
             .ToList();
```

**建议：** 优先使用方法语法。它更灵活（不是所有操作符都有查询语法对应），链式调用更直观，也更符合现代 C# 的风格。查询表达式适合简单的多源 join 场景。

---

## 三、核心操作符分类

### 1. 过滤

```csharp
// Where — 条件过滤
var adults = people.Where(p => p.Age >= 18);

// OfType — 按类型过滤
var strings = objects.OfType<string>();

// Distinct — 去重
var unique = items.Distinct();
var uniqueBy = people.DistinctBy(p => p.Email); // .NET 6+
```

### 2. 投影

```csharp
// Select — 一对一映射
var names = people.Select(p => p.Name);

// SelectMany — 一对多展平（类似 SQL 的 CROSS JOIN）
var allSkills = people.SelectMany(p => p.Skills);

// 带索引的 Select
var indexed = items.Select((item, index) => new { index, item });
```

**SelectMany 实战示例：**

```csharp
// 扁平化嵌套集合
var orders = new List<Order>
{
    new() { Id = 1, Items = new[] { "A", "B" } },
    new() { Id = 2, Items = new[] { "C", "D", "E" } }
};

// 所有订单项展平
var allItems = orders.SelectMany(o => o.Items);
// 结果: ["A", "B", "C", "D", "E"]

// 带父子关联的展平
var itemWithOrder = orders.SelectMany(
    o => o.Items,
    (order, item) => new { OrderId = order.Id, Item = item });
```

### 3. 排序

```csharp
// 单字段排序
var sorted = people.OrderBy(p => p.Age);
var sortedDesc = people.OrderByDescending(p => p.Salary);

// 多字段排序
var multiSorted = people.OrderBy(p => p.Department)
                        .ThenByDescending(p => p.Salary)
                        .ThenBy(p => p.Name);
```

### 4. 分组

```csharp
// 基本分组
var byDept = people.GroupBy(p => p.Department);

foreach (var group in byDept)
{
    Console.WriteLine($"部门: {group.Key}, 人数: {group.Count()}");
    foreach (var person in group)
    {
        Console.WriteLine($"  - {person.Name}");
    }
}

// 分组 + 聚合
var deptStats = people.GroupBy(p => p.Department)
                      .Select(g => new
                      {
                          Department = g.Key,
                          Count = g.Count(),
                          AvgSalary = g.Average(p => p.Salary),
                          MaxSalary = g.Max(p => p.Salary)
                      });
```

### 5. 连接（Join）

```csharp
var result = from person in people
             join dept in departments
             on person.DeptId equals dept.Id
             select new { person.Name, dept.DeptName };

// 方法语法
var result2 = people.Join(
    departments,
    person => person.DeptId,
    dept => dept.Id,
    (person, dept) => new { person.Name, dept.DeptName });

// 左外连接
var leftJoin = from person in people
               join dept in departments
               on person.DeptId equals dept.Id into gj
               from sub in gj.DefaultIfEmpty()
               select new
               {
                   person.Name,
                   DeptName = sub?.DeptName ?? "未分配"
               };
```

### 6. 聚合

```csharp
var count = numbers.Count();
var sum = numbers.Sum();
var average = numbers.Average();
var min = numbers.Min();
var max = numbers.Max();
var aggregate = numbers.Aggregate(1, (acc, n) => acc * n); // 累乘
```

### 7. 集合操作

```csharp
var union = set1.Union(set2);          // 并集（去重）
var intersect = set1.Intersect(set2);  // 交集
var except = set1.Except(set2);        // 差集
var concat = set1.Concat(set2);        // 连接（不去重）
var zip = set1.Zip(set2, (a, b) => a + b); // 拉链合并
```

### 8. 元素操作

```csharp
var first = numbers.First(n => n > 10);     // 找不到抛异常
var firstOrDefault = numbers.FirstOrDefault(n => n > 10); // 找不到返回 default
var single = numbers.Single(n => n == 42);  // 必须恰好一个，否则抛异常
var elementAt = numbers.ElementAt(5);       // 按索引取元素
var last = numbers.Last();                  // 最后一个
```

### 9. 量词判断

```csharp
bool allAdults = people.All(p => p.Age >= 18);
bool hasAdult = people.Any(p => p.Age >= 18);
bool contains42 = numbers.Contains(42);
```

---

## 四、延迟执行：LINQ 的核心机制

### 什么是延迟执行？

LINQ 查询在定义时**不会立即执行**，而是在**枚举结果时**才执行。

```csharp
var query = numbers.Where(n => n > 5); // 此时什么都没做

// 真正执行发生在以下任一时刻：
foreach (var n in query) { }  // 枚举
var list = query.ToList();    // 物化
var count = query.Count();    // 聚合
var first = query.First();    // 元素操作
```

**延迟执行的好处：**

- ✅ 查询可以组合而不会重复遍历
- ✅ 避免不必要的数据加载
- ✅ 查询反映数据的最新状态

### 陷阱：延迟执行导致的重复执行

```csharp
var query = db.Orders.Where(o => o.Status == "Pending");

int count = query.Count();    // 执行一次数据库查询
var list = query.ToList();    // 又执行一次数据库查询！
```

**正确做法：**

```csharp
var query = db.Orders.Where(o => o.Status == "Pending");

// 只查询一次
var list = query.ToList();  // 物化到内存
int count = list.Count;      // 内存操作，不查数据库
```

### 陷阱：延迟执行 + 数据源变更

```csharp
var numbers = new List<int> { 1, 2, 3 };
var query = numbers.Where(n => n > 1);

Console.WriteLine(string.Join(", ", query)); // 2, 3

numbers.Add(5); // 修改了数据源

Console.WriteLine(string.Join(", ", query)); // 2, 3, 5 ← 结果变了！
```

如果查询时数据源已经变更，结果也会跟着变。需要固定结果时，使用 `.ToList()` 或 `.ToArray()` 物化。

### 立即执行的方法

以下方法会触发立即执行：

| 类别 | 方法 |
|------|------|
| 物化 | `ToList()`, `ToArray()`, `ToDictionary()`, `ToHashSet()`, `ToLookup()` |
| 元素 | `First()`, `FirstOrDefault()`, `Last()`, `Single()`, `ElementAt()` |
| 聚合 | `Count()`, `Sum()`, `Average()`, `Min()`, `Max()`, `Aggregate()` |
| 量词 | `Any()`, `All()`, `Contains()` |
| 比较 | `SequenceEqual()` |

---

## 五、实战场景

### 场景一：复杂数据转换

```csharp
// 从原始数据构建 DTO
var dto = orders
    .Where(o => o.Date >= DateTime.UtcNow.AddDays(-30))
    .GroupBy(o => o.CustomerId)
    .Select(g => new CustomerOrderSummary
    {
        CustomerId = g.Key,
        CustomerName = g.First().CustomerName,
        TotalOrders = g.Count(),
        TotalAmount = g.Sum(o => o.Amount),
        AvgOrderAmount = g.Average(o => o.Amount),
        LatestOrderDate = g.Max(o => o.Date),
        Products = g.SelectMany(o => o.Items)
                    .GroupBy(i => i.ProductName)
                    .Select(pg => new ProductFrequency
                    {
                        ProductName = pg.Key,
                        OrderCount = pg.Count()
                    })
                    .OrderByDescending(pf => pf.OrderCount)
                    .Take(5)
                    .ToList()
    })
    .OrderByDescending(s => s.TotalAmount)
    .ToList();
```

### 场景二：数据清洗

```csharp
var cleanData = rawData
    .Where(r => r != null)                    // 过滤 null
    .Select(r => r.Trim())                    // 去空格
    .Where(r => !string.IsNullOrEmpty(r))     // 过滤空串
    .Distinct()                               // 去重
    .OrderBy(r => r)                          // 排序
    .ToList();
```

### 场景三：动态条件构建

```csharp
IQueryable<Product> query = context.Products;

if (!string.IsNullOrEmpty(search))
{
    query = query.Where(p =>
        p.Name.Contains(search) ||
        p.Description.Contains(search));
}

if (minPrice.HasValue)
{
    query = query.Where(p => p.Price >= minPrice.Value);
}

if (maxPrice.HasValue)
{
    query = query.Where(p => p.Price <= maxPrice.Value);
}

if (categoryId.HasValue)
{
    query = query.Where(p => p.CategoryId == categoryId.Value);
}

var results = query.OrderBy(p => p.Name)
                   .Skip(page * pageSize)
                   .Take(pageSize)
                   .ToList();
```

### 场景四：分组统计报表

```csharp
var report = sales
    .GroupBy(s => new { Year = s.Date.Year, Month = s.Date.Month })
    .Select(g => new MonthlyReport
    {
        Year = g.Key.Year,
        Month = g.Key.Month,
        TotalRevenue = g.Sum(s => s.Amount),
        OrderCount = g.Count(),
        AvgOrderValue = g.Average(s => s.Amount),
        TopProduct = g.GroupBy(s => s.ProductName)
                      .OrderByDescending(pg => pg.Sum(s => s.Amount))
                      .First().Key
    })
    .OrderBy(r => r.Year)
    .ThenBy(r => r.Month)
    .ToList();
```

---

## 六、性能优化

### 1. 避免不必要的 ToList()

```csharp
// ❌ 不好：中间物化产生多余的 List 分配
var result = items.Where(x => x.IsActive)
                  .ToList()     // ← 多余
                  .OrderBy(x => x.Name)
                  .ToList();

// ✅ 好：延迟执行到最后一步才物化
var result = items.Where(x => x.IsActive)
                  .OrderBy(x => x.Name)
                  .ToList();    // 只在最终需要时物化
```

### 2. 用 Any() 替代 Count() > 0

```csharp
// ❌ 不好：需要遍历整个集合
bool hasItems = items.Count() > 0;

// ✅ 好：找到第一个就停止
bool hasItems = items.Any();
```

对于 LINQ to Objects，`Count()` 会先检查 `ICollection<T>` 接口走快速路径，但对于 `IQueryable`（数据库查询），`Count()` 会执行全表 `COUNT(*)`，而 `Any()` 只需要 `EXISTS` 子查询，效率天差地别。

### 3. 用 ToLookup 替代重复 GroupBy

```csharp
// ❌ 不好：每次访问都重新分组
var group1 = people.GroupBy(p => p.Department).First(g => g.Key == "HR");
var group2 = people.GroupBy(p => p.Department).First(g => g.Key == "IT");

// ✅ 好：分组一次，多次查找
var lookup = people.ToLookup(p => p.Department);
var group1 = lookup["HR"];
var group2 = lookup["IT"];
```

### 4. 大数据量用 ForEach 替代 foreach + LINQ

```csharp
// 纯遍历操作不需要 LINQ
// ❌
items.Where(x => x.IsActive).ToList().ForEach(x => Process(x));

// ✅
foreach (var item in items)
{
    if (item.IsActive)
        Process(item);
}
```

LINQ 适合**查询和转换**，不适合**副作用操作**。如果只是为了遍历执行某个方法，直接用 `foreach`。

### 5. 用 Enumerable.Range 替代手写循环

```csharp
// ❌
var numbers = new List<int>();
for (int i = 0; i < 100; i++)
    numbers.Add(i);

// ✅
var numbers = Enumerable.Range(0, 100).ToList();
```

### 6. Select 中的对象创建优化

```csharp
// ❌ 每次 Select 都创建新的匿名对象
var query = items.Select(x => new { Name = x.Name, Age = x.Age });

// 如果只需要部分字段，考虑用 ValueTuple（.NET 7+ 进一步优化）
var query = items.Select(x => (x.Name, x.Age));
```

### 7. 注意闭包捕获

```csharp
// ❌ 每次迭代创建新闭包
var funcs = new List<Func<int>>();
for (int i = 0; i < 10; i++)
{
    funcs.Add(() => i); // 经典陷阱：所有 lambda 捕获同一个 i
}
// 所有 func 都返回 10

// ✅ 使用 foreach 或复制变量
var funcs2 = Enumerable.Range(0, 10)
                       .Select(i => new Func<int>(() => i))
                       .ToList();
```

### 8. 数据库查询：Select 投影减少数据传输

```csharp
// ❌ 加载整个实体（可能包含大量不需要的字段）
var users = context.Users.Where(u => u.IsActive).ToList();

// ✅ 只查询需要的字段
var users = context.Users.Where(u => u.IsActive)
                         .Select(u => new { u.Id, u.Name, u.Email })
                         .ToList();
```

### 9. AsEnumerable vs AsQueryable

```csharp
// AsEnumerable: 将后续操作切换到内存中执行
// 适合数据库不支持的函数（如自定义 C# 方法）
var result = context.Orders
    .Where(o => o.Status == "Completed")  // 翻译为 SQL
    .AsEnumerable()                        // 切换到内存
    .Select(o => FormatOrder(o))           // C# 方法，无法翻译为 SQL
    .ToList();

// AsQueryable: 保持查询可翻译
// 适合继续使用 LINQ Provider 的场景
```

### 10. 使用 .NET 6+ 的高性能 API

```csharp
// DistinctBy — 按指定字段去重（比 GroupBy + Select 更高效）
var unique = people.DistinctBy(p => p.Email);

// MaxBy / MinBy — 按比较键取最大/最小（无需先排序）
var oldest = people.MaxBy(p => p.Age);
var cheapest = products.MinBy(p => p.Price);

// Chunk — 分块处理大数据集
foreach (var chunk in largeList.Chunk(100))
{
    ProcessBatch(chunk);
}

// Zip 增强 — 支持三路合并
var zipped = names.Zip(ages, scores, (n, a, s) => (n, a, s));

// Index — 带索引的枚举
foreach (var (item, index) in items.Index())
{
    Console.WriteLine($"{index}: {item}");
}
```

---

## 七、EF Core 中 LINQ 的注意事项

### 1. 客户端 vs 服务端评估

EF Core 尝试将 LINQ 表达式翻译为 SQL。无法翻译的部分：

- **EF Core 3.0+**：直接抛异常（避免隐式客户端评估导致的性能问题）
- 需要用 `AsEnumerable()` 显式切换到客户端

```csharp
// ❌ EF Core 无法翻译 Math.Round 到 SQL（某些版本/数据库）
var result = context.Products
    .Select(p => Math.Round(p.Price, 2))
    .ToList();

// ✅ 先查数据，再在客户端处理
var result = context.Products
    .AsEnumerable()
    .Select(p => Math.Round(p.Price, 2))
    .ToList();

// ✅ 或者在 SQL 端处理（EF Core 7+ 已支持部分 Math 函数）
var result = context.Products
    .Select(p => EF.Functions.Round(p.Price, 2))
    .ToList();
```

### 2. 避免 N+1 查询

```csharp
// ❌ N+1 问题：先查部门，再循环查每个部门的员工
var departments = context.Departments.ToList();
foreach (var dept in departments)
{
    var employees = context.Employees
        .Where(e => e.DeptId == dept.Id)
        .ToList(); // 每个部门执行一次查询！
}

// ✅ 使用 Include 预加载
var departments = context.Departments
    .Include(d => d.Employees)
    .ToList(); // 一次 JOIN 查询搞定

// ✅ 或使用 Select + JOIN
var result = context.Departments
    .Select(d => new
    {
        d.Name,
        Employees = d.Employees.ToList()
    })
    .ToList();
```

### 3. 避免在 Where 中调用无法翻译的方法

```csharp
// ❌ IsSpecialCustomer 无法翻译为 SQL
var query = context.Customers
    .Where(c => IsSpecialCustomer(c))
    .ToList();

// ✅ 将逻辑写在 LINQ 表达式内
var query = context.Customers
    .Where(c => c.TotalSpent > 10000 && c.YearsAsCustomer >= 3)
    .ToList();
```

### 4. 字符串比较的注意事项

```csharp
// EF Core 对 Contains 的翻译取决于配置
// 默认是区分大小写的（取决于数据库排序规则）

// 不区分大小写搜索
var result = context.Products
    .Where(p => EF.Functions.ILike(p.Name, "%laptop%"))  // PostgreSQL
    // .Where(p => p.Name.Contains("laptop"))            // SQL Server 默认不区分
    .ToList();
```

---

## 八、常见陷阱汇总

| 陷阱 | 错误做法 | 正确做法 |
|------|----------|----------|
| 重复执行查询 | 多次枚举同一个 `IQueryable` | 先 `ToList()` 物化 |
| Count() > 0 | `items.Count() > 0` | `items.Any()` |
| 闭包捕获循环变量 | `for` 循环中 lambda 捕获变量 | 用 LINQ 或复制变量 |
| 意外修改源数据 | 查询后修改了原始集合 | 用 `Select` 投影创建新对象 |
| 大数据集全量加载 | 未分页直接 `ToList()` | 使用 `Skip`/`Take` 或流式处理 |
| 在 LINQ 中做副作用 | `items.ForEach(x => db.Save(x))` | 用 `foreach` 循环 |
| 混用 Enumerable 和 Queryable | `IEnumerable` 上执行导致内存过滤 | 保持 `IQueryable` 到数据库端 |
| null 聚合结果 | `Enumerable.Empty<int>().Sum()` 返回 0，但 `Average()` 抛异常 | 先检查 `Any()` 或用 `DefaultIfEmpty` |

---

## 九、调试技巧

### 1. 查看 LINQ to Objects 的执行过程

```csharp
var query = numbers
    .Where(n => { Console.WriteLine($"Where: {n}"); return n > 5; })
    .Select(n => { Console.WriteLine($"Select: {n}"); return n * 2; });

foreach (var item in query)
{
    Console.WriteLine($"Result: {item}");
}
// 输出展示延迟执行的逐项处理过程
```

### 2. 查看 EF Core 生成的 SQL

```csharp
// 在 DbContext 配置中启用日志
optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information);

// 或直接获取 SQL（EF Core 5+）
var sql = query.ToQueryString();
Console.WriteLine(sql);
```

### 3. 使用 LINQPad

[LINQPad](https://www.linqpad.net/) 是调试和探索 LINQ 的最佳工具，支持：
- 即时执行 LINQ 表达式
- 查看生成的 SQL
- 结果可视化
- 性能分析

---

## 十、总结

LINQ 是 C# 最强大的特性之一，但要用好它需要理解三个层次：

1. **语法层** — 熟悉各操作符的含义和用法
2. **机制层** — 理解延迟执行、IEnumerable vs IQueryable、闭包捕获
3. **性能层** — 知道何时该用、何时不该用、如何避免常见陷阱

**核心原则：**

- 查询用 LINQ，副作用用 foreach
- 保持 IQueryable 到最后一刻才物化
- Any() 优于 Count() > 0
- EF Core 中关注 SQL 翻译和 N+1 问题
- 善用 .NET 6+ 新增的 DistinctBy、MaxBy 等高效 API
- 大数据量考虑流式处理，避免全量加载

掌握了这些，LINQ 就会从"好用的语法糖"变成你手中真正的高效工具。

---

## 参考链接

- [Microsoft Docs: LINQ (Language-Integrated Query)](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/)
- [Microsoft Docs: 101 LINQ Samples](https://learn.microsoft.com/en-us/samples/browse/?expanded=dotnet&products=dotnet&terms=linq)
- [LINQPad — LINQ Debug Tool](https://www.linqpad.net/)
- [EF Core Query Documentation](https://learn.microsoft.com/en-us/ef/core/querying/)
- [What's New in LINQ (.NET 6+)](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-6)
