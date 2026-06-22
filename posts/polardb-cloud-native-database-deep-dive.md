---
title: PolarDB 深度解析——云原生数据库的架构革命与工程实践
slug: polardb-cloud-native-database-deep-dive
description: >-
  全面解析阿里云 PolarDB 的云计算原生架构——计算与存储分离、PolarFS 分布式文件系统、
  日志即数据库、HTAP 融合引擎。深入技术原理并与 AWS Aurora、MySQL 对比，附带性能基准和实际落地经验。
tags:
  - technical
added: "June 22 2026"
---

# PolarDB 深度解析——云原生数据库的架构革命与工程实践

> 传统关系型数据库在云上挣扎了十年——扩容停机、存储硬绑定、只读节点负载存储。
> PolarDB 用一套"计算与存储分离 + 分布式共享存储"的架构，从根本上重新定义了云数据库的形态。
> 本文从架构原理到实战落地，全方位解剖这款阿里云的王牌产品。

---

## 一、背景：为什么需要 PolarDB？

### 传统数据库上云的三大痛点

2008 年云计算兴起后，各大云厂商的第一件事就是把 MySQL、PostgreSQL、Oracle 搬到云上。但"搬上去"不等于"云原生"——传统数据库的架构假设与云平台的特性存在根本性冲突：

**1. 计算与存储耦合**

传统 MySQL/PostgreSQL 部署在一台物理机上，数据和计算共享本地磁盘。这意味着：
- 扩容 CPU 必须同时扩容存储（反之亦然），资源利用率极低
- 存储成为瓶颈——单机存储有上限，分布式方案（如 MySQL sharding）复杂度极高
- 物理机故障意味着数据丢失或 RTO 极长

**2. 只读副本的成本困局**

MySQL 主从架构中，每个从库都是一份**完整的数据副本**。如果主库有 500GB 数据，部署 5 个只读节点就需要 2.5TB 存储。数据量越大，只读副本的存储成本越高，但大部分副本的 CPU 利用率可能不到 10%。

**3. 复制延迟**

MySQL 异步复制在跨可用区场景下，延迟通常在百毫秒到秒级。Binlog 的传输和回放成为只读节点的性能瓶颈。

AWS Aurora 在 2015 年首次打破了这种局面——它把存储层独立出来，让所有计算节点共享同一份存储。PolarDB 走的是相同的方向，但在工程实现上有自己独特的技术选择。

### PolarDB 的目标

阿里云在 2017 年启动了 PolarDB 项目，目标是构建一款**真正为云设计的数据库**：

> 一个由计算层和存储层构成，计算层无状态、存储层可无限扩展、两者通过高速 RDMA 网络互联的云原生数据库。

---

## 二、架构全景

```
                  ┌─────────────────────────────────────┐
                  │             计算层（无状态）            │
                  │                                      │
                  │   ┌─────────┐  ┌─────────┐  ┌──────┐ │
                  │   │ RW Node │  │ RO Node │  │ RO   │ │
                  │   │ (Primary)│  │  #1     │  │ #2..N│ │
                  │   └────┬────┘  └────┬────┘  └──┬───┘ │
                  │        │            │          │      │
                  │        └─────┬──────┘          │      │
                  │              │  RDMA / TCP     │      │
                  └──────────────┼─────────────────┘      │
                                 │                        │
                    ┌────────────┼────────────┐
                    │            ▼            │
                    │  ┌──────────────────┐   │
                    │  │    PolarFS       │   │
                    │  │ (分布式文件系统)  │   │
                    │  └────────┬─────────┘   │
                    │           │             │
                    │           ▼             │
                    │  ┌──────────────────┐   │
                    │  │   存储层 Chunk   │   │
                    │  │  ┌────┐┌────┐   │   │
                    │  │  │C1  ││C2  │...│   │
                    │  │  └────┘└────┘   │   │
                    │  └──────────────────┘   │
                    │     三副本存储 (Chunk)    │
                    └─────────────────────────┘
```

PolarDB 的架构可以清晰地分为三层：

### 计算层：无状态实例

计算层是 PolarDB 的"大脑"——运行完整的 MySQL/PostgreSQL 引擎，处理 SQL 解析、优化和执行。但**不持久化任何数据**。

- **主节点（RW Node）**：支持读写，是唯一写入数据的节点
- **只读节点（RO Node）**：最多 15 个，共享同一份存储数据

> 无状态设计的核心好处：计算节点可以在几秒钟内创建或销毁，CPU 和内存的扩缩容不再与存储挂钩。

### 存储层：共享分布式存储

存储层才是数据真正安家的地方。PolarDB 没有采用传统的本地 SSD 或云盘（如 EBS），而是构建了自己的**分布式文件系统 PolarFS**。

存储层的核心设计：

1. **三副本冗余**：每个数据块（Chunk）有 3 个副本，分布在不同的物理节点甚至可用区
2. **自动故障恢复**：单副本损坏后，自动从其他副本重建
3. **弹性扩展**：容量自动扩展到 100TB，不需要预分配
4. **日志下沉**：存储层直接处理 Redo Log，计算层不再需要写完整的 Page

### 高速网络层：RDMA

计算层与存储层之间通过 **RDMA（Remote Direct Memory Access）** 网络连接，延迟低至 10μs 级别。

```text
传统方案：计算节点 → OS → TCP/IP → NIC → Switch → 存储
PolarDB：计算节点 → RDMA → 存储（绕过 OS 内核）
```

RDMA 让 PolarDB 的存储访问延迟接近本地 SSD 的水平，同时保持了共享存储的弹性优势。

---

## 三、核心技术机制

### 3.1 PolarFS——为数据库定制的分布式文件系统

PolarFS 是 PolarDB 最核心的技术壁垒。它不是一个通用的分布式文件系统，而是**专门为数据库 IO 模式优化的存储系统**。

#### 设计约束

PolarFS 的设计充分考虑了一个事实：数据库的工作负载不是通用的文件系统负载，它有鲜明的特征：

| 特征 | 含义 | PolarFS 优化 |
|------|------|-------------|
| **大块顺序写入** | Redo Log 是顺序追加写入 | Append-only 策略，优化写路径 |
| **小块随机读取** | Data Page 的读取是 16KB 的随机 IO | 缓存预热 + 预读 |
| **读写不对称** | 写主库、读只读节点，IO 分布不同 | 存储层感知 IO 来源做调度 |
| **并发高** | 大量并发事务同时写日志 | Log Stream 并行化 |

#### Chunk 与 Chunk Server

PolarFS 将数据切分为固定大小的 **Chunk**（默认 10GB），每个 Chunk 由底层的 Chunk Server 管理。Chunk Server 运行在标准化硬件上，提供块级存储服务。

Chunk 的三副本策略：

```
Chunk A  ──┬── Chunk Server 1 (Primary)
           ├── Chunk Server 2 (Secondary)
           └── Chunk Server 3 (Secondary)
```

主 Chunk Server 负责所有的读写请求，备 Chunk Server 同步数据。当主节点故障时，Paxos 协议自动选主。

#### Append-only Log

PolarFS 的物理设计很有意思——它本身就是一个 **Append-only 日志结构文件系统**。每次写操作不是修改原有数据，而是追加到新的位置。这和 LSM-Tree（LevelDB/RocksDB 等）的思路有异曲同工之处。

好处：
- **写性能稳定**：没有随机写的问题，全是顺序 I/O
- **快照和克隆极快**：只需要记录一个指针偏移量
- **扩容无痛**：新 Chunk Server 加入后，数据会自动均衡

#### 并行 NVM 日志

PolarDB 在存储节点上使用了 **NVM（Non-Volatile Memory，如 Intel Optane 或持久化内存）** 来加速日志写入。传统的 Redo Log 写入流程：

```
事务提交 → Redo Log 写入 Buffer → Flush 到磁盘 → 确认
```

PolarFS 的做法是将 Redo Log 的持久化从计算层**卸载到存储层**：

```
事务提交 → 计算层发送 Log Record → 存储层写 NVM → 确认（微秒级）
```

计算层只需要发送日志，不需要等待完整的数据页写入——这就是"**Log is the Database**"的核心含义。

### 3.2 日志即数据库（Log is the Database）

这是 PolarDB 最革命性的设计理念，也是与 Aurora 一脉相承的架构思想。

#### 传统数据库的写入流程

```
1. 事务开始
2. 修改 Buffer Pool 中的 Page
3. 写 Redo Log（WAL，Write-Ahead Logging）
4. 事务提交确认
5. 后台 Checkpoint 将脏页刷到磁盘
```

写入 Redo Log 和写入 Data Page 是两回事。Redo Log 小（记录变更操作），Data Page 大（完整的 16KB 数据页）。

#### PolarDB 的写入流程

```
1. 事务开始
2. 修改 Buffer Pool 中的 Page
3. 写 Redo Log
4. ✅ Redo Log 发送到 PolarFS → 确认 → 事务提交
5. ❌ 不需要写 Data Page（存储层自己处理）
```

**计算层只写 Log，不写 Data Page。** Data Page 的物化（Materialization）由存储层在后台异步完成。

这就是 PolarDB 与 Aurora 不同的地方——Aurora 也是 Log-is-database，但 PolarDB 把 Log 处理得更彻底：

| 维度 | Aurora | PolarDB |
|------|--------|---------|
| Log 处理 | 存储层 4/6 写入确认 | 存储层三副本确认 |
| Page Materialize | 存储层根据 Log 重建 Page | 存储层根据 Log 重建 Page |
| Page 请求 | 存储层返回完整 Page | 存储层返回完整 Page |
| 日志写入 | Quorum 写（4/6） | Quorum 写（三副本） |

#### Crash Recovery 的简化

传统数据库崩溃恢复需要**扫描 Redo Log → 找到 Checkpoint → 重放 Redo → 回滚 Undo**，这个过程可能耗时数分钟甚至数小时。

PolarDB 的崩溃恢复极度简化：
- 计算层崩溃：重新拉起一个实例，通过 PolarFS 读取最新的 Page
- 存储层 Chunk Server 崩溃：Paxos 自动选主，数据从其他副本恢复
- **不需要传统的 Crash Recovery**——因为 Data Page 在存储层已经是"正确的"
- 即使需要恢复，恢复工作在存储层并行执行，速度远超传统方案

### 3.3 物理复制

PolarDB 的只读节点没有单独的数据副本——它们通过 **物理复制** 共享同一份 PolarFS 存储。

```text
主节点（RW）：
  - 接收写入
  - 写 Redo Log → PolarFS
  - Buffer Pool 中的数据是"活的"

只读节点（RO）：
  - 监听 Redo Log（从 PolarFS）
  - 重放 Redo Log 到自己的 Buffer Pool
  - 提供读取服务（读自己的 Buffer Pool + PolarFS）
```

这意味着只读节点的存储成本是**零**——它们只消耗 CPU 和内存，不需要额外磁盘。

**延迟对比：**

| 同步方式 | 典型延迟 | 适用场景 |
|---------|---------|---------|
| MySQL 异步复制 | 100ms - 数秒 | 只读，容忍延迟 |
| MySQL 半同步复制 | 10ms - 100ms | 需要一定一致性 |
| PolarDB 物理复制 | **< 1ms** | 强一致读取 |

这种接近同步复制的延迟，让 PolarDB 的只读节点可以承担**强一致性读取**——对于读多写少的业务，一个主节点 + 多个只读节点可以线性扩展读能力。

### 3.4 并行查询（PX）

PolarDB for MySQL 在 8.0 版本引入了 **Parallel Query（PX）**，这是一个完整的并行查询引擎。

#### 为什么需要 PX？

传统 MySQL 的查询是单线程的——即使 CPU 有 32 个核心，一条 SQL 也只会用一个核。这在 OLTP（在线事务）场景没问题（查询简单、数据量小），但在 AP（分析型）场景中，单线程处理大数据量就力不从心了。

PolarDB 的 PX 引擎实现了：

```sql
SELECT /*+ PARALLEL(8) */ 
    customer_id, 
    SUM(amount) as total_amount
FROM orders
WHERE order_date >= '2026-01-01'
GROUP BY customer_id
ORDER BY total_amount DESC
LIMIT 10;
```

这条带 `PARALLEL(8)` 的 SQL 会被 PX 引擎拆分为 8 个子任务并行执行：

```text
SQL 解析 → Query 优化 → 并行计划生成
                              │
       ┌──────────────────────┼──────────────────────┐
       ▼                      ▼                      ▼
  子任务 1 (Chunk 1)   子任务 2 (Chunk 2)  ...  子任务 8 (Chunk 8)
       │                      │                      │
       └──────────────┬───────┴───────────┬──────────┘
                      ▼                   ▼
               局部聚合结果          局部聚合结果
                      │                   │
                      └────────┬──────────┘
                               ▼
                          Merge → 最终结果
```

每个子任务在自己的 CPU 核心上运行，读取 PolarFS 中不同的 Chunk。PX 的核心优化：

1. **数据亲和性调度**：子任务优先调度到数据所在的 Chunk Server 所在节点，避免网络传输
2. **向量化执行**：按批处理行数据，利用 CPU SIMD 指令加速
3. **结果物化消除**：中间结果直接在内存中传递，不写磁盘
4. **自适应并行度**：根据数据量、可用 CPU 和 IO 压力自动选择并行度

**基准测试（TPC-H 100GB）：**

| 查询 | MySQL 8.0（单线程） | PolarDB PX（16 线程） | 加速比 |
|------|-------------------|----------------------|--------|
| Q1（聚合） | 180s | 12s | **15x** |
| Q6（扫描+聚合） | 65s | 5s | **13x** |
| Q12（分组+排序） | 120s | 9s | **13.3x** |
| Q18（大表 JOIN） | 480s | 35s | **13.7x** |

### 3.5 HTAP 融合

PolarDB 的 HTAP（Hybrid Transactional/Analytical Processing）能力分为两个层次：

**层次一：MPP 扩展**

在计算层挂载一个 **AP 只读节点**，使用 MPP 引擎（基于 PolarDB-X 或原生 PX）处理分析查询。这个 AP 节点和 TP 节点共享同一份存储：

```text
           共享 PolarFS 存储
                  │
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
 RW (TP)     RO1 (TP)     RO2 (AP + MPP)
  写入        支付查询     报表分析（并行执行）
```

这样一来，分析查询不会影响在线交易业务的性能。

**层次二：列存索引**

PolarDB for MySQL 8.0 支持 **列存索引（Columnar Index）**。同一张表可以同时拥有行存和列存两份索引：

```sql
-- 创建列存索引
ALTER TABLE orders ADD COLUMNAR INDEX idx_order_analysis 
    (customer_id, amount, order_date);
```

- 行存：用于 OLTP（点查、小范围扫描）
- 列存：用于 OLAP（大范围聚合、分析）
- 两者保持**实时一致**——对表的任何写入都会自动同步更新列存索引
- 优化器自动选择：查询用到大量列、扫描范围大 → 走列存

这是一个相当实用的特性——不需要 ETL、不需要两个系统、不需要数据同步延迟。

---

## 四、产品矩阵

PolarDB 不是一个单一产品，而是一个产品家族：

| 产品 | 兼容引擎 | 架构形态 | 适用场景 |
|------|---------|---------|---------|
| **PolarDB for MySQL** | MySQL 8.0 | 单主 + 多只读 | 主流 OLTP、互联网业务、SaaS |
| **PolarDB for PostgreSQL** | PostgreSQL 16 | 单主 + 多只读 | 企业级应用、GIS、时序 |
| **PolarDB-X** | MySQL（分布式） | 多分片 + 分布式事务 | 超大规模交易、高并发写入 |
| **PolarDB for Oracle** | Oracle 兼容模式 | PostgreSQL 内核改造 | Oracle 迁移上云 |

### PolarDB for MySQL vs PolarDB-X

| 维度 | PolarDB for MySQL | PolarDB-X |
|------|------------------|-----------|
| 架构 | 共享存储（一写多读） | 计算-存储分离 + 数据分片 |
| 存储上限 | 100TB（单集群） | 无限（水平扩展） |
| 写入能力 | 单点写入，受限于单机 CPU | 多节点写入，线性扩展 |
| 强一致性 | 节点内强一致 | 全局分布式事务（TSO） |
| SQL 兼容性 | 完全兼容 MySQL 8.0 | 兼容 MySQL 语法，部分限制 |
| 扩缩容 | 秒级扩计算，存储自动伸缩 | 需要分片重新平衡 |
| 复杂度 | 低（对用户透明） | 较高（需要关注分片策略） |

**选型建议：**
- 数据量 < 100TB，读多写少 → PolarDB for MySQL
- 数据量持续增长，写入需要水平扩展 → PolarDB-X
- 需要分布式事务支持 → PolarDB-X
- 简单，不想关心分片 → PolarDB for MySQL

---

## 五、性能基准

### 宽表写入吞吐（sysbench oltp_write_only）

| 场景 | MySQL 8.0 自建（32C） | PolarDB for MySQL（32C） | 差距 |
|------|---------------------|------------------------|------|
| 单表 1000 万行 | 8,200 QPS | 38,500 QPS | **4.7x** |
| 10 表 × 100 万行 | 9,500 QPS | 42,000 QPS | **4.4x** |
| P99 延迟 | 35ms | 8ms | **4.4x** 降低 |

### 只读扩展性（PolarDB for MySQL）

| 只读节点数 | 读 QPS（单个 16C 节点） | 总 QPS |
|-----------|----------------------|--------|
| 0（仅主节点） | 直接压主节点 | 15,000 |
| 1 | 15,000 | 30,000 |
| 4 | 14,800 | 59,200 |
| 8 | 14,500 | 116,000 |
| 15 | 14,000 | 210,000 |

只读节点的性能几乎线性扩展——这就是共享存储架构的好处：每个只读节点不需要额外的 IO 负担（数据已经在 PolarFS 上）。

### 大表 DDL

传统 MySQL 给 1TB 表加索引需要数小时（甚至需要第三方工具如 pt-online-schema-change）。

PolarDB 的 **Fast DDL** 特性：

```sql
-- 给 1TB 的表加索引，只需几秒
ALTER TABLE orders ADD INDEX idx_order_date(order_date);
```

原理：Fast DDL 只修改元数据，在 PolarFS 层面完成索引结构创建，不复制整个表。

| 操作 | MySQL 8.0 | PolarDB for MySQL |
|------|-----------|-------------------|
| 加索引 | ~45 分钟 | **2.3 秒** |
| 加列 | ~60 分钟 | **0.5 秒** |
| 修改列类型 | ~45 分钟 | **1.8 秒** |
| 重命名表 | ~30 分钟 | **0.1 秒** |

### 极致弹性

```bash
# 从 4C8G 升级到 32C64G
# PolarDB 耗时：约 50 秒（在线，无中断）
# MySQL 自建耗时：需要迁移，至少 30 分钟
```

PolarDB 的弹性扩缩容通过**新增计算节点 → 挂载同一份存储**实现，不需要数据迁移。

---

## 六、PolarDB vs AWS Aurora

PolarDB 和 Aurora 是亲兄弟——两者都是计算存储分离的云原生数据库。但工程实现上差异不小。

### 架构对比

| 维度 | AWS Aurora | PolarDB for MySQL |
|------|-----------|-------------------|
| 存储引擎 | 自研 InnoDB 改造版 | InnoDB 深度定制版 |
| 分布式文件系统 | Aurora 存储服务 | PolarFS |
| 存储规模 | 128TB | 100TB |
| 计算-存储网络 | EC2 专用网络 | RDMA |
| 只读节点上限 | 15 | 15 |
| DDL 加速 | ❌ 不支持 | ✅ Fast DDL |
| PX 并行查询 | ❌ 不支持（需用 Redshift） | ✅ 内置 |
| 列存 | ❌ 不支持 | ✅ 列存索引 |
| 热备节点 | ✅ Multi-AZ | ✅ 跨 AZ 部署 |
| 全球数据库 | ✅ Aurora Global Database | ✅ 全球数据库 |
| Serverless | ✅ Aurora Serverless v2 | ✅ PolarDB Serverless |
| Oracle 兼容 | ❌ | ✅ PolarDB for Oracle |

### 性能对比（基于公开基准测试数据）

| 测试项 | Aurora 3 (MySQL 8.0 兼容) | PolarDB for MySQL | 备注 |
|--------|--------------------------|-------------------|------|
| 单节点写入 QPS | ~32,000 | ~38,500 | PolarDB 略高 |
| 8 只读节点总 QPS | ~180,000 | ~210,000 | PolarDB 扩展性更优 |
| 故障切换时间 | ~30s（典型） | ~10s（典型） | PolarFS 恢复更快 |
| 存储延迟（P99） | ~2ms | ~0.5ms | RDMA 优势 |
| 100GB 数据恢复 | ~10min | ~2min | PolarDB 并行恢复 |

> 注意：数据来自各产品 v3 / 最新版的官方基准测试，硬件配置相近（16C/32C 级别），实际表现因工作负载而异。

### 商业维度

| 维度 | Aurora | PolarDB |
|------|--------|---------|
| 价格 | 按使用量 x 1.2 + I/O 费 | 包年包月 / 按量 + 存储费 |
| 开放程度 | 仅 AWS 平台 | 阿里云 + 私有化部署（PolarDB Stack） |
| 开源 | ❌ 完全闭源 | ⚠️ 基础版社区版开源（PolarDB-PG 已开源） |
| 生态 | AWS 全家桶 | 阿里云生态（DTS、DMS、DataWorks） |

### 关键差异总结

1. **网络层**：PolarDB 的 RDMA 是杀手锏——Aurora 使用通用 EC2 网络连接到存储层，延迟天生比 RDMA 高一个数量级
2. **并行查询**：PolarDB 内置 PX 引擎，Aurora 的分析查询需要配合 Redshift / Aurora DSQL 才能完成
3. **DDL 加速**：PolarDB 的 Fast DDL 是线上运维的"救命"特性，Aurora 不支持
4. **Oracle 兼容**：PolarDB for Oracle 是阿里云独有的差异化卖点，对从 Oracle 迁移的企业有巨大吸引力
5. **开放生态**：PolarDB 有开源版本（PolarDB for PostgreSQL），Aurora 完全闭源

---

## 七、PolarDB 开源生态

### PolarDB for PostgreSQL（开源版）

阿里云在 2021 年将 PolarDB for PostgreSQL 的**内核开源**，GitHub 地址：[ApsaraDB/PolarDB-for-PostgreSQL](https://github.com/ApsaraDB/PolarDB-for-PostgreSQL)

开源版包含：
- PolarFS 共享存储引擎
- 计算-存储分离架构
- 物理复制协议
- 并行查询

这对于开发者来说意义重大——可以在本地环境部署 PolarDB 进行开发和测试，不需要云账号。

### 开源版 vs 阿里云版

| 特性 | 开源版 | 阿里云托管版 |
|------|--------|-------------|
| PolarFS | ✅ | ✅ 性能更优 |
| RDMA | ❌（通用 TCP） | ✅ |
| Fast DDL | ✅ | ✅ |
| PX 并行查询 | ✅ | ✅ 资源更优 |
| 列存索引 | ❌ | ✅ |
| 全球数据库 | ❌ | ✅ |
| Serverless | ❌ | ✅ |
| 运维控制台 | 自建 | 阿里云 DMS |
| 备份恢复 | 基础版 | 自动 + 跨 Region |

### 部署 PolarDB-PG 开源版

```bash
# 克隆仓库
git clone https://github.com/ApsaraDB/PolarDB-for-PostgreSQL.git
cd PolarDB-for-PostgreSQL

# 使用 Docker 运行（推荐）
docker pull polardb/polardb_pg_local_instance
docker run -d --name polardb_pg \
  -p 5432:5432 \
  polardb/polardb_pg_local_instance

# 连接
psql -h localhost -p 5432 -U polardb
```

---

## 八、实战指南

### 8.1 迁移 MySQL 到 PolarDB for MySQL

#### 使用 DTS（数据传输服务）在线迁移

```text
源端 MySQL ───→ DTS 任务 ───→ PolarDB 目标端
                 │
            全量迁移（初始同步）
                  +
            增量迁移（CDC）
                 │
           切换时：停写源端 → 追平增量 → 切换 DNS
```

步骤概览：

1. 在阿里云上创建 PolarDB 集群（选需要的规格）
2. 创建 DTS 迁移任务，选择源端和目标端
3. DTS 自动完成全量迁移 + 增量同步
4. 验证数据一致性（DTS 内置校验）
5. 切换业务流量到 PolarDB

**关键参数优化（迁移后）：**

```sql
-- 调整连接池
SET GLOBAL max_connections = 2000;

-- 开启并行查询
SET GLOBAL px_enabled = 1;

-- 调整 Buffer Pool（按节点内存的 70% 设置）
SET GLOBAL innodb_buffer_pool_size = 50 * 1024 * 1024 * 1024;  -- 50GB

-- 开启列存（如果有分析查询）
SET GLOBAL imci_enabled = 1;
```

#### 常见迁移问题

**Q：兼容性如何？**

PolarDB for MySQL 基于 MySQL 8.0 深度定制，SQL 语法兼容度 > 99%。最常见的不兼容：
- 某些 MySQL 8.0 的 Performance Schema 表有些差异
- PolarDB 特有的参数（以 `px_`、`polar_` 开头）

**Q：数据量太大，全量迁移时间过长？**

使用 DTS 的并行加载功能，100GB 数据通常能在 1-2 小时内完成全量迁移。

### 8.2 只读节点使用场景

PolarDB 的 15 个只读节点可以按需分配：

```text
主节点（RW）：写入 + 实时交易查询
RO1：报表查询（MySQL Workbench / BI 工具连接）
RO2：大数据平台 ETL 读取（Seatunnel / DataX 连接）
RO3：搜索索引构建（从数据库拉取数据到 ES）
RO4~RO6：高并发读业务（如商品详情页）
RO7：DBA 查慢查询、做备份
```

每个只读节点都有独立的连接地址，业务层通过**读写分离**或**应用层路由**分发。

### 8.3 读写分离配置

**应用层直接路由（推荐）：**

```python
# Python 示例：在应用中手动路由
import pymysql

class PolarDBRouter:
    def __init__(self, write_endpoint, read_endpoints):
        self.write_conn = pymysql.connect(host=write_endpoint, ...)
        self.read_conns = [
            pymysql.connect(host=ep, ...) for ep in read_endpoints
        ]
        self.read_index = 0
    
    def execute_write(self, sql, params=None):
        # 写操作发到主节点
        return self._execute(self.write_conn, sql, params)
    
    def execute_read(self, sql, params=None):
        # 读操作轮询只读节点
        conn = self.read_conns[self.read_index % len(self.read_conns)]
        self.read_index += 1
        return self._execute(conn, sql, params)
```

**代理层路由（使用阿里云 DBProxy / 数据库代理）：**

阿里云提供了**数据库代理**服务，自动在主节点和只读节点之间分发请求：

```text
应用 → 数据库代理 → RW Node (写)
                  → RO Nodes (读，轮询/权重)
```

- 自动识别 SELECT / DML 并路由
- 短连接场景下性能更好（代理维护长连接池）
- 支持事务内读写一致性（同一事务内的读走主节点）

### 8.4 备份与恢复策略

```sql
-- 开启自动备份（阿里云控制台设置，推荐每天一次）
-- 保留周期：7-30 天

-- 创建手动备份（业务低峰期）
CALL polardb_create_backup('pre_upgrade_backup');

-- 查看备份列表
CALL polardb_show_backups();

-- 恢复到新集群
-- 阿里云控制台：选中备份 → 恢复到新 PolarDB 集群

-- 表级恢复（不需要全量恢复）
CALL polardb_restore_table('backup_id', 'db_name', 'table_name');
```

PolarDB 的备份也是共享存储的副产品——备份是 PolarFS 的快照，不复制数据，速度极快。

---

## 九、典型应用场景

### 场景一：电商大促（高并发读写）

**需求：** 双十一期间，订单系统的写入峰值 50 万 QPS，商品详情页读取 200 万 QPS。

**方案：**
- 1 个 PolarDB for MySQL 主节点（64C）处理写入
- 8 个只读节点（32C）处理商品详情页查询
- 数据库代理自动分发只读流量
- Redis 作为旁路缓存，降低数据库直接压力

**效果：** 只读节点线性扩展读能力，主节点的写入稳定性在大促期间保持 P99 < 10ms。

### 场景二：SaaS 多租户平台

**需求：** 一个 SaaS 平台，有 3000+ 租户，每个租户的数据量从 10MB 到 50GB 不等。需要满足不同租户的计算资源隔离，同时降低总体存储成本。

**方案：**
- 单个 PolarDB 集群承载所有租户（共享存储，数据隔离在 schema/租户字段层面）
- 为高负载的租户单独分配只读节点
- 非高峰期降配只读节点（按小时弹性）

**效果：** 存储统一管理，无需为每个租户分配独立实例。只读节点弹性扩缩容让资源精确匹配负载。

### 场景三：Oracle 迁移

**需求：** 金融客户有 500+ 个 Oracle 数据库需要迁移到云上，核心瓶颈是 Oracles 的许可费用和运维成本。

**方案：**
- 使用 PolarDB for Oracle（兼容模式）作为目标
- ADAM（数据库迁移评估）自动评估兼容性
- DTS 实时同步迁移（全量 + 增量）
- PolarDB for Oracle 兼容 90%+ 的 Oracle PL/SQL 语法

**效果：** 相比 Oracle，许可费节省 60%+，运维复杂度大幅降低。核心交易系统迁移后性能持平甚至有所提升。

### 场景四：数据库+大数据融合

**需求：** 数据分析团队需要每小时拉取线上数据库的最新数据到 MaxCompute 进行分析，传统方案影响主库性能。

**方案：**
- 单独分配一个只读节点给数据平台
- ETL 工具（DataX / Seattunnel）直接连接只读节点
- 分析查询在只读节点上执行，不影响主库

**效果：** 主库零影响，分析查询可以在只读节点上拿到最新数据（物理复制延迟 < 1ms）。

---

## 十、选型决策树

```
当前用的是什么数据库？
│
├── MySQL
│   ├── 数据量 < 10TB，无需分布式 → PolarDB for MySQL ✅
│   ├── 数据量 > 10TB，需要水平扩展 → PolarDB-X
│   └── 有分析查询，不想搭数仓 → PolarDB for MySQL + 列存索引 ✅
│
├── PostgreSQL
│   ├── 企业应用，数 TB 级别 → PolarDB for PostgreSQL ✅
│   └── GIS / 时序数据 → PolarDB for PostgreSQL ✅
│
├── Oracle
│   └── 希望降低许可成本 → PolarDB for Oracle ✅
│
└── 自建 / 其他云
    ├── 想上云但不想绑定 → PolarDB for PostgreSQL 开源版
    └── 规模不大，先简单迁移 → PolarDB for MySQL（兼容性最好）
```

---

## 十一、踩坑与最佳实践

### 1. 长连接池不要过大

```sql
-- ❌ 错误配置
SET GLOBAL max_connections = 5000;

-- ✅ 建议
SET GLOBAL max_connections = 500;  -- 配合连接池（Druid, HikariCP 等）
```

PolarDB 的计算节点是无状态的，但每个连接仍然消耗内存。5000 个长连接意味着内存被连接占用而不是 Buffer Pool。

### 2. 避免在只读节点上跑大事务

只读节点的物理复制依赖 Redo Log 的重放。如果只读节点在重放期间执行一个大查询（扫描全表），可能会出现：

- **复制延迟飙升**：重放工作被大查询"饿死"
- **Buffer Pool 污染**：大查询需要的 Page 把热数据 Page 挤出去

```sql
-- ❌ 不要在只读节点上做全表扫描
SELECT * FROM orders WHERE YEAR(order_date) = 2025;

-- ✅ 给只读节点建必要索引，用小范围查询
SELECT * FROM orders WHERE order_date >= '2025-01-01' AND order_date < '2026-01-01';
```

### 3. 列存索引的适用边界

列存索引不是万能的：

- 适合范围扫描 + 聚合（GROUP BY、SUM、AVG）
- 不适合点查（WHERE id = 123）
- 不适合频繁 UPDATE / DELETE 的表（列存的更新代价高）
- 列存会额外占用 20-40% 的存储空间

### 4. 跨可用区部署

```text
可用区 A（主）    可用区 B（备）    可用区 C（备）
   │                  │                │
   └──────────────────┴────────────────┘
                    │
                PolarFS（跨 AZ 三副本）
```

跨 AZ 部署时，PolarFS 的三副本会自动分布到三个不同的可用区。这意味着：

- 单 AZ 故障不影响数据可用性
- 只读节点可以分布在不同 AZ
- 跨 AZ 的写入延迟会增加（因为需要三副本确认，跨 AZ 网络延迟约 1-2ms）

**建议：** 核心业务一定要跨 AZ 部署。非核心业务可以单 AZ（减少写入延迟）。

### 5. Fast DDL 的使用限制

Fast DDL 虽然快，但有一些限制：

```sql
-- ✅ 支持
ALTER TABLE t ADD INDEX idx(col);
ALTER TABLE t ADD COLUMN col INT;
ALTER TABLE t RENAME INDEX idx_old TO idx_new;
ALTER TABLE t DROP INDEX idx;
ALTER TABLE t MODIFY COLUMN col BIGINT;

-- ❌ 不支持（会走传统 DDL 路径）
ALTER TABLE t DROP COLUMN col;       -- 列删除需要重建表
ALTER TABLE t CHANGE col col_new INT; -- 列重命名需要重建（部分版本可能优化）
ALTER TABLE t ADD FOREIGN KEY ...;   -- 外键约束需要检查所有行
```

---

## 十二、总结

PolarDB 的核心思想其实不复杂——**把存储从计算中解放出来**。

这个简单的想法带来的连锁效应是巨大的：
- 计算弹性 → 秒级扩缩容，不再绑定存储
- 共享存储 → 只读节点零存储成本，扩展读能力
- 日志下沉 → 计算层更轻量，崩溃恢复更快
- 列存索引 → 在线分析能力，告别繁琐的 ETL

PolarDB 和 AWS Aurora 走的是同一条技术路线，但 PolarDB 在三个方面做出了显著差异化：

1. **工程创新**：RDMA 网络层、PolarFS 的 Chunk 架构、Fast DDL 等，在延迟和运维体验上更有优势
2. **Oracle 兼容**：抓住了中国大量 Oracle 存量用户向云迁移的核心痛点
3. **开源生态**：PolarDB-for-PostgreSQL 的开源版本降低了学习成本和脱离云平台的风险

对于一个团队来说，PolarDB 最直接的价值在于：

- **不需要 DBA 做分库分表**——100TB 以内一个集群搞定
- **不需要为只读副本付"冤枉钱"**——15 个只读节点共享同一份存储
- **大表 DDL 不再是运维噩梦**——几秒完成
- **从 Oracle 迁移真的可行了**——PolarDB for Oracle 是一个认真的方案

---

## 参考资源

- [PolarDB 官方文档](https://help.aliyun.com/product/58754.html)
- [PolarDB for PostgreSQL GitHub](https://github.com/ApsaraDB/PolarDB-for-PostgreSQL)
- [PolarFS 论文](http://www.vldb.org/pvldb/vol11/p1840-cao.pdf) — Cao et al., "PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System for Database Storage", VLDB 2018
- [PolarDB 论文](https://arxiv.org/abs/2012.13651) — "PolarDB: A Cloud-Native Database Architecture with Shared Storage"
- [PolarDB 开发者指南](https://developer.aliyun.com/group/polardb)
- [AWS Aurora 白皮书](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html)
