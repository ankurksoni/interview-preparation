# PostgreSQL Interview Questions & Answers

A curated list of **PostgreSQL interview questions** — easy to medium difficulty with practical examples. Covers fundamentals, indexing, transactions, JSON, CTEs, performance tuning, replication, and real-world patterns.

---

## Fundamentals

### 1. **What is PostgreSQL?**

**Answer:** PostgreSQL (Postgres) is an open-source, **relational (RDBMS)** database known for standards compliance, extensibility, and reliability. It supports SQL, JSON, full-text search, geospatial (PostGIS), and custom types.

Key highlights:

- **ACID compliant** with MVCC (Multi-Version Concurrency Control)
- **Extensible** — custom types, operators, index methods, languages
- **Rich SQL** — CTEs, window functions, lateral joins, recursive queries
- **JSON/JSONB** — Document storage within a relational DB
- **Mature replication** — Streaming, logical, and physical

---

### 2. **What are the basic data types in PostgreSQL?**

**Answer:**

| Category  | Types                                                                                    |
| --------- | ---------------------------------------------------------------------------------------- |
| Numeric   | `smallint`, `integer`, `bigint`, `decimal/numeric`, `real`, `double precision`, `serial` |
| Character | `char(n)`, `varchar(n)`, `text` (unlimited)                                              |
| Boolean   | `boolean` (true/false/null)                                                              |
| Date/Time | `date`, `time`, `timestamp`, `timestamptz`, `interval`                                   |
| UUID      | `uuid` (128-bit)                                                                         |
| JSON      | `json` (text), `jsonb` (binary, indexable)                                               |
| Array     | `integer[]`, `text[]`, etc.                                                              |
| Network   | `inet`, `cidr`, `macaddr`                                                                |
| Geometric | `point`, `line`, `polygon`, `circle`                                                     |
| Range     | `int4range`, `tsrange`, `daterange`                                                      |

> **Always use `timestamptz`** (with time zone) over `timestamp`. It stores UTC internally and converts to the session's timezone on display.

---

### 3. **What is the difference between `CHAR`, `VARCHAR`, and `TEXT`?**

**Answer:**

| Type         | Behavior                                                             |
| ------------ | -------------------------------------------------------------------- |
| `CHAR(n)`    | Fixed-length, padded with spaces. Rarely useful.                     |
| `VARCHAR(n)` | Variable-length with limit. Use when you need a constraint.          |
| `TEXT`       | Variable-length, no limit. Internally same performance as `VARCHAR`. |

> **In PostgreSQL**, `TEXT` and `VARCHAR` have identical performance. Use `TEXT` unless you need a length constraint for business rules.

---

### 4. **What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?**

**Answer:**

| Command    | What it does                       | Logged?         | Rollback?            | Triggers?          |
| ---------- | ---------------------------------- | --------------- | -------------------- | ------------------ |
| `DELETE`   | Removes rows (with WHERE)          | Row-by-row      | ✅ Yes               | ✅ Fires triggers  |
| `TRUNCATE` | Removes all rows, resets sequences | Minimal logging | ✅ Yes (in Postgres) | ❌ No row triggers |
| `DROP`     | Deletes entire table + structure   | DDL             | ✅ Yes (in Postgres) | ❌ No              |

```sql
DELETE FROM logs WHERE created_at < '2024-01-01';  -- Selective, slow for large tables
TRUNCATE TABLE temp_import;                         -- Instant, all rows gone
DROP TABLE IF EXISTS old_backups;                    -- Table no longer exists
```

---

### 5. **What is a primary key vs unique constraint?**

**Answer:**

| Feature     | Primary Key                         | Unique Constraint                   |
| ----------- | ----------------------------------- | ----------------------------------- |
| NULL values | ❌ Not allowed                      | ✅ Allowed (one NULL per column)    |
| Per table   | One only                            | Multiple allowed                    |
| Index       | Automatically creates unique B-tree | Automatically creates unique B-tree |
| Purpose     | Row identity                        | Business rule enforcement           |

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,          -- Unique constraint
  username TEXT NOT NULL UNIQUE         -- Another unique constraint
);
```

---

### 6. **What are the types of JOINs?**

**Answer:**

```sql
-- INNER JOIN: only matching rows from both tables
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all users, even with no orders (NULL for order columns)
SELECT u.name, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- RIGHT JOIN: all orders, even with no matching user
SELECT u.name, o.total
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN: all rows from both, NULLs where no match
SELECT u.name, o.total
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- CROSS JOIN: cartesian product (every combination)
SELECT colors.name, sizes.label
FROM colors CROSS JOIN sizes;
```

---

## Indexing

### 7. **What index types does PostgreSQL support?**

**Answer:**

| Type                               | When to use                                                       |
| ---------------------------------- | ----------------------------------------------------------------- |
| **B-tree** (default)               | Equality and range queries (`=`, `<`, `>`, `BETWEEN`, `ORDER BY`) |
| **Hash**                           | Equality only (`=`). Rarely better than B-tree.                   |
| **GIN** (Generalized Inverted)     | Full-text search, JSONB, arrays, tsvector                         |
| **GiST** (Generalized Search Tree) | Geospatial (PostGIS), range types, full-text                      |
| **BRIN** (Block Range Index)       | Very large naturally-ordered tables (timestamps, serial IDs)      |
| **SP-GiST**                        | Non-balanced structures (phone numbers, IP ranges)                |

```sql
-- B-tree (default)
CREATE INDEX idx_users_email ON users (email);

-- GIN for JSONB queries
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata);

-- GIN for full-text search
CREATE INDEX idx_articles_search ON articles USING GIN (to_tsvector('english', title || ' ' || body));

-- BRIN for time-series data (very small index for huge tables)
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at);

-- Partial index (only index active users)
CREATE INDEX idx_active_users ON users (email) WHERE is_active = true;

-- Covering index (includes extra columns to enable index-only scans)
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (total, status);
```

---

### 8. **What is a composite index and does column order matter?**

**Answer:** A composite index indexes multiple columns. **Column order matters a lot** — the index is used left-to-right.

```sql
CREATE INDEX idx_orders_user_date ON orders (user_id, created_at);

-- ✅ Uses index (leftmost prefix)
SELECT * FROM orders WHERE user_id = 123;

-- ✅ Uses index (both columns, left to right)
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';

-- ❌ Cannot use this index efficiently (missing left column)
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

> **Rule of thumb:** Put the most selective (filtering) column first, range conditions last.

---

### 9. **What is EXPLAIN ANALYZE?**

**Answer:** `EXPLAIN ANALYZE` runs the query and shows the actual execution plan with timing.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE user_id = 123 AND status = 'shipped';

-- Output:
-- Index Scan using idx_orders_user on orders  (cost=0.43..8.45 rows=1 width=82) (actual time=0.023..0.025 rows=3 loops=1)
--   Index Cond: (user_id = 123)
--   Filter: (status = 'shipped')
--   Rows Removed by Filter: 2
-- Planning Time: 0.112 ms
-- Execution Time: 0.048 ms
```

**Key things to look for:**

- **Seq Scan** on large tables → missing index
- **Rows Removed by Filter** being large → index not selective enough
- **Nested Loop** with large outer table → consider Hash Join
- **Sort** → consider adding an index to avoid sort

---

## Transactions & Concurrency

### 10. **What are the ACID properties?**

**Answer:**

| Property        | Meaning                                          |
| --------------- | ------------------------------------------------ |
| **Atomicity**   | Transaction is all-or-nothing                    |
| **Consistency** | Data satisfies all constraints after transaction |
| **Isolation**   | Concurrent transactions don't interfere          |
| **Durability**  | Committed data survives crashes                  |

---

### 11. **What are PostgreSQL isolation levels?**

**Answer:**

| Level                    | Dirty read | Non-repeatable read | Phantom read     | Serialization anomaly |
| ------------------------ | ---------- | ------------------- | ---------------- | --------------------- |
| Read Uncommitted\*       | ❌         | ✅ Possible         | ✅ Possible      | ✅ Possible           |
| Read Committed (default) | ❌         | ✅ Possible         | ✅ Possible      | ✅ Possible           |
| Repeatable Read          | ❌         | ❌                  | ❌ (in Postgres) | ✅ Possible           |
| Serializable             | ❌         | ❌                  | ❌               | ❌                    |

\*Postgres treats Read Uncommitted as Read Committed.

```sql
-- Set isolation level per transaction
BEGIN ISOLATION LEVEL SERIALIZABLE;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

> **Serializable** is the safest but can cause serialization failures. Your application must **retry** on error code `40001`.

---

### 12. **What is MVCC (Multi-Version Concurrency Control)?**

**Answer:** MVCC allows readers and writers to not block each other. Each transaction sees a **snapshot** of the data as of its start time.

How it works:

- Each row has hidden `xmin` (transaction that created it) and `xmax` (transaction that deleted/updated it) columns
- Readers see rows where `xmin` is committed and `xmax` is not yet committed (or not set)
- Writers create new row versions instead of modifying in-place
- Old versions are cleaned up by **VACUUM**

> **Consequence:** Unlike MySQL's locking reads, `SELECT` in Postgres **never blocks** and is **never blocked** by writes.

---

### 13. **What is VACUUM and why is it important?**

**Answer:** VACUUM reclaims storage from dead tuple versions (left by UPDATE/DELETE due to MVCC).

| Command          | What it does                                        |
| ---------------- | --------------------------------------------------- |
| `VACUUM`         | Marks dead tuples as reusable (doesn't shrink file) |
| `VACUUM FULL`    | Rewrites table to reclaim disk space (locks table!) |
| `VACUUM ANALYZE` | Vacuum + update statistics for query planner        |
| `autovacuum`     | Background daemon that runs VACUUM automatically    |

```sql
-- Check dead tuples
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

> **Never disable autovacuum.** Without it, table bloat grows, query performance degrades, and you risk **transaction ID wraparound** (catastrophic — Postgres shuts down to prevent data corruption).

---

## JSON & Advanced Queries

### 14. **What is the difference between JSON and JSONB in PostgreSQL?**

**Answer:**

| Feature        | `json`              | `jsonb`                           |
| -------------- | ------------------- | --------------------------------- |
| Storage        | Text (preserved)    | Binary (parsed)                   |
| Duplicate keys | Preserved           | Last value wins                   |
| Key order      | Preserved           | Not preserved                     |
| Indexing       | ❌ No               | ✅ GIN index                      |
| Operators      | Basic extraction    | Full set (containment, existence) |
| Insert speed   | Faster (no parsing) | Slightly slower                   |
| Query speed    | Slower (re-parse)   | **Much faster**                   |

> **Use `jsonb`** in almost all cases. Use `json` only if you need exact text preservation.

```sql
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL,
  attributes JSONB DEFAULT '{}'
);

-- Insert
INSERT INTO products (name, attributes) VALUES
  ('Laptop', '{"brand": "Dell", "ram": 16, "tags": ["work", "portable"]}');

-- Query JSONB
SELECT name FROM products WHERE attributes->>'brand' = 'Dell';
SELECT name FROM products WHERE attributes @> '{"ram": 16}';     -- containment
SELECT name FROM products WHERE attributes ? 'brand';             -- key exists
SELECT name FROM products WHERE attributes->'tags' ? 'portable';  -- array contains

-- GIN index for fast JSONB queries
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
```

---

### 15. **What are CTEs (Common Table Expressions)?**

**Answer:** CTEs define temporary named result sets within a query using `WITH`. They improve readability and enable recursive queries.

```sql
-- Simple CTE
WITH monthly_sales AS (
  SELECT
    date_trunc('month', created_at) AS month,
    SUM(total) AS revenue,
    COUNT(*) AS order_count
  FROM orders
  WHERE created_at >= '2024-01-01'
  GROUP BY 1
)
SELECT month, revenue, order_count,
  revenue - LAG(revenue) OVER (ORDER BY month) AS growth
FROM monthly_sales;

-- Recursive CTE: org chart hierarchy
WITH RECURSIVE org_tree AS (
  -- Base case: top-level managers
  SELECT id, name, manager_id, 1 AS level
  FROM employees
  WHERE manager_id IS NULL

  UNION ALL

  -- Recursive case: employees under managers
  SELECT e.id, e.name, e.manager_id, t.level + 1
  FROM employees e
  JOIN org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree ORDER BY level, name;
```

---

### 16. **What are window functions?**

**Answer:** Window functions compute values across related rows without collapsing them (unlike GROUP BY).

```sql
SELECT
  name,
  department,
  salary,
  -- Rank within department
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
  -- Running total
  SUM(salary) OVER (ORDER BY hire_date) AS running_total,
  -- Moving average (last 3 employees)
  AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg,
  -- Percentage of department total
  ROUND(100.0 * salary / SUM(salary) OVER (PARTITION BY department), 2) AS pct_of_dept
FROM employees;
```

Common window functions: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()`, `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()`.

---

### 17. **What is an UPSERT in PostgreSQL?**

**Answer:** `INSERT ... ON CONFLICT` — inserts a row, or updates it if a unique constraint is violated.

```sql
-- Upsert: insert or update on email conflict
INSERT INTO users (email, name, login_count)
VALUES ('alice@example.com', 'Alice', 1)
ON CONFLICT (email) DO UPDATE SET
  name = EXCLUDED.name,
  login_count = users.login_count + 1,
  updated_at = NOW();

-- Upsert: insert or ignore
INSERT INTO page_views (page_url, user_id)
VALUES ('/home', 'u-123')
ON CONFLICT DO NOTHING;
```

> `EXCLUDED` refers to the row that was proposed for insertion (the conflicting row).

---

## Performance & Configuration

### 18. **What are the key PostgreSQL configuration parameters?**

**Answer:**

| Parameter              | Default   | Recommendation      | Purpose                               |
| ---------------------- | --------- | ------------------- | ------------------------------------- |
| `shared_buffers`       | 128 MB    | 25% of RAM          | In-memory page cache                  |
| `effective_cache_size` | 4 GB      | 50–75% of RAM       | Planner's estimate of OS cache        |
| `work_mem`             | 4 MB      | 64–256 MB           | Memory for sorts, joins per operation |
| `maintenance_work_mem` | 64 MB     | 512 MB – 2 GB       | Memory for VACUUM, CREATE INDEX       |
| `wal_buffers`          | -1 (auto) | 64 MB               | WAL write buffer                      |
| `max_connections`      | 100       | 100–200 + pgbouncer | Connection limit                      |
| `random_page_cost`     | 4.0       | 1.1 (SSD)           | Cost estimate for random I/O          |

> **Connection pooling:** Don't raise `max_connections` to 1000 — each connection uses ~10 MB RAM. Use **PgBouncer** or **pgpool-II** for connection pooling.

---

### 19. **What is the difference between a materialized view and a regular view?**

**Answer:**

| Feature     | View                         | Materialized View          |
| ----------- | ---------------------------- | -------------------------- |
| Storage     | No storage (query rewritten) | Results stored on disk     |
| Freshness   | Always current               | Snapshot (must refresh)    |
| Performance | Same as underlying query     | Fast reads (cached result) |
| Indexable   | ❌ No                        | ✅ Yes                     |

```sql
-- Regular view (always fresh, but runs query each time)
CREATE VIEW active_users AS
SELECT * FROM users WHERE last_login > NOW() - INTERVAL '30 days';

-- Materialized view (cached, fast reads)
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
  date_trunc('month', created_at) AS month,
  SUM(total) AS revenue
FROM orders
GROUP BY 1;

-- Create index on materialized view
CREATE INDEX idx_monthly_rev ON monthly_revenue (month);

-- Must refresh manually
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
-- CONCURRENTLY: doesn't lock reads during refresh (requires unique index)
```

---

## Replication & High Availability

### 20. **What are the replication types in PostgreSQL?**

**Answer:**

| Type                     | How it works                                     | Use case                                  |
| ------------------------ | ------------------------------------------------ | ----------------------------------------- |
| **Streaming (physical)** | Ships WAL bytes to replica (exact copy)          | HA failover, read replicas                |
| **Logical**              | Ships decoded row changes (INSERT/UPDATE/DELETE) | Selective replication, cross-version, ETL |
| **Synchronous**          | Primary waits for replica ACK before commit      | Zero data loss (higher latency)           |
| **Asynchronous**         | Primary doesn't wait (default)                   | Lower latency, eventual consistency       |

```sql
-- Check replication status
SELECT client_addr, state, sent_lsn, write_lsn, replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

---

### 21. **What is a connection pooler and why do you need one?**

**Answer:** Each Postgres connection forks a process (~10 MB). At 500+ connections, the overhead is significant. A pooler multiplexes many app connections onto fewer database connections.

**PgBouncer modes:**

| Mode        | Behavior                             | Transactions supported   |
| ----------- | ------------------------------------ | ------------------------ |
| Session     | Conn held for entire session         | All features             |
| Transaction | Conn returned after each transaction | Most features            |
| Statement   | Conn returned after each statement   | No transactions, limited |

> **Production standard:** PgBouncer in **transaction mode** with `max_connections=100` on Postgres and PgBouncer pool of 200+ for application.

---

### 22. **What are the common PostgreSQL extensions?**

**Answer:**

| Extension                | Purpose                                        |
| ------------------------ | ---------------------------------------------- |
| `pg_stat_statements`     | Query performance statistics (most important!) |
| `PostGIS`                | Geospatial data and queries                    |
| `pgcrypto`               | Cryptographic functions                        |
| `uuid-ossp` / `pgcrypto` | UUID generation                                |
| `pg_trgm`                | Trigram similarity (fuzzy text search)         |
| `hstore`                 | Key-value pairs in a single column             |
| `tablefunc`              | Pivot tables (crosstab)                        |
| `pgvector`               | Vector similarity search (AI embeddings)       |
| `timescaledb`            | Time-series optimization                       |

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pgvector;

-- Find slowest queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

---

### 23. **How do you handle full-text search in PostgreSQL?**

**Answer:** Postgres has built-in full-text search using `tsvector` (document) and `tsquery` (search terms).

```sql
-- Add a search vector column
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B')
  ) STORED;

CREATE INDEX idx_articles_fts ON articles USING GIN (search_vector);

-- Search
SELECT title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgres & performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

---

### 24. **What is a partition in PostgreSQL and when should you use it?**

**Answer:** Table partitioning splits a large table into smaller physical tables while appearing as one logical table.

| Strategy  | Split by        | Use case                    |
| --------- | --------------- | --------------------------- |
| **Range** | Value ranges    | Time-series (by month/year) |
| **List**  | Specific values | By region, status, category |
| **Hash**  | Hash of column  | Even distribution           |

```sql
-- Range partitioning by date
CREATE TABLE events (
  id BIGSERIAL,
  event_type TEXT,
  payload JSONB,
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_q1 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE events_2024_q2 PARTITION OF events
  FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Queries automatically prune partitions
EXPLAIN SELECT * FROM events WHERE created_at = '2024-02-15';
-- Only scans events_2024_q1
```

> **Use partitioning when:** Table exceeds tens of millions of rows, queries consistently filter on the partition key, and you need efficient data lifecycle management (DROP old partitions instead of DELETE).

---

### 25. **What is LISTEN/NOTIFY in PostgreSQL?**

**Answer:** A built-in pub/sub mechanism for real-time notifications between database sessions.

```sql
-- Listener (application subscribes)
LISTEN order_events;

-- Notifier (trigger or application publishes)
NOTIFY order_events, '{"orderId": "ORD-123", "status": "shipped"}';

-- Trigger-based notification
CREATE OR REPLACE FUNCTION notify_order_change() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify('order_events', json_build_object(
    'operation', TG_OP,
    'id', NEW.id,
    'status', NEW.status
  )::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_change_trigger
AFTER INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION notify_order_change();
```

```ts
// Node.js listener
import { Client } from "pg";

const client = new Client();
await client.connect();

client.on("notification", (msg) => {
  const payload = JSON.parse(msg.payload!);
  console.log(`Order ${payload.id}: ${payload.status}`);
});

await client.query("LISTEN order_events");
```

> **Limitations:** Payload max 8000 bytes. Not durable — if no one is listening, notifications are lost. For durable messaging, use a proper queue (SQS, Kafka).

---

## Scalability, Sharding, Error Handling & Performance

### 26. **How do you horizontally scale PostgreSQL?**

**Answer:** PostgreSQL is single-primary by default, but several strategies exist for horizontal scaling.

| Strategy                              |  Read Scaling  |   Write Scaling    | Complexity |
| ------------------------------------- | :------------: | :----------------: | :--------: |
| Read replicas (streaming replication) |       ✅       |         ❌         |    Low     |
| Connection pooling (PgBouncer)        | ✅ (effective) |   ✅ (effective)   |    Low     |
| Table partitioning                    |       ✅       | ✅ (localized I/O) |   Medium   |
| Citus extension (distributed)         |       ✅       |         ✅         |   Medium   |
| Application-level sharding            |       ✅       |         ✅         |    High    |
| Foreign Data Wrappers (federated)     |       ✅       |      Limited       |   Medium   |

```sql
-- Read replicas: application routes reads to replicas
-- In your connection config:
-- Primary: postgresql://primary:5432/mydb  (writes)
-- Replica: postgresql://replica1:5432/mydb (reads)

-- Table partitioning for large tables (keeps data on single server but improves perf)
CREATE TABLE events (
  id         BIGSERIAL,
  created_at TIMESTAMPTZ NOT NULL,
  event_type TEXT,
  payload    JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_q1 PARTITION OF events
  FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE events_2026_q2 PARTITION OF events
  FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

-- PostgreSQL automatically routes queries to the right partition
SELECT * FROM events WHERE created_at BETWEEN '2026-02-01' AND '2026-03-01';
-- Only scans events_2026_q1 — skips all other partitions (partition pruning)
```

---

### 27. **How does Citus enable sharding and distributed queries in PostgreSQL?**

**Answer:** Citus is a PostgreSQL extension that distributes tables across multiple nodes while keeping standard SQL.

```sql
-- Step 1: Create a distributed table (sharded)
SELECT create_distributed_table('orders', 'customer_id');
-- Rows are now spread across worker nodes based on customer_id hash

-- Step 2: Colocated tables — join-friendly sharding
SELECT create_distributed_table('order_items', 'customer_id');
-- orders and order_items for the same customer are on the same node
-- JOINs between them are LOCAL (fast, no network)

-- Step 3: Reference tables — small lookups replicated to all nodes
SELECT create_reference_table('product_categories');
-- JOINs with reference tables are always local

-- Distributed queries — Citus parallelizes automatically
SELECT customer_id, SUM(total) as lifetime_value
FROM orders
GROUP BY customer_id
ORDER BY lifetime_value DESC
LIMIT 10;
-- Runs in parallel across all shards, results merged at coordinator
```

| Citus Table Type | Distribution              | Best For                              |
| ---------------- | ------------------------- | ------------------------------------- |
| Distributed      | Hash-sharded across nodes | Large tables (orders, events)         |
| Reference        | Replicated to all nodes   | Small lookups (categories, countries) |
| Local            | Only on coordinator       | Metadata, config                      |

---

### 28. **How do you handle errors and deadlocks in PostgreSQL transactions?**

**Answer:**

```sql
-- Detect deadlocks — PostgreSQL auto-kills one transaction after deadlock_timeout (default 1s)
-- Application should catch and retry:
SHOW deadlock_timeout; -- default: 1s
```

```ts
// Node.js: retry on deadlock or serialization failure
async function executeWithRetry(pool, fn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    const client = await pool.connect();
    try {
      await client.query("BEGIN");
      const result = await fn(client);
      await client.query("COMMIT");
      return result;
    } catch (error) {
      await client.query("ROLLBACK");

      const retryableCodes = [
        "40001", // serialization_failure
        "40P01", // deadlock_detected
        "23505", // unique_violation (if using INSERT ... ON CONFLICT)
      ];

      if (retryableCodes.includes(error.code) && attempt < maxRetries) {
        const delay = Math.min(100 * 2 ** attempt, 2000);
        console.warn(
          `Retryable error ${error.code}, attempt ${attempt}/${maxRetries}`,
        );
        await new Promise((r) => setTimeout(r, delay));
        continue;
      }
      throw error; // non-retryable or max retries exceeded
    } finally {
      client.release();
    }
  }
}

// Usage
const order = await executeWithRetry(pool, async (client) => {
  await client.query(
    "UPDATE inventory SET stock = stock - $1 WHERE product_id = $2",
    [qty, productId],
  );
  const { rows } = await client.query(
    "INSERT INTO orders (product_id, quantity) VALUES ($1, $2) RETURNING *",
    [productId, qty],
  );
  return rows[0];
});
```

| Error Code | Name                  | Meaning                                | Action                             |
| ---------- | --------------------- | -------------------------------------- | ---------------------------------- |
| 40001      | serialization_failure | Concurrent modification conflict       | Retry transaction                  |
| 40P01      | deadlock_detected     | Two transactions waiting on each other | Retry (PG killed one)              |
| 23505      | unique_violation      | Duplicate key                          | Use ON CONFLICT or handle          |
| 57014      | query_canceled        | statement_timeout triggered            | Optimize query or increase timeout |
| 53300      | too_many_connections  | Connection limit hit                   | Use PgBouncer                      |

---

### 29. **What are the performance tuning factors for PostgreSQL?**

**Answer:**

| Parameter                  | Default | Recommended             | Impact                                        |
| -------------------------- | ------- | ----------------------- | --------------------------------------------- |
| `shared_buffers`           | 128 MB  | 25% of RAM              | Pages cached in PostgreSQL                    |
| `effective_cache_size`     | 4 GB    | 75% of RAM              | Planner estimates for OS cache                |
| `work_mem`                 | 4 MB    | 64–256 MB               | Per-sort/hash memory (careful: per-operation) |
| `maintenance_work_mem`     | 64 MB   | 1–2 GB                  | VACUUM, CREATE INDEX speed                    |
| `max_connections`          | 100     | 100–200 (use PgBouncer) | More = more memory per connection             |
| `random_page_cost`         | 4.0     | 1.1 (SSD)               | Helps planner choose index scans              |
| `effective_io_concurrency` | 1       | 200 (SSD)               | Parallel I/O for bitmap scans                 |
| `wal_level`                | replica | logical (if CDC needed) | WAL verbosity                                 |

```sql
-- Find expensive queries (requires pg_stat_statements extension)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
  substring(query, 1, 80) AS short_query,
  calls,
  round(total_exec_time::numeric, 2) AS total_ms,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Check index usage — find missing indexes
SELECT
  schemaname, tablename,
  seq_scan, seq_tup_read,
  idx_scan, idx_tup_fetch,
  CASE WHEN seq_scan > 0 THEN round(seq_tup_read::numeric / seq_scan, 0) END AS avg_rows_per_seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
ORDER BY seq_tup_read DESC;
-- Tables with high seq_scan + high avg_rows_per_seq_scan need indexes

-- Check table bloat
SELECT
  schemaname, tablename,
  pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
  n_dead_tup,
  last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

### 30. **How do you achieve high availability and disaster recovery with PostgreSQL?**

**Answer:**

```
Architecture: Active-Standby with auto-failover

┌─────────────┐       Streaming         ┌──────────────┐
│   Primary   │ ──── Replication ─────→  │   Standby    │
│  (writes)   │    (sync or async)       │   (reads)    │
└──────┬──────┘                          └──────┬───────┘
       │                                        │
       └──────── Patroni / pg_auto_failover ────┘
                 (auto-promotes standby if primary fails)
```

| HA Solution              | How It Works                            | Failover Speed |
| ------------------------ | --------------------------------------- | -------------- |
| **Patroni** (Kubernetes) | DCS (etcd/consul) based leader election | 10–30s         |
| **pg_auto_failover**     | Citus extension, monitor node           | 10–20s         |
| **AWS RDS Multi-AZ**     | Synchronous standby, auto-failover      | 60–120s        |
| **Aurora PostgreSQL**    | Shared storage, 6 copies across 3 AZs   | < 30s          |
| **Stolon**               | Kubernetes-native, uses etcd            | 10–30s         |

```sh
# Patroni: check cluster status
patronictl list

# Output:
# + Cluster: mydb (12345678) -------+
# | Member   | Host     | Role    | State   | TL | Lag |
# +----------+----------+---------+---------+----+-----+
# | node-1   | 10.0.1.1 | Leader  | running |  5 |     |
# | node-2   | 10.0.1.2 | Replica | running |  5 |  0  |
# | node-3   | 10.0.1.3 | Replica | running |  5 |  0  |
# +----------+----------+---------+---------+----+-----+

# Manual failover (if needed)
patronictl failover --candidate node-2
```

> **Sync vs Async Replication:**
>
> - **Synchronous:** Zero data loss (RPO=0), but writes wait for standby ACK (higher latency)
> - **Asynchronous:** Lower latency, but risk of losing last few transactions on failover (RPO > 0)
