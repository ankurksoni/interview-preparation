# Amazon Redshift Interview Questions & Answers

A curated list of **Amazon Redshift interview questions** with practical, production-grade answers. Covers architecture, data modeling, query optimization, Redshift Serverless, Spectrum, data sharing, and real-world data warehouse patterns.

---

### 1. **What is Amazon Redshift?**

**Answer:** Redshift is a fully managed, petabyte-scale cloud data warehouse. It uses columnar storage, massively parallel processing (MPP), and automatic compression to deliver fast SQL analytics over large datasets. It's designed for OLAP (Online Analytical Processing), not OLTP.

---

### 2. **What is the difference between Redshift Provisioned and Redshift Serverless?**

**Answer:**

| Feature    | Provisioned             | Serverless                          |
| ---------- | ----------------------- | ----------------------------------- |
| Capacity   | Fixed node count        | Auto-scales                         |
| Pricing    | Per-node per-hour       | Per RPU-hour (pay for compute used) |
| Management | Choose node type, count | Fully managed                       |
| Idle cost  | Pay even when idle      | Minimal when idle                   |
| Use case   | Predictable workloads   | Variable/bursty workloads           |

For new projects, Redshift Serverless is recommended unless you have highly predictable workloads and need cost optimization through reserved instances.

---

### 3. **How does Redshift store data internally?**

**Answer:** Redshift uses **columnar storage**. Each column is stored separately on disk, which:

- Reduces I/O for analytical queries (reads only needed columns)
- Enables high compression ratios (similar values stored together)
- Supports zone maps (min/max per block) for automatic data skipping

```sql
-- This only reads the 'amount' and 'order_date' columns from disk
SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS revenue
FROM orders
WHERE order_date >= '2025-01-01'
GROUP BY 1
ORDER BY 1;
```

---

### 4. **What are distribution styles and how do you choose one?**

**Answer:** Distribution style determines how Redshift spreads rows across compute nodes.

| Style  | Behavior                           | Best for                           |
| ------ | ---------------------------------- | ---------------------------------- |
| `KEY`  | Rows with same key go to same node | Large tables joined on that key    |
| `EVEN` | Round-robin distribution           | Tables not used in joins           |
| `ALL`  | Full copy on every node            | Small dimension tables (< 5M rows) |
| `AUTO` | Redshift decides dynamically       | Default, good starting point       |

```sql
-- ❌ Bad: Large fact table with EVEN distribution — causes data shuffling on joins
CREATE TABLE orders (
  order_id BIGINT,
  customer_id BIGINT,
  amount DECIMAL(10,2)
) DISTSTYLE EVEN;

-- ✅ Good: Distribute on join key to co-locate data
CREATE TABLE orders (
  order_id BIGINT,
  customer_id BIGINT,
  amount DECIMAL(10,2)
) DISTSTYLE KEY DISTKEY(customer_id);

CREATE TABLE customers (
  customer_id BIGINT,
  name VARCHAR(200)
) DISTSTYLE KEY DISTKEY(customer_id);

-- Now JOIN on customer_id is local to each node — no shuffling
SELECT c.name, SUM(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
GROUP BY 1;
```

---

### 5. **What are sort keys and how do they affect performance?**

**Answer:** Sort keys determine the physical order of data on disk. They enable zone-map-based data skipping.

| Type          | Behavior                       | Use case                                  |
| ------------- | ------------------------------ | ----------------------------------------- |
| `COMPOUND`    | Sorts by columns in order      | Queries always filter by the first column |
| `INTERLEAVED` | Equal weight to each column    | Queries filter by any column combination  |
| `AUTO`        | Redshift manages automatically | Default, recommended for most cases       |

```sql
-- ✅ Compound sort key: best when queries always filter by order_date first
CREATE TABLE orders (
  order_id BIGINT,
  order_date DATE,
  customer_id BIGINT,
  amount DECIMAL(10,2)
)
DISTSTYLE KEY DISTKEY(customer_id)
COMPOUND SORTKEY(order_date, customer_id);

-- This query benefits from the sort key (skips blocks outside date range)
SELECT * FROM orders
WHERE order_date BETWEEN '2025-01-01' AND '2025-03-31'
  AND customer_id = 12345;
```

---

### 6. **How do you load data into Redshift efficiently?**

**Answer:** Use the `COPY` command for bulk loading — it's parallelized across all nodes.

```sql
-- ✅ Best practice: COPY from S3 with multiple files
COPY orders
FROM 's3://my-bucket/data/orders/'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-s3-role'
FORMAT AS PARQUET;

-- For CSV:
COPY orders
FROM 's3://my-bucket/data/orders/'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-s3-role'
CSV
IGNOREHEADER 1
DATEFORMAT 'YYYY-MM-DD'
MAXERROR 100
COMPUPDATE ON;
```

**Loading best practices:**

- Split files into multiple parts (Redshift loads them in parallel)
- Use Parquet or ORC over CSV (columnar, compressed, typed)
- Use COMPUPDATE ON for first load (auto-selects compression)
- Use a staging table → merge pattern for incremental loads

---

### 7. **How do you implement incremental (upsert) loads?**

**Answer:** Redshift doesn't support native `UPSERT`. Use a staging → merge → swap pattern.

```sql
-- Step 1: Load new data into staging table
CREATE TEMP TABLE orders_staging (LIKE orders);

COPY orders_staging
FROM 's3://my-bucket/data/orders-delta/'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-s3-role'
FORMAT AS PARQUET;

-- Step 2: Delete existing rows that will be replaced
DELETE FROM orders
USING orders_staging
WHERE orders.order_id = orders_staging.order_id;

-- Step 3: Insert new/updated rows
INSERT INTO orders
SELECT * FROM orders_staging;

-- Step 4: Clean up
DROP TABLE orders_staging;
```

Alternatively, use `MERGE` (available in Redshift since late 2023):

```sql
MERGE INTO orders
USING orders_staging AS s
ON orders.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET amount = s.amount, status = s.status
WHEN NOT MATCHED THEN INSERT VALUES (s.order_id, s.order_date, s.customer_id, s.amount, s.status);
```

---

### 8. **What is Redshift Spectrum and when should you use it?**

**Answer:** Spectrum lets you query data directly in S3 without loading it into Redshift. Uses external tables.

```sql
-- Create external schema pointing to AWS Glue Data Catalog
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'my_data_lake'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-spectrum-role';

-- Query S3 data directly (no COPY needed)
SELECT DATE_TRUNC('month', event_date) AS month, COUNT(*) AS events
FROM spectrum_schema.clickstream_events
WHERE event_date >= '2025-01-01'
GROUP BY 1;

-- Join Redshift local tables with S3 Spectrum tables
SELECT c.name, COUNT(e.event_id) AS clicks
FROM customers c
JOIN spectrum_schema.clickstream_events e ON c.customer_id = e.user_id
GROUP BY 1
ORDER BY 2 DESC
LIMIT 20;
```

**Use Spectrum for:** historical/cold data, data exploration, data lake queries.
**Use Redshift local for:** hot data, frequently joined dimensions, low-latency dashboards.

---

### 9. **What is Redshift Data Sharing?**

**Answer:** Data Sharing lets you share live data across Redshift clusters and accounts without copying data.

```sql
-- Producer cluster: create a datashare
CREATE DATASHARE sales_share;
ALTER DATASHARE sales_share ADD SCHEMA public;
ALTER DATASHARE sales_share ADD TABLE public.orders;
ALTER DATASHARE sales_share ADD TABLE public.customers;

-- Grant access to consumer
GRANT USAGE ON DATASHARE sales_share TO NAMESPACE 'consumer-namespace-id';

-- Consumer cluster: create a database from the datashare
CREATE DATABASE sales_db FROM DATASHARE sales_share OF NAMESPACE 'producer-namespace-id';

-- Query shared data (always up-to-date)
SELECT * FROM sales_db.public.orders WHERE order_date = CURRENT_DATE;
```

---

### 10. **How do you optimize slow queries in Redshift?**

**Answer:** Use `EXPLAIN` and system tables to diagnose:

```sql
-- Check query plan
EXPLAIN
SELECT c.name, SUM(o.amount)
FROM orders o JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= '2025-01-01'
GROUP BY 1;
```

**Common issues and fixes:**

| Problem             | Symptom in EXPLAIN                        | Fix                                     |
| ------------------- | ----------------------------------------- | --------------------------------------- |
| Data redistribution | `DS_BCAST_INNER` or `DS_DIST_BOTH`        | Use matching DISTKEY on join columns    |
| Full table scan     | Large `rows` count, no zone map filtering | Add/fix SORTKEY on filter columns       |
| Disk-based query    | `maxtime` in minutes                      | Increase cluster size or optimize query |
| Stale statistics    | Suboptimal plan                           | Run `ANALYZE table_name`                |
| Unsorted data       | Zone maps ineffective                     | Run `VACUUM SORT ONLY table_name`       |

```sql
-- Maintenance queries
ANALYZE orders;              -- Update statistics
VACUUM SORT ONLY orders;     -- Re-sort data
VACUUM DELETE ONLY orders;   -- Reclaim space from deleted rows
```

---

### 11. **What is the difference between VACUUM and ANALYZE?**

**Answer:**

- **VACUUM** — Reclaims disk space from deleted rows and re-sorts data. Types: `FULL`, `SORT ONLY`, `DELETE ONLY`, `REINDEX`.
- **ANALYZE** — Updates table statistics used by the query planner. Run after significant data changes.

> **Note:** Redshift runs automatic vacuum and analyze in the background, but for large batch loads you should run them manually.

---

### 12. **How do you implement slowly changing dimensions (SCD Type 2) in Redshift?**

**Answer:**

```sql
CREATE TABLE dim_customers (
  customer_key BIGINT IDENTITY(1,1),  -- Surrogate key
  customer_id BIGINT,                  -- Business key
  name VARCHAR(200),
  email VARCHAR(200),
  tier VARCHAR(50),
  effective_date DATE,
  end_date DATE DEFAULT '9999-12-31',
  is_current BOOLEAN DEFAULT TRUE
)
DISTSTYLE ALL
SORTKEY(customer_id, is_current);

-- Close the current record and insert new one
BEGIN;

UPDATE dim_customers
SET end_date = CURRENT_DATE - 1, is_current = FALSE
WHERE customer_id = 12345 AND is_current = TRUE;

INSERT INTO dim_customers (customer_id, name, email, tier, effective_date)
VALUES (12345, 'Alice Smith', 'alice@new-email.com', 'Gold', CURRENT_DATE);

COMMIT;
```

---

### 13. **How do you handle concurrent writes in Redshift?**

**Answer:** Redshift uses **serializable isolation** by default and has table-level locks. It's not designed for high-concurrency OLTP writes.

**Best practices:**

- Batch inserts (use `COPY`, not individual `INSERT` statements)
- Avoid long-running transactions
- Use WLM (Workload Management) queues to separate ETL from analytics
- For high-concurrency reads, use concurrency scaling (auto-adds clusters)

---

### 14. **What is Workload Management (WLM)?**

**Answer:** WLM manages query queues, assigning resources based on query type. Use it to prevent heavy ETL jobs from starving dashboard queries.

```sql
-- Check current WLM configuration
SELECT * FROM stv_wlm_service_class_config;

-- Monitor running and queued queries
SELECT * FROM stv_wlm_query_state;
```

**Recommended setup:**

- Queue 1: Dashboard queries (high priority, 50% memory, short timeout)
- Queue 2: ETL/batch jobs (lower priority, 40% memory)
- Queue 3: Ad-hoc/superuser (10% memory)

---

### 15. **What are materialized views in Redshift?**

**Answer:** Materialized views pre-compute and store query results. Redshift can auto-refresh them and use them for automatic query rewriting.

```sql
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT
  DATE_TRUNC('day', order_date) AS day,
  region,
  COUNT(*) AS order_count,
  SUM(amount) AS total_revenue
FROM orders
GROUP BY 1, 2;

-- Auto-refresh when base table changes
ALTER MATERIALIZED VIEW mv_daily_revenue AUTO REFRESH YES;

-- This query is automatically rewritten to use the MV
SELECT day, total_revenue
FROM orders
WHERE region = 'US'
GROUP BY DATE_TRUNC('day', order_date);
```

---

### 16. **How do you unload data from Redshift to S3?**

**Answer:**

```sql
UNLOAD ('SELECT * FROM orders WHERE order_date >= ''2025-01-01''')
TO 's3://my-bucket/exports/orders_2025/'
IAM_ROLE 'arn:aws:iam::123456789:role/redshift-s3-role'
FORMAT AS PARQUET
PARTITION BY (region)
ALLOWOVERWRITE;
```

This creates partitioned Parquet files in S3, queryable by Athena, Spectrum, or Spark.

---

### 17. **What is the difference between Redshift and Athena?**

**Answer:**

| Feature      | Redshift                                 | Athena                             |
| ------------ | ---------------------------------------- | ---------------------------------- |
| Architecture | Provisioned/serverless cluster           | Serverless, query engine only      |
| Storage      | Managed columnar storage + S3 (Spectrum) | S3 only                            |
| Performance  | Very fast (local storage, caching)       | Good, but slower for complex joins |
| Pricing      | Per-node or per-RPU                      | Per-query ($5/TB scanned)          |
| Concurrency  | High (concurrency scaling)               | Limited (default 20 concurrent)    |
| Use case     | Primary data warehouse                   | Ad-hoc queries on data lake        |

**Use Redshift** when: you have a dedicated analytics platform with complex reporting.
**Use Athena** when: you need occasional queries on S3 data without infrastructure.

---

### 18. **How do you secure data in Redshift?**

**Answer:**

- **Network** — Deploy in VPC, use security groups, no public access
- **Encryption at rest** — AES-256 with AWS KMS or HSM
- **Encryption in transit** — SSL/TLS connections enforced via parameter group
- **Access control** — Database users, groups, and `GRANT`/`REVOKE` on schemas, tables, columns
- **Row-level security** — RLS policies restrict which rows users can see
- **Column-level security** — `GRANT SELECT(column_name)` on specific columns

```sql
-- Row-level security: users can only see their own region's data
CREATE RLS POLICY region_filter
WITH (region VARCHAR(50))
USING (region = current_setting('app.user_region'));

ATTACH RLS POLICY region_filter ON orders TO analyst_role;
ALTER TABLE orders ROW LEVEL SECURITY ON;
```

---

### 19. **What are zero-ETL integrations?**

**Answer:** Zero-ETL integrations replicate data from operational databases (Aurora MySQL, Aurora PostgreSQL, RDS) directly into Redshift without building ETL pipelines.

Data flows continuously from the source to Redshift in near real-time. This eliminates the need for custom ETL jobs, Glue jobs, or CDC pipelines for many analytics use cases.

```sql
-- Query replicated Aurora data directly in Redshift
SELECT * FROM aurora_integration_db.public.orders
WHERE created_at >= CURRENT_DATE - INTERVAL '1 day';
```

---

### 20. **How do you estimate Redshift costs?**

**Answer:**

**Provisioned:**

- Node costs (on-demand or reserved): e.g., `ra3.xlplus` ~$1.086/hr
- Spectrum queries: $5/TB scanned from S3
- Concurrency scaling: per-second billing for additional clusters
- Storage: included with ra3 nodes (managed storage)

**Serverless:**

- RPU-hours: $0.375/RPU-hour (min 8 RPU)
- No charge when idle (after cooldown period)

**Cost optimization tips:**

- Use reserved nodes for predictable workloads (up to 75% savings)
- Compress data (Redshift auto-selects compression)
- Partition Spectrum data for minimal scanning
- Use materialized views to avoid re-computing heavy queries

---

## Practical System Design Supplement

---

### 21. **How do you provision a Redshift cluster with CDK?**

**Answer:**

```ts
import * as redshift from "aws-cdk-lib/aws-redshift";
import * as ec2 from "aws-cdk-lib/aws-ec2";

// Provisioned cluster
const cluster = new redshift.Cluster(this, "DataWarehouse", {
  masterUser: { masterUsername: "admin" },
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
  nodeType: redshift.NodeType.RA3_XLPLUS,
  numberOfNodes: 2,
  defaultDatabaseName: "analytics",
  encrypted: true,
  enhancedVpcRouting: true, // All traffic through VPC (auditing)
  publiclyAccessible: false,
  preferredMaintenanceWindow: "sun:05:00-sun:06:00",
});

// For serverless (via CfnWorkgroup):
const namespace = new redshift.CfnNamespace(this, "Namespace", {
  namespaceName: "analytics",
  adminUsername: "admin",
  adminUserPassword: secretValue,
  dbName: "analytics",
});

const workgroup = new redshift.CfnWorkgroup(this, "Workgroup", {
  workgroupName: "analytics",
  namespaceName: "analytics",
  baseCapacity: 32, // RPUs (8–512, in units of 8)
  publiclyAccessible: false,
  subnetIds: vpc.selectSubnets({ subnetType: ec2.SubnetType.PRIVATE_ISOLATED })
    .subnetIds,
});
```

---

### 22. **How do you query Redshift from TypeScript?**

**Answer:**

```ts
import {
  RedshiftDataClient,
  ExecuteStatementCommand,
  DescribeStatementCommand,
  GetStatementResultCommand,
} from "@aws-sdk/client-redshift-data";

const client = new RedshiftDataClient({});

async function queryRedshift(sql: string): Promise<any[]> {
  // 1. Execute (async)
  const { Id } = await client.send(
    new ExecuteStatementCommand({
      WorkgroupName: "analytics", // Serverless
      Database: "analytics",
      Sql: sql,
    }),
  );

  // 2. Poll for completion
  let status = "SUBMITTED";
  while (["SUBMITTED", "PICKED", "STARTED"].includes(status)) {
    await new Promise((r) => setTimeout(r, 1000));
    const desc = await client.send(new DescribeStatementCommand({ Id }));
    status = desc.Status!;
    if (status === "FAILED") throw new Error(desc.Error);
  }

  // 3. Fetch results
  const result = await client.send(new GetStatementResultCommand({ Id }));
  return result.Records!.map((row) =>
    Object.fromEntries(
      row.map((col, i) => [
        result.ColumnMetadata![i].name,
        Object.values(col)[0],
      ]),
    ),
  );
}

const topCustomers = await queryRedshift(`
  SELECT customer_id, SUM(amount) as total_spend
  FROM orders WHERE order_date >= DATEADD(month, -3, GETDATE())
  GROUP BY customer_id ORDER BY total_spend DESC LIMIT 10
`);
```

> **Hidden Gem:** The Redshift Data API is serverless and doesn't require a persistent connection or VPC access. Use it from Lambda instead of maintaining connection pools.

---

### 23. **When should you use Redshift vs Redshift Serverless vs Athena?**

**Answer:**

| Criteria          | Redshift Provisioned            | Redshift Serverless          | Athena                |
| ----------------- | ------------------------------- | ---------------------------- | --------------------- |
| **Best for**      | Predictable, heavy BI workloads | Variable analytics workloads | Ad-hoc queries on S3  |
| **Pricing**       | Per-node-hour                   | Per RPU-hour (idle = free)   | Per TB scanned        |
| **Min cost**      | ~$180/mo (dc2.large)            | $0 when idle                 | $0 when idle          |
| **Concurrency**   | 50+ with scaling                | Auto-scales                  | 25 concurrent queries |
| **Latency**       | Sub-second (warm)               | Seconds (cold start)         | Seconds to minutes    |
| **Setup**         | High (cluster config)           | Medium                       | Zero                  |
| **Complex joins** | Excellent                       | Excellent                    | Limited at scale      |

---

### 24. **What are Redshift's hidden/useful features for production?**

**Answer:**

| Feature                    | What It Does                                    | Why It Matters                      |
| -------------------------- | ----------------------------------------------- | ----------------------------------- |
| **Data Sharing**           | Share live data across clusters without copying | Multi-team without ETL              |
| **AQUA**                   | Hardware-accelerated query processing           | 10x faster for scan-heavy queries   |
| **Materialized views**     | Auto-refresh precomputed results                | Dashboard performance               |
| **Concurrency scaling**    | Auto-add clusters for burst reads               | No throttling during peak           |
| **Federated queries**      | Query RDS/Aurora from Redshift SQL              | Join transactional + analytics data |
| **Streaming ingestion**    | Ingest from Kinesis/MSK directly                | Near-real-time analytics            |
| **Query monitoring rules** | Auto-kill expensive queries                     | Prevent runaway resources           |
| **Automatic table tuning** | Auto-selects distribution/sort keys             | Optimization without manual effort  |

> **Hidden Gem:** `AUTO` distribution style and sort key let Redshift automatically optimize data layout based on query patterns. Use them unless you have strong reasons for manual settings.
