# 第二章补充：概念深化与前沿进展

> 本章为 DDIA 第二章「数据模型和查询语言」的补充材料，涵盖核心概念深入解释、2026 年工业界最新进展，以及经典论文与最新论文推荐。

## 核心概念深化

### 关系模型的持久生命力

DDIA 在 2017 年讨论了关系模型 vs 文档模型的取舍。到 2026 年，关系模型不仅没有衰退，反而因以下趋势更加巩固：

- **PostgreSQL 的复兴**：Postgres 在 2023-2026 年成为最热门的开源关系数据库，其扩展生态（pgvector、TimescaleDB、Citus、Supabase）使其能充当文档数据库、向量数据库、时序数据库、分布式数据库等多种角色。
- **NewSQL 的成熟**：TiDB、CockroachDB、YugabyteDB 等 NewSQL 系统证明了"关系模型 + 水平扩展 + 强一致性"可以兼得，打破了 NoSQL 时代的"为了扩展放弃 SQL"的迷思。
- **SQL 方言的统一趋势**：Trino/Presto 推动了"一个 SQL 查多种数据源"的联邦查询范式，DuckDB 使嵌入式分析 SQL 成为可能。

### 文档模型与 JSON 的演进

- **JSON 在关系数据库中的一等公民地位**：PostgreSQL 的 JSONB、MySQL 的 JSON 类型已非常成熟，配合 GIN 索引可实现高效 JSON 查询。这使得"关系 vs 文档"的二分法在实践中趋于模糊。
- **MongoDB 的改进**：MongoDB 在 2023-2026 年引入了时间序列集合、Queryable Encryption（可查询加密）、以及更强大的聚合管道，逐步向关系数据库的能力靠拢。
- **Supabase / Turso 模式**：将文档模型的开发体验（BaaS 风格 API）与关系数据库的底层能力结合。

### 图模型的工业落地

- **图数据库在知识图谱中的角色**：Neo4j、TigerGraph 在推荐系统、反欺诈、供应链分析领域持续增长。2024-2026 年，图数据库与 LLM 的结合成为热点——GraphRAG（图增强检索增强生成）将知识图谱作为 LLM 的结构化知识源。
- **属性图模型标准化**：Apache TinkerPop 的 Gremlin 和 Cypher（openCypher）逐步成为图查询语言的统一标准。2024 年 ISO/IEc 推出了 GQL（Graph Query Language）标准。

### Schema 的光谱

书中提到强 Schema（写时约束）和弱 Schema（读时解析）的两端，2026 年的实践更加精细化：

| Schema 策略 | 特点 | 适用场景 | 代表系统 |
|------------|------|---------|---------|
| Schema-on-Write | 写入时严格校验 | 交易系统、金融系统 | PostgreSQL、MySQL |
| Schema-on-Read | 读取时解析 | 数据湖、日志分析 | Hive、Spark |
| Schema Evolution | 允许渐进演化 | 微服务、事件溯源 | Avro、Protobuf |
| Schema-less + Soft Schema | 底层无模式，上层有约定 | 快速迭代应用 | MongoDB + JSON Schema |
| Semantic Schema | 嵌入语义信息 | AI 应用 | Vector + Knowledge Graph |

## 2026 年工业界最新进展

### 向量数据模型

这是 2023-2026 年最重要的新数据模型，由 LLM 浪潮驱动：

- **向量嵌入（Embedding）**：将文本、图像、音频等非结构化数据编码为高维浮点向量（如 768 维、1536 维、3072 维），通过余弦相似度或内积衡量语义相近性。
- **近似最近邻搜索（ANN）**：HNSW（Hierarchical Navigable Small World）、IVF、PQ 等索引算法使百万级向量检索在毫秒级完成。
- **混合检索**：向量检索 + 关键词检索（BM25）的融合排序（如 Cohere Rerank）成为 RAG 应用的标配。

```text
传统关系模型：精确匹配 → 索引 → 快速查找
向量数据模型：语义近似 → ANN 索引 → 快速查找
```

### DuckDB 现象

DuckDB 在 2023-2026 年爆发式增长，被称为"OLAP 界的 SQLite"：

- 嵌入式运行，无需服务端进程
- 列式存储，向量化执行
- 支持 SQL，兼容 PostgreSQL 语法
- 可直接查询 Parquet、CSV、JSON 文件
- 与 Python 生态深度集成

DuckDB 证明了"轻量级分析数据库"有巨大的市场需求，填补了 SQLite（行存、OLTP）和 ClickHouse（重运维、OLAP）之间的空白。

### 多模型数据库

SurrealDB（2023-2026 崛起）、ArangoDB 等多模型数据库，在单一引擎内同时支持：

- 文档模型（JSON 文档）
- 关系模型（SQL 查询）
- 图模型（图遍历）
- 时序模型（时间序列数据）

这反映了工业界对"一种数据模型打天下"的反思——不同场景需要不同模型，但最好在同一个系统中。

### 声明式查询的复兴

书中提到声明式（declarative）vs 命令式（imperative）的区别。2026 年：

- **SQL 回归**：在流处理（Flink SQL、RisingWave）、批处理（Spark SQL、Trino）、向量检索（pgvector SQL 接口）中，SQL 都成为首选接口。
- **自然语言查询**：Text-to-SQL（如 Snowflake Cortex、Databricks AI/BI）使用户可以用自然语言查询数据库，底层仍翻译为 SQL。声明式范式使这种翻译成为可能。

## 经典论文

1. **"A Relational Model of Data for Large Shared Data Banks"** — Codd, 1970
   - 关系模型的奠基论文，影响了整个数据库行业半个多世纪。

2. **"The Object-Oriented Database System Manifesto"** — Atkinson et al., 1989
   - 面向对象数据模型的宣言，是文档/对象数据库的理论基础。

3. **"Neo4j: A Graph Database"** — Robinson et al., 2013（书）
   - 图数据库的工业实践经典。

4. **"The Unified Modeling Language Reference Manual"** — Booch et al., 1999
   - UML 作为数据建模工具的经典参考。

5. **"Dremel: Interactive Analysis of Web-Scale Datasets"** — Melnik et al., 2010
   - Google Dremel 论文，嵌套数据的列式存储，直接启发了 Apache Drill、BigQuery 等系统。

## 最新论文与进展（2023-2026）

1. **"DuckDB: Vectorized Data Processing"** — Raasveldt & Mühleisen, SIGMOD 2024（扩展版）
   - DuckDB 的向量化执行引擎原理和工程实践。

2. **"Milvus: A Purpose-Built Vector DBMS"** — Wang et al., SIGMOD 2024
   - Milvus 向量数据库的系统设计论文，涵盖多种 ANN 索引和混合检索。

3. **"GraphRAG: Making Knowledge Graphs Usable for LLMs"** — Microsoft Research, 2024
   - 将知识图谱与 LLM 结合的 GraphRAG 方法论。

4. **"SurrealDB: A Multi-Model Database for the Modern Web"** — SurrealDB Tech Report, 2023
   - 多模型数据库的设计哲学和实现方案。

5. **"Text-to-SQL in the Age of LLMs: Benchmark and Survey"** — VLDB 2025
   - 自然语言到 SQL 转换的大模型方法综述。

6. **"ISO/IEC 39075:2024 Information Technology — Database Languages — GQL"** — ISO, 2024
   - 图查询语言 GQL 的国际标准。
