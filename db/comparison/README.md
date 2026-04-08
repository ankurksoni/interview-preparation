# DynamoDB vs Redshift vs Redis vs PostgreSQL vs Kafka — 100-Parameter Comparison

A comprehensive **side-by-side comparison** of five widely used data technologies across 100 real-world parameters. Use this as a quick-reference decision guide.

---

## Legend

- **DynamoDB** — AWS managed NoSQL (key-value + document)
- **Redshift** — AWS managed columnar data warehouse
- **Redis** — In-memory data structure store / cache
- **PostgreSQL** — Open-source relational database
- **Kafka** — Distributed event streaming platform

---

## 1. Core Identity (Parameters 1–10)

| #   | Parameter              | DynamoDB                         | Redshift                             | Redis                                 | PostgreSQL                          | Kafka                             |
| --- | ---------------------- | -------------------------------- | ------------------------------------ | ------------------------------------- | ----------------------------------- | --------------------------------- |
| 1   | **Category**           | NoSQL database                   | Data warehouse (OLAP)                | In-memory store / cache               | Relational database (OLTP)          | Event streaming platform          |
| 2   | **Data model**         | Key-value + document             | Columnar relational                  | Key → data structure                  | Relational (tables + JSON)          | Append-only log (topics)          |
| 3   | **Primary use case**   | Low-latency CRUD at scale        | Analytical queries on large datasets | Caching, sessions, real-time counters | Transactional apps, complex queries | Real-time event pipelines         |
| 4   | **Query language**     | PartiQL / API (GetItem, Query)   | SQL (Redshift SQL)                   | Commands (GET, SET, ZADD...)          | SQL (most standards-compliant)      | None (produce/consume API)        |
| 5   | **Schema**             | Schema-less (key-only schema)    | Schema-on-write (DDL required)       | Schema-less                           | Schema-on-write (DDL required)      | Schema-optional (Schema Registry) |
| 6   | **Managed by AWS**     | ✅ Fully managed                 | ✅ Fully managed                     | ✅ ElastiCache / MemoryDB             | ✅ RDS / Aurora                     | ✅ MSK (Managed Streaming)        |
| 7   | **Self-hosted option** | ❌ (DynamoDB Local for dev only) | ❌ AWS only                          | ✅ Open-source                        | ✅ Open-source                      | ✅ Open-source                    |
| 8   | **Open source**        | ❌                               | ❌                                   | ✅ (BSD license)                      | ✅ (PostgreSQL license)             | ✅ (Apache 2.0)                   |
| 9   | **First released**     | 2012                             | 2013                                 | 2009                                  | 1996                                | 2011                              |
| 10  | **Best described as**  | "Serverless NoSQL"               | "Petabyte-scale analytics"           | "Sub-millisecond data store"          | "The world's most advanced RDBMS"   | "Distributed commit log"          |

---

## 2. Data & Storage (Parameters 11–25)

| #   | Parameter                   | DynamoDB                                   | Redshift                                        | Redis                                                 | PostgreSQL                                           | Kafka                                    |
| --- | --------------------------- | ------------------------------------------ | ----------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------- |
| 11  | **Storage location**        | SSD (AWS-managed)                          | SSD/HDD (managed nodes)                         | RAM (primary), disk (persistence)                     | Disk (SSD/HDD)                                       | Disk (broker nodes)                      |
| 12  | **Max item/row size**       | 400 KB                                     | Varies (max VARCHAR 64 KB)                      | 512 MB per value                                      | 1.6 TB per row (theoretical)                         | 1 MB default per message (configurable)  |
| 13  | **Max database/table size** | Unlimited                                  | 8 PB compressed (ra3.16xlarge)                  | Limited by RAM                                        | 32 TB per table (practical)                          | Unlimited (disk-bound)                   |
| 14  | **Data compression**        | Automatic (internal)                       | Automatic columnar (AZ64, LZO, ZSTD)            | No (data is in memory)                                | TOAST for large values                               | Producer-side (LZ4, ZSTD, Snappy, GZIP)  |
| 15  | **Column/field types**      | S, N, B, SS, NS, BS, L, M, BOOL, NULL      | Standard SQL types                              | Strings, Hashes, Lists, Sets, Sorted Sets, Streams... | 40+ types including JSON, arrays, ranges, geospatial | Bytes (schema via Schema Registry)       |
| 16  | **Nested/document data**    | ✅ Maps and Lists (nested up to 32 levels) | ❌ Flat tables (SUPER type for semi-structured) | ✅ Hashes, JSON (with RedisJSON)                      | ✅ JSONB (indexable)                                 | ✅ Any format (JSON, Avro, Protobuf)     |
| 17  | **Secondary indexes**       | GSI (max 20), LSI (max 5)                  | Sort keys, DIST keys (no traditional indexes)   | ❌ No indexes (hash-based lookup)                     | B-tree, GIN, GiST, BRIN, Hash, SP-GiST               | ❌ No indexes (offset-based)             |
| 18  | **Full-text search**        | ❌ (use OpenSearch)                        | Basic (LIKE, SIMILAR TO)                        | ❌ (use RediSearch module)                            | ✅ Built-in (tsvector/tsquery + GIN)                 | ❌ Not applicable                        |
| 19  | **Geospatial**              | ❌                                         | ✅ Geometry type (limited)                      | ✅ GEO commands (GEOADD, GEORADIUS)                   | ✅ PostGIS extension (industry standard)             | ❌ Not applicable                        |
| 20  | **Time-series support**     | ❌ (model with sort key)                   | ✅ Good for time-series analytics               | ✅ Sorted Sets, Streams with TTL                      | ✅ TimescaleDB extension                             | ✅ Native (topic as time-ordered log)    |
| 21  | **Data retention**          | Unlimited (TTL for auto-delete)            | Unlimited                                       | TTL per key, eviction policies                        | Unlimited (manual management)                        | Configurable (time or size based)        |
| 22  | **Versioning**              | ❌ (implement with sort key)               | ❌                                              | ❌                                                    | ❌ (use triggers or audit tables)                    | ✅ Immutable log (all versions retained) |
| 23  | **ACID transactions**       | ✅ (TransactWriteItems, up to 100 items)   | ✅ (Serializable isolation)                     | ✅ MULTI/EXEC (atomic, no rollback)                   | ✅ Full ACID with MVCC                               | ✅ Transactions across partitions        |
| 24  | **NULL handling**           | Attributes can be absent (no NULL concept) | SQL NULLs                                       | Key exists or doesn't                                 | SQL NULLs                                            | N/A (empty fields are application-level) |
| 25  | **Data durability**         | 3+ AZ replication (automatic)              | 3 copies across cluster                         | Configurable (RDB/AOF/none)                           | WAL + streaming replication                          | Configurable (replication factor)        |

---

## 3. Performance (Parameters 26–40)

| #   | Parameter                      | DynamoDB                                            | Redshift                                            | Redis                                     | PostgreSQL                                    | Kafka                              |
| --- | ------------------------------ | --------------------------------------------------- | --------------------------------------------------- | ----------------------------------------- | --------------------------------------------- | ---------------------------------- |
| 26  | **Read latency (typical)**     | Single-digit ms                                     | Seconds to minutes                                  | Sub-millisecond                           | 1–100 ms (depends on query)                   | N/A (consumer pull latency: ms)    |
| 27  | **Write latency (typical)**    | Single-digit ms                                     | Seconds (batch-optimized)                           | Sub-millisecond                           | 1–50 ms                                       | 2–10 ms (acks=all)                 |
| 28  | **Throughput model**           | Provisioned (RCU/WCU) or on-demand                  | Concurrency scaling / node slots                    | 100K+ ops/sec per node                    | Thousands of TPS (connection limited)         | Millions of msgs/sec per cluster   |
| 29  | **Query complexity**           | Simple key lookups, scans, filters                  | Complex SQL (joins, aggregations, window functions) | Single-key or multi-key commands          | Full SQL (CTEs, window functions, subqueries) | N/A (no query engine)              |
| 30  | **Aggregations**               | ❌ (client-side only)                               | ✅ Optimized (columnar storage)                     | ❌ Limited (Lua scripts)                  | ✅ Full SQL aggregations                      | ✅ Kafka Streams / ksqlDB          |
| 31  | **JOINs**                      | ❌ No joins (denormalize)                           | ✅ Distributed joins                                | ❌ No joins                               | ✅ Full JOIN support                          | ❌ (join via Kafka Streams)        |
| 32  | **Batch operations**           | BatchWriteItem (25 items), BatchGetItem (100 items) | COPY command (bulk)                                 | Pipeline (batched commands)               | COPY, multi-row INSERT                        | Batch produce/consume              |
| 33  | **Connection model**           | HTTP API (stateless)                                | TCP (JDBC/ODBC, limited connections)                | TCP (persistent connections)              | TCP (process per connection)                  | TCP (persistent)                   |
| 34  | **Connection pooling needed?** | ❌ (HTTP, no connections)                           | ✅ Yes (WLM queues)                                 | ✅ Recommended at scale                   | ✅ Yes (PgBouncer)                            | ❌ (client manages internally)     |
| 35  | **Caching layer**              | DAX (in-memory, microsecond)                        | Result caching                                      | IS the cache                              | Shared buffers + OS cache                     | Page cache (OS filesystem)         |
| 36  | **Hot partition/key problem**  | ✅ Yes (uneven key distribution)                    | ✅ Yes (skewed distribution keys)                   | ✅ Yes (hot keys)                         | ❌ Rare (index-based access)                  | ✅ Yes (hot partitions)            |
| 37  | **Scan performance**           | Slow (reads every item, 1 MB pages)                 | Fast (columnar, parallel)                           | Keys scan with SCAN cursor (non-blocking) | Seq scan with parallel workers                | Fast sequential read (from offset) |
| 38  | **Concurrent users**           | Unlimited (HTTP)                                    | 500 connections (default)                           | 10,000+ connections                       | 100–500 (process per connection)              | Thousands (per consumer group)     |
| 39  | **Auto-scaling**               | ✅ On-demand mode (automatic)                       | ✅ Concurrency scaling, Serverless                  | ✅ ElastiCache auto-scaling               | ✅ Aurora auto-scaling (read replicas)        | ✅ MSK auto-scaling (storage)      |
| 40  | **Compute-storage separation** | ✅ Yes (inherent)                                   | ✅ Yes (RA3 nodes + managed storage)                | ❌ No (RAM-bound)                         | ❌ No (coupled)                               | ✅ Tiered storage (Kafka 3.6+)     |

---

## 4. Scalability (Parameters 41–50)

| #   | Parameter                         | DynamoDB                          | Redshift                                | Redis                          | PostgreSQL                                 | Kafka                               |
| --- | --------------------------------- | --------------------------------- | --------------------------------------- | ------------------------------ | ------------------------------------------ | ----------------------------------- |
| 41  | **Horizontal scaling (writes)**   | ✅ Automatic partitioning         | Add nodes (resize)                      | ✅ Redis Cluster (auto-shard)  | ❌ Single-writer (use Citus for sharding)  | ✅ Add partitions + brokers         |
| 42  | **Horizontal scaling (reads)**    | ✅ eventually consistent + DAX    | Add nodes                               | ✅ Read replicas               | ✅ Read replicas                           | ✅ Add consumers to group           |
| 43  | **Max cluster size**              | Unlimited (managed)               | 128 nodes (ra3.16xlarge)                | 1,000 nodes (Redis Cluster)    | Single primary + 15 read replicas (Aurora) | Thousands of brokers                |
| 44  | **Partitioning/sharding**         | Automatic (hash on partition key) | Distribution key (EVEN, KEY, ALL, AUTO) | Hash slots (16,384)            | Table partitioning (range, list, hash)     | Topic partitions (explicit count)   |
| 45  | **Replication**                   | 3+ AZs (automatic, transparent)   | Within cluster (3 copies)               | Async primary→replica          | Streaming (physical), Logical              | ISR replicas per partition          |
| 46  | **Multi-region**                  | Global Tables (active-active)     | ❌ Single-region (snapshot to another)  | ❌ (application-level)         | ❌ (logical replication cross-region)      | ✅ MirrorMaker 2 / Cluster Linking  |
| 47  | **Zero-downtime scaling**         | ✅ Yes                            | ✅ Elastic resize, Serverless           | ✅ Cluster resharding (online) | ✅ Add read replicas online                | ✅ Add brokers, reassign partitions |
| 48  | **Data partitioning granularity** | Per item (partition key)          | Per row (distribution key)              | Per key (hash slot)            | Per row (partition key)                    | Per message (by key or round-robin) |
| 49  | **Re-partitioning**               | Automatic (transparent)           | Manual (change DISTKEY, rebuild)        | Online resharding              | Manual (alter partitions)                  | Manual (reassign partitions)        |
| 50  | **Serverless option**             | ✅ On-demand mode                 | ✅ Redshift Serverless                  | ✅ ElastiCache Serverless      | ✅ Aurora Serverless v2                    | ✅ MSK Serverless                   |

---

## 5. Consistency & Reliability (Parameters 51–60)

| #   | Parameter                  | DynamoDB                                                      | Redshift                            | Redis                                                         | PostgreSQL                                    | Kafka                                                            |
| --- | -------------------------- | ------------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------- |
| 51  | **Consistency model**      | Eventually consistent (default), strongly consistent (option) | Strong (within cluster)             | Eventually consistent (replication), strong (single instance) | Strong (ACID)                                 | Eventually consistent (replication), exactly-once (transactions) |
| 52  | **Isolation levels**       | Serializable (transactions)                                   | Serializable only                   | None (single-threaded, atomic per command)                    | Read Committed, Repeatable Read, Serializable | Read committed (consumer reads committed offsets)                |
| 53  | **Conflict resolution**    | Last-writer-wins (Global Tables)                              | N/A (single writer)                 | Last-writer-wins                                              | MVCC + locks                                  | Partition leader handles ordering                                |
| 54  | **Backup/restore**         | PITR (35-day continuous), on-demand backups                   | Automated snapshots (up to 35 days) | RDB snapshots, AOF                                            | pg_dump, pg_basebackup, PITR with WAL         | Topic retention is the "backup" + MirrorMaker                    |
| 55  | **Point-in-time recovery** | ✅ Yes (per-second granularity)                               | ✅ Yes (from snapshots)             | ❌ (restore from snapshot time)                               | ✅ Yes (WAL-based)                            | ✅ (seek to timestamp)                                           |
| 56  | **Disaster recovery**      | Global Tables (cross-region)                                  | Snapshot to another region          | Manual failover, MemoryDB multi-AZ                            | Streaming replication to standby              | MirrorMaker 2, cross-region replication                          |
| 57  | **Automatic failover**     | ✅ (transparent, built-in)                                    | ✅ (within cluster)                 | ✅ Sentinel / Cluster                                         | ✅ (RDS Multi-AZ, Patroni for self-hosted)    | ✅ (ISR leader election)                                         |
| 58  | **Data loss on failover**  | None (synchronous within region)                              | None (within cluster)               | Possible (async replication)                                  | None (synchronous replication option)         | None (acks=all + min.insync.replicas)                            |
| 59  | **Durability guarantee**   | 99.999999999% (11 nines)                                      | High (multi-copy within cluster)    | Configurable (RDB/AOF = possible loss of last second)         | WAL + fsync = no loss after commit            | replication.factor × acks determines level                       |
| 60  | **SLA (availability)**     | 99.999% (Global Tables), 99.99% (single region)               | 99.9%                               | 99.99% (MemoryDB)                                             | 99.99% (Aurora), 99.95% (RDS Multi-AZ)        | 99.95% (MSK)                                                     |

---

## 6. Operations & Management (Parameters 61–70)

| #   | Parameter                  | DynamoDB                         | Redshift                       | Redis                              | PostgreSQL                                | Kafka                                  |
| --- | -------------------------- | -------------------------------- | ------------------------------ | ---------------------------------- | ----------------------------------------- | -------------------------------------- |
| 61  | **Operational complexity** | Very low (serverless)            | Medium (cluster management)    | Low–Medium                         | Medium–High (VACUUM, tuning)              | High (brokers, ZK/KRaft, topics)       |
| 62  | **Patching/upgrades**      | ✅ Automatic (transparent)       | ✅ Maintenance windows         | ✅ ElastiCache: automatic          | ✅ RDS: maintenance windows               | Manual (rolling restart)               |
| 63  | **Monitoring**             | CloudWatch, Contributor Insights | CloudWatch, query monitoring   | CloudWatch, INFO command, SLOWLOG  | pg_stat_statements, CloudWatch            | JMX metrics, consumer lag monitoring   |
| 64  | **Migration tools**        | AWS DMS                          | AWS DMS, COPY from S3          | Redis MIGRATE, dump/restore        | pg_dump/restore, DMS, logical replication | MirrorMaker 2, Confluent Replicator    |
| 65  | **Schema migration**       | N/A (schema-less)                | ALTER TABLE, SQL scripts       | N/A (schema-less)                  | SQL DDL, tools: Flyway, Prisma, Knex      | Schema Registry (compatibility checks) |
| 66  | **Maintenance tasks**      | None                             | VACUUM, ANALYZE (auto)         | None (automatic memory management) | VACUUM, ANALYZE, REINDEX                  | Log retention cleanup (automatic)      |
| 67  | **Configuration changes**  | Instant (API)                    | Requires restart for some      | CONFIG SET (runtime)               | ALTER SYSTEM + reload (some need restart) | Requires broker restart for some       |
| 68  | **Multi-tenancy**          | Separate tables or prefix keys   | Schemas, row-level security    | Database numbers (0–15), ACLs      | Schemas, row-level security               | Topics per tenant, ACLs                |
| 69  | **Infrastructure as Code** | CloudFormation, CDK, Terraform   | CloudFormation, CDK, Terraform | CloudFormation, CDK, Terraform     | CloudFormation, CDK, Terraform            | CloudFormation (MSK), Terraform        |
| 70  | **CLI/Admin tools**        | AWS CLI, NoSQL Workbench         | AWS CLI, Query Editor v2       | redis-cli, RedisInsight            | psql, pgAdmin, DBeaver                    | kafka-topics.sh, AKHQ, Conduktor       |

---

## 7. Security (Parameters 71–80)

| #   | Parameter                     | DynamoDB                                  | Redshift                             | Redis                                        | PostgreSQL                                       | Kafka                                 |
| --- | ----------------------------- | ----------------------------------------- | ------------------------------------ | -------------------------------------------- | ------------------------------------------------ | ------------------------------------- |
| 71  | **Authentication**            | IAM (AWS Sig v4)                          | IAM, database users                  | Password, ACLs (Redis 6+), IAM (ElastiCache) | Password, SCRAM-SHA-256, certificates, IAM (RDS) | SASL (SCRAM, GSSAPI), mTLS, IAM (MSK) |
| 72  | **Authorization**             | IAM policies (fine-grained to item level) | GRANT/REVOKE, row-level security     | ACLs (per-command, per-key pattern)          | GRANT/REVOKE, row-level security, column-level   | ACLs (per-topic, per-consumer group)  |
| 73  | **Encryption at rest**        | ✅ AES-256 (default, KMS optional)        | ✅ AES-256 (KMS)                     | ✅ (ElastiCache/MemoryDB)                    | ✅ (RDS, Aurora)                                 | ✅ (MSK: KMS)                         |
| 74  | **Encryption in transit**     | ✅ HTTPS (always)                         | ✅ SSL/TLS                           | ✅ TLS (Redis 6+)                            | ✅ SSL/TLS                                       | ✅ TLS                                |
| 75  | **VPC isolation**             | ✅ VPC endpoints                          | ✅ VPC                               | ✅ VPC                                       | ✅ VPC                                           | ✅ VPC (MSK)                          |
| 76  | **Audit logging**             | CloudTrail (API), DynamoDB Streams (data) | CloudTrail + audit logs (STL tables) | ACL logs, SLOWLOG                            | pgAudit extension, log_statement                 | Authorizer logs, CloudTrail (MSK)     |
| 77  | **Data masking**              | ❌ (application-level)                    | Dynamic data masking (DDM)           | ❌ (application-level)                       | ❌ (use views or extensions)                     | ❌ (application-level)                |
| 78  | **Compliance certifications** | SOC, PCI, HIPAA, FedRAMP...               | SOC, PCI, HIPAA, FedRAMP...          | SOC, PCI, HIPAA (ElastiCache)                | SOC, PCI, HIPAA (RDS/Aurora)                     | SOC, PCI, HIPAA (MSK)                 |
| 79  | **Fine-grained access**       | IAM conditions on partition/sort key      | Column + row-level security          | ACL per key pattern + command                | Column + row-level security                      | Per-topic + per-consumer-group ACL    |
| 80  | **Network isolation**         | VPC endpoints, PrivateLink                | VPC, enhanced VPC routing            | VPC, security groups                         | VPC, security groups                             | VPC, security groups (MSK)            |

---

## 8. Cost & Pricing (Parameters 81–90)

| #   | Parameter                       | DynamoDB                                                       | Redshift                                  | Redis                           | PostgreSQL                                    | Kafka                                   |
| --- | ------------------------------- | -------------------------------------------------------------- | ----------------------------------------- | ------------------------------- | --------------------------------------------- | --------------------------------------- |
| 81  | **Pricing model**               | Per-request OR provisioned capacity                            | Per-node-hour OR serverless RPU           | Per-node-hour OR serverless     | Per-instance-hour OR serverless ACU           | Per-broker-hour + storage (MSK)         |
| 82  | **Free tier**                   | 25 GB + 25 WCU + 25 RCU (always free)                          | 2-month trial (DC2.large)                 | ❌ No                           | ✅ RDS 750 hrs/month (12 months)              | ❌ No                                   |
| 83  | **Storage cost**                | $0.25/GB/month                                                 | $0.024/GB/month (managed storage)         | RAM pricing (expensive per GB)  | ~$0.115/GB/month (gp3 EBS)                    | ~$0.10/GB/month (EBS)                   |
| 84  | **Cheapest production setup**   | ~$1/month (on-demand, low traffic)                             | ~$180/month (Serverless min)              | ~$40/month (cache.t4g.micro)    | ~$30/month (db.t4g.micro)                     | ~$250/month (MSK Serverless)            |
| 85  | **Cost scales with**            | Read/write requests + storage                                  | Compute nodes + storage                   | Instance size (RAM)             | Instance size + storage + IOPS                | Brokers + storage + data transfer       |
| 86  | **Reserved pricing**            | ✅ Reserved capacity (1–3 yr)                                  | ✅ Reserved nodes (1–3 yr)                | ✅ Reserved nodes               | ✅ Reserved instances                         | ❌ (MSK)                                |
| 87  | **Data transfer cost**          | ✅ Cross-region replication cost                               | ✅ Data transfer + Spectrum scan          | ✅ Cross-AZ data transfer       | ✅ Cross-AZ and cross-region                  | ✅ Data transfer between AZs            |
| 88  | **Cost optimization tip**       | Use on-demand for spiky, provisioned + auto-scaling for steady | Use Serverless for bursty, RA3 for steady | Right-size instances, use TTLs  | Use Aurora Serverless v2 for variable loads   | Use tiered storage, tune retention      |
| 89  | **Hidden costs**                | GSI storage + throughput, Streams, PITR                        | Spectrum per-scan, concurrency scaling    | Backup storage, data transfer   | PITR storage, Performance Insights, snapshots | Cross-AZ data transfer, Schema Registry |
| 90  | **Cost at 1 TB, moderate load** | ~$250–500/month                                                | ~$500–1000/month                          | N/A (1 TB RAM = very expensive) | ~$200–400/month                               | ~$400–800/month                         |

---

## 9. Ecosystem & Integration (Parameters 91–100)

| #   | Parameter                        | DynamoDB                           | Redshift                                  | Redis                                | PostgreSQL                                       | Kafka                                      |
| --- | -------------------------------- | ---------------------------------- | ----------------------------------------- | ------------------------------------ | ------------------------------------------------ | ------------------------------------------ |
| 91  | **Change data capture**          | DynamoDB Streams, Kinesis          | ❌ (poll-based ETL)                       | Keyspace notifications, Streams      | Logical replication, WAL                         | IS the CDC platform (Debezium → Kafka)     |
| 92  | **Event-driven integration**     | Streams → Lambda                   | ❌ (batch/scheduled)                      | Pub/Sub, Streams → consumers         | LISTEN/NOTIFY, logical decoding                  | Native (producers → consumers)             |
| 93  | **BI / Analytics tools**         | Export to S3 → Athena/QuickSight   | Native SQL, QuickSight, Looker, Tableau   | ❌ Not designed for BI               | Any SQL BI tool                                  | Kafka → data lake → BI tools               |
| 94  | **ETL/ELT integration**          | Glue, DMS, Zero-ETL to Redshift    | Glue, COPY, Zero-ETL from Aurora/DynamoDB | ❌ (not a data source for ETL)       | DMS, Glue, Foreign Data Wrappers                 | Kafka Connect (200+ connectors)            |
| 95  | **Supported languages/SDKs**     | All AWS SDKs (20+ languages)       | JDBC/ODBC, Python (redshift_connector)    | Clients in 50+ languages             | Clients in 30+ languages (native libpq)          | Java, Python, Go, Node.js, C/C++, .NET     |
| 96  | **Popular ORMs / clients**       | AWS SDK, Dynamoose, ElectroDB      | SQLAlchemy, dbt                           | ioredis, redis-py, Jedis, Lettuce    | Prisma, TypeORM, Sequelize, Knex, SQLAlchemy     | KafkaJS, confluent-kafka-python, Sarama    |
| 97  | **Streaming/real-time**          | DynamoDB Streams (24 hr retention) | ❌ (batch)                                | ✅ Pub/Sub, Streams                  | ✅ LISTEN/NOTIFY, logical replication slots      | ✅ Core purpose                            |
| 98  | **Machine learning integration** | Export to SageMaker                | Redshift ML (SQL-based)                   | ✅ Vector similarity (Redis modules) | pgvector (embeddings), MADlib                    | Kafka → ML pipeline                        |
| 99  | **Graph queries**                | ❌                                 | ❌                                        | ❌ (RedisGraph deprecated)           | ✅ Apache AGE extension                          | ❌                                         |
| 100 | **Community & ecosystem size**   | Large (AWS-specific)               | Medium (AWS-specific)                     | Very large (open-source)             | Very large (largest open-source RDBMS community) | Very large (Apache foundation + Confluent) |

---

## Quick Decision Guide

### When to pick each technology:

| Scenario                                        | Best choice                                  | Why                                                   |
| ----------------------------------------------- | -------------------------------------------- | ----------------------------------------------------- |
| Need single-digit ms CRUD at any scale          | **DynamoDB**                                 | Fully managed, auto-scaling, predictable latency      |
| Complex analytical SQL on TBs/PBs               | **Redshift**                                 | Columnar storage, parallel query, SQL interface       |
| Sub-millisecond caching / sessions / counters   | **Redis**                                    | In-memory, rich data structures, simplest to use      |
| Transactional app with complex queries + joins  | **PostgreSQL**                               | ACID, full SQL, extensions, most versatile            |
| Real-time event streaming / decoupling services | **Kafka**                                    | High throughput, durable log, exactly-once, replay    |
| User-facing product with mixed reads/writes     | **PostgreSQL** + **Redis** (cache)           | Postgres for source of truth, Redis for speed         |
| E-commerce order pipeline                       | **Kafka** → **DynamoDB** + **Redis**         | Kafka for events, DynamoDB for orders, Redis for cart |
| Data lake ingestion from multiple sources       | **Kafka** → **Redshift** (or S3)             | Kafka Connect for ingestion, Redshift for analysis    |
| Leaderboard / rate limiting / real-time ranking | **Redis**                                    | Sorted Sets = purpose-built for this                  |
| Audit log / event sourcing                      | **Kafka**                                    | Immutable append-only log, replay from any point      |
| Time-series with analytics                      | **PostgreSQL** (TimescaleDB) or **Redshift** | TimescaleDB for operational, Redshift for analytical  |
| Session store for web app                       | **Redis** or **DynamoDB**                    | Redis if you have it; DynamoDB if fully serverless    |
| CDC (database change capture)                   | **Kafka** + Debezium                         | Industry standard for streaming DB changes            |
| Report generation from large dataset            | **Redshift**                                 | Columnar + SQL = fast aggregations                    |
| Chat / notifications / real-time features       | **Redis** Pub/Sub or Streams                 | Low latency, built-in pub/sub                         |

---

## Common Architecture Patterns

### Pattern 1: CQRS (Command Query Responsibility Segregation)

```
Writes → PostgreSQL → CDC (Kafka) → Read models:
                                      ├── Redis (fast lookups)
                                      ├── Redshift (analytics)
                                      └── DynamoDB (API reads at scale)
```

### Pattern 2: Event-Driven Microservices

```
Service A → Kafka topic → Service B → DynamoDB
                       → Service C → PostgreSQL
                       → Service D → Redis (cache invalidation)
```

### Pattern 3: Real-Time Analytics Pipeline

```
App events → Kafka → Kafka Streams (aggregate) → Redshift (warehouse)
                                                → Redis (real-time dashboard)
```

### Pattern 4: E-Commerce Platform

```
User sessions   → Redis (fast, TTL-based)
Product catalog → PostgreSQL (complex queries, full-text search)
Shopping cart    → Redis or DynamoDB (fast, per-user)
Orders          → PostgreSQL → Kafka (CDC) → DynamoDB (read API)
Analytics       → Kafka → Redshift
Recommendations → PostgreSQL (pgvector) or Redis (vector similarity)
```
