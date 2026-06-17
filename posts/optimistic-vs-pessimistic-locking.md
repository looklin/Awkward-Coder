---
title: 乐观锁与悲观锁全面解析——从数据库到前端的并发控制实战
slug: optimistic-vs-pessimistic-locking
description: >-
  深入对比乐观锁与悲观锁的原理、实现方式与适用场景，覆盖数据库层（MySQL/Oracle）、分布式缓存（Redis）、后端服务（C#/Java/Go）、前端 UI 层四种维度的实战方案，帮你构建完整的并发控制知识体系。
tags:
  - technical
added: "June 17 2026"
---

## 引言

在并发编程中，"锁"是最核心的控制手段。但很多人对锁的理解停留在"加锁就是 synchronized"的层面，遇到真正的并发问题时无从下手。

> 乐观锁和悲观锁不是具体的 API，而是**两种解决并发冲突的哲学**。

它们的区别可以用一句话概括：

- **悲观锁**：觉得别人一定会抢，先锁上再说。
- **乐观锁**：觉得别人大概率不会抢，冲突了再重试。

但现实远比这句话复杂——数据库锁、分布式锁、版本号、CAS、ETag、前端节流……这些技术的本质都是这两种哲学的体现。

本文将从四个维度全面展开：
1. 核心原理对比
2. 数据库层的实现（MySQL + Oracle）
3. 后端服务层的实现（C# / Java / Go 实战）
4. 分布式场景的实现（Redis 分布式锁 + 分布式版本号）
5. 前端层的实现（竞争提交、乐观更新、幂等控制）

---

## 一、核心原理：两种哲学

### 1.1 悲观锁（Pessimistic Locking）

**思想**：先锁定，再操作，操作完成后释放锁。操作期间别人无法修改。

```
线程A: ──[加锁]──[读取]──[修改]──[写回]──[解锁]──→
线程B: ──等待──等待──等待──等待──[加锁]──→
```

**优点**：数据一致性极强，实现简单，没有重试逻辑。
**缺点**：并发性能差，容易死锁，长时间持有锁会严重降低吞吐。

**典型场景**：
- 库存扣减（超卖敏感）
- 金融转账
- 计数器增减
- DBMS 的 `SELECT … FOR UPDATE`

### 1.2 乐观锁（Optimistic Locking）

**思想**：先操作，提交时检查冲突，有冲突则重试或回滚。

```
线程A: ──[读取(版本1)]──[修改]──[更新(版本1→2)]──成功──→
线程B: ──[读取(版本1)]──[修改]──[更新(版本1→2)]──失败，重试──→
```

**优点**：高并发性能好，无死锁风险。
**缺点**：冲突率高时大量重试，场景受限（不适合写密集）。

**典型场景**：
- 文章编辑保存（冲突概率低）
- 配置更新
- 分布式环境下的 CAS 操作
- 前端的"乐观更新"（Optimistic UI）

### 1.3 核心对比表

| 对比维度 | 悲观锁 | 乐观锁 |
|---------|--------|--------|
| 思想 | 先锁后做 | 先做再验证 |
| 并发性能 | 低 | 高 |
| 死锁风险 | 有 | 无 |
| 实现复杂度 | 低（依赖 DB/OS） | 中（需版本号/CAS/重试逻辑） |
| 适用场景 | 写冲突率高 | 写冲突率低 |
| 典型实现 | `SELECT … FOR UPDATE`, `lock`, `Mutex` | 版本号, CAS, ETag |

---

## 二、数据库层（MySQL）的锁实现

### 2.1 悲观锁：`SELECT … FOR UPDATE`

MySQL InnoDB 的 `SELECT … FOR UPDATE` 是最直接的悲观锁：

```sql
-- 事务A
START TRANSACTION;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;
-- 此时其他线程的 SELECT … FOR UPDATE 会阻塞
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

**关键要点**：
- `FOR UPDATE` 会加**行级排他锁（X锁）**
- 必须配合事务使用，事务结束（COMMIT/ROLLBACK）锁才释放
- **加锁是有代价的**——高并发下行锁也会导致锁等待、死锁
- 加锁要命中索引，否则行锁→表锁，性能灾难

**死锁经典场景**：

```sql
-- 线程A
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- 线程B
UPDATE accounts SET balance = balance - 100 WHERE id = 2;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
```

两个事务互相等待对方释放锁，形成死锁。InnoDB 会自动检测并回滚其中一个事务，但代价是回滚开销和应用层重试。

**解决思路**：
- 对所有竞争资源按固定顺序加锁（比如按 id 升序）
- 减少事务持有锁的时间（快进快出）
- 使用事务超时 `innodb_lock_wait_timeout`

### 2.2 乐观锁：版本号方案

不依赖数据库锁，而是用**版本号**或**时间戳**来实现：

```sql
-- 1. 读取数据，获取当前版本号
SELECT id, stock, version FROM products WHERE id = 1;
-- 结果：id=1, stock=100, version=3

-- 2. 更新时带上版本号条件
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 3;
```

**核心逻辑**：`UPDATE` 返回受影响行数：
- 如果为 1 → 更新成功，版本号已递增
- 如果为 0 → 版本不匹配，别人已经改了，需要重试

**状态机版本号**（更高级）：

在一些场景中，简单的递增版本号不够，还需要校验**状态流转的合法性**：

```sql
-- 订单状态：pending → paid → shipped → completed
UPDATE orders
SET status = 'paid', version = version + 1
WHERE id = 100
  AND version = 5
  AND status = 'pending'; -- 额外校验状态：只能从 pending 变为 paid
```

这样既防止了并发冲突，也防止了非法状态跳转。

### 2.3 乐观锁 vs 悲观锁在数据库层的抉择

| 维度 | `SELECT … FOR UPDATE` | 版本号 |
|------|----------------------|--------|
| 是否依赖事务 | 是，必须 | 否，单条 SQL 就够 |
| 并发压测 | 300 TPS 以下稳定 | 3000 TPS 以上稳定 |
| 锁粒度 | 行级 | 无锁（靠条件判断） |
| 热点数据 | 容易成为瓶颈 | 可承受超高并发 |
| 实现依赖 | DB 引擎 | 应用代码 + WHERE 条件 |

**实战建议**：
- **扣库存**：乐观锁 + 重试（高并发，冲突概率高但重试成本低）
- **转账**：悲观锁（数据一致性要求极高，冲突概率中，重试代价大）
- **报表统计**：乐观锁（读多写少，几乎无冲突）

---

## 三、后端服务层的锁实现

### 3.1 C#：从 Monitor 到 CAS

#### 悲观锁：`lock` 关键字

```csharp
private readonly object _lockObj = new();
private int _counter;

public void Increment()
{
    lock (_lockObj)
    {
        _counter++;
    }
}
```

**本质**：`lock` 是 `Monitor.Enter`/`Monitor.Exit` 的语法糖，底层依赖 CLR 的 `SyncBlock`。

#### 乐观锁：`Interlocked`（CAS 无锁操作）

```csharp
private int _counter;

public void Increment()
{
    Interlocked.Increment(ref _counter);
}
```

**底层原理**：`Interlocked.Increment` 生成 CPU 指令 `lock xadd`，由 CPU 保证原子性，比 `lock` 关键字快 10-50 倍。

#### 自定义版本号乐观锁

```csharp
public class OptimisticEntity
{
    public int Id { get; set; }
    public string Data { get; set; }
    public int Version { get; set; }
}

// 更新逻辑
public bool TryUpdate(OptimisticEntity entity, string newData)
{
    // 模拟 UPDATE … WHERE version = @oldVersion
    var sql = "UPDATE entities SET data = @data, version = version + 1 " +
              "WHERE id = @id AND version = @version";
    var rows = db.Execute(sql, new { data = newData, id = entity.Id, version = entity.Version });

    if (rows == 0) return false; // 冲突，外部重试
    entity.Version++; // 本地版本号同步
    entity.Data = newData;
    return true;
}
```

### 3.2 Java：synchronized 到 Atomic 家族

#### 悲观锁：`synchronized` + `ReentrantLock`

```java
// built-in
public synchronized void increment() { counter++; }

// 显式锁
private final ReentrantLock lock = new ReentrantLock();
public void increment() {
    lock.lock();
    try { counter++; }
    finally { lock.unlock(); }
}
```

#### 乐观锁：`AtomicInteger` / `LongAdder`

```java
private AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet(); // CAS 操作

// LongAdder 更高并发（分段 CAS）
private LongAdder adder = new LongAdder();
adder.increment(); // 内部使用 Cell[] 分段，减少 CAS 冲突
```

**选择建议**：
- **简单计数** → `AtomicInteger`（CAS）
- **超高并发计数** → `LongAdder`（分段 CAS）
- **复杂逻辑加锁** → `synchronized` 或 `ReentrantLock`（悲观）
- **读多写少** → `ReadWriteLock` 或 `StampedLock`

### 3.3 Go：channel 与 sync 包

Go 的哲学是——**"Don't communicate by sharing memory; share memory by communicating."**

#### 悲观锁：`sync.Mutex`

```go
var mu sync.Mutex
counter := 0

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}
```

#### 乐观锁：`sync/atomic`

```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}
```

#### Go 特色：用 channel 实现乐观协作

```go
type Account struct {
    balance int64
}

type Op struct {
    amount int64
    result chan bool
}

func (a *Account) Apply(ops chan Op) {
    for op := range ops {
        if a.balance + op.amount >= 0 {
            a.balance += op.amount
            op.result <- true
        } else {
            op.result <- false
        }
    }
}

// 使用
ops := make(chan Op, 100)
account := &Account{balance: 1000}
go account.Apply(ops) // 单 goroutine 串行处理，天然无锁
```

这种模式相当于用一个"串行化的 Actor"来避免共享数据上的并发竞争，是乐观锁思想的 Go 化表达。

---

## 四、分布式场景的锁实现

### 4.1 分布式悲观锁：Redis `SET NX`

```redis
SET lock:product:1 "thread-A" NX EX 10
-- 返回 OK → 加锁成功
-- 返回 nil → 加锁失败，别人持有着

-- 释放锁（**必须用 Lua 脚本保证原子性**）
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**实现一个 Redlock 风格的锁**需要解决：持有者标识、锁超时自动释放、看门狗续期。

C# 的 `RedLock.net` 或 Java 的 `Redisson` 已经封装好这些细节：

```csharp
using (var redLock = await redLockFactory.CreateLockAsync("resource-key", expiry))
{
    if (redLock.IsAcquired)
    {
        // 操作共享资源
    }
}
```

> Redlock 的争论（Martin Kleppmann vs Antirez）此处不展开，实际工程中单 Redis Master 加 NX EX 已能满足大多数场景。

### 4.2 分布式乐观锁：Redis Lua + 版本号

不依赖锁，而用 Lua 脚本保证原子比较+更新：

```lua
-- KEYS[1] = 资源 key, ARGV[1] = 期望版本号, ARGV[2] = 新数据
local curVersion = redis.call("GET", KEYS[1] .. ":version")
if curVersion == ARGV[1] then
    redis.call("SET", KEYS[1], ARGV[2])
    redis.call("INCR", KEYS[1] .. ":version")
    return 1  -- 成功
end
return 0  -- 冲突
```

### 4.3 基于数据库的分布式乐观锁

利用数据库的行锁 + 版本号实现跨服务的乐观锁：

```sql
-- 服务A 和服务B 共享同一个数据库
-- 它们在各自的数据库连接中执行同样的 UPDATE … WHERE version = ?
-- 只有一条能成功，另一条受影响行数为 0，触发重试
```

这种方案**不需要引入 Redis/ZK**，纯粹依赖数据库自身的原子更新语义，架构极为简洁。

**适用场景**：多个微服务共享数据库、写冲突概率低、不希望引入额外中间件的团队。

### 4.4 分布式锁抉择树

```
需要分布式锁？
├─ 有无共享数据库？
│   ├─ 有 → 数据库乐观锁（版本号）+ 重试（推荐）
│   └─ 无 → 继续
├─ 场景是否允许偶尔的锁失效？
│   ├─ 是 → Redis SET NX（性能好，不完美但够用）
│   └─ 否 → Redlock 或 ZooKeeper（强一致）
└─ 冲突概率高吗？
    ├─ 高 → 悲观锁（Redis / ZK 分布式锁）
    └─ 低 → 乐观锁（版本号 + Lua 脚本）
```

---

## 五、前端的并发控制

很多人以为锁是后端的事，前端是单线程的，不需要锁。——这是天大的误解。

前端同样存在并发问题：用户的连续双击、WebSocket 消息乱序到达、浏览器的异步任务竞态、表单重复提交……这些都是**前端的并发控制问题**。

### 5.1 表单重复提交（悲观实现）

```typescript
// Vue 示例：提交一次后禁用按钮
const submitting = ref(false)

async function submit() {
  if (submitting.value) return  // 悲观锁：先判断
  submitting.value = true
  try {
    await api.post('/order', formData)
  } finally {
    submitting.value = false
  }
}
```

```html
<button :disabled="submitting" @click="submit">
  {{ submitting ? '提交中...' : '提交订单' }}
</button>
```

**原理**：这是一种最朴素的**悲观锁**——先锁定 UI 状态，再执行操作，操作完成后解锁。

**缺点**：如果用户强制刷新页面、关闭浏览器，状态丢失（后端仍需保证幂等）。

### 5.2 请求去重 + 幂等（悲观 + 乐观混合）

```typescript
// 基于请求 URL + body 的请求缓存（悲观去重）
const pendingRequests = new Map<string, Promise<any>>()

async function deduplicatedFetch(url: string, body: object) {
  const key = `${url}:${JSON.stringify(body)}`

  if (pendingRequests.has(key)) {
    return pendingRequests.get(key)  // 同一请求复用，不做第二次
  }

  const promise = fetch(url, {
    method: 'POST',
    body: JSON.stringify(body),
    headers: { 'Idempotency-Key': crypto.randomUUID() }  // 后端幂等 key
  }).finally(() => pendingRequests.delete(key))

  pendingRequests.set(key, promise)
  return promise
}
```

**注**：前端的请求去重是**乐观的**（认为同时发两个一样的请求的概率低，但做了兜底），后端的幂等 key 是**悲观的**（必须保证同一个 key 不会产生多次扣减）。

### 5.3 基于 ETag 的乐观编辑

当多个标签页或用户在 WebSocket 场景下编辑同一文档时，可以采用 HTTP 的 ETag 机制：

```typescript
// 保存文档
async function saveDocument(doc: Document) {
  const response = await fetch(`/api/docs/${doc.id}`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      'If-Match': doc.etag  // 乐观锁：提交时校验版本
    },
    body: JSON.stringify({ content: doc.content })
  })

  if (response.status === 409) {
    // 冲突！别人已经修改了
    // 通知用户：文档已被他人修改，请重新加载
    throw new ConflictError('文档已被修改')
  }

  // 更新本地的 ETag
  doc.etag = response.headers.get('ETag')
}
```

**这是前端最常见的乐观锁模式**：提交时不锁定，提交后验证。本质和数据库的版本号完全一样。

### 5.4 WebSocket 消息乱序

WebSocket 是单连接，但服务器可能**并行发送消息**，到达时间不确定：

```
服务端发送： 消息 A(v1) → 消息 B(v2) → 消息 C(v3)
实际到达：   消息 B(v2) → 消息 A(v1) → 消息 C(v3)
                         ↑ 老版本覆盖新版本！数据不一致
```

**方案一：序列号悲观过滤（推荐）**

```typescript
class MessageSequencer {
  private lastSeq: number = 0

  apply(message: { seq: number; payload: any }) {
    if (message.seq <= this.lastSeq) {
      return // 老消息或重复消息，丢弃
    }
    this.lastSeq = message.seq
    this.render(message.payload) // 应用最新数据
  }
}
```

**方案二：版本号乐观覆盖**

```typescript
// 每个数据片段带版本号，UI 只保留最新版本号对应的数据
const state = reactive<Record<string, { data: any; version: number }>>({})

function updateField(field: string, data: any, version: number) {
  if (!state[field] || version > state[field].version) {
    state[field] = { data, version }
  }
}
```

### 5.5 React/Vue 中的"乐观更新"（Optimistic UI）

乐观更新是乐观锁思想在 UI 层的典型应用：**先更新 UI，再请求后端，失败后回滚**。

```typescript
// React + React Query 示例
function useLikePost(postId: string) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (postId: string) => api.likePost(postId),
    // 乐观更新：在请求完成前就更新 UI
    onMutate: async (postId) => {
      await queryClient.cancelQueries({ queryKey: ['post', postId] })
      const previousPost = queryClient.getQueryData(['post', postId])

      queryClient.setQueryData(['post', postId], (old: Post) => ({
        ...old,
        likes: old.likes + 1,
        liked: true,
      }))

      return { previousPost } // 用于回滚
    },
    onError: (err, postId, context) => {
      // 请求失败 → 回滚到之前的状态
      queryClient.setQueryData(['post', postId], context?.previousPost)
    },
  })
}
```

**原理**：认为后端大概率会成功（乐观），先更新 UI 提升用户体验；如果冲突了再回滚。这和数据库乐观锁的"先操作，冲突重试"有异曲同工之妙。

---

## 六、实战问题与避坑指南

### 6.1 乐观锁的 ABA 问题

ABA 问题是 CAS 操作的一个经典陷阱：

```
线程1: 读取 A，准备 CAS(A→C)
线程2: 将 A 改为 B，又将 B 改回 A
线程1: CAS 比较发现还是 A，CAS 成功
       ↑ 但实际上数据已经被改变了两次
```

**解决方案**：
- 使用版本号（每次修改版本号 +1，ABA 无法重复版本号）
- 使用时间戳（每次修改记录时间戳）
- C#/Java 的带版本号的原子引用：`AtomicStampedReference`、`AtomicMarkableReference`

### 6.2 高冲突率下乐观锁的性能退化

乐观锁在冲突率 > 30% 时，大量 CPU 时间花在重试上，甚至不如悲观锁。

**应对策略**：
1. **退化为悲观锁**——重试 N 次后，加悲观锁兜底
2. **削峰填谷**——用队列将并发请求串行化（Go 的 channel、消息队列）
3. **分段锁**——将热点资源拆分为多个槽位（类似 ConcurrentHashMap 的分段思想）

### 6.3 悲观锁的常见陷阱

| 陷阱 | 现象 | 解决 |
|------|------|------|
| forgot to unlock | 死锁 | 用 `using` / `try-finally` / `defer` |
| 锁粒度太大 | 吞吐极低 | 缩小临界区，考虑读写锁 |
| 锁顺序不一致 | 死锁 | 固定加锁顺序、使用超时 |
| 分布式锁超时释放 | 并发进入临界区 | 看门狗续期 + 持有者标识 |
| 锁了非 final 对象 | 锁失效 | 锁定只读的 `object` 或 `string` 常量 |

### 6.4 终极建议：组合使用才是王道

> 没有银弹，好的架构往往是两种锁的有机组合。

**完美工程实践**：

```
用户请求
  │
  ▼
前端乐观更新（UI 先响应，失败回滚）
  │
  ▼
前端请求去重（防止相同请求发两次）
  │
  ▼ [网关层]
接口幂等校验（幂等 key，防重放）
  │
  ▼ [后端服务]
轻量级 CAS + 版本号（乐观锁，90% 场景一次成功）
  │
  ▼ [后端服务]
重试 3 次失败 → 分布式悲观锁兜底（Redis NX EX）
  │
  ▼ [数据库]
最后的防线：WHERE version = @version（数据库乐观锁）
```

每一层都有自己擅长的防御范围，层层叠加，构成完整的并发控制体系。

---

## 七、总结

| 层级 | 推荐策略 | 说明 |
|------|---------|------|
| **前端 UI** | 乐观更新 | 先更新界面，失败回滚 |
| **前端请求** | 请求去重 + 幂等 key | 防止重复提交 |
| **前端 WebSocket** | 序列号校验（悲观） | 过滤乱序/重复消息 |
| **后端方法内** | `Interlocked` / `Atomic` / `sync/atomic` | 无锁 CAS，最快 |
| **后端代码段** | `lock` / `Mutex` / `synchronized` | 复杂逻辑悲观锁 |
| **分布式锁** | Redis NX EX + 看门狗 | 跨进程协调 |
| **数据库** | 版本号乐观锁 | 最后防线 |

**乐观锁和悲观锁不是二选一**——它们是工具箱里的两种工具，为不同场景准备。

- 冲突概率低、性能敏感 → 乐观锁
- 冲突概率高、数据敏感 → 悲观锁
- 跨进程协调 → 分布式锁
- UI 体验优先 → 乐观更新

理解了这两种哲学，你就能在任何层级——数据库、后端、分布式、前端——都做出正确的并发控制决策。

---

*这篇文档讨论了乐观锁和悲观锁在前后端及数据库层全面应用方案。如果你有具体的并发场景需要评估，欢迎讨论。*
