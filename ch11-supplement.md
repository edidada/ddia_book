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
