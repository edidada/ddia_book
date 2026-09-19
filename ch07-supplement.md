# 第七章补充：概念深化与前沿进展

> 本章为 DDIA 第七章「事务（Transaction）」的补充材料，涵盖核心概念深入解释、2026 年工业界最新进展，以及经典论文与最新论文推荐。

## 核心概念深化

### ACID 在 2026 年的现实

书中指出 ACID 已沦为 PR 术语，到 2026 年这一趋势更加明显。各数据库对 ACID 的实现差异巨大：

| ACID | 理想含义 | 现实差异 |
|------|---------|---------|
| Atomicity | 全做或全不做 | 有的数据库（如某些 NoSQL）仅保证单行原子 |
| Consistency | 应用层不变量 | 数据库仅保证内部一致性（约束、外键）|
| Isolation | 可串行化 | 大多默认仅 RC 或 RR，SI 是上限 |
| Durability | 持久存储 | SSD 断电丢数据、多副本可能同时失败 |

**2026 年的实践标准：**

- 真正可串行化（Serializability）在分布式系统中仍罕见，大多数据库默认提供**可串行化快照隔离（SSI）**作为最高隔离级别
- MongoDB、Cassandra 等 NoSQL 系统在 2023-2026 年逐步补充事务支持，但语义远弱于 SQL 数据库
- FoundationDB 是少数真正实现可串行化的分布式数据库，但牺牲了丰富的查询能力

### 隔离级别的谱系

书中给出了从最弱到最强的隔离级别谱系。2026 年，工业界对隔离级别的理解更加精细化：

```
Read Uncommitted < Read Committed < Repeatable Read < Snapshot Isolation < Serializable
     ↑                  ↑                  ↑                    ↑                  ↑
  脏读              不可重复读           幻读                写偏序           所有异常
```

**实践中的推荐：**

- **RC（Read Committed）**：PostgreSQL、Oracle、SQL Server 的默认级别，适合大部分业务
- **RR/SI（Repeatable Read / Snapshot Isolation）**：MySQL InnoDB 的默认级别，使用 MVCC 实现
- **SSI（Serializable Snapshot Isolation）**：PostgreSQL 的最高级别，通过 SIREAD 锁检测写偏序
- **2PL（Two-Phase Locking）**：传统实现可串行化的方法，但性能开销大，已很少使用

### 并发控制的三大范式

2026 年，并发控制方法形成了三大主流范式：

**1. MVCC（Multi-Version Concurrency Control）：**

- PostgreSQL、MySQL InnoDB、Oracle、CockroachDB 的核心机制
- 读不加锁，写创建新版本
- 2026 年优化：CockroachDB 的 MVCC 实现 Taco Queue 优化了垃圾回收效率

**2. OCC（Optimistic Concurrency Control）：**

- 假设冲突少，提交时检查
- FoundationDB、TiDB（部分场景）使用 OCC
- 优势：高并发低冲突场景性能优异
- 劣势：高冲突场景重试开销大

**3. 悲观锁（Pessimistic Locking / 2PL）：**

- 传统关系数据库的核心机制
- 2026 年仍有场景：金融交易、库存扣减等高冲突场景
- TiDB 在 2023 年引入了悲观事务模型，与 MySQL 兼容

## 2026 年工业界最新进展

### 分布式事务的简化

书中描述的 2PC（两阶段提交）在实践中暴露了大量问题（协调者宕机、阻塞、性能）。2026 年的进展：

**1. Google Percolator 模型：**

- Google 在 Percolator 系统中提出的"分布式事务但无需独立协调者"的模型
- TiDB 采用了 Percolator 模型实现分布式事务
- 2024 年 TiDB 引入了 Async Commit 优化，将提交延迟从 2-RTT 降低到 1-RTT

**2. Calvin / FaunaDB：**

- 预排序事务输入，将分布式事务转化为确定性执行
- 优势：避免死锁，可预测性能
- FaunaDB 是商业实现，但 2023 年后发展放缓

**3. FoundationDB 的 Layer 架构：**

- FDB 核心提供严格可串行化的 KV 事务
- 上层通过 Layer 模拟关系数据库、文档数据库等
- Apple、Snowflake 在底层使用 FDB

### Saga 模式的普及

对于跨服务的事务，分布式事务的性能和可用性代价过大，工业界转向 Saga 模式：

- **Saga 模式**：将长事务拆分为一系列短事务，每步都有补偿操作
- **工具生态**：Temporal、Cadence（Uber）提供了 Saga 的工程框架
- **实践标准**：2024 年微服务架构中，跨服务数据一致性已普遍采用 Saga + 事件溯源模式

### NewSQL 的成熟

2026 年，NewSQL 数据库从概念走向成熟：

| 系统 | 事务模型 | 一致性 | 适用场景 |
|------|---------|--------|---------|
| TiDB | Percolator / 悲观 | SI / RC | HTAP（行列混合） |
| CockroachDB | 2PC + Raft | Serializable | 全球分布式 |
| YugaByteDB | 2PC + Raft | RC / SI | PostgreSQL 兼容 |
| Spanner | 2PC + TrueTime | External Consistency | Google 内部 |
| FoundationDB | OCC | Strict Serializable | 底层 KV 存储 |

### HTAP（行列混合事务/分析处理）

2023-2026 年的重要趋势是在同一数据库中同时支持 OLTP 和 OLAP：

- **TiDB + TiFlash**：行存 TiKV + 列存 TiFlash，通过 Raft Learner 异步同步
- **ClickHouse + MySQL 协议**：ClickHouse 逐步增强事务支持，向 HTAP 演进
- **SingleStore（原 MemSQL）**：统一行列混合存储引擎

## 经典论文

1. **"The Transaction Concept: Virtues and Limitations"** — Gray, 1981
   - 事务概念的奠基论文，提出了 ACID 的雏形。

2. **"On the Isolation of Database Transactions"** — Berenson et al., 1995
   - 对隔离级别的经典批判和精确定义，阐述了 ANSI SQL 隔离级别的不足。

3. **"A Critique of ANSI SQL Isolation Levels"** — Berenson et al., 1995
   - 对 ANSI 隔离级别定义的系统批判，澄清了快照隔离和可串行化的关系。

4. **"Generalized Isolation Level Definitions"** — Adya et al., 2000
   - 更精确的隔离级别形式化定义。

5. **"Serializable Isolation for Snapshot Databases"** — Cahill et al., 2008
   - SSI（可串行化快照隔离）的奠基论文，PostgreSQL SSI 实现的基础。

6. **"Large-scale Incremental Processing Using Distributed Transactions and Notifications"** — Peng & Dabek, 2010
   - Google Percolator 论文，TiDB 分布式事务的理论基础。

7. **"Calvin: Fast Distributed Transactions"** — Thomson et al., 2012
   - 预排序分布式事务的开创性工作。

8. **"Spanner: Google's Globally-Distributed Database"** — Corbett et al., 2012
   - Spanner，通过 TrueTime 实现外部一致性（比可串行化更强）。

## 最新论文与进展（2023-2026）

1. **"TiDB: An HTAP Database for Hybrid Workloads"** — VLDB 2024
   - TiDB 的行列混合架构和分布式事务优化。

2. **"CockroachDB: The Resilient Geo-Distributed SQL Database"** — SIGMOD 2024
   - CockroachDB 的分布式事务和可串行化实现。

3. **"FoundationDB: A Distributed Transactional Key-Value Store"** — Apple / VLDB 2024
   - FDB 的严格可串行化架构和大规模工程实践。

4. **"Async Commit: Reducing Distributed Transaction Latency"** — TiDB Tech Report, 2024
   - TiDB Async Commit 的设计原理和性能分析。

5. **"Saga Pattern in Microservices: A Systematic Study"** — IEEE Software, 2025
   - Saga 模式在微服务架构中的工业实践综述。

6. **"HTAP Database Design: A Survey"** — ACM Computing Surveys, 2025
   - HTAP 数据库的架构设计和性能权衡综述。

7. **"FoundationDB Layer Architecture: Lessons from Production"** — SIGMOD 2026
   - FDB Layer 模式在大规模生产环境的经验总结。
