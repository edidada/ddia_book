# 第十一章补充：概念深化与前沿进展

> 本章为 DDIA 第十一章「流处理（Stream Processing）」的补充材料，涵盖核心概念深入解释、2026 年工业界最新进展，以及经典论文与最新论文推荐。

## 核心概念深化

### 流与批的统一

书中结尾提到流处理是批处理的"无界化、增量化"。到 2026 年，流批统一（Stream-Batch Unification）已成为主流范式：

**Flink 的流批一体：**

- Apache Flink 2.0（2024）实现了真正的流批统一——同一套 API，同一套引擎
- 流模式：逐条处理，低延迟
- 批模式：批量处理，高吞吐
- Flink SQL 同时支持流和批查询

**-table API / SQL 统一：**

- RisingWave、Materialize 将流处理表达为 SQL 持续查询（Continuous Query）
- 用户写 SQL 定义物化视图，系统自动维护增量更新
- "流处理即持续物化视图"范式降低了流处理的使用门槛

### 事件时间 vs 处理时间

书中区分了事件时间和处理时间，2026 年的工程实践更加成熟：

**Watermark（水位线）机制：**

- Watermark 表示"所有时间戳 ≤ T 的事件都已到达"
- 用于处理乱序事件——即使事件延迟到达，仍可正确处理
- Flink 的 Watermark 策略：Bounded Out Of Orderness、Periodic Watermark

**Late Event 处理：**

- 事件时间晚于 Watermark 的事件称为 Late Event
- 允许延迟（Allowed Lateness）：在窗口关闭后仍接受一定时间内的迟到事件
- Side Output：将迟到事件输出到单独的流，不丢弃

**工业实践：**

- 设定 Watermark 延迟需要权衡：太小 → 数据可能不完整；太大 → 延迟增加
- 典型值：秒级延迟（实时分析）到分钟级（离线近似）

### 消息系统（Messaging System）的演进

书中讨论了消息系统的多种模型。2026 年，消息系统格局发生了重大变化：

| 系统 | 定位 | 2026 年状态 |
|------|------|------------|
| Apache Kafka | 日志流平台 | 仍是绝对主导，KRaft 模式成熟 |
| Redpanda / WarpStream | Kafka 替代 | 线程模型 / 对象存储后端 |
| Apache Pulsar | 云原生消息 | 存算分离，但生态不及 Kafka |
| RabbitMQ | AMQP 消息队列 | 仍在传统企业应用中使用 |
| NATS | 轻量消息 | 云原生边缘场景 |
| AWS Kinesis | 云消息服务 | AWS 生态内使用 |

**Kafka 的演进：**

- KRaft 模式：废弃 ZooKeeper，元数据自管理
- Tiered Storage：热数据在本地，冷数据在 S3
- Kafka 4.0（2024）：队列语义（Queue Semantics），支持点对点消费
- Partition 数不再受 ZooKeeper 限制

**WarpStream 的创新：**

- 将 Kafka 的日志直接存储在 S3 上，无需本地磁盘
- 生产者先写到 S3 的 Write-Ahead 文件，消费者从 S3 拉取
- 延迟增加但成本降低 10 倍以上
- 2024 年被 Confluent 收购

## 工业界中间件软件实践

### 流处理中间件：Apache Kafka

Kafka 是流处理生态的核心基础设施，提供分区日志（Partitioned Log）抽象：

```
Kafka 核心概念：
  Topic → 分区（Partition）→ 段（Segment）→ 偏移量（Offset）

  生产者写入：Key → hash(Key) mod num_partitions → Partition Leader
  消费者读取：Consumer Group → 每个分区被一个消费者消费
```

**Kafka 生产者配置：**

```java
// Java Kafka 生产者：精确一次语义
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// 幂等生产者：避免重试导致重复
props.put("enable.idempotence", true);
props.put("acks", "all");
props.put("max.in.flight.requests.per.connection", 5);
props.put("retries", 3);

// 批量和压缩
props.put("batch.size", 16384);        // 批量大小 16KB
props.put("linger.ms", 10);            // 等待 10ms 凑批
props.put("compression.type", "zstd");  // ZSTD 压缩
props.put("buffer.memory", 33554432);   // 32MB 缓冲区

Producer<String, String> producer = new KafkaProducer<>(props);
```

**Kafka 消费者配置：**

```java
// Java Kafka 消费者：精确一次语义
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "order-processor");
props.put("enable.auto.commit", "false");           // 手动提交偏移量
props.put("auto.offset.reset", "earliest");
props.put("isolation.level", "read_committed");     // 只读取已提交的事务消息
props.put("max.poll.records", 500);                 // 单次拉取最多 500 条
props.put("max.poll.interval.ms", 300000);          // 5分钟超时

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("orders"));

while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        process(record.value());
    }
    // 手动同步提交
    consumer.commitSync();
}
```

### 流处理中间件：Apache Flink

Flink 是 2026 年流处理领域的标杆，支持精确一次、事件时间语义和状态管理：

**Flink DataStream API：**

```java
// Flink Java：实时订单处理
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 精确一次语义
env.enableCheckpointing(60000);  // 每60秒 checkpoint
env.getCheckpointConfig().setCheckpointingMode(EXACTLY_ONCE);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
env.getCheckpointConfig().setCheckpointTimeout(60000);
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// 从 Kafka 消费
KafkaSource<String> source = KafkaSource.<String>builder()
    .setBootstrapServers("kafka:9092")
    .setTopics("orders")
    .setGroupId("order-processor")
    .setStartingOffsets(OffsetsInitializer.earliest())
    .setValueOnlyDeserializer(new SimpleStringSchema())
    .build();

DataStream<String> orders = env.fromSource(
    source,
    WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5)),  // 允许5秒乱序
    "kafka-orders"
);

// 按用户分组，每5分钟窗口统计消费总额
orders
    .map(s -> parseOrder(s))  // JSON → Order 对象
    .keyBy(Order::getCustomerId)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new SumAggregator())
    .sinkTo(KafkaSink.<Double>builder()
        .setBootstrapServers("kafka:9092")
        .setRecordSerializer(new UserSpentSerializer("user-spent"))
        .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
        .build()
    );

env.execute("Order Processing");
```

**Flink SQL（Table API）：**

```sql
-- Flink SQL：定义流表
-- Kafka 源表（流式）
CREATE TABLE orders_stream (
    order_id STRING,
    customer_id STRING,
    total DECIMAL(10, 2),
    order_time TIMESTAMP(3),
    -- 事件时间字段和水印
    WATERMARK FOR order_time AS order_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'kafka',
    'topic' = 'orders',
    'properties.bootstrap.servers' = 'kafka:9092',
    'format' = 'avro-confluent',
    'avro-confluent.url' = 'http://schema-registry:8081',
    'scan.startup.mode' = 'group-offsets'
);

-- 滚动窗口聚合
SELECT
    customer_id,
    TUMBLE_START(order_time, INTERVAL '5' MINUTE) AS window_start,
    SUM(total) AS total_spent,
    COUNT(*) AS order_count
FROM orders_stream
GROUP BY
    customer_id,
    TUMBLE(order_time, INTERVAL '5' MINUTE);
```

### 流式数据库中间件：RisingWave

RisingWave 让流处理像使用普通数据库一样简单：

```sql
-- RisingWave：定义流式物化视图

-- 从 Kafka 消费订单流
CREATE SOURCE orders (
    order_id VARCHAR,
    customer_id VARCHAR,
    total NUMERIC,
    order_time TIMESTAMP
) WITH (
    connector = 'kafka',
    topic = 'orders',
    properties.bootstrap.server = 'kafka:9092',
    scan.startup.mode = 'earliest'
) FORMAT PLAIN ENCODE JSON;

-- 实时物化视图：自动增量维护
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT
    customer_id,
    date_trunc('day', order_time) AS day,
    SUM(total) AS revenue,
    COUNT(*) AS order_count
FROM orders
GROUP BY customer_id, date_trunc('day', order_time);

-- 查询物化视图（延迟极低，因为已预计算）
SELECT * FROM daily_revenue WHERE day = '2026-09-19';

-- 复杂 JOIN：流 + 流
CREATE MATERIALIZED VIEW high_value_customers AS
SELECT
    d.customer_id,
    d.revenue,
    c.country
FROM daily_revenue d
JOIN customers c ON d.customer_id = c.id
WHERE d.revenue > 10000;
```

### CDC 中间件：Debezium + Flink CDC

**Flink CDC：无锁全量 + 增量同步：**

```sql
-- Flink CDC 3.0：完整管道定义
-- 源表：MySQL CDC
CREATE TABLE mysql_orders (
    order_id BIGINT,
    customer_id VARCHAR,
    total DECIMAL(10,2),
    status VARCHAR,
    create_time TIMESTAMP(3),
    update_time TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'mysql.db',
    'port' = '3306',
    'username' = 'flink',
    'password' = '***',
    'database-name' = 'ecommerce',
    'table-name' = 'orders',
    'scan.startup.mode' = 'initial',
    'scan.incremental.snapshot.enabled' = 'true',  -- 无锁快照
    'scan.snapshot.fetch.size' = '1024'
);

-- Sink：Iceberg
CREATE TABLE iceberg_orders (
    order_id BIGINT,
    customer_id VARCHAR,
    total DECIMAL(10,2),
    status VARCHAR,
    create_time TIMESTAMP(3),
    update_time TIMESTAMP(3),
    PRIMARY KEY (order_id) NOT ENFORCED
) WITH (
    'connector' = 'iceberg',
    'catalog' = 'default_catalog',
    'warehouse' = 's3://warehouse/',
    'format' = 'parquet',
    'write.format.default' = 'parquet'
);

-- 同步管道
INSERT INTO iceberg_orders
SELECT * FROM mysql_orders;
```

### 消息系统中间件：Pulsar

Pulsar 的存算分离架构使其在云原生场景有独特优势：

```
Pulsar 架构：
  Producer → Broker（无状态，只转发）→ BookKeeper（分布式日志存储）
              ↑                              ↑
         计算层（可弹性伸缩）         存储层（独立扩展）
```

```java
// Pulsar Java 生产者
Producer<String> producer = client.newProducer(Schema.STRING)
    .topic("orders")
    .enableBatching(true)
    .batchingMaxMessages(1000)
    .batchingMaxPublishDelay(10, TimeUnit.MILLISECONDS)
    .compressionType(CompressionType.ZSTD)
    .sendTimeout(10, TimeUnit.SECONDS)
    .create();

// Pulsar 事务
client.newTransaction()
    .withTransactionTimeout(30, TimeUnit.SECONDS)
    .build()
    .thenAccept(txn -> {
        producer.newMessage(txn)
            .value("order-1")
            .send();

        producer.newMessage(txn)
            .value("order-2")
            .send();

        txn.commit();  // 原子提交
    });
```

### 消息系统中间件：Redpanda / WarpStream

**Redpanda——C++ 实现的高性能 Kafka 替代：**

```bash
# Redpanda 单二进制部署，无需 JVM/ZooKeeper
rpk cluster config set kafka_batch_max_bytes 1048576
rpk topic create orders --partitions 6 --replicas 3

# Redpanda 完全兼容 Kafka 协议
# 应用代码无需修改，只需改 bootstrap servers
# kafka:9092 → redpanda:9092
```

**WarpStream——S3 原生 Kafka：**

```bash
# WarpStream：日志直接存储在 S3
# 生产者配置
producer.boostrap.servers=warpstream-agent:9092
producer.client.id=order-producer
producer.acks=all

# Agent 将日志写入 S3 的 WAL 对象
# 消费者从 S3 拉取数据
# 无需本地磁盘，成本降低 10 倍
```

### 流处理 AI 中间件：实时 RAG 管道

```python
# 实时 RAG：CDC → 流处理 → 向量数据库 → LLM
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment

env = StreamExecutionEnvironment.getExecutionEnvironment()
t_env = StreamTableEnvironment.create(env)

# 1. 从 CDC 消费文档变更
t_env.execute_sql("""
    CREATE TABLE document_changes (
        doc_id STRING,
        content STRING,
        operation STRING,
        change_time TIMESTAMP(3)
    ) WITH (
        'connector' = 'mongodb-cdc',
        'hosts' = 'mongo:27017',
        'database' = 'knowledge',
        'collection' = 'documents'
    )
""")

# 2. 调用 Embedding 服务（UDF）
t_env.create_temporary_function(
    "generate_embedding",
    EmbeddingUDF,  # 调用 LLM API 生成向量
    ["STRING"]
)

# 3. 写入向量数据库（实时更新知识库）
t_env.execute_sql("""
    CREATE TABLE vector_store (
        doc_id STRING,
        content STRING,
        embedding ARRAY<FLOAT>,
        change_time TIMESTAMP(3)
    ) WITH (
        'connector' = 'milvus',
        'host' = 'milvus:19530',
        'collection' = 'documents'
    )
""")

t_env.execute_sql("""
    INSERT INTO vector_store
    SELECT doc_id, content, generate_embedding(content), change_time
    FROM document_changes
    WHERE operation IN ('INSERT', 'UPDATE')
""")
```

## 2026 年工业界最新进展

### 流式数据库的兴起

2023-2026 年最重要的新趋势是"流式数据库"——将流处理和数据库融合：

**1. RisingWave：**

- 用 SQL 定义物化视图，系统自动维护流式更新
- 底层基于流处理引擎，支持复杂 JOIN 和窗口
- 存算分离，可弹性伸缩

**2. Materialize：**

- 基于 Differential Dataflow 的流式 SQL 引擎
- 支持 ACID 事务的流式物化视图
- 适合实时分析和数据服务

**3. Apache Flink Table API：**

- Flink 的 SQL 接口，可同时用于流和批
- 支持复杂的事件时间语义和 Watermark
- 2024 年 Flink 2.0 引入了 Adaptive Batch 模式

**范式意义：** 流式数据库让用户像使用普通数据库一样做实时分析——定义视图，查询视图，无需编写流处理代码。

### CDC 驱动的实时数据管道

2024-2026 年，CDC + 流处理成为实时数据同步的标准范式：

```
源数据库 → CDC（Debezium/Flink CDC） → Kafka/流处理 → 目标系统
                                              ↓
                                    Elasticsearch / Redis / 数据仓库
```

**关键进展：**

- **Flink CDC 3.0**：支持 Schema 演化、无锁全量增量切换
- **MongoDB Kafka Connector**：原生支持 MongoDB 变更流
- **Debezium 2.x**：支持增量快照（Incremental Snapshot），无锁全量同步

### Serverless 流处理

- **AWS Kinesis Data Analytics**：Serverless Flink，按实际处理量计费
- **Google Dataflow**：Serverless 流批一体，Auto-scaling
- **Confluent Cloud**：Serverless Kafka + Flink，无需管理集群
- **局限**：冷启动延迟、状态管理复杂，不适合严格低延迟场景

### 流处理与 AI 的结合

2023-2026 年，LLM 和流处理的结合成为热点：

- **实时 RAG**：CDC 将文档变更实时同步到向量数据库，保证 RAG 知识库实时性
- **流式特征工程**：Flink 做 ML 特征的实时计算，推送到在线特征存储（Feast / Tecton）
- **实时推理管道**：Kafka → 流处理 → LLM 推理 → Kafka，实现实时 AI 服务

### 流处理的精确一次（Exactly-Once）

书中提到"精确一次"语义的挑战，2026 年的实践：

- **Kafka Transactions**：Kafka 0.11+ 支持事务性生产者，实现精确一次
- **Flink Checkpointing**：基于 Chandy-Lamport 算法的分布式快照
- **Two-Phase Commit Sink**：Flink + Kafka 的事务性 Sink，端到端精确一次
- **Idempotent Producer**：Kafka 的幂等生产者，避免重试导致重复

## 经典论文

1. **"Kafka: a Distributed Messaging System for Log Processing"** — Kreps et al., 2011
   - Kafka 论文，分区日志的工业实践。

2. **"Discretized Streams: Fault-Tolerant Streaming Computation"** — Zaharia et al., 2013
   - Spark Streaming 论文，微批处理模型。

3. **"Lightweight Asynchronous Snapshots for Distributed Dataflows"** — Carbone et al., 2015
   - Flink Checkpointing 的理论基础，基于 Chandy-Lamport 算法。

4. **"MillWheel: Fault-Tolerant Stream Processing at Internet Scale"** — Akidau et al., 2013
   - Google MillWheel 论文，Dataflow 的基础。

5. **"The Dataflow Model"** — Akidau et al., 2015
   - Google Dataflow 论文，统一了流和批的编程模型，引入事件时间和 Watermark 概念。

6. **"Apache Flink: Stream and Batch Processing in a Single Engine"** — Carbone et al., 2015
   - Flink 论文，流批一体的架构设计。

7. **"Differential Dataflow"** — Murray et al., 2013
   - Materialize 的理论基础，增量计算的形式化框架。

8. **"Google Cloud Pub/Sub: A Global Messaging Service"** — Isard et al., 2007（最初的 Pub/Sub 设计）
   - Google 的分布式消息系统设计。

## 最新论文与进展（2023-2026）

1. **"RisingWave: A Cloud-Native Streaming Database"** — RisingWave Labs / VLDB 2024
   - RisingWave 的流式物化视图架构和存算分离设计。

2. **"Materialize: Differential Dataflow at Scale"** — Materialize Tech Report, 2024
   - Materialize 的增量计算引擎工程实践。

3. **"Apache Flink 2.0: Stream-Batch Unification"** — Apache Flink Community, 2024
   - Flink 2.0 的流批统一架构和自适应执行。

4. **"WarpStream: Kafka on Object Storage"** — Confluent / Kafka Summit 2024
   - WarpStream 的 S3 原生 Kafka 架构。

5. **"Flink CDC 3.0: Schema Evolution and Lock-Free Snapshot"** — Apache Flink Community, 2024
   - Flink CDC 3.0 的无锁全量同步和 Schema 演化。

6. **"Streaming Databases: A New Paradigm"** — ACM Computing Surveys, 2025
   - 流式数据库的系统性综述和对比分析。

7. **"Real-Time RAG: Integrating LLMs with Stream Processing"** — NeurIPS Workshop 2025
   - LLM 与流处理结合的实时 RAG 架构。

8. **"Serverless Stream Processing: State of the Art"** — IEEE Cloud Computing, 2026
   - Serverless 流处理的架构和挑战综述。
