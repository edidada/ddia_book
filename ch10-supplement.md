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

## 工业界中间件软件实践

### 批处理中间件：Apache Spark

Spark 是 2026 年大规模批处理的事实标准，其核心抽象是 RDD/DataFrame：

**Spark DataFrame API（结构化数据处理）：**

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder \
    .appName("BatchProcessing") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .getOrCreate()

# 读取数据（支持 Parquet、Delta、Iceberg 等格式）
orders = spark.read.parquet("s3://data/orders/")
customers = spark.read.parquet("s3://data/customers/")

# Spark SQL 风格：声明式批处理
result = spark.sql("""
    SELECT
        c.country,
        o.order_date,
        COUNT(*) AS order_count,
        SUM(o.total) AS revenue
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    WHERE o.order_date >= '2026-01-01'
    GROUP BY c.country, o.order_date
    ORDER BY revenue DESC
""")

# 写入结果（支持多种格式）
result.write \
    .mode("overwrite") \
    .parquet("s3://data/output/daily_revenue/")

# 写入到数据湖仓（Delta Lake）
result.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("analytics.daily_revenue")
```

**Spark 的核心设计：**

| 抽象 | 说明 | 特点 |
|------|------|------|
| RDD | 弹性分布式数据集 | 不可变、分区、血缘关系 |
| DataFrame | 结构化数据 | 带 Schema 的 RDD，向量化执行 |
| Catalyst Optimizer | 查询优化器 | 逻辑计划→物理计划，RBO+CBO |
| Tungsten | 执行引擎 | 堆外内存、向量化、代码生成 |

### 批处理中间件：Apache Iceberg + Spark

Iceberg 表格式配合 Spark 实现数据湖仓：

```sql
-- Spark SQL：创建 Iceberg 表
CREATE TABLE iceberg.orders (
    order_id STRING,
    customer_id STRING,
    total DECIMAL(10,2),
    status STRING,
    event_time TIMESTAMP
)
USING iceberg
PARTITIONED BY (days(event_time))
STORED AS PARQUET
TBLPROPERTIES (
    'write.format.default' = 'parquet',
    'write.parquet.compression-codec' = 'zstd',
    'write.distribution-mode' = 'hash'
);

-- 批量写入
INSERT INTO iceberg.orders
SELECT * FROM staging.orders;

-- 时间旅行：查询历史版本
SELECT * FROM iceberg.orders.snapshots;
SELECT * FROM iceberg.orders VERSION AS OF 1234567890;

-- 增量读取（CDC 式消费）
SELECT * FROM iceberg.orders
WHERE event_time > '2026-09-19 00:00:00';

-- Schema 演化：安全添加列
ALTER TABLE iceberg.orders ADD COLUMN session_id STRING AFTER status;

-- 分区演化：从日分区改为月分区
ALTER TABLE iceberg.orders REPLACE PARTITION FIELD
    months(event_time) AS month;
```

### 批处理中间件：Trino（联邦查询）

Trino 支持 SQL 查询多种数据源，无需预先 ETL：

```sql
-- Trino：联邦查询 MySQL 订单 + Hive 日志
SELECT
    o.order_id,
    c.customer_name,
    l.page_count,
    o.total
FROM mysql.ecommerce.orders o
JOIN mysql.ecommerce.customers c ON o.customer_id = c.id
LEFT JOIN (
    SELECT customer_id, count(*) AS page_count
    FROM hive.web_logs.page_views
    WHERE dt = '2026-09-19'
    GROUP BY customer_id
) l ON l.customer_id = c.id
WHERE o.order_date = DATE '2026-09-19'
ORDER BY o.total DESC
LIMIT 100;
```

```sql
-- Trino 配置数据源连接器（catalog）
-- catalog/mysql.properties
connector.name=mysql
connection-url=jdbc:mysql://mysql.db:3306
connection-user=trino
connection-password=***

-- catalog/hive.properties
connector.name=hive
hive.metastore=thrift
hive.metastore.uri=thrift://hive-metastore:9083
hive.s3.endpoint=https://s3.amazonaws.com
hive.s3.aws-access-key=***
hive.s3.aws-secret-key=***
```

### 批处理中间件：DuckDB

DuckDB 是嵌入式分析数据库，无需部署，适合"最后一公里"分析：

```python
import duckdb

# 直接查询 Parquet 文件（无需导入）
result = duckdb.sql("""
    SELECT
        order_date,
        count(*) AS orders,
        sum(total) AS revenue,
        avg(total) AS avg_order_value
    FROM read_parquet('orders/*.parquet')
    WHERE order_date >= '2026-01-01'
    GROUP BY order_date
    ORDER BY order_date
""").fetchdf()

# 直接查询 CSV
csv_result = duckdb.sql("""
    SELECT * FROM 'logs/access_log_*.csv'
    WHERE status = 200
""").fetchall()

# 跨格式查询
mixed = duckdb.sql("""
    SELECT o.order_id, c.customer_name
    FROM read_parquet('orders.parquet') o
    JOIN read_csv('customers.csv') c ON o.customer_id = c.id
""").df()
```

### 数据管道中间件：Apache Airflow

Airflow 是批处理管道编排的事实标准：

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'daily_revenue_pipeline',
    default_args=default_args,
    schedule_interval='0 2 * * *',  # 每天2点
    start_date=datetime(2026, 1, 1),
    catchup=False,
)

# 1. 从 Kafka 同步数据到 Iceberg
sync_task = SparkSubmitOperator(
    task_id='sync_kafka_to_iceberg',
    application='/jobs/sync_kafka.py',
    conn_id='spark_cluster',
    dag=dag,
)

# 2. 计算每日收入
compute_task = SparkSubmitOperator(
    task_id='compute_daily_revenue',
    application='/jobs/compute_revenue.py',
    conn_id='spark_cluster',
    dag=dag,
)

# 3. 加载到 Snowflake 数据仓库
load_task = SnowflakeOperator(
    task_id='load_to_snowflake',
    sql="""
        COPY INTO analytics.daily_revenue
        FROM @iceberg_stage/daily_revenue/
        FILE_FORMAT = (TYPE = PARQUET);
    """,
    snowflake_conn_id='snowflake_prod',
    dag=dag,
)

# 4. 通知下游
notify_task = BashOperator(
    task_id='notify_downstream',
    bash_command='curl -X POST https://api.example.com/notify',
    dag=dag,
)

# 依赖关系
sync_task >> compute_task >> load_task >> notify_task
```

### 数据转换中间件：dbt

dbt 将数据管道的定义从命令式脚本转变为声明式 SQL：

```sql
-- dbt 模型：每日收入分析
{{ config(materialized='incremental', unique_key='order_date') }}

SELECT
    order_date,
    count(*) AS order_count,
    sum(total) AS revenue,
    avg(total) AS avg_order_value
FROM {{ ref('stg_orders') }}
WHERE status = 'PAID'

{% if is_incremental() %}
    AND order_date > (select max(order_date) from {{ this }})
{% endif %}

GROUP BY order_date
```

```yaml
# dbt 项目配置
# dbt_project.yml
models:
  ecommerce:
    staging:
      +materialized: view
      +schema: staging
    marts:
      +materialized: incremental
      +schema: analytics
      +cluster_by: ['order_date']
```

### 向量化中间件：Polars

Polars（Rust）是 DataFrame 领域的性能标杆：

```python
import polars as pl

# Polars 惰性执行（查询优化）
lf = pl.scan_parquet("orders/*.parquet")

result = (
    lf.filter(pl.col("order_date") >= "2026-01-01")
      .groupby("customer_id")
      .agg([
          pl.col("total").sum().alias("total_spent"),
          pl.col("order_id").count().alias("order_count"),
          pl.col("total").mean().alias("avg_order_value")
      ])
      .filter(pl.col("total_spent") > 1000)
      .sort("total_spent", descending=True)
      .collect()  # 触发执行
)

# Polars 比 Pandas 快 5-50 倍，原因：
# 1. Rust 实现，无 GIL
# 2. 多线程并行
# 3. 向量化执行（SIMD）
# 4. 惰性求值 + 查询优化
```

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
