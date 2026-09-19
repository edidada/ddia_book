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

## 工业界中间件软件实践

### MVCC 中间件：MySQL InnoDB

InnoDB 的 MVCC 实现是理解事务隔离的最佳入口：

```
InnoDB MVCC 实现：
  每行记录有两个隐藏列：
    - trx_id：最后修改该行的事务ID
    - roll_pointer：指向 undo log（历史版本）

  读取流程：
    1. 获取当前 Read View（可见事务快照）
    2. 读取行的 trx_id
    3. 如果 trx_id 在 Read View 中不可见 → 沿 roll_pointer 找历史版本
    4. 直到找到可见版本
```

```sql
-- MySQL 隔离级别设置
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别（会话级）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;  -- InnoDB 默认
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 查看当前事务ID和活跃事务
SELECT * FROM information_schema.innodb_trx;
SELECT * FROM information_schema.innodb_locks;
SELECT * FROM information_schema.innodb_lock_waits;

-- 查看某个事务的锁等待
SELECT
    r.trx_id AS waiting_trx,
    r.trx_mysql_thread_id AS waiting_thread,
    b.trx_id AS blocking_trx,
    b.trx_mysql_thread_id AS blocking_thread
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;
```

### 分布式事务中间件：Seata

Seata（阿里开源）是微服务分布式事务的标杆中间件，支持多种事务模式：

**AT 模式（自动补偿）：**

```java
// Seata AT 模式：对业务无侵入
@GlobalTransactional(timeout = 60000, name = "createOrder")
public void createOrder(Order order) {
    // 1. 扣减库存（本地事务）
    storageService.deduct(order.getProductId(), order.getCount());

    // 2. 扣减余额（本地事务）
    accountService.debit(order.getUserId(), order.getTotal());

    // 3. 创建订单（本地事务）
    orderDao.insert(order);

    // Seata 自动管理全局事务
    // 任何一步失败，自动执行反向补偿（undo_log）
}
```

**TCC 模式（Try-Confirm-Cancel）：**

```java
// Seata TCC 模式：需要业务实现三个方法
@LocalTCC
public interface OrderTccAction {

    @TwoPhaseBusinessAction(name = "createOrder",
        commitMethod = "confirm", rollbackMethod = "cancel")
    boolean tryCreate(BusinessActionContext ctx,
                      @BusinessActionParameter String userId,
                      @BusinessActionParameter BigDecimal total);

    boolean confirm(BusinessActionContext ctx);
    boolean cancel(BusinessActionContext ctx);
}
```

### 分布式事务中间件：TiDB Percolator

TiDB 使用 Percolator 模型实现分布式事务，无需独立协调者：

```
Percolator 事务流程：
  1. Prewrite：
     - 对每个 Key 加锁（写入 primary 锁和 secondary 锁）
     - primary 锁是事务的"锚点"
  2. Commit：
     - 先提交 primary Key（删锁，写新版本）
     - 异步清理 secondary Key
  3. 故障恢复：
     - 如果遇到锁，检查 primary Key 状态
     - primary 已提交 → 提交；primary 未提交 → 回滚
```

```sql
-- TiDB 事务执行
BEGIN;
-- 乐观事务（默认）
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- TiDB 乐观事务在高冲突场景需要重试
-- 悲观事务模式（4.0+，与 MySQL 兼容）
SET tidb_txn_mode = 'pessimistic';
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 加悲观锁
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### 事务隔离中间件：PostgreSQL SSI

PostgreSQL 的 SSI（可串行化快照隔离）通过 SIREAD 锁检测写偏序异常：

```sql
-- PostgreSQL 隔离级别设置
SET default_transaction_isolation = 'serializable';

-- SSI 示例：医院值班管理
-- 两个事务同时检查"至少有2人值班"条件，都可能通过检查后都退出

-- 事务 A
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM oncall WHERE active = true;
-- count = 3
-- 检查 count >= 2，满足
UPDATE oncall SET active = false WHERE doctor = 'Alice';
COMMIT;  -- 可能成功

-- 事务 B（同时执行）
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM oncall WHERE active = true;
-- count = 3（事务A还没提交）
UPDATE oncall SET active = false WHERE doctor = 'Bob';
COMMIT;  -- PostgreSQL SSI 检测到写偏序，中止此事务
-- ERROR: could not serialize access due to read/write dependencies
```

### Saga 模式中间件：Temporal

Temporal 是 Saga 模式的标杆执行引擎：

```go
// Temporal Saga 模式：订单创建流程
func CreateOrderWorkflow(ctx workflow.Context, order Order) error {
    // Saga 补偿函数链
    var compensations []func(ctx workflow.Context) error

    // Step 1: 扣减库存
    err := workflow.ExecuteActivity(ctx, DeductInventory, order).Get(ctx, nil)
    if err != nil {
        return err
    }
    compensations = append(compensations, RestoreInventory)

    // Step 2: 扣减余额
    err = workflow.ExecuteActivity(ctx, DeductBalance, order).Get(ctx, nil)
    if err != nil {
        // 执行补偿：回滚库存
        return workflow.ExecuteActivity(ctx, RestoreInventory, order).Get(ctx, nil)
    }
    compensations = append(compensations, RestoreBalance)

    // Step 3: 创建订单
    err = workflow.ExecuteActivity(ctx, CreateOrder, order).Get(ctx, nil)
    if err != nil {
        // 按逆序执行所有补偿
        for i := len(compensations) - 1; i >= 0; i-- {
            workflow.ExecuteActivity(ctx, compensations[i], order).Get(ctx, nil)
        }
        return err
    }

    return nil
}
```

### 事务性 Kafka 中间件

Kafka 的事务 API 实现端到端精确一次语义：

```java
// Kafka 事务性生产者
Properties props = new Properties();
props.put("transactional.id", "order-processor-1");  // 事务 ID（跨重启幂等）
props.put("enable.idempotence", true);                // 幂等生产者
props.put("acks", "all");

Producer<String, String> producer = new KafkaProducer<>(props);

// 初始化事务
producer.initTransactions();

try {
    producer.beginTransaction();

    // 消费 → 处理 → 生产（原子操作）
    for (ConsumerRecord<String, String> record : consumer.poll(Duration.ofMillis(100))) {
        String result = process(record.value());
        producer.send(new ProducerRecord<>("processed", record.key(), result));
    }

    // 提交消费者偏移量（作为事务的一部分）
    producer.sendOffsetsToTransaction(
        consumer_offsets,
        consumer.groupMetadata()
    );

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

### FoundationDB 中间件

FDB 提供严格可串行化的 KV 事务，Apple 用其支撑 iCloud 存储：

```python
# FDB Python 客户端
import fdb

db = fdb.open()

# FDB 事务
@fdb.transactional
def transfer(db, from_account, to_account, amount):
    # 读取余额（乐观并发）
    from_balance = db[from_account]
    to_balance = db[to_account]

    if int(from_balance) < amount:
        raise Exception("Insufficient balance")

    # 原子写入
    db[from_account] = str(int(from_balance) - amount)
    db[to_account] = str(int(to_balance) + amount)

# 执行事务
transfer(db, b"account:1", b"account:2", 100)

# FDB 自动处理冲突检测、重试、快照隔离
# 严格可串行化：所有事务按某顺序排列执行
```

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
