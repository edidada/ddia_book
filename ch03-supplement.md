# 第三章补充：概念深化与前沿进展

> 本章为 DDIA 第三章「存储与查询」的补充材料，涵盖核心概念深入解释、2026 年工业界最新进展，以及经典论文与最新论文推荐。

## 核心概念深化

### LSM-Tree vs B-Tree 的持续竞争

书中对两种存储引擎流派做了经典对比。到 2026 年，两者的竞争和融合仍在持续：

**LSM-Tree 的优势与劣势：**

- 优势：顺序写入性能优异、空间利用率高、适合写密集场景
- 劣势：读放大（Read Amplification）、写放大（Write Amplification）、空间放大（Space Amplification）的"三放大"问题
- 2026 年优化：RocksDB 的 Tiered Compaction、Tiered+Leveled 混合压缩策略、BlobDB（值分离存储）显著降低了写放大

**B-Tree 的新发展：**

- MySQL 的 InnoDB 在 8.x 版本引入了更高效的 Change Buffer 和 Adaptive Hash Index
- PostgreSQL 17 引入了更先进的 vacuum 策略和更高效的 B-Tree 变体
- Bw-Tree（Bw-Tree，Buzz Word Tree）在 SQL Server Hekaton（内存 OLTP）中实现，结合了 B-Tree 的查询效率和 LSM 的无锁更新

**融合趋势：**

- **RocksDB 作为存储引擎底座**：TiDB、CockroachDB、MongoDB WiredTiger 等系统都支持以 RocksDB（LSM-Tree）作为底层存储引擎，但上层提供 SQL 接口——存储引擎和数据库解耦。
- **DAXIO/Pebble**：CockroachDB 团队开发的 Pebble（RocksDB 的 Go 语言重新实现），针对分布式数据库场景做了深度优化。

### 列式存储的全面普及

2026 年，列式存储已从数据仓库扩展到更广泛场景：

**存储格式标准化：**

- **Apache Parquet**：已成为大数据生态的列式存储事实标准。2023-2026 年，Parquet 的向后兼容性和 schema evolution 能力持续增强，ZSTD 压缩成为默认选择。
- **Apache Arrow**：内存中的列式格式，实现了"零拷贝"跨进程数据交换。Arrow Flight 升级到 v2，支持更高吞吐的数据传输。
- **Apache Iceberg / Delta Lake / Hudi**：三者构成的"表格式"（Table Format）生态，将列式存储与事务管理结合，支持 ACID、时间旅行（Time Travel）、Schema 演化。

**向量化执行（Vectorized Execution）：**

不同于 SIMD 向量化，向量化执行引擎是指"一次处理一批数据（batch）而非一行数据"：

- ClickHouse 的向量化执行引擎是其极致性能的核心原因
- DuckDB 的向量化引擎使嵌入式分析达到服务端性能
- Apache DataFusion（Rust）提供了可嵌入的向量化执行框架

### 内存数据库与持久化内存

- **Redis 的演进**：Redis 在 2023 年改变了开源协议（RSALv2/SSPL），引发了 Lavabit（Valkey）分叉。到 2026 年，Valkey 已成为社区主流的 Redis 替代品。
- **DRAM 仍然是主流**：Intel Optane 持久化内存（PMem）在 2022 年宣布停产，验证了"持久化内存"作为过渡技术的命运。2026 年，CXL（Compute Express Link）成为新的内存扩展方向。

## 2026 年工业界最新进展

### 闪存存储的革命

- **NVMe SSD 成为标配**：PCIe 5.0 NVMe SSD 的顺序读取带宽可达 14 GB/s，随机 IOPS 可达 200 万+。这使得"磁盘寻道"的瓶颈进一步减弱，存储引擎设计的前提发生变化。
- **ZNS（Zoned Namespace）SSD**：将 SSD 的物理区域暴露给主机，让存储引擎直接管理写入位置，与 LSM-Tree 的顺序写入理念天然契合。Western Digital、Samsung 已量产 ZNS SSD。
- **KVSSD（Key-Value SSD）**：将 KV 接口直接下推到 SSD 固件层，绕过文件系统开销。Samsung 的 KV SSD 已有产品出货。

### 对象存储作为数据库后端

2024-2026 年最重要的存储趋势之一是"对象存储优先"（Object Storage First）：

- **Snowflake、Databricks** 的存储计算分离架构天然适合对象存储
- **Apache Iceberg on S3** 已成为数据湖仓的标准范式
- **WarpStream**（被 Confluent 收购）：将 Kafka 的日志存储直接放在 S3 上，无需本地磁盘
- **Turso（libSQL）**、Neon：将 SQLite/PostgreSQL 的页面存储放到对象存储上，实现极低成本的 serverless 数据库

这一趋势的核心驱动力是：对象存储（S3等）的成本比本地 SSD 低 10-50 倍，且具有 11 个 9 的持久性。

### 计算存储分离（Disaggregated Storage）

主流分布式数据库已普遍采用存储计算分离架构：

| 系统 | 计算层 | 存储层 | 特点 |
|------|-------|-------|------|
| Aurora | MySQL/PG 引擎 | 分布式存储层 | 日志即数据库（Log is Database）|
| TiDB | TiDB SQL 层 | TiKV + TiFlash | 行列混合 |
| Snowflake | 弹性计算 | 云对象存储 | 多集群共享存储 |
| Neon | Postgres 计算层 | Pageserver | 分页级存储分离 |
| CockroachDB | SQL + KV 层 | Pebble（本地） | 可选分离模式 |

### 写优化 SSD 与存储引擎协同

一种新兴的研究方向是让存储引擎感知底层闪存特性：

- **FTL 感知**：绕过 SSD 的 FTL（Flash Translation Layer），直接管理物理页
- **Garbage Collection 感知**：将 LSM-Tree 的 compaction 与 SSD 的 GC 协同调度，减少写放大
- **Open-Channel SSD**：将原始闪存空间暴露给上层，存储引擎全权管理

## 经典论文

1. **"The Log-Structured Merge-Tree (LSM-Tree)"** — O'Neil et al., 1996
   - LSM-Tree 的奠基论文。

2. **"Efficient Locking for Concurrency Operations on B-Trees"** — Lehman & Yao, 1981
   - B-Link 树，B-Tree 的并发优化经典。

3. **"The Five-Minute Rule Ten Years Later"** — Gray & Graefe, 2007
   - 磓盘与内存访问成本的经典权衡框架，对存储引擎设计有重要指导意义。

4. **"Column-Stores vs. Row-Stores: How Different Are They Really?"** — Abadi et al., 2008
   - 列存与行存的系统性对比，阐述列存优势的本质来源。

5. **"C-Store: A Column-oriented DBMS"** — Stonebraker et al., 2005
   - 列式数据库的里程碑论文，直接影响了 Vertica、Redshift 等。

6. **"Dremel: Interactive Analysis of Web-Scale Datasets"** — Melnik et al., 2010
   - Google Dremel，嵌套数据列式存储的开创性工作。

## 最新论文与进展（2023-2026）

1. **"Pebble: A Scalable Log-Structured Key-Value Store"** — CockroachDB Tech Report, 2023
   - Pebble 存储引擎的设计与 RocksDB 对比分析。

2. **"Neon: Serverless Postgres with Disaggregated Storage"** — VLDB 2024
   - Neon 的存储计算分离架构和页面级日志设计。

3. **"Apache Iceberg: A Table Format for Large Analytic Workloads"** — SIGMOD 2024
   - Iceberg 表格式的设计原理和工业应用。

4. **"WarpStream: Kafka on Object Storage"** — Kafka Summit 2024
   - 将 Kafka 日志直接存储在 S3 上的架构和性能分析。

5. **"ZNS SSD: Enabling Storage Engine Co-design"** — FAST 2024
   - ZNS SSD 与 LSM-Tree 协同设计的性能收益。

6. **"DuckDB: An Embedded Analytical Database"** — Journal of Database Management, 2025
   - DuckDB 嵌入式列存引擎的完整设计论文。

7. **"Valkey: Lessons from Forking Redis"** — USENIX ATC 2025
   - Valkey 分叉 Redis 的工程经验和存储引擎优化。
