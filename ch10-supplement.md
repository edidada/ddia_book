# 第十章补充：概念深化与前沿进展

> 本章为 DDIA 第十章「批处理（Batch Processing）」的补充材料，涵盖核心概念深入解释、2026 年工业界最新进展，以及经典论文与最新论文推荐。

## 核心概念深化

### MapReduce 的退潮与遗产

书中以 MapReduce 为核心展开讨论。到 2026 年，MapReduce 已基本退潮，但其思想遗产深远：

**MapReduce 的局限：**

- 多次中间结果落盘，I/O 开销大
- 编程模型过于底层，开发效率低
- 任务调度开销大，不适合短任务
- 不支持迭代计算，机器学习场景不友好

**遗产：**

- 数据本地性（Data Locality）原则：计算向数据靠拢
- 容错重试：任务失败自动重试
- 逻辑与物理分离：声明式查询 + 物理执行计划

### 新一代批处理引擎

**1. Spark：**

- 基于内存的 RDD（Resilient Distributed Dataset），中间结果可缓存
- 2023-2026 年，Spark 4.0 引入了 Photon（C++ 向量化执行引擎）和 Scala 3 支持
- 仍是大规模批处理的事实标准

**2. Trino（原 PrestoSQL）：**

- 专注交互式分析查询，无独立存储层
- 联邦查询：可同时查询 Hive、MySQL、Kafka、S3 等多种数据源
- 2024 年 Trino 上的 Starburst Galaxy 成为云原生数据仓库的主流选择

**3. DuckDB：**

- 嵌入式分析数据库，无需集群
- 单机即可处理 TB 级数据
- 适合数据分析的"最后一公里"

**4. Daft / DataFusion：**

- Rust 生态的分布式数据框架
- Daft：分布式 DataFrame，AI/ML 工作流
- Apache DataFusion：可嵌入的向量化 SQL 引擎

### 批处理的 Unix 哲学

书中将 MapReduce 与 Unix 管道做了类比。2026 年，这种"函数式组合"思想仍在延续：

- **Data Pipeline as Code**：用代码而非配置定义数据处理流水线
- **DAG（有向无环图）**：所有现代批处理引擎（Spark、Flink、Airflow）都以 DAG 表达计算逻辑
- **不可变输入**：函数式编程的纯粹性——输入数据不可变，输出是新文件

## 2026 年工业界最新进展

### 数据湖仓（Data Lakehouse）

2023-2026 年最重要的数据架构变革是"数据湖仓"——在数据湖上实现数据仓库的能力：

| 表格式 | 提出/维护方 | 特点 |
|--------|-----------|------|
| Apache Iceberg | Netflix → Apache | 快照隔离、Schema 演化、分区演化 |
| Delta Lake | Databricks | ACID 事务、时间旅行、Z-Ordering |
| Apache Hudi | Uber → Apache | 增量处理、Upsert、CDC |
| Apache Paimi（原 Incubating） | Apache | 流批一体 |

**核心能力：**

- ACID 事务：在对象存储（S3）上实现数据一致性
- Schema 演化：增加/删除/重命名列，不中断查询
- 时间旅行：查询历史版本数据
- 增量处理：支持 CDC 式增量读取

**为什么重要：** 数据湖仓打破了"数据湖（廉价但不一致）vs 数据仓库（昂贵但可靠）"的二分法，使同一份数据既可做 BI 分析，又可做 ML 训练。

### 向量化与 GPU 加速

- **Photon（Databricks）**：C++ 重写的 Spark 执行引擎，性能提升 5-10 倍
- **RAPIDS（NVIDIA）**：GPU 加速的数据处理库，可与 Spark 集成
- **Polars（Rust）**：DataFrame 库，利用多线程和 SIMD 实现极致性能
- **GPU for ML Pipeline**：数据预处理（ETL）环节开始利用 GPU 加速

### 声明式数据管道

2024-2026 年，数据管道的定义从命令式脚本向声明式转变：

- **dbt（Data Build Tool）**：用 SQL 定义数据转换，自动管理依赖和增量
- **SQLGlot**：Python 库，可在不同 SQL 方言间转换
- ** declarative pipeline**：用 YAML/SQL 声明"我想要什么数据"，系统决定"如何计算"

### 数据网格（Data Mesh）的落地

书中最后一节提到数据集成中的"合并分析"和"分而治之"。Data Mesh 将这一思想推到组织层面：

**核心原则（Zhamak Dehghani, 2020）：**

1. **领域驱动的数据所有权**：每个业务团队管理自己的数据产品
2. **数据即产品**：数据有明确的所有者、SLA、Schema
3. **自助式数据基础设施**：平台团队提供数据工具
4. **联邦计算治理**：统一标准，分散执行

**2026 年实践：**

- Data Mesh 概念逐渐与 Data Lakehouse 融合
- Snowflake / Databricks 提供了 Data Mesh 友好的多团队协作能力
- 但真正落地 Data Mesh 的组织仍属少数——它更多是一种组织变革而非技术选型

## 经典论文

1. **"MapReduce: Simplified Data Processing on Large Clusters"** — Dean & Ghemawat, 2004
   - MapReduce 论文，开启了大规模数据处理的民主化。

2. **"The Google File System"** — Ghemawat et al., 2003
   - GFS 论文，为 MapReduce 提供了存储基础设施。

3. **"Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing"** — Zaharia et al., 2012
   - Spark RDD 论文，开启内存计算时代。

4. **"Shark: SQL and Rich Analytics at Scale"** — Xin et al., 2013
   - Spark SQL 的前身，在 Spark 上实现 SQL 查询。

5. **"Presto: Distributed SQL Query Engine"** — Setrakyan et al., 2014
   - Presto/Trino 的设计理念。

6. **"Dremel: Interactive Analysis of Web-Scale Datasets"** — Melnik et al., 2010
   - Google Dremel，列式存储 + 多级服务树的交互式分析。

7. **"Data Mesh: A New Paradigm for Data Management"** — Dehghani, 2020（书）
   - Data Mesh 概念的提出。

8. **"Apache Iceberg: A Table Format for Big Data"** — Netflix Tech Blog, 2019
   - Iceberg 表格式的早期设计。

## 最新论文与进展（2023-2026）

1. **"Apache Iceberg: The Definitive Guide"** — O'Reilly / Apache, 2024
   - Iceberg 的完整设计和工程实践。

2. **"Photon: A Vectorized Query Execution Engine"** — Databricks / VLDB 2024
   - Databricks Photon 的 C++ 向量化引擎设计。

3. **"Delta Lake: High-Performance ACID Table Storage"** — Databricks Tech Report, 2024
   - Delta Lake 的 ACID 事务和时间旅行实现。

4. **"DuckDB: An Embedded Analytical Database"** — Journal of Database Management, 2025
   - DuckDB 的设计和性能分析。

5. **"Polars: A High-Performance DataFrame Library"** — Ritchie & Vink, 2025
   - Polars 的多线程和 SIMD 优化策略。

6. **"Data Lakehouse Architecture: A Survey"** — ACM Computing Surveys, 2025
   - 数据湖仓架构的系统性综述。

7. **"dbt: Declarative Data Transformation"** — dbt Labs, 2024
   - dbt 的设计理念和在数据管道中的应用。

8. **"Data Mesh in Practice: Lessons from Industry"** — IEEE Data Eng. Bulletin, 2026
   - Data Mesh 在大型组织的落地经验。
