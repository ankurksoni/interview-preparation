# Amazon Athena Interview Questions & Answers

A curated list of **Amazon Athena interview questions** with practical, production-grade answers. Covers serverless SQL querying, partitioning, performance tuning, cost control, CDK/TypeScript examples, and real-world system design decisions.

---

### 1. **What is Amazon Athena?**

**Answer:** Athena is a **serverless, interactive query service** that lets you analyze data in S3 using standard SQL (Presto/Trino engine). No infrastructure to manage — you pay only per query based on data scanned.

**Key characteristics:**

- **Serverless** — no clusters to provision or manage
- **Pay-per-query** — $5 per TB of data scanned
- **Schema-on-read** — define schema at query time via Glue Data Catalog
- **Supports:** CSV, JSON, ORC, Parquet, Avro, and more
- **Engine:** Trino (v3) or PrestoDB (v2)

```ts
import { AthenaClient, StartQueryExecutionCommand } from "@aws-sdk/client-athena";

const athena = new AthenaClient({});

await athena.send(
  new StartQueryExecutionCommand({
    QueryString: "SELECT * FROM my_database.orders LIMIT 10",
    QueryExecutionContext: { Database: "my_database" },
    ResultConfiguration: {
      OutputLocation: "s3://my-athena-results/output/",
    },
  }),
);
```

---

### 2. **When should you use Athena vs when NOT to?**

**Answer:**

| Use Athena When | Don't Use Athena When |
| --- | --- |
| Ad-hoc querying on S3 data | Sub-second latency queries (use Redshift/DynamoDB) |
| Log analysis (ALB, CloudTrail, VPC flow logs) | Frequent repeated queries on same data (use Redshift) |
| Data lake exploration | Heavy write workloads (use RDS/DynamoDB) |
| Cost-sensitive infrequent queries | Real-time streaming analysis (use Kinesis Analytics) |
| One-off analytics & reporting | OLTP transactional workloads |
| Quick prototyping before Redshift | Complex multi-join queries at scale (use Redshift) |

> **System Design Tip:** If your team queries the same dataset repeatedly throughout the day, Redshift Serverless may be cheaper. Athena shines for infrequent, varied queries across large S3 datasets.

---

### 3. **How do you optimize Athena query performance?**

**Answer:** The #1 optimization is **reducing data scanned**:

**1. Use columnar formats (Parquet/ORC):**

```sql
-- Convert CSV to Parquet using CTAS
CREATE TABLE my_database.orders_parquet
WITH (format = 'PARQUET', external_location = 's3://bucket/orders-parquet/')
AS SELECT * FROM my_database.orders_csv;
```

**2. Partition your data:**

```sql
-- Partitioned table — Athena only scans relevant partitions
CREATE EXTERNAL TABLE logs (
  request_id STRING,
  status INT,
  response_time DOUBLE
)
PARTITIONED BY (year STRING, month STRING, day STRING)
STORED AS PARQUET
LOCATION 's3://my-logs-bucket/logs/';

-- Query only scans 1 day instead of entire dataset
SELECT * FROM logs WHERE year='2026' AND month='04' AND day='10';
```

**3. Use bucketing for frequently joined columns:**

```sql
CREATE TABLE orders_bucketed
WITH (bucketed_by = ARRAY['customer_id'], bucket_count = 64, format = 'PARQUET')
AS SELECT * FROM orders;
```

**Performance comparison:**

| Format | 1TB Dataset Scan | Cost | Query Time |
| --- | --- | --- | --- |
| Raw CSV | 1 TB | $5.00 | ~60s |
| Parquet (columnar) | ~100 GB | $0.50 | ~10s |
| Parquet + Partitioned | ~5 GB | $0.025 | ~2s |

---

### 4. **What are Athena's important config parameters?**

**Answer:**

| Parameter | Description | Production Recommendation |
| --- | --- | --- |
| `WorkGroup` | Isolates queries, budgets, settings per team | Always use — set per-query data scan limits |
| `OutputLocation` | S3 path for query results | Use lifecycle rules to auto-delete old results |
| `EngineVersion` | Trino (v3) or Presto (v2) | Use v3 (Trino) — better performance, more SQL features |
| `BytesScannedCutoffPerQuery` | Max bytes a query can scan | Set to prevent runaway costs (e.g., 10 GB) |
| `RequesterPaysEnabled` | Who pays for S3 requests | Enable for cross-account data access |
| `EnforceWorkGroupConfiguration` | Override client-side settings | Enable to enforce output location and encryption |

```ts
// CDK: Create a workgroup with cost controls
import * as athena from "aws-cdk-lib/aws-athena";

const workgroup = new athena.CfnWorkGroup(this, "AnalyticsWorkGroup", {
  name: "analytics-team",
  workGroupConfiguration: {
    engineVersion: { selectedEngineVersion: "Athena engine version 3" },
    resultConfiguration: {
      outputLocation: "s3://athena-results-bucket/analytics/",
      encryptionConfiguration: { encryptionOption: "SSE_S3" },
    },
    bytesScannedCutoffPerQuery: 10_737_418_240, // 10 GB limit
    enforceWorkGroupConfiguration: true,
    publishCloudWatchMetricsEnabled: true,
  },
});
```

---

### 5. **How do you connect to Athena from TypeScript and handle async query execution?**

**Answer:** Athena queries are **asynchronous** — you start execution, then poll for results.

```ts
import {
  AthenaClient,
  StartQueryExecutionCommand,
  GetQueryExecutionCommand,
  GetQueryResultsCommand,
} from "@aws-sdk/client-athena";

const athena = new AthenaClient({});

async function runQuery(sql: string): Promise<Record<string, string>[]> {
  // 1. Start query
  const { QueryExecutionId } = await athena.send(
    new StartQueryExecutionCommand({
      QueryString: sql,
      QueryExecutionContext: { Database: "analytics" },
      WorkGroup: "analytics-team",
    }),
  );

  // 2. Poll until complete
  let state = "RUNNING";
  while (state === "RUNNING" || state === "QUEUED") {
    await new Promise((r) => setTimeout(r, 1000));
    const exec = await athena.send(
      new GetQueryExecutionCommand({ QueryExecutionId }),
    );
    state = exec.QueryExecution?.Status?.State ?? "FAILED";
  }

  if (state !== "SUCCEEDED") throw new Error(`Query failed: ${state}`);

  // 3. Fetch results (paginated)
  const rows: Record<string, string>[] = [];
  let nextToken: string | undefined;

  do {
    const result = await athena.send(
      new GetQueryResultsCommand({ QueryExecutionId, NextToken: nextToken }),
    );
    const columns = result.ResultSet?.ResultSetMetadata?.ColumnInfo ?? [];
    const dataRows = result.ResultSet?.Rows ?? [];

    for (const row of dataRows.slice(nextToken ? 0 : 1)) { // skip header on first page
      const record: Record<string, string> = {};
      row.Data?.forEach((cell, i) => {
        record[columns[i].Name!] = cell.VarCharValue ?? "";
      });
      rows.push(record);
    }
    nextToken = result.NextToken;
  } while (nextToken);

  return rows;
}

const results = await runQuery(
  "SELECT status, COUNT(*) as cnt FROM logs WHERE year='2026' GROUP BY status",
);
```

> **Hidden Gem:** Use `ResultReuseConfiguration` in workgroup settings to cache and reuse previous query results (up to 60 min). This avoids re-scanning data and is free.

---

### 6. **How does Athena handle partitioning with Glue Data Catalog?**

**Answer:** Athena uses **AWS Glue Data Catalog** as its metastore. Partitions map to S3 prefixes.

```
s3://logs-bucket/logs/year=2026/month=04/day=10/data.parquet
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                       These become partition columns
```

**Three ways to manage partitions:**

```sql
-- 1. Manual: Add partition explicitly
ALTER TABLE logs ADD PARTITION (year='2026', month='04', day='10')
LOCATION 's3://logs-bucket/logs/year=2026/month=04/day=10/';

-- 2. MSCK REPAIR: Scan S3 and add all partitions (slow for many partitions)
MSCK REPAIR TABLE logs;

-- 3. Partition projection (BEST for production):
-- Athena calculates partitions at query time — no Glue Catalog updates needed
CREATE EXTERNAL TABLE logs (
  request_id STRING, status INT
)
PARTITIONED BY (year STRING, month STRING, day STRING)
STORED AS PARQUET
LOCATION 's3://logs-bucket/logs/'
TBLPROPERTIES (
  'projection.enabled' = 'true',
  'projection.year.type' = 'integer',
  'projection.year.range' = '2020,2030',
  'projection.month.type' = 'integer',
  'projection.month.range' = '1,12',
  'projection.month.digits' = '2',
  'projection.day.type' = 'integer',
  'projection.day.range' = '1,31',
  'projection.day.digits' = '2',
  'storage.location.template' = 's3://logs-bucket/logs/year=${year}/month=${month}/day=${day}/'
);
```

> **Hidden Property:** **Partition projection** eliminates `MSCK REPAIR TABLE` entirely. It's faster (no Glue API calls) and handles millions of partitions without catalog bloat. **Always use this in production for time-series data.**

---

### 7. **What are Athena's cost optimization strategies?**

**Answer:**

| Strategy | Savings | Effort |
| --- | --- | --- |
| Parquet/ORC instead of CSV | 80–90% | Medium (ETL conversion) |
| Partition pruning | 90–99% | Low (organize S3 paths) |
| Columnar projection (SELECT specific cols) | 50–80% | Zero |
| Workgroup scan limits | Prevents runaway costs | Low |
| Result reuse (caching) | 100% on cache hit | Low |
| Compress data (Snappy/ZSTD) | 50–70% | Low |

```sql
-- ❌ Bad: Full table scan, all columns
SELECT * FROM logs;

-- ✅ Good: Specific columns, partitioned, filtered
SELECT request_id, status FROM logs
WHERE year = '2026' AND month = '04' AND status = 500;
```

> **Production Tip:** Set `BytesScannedCutoffPerQuery` on every workgroup. A single `SELECT *` on an unpartitioned TB-scale table can cost $5+ in one query.

---

### 8. **How do you set up Athena with CDK for a production data lake?**

**Answer:**

```ts
import * as cdk from "aws-cdk-lib";
import * as s3 from "aws-cdk-lib/aws-s3";
import * as glue from "aws-cdk-lib/aws-glue";
import * as athena from "aws-cdk-lib/aws-athena";

// Data lake bucket
const dataBucket = new s3.Bucket(this, "DataLake", {
  bucketName: "company-data-lake",
  encryption: s3.BucketEncryption.S3_MANAGED,
  lifecycleRules: [
    { transition: [{ storageClass: s3.StorageClass.INTELLIGENT_TIERING, transitionAfter: cdk.Duration.days(90) }] },
  ],
});

// Results bucket with auto-cleanup
const resultsBucket = new s3.Bucket(this, "AthenaResults", {
  lifecycleRules: [{ expiration: cdk.Duration.days(7) }],
});

// Glue database
const database = new glue.CfnDatabase(this, "AnalyticsDB", {
  catalogId: cdk.Aws.ACCOUNT_ID,
  databaseInput: { name: "analytics" },
});

// Glue table with partition projection
new glue.CfnTable(this, "EventsTable", {
  catalogId: cdk.Aws.ACCOUNT_ID,
  databaseName: "analytics",
  tableInput: {
    name: "events",
    tableType: "EXTERNAL_TABLE",
    parameters: {
      "projection.enabled": "true",
      "projection.dt.type": "date",
      "projection.dt.range": "2024-01-01,NOW",
      "projection.dt.format": "yyyy-MM-dd",
      "projection.dt.interval": "1",
      "projection.dt.interval.unit": "DAYS",
      "storage.location.template": `s3://${dataBucket.bucketName}/events/dt=\${dt}/`,
    },
    storageDescriptor: {
      location: `s3://${dataBucket.bucketName}/events/`,
      inputFormat: "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat",
      outputFormat: "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat",
      serdeInfo: { serializationLibrary: "org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe" },
      columns: [
        { name: "event_id", type: "string" },
        { name: "user_id", type: "string" },
        { name: "event_type", type: "string" },
        { name: "payload", type: "string" },
      ],
    },
    partitionKeys: [{ name: "dt", type: "string" }],
  },
});

// Workgroup with cost controls
new athena.CfnWorkGroup(this, "WorkGroup", {
  name: "analytics-team",
  workGroupConfiguration: {
    engineVersion: { selectedEngineVersion: "Athena engine version 3" },
    resultConfiguration: {
      outputLocation: `s3://${resultsBucket.bucketName}/`,
      encryptionConfiguration: { encryptionOption: "SSE_S3" },
    },
    bytesScannedCutoffPerQuery: 10_737_418_240, // 10 GB
    enforceWorkGroupConfiguration: true,
    publishCloudWatchMetricsEnabled: true,
  },
});
```

---

### 9. **What are Athena's resiliency and robustness patterns?**

**Answer:**

**Query retry strategy:**

```ts
async function resilientQuery(sql: string, maxRetries = 3): Promise<any[]> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await runQuery(sql);
    } catch (err: any) {
      if (err.name === "TooManyRequestsException" && attempt < maxRetries) {
        // Exponential backoff for throttling
        await new Promise((r) => setTimeout(r, 2 ** attempt * 1000));
        continue;
      }
      throw err;
    }
  }
  return [];
}
```

**Key resiliency considerations:**

| Concern | Solution |
| --- | --- |
| S3 data corruption | Enable S3 versioning on data bucket |
| Query failures | Implement retry with exponential backoff |
| Throttling | Use multiple workgroups to isolate traffic |
| Catalog outage | Partition projection bypasses Glue catalog |
| Schema evolution | Use Parquet (supports column additions) |
| Cross-region DR | Replicate S3 data + Glue catalog export |

> **Hidden Gem:** Athena queries using **partition projection** continue to work even if Glue Data Catalog is throttled, because partition resolution happens locally in Athena.

---

### 10. **How does Athena integrate with other AWS services?**

**Answer:**

```
S3 (source) → Glue Crawler → Glue Catalog → Athena → QuickSight
    ↑                                          ↓
Kinesis Firehose                         S3 (results) → Lambda (post-processing)
```

**Common integration patterns:**

- **CloudTrail analysis:** Query CloudTrail JSON logs directly in S3
- **ALB access logs:** Create Athena table over ALB log bucket
- **VPC Flow Logs:** Analyze network traffic with SQL
- **Kinesis Firehose → S3 → Athena:** Near real-time analytics pipeline
- **Step Functions → Athena:** Orchestrate multi-query ETL workflows
- **Lambda → Athena:** Trigger scheduled queries and process results

```ts
// Lambda: Run scheduled Athena query and send results to SNS
import { SNSClient, PublishCommand } from "@aws-sdk/client-sns";

export const handler = async () => {
  const results = await runQuery(`
    SELECT status, COUNT(*) as error_count
    FROM alb_logs WHERE dt = current_date - interval '1' day
    AND status >= 500 GROUP BY status
  `);

  if (results.length > 0) {
    const sns = new SNSClient({});
    await sns.send(
      new PublishCommand({
        TopicArn: process.env.ALERT_TOPIC_ARN,
        Subject: "Daily Error Report",
        Message: JSON.stringify(results, null, 2),
      }),
    );
  }
};
```

---

### 11. **What are Athena's scalability limits and how to work around them?**

**Answer:**

| Limit | Value | Workaround |
| --- | --- | --- |
| Concurrent queries per account | 25 (DML), 20 (DDL) | Use multiple workgroups / accounts |
| Query timeout | 30 minutes | Break large queries into smaller CTASes |
| Result size | 2 GB uncompressed | Use CTAS to write results to S3 directly |
| Partitions per table | Unlimited with projection, 20M in Glue | Always use partition projection |
| Databases per catalog | 10,000 | Organize with clear naming conventions |
| Query string size | 256 KB | Use views for complex reusable logic |

**Scaling pattern for high-concurrency analytics:**

```ts
// ❌ Bad: All teams share one workgroup → throttling
// ✅ Good: Separate workgroups per team with individual limits

const teams = ["data-eng", "product", "finance"];

teams.forEach((team) => {
  new athena.CfnWorkGroup(this, `WG-${team}`, {
    name: team,
    workGroupConfiguration: {
      bytesScannedCutoffPerQuery: team === "finance" ? 1_073_741_824 : 10_737_418_240,
      enforceWorkGroupConfiguration: true,
    },
  });
});
```

---

### 12. **What are CTAS and INSERT INTO, and when do you use them?**

**Answer:** **CTAS** (Create Table As Select) creates a new table from query results. **INSERT INTO** appends to an existing table.

```sql
-- CTAS: Create optimized table from raw data (one-time ETL)
CREATE TABLE analytics.orders_optimized
WITH (
  format = 'PARQUET',
  partitioned_by = ARRAY['order_date'],
  bucketed_by = ARRAY['customer_id'],
  bucket_count = 64,
  external_location = 's3://data-lake/orders-optimized/'
)
AS SELECT order_id, customer_id, amount, status,
   date_format(created_at, '%Y-%m-%d') as order_date
FROM raw.orders;

-- INSERT INTO: Incremental daily load
INSERT INTO analytics.orders_optimized
SELECT order_id, customer_id, amount, status,
   date_format(created_at, '%Y-%m-%d') as order_date
FROM raw.orders
WHERE created_at >= current_date - interval '1' day;
```

> **Production Tip:** CTAS queries have a 100-partition limit per query. For large datasets, use multiple INSERT INTO statements or AWS Glue ETL instead.

---

### 13. **What hidden/lesser-known Athena features are useful in production?**

**Answer:**

| Feature | What It Does | Why It Matters |
| --- | --- | --- |
| **Partition projection** | Calculate partitions at query time | Eliminates MSCK REPAIR, faster queries |
| **Result reuse** | Cache query results up to 60 min | Free repeated queries |
| **Prepared statements** | Parameterized queries | Prevent SQL injection, reuse plans |
| **Federated queries** | Query RDS, DynamoDB, Redis via Lambda connectors | Single SQL across multiple sources |
| **EXPLAIN** | Show query execution plan | Debug slow queries |
| **UNLOAD** | Export results in Parquet/ORC/JSON to S3 | Better than CSV for downstream use |
| **Workgroup tags** | Cost allocation tags on workgroups | Track costs per team/project |

```sql
-- Prepared statement: Prevent SQL injection
PREPARE my_query FROM
SELECT * FROM orders WHERE customer_id = ? AND order_date = ?;

EXECUTE my_query USING 'cust-123', '2026-04-10';

-- EXPLAIN: Debug query plan
EXPLAIN (TYPE DISTRIBUTED)
SELECT customer_id, SUM(amount)
FROM orders WHERE year = '2026' GROUP BY customer_id;

-- UNLOAD: Export results as Parquet
UNLOAD (SELECT * FROM orders WHERE year = '2026')
TO 's3://export-bucket/orders/'
WITH (format = 'PARQUET', compression = 'SNAPPY');
```

---

### 14. **How do you monitor and troubleshoot Athena queries?**

**Answer:**

```ts
// CDK: CloudWatch alarm for expensive queries
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";

const bytesScannedMetric = new cloudwatch.Metric({
  namespace: "AWS/Athena",
  metricName: "ProcessedBytes",
  dimensionsMap: { WorkGroup: "analytics-team" },
  statistic: "Sum",
  period: cdk.Duration.hours(1),
});

new cloudwatch.Alarm(this, "HighScanAlarm", {
  metric: bytesScannedMetric,
  threshold: 107_374_182_400, // 100 GB per hour
  evaluationPeriods: 1,
  alarmDescription: "Athena team scanning too much data — check queries",
});
```

**Troubleshooting checklist:**

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Query very slow | Scanning too much data | Add partitions, use Parquet |
| `HIVE_PARTITION_SCHEMA_MISMATCH` | Schema drift across partitions | Standardize schema in Glue |
| `GENERIC_INTERNAL_ERROR` | Corrupt files in S3 | Identify and replace bad files |
| Query cancelled | Hit `BytesScannedCutoff` | Optimize query or increase limit |
| Throttled (429) | Too many concurrent queries | Use workgroups, add retry logic |
| Empty results | Wrong partition values | Check S3 paths match partition columns |

---

### 15. **How does Athena compare with Redshift and Redshift Spectrum for system design?**

**Answer:**

| Criteria | Athena | Redshift | Redshift Spectrum |
| --- | --- | --- | --- |
| **Setup** | Zero — serverless | Provision cluster | Needs Redshift cluster |
| **Pricing** | $5/TB scanned | Per-node hourly | $5/TB scanned + cluster cost |
| **Latency** | Seconds to minutes | Sub-second (warm) | Seconds |
| **Concurrency** | 25 queries/account | 50+ with scaling | Shares Redshift concurrency |
| **Data location** | S3 only | Local SSD + S3 | S3 (cold) + local (hot) |
| **Best for** | Ad-hoc, exploration | Repeated dashboards, BI | Hot/cold data separation |
| **Scaling** | Automatic | Manual (resize) or serverless | Automatic for S3 scans |

**Decision framework:**

```
Infrequent queries on S3 data? → Athena
Frequent dashboards + BI? → Redshift Serverless
Mix of hot + cold data? → Redshift + Spectrum
Real-time streaming → Kinesis Analytics
Sub-10ms lookups → DynamoDB
```
