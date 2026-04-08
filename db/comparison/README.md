# DynamoDB vs Redshift vs Redis vs PostgreSQL vs Kafka vs Elasticsearch — 100-Parameter Comparison

A comprehensive **side-by-side comparison** of six widely used data technologies across 100 real-world parameters. Use this as a quick-reference decision guide.

---

## Legend

- **DynamoDB** — AWS managed NoSQL (key-value + document)
- **Redshift** — AWS managed columnar data warehouse
- **Redis** — In-memory data structure store / cache
- **PostgreSQL** — Open-source relational database
- **Kafka** — Distributed event streaming platform
- **Elasticsearch** — Distributed search & analytics engine

---

## 1. Core Identity (Parameters 1–10)

| #   | Parameter              | DynamoDB                         | Redshift                             | Redis                                 | PostgreSQL                          | Kafka                             | Elasticsearch                                      |
| --- | ---------------------- | -------------------------------- | ------------------------------------ | ------------------------------------- | ----------------------------------- | --------------------------------- | -------------------------------------------------- |
| 1   | **Category**           | NoSQL database                   | Data warehouse (OLAP)                | In-memory store / cache               | Relational database (OLTP)          | Event streaming platform          | Search & analytics engine                          |
| 2   | **Data model**         | Key-value + document             | Columnar relational                  | Key → data structure                  | Relational (tables + JSON)          | Append-only log (topics)          | Document (JSON) with inverted index                |
| 3   | **Primary use case**   | Low-latency CRUD at scale        | Analytical queries on large datasets | Caching, sessions, real-time counters | Transactional apps, complex queries | Real-time event pipelines         | Full-text search, log analytics                    |
| 4   | **Query language**     | PartiQL / API (GetItem, Query)   | SQL (Redshift SQL)                   | Commands (GET, SET, ZADD...)          | SQL (most standards-compliant)      | None (produce/consume API)        | Query DSL (JSON) + SQL                             |
| 5   | **Schema**             | Schema-less (key-only schema)    | Schema-on-write (DDL required)       | Schema-less                           | Schema-on-write (DDL required)      | Schema-optional (Schema Registry) | Dynamic or explicit mappings                       |
| 6   | **Managed by AWS**     | ✅ Fully managed                 | ✅ Fully managed                     | ✅ ElastiCache / MemoryDB             | ✅ RDS / Aurora                     | ✅ MSK (Managed Streaming)        | ✅ OpenSearch Service / Serverless                 |
| 7   | **Self-hosted option** | ❌ (DynamoDB Local for dev only) | ❌ AWS only                          | ✅ Open-source                        | ✅ Open-source                      | ✅ Open-source                    | ✅ Open-source (OpenSearch: Apache 2.0)            |
| 8   | **Open source**        | ❌                               | ❌                                   | ✅ (BSD license)                      | ✅ (PostgreSQL license)             | ✅ (Apache 2.0)                   | ⚠️ SSPL / Elastic License (OpenSearch: Apache 2.0) |
| 9   | **First released**     | 2012                             | 2013                                 | 2009                                  | 1996                                | 2011                              | 2010                                               |
| 10  | **Best described as**  | "Serverless NoSQL"               | "Petabyte-scale analytics"           | "Sub-millisecond data store"          | "The world's most advanced RDBMS"   | "Distributed commit log"          | "Search everything, instantly"                     |

---

## 2. Data & Storage (Parameters 11–25)

| #   | Parameter                   | DynamoDB                                   | Redshift                                        | Redis                                                 | PostgreSQL                                           | Kafka                                    | Elasticsearch                                                    |
| --- | --------------------------- | ------------------------------------------ | ----------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------- | ---------------------------------------------------------------- |
| 11  | **Storage location**        | SSD (AWS-managed)                          | SSD/HDD (managed nodes)                         | RAM (primary), disk (persistence)                     | Disk (SSD/HDD)                                       | Disk (broker nodes)                      | Disk (SSD recommended for hot tier)                              |
| 12  | **Max item/row size**       | 400 KB                                     | Varies (max VARCHAR 64 KB)                      | 512 MB per value                                      | 1.6 TB per row (theoretical)                         | 1 MB default per message (configurable)  | 2 GB per document (default: 100 MB `http.max_content_length`)    |
| 13  | **Max database/table size** | Unlimited                                  | 8 PB compressed (ra3.16xlarge)                  | Limited by RAM                                        | 32 TB per table (practical)                          | Unlimited (disk-bound)                   | Unlimited (shard-based, disk-bound)                              |
| 14  | **Data compression**        | Automatic (internal)                       | Automatic columnar (AZ64, LZO, ZSTD)            | No (data is in memory)                                | TOAST for large values                               | Producer-side (LZ4, ZSTD, Snappy, GZIP)  | Automatic (Lucene codec: LZ4, DEFLATE, ZSTD)                     |
| 15  | **Column/field types**      | S, N, B, SS, NS, BS, L, M, BOOL, NULL      | Standard SQL types                              | Strings, Hashes, Lists, Sets, Sorted Sets, Streams... | 40+ types including JSON, arrays, ranges, geospatial | Bytes (schema via Schema Registry)       | text, keyword, numeric, date, geo_point, nested, dense_vector... |
| 16  | **Nested/document data**    | ✅ Maps and Lists (nested up to 32 levels) | ❌ Flat tables (SUPER type for semi-structured) | ✅ Hashes, JSON (with RedisJSON)                      | ✅ JSONB (indexable)                                 | ✅ Any format (JSON, Avro, Protobuf)     | ✅ Native JSON documents (nested, object types)                  |
| 17  | **Secondary indexes**       | GSI (max 20), LSI (max 5)                  | Sort keys, DIST keys (no traditional indexes)   | ❌ No indexes (hash-based lookup)                     | B-tree, GIN, GiST, BRIN, Hash, SP-GiST               | ❌ No indexes (offset-based)             | ✅ Every field has an inverted index automatically               |
| 18  | **Full-text search**        | ❌ (use OpenSearch)                        | Basic (LIKE, SIMILAR TO)                        | ❌ (use RediSearch module)                            | ✅ Built-in (tsvector/tsquery + GIN)                 | ❌ Not applicable                        | ✅ Core purpose (BM25, analyzers, fuzzy, highlighting)           |
| 19  | **Geospatial**              | ❌                                         | ✅ Geometry type (limited)                      | ✅ GEO commands (GEOADD, GEORADIUS)                   | ✅ PostGIS extension (industry standard)             | ❌ Not applicable                        | ✅ geo_point, geo_shape with distance/bounding queries           |
| 20  | **Time-series support**     | ❌ (model with sort key)                   | ✅ Good for time-series analytics               | ✅ Sorted Sets, Streams with TTL                      | ✅ TimescaleDB extension                             | ✅ Native (topic as time-ordered log)    | ✅ Data streams + ILM (hot/warm/cold tiers)                      |
| 21  | **Data retention**          | Unlimited (TTL for auto-delete)            | Unlimited                                       | TTL per key, eviction policies                        | Unlimited (manual management)                        | Configurable (time or size based)        | ILM policies (automatic tier + delete)                           |
| 22  | **Versioning**              | ❌ (implement with sort key)               | ❌                                              | ❌                                                    | ❌ (use triggers or audit tables)                    | ✅ Immutable log (all versions retained) | ✅ \_version field (optimistic concurrency)                      |
| 23  | **ACID transactions**       | ✅ (TransactWriteItems, up to 100 items)   | ✅ (Serializable isolation)                     | ✅ MULTI/EXEC (atomic, no rollback)                   | ✅ Full ACID with MVCC                               | ✅ Transactions across partitions        | ❌ No transactions (near-real-time, eventual)                    |
| 24  | **NULL handling**           | Attributes can be absent (no NULL concept) | SQL NULLs                                       | Key exists or doesn't                                 | SQL NULLs                                            | N/A (empty fields are application-level) | Fields can be absent or null                                     |
| 25  | **Data durability**         | 3+ AZ replication (automatic)              | 3 copies across cluster                         | Configurable (RDB/AOF/none)                           | WAL + streaming replication                          | Configurable (replication factor)        | Translog + replica shards across nodes                           |

---

## 3. Performance (Parameters 26–40)

| #   | Parameter                      | DynamoDB                                            | Redshift                                            | Redis                                     | PostgreSQL                                    | Kafka                              | Elasticsearch                                          |
| --- | ------------------------------ | --------------------------------------------------- | --------------------------------------------------- | ----------------------------------------- | --------------------------------------------- | ---------------------------------- | ------------------------------------------------------ |
| 26  | **Read latency (typical)**     | Single-digit ms                                     | Seconds to minutes                                  | Sub-millisecond                           | 1–100 ms (depends on query)                   | N/A (consumer pull latency: ms)    | 10–100 ms (depends on query complexity)                |
| 27  | **Write latency (typical)**    | Single-digit ms                                     | Seconds (batch-optimized)                           | Sub-millisecond                           | 1–50 ms                                       | 2–10 ms (acks=all)                 | 10–200 ms (refresh_interval for visibility)            |
| 28  | **Throughput model**           | Provisioned (RCU/WCU) or on-demand                  | Concurrency scaling / node slots                    | 100K+ ops/sec per node                    | Thousands of TPS (connection limited)         | Millions of msgs/sec per cluster   | Tens of thousands of queries/sec per node              |
| 29  | **Query complexity**           | Simple key lookups, scans, filters                  | Complex SQL (joins, aggregations, window functions) | Single-key or multi-key commands          | Full SQL (CTEs, window functions, subqueries) | N/A (no query engine)              | Rich Query DSL (bool, nested, script, aggregations)    |
| 30  | **Aggregations**               | ❌ (client-side only)                               | ✅ Optimized (columnar storage)                     | ❌ Limited (Lua scripts)                  | ✅ Full SQL aggregations                      | ✅ Kafka Streams / ksqlDB          | ✅ Bucket, metric, pipeline aggregations               |
| 31  | **JOINs**                      | ❌ No joins (denormalize)                           | ✅ Distributed joins                                | ❌ No joins                               | ✅ Full JOIN support                          | ❌ (join via Kafka Streams)        | ❌ No joins (denormalize, nested/parent-child)         |
| 32  | **Batch operations**           | BatchWriteItem (25 items), BatchGetItem (100 items) | COPY command (bulk)                                 | Pipeline (batched commands)               | COPY, multi-row INSERT                        | Batch produce/consume              | Bulk API (\_bulk endpoint for index/update/delete)     |
| 33  | **Connection model**           | HTTP API (stateless)                                | TCP (JDBC/ODBC, limited connections)                | TCP (persistent connections)              | TCP (process per connection)                  | TCP (persistent)                   | HTTP/REST API (stateless)                              |
| 34  | **Connection pooling needed?** | ❌ (HTTP, no connections)                           | ✅ Yes (WLM queues)                                 | ✅ Recommended at scale                   | ✅ Yes (PgBouncer)                            | ❌ (client manages internally)     | ❌ (HTTP, but use connection keep-alive)               |
| 35  | **Caching layer**              | DAX (in-memory, microsecond)                        | Result caching                                      | IS the cache                              | Shared buffers + OS cache                     | Page cache (OS filesystem)         | Node query cache + request cache + OS filesystem cache |
| 36  | **Hot partition/key problem**  | ✅ Yes (uneven key distribution)                    | ✅ Yes (skewed distribution keys)                   | ✅ Yes (hot keys)                         | ❌ Rare (index-based access)                  | ✅ Yes (hot partitions)            | ✅ Yes (hot shards from uneven routing)                |
| 37  | **Scan performance**           | Slow (reads every item, 1 MB pages)                 | Fast (columnar, parallel)                           | Keys scan with SCAN cursor (non-blocking) | Seq scan with parallel workers                | Fast sequential read (from offset) | match_all with scroll/search_after (parallel shards)   |
| 38  | **Concurrent users**           | Unlimited (HTTP)                                    | 500 connections (default)                           | 10,000+ connections                       | 100–500 (process per connection)              | Thousands (per consumer group)     | Thousands (HTTP, thread-pool based)                    |
| 39  | **Auto-scaling**               | ✅ On-demand mode (automatic)                       | ✅ Concurrency scaling, Serverless                  | ✅ ElastiCache auto-scaling               | ✅ Aurora auto-scaling (read replicas)        | ✅ MSK auto-scaling (storage)      | ✅ OpenSearch Serverless auto-scales                   |
| 40  | **Compute-storage separation** | ✅ Yes (inherent)                                   | ✅ Yes (RA3 nodes + managed storage)                | ❌ No (RAM-bound)                         | ❌ No (coupled)                               | ✅ Tiered storage (Kafka 3.6+)     | ✅ Searchable snapshots (warm/cold tiers)              |

---

## 4. Scalability (Parameters 41–50)

| #   | Parameter                         | DynamoDB                          | Redshift                                | Redis                          | PostgreSQL                                 | Kafka                               | Elasticsearch                                        |
| --- | --------------------------------- | --------------------------------- | --------------------------------------- | ------------------------------ | ------------------------------------------ | ----------------------------------- | ---------------------------------------------------- |
| 41  | **Horizontal scaling (writes)**   | ✅ Automatic partitioning         | Add nodes (resize)                      | ✅ Redis Cluster (auto-shard)  | ❌ Single-writer (use Citus for sharding)  | ✅ Add partitions + brokers         | ✅ Add shards (split index across more nodes)        |
| 42  | **Horizontal scaling (reads)**    | ✅ eventually consistent + DAX    | Add nodes                               | ✅ Read replicas               | ✅ Read replicas                           | ✅ Add consumers to group           | ✅ Replica shards serve read requests                |
| 43  | **Max cluster size**              | Unlimited (managed)               | 128 nodes (ra3.16xlarge)                | 1,000 nodes (Redis Cluster)    | Single primary + 15 read replicas (Aurora) | Thousands of brokers                | Hundreds of nodes (1000+ at large orgs)              |
| 44  | **Partitioning/sharding**         | Automatic (hash on partition key) | Distribution key (EVEN, KEY, ALL, AUTO) | Hash slots (16,384)            | Table partitioning (range, list, hash)     | Topic partitions (explicit count)   | Configurable shards per index (hash routing by \_id) |
| 45  | **Replication**                   | 3+ AZs (automatic, transparent)   | Within cluster (3 copies)               | Async primary→replica          | Streaming (physical), Logical              | ISR replicas per partition          | Replica shards (configurable factor per index)       |
| 46  | **Multi-region**                  | Global Tables (active-active)     | ❌ Single-region (snapshot to another)  | ❌ (application-level)         | ❌ (logical replication cross-region)      | ✅ MirrorMaker 2 / Cluster Linking  | ✅ Cross-cluster replication (CCR)                   |
| 47  | **Zero-downtime scaling**         | ✅ Yes                            | ✅ Elastic resize, Serverless           | ✅ Cluster resharding (online) | ✅ Add read replicas online                | ✅ Add brokers, reassign partitions | ✅ Add nodes, reindex with aliases (zero downtime)   |
| 48  | **Data partitioning granularity** | Per item (partition key)          | Per row (distribution key)              | Per key (hash slot)            | Per row (partition key)                    | Per message (by key or round-robin) | Per document (routing by \_id or custom routing key) |
| 49  | **Re-partitioning**               | Automatic (transparent)           | Manual (change DISTKEY, rebuild)        | Online resharding              | Manual (alter partitions)                  | Manual (reassign partitions)        | Reindex with \_reindex API + alias flip (manual)     |
| 50  | **Serverless option**             | ✅ On-demand mode                 | ✅ Redshift Serverless                  | ✅ ElastiCache Serverless      | ✅ Aurora Serverless v2                    | ✅ MSK Serverless                   | ✅ OpenSearch Serverless (auto-provisioned)          |

---

## 5. Consistency & Reliability (Parameters 51–60)

| #   | Parameter                  | DynamoDB                                                      | Redshift                            | Redis                                                         | PostgreSQL                                    | Kafka                                                            | Elasticsearch                                               |
| --- | -------------------------- | ------------------------------------------------------------- | ----------------------------------- | ------------------------------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------- | ----------------------------------------------------------- |
| 51  | **Consistency model**      | Eventually consistent (default), strongly consistent (option) | Strong (within cluster)             | Eventually consistent (replication), strong (single instance) | Strong (ACID)                                 | Eventually consistent (replication), exactly-once (transactions) | Near-real-time (refresh_interval, default 1s)               |
| 52  | **Isolation levels**       | Serializable (transactions)                                   | Serializable only                   | None (single-threaded, atomic per command)                    | Read Committed, Repeatable Read, Serializable | Read committed (consumer reads committed offsets)                | None (optimistic concurrency via \_version)                 |
| 53  | **Conflict resolution**    | Last-writer-wins (Global Tables)                              | N/A (single writer)                 | Last-writer-wins                                              | MVCC + locks                                  | Partition leader handles ordering                                | Optimistic concurrency (version conflict → 409)             |
| 54  | **Backup/restore**         | PITR (35-day continuous), on-demand backups                   | Automated snapshots (up to 35 days) | RDB snapshots, AOF                                            | pg_dump, pg_basebackup, PITR with WAL         | Topic retention is the "backup" + MirrorMaker                    | Snapshot/restore API (to S3, shared filesystem)             |
| 55  | **Point-in-time recovery** | ✅ Yes (per-second granularity)                               | ✅ Yes (from snapshots)             | ❌ (restore from snapshot time)                               | ✅ Yes (WAL-based)                            | ✅ (seek to timestamp)                                           | ❌ (restore from snapshot time)                             |
| 56  | **Disaster recovery**      | Global Tables (cross-region)                                  | Snapshot to another region          | Manual failover, MemoryDB multi-AZ                            | Streaming replication to standby              | MirrorMaker 2, cross-region replication                          | Cross-cluster replication (CCR), snapshot to remote repo    |
| 57  | **Automatic failover**     | ✅ (transparent, built-in)                                    | ✅ (within cluster)                 | ✅ Sentinel / Cluster                                         | ✅ (RDS Multi-AZ, Patroni for self-hosted)    | ✅ (ISR leader election)                                         | ✅ Automatic shard reallocation on node failure             |
| 58  | **Data loss on failover**  | None (synchronous within region)                              | None (within cluster)               | Possible (async replication)                                  | None (synchronous replication option)         | None (acks=all + min.insync.replicas)                            | None if replica shards exist (translog + replicas)          |
| 59  | **Durability guarantee**   | 99.999999999% (11 nines)                                      | High (multi-copy within cluster)    | Configurable (RDB/AOF = possible loss of last second)         | WAL + fsync = no loss after commit            | replication.factor × acks determines level                       | Translog (fsync) + replica shards (configurable durability) |
| 60  | **SLA (availability)**     | 99.999% (Global Tables), 99.99% (single region)               | 99.9%                               | 99.99% (MemoryDB)                                             | 99.99% (Aurora), 99.95% (RDS Multi-AZ)        | 99.95% (MSK)                                                     | 99.9% (OpenSearch Service)                                  |

---

## 6. Operations & Management (Parameters 61–70)

| #   | Parameter                  | DynamoDB                         | Redshift                       | Redis                              | PostgreSQL                                | Kafka                                  | Elasticsearch                                         |
| --- | -------------------------- | -------------------------------- | ------------------------------ | ---------------------------------- | ----------------------------------------- | -------------------------------------- | ----------------------------------------------------- |
| 61  | **Operational complexity** | Very low (serverless)            | Medium (cluster management)    | Low–Medium                         | Medium–High (VACUUM, tuning)              | High (brokers, ZK/KRaft, topics)       | Medium–High (shard tuning, mappings, ILM)             |
| 62  | **Patching/upgrades**      | ✅ Automatic (transparent)       | ✅ Maintenance windows         | ✅ ElastiCache: automatic          | ✅ RDS: maintenance windows               | Manual (rolling restart)               | Rolling upgrade (node by node)                        |
| 63  | **Monitoring**             | CloudWatch, Contributor Insights | CloudWatch, query monitoring   | CloudWatch, INFO command, SLOWLOG  | pg_stat_statements, CloudWatch            | JMX metrics, consumer lag monitoring   | \_cluster/health, \_cat APIs, CloudWatch (OpenSearch) |
| 64  | **Migration tools**        | AWS DMS                          | AWS DMS, COPY from S3          | Redis MIGRATE, dump/restore        | pg_dump/restore, DMS, logical replication | MirrorMaker 2, Confluent Replicator    | \_reindex API, Logstash, Snapshot/Restore             |
| 65  | **Schema migration**       | N/A (schema-less)                | ALTER TABLE, SQL scripts       | N/A (schema-less)                  | SQL DDL, tools: Flyway, Prisma, Knex      | Schema Registry (compatibility checks) | Reindex with new mapping + alias flip                 |
| 66  | **Maintenance tasks**      | None                             | VACUUM, ANALYZE (auto)         | None (automatic memory management) | VACUUM, ANALYZE, REINDEX                  | Log retention cleanup (automatic)      | \_forcemerge, ILM rollover, shard rebalancing         |
| 67  | **Configuration changes**  | Instant (API)                    | Requires restart for some      | CONFIG SET (runtime)               | ALTER SYSTEM + reload (some need restart) | Requires broker restart for some       | Dynamic settings (API), static (requires restart)     |
| 68  | **Multi-tenancy**          | Separate tables or prefix keys   | Schemas, row-level security    | Database numbers (0–15), ACLs      | Schemas, row-level security               | Topics per tenant, ACLs                | Index per tenant, filtered aliases, or DLS            |
| 69  | **Infrastructure as Code** | CloudFormation, CDK, Terraform   | CloudFormation, CDK, Terraform | CloudFormation, CDK, Terraform     | CloudFormation, CDK, Terraform            | CloudFormation (MSK), Terraform        | CloudFormation (OpenSearch), CDK, Terraform           |
| 70  | **CLI/Admin tools**        | AWS CLI, NoSQL Workbench         | AWS CLI, Query Editor v2       | redis-cli, RedisInsight            | psql, pgAdmin, DBeaver                    | kafka-topics.sh, AKHQ, Conduktor       | Kibana/OpenSearch Dashboards, Dev Tools, curl         |

---

## 7. Security (Parameters 71–80)

| #   | Parameter                     | DynamoDB                                  | Redshift                             | Redis                                        | PostgreSQL                                       | Kafka                                 | Elasticsearch                                           |
| --- | ----------------------------- | ----------------------------------------- | ------------------------------------ | -------------------------------------------- | ------------------------------------------------ | ------------------------------------- | ------------------------------------------------------- |
| 71  | **Authentication**            | IAM (AWS Sig v4)                          | IAM, database users                  | Password, ACLs (Redis 6+), IAM (ElastiCache) | Password, SCRAM-SHA-256, certificates, IAM (RDS) | SASL (SCRAM, GSSAPI), mTLS, IAM (MSK) | Native realm, LDAP, SAML, OIDC, IAM (OpenSearch)        |
| 72  | **Authorization**             | IAM policies (fine-grained to item level) | GRANT/REVOKE, row-level security     | ACLs (per-command, per-key pattern)          | GRANT/REVOKE, row-level security, column-level   | ACLs (per-topic, per-consumer group)  | Role-based (RBAC), field-level, document-level security |
| 73  | **Encryption at rest**        | ✅ AES-256 (default, KMS optional)        | ✅ AES-256 (KMS)                     | ✅ (ElastiCache/MemoryDB)                    | ✅ (RDS, Aurora)                                 | ✅ (MSK: KMS)                         | ✅ AES-256 (KMS for OpenSearch, node-level encryption)  |
| 74  | **Encryption in transit**     | ✅ HTTPS (always)                         | ✅ SSL/TLS                           | ✅ TLS (Redis 6+)                            | ✅ SSL/TLS                                       | ✅ TLS                                | ✅ TLS (node-to-node + client-to-node)                  |
| 75  | **VPC isolation**             | ✅ VPC endpoints                          | ✅ VPC                               | ✅ VPC                                       | ✅ VPC                                           | ✅ VPC (MSK)                          | ✅ VPC (OpenSearch)                                     |
| 76  | **Audit logging**             | CloudTrail (API), DynamoDB Streams (data) | CloudTrail + audit logs (STL tables) | ACL logs, SLOWLOG                            | pgAudit extension, log_statement                 | Authorizer logs, CloudTrail (MSK)     | Audit logs (X-Pack / OpenSearch plugin)                 |
| 77  | **Data masking**              | ❌ (application-level)                    | Dynamic data masking (DDM)           | ❌ (application-level)                       | ❌ (use views or extensions)                     | ❌ (application-level)                | ❌ (field-level security to restrict, not mask)         |
| 78  | **Compliance certifications** | SOC, PCI, HIPAA, FedRAMP...               | SOC, PCI, HIPAA, FedRAMP...          | SOC, PCI, HIPAA (ElastiCache)                | SOC, PCI, HIPAA (RDS/Aurora)                     | SOC, PCI, HIPAA (MSK)                 | SOC, PCI, HIPAA (OpenSearch Service)                    |
| 79  | **Fine-grained access**       | IAM conditions on partition/sort key      | Column + row-level security          | ACL per key pattern + command                | Column + row-level security                      | Per-topic + per-consumer-group ACL    | Document-level security (DLS) + field-level security    |
| 80  | **Network isolation**         | VPC endpoints, PrivateLink                | VPC, enhanced VPC routing            | VPC, security groups                         | VPC, security groups                             | VPC, security groups (MSK)            | VPC, security groups, fine-grained access policies      |

---

## 8. Cost & Pricing (Parameters 81–90)

| #   | Parameter                       | DynamoDB                                                       | Redshift                                  | Redis                           | PostgreSQL                                    | Kafka                                   | Elasticsearch                                         |
| --- | ------------------------------- | -------------------------------------------------------------- | ----------------------------------------- | ------------------------------- | --------------------------------------------- | --------------------------------------- | ----------------------------------------------------- |
| 81  | **Pricing model**               | Per-request OR provisioned capacity                            | Per-node-hour OR serverless RPU           | Per-node-hour OR serverless     | Per-instance-hour OR serverless ACU           | Per-broker-hour + storage (MSK)         | Per-instance-hour OR OpenSearch Serverless (OCU)      |
| 82  | **Free tier**                   | 25 GB + 25 WCU + 25 RCU (always free)                          | 2-month trial (DC2.large)                 | ❌ No                           | ✅ RDS 750 hrs/month (12 months)              | ❌ No                                   | ❌ No (OpenSearch has no free tier)                   |
| 83  | **Storage cost**                | $0.25/GB/month                                                 | $0.024/GB/month (managed storage)         | RAM pricing (expensive per GB)  | ~$0.115/GB/month (gp3 EBS)                    | ~$0.10/GB/month (EBS)                   | ~$0.10–$0.135/GB/month (EBS, tier-dependent)          |
| 84  | **Cheapest production setup**   | ~$1/month (on-demand, low traffic)                             | ~$180/month (Serverless min)              | ~$40/month (cache.t4g.micro)    | ~$30/month (db.t4g.micro)                     | ~$250/month (MSK Serverless)            | ~$70/month (t3.small.search single node)              |
| 85  | **Cost scales with**            | Read/write requests + storage                                  | Compute nodes + storage                   | Instance size (RAM)             | Instance size + storage + IOPS                | Brokers + storage + data transfer       | Instance count + storage + data transfer              |
| 86  | **Reserved pricing**            | ✅ Reserved capacity (1–3 yr)                                  | ✅ Reserved nodes (1–3 yr)                | ✅ Reserved nodes               | ✅ Reserved instances                         | ❌ (MSK)                                | ✅ Reserved instances (OpenSearch)                    |
| 87  | **Data transfer cost**          | ✅ Cross-region replication cost                               | ✅ Data transfer + Spectrum scan          | ✅ Cross-AZ data transfer       | ✅ Cross-AZ and cross-region                  | ✅ Data transfer between AZs            | ✅ Cross-AZ data transfer (multi-AZ deployment)       |
| 88  | **Cost optimization tip**       | Use on-demand for spiky, provisioned + auto-scaling for steady | Use Serverless for bursty, RA3 for steady | Right-size instances, use TTLs  | Use Aurora Serverless v2 for variable loads   | Use tiered storage, tune retention      | Use ILM (hot/warm/cold), UltraWarm for older data     |
| 89  | **Hidden costs**                | GSI storage + throughput, Streams, PITR                        | Spectrum per-scan, concurrency scaling    | Backup storage, data transfer   | PITR storage, Performance Insights, snapshots | Cross-AZ data transfer, Schema Registry | UltraWarm storage, cross-AZ, snapshot storage to S3   |
| 90  | **Cost at 1 TB, moderate load** | ~$250–500/month                                                | ~$500–1000/month                          | N/A (1 TB RAM = very expensive) | ~$200–400/month                               | ~$400–800/month                         | ~$300–700/month (depends on instance type + replicas) |

---

## 9. Ecosystem & Integration (Parameters 91–100)

| #   | Parameter                        | DynamoDB                           | Redshift                                  | Redis                                | PostgreSQL                                       | Kafka                                      | Elasticsearch                                                        |
| --- | -------------------------------- | ---------------------------------- | ----------------------------------------- | ------------------------------------ | ------------------------------------------------ | ------------------------------------------ | -------------------------------------------------------------------- |
| 91  | **Change data capture**          | DynamoDB Streams, Kinesis          | ❌ (poll-based ETL)                       | Keyspace notifications, Streams      | Logical replication, WAL                         | IS the CDC platform (Debezium → Kafka)     | ❌ (not a source of truth; use Logstash/Beats input)                 |
| 92  | **Event-driven integration**     | Streams → Lambda                   | ❌ (batch/scheduled)                      | Pub/Sub, Streams → consumers         | LISTEN/NOTIFY, logical decoding                  | Native (producers → consumers)             | Watcher (alerting), webhooks via plugins                             |
| 93  | **BI / Analytics tools**         | Export to S3 → Athena/QuickSight   | Native SQL, QuickSight, Looker, Tableau   | ❌ Not designed for BI               | Any SQL BI tool                                  | Kafka → data lake → BI tools               | Kibana / OpenSearch Dashboards (native visualization)                |
| 94  | **ETL/ELT integration**          | Glue, DMS, Zero-ETL to Redshift    | Glue, COPY, Zero-ETL from Aurora/DynamoDB | ❌ (not a data source for ETL)       | DMS, Glue, Foreign Data Wrappers                 | Kafka Connect (200+ connectors)            | Logstash, Beats, Kafka Connect sink                                  |
| 95  | **Supported languages/SDKs**     | All AWS SDKs (20+ languages)       | JDBC/ODBC, Python (redshift_connector)    | Clients in 50+ languages             | Clients in 30+ languages (native libpq)          | Java, Python, Go, Node.js, C/C++, .NET     | REST API (any language), official Java/Python/JS/Go/PHP/.NET clients |
| 96  | **Popular ORMs / clients**       | AWS SDK, Dynamoose, ElectroDB      | SQLAlchemy, dbt                           | ioredis, redis-py, Jedis, Lettuce    | Prisma, TypeORM, Sequelize, Knex, SQLAlchemy     | KafkaJS, confluent-kafka-python, Sarama    | @elastic/elasticsearch, elasticsearch-py, olivere                    |
| 97  | **Streaming/real-time**          | DynamoDB Streams (24 hr retention) | ❌ (batch)                                | ✅ Pub/Sub, Streams                  | ✅ LISTEN/NOTIFY, logical replication slots      | ✅ Core purpose                            | Near-real-time search (1s refresh), Watcher alerts                   |
| 98  | **Machine learning integration** | Export to SageMaker                | Redshift ML (SQL-based)                   | ✅ Vector similarity (Redis modules) | pgvector (embeddings), MADlib                    | Kafka → ML pipeline                        | kNN / vector search (dense_vector), LTR plugin                       |
| 99  | **Graph queries**                | ❌                                 | ❌                                        | ❌ (RedisGraph deprecated)           | ✅ Apache AGE extension                          | ❌                                         | ❌ (parent-child = limited graph-like relationships)                 |
| 100 | **Community & ecosystem size**   | Large (AWS-specific)               | Medium (AWS-specific)                     | Very large (open-source)             | Very large (largest open-source RDBMS community) | Very large (Apache foundation + Confluent) | Very large (Elastic + OpenSearch communities)                        |

---

## Quick Decision Guide

### When to pick each technology:

| Scenario                                        | Best choice                                   | Why                                                          |
| ----------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------ |
| Need single-digit ms CRUD at any scale          | **DynamoDB**                                  | Fully managed, auto-scaling, predictable latency             |
| Complex analytical SQL on TBs/PBs               | **Redshift**                                  | Columnar storage, parallel query, SQL interface              |
| Sub-millisecond caching / sessions / counters   | **Redis**                                     | In-memory, rich data structures, simplest to use             |
| Transactional app with complex queries + joins  | **PostgreSQL**                                | ACID, full SQL, extensions, most versatile                   |
| Real-time event streaming / decoupling services | **Kafka**                                     | High throughput, durable log, exactly-once, replay           |
| Full-text search / autocomplete / fuzzy match   | **Elasticsearch**                             | Inverted index, BM25 scoring, analyzers, highlighting        |
| Log aggregation & observability                 | **Elasticsearch** (ELK Stack)                 | Kibana dashboards, structured + unstructured log search      |
| User-facing product with mixed reads/writes     | **PostgreSQL** + **Redis** (cache)            | Postgres for source of truth, Redis for speed                |
| E-commerce with product search                  | **Kafka** → **DynamoDB** + **Elasticsearch**  | Kafka for events, DynamoDB for orders, ES for product search |
| Data lake ingestion from multiple sources       | **Kafka** → **Redshift** (or S3)              | Kafka Connect for ingestion, Redshift for analysis           |
| Leaderboard / rate limiting / real-time ranking | **Redis**                                     | Sorted Sets = purpose-built for this                         |
| Audit log / event sourcing                      | **Kafka**                                     | Immutable append-only log, replay from any point             |
| Time-series with analytics                      | **PostgreSQL** (TimescaleDB) or **Redshift**  | TimescaleDB for operational, Redshift for analytical         |
| Session store for web app                       | **Redis** or **DynamoDB**                     | Redis if you have it; DynamoDB if fully serverless           |
| CDC (database change capture)                   | **Kafka** + Debezium                          | Industry standard for streaming DB changes                   |
| Report generation from large dataset            | **Redshift**                                  | Columnar + SQL = fast aggregations                           |
| Chat / notifications / real-time features       | **Redis** Pub/Sub or Streams                  | Low latency, built-in pub/sub                                |
| Geospatial search (nearby stores, routes)       | **Elasticsearch** or **PostgreSQL** (PostGIS) | ES for fast geo queries at scale, PostGIS for precision      |
| Faceted navigation (filters + counts)           | **Elasticsearch**                             | Aggregations + filters = built for faceted search            |

---

## Common Architecture Patterns

### Pattern 1: CQRS (Command Query Responsibility Segregation)

```
Writes → PostgreSQL → CDC (Kafka) → Read models:
                                      ├── Redis (fast lookups)
                                      ├── Elasticsearch (search)
                                      ├── Redshift (analytics)
                                      └── DynamoDB (API reads at scale)
```

### Pattern 2: Event-Driven Microservices

```
Service A → Kafka topic → Service B → DynamoDB
                       → Service C → PostgreSQL
                       → Service D → Redis (cache invalidation)
                       → Service E → Elasticsearch (search index)
```

### Pattern 3: Real-Time Analytics Pipeline

```
App events → Kafka → Kafka Streams (aggregate) → Redshift (warehouse)
                                                → Redis (real-time dashboard)
                                                → Elasticsearch (log search)
```

### Pattern 4: E-Commerce Platform

```
User sessions   → Redis (fast, TTL-based)
Product catalog → PostgreSQL (complex queries, source of truth)
Product search  → Elasticsearch (full-text, faceted, autocomplete)
Shopping cart    → Redis or DynamoDB (fast, per-user)
Orders          → PostgreSQL → Kafka (CDC) → DynamoDB (read API)
Analytics       → Kafka → Redshift
Recommendations → PostgreSQL (pgvector) or Redis (vector similarity)
Logs & monitoring → Elasticsearch (ELK Stack)
```

### Pattern 5: Search-Powered Application

```
Source DB (PostgreSQL/MongoDB) → CDC (Kafka/Debezium) → Elasticsearch (search index)
                                                       ↓
User query → API Gateway → Search Service → Elasticsearch → Response
                                          → Redis (cache frequent queries)
```
