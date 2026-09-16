# TimescaleDB Interview Questions & Answers

A curated list of **TimescaleDB interview questions** — easy to hard difficulty with practical examples. Covers fundamentals, hypertables, chunking, indexing, compression, continuous aggregates, retention policies, and real-world time-series patterns.

---

## Fundamentals

### 1. **What is TimescaleDB?**

**Answer:** TimescaleDB is an open-source **time-series database** built as an **extension on top of PostgreSQL**. It gives you full SQL, joins, and the entire PostgreSQL ecosystem, while automatically optimizing storage and queries for time-stamped data.

Key highlights:

- **It's still PostgreSQL** — same drivers, same SQL, same tooling (psql, pg_dump, ORMs)
- **Hypertables** — the core abstraction that auto-partitions data by time (and optionally space)
- **Built for time-series + relational data together** — no need for a separate OLTP database
- **Native compression** — columnar-style compression on top of a row store
- **Continuous aggregates** — incrementally-maintained materialized views for rollups
- **Automated data lifecycle** — retention policies, tiering to object storage, downsampling

```sql
-- Enable the extension (once per database)
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

> **Interview framing:** TimescaleDB's pitch is "you already know SQL — now get Postgres to handle billions of time-stamped rows without hand-rolling partitioning." That's the answer interviewers are usually listening for.

---

### 2. **How is TimescaleDB different from vanilla PostgreSQL partitioning?**

**Answer:**

| Feature                | Native Postgres Partitioning     | TimescaleDB Hypertables                              |
| ----------------------- | --------------------------------- | ------------------------------------------------------ |
| Partition creation       | Manual (`CREATE TABLE ... PARTITION OF`) | Automatic, created on insert as data arrives      |
| Partition sizing         | You decide ranges upfront         | Auto-sized ("chunks") based on target size/time interval |
| Compression               | Manual / none                     | Built-in native columnar compression                  |
| Continuous rollups        | Manual materialized views + cron  | Continuous aggregates (auto-refreshing)                |
| Retention                 | Manual `DROP TABLE` scripts       | `add_retention_policy()` — declarative, scheduled      |
| Query planner awareness   | Partition pruning only            | Chunk exclusion + time-series-aware query planning     |

```sql
-- Vanilla Postgres: you manage this yourself
CREATE TABLE sensor_data (id BIGSERIAL, ts TIMESTAMPTZ, value DOUBLE PRECISION)
  PARTITION BY RANGE (ts);
CREATE TABLE sensor_data_2026_01 PARTITION OF sensor_data
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ...you'd need a cron job to create next month's partition before it's needed

-- TimescaleDB: partitions ("chunks") are created automatically
SELECT create_hypertable('sensor_data', 'ts');
-- Just insert. Timescale creates chunks behind the scenes as data arrives.
```

---

### 3. **What is a hypertable?**

**Answer:** A **hypertable** is TimescaleDB's abstraction for a table that is automatically partitioned by time (and optionally by a "space" dimension like `device_id`). To the application, it looks and behaves like one normal table — you `INSERT`, `SELECT`, `JOIN` it like any Postgres table. Under the hood, TimescaleDB splits it into many smaller physical tables called **chunks**.

```sql
CREATE TABLE conditions (
  time        TIMESTAMPTZ NOT NULL,
  device_id   TEXT NOT NULL,
  temperature DOUBLE PRECISION,
  humidity    DOUBLE PRECISION
);

-- Convert it into a hypertable, partitioned on "time"
SELECT create_hypertable('conditions', by_range('time'));

-- Insert normally — Timescale routes rows to the right chunk automatically
INSERT INTO conditions VALUES (now(), 'sensor-1', 22.5, 55.1);

-- Query normally — looks like a single table
SELECT device_id, avg(temperature)
FROM conditions
WHERE time > now() - INTERVAL '1 day'
GROUP BY device_id;
```

> **Note (Timescale 2.13+):** `create_hypertable('conditions', by_range('time'))` is the modern dimension-builder syntax. Older tutorials use `create_hypertable('conditions', 'time')` — both work, but `by_range`/`by_hash` is now preferred because it's explicit about partitioning type.

---

### 4. **What is a "chunk" and why does chunk sizing matter?**

**Answer:** A **chunk** is the physical child table behind a hypertable, covering one time range (e.g., one day or one week of data). Timescale creates a new chunk automatically once data for a new interval arrives.

Why sizing matters:

- **Too small** (e.g., 1-minute chunks) → thousands of tiny tables, high planning overhead, too many files on disk
- **Too large** (e.g., 1-year chunks) → each chunk's B-tree indexes and any in-memory working set become huge, hurting insert and query performance, and a single chunk won't compress/evict cleanly

**Rule of thumb:** Size chunks so the **most recent chunk's indexes fit comfortably in memory** (a common starting point is 25% of RAM for your recent working set, but the real target is "recent chunk fits in cache").

```sql
-- Set/change the chunk time interval (default is 7 days)
SELECT set_chunk_time_interval('conditions', INTERVAL '1 day');

-- Inspect chunk sizes
SELECT chunk_name, range_start, range_end, total_bytes
FROM chunks_detailed_size('conditions');
```

---

### 5. **What data types are commonly used for the time column?**

**Answer:** Almost always `TIMESTAMPTZ` (timestamp with time zone). Timescale also supports partitioning on `TIMESTAMP`, `DATE`, `BIGINT`, or `INTEGER` (e.g., epoch millis or an auto-incrementing sequence id), which is useful for IoT devices that emit raw epoch integers.

```sql
-- Standard: TIMESTAMPTZ, stored as UTC internally (same as vanilla Postgres advice)
CREATE TABLE metrics (
  time  TIMESTAMPTZ NOT NULL DEFAULT now(),
  host  TEXT,
  cpu   DOUBLE PRECISION
);
SELECT create_hypertable('metrics', by_range('time'));

-- Epoch bigint partitioning (e.g., device sends unix millis)
CREATE TABLE raw_events (
  ts_ms  BIGINT NOT NULL,
  payload JSONB
);
SELECT create_hypertable('raw_events', by_range('ts_ms'), chunk_time_interval => 86400000); -- 1 day in ms
```

> **Always use `TIMESTAMPTZ`**, same guidance as plain Postgres — avoid timezone bugs entirely by storing UTC and converting on display.

**Setting the chunk interval at creation time:** you don't have to accept the 7-day default and change it later with `set_chunk_time_interval()` (see Q4) — pass `chunk_time_interval` directly to `create_hypertable()` up front, which is the more common pattern in real schemas:

```sql
-- TIMESTAMPTZ/TIMESTAMP/DATE columns: pass an INTERVAL
SELECT create_hypertable(
  'metrics',
  by_range('time'),
  chunk_time_interval => INTERVAL '1 day'
);

-- Integer/bigint (epoch) columns: pass a plain number in the column's own units
SELECT create_hypertable(
  'raw_events',
  by_range('ts_ms'),
  chunk_time_interval => 86400000  -- 1 day, in milliseconds — no INTERVAL literal here
);
```

> **Gotcha:** the type of `chunk_time_interval` must match the partitioning column's type family. `TIMESTAMPTZ`/`TIMESTAMP`/`DATE` columns take an `INTERVAL` value; `BIGINT`/`INTEGER` columns take a raw integer expressed in *whatever unit that column already uses* (ms, µs, or a plain sequence id) — Timescale has no idea a `BIGINT` column means "milliseconds" unless you tell it via the interval size you pass.

Other data-type details interviewers probe for:

- **`TIMESTAMP` (no time zone) is discouraged**: it's accepted, but you inherit the classic Postgres footgun of ambiguous local time on DST transitions — always prefer `TIMESTAMPTZ`.
- **Integer/epoch partitioning has a trade-off**: it's faster to insert (no timezone conversion) and common for IoT/embedded pipelines, but you lose calendar-aware functions like `time_bucket('1 month', ...)`, `now()`-relative queries, and DST-aware bucketing — you're on your own for unit conversions in every query.
- **`time_bucket()` vs `date_trunc()`**: for grouping/rollups, Timescale's `time_bucket(INTERVAL, time_col)` is preferred over Postgres's `date_trunc()` because it supports arbitrary bucket widths (e.g., `'15 minutes'`, `'6 hours'`) instead of only fixed calendar units, and it works transparently on both timestamp and integer time columns.

---

## Core Operations

### 6. **How do you insert data efficiently into a hypertable?**

**Answer:** Hypertables support normal `INSERT`, but at time-series scale you want **batched inserts** — single-row inserts per network round trip will bottleneck you far before Timescale does.

```sql
-- ❌ Slow: one round trip per row
INSERT INTO conditions VALUES (now(), 'sensor-1', 22.5, 55.1);
INSERT INTO conditions VALUES (now(), 'sensor-2', 21.9, 54.8);

-- ✅ Fast: multi-row insert, single round trip
INSERT INTO conditions (time, device_id, temperature, humidity) VALUES
  (now(), 'sensor-1', 22.5, 55.1),
  (now(), 'sensor-2', 21.9, 54.8),
  (now(), 'sensor-3', 23.1, 56.0);
```

```sql
-- ✅ Fastest for bulk backfill: COPY
COPY conditions (time, device_id, temperature, humidity) FROM STDIN WITH (FORMAT csv);
```

> **Tip:** For high-throughput ingestion pipelines, batch 100–5,000 rows per `INSERT`/`COPY` and use multiple parallel connections — insert throughput scales roughly linearly with client parallelism up to your CPU/disk limits.

---

### 7. **How do you query the most recent value per device ("last point" queries)?**

**Answer:** TimescaleDB provides `last()` and `first()` aggregate functions specifically for this — much faster than a manual `ORDER BY ... LIMIT 1` per group when done naively.

```sql
-- Last recorded temperature per device
SELECT
  device_id,
  last(temperature, time) AS last_temp,
  last(time, time)        AS last_seen
FROM conditions
GROUP BY device_id;
```

```sql
-- Common alternative: DISTINCT ON, works but needs a supporting index
SELECT DISTINCT ON (device_id) device_id, time, temperature
FROM conditions
ORDER BY device_id, time DESC;
```

> `last(value, time)` returns the `value` corresponding to the maximum `time` in the group — it's a single aggregate pass, no self-join needed.

---

### 8. **How do you downsample/bucket time-series data?**

**Answer:**

**What downsampling actually is (small example first).** Downsampling means taking many high-frequency data points and collapsing them into fewer, lower-frequency points — usually by grouping them into fixed time windows ("buckets") and computing one summary value (avg, max, min, sum, count...) per window. You trade granularity for volume: you can't recover the original per-second reading, but you keep the *shape* of the data at a fraction of the size.

Say a sensor reports temperature every minute:

```
09:00 → 20°C
09:01 → 21°C
09:02 → 22°C
09:03 → 19°C
09:04 → 20°C
```

If you downsample into 5-minute buckets using the average, all five readings collapse into one row:

```
09:00-09:05 bucket → avg = 20.4°C
```

A month of minute-level data (43,200 rows) becomes a month of 5-minute buckets (8,640 rows) — ~5x smaller, still representative for a chart or trend. That's the whole idea; TimescaleDB just gives you a purpose-built function to do the "group into fixed time windows" step correctly and efficiently.

**Going deep — `time_bucket()`.** This is TimescaleDB's core function for that grouping step (like `date_trunc`, but far more flexible: arbitrary intervals, not just calendar units).

```sql
-- Average temperature per 5-minute bucket, per device
SELECT
  time_bucket('5 minutes', time) AS bucket,
  device_id,
  avg(temperature) AS avg_temp,
  max(temperature) AS max_temp,
  min(temperature) AS min_temp
FROM conditions
WHERE time > now() - INTERVAL '1 day'
GROUP BY bucket, device_id
ORDER BY bucket DESC;
```

```sql
-- Arbitrary intervals — not possible with date_trunc
SELECT time_bucket('15 minutes', time), avg(temperature)
FROM conditions
GROUP BY 1
ORDER BY 1;

-- Offset buckets (e.g., business day starting at 6am instead of midnight)
SELECT time_bucket('1 day', time, INTERVAL '6 hours'), avg(temperature)
FROM conditions
GROUP BY 1;
```

How it works under the hood:

- **Bucket alignment.** By default, buckets are aligned to an origin (epoch/`2000-01-03` for calendar-based intervals), so `time_bucket('5 minutes', time)` always lands on `:00, :05, :10...` boundaries — not on whatever timestamp happened to be first in your data. This is what makes results deterministic and joinable across queries/devices.
- **Arbitrary vs. calendar intervals.** For intervals ≤ 1 day (seconds, minutes, hours), `time_bucket` treats the interval as a fixed-duration window and can bucket by *any* value — `'90 seconds'`, `'7 minutes'`, `'13 hours'`. For month/year intervals, it falls back to calendar semantics (months have variable length) similar to `date_trunc`.
- **Query planner awareness.** Because chunks are time-ranged, a `WHERE time > ...` predicate combined with `time_bucket` in the `SELECT`/`GROUP BY` lets Timescale prune irrelevant chunks *before* bucketing — downsampling a day out of a table with years of data doesn't scan the years.
- **Continuous aggregates are built on this.** A continuous aggregate (`CREATE MATERIALIZED VIEW ... WITH (timescaledb.continuous)`) is essentially a `time_bucket()` query that Timescale incrementally maintains for you, so you don't recompute the rollup from raw rows on every read. If you find yourself running the same `time_bucket` query repeatedly on a dashboard, that's the sign to promote it to a continuous aggregate instead.
- **Choosing the bucket size** is a trade-off: smaller buckets preserve more detail but downsample less (less storage/query win); larger buckets compress more aggressively but can hide spikes (e.g., a 1-hour avg can smooth over a 2-minute outage). Pick the bucket size based on what the consumer (dashboard, alert, model) actually needs to see.

> **Interview signal:** `time_bucket()` vs `date_trunc()` is a favorite question. `date_trunc` only supports calendar-aligned units (hour, day, month). `time_bucket` supports **arbitrary intervals** (`'7 minutes'`, `'90 seconds'`) and per-bucket offsets. A strong follow-up: explain that continuous aggregates are just pre-computed, incrementally-refreshed `time_bucket` rollups — the mechanism, not a separate feature.

---

### 9. **What are gap-filling functions and when do you need them?**

**Answer:** Time-series queries often need every bucket represented, even when there's no data (e.g., a dashboard chart with holes for down sensors). `time_bucket_gapfill()` plus `locf()` (last-observation-carried-forward) or `interpolate()` solve this.

```sql
SELECT
  time_bucket_gapfill('1 hour', time) AS bucket,
  device_id,
  avg(temperature) AS avg_temp,
  locf(avg(temperature)) AS carried_forward,       -- fill gaps with last known value
  interpolate(avg(temperature)) AS interpolated     -- fill gaps by linear interpolation
FROM conditions
WHERE time BETWEEN '2026-01-01' AND '2026-01-02'
  AND device_id = 'sensor-1'
GROUP BY bucket, device_id
ORDER BY bucket;
```

> `time_bucket_gapfill` **requires** a `WHERE` clause with explicit start/end bounds — Timescale needs to know the range to know which buckets to synthesize.

---

### 10. **How do you handle multi-dimensional partitioning (time + space)?**

**Answer:** In addition to partitioning by time, you can add a second ("space") dimension — commonly a device/tenant/sensor ID — to spread writes across chunks and parallelize queries that filter or join on that key.

```sql
SELECT create_hypertable(
  'conditions',
  by_range('time'),
  by_hash('device_id', 4)   -- 4 hash partitions on device_id
);
```

**When to add a space dimension:**

- Very high ingest rate where a single time-partition becomes a write hotspot
- Multi-tenant systems where you frequently filter/aggregate by tenant and want chunk exclusion on that key too

> **Caution:** Adding a space dimension multiplies chunk count (time chunks × space partitions). Only do it when you have a proven need — most workloads are fine with time-only partitioning.

---

## Indexing

### 11. **What indexes does TimescaleDB create automatically on a hypertable?**

**Answer:** When you run `create_hypertable()`, Timescale automatically creates a B-tree index on the partitioning ("time") column, unless one already exists. No other indexes are automatic — everything else (on `device_id`, `tenant_id`, composite columns, etc.) you must create yourself, exactly like plain Postgres, and it gets propagated to every chunk.

```sql
CREATE TABLE conditions (
  time        TIMESTAMPTZ NOT NULL,
  device_id   TEXT NOT NULL,
  temperature DOUBLE PRECISION
);

SELECT create_hypertable('conditions', by_range('time'));
-- ✅ Timescale auto-creates: CREATE INDEX ON conditions (time DESC);

-- ❌ No index on device_id yet — you must add it yourself
CREATE INDEX idx_conditions_device ON conditions (device_id, time DESC); -- speeds up per-device time-range scans
```

> **Interview signal:** a common trick question is "does Timescale index everything for me?" — no, only the time column. Query-pattern-specific indexes are still your job.

---

### 12. **Why is `(device_id, time DESC)` a better index than `(device_id, time ASC)` for time-series dashboards?**

**Answer:** Most time-series queries ask for the **most recent** data first (`ORDER BY time DESC LIMIT N`, or `WHERE time > now() - INTERVAL '1 hour'`). A `DESC` index on time lets Postgres scan directly in that order without a separate sort step.

```sql
-- Matches "latest N readings for a device" without an extra sort
CREATE INDEX idx_device_time_desc ON conditions (device_id, time DESC); -- avoids a Sort node in the query plan

SELECT * FROM conditions
WHERE device_id = 'sensor-1'
ORDER BY time DESC
LIMIT 10; -- index scan returns rows already in the right order
```

> `ASC` and `DESC` B-tree indexes can both be scanned backward by Postgres, so this matters less on vanilla Postgres than people think — but explicit `DESC` still documents intent and avoids edge cases with multi-column sort direction mismatches.

---

### 13. **How do you index for "filter by device, range by time" queries — the most common time-series access pattern?**

**Answer:** Put the equality-filtered column(s) first in a composite index, then the range-filtered/sorted time column last. This lets Postgres do an efficient index range scan: jump straight to the matching `device_id`, then scan a contiguous time range within it.

```sql
-- Equality column first, time range column last — the standard time-series index shape
CREATE INDEX idx_conditions_device_time ON conditions (device_id, time DESC); -- supports WHERE device_id = ? AND time > ?

EXPLAIN ANALYZE
SELECT * FROM conditions
WHERE device_id = 'sensor-1' AND time > now() - INTERVAL '1 day'
ORDER BY time DESC;
-- Index Scan using idx_conditions_device_time -- no separate filter/sort step needed
```

> **Rule of thumb:** equality columns before range columns in a composite index — reversing the order (`time, device_id`) forces a much less selective scan since time is queried as a range, not an exact match.

---

### 14. **How do indexes interact with chunk exclusion — do you need an index on every chunk?**

**Answer:** Indexes are created per chunk automatically when you index the hypertable — Timescale propagates the `CREATE INDEX` statement to every existing chunk and to all future chunks. Chunk exclusion (via the time predicate) happens *before* indexes are even considered: Timescale first prunes whole chunks outside the queried time range, then uses each remaining chunk's own index to narrow rows within it.

```sql
-- One statement, applied to every chunk (existing + future)
CREATE INDEX idx_conditions_device ON conditions (device_id); -- Timescale fans this out per-chunk automatically

-- Two-stage pruning in one query:
-- 1) chunk exclusion via WHERE time > ... (whole chunks skipped)
-- 2) per-chunk index scan via WHERE device_id = ... (rows skipped within remaining chunks)
SELECT * FROM conditions
WHERE time > now() - INTERVAL '2 days' AND device_id = 'sensor-1';
```

> This two-stage pruning (chunk exclusion, then per-chunk index) is why Timescale queries stay fast even as total row count grows into the billions — the index only ever has to work over the handful of chunks that survived exclusion.

---

### 15. **How do you index JSONB tag/metadata columns for filtering?**

**Answer:** Use a **GIN index**, same as plain Postgres, when you need to filter on arbitrary keys inside a `JSONB` column (e.g., tags, labels, dynamic metadata).

```sql
CREATE TABLE metrics (
  time  TIMESTAMPTZ NOT NULL,
  tags  JSONB
);
SELECT create_hypertable('metrics', by_range('time'));

-- GIN index for containment/key-existence queries on JSONB
CREATE INDEX idx_metrics_tags ON metrics USING GIN (tags); -- speeds up @>, ?, ?| operators

SELECT * FROM metrics
WHERE tags @> '{"region": "us-east"}' AND time > now() - INTERVAL '1 day'; -- containment lookup uses the GIN index
```

> **Trade-off:** GIN indexes are larger and slower to write than B-tree, and on a high-ingest hypertable they add real overhead per insert — only add one if you actually filter on JSONB contents, not just store it.

---

### 16. **Should you index columns you only ever filter through compressed `segmentby`?**

**Answer:** No — for a compressed chunk, `compress_segmentby` already lets Timescale skip whole segments without decompressing them, which does the same job a B-tree index would do on the equivalent uncompressed data. Adding a redundant B-tree index on that column on a compressed hypertable wastes space and doesn't meaningfully speed up equality lookups already served by segment exclusion.

```sql
ALTER TABLE conditions SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id', -- already gives fast equality filtering on device_id
  timescaledb.compress_orderby   = 'time DESC'
);

-- Redundant on compressed chunks — segmentby already covers this access pattern
-- CREATE INDEX idx_conditions_device ON conditions (device_id);

-- Still worth indexing separately if you filter on a column NOT in segmentby/orderby
CREATE INDEX idx_conditions_location ON conditions (location); -- not covered by segment exclusion
```

> **Interview framing:** this is a favorite "do you actually understand compression" question — segmentby is functionally an index substitute for equality filters on compressed data, so don't double up unless you query a different column.

---

### 17. **How do you find unused or redundant indexes on a hypertable?**

**Answer:** Same tooling as plain Postgres — `pg_stat_user_indexes` tracks scan counts per index, but remember it's tracked **per chunk**, so you typically want to aggregate across all chunks belonging to a hypertable.

```sql
-- Find indexes with zero (or very low) scans across all chunks of a hypertable
SELECT
  indexrelname,
  sum(idx_scan) AS total_scans
FROM pg_stat_user_indexes
WHERE relname LIKE '_hyper_%_chunk' -- underlying chunk tables
GROUP BY indexrelname
ORDER BY total_scans ASC; -- near-zero scans = candidate for removal

-- Timescale helper view — index sizes across all chunks of a hypertable, summed
SELECT * FROM hypertable_index_size('idx_conditions_device_time');
```

> Every unused index still costs write throughput (each insert updates every index) and storage per chunk — pruning unused indexes matters even more on hypertables because the cost is multiplied across potentially hundreds of chunks.

---

## Compression

### 18. **How does TimescaleDB compression work?**

**Answer:** TimescaleDB compresses chunks using a **hybrid row-columnar** format. Older/cold chunks are rewritten so that each column is stored contiguously (like a columnar database) and compressed with type-specific algorithms (delta-of-delta for timestamps, Gorilla for floats, dictionary encoding for low-cardinality text). This routinely achieves **90–96% storage reduction** on typical metrics/IoT data.

```sql
-- Enable compression, defining how rows should be grouped and ordered within compressed chunks
ALTER TABLE conditions SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id',   -- group by this (like a GROUP BY column)
  timescaledb.compress_orderby   = 'time DESC'    -- sort within segment (like an ORDER BY)
);

-- Automatically compress chunks older than 7 days
SELECT add_compression_policy('conditions', INTERVAL '7 days');

-- Manually compress a specific chunk
SELECT compress_chunk('_timescaledb_internal._hyper_1_2_chunk');

-- Check compression ratio
SELECT
  pg_size_pretty(before_compression_total_bytes) AS before,
  pg_size_pretty(after_compression_total_bytes)  AS after
FROM hypertable_compression_stats('conditions');
```

> **`compress_segmentby` is the single most important tuning knob.** It should be the column(s) you most often filter/group by (e.g., `device_id`). Segmenting correctly lets Timescale skip decompressing whole segments that don't match your `WHERE` clause.

---

### 19. **What is the tradeoff of compression — can you still write to compressed chunks?**

**Answer:** Yes, but with caveats:

| Operation                       | On compressed chunk                                           |
| -------------------------------- | --------------------------------------------------------------- |
| `SELECT`                        | ✅ Works, decompressed transparently at query time               |
| `INSERT` (new rows)             | ✅ Supported — goes into an uncompressed "staging" area automatically |
| `UPDATE` / `DELETE`             | ✅ Supported since Timescale 2.11+, but slower (decompress → modify → recompress) |
| Adding an index                  | ⚠️ Only on segmentby columns is it as fast as uncompressed |

**The real tradeoff:** compression trades **write/update speed on old data** for **massive storage savings and faster analytical scans** (fewer bytes read from disk = faster aggregate queries). This is why the standard pattern is: keep recent data (still being actively written/corrected) **uncompressed**, and compress data past a certain age via `add_compression_policy`.

---

### 20. **Walk through choosing `compress_segmentby` and `compress_orderby` for a real schema.**

**Answer:** Consider a multi-tenant metrics table:

```sql
CREATE TABLE metrics (
  time      TIMESTAMPTZ NOT NULL,
  tenant_id UUID NOT NULL,
  metric    TEXT NOT NULL,
  value     DOUBLE PRECISION
);
SELECT create_hypertable('metrics', by_range('time'));
```

Typical queries:

```sql
-- Query pattern: always filters by tenant, often by metric name too, always ordered by time
SELECT time, value FROM metrics
WHERE tenant_id = $1 AND metric = 'cpu_usage' AND time > now() - INTERVAL '1 hour'
ORDER BY time;
```

**Decision:**

```sql
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'tenant_id, metric',  -- match WHERE-clause equality filters
  timescaledb.compress_orderby   = 'time DESC'            -- match ORDER BY
);
```

- `segmentby` = columns you filter on with **equality** — Timescale can skip entire segments that don't match
- `orderby` = columns you range-filter/sort on **within** a segment — usually `time DESC` since most queries want recent-first
- Avoid segmenting by high-cardinality columns with few rows each (e.g., a unique request ID) — that produces too many tiny segments and defeats compression

---

## Continuous Aggregates & Rollups

### 21. **What is a continuous aggregate, and how is it different from a materialized view?**

**Answer:** A **continuous aggregate** is a special materialized view that TimescaleDB **incrementally and automatically refreshes** as new data arrives, instead of recomputing from scratch. It's built specifically for `time_bucket()` rollups (hourly, daily averages, etc.) over hypertables.

| Feature                     | Regular Materialized View                | Continuous Aggregate                                  |
| ---------------------------- | ------------------------------------------ | --------------------------------------------------------- |
| Refresh                     | Full recompute (`REFRESH MATERIALIZED VIEW`) | Incremental — only new/changed time buckets recomputed |
| Refresh trigger             | Manual, or cron                            | Background job on a schedule (`add_continuous_aggregate_policy`) |
| Query on stale + new data    | Not automatic                              | Real-time aggregation option merges live data automatically |
| Storage                     | Full result set                            | Stores pre-aggregated rollups (much smaller)              |

```sql
CREATE MATERIALIZED VIEW conditions_hourly
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  device_id,
  avg(temperature) AS avg_temp,
  max(temperature) AS max_temp,
  min(temperature) AS min_temp,
  count(*) AS reading_count
FROM conditions
GROUP BY bucket, device_id;

-- Automatically keep it refreshed
SELECT add_continuous_aggregate_policy('conditions_hourly',
  start_offset      => INTERVAL '3 hours',
  end_offset        => INTERVAL '1 hour',
  schedule_interval  => INTERVAL '1 hour'
);
```

> **`start_offset`/`end_offset`** define the refresh window relative to "now" — here, it refreshes buckets from 3 hours ago up to 1 hour ago every hour, leaving a 1-hour buffer for late-arriving data before finalizing a bucket.

---

### 22. **What is "real-time aggregation" in continuous aggregates?**

**Answer:** By default, when you `SELECT` from a continuous aggregate, Timescale **transparently unions** the materialized (already-rolled-up) data with a live aggregate over the raw data that hasn't been materialized yet. This means querying `conditions_hourly` always reflects the very latest inserts, not just what's been refreshed so far — without you writing any extra SQL.

```sql
-- This always includes the most recent, not-yet-materialized minute of data too
SELECT * FROM conditions_hourly WHERE bucket > now() - INTERVAL '1 day';

-- Real-time aggregation can be disabled if you want raw materialized-only speed
ALTER MATERIALIZED VIEW conditions_hourly SET (timescaledb.materialized_only = true);
```

> Real-time aggregation is a big reason continuous aggregates beat hand-rolled cron + `REFRESH MATERIALIZED VIEW` pipelines — no "stale until next refresh" window for dashboards.

---

### 23. **How do you build hierarchical rollups (hourly → daily → monthly)?**

**Answer:** Continuous aggregates can be built **on top of other continuous aggregates**, so you don't re-scan raw data for every rollup level.

```sql
-- Level 1: raw → hourly (already created above: conditions_hourly)

-- Level 2: hourly → daily, built FROM the hourly aggregate, not raw data
CREATE MATERIALIZED VIEW conditions_daily
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 day', bucket) AS day,
  device_id,
  avg(avg_temp) AS avg_temp,        -- average of averages (fine for evenly-sized buckets)
  max(max_temp) AS max_temp,
  min(min_temp) AS min_temp,
  sum(reading_count) AS reading_count
FROM conditions_hourly
GROUP BY day, device_id;

SELECT add_continuous_aggregate_policy('conditions_daily',
  start_offset => INTERVAL '3 days',
  end_offset   => INTERVAL '1 day',
  schedule_interval => INTERVAL '1 day');
```

> This is a common **interview follow-up**: "how would you avoid re-aggregating billions of raw rows every time you need a monthly report?" — the answer is chained/hierarchical continuous aggregates.

---

## Data Lifecycle Management

### 24. **How do you automatically expire old data?**

**Answer:** `add_retention_policy()` schedules a background job that drops entire chunks once all their data is older than the given interval. Because it operates at the **chunk level** (dropping whole files), it's dramatically cheaper than `DELETE FROM ... WHERE time < ...`, which has to scan and remove individual rows and leaves dead tuples for `VACUUM`.

```sql
-- Drop chunks entirely older than 90 days
SELECT add_retention_policy('conditions', INTERVAL '90 days');

-- Inspect scheduled policies
SELECT * FROM timescaledb_information.jobs
WHERE proc_name = 'policy_retention';

-- Remove a policy
SELECT remove_retention_policy('conditions');
```

```sql
-- ❌ Avoid this at scale — row-by-row delete, generates WAL + dead tuples
DELETE FROM conditions WHERE time < now() - INTERVAL '90 days';

-- ✅ What retention policies do under the hood, essentially:
SELECT drop_chunks('conditions', older_than => INTERVAL '90 days');
```

---

### 25. **What is a typical "hot-warm-cold" data lifecycle policy in TimescaleDB?**

**Answer:** Combine compression, tiering, and retention into stages matched to how data is actually accessed:

| Stage    | Age            | Storage                     | Policy                                              |
| -------- | -------------- | ---------------------------- | ---------------------------------------------------- |
| **Hot**  | 0–7 days       | Uncompressed, on fast SSD     | Actively written/updated, queried frequently          |
| **Warm** | 7 days–1 year  | Compressed, on primary storage | Read-mostly, compressed for storage + scan efficiency |
| **Cold** | > 1 year       | Tiered to object storage (S3) | Rarely queried, cheapest storage, still SQL-queryable |
| **Expired** | > retention window | Deleted                  | Dropped entirely via retention policy                  |

```sql
-- Warm: compress after 7 days
SELECT add_compression_policy('conditions', INTERVAL '7 days');

-- Cold: tier to low-cost object storage after 1 year (Timescale's tiered storage)
SELECT add_tiering_policy('conditions', INTERVAL '1 year');

-- Expired: drop entirely after 5 years (e.g., for compliance/cost reasons)
SELECT add_retention_policy('conditions', INTERVAL '5 years');
```

> **Interview framing:** this three-tier pattern (hot/warm/cold) is the single most-asked "design a time-series pipeline" question — being able to name all three policy functions and explain *why* each stage exists is what separates a strong answer.

---

### 26. **How do you resize an existing table into a hypertable without downtime? How do you resize chunk intervals afterward?**

**Answer:** "Resizing" here covers two different operations that are often conflated in interviews:

1. **Converting a plain table into a hypertable** — `create_hypertable()` on a table that already has data works directly. It scans existing rows, buckets them by timestamp into the configured chunk interval, and creates the chunk tables under the hood. This runs as a single `ALTER TABLE`-like operation that takes a brief lock, so for small-to-medium tables it's effectively instant; for very large tables you migrate via a new table + backfill to avoid blocking writes.

2. **Resizing chunk time intervals on an already-existing hypertable** — the chunk interval (e.g. 1 day vs 1 week) is not fixed for life. You can change it going forward with `set_chunk_time_interval()`, which affects only *newly created* chunks; existing chunks keep their original size unless you explicitly reshape them.

```sql
-- Convert an existing table in place (works if the table isn't huge / a short lock is acceptable)
SELECT create_hypertable('conditions', by_range('time'), migrate_data => true);
```

```sql
-- Resize the chunk interval on an existing hypertable (applies only to future chunks)
SELECT set_chunk_time_interval('conditions', INTERVAL '1 day');
```

> **Interview framing:** interviewers probe whether you know chunk interval changes are *not retroactive*. If old chunks were sized too large (causing slow queries/compression) or too small (causing chunk-count bloat), you must either let old data age out naturally under the new policy or actively reshape old chunks — Timescale doesn't silently re-chunk historical data for you.

**How resizing existing (already-written) chunks is actually achieved**, since `set_chunk_time_interval()` alone won't touch them:

```sql
-- Zero-downtime pattern for huge tables — also the way to effectively
-- "re-chunk" historical data to a new interval size:
-- 1. Create a new hypertable with the desired (corrected) chunk interval
CREATE TABLE conditions_new (LIKE conditions INCLUDING ALL);
SELECT create_hypertable('conditions_new', by_range('time'), chunk_time_interval => INTERVAL '1 day');

-- 2. Backfill in batches (avoids one giant long-running transaction/lock,
--    and lets you throttle I/O instead of copying the whole table at once)
INSERT INTO conditions_new
SELECT * FROM conditions WHERE time BETWEEN '2025-01-01' AND '2025-02-01';
-- ... repeat per time range until fully backfilled ...

-- 3. Cut application writes over to conditions_new, then swap names
--    so the table name your app queries never has to change
BEGIN;
ALTER TABLE conditions RENAME TO conditions_old;
ALTER TABLE conditions_new RENAME TO conditions;
COMMIT;

-- 4. Once verified, drop the old table to reclaim space
DROP TABLE conditions_old;
```

This backfill-and-swap is the general-purpose tool for both problems: it's how you convert a huge existing table into a hypertable without a long lock, *and* it's how you rewrite historical data into differently-sized chunks — because `set_chunk_time_interval()` genuinely cannot resize chunks that already exist on disk.

---

## Advanced Topics

### 27. **What is `pg_partman` vs TimescaleDB's native chunking, and when would you still consider plain Postgres partitioning?**

**Answer:** `pg_partman` is a general-purpose Postgres extension that automates *native* partition management (creating/dropping partitions on a schedule) without requiring hypertables. TimescaleDB's chunking is purpose-built for time-series and adds compression, continuous aggregates, and time-series-aware query planning on top — capabilities `pg_partman` alone doesn't provide.

**When plain Postgres partitioning (± pg_partman) may still be enough:**

- Data isn't really time-series (e.g., partitioning by tenant/region for isolation, not by time)
- You don't need compression or continuous aggregates — just want to bound partition size for maintenance
- You want to avoid adding an extension/dependency and your scale is modest (tens of millions of rows, not billions)

**When to reach for TimescaleDB specifically:**

- High-volume time-stamped data (metrics, IoT, events, financial ticks)
- You need `time_bucket()`/gapfill/continuous aggregates for rollups
- You need compression for storage cost, and hot/warm/cold tiering

---

### 28. **How do JOINs between a hypertable and a regular relational table work?**

**Answer:** Exactly like normal Postgres — a hypertable is queried with standard SQL, so you can `JOIN` it against normal tables (dimension tables, metadata, users) with zero special syntax. This is one of TimescaleDB's biggest selling points over purpose-built time-series databases (like InfluxDB) that can't easily join against relational data.

```sql
CREATE TABLE devices (
  device_id TEXT PRIMARY KEY,
  location  TEXT,
  model     TEXT
);

-- Normal JOIN — Timescale pushes the time filter down to hypertable chunk exclusion first
SELECT d.location, avg(c.temperature) AS avg_temp
FROM conditions c
JOIN devices d ON c.device_id = d.device_id
WHERE c.time > now() - INTERVAL '1 day'
  AND d.location = 'warehouse-3'
GROUP BY d.location;
```

> This is a strong talking point in interviews: "why TimescaleDB over InfluxDB/a dedicated TSDB?" — you get relational joins, foreign keys, and full SQL for free, because it *is* Postgres.

---

### 29. **How does `approx_percentile` work, and why use it instead of `percentile_cont`?**

**Answer:** `approx_percentile` is one of TimescaleDB's **hyperfunctions** — it computes an approximate percentile from a **T-Digest sketch** (built by `percentile_agg`) instead of sorting the full dataset. Exact `percentile_cont` requires holding and sorting every row per group in memory, which is infeasible once a group spans millions/billions of rows. The T-Digest sketch is small, mergeable, and gives accuracy within a small, tunable error bound — more than good enough for p95/p99 latency or sensor dashboards.

```sql
-- percentile_agg() builds a compact T-Digest sketch per group;
-- approx_percentile() reads a percentile out of that sketch
SELECT
  device_id,
  approx_percentile(0.95, percentile_agg(temperature)) AS p95_temp,
  approx_percentile(0.99, percentile_agg(temperature)) AS p99_temp
FROM conditions
WHERE time > now() - INTERVAL '7 days'
GROUP BY device_id;

-- Because the sketch is mergeable, you can pre-aggregate it in a continuous
-- aggregate and cheaply roll it up further (hourly -> daily) without re-scanning
-- raw rows or losing accuracy from double-approximation
CREATE MATERIALIZED VIEW conditions_hourly_stats
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 hour', time) AS bucket,
  device_id,
  percentile_agg(temperature) AS temp_sketch   -- stores the sketch, not the raw values
FROM conditions
GROUP BY bucket, device_id;

-- Roll the hourly sketches up into a daily p95 later — merges sketches instead
-- of re-reading raw data
SELECT
  time_bucket('1 day', bucket) AS day,
  device_id,
  approx_percentile(0.95, rollup(temp_sketch)) AS p95_temp
FROM conditions_hourly_stats
GROUP BY day, device_id;
```

> **Why this matters:** storing the *sketch* (via `percentile_agg`) rather than the final number in a continuous aggregate lets you merge/re-aggregate percentiles correctly across time buckets — you can't average or re-derive a p95 from already-computed p95 values, but you *can* merge T-Digest sketches with `rollup()`.

---

### 30. **How does `candlestick_agg` compute OHLC data, and when would you use it?**

**Answer:** `candlestick_agg` is a hyperfunction purpose-built for financial/tick-style data — it computes **Open/High/Low/Close (OHLC)** plus volume in a single aggregation pass over `(time, price, volume)` tuples, instead of writing four separate `first()`/`max()`/`min()`/`last()` queries (or worse, self-joins) to derive each value.

```sql
-- One pass builds a candlestick object per (day, symbol) bucket
SELECT
  time_bucket('1 day', time) AS day,
  symbol,
  candlestick_agg(time, price, volume) AS candlestick
FROM trades
GROUP BY day, symbol;

-- Extract individual OHLC + volume fields from the candlestick object
SELECT
  day,
  symbol,
  open(candlestick)         AS open_price,
  high(candlestick)         AS high_price,
  low(candlestick)          AS low_price,
  close(candlestick)        AS close_price,
  volume(candlestick)       AS total_volume
FROM (
  SELECT
    time_bucket('1 day', time) AS day,
    symbol,
    candlestick_agg(time, price, volume) AS candlestick
  FROM trades
  GROUP BY day, symbol
) t;

-- Candlesticks are mergeable too — store per-minute candlesticks in a continuous
-- aggregate, then roll them up into hourly/daily candles without re-touching raw trades
SELECT
  time_bucket('1 hour', day) AS hour,
  symbol,
  rollup(candlestick) AS hourly_candlestick
FROM minute_candlesticks
GROUP BY hour, symbol;
```

> **Why this matters:** without `candlestick_agg`, computing OHLC requires either 4 separate window/aggregate functions or an ordered-set self-join — both far more expensive at scale. Because the candlestick object is mergeable via `rollup()`, you get the same incremental-rollup benefit as `approx_percentile`: compute once at fine granularity, merge cheaply at coarser granularities.

---

### 31. **How would you design a schema for a multi-tenant SaaS metrics platform on TimescaleDB?**

**Answer:** A realistic system-design-style question. Key decisions to call out:

```sql
CREATE TABLE metrics (
  time      TIMESTAMPTZ NOT NULL,
  tenant_id UUID NOT NULL,
  metric_name TEXT NOT NULL,
  value     DOUBLE PRECISION NOT NULL,
  tags      JSONB
);

SELECT create_hypertable('metrics', by_range('time'));

-- Index to support "give me this tenant's metric over a time range" — the dominant query pattern
CREATE INDEX idx_metrics_tenant_metric_time ON metrics (tenant_id, metric_name, time DESC);

-- Compress older data, segmented by the columns most queries filter on
ALTER TABLE metrics SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'tenant_id, metric_name',
  timescaledb.compress_orderby   = 'time DESC'
);
SELECT add_compression_policy('metrics', INTERVAL '3 days');

-- Pre-aggregate for dashboards so tenant dashboards never scan raw rows
CREATE MATERIALIZED VIEW metrics_5min
WITH (timescaledb.continuous) AS
SELECT time_bucket('5 minutes', time) AS bucket, tenant_id, metric_name,
       avg(value) AS avg_value, max(value) AS max_value, min(value) AS min_value
FROM metrics
GROUP BY bucket, tenant_id, metric_name;

SELECT add_continuous_aggregate_policy('metrics_5min',
  start_offset => INTERVAL '15 minutes', end_offset => INTERVAL '5 minutes',
  schedule_interval => INTERVAL '5 minutes');

-- Tiered lifecycle: keep raw data lean, roll off cold data
SELECT add_retention_policy('metrics', INTERVAL '30 days');       -- raw data expires fast
SELECT add_retention_policy('metrics_5min', INTERVAL '2 years');   -- rollups kept much longer
```

**Talking points an interviewer wants to hear:**

- Composite index (or `compress_segmentby`) matching the dominant `WHERE tenant_id = ... AND metric_name = ...` filter
- Raw high-resolution data has a **short** retention (cheap to store briefly, expensive forever); aggregates have **long** retention (tiny relative to raw volume)
- Continuous aggregates offload dashboard load from raw hypertable scans entirely
- `tags JSONB` gives schema flexibility for heterogeneous per-tenant metric metadata without a table per tenant

---

### 32. **What are common pitfalls / anti-patterns when using TimescaleDB?**

**Answer:**

| Pitfall                                              | Why it hurts                                                                 | Fix                                                              |
| ------------------------------------------------------ | ------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| Single-row inserts in a tight loop                    | Network round-trip overhead dominates                                          | Batch inserts / `COPY`                                              |
| Chunk interval too small (e.g., minutes)               | Thousands of chunks → planning overhead, too many files                        | Size chunks so recent chunk + indexes fit in memory (often 1 day–1 week) |
| Chunk interval too large (e.g., years)                 | Huge indexes per chunk, poor compression granularity, slow chunk exclusion    | Match interval to query/retention patterns                          |
| `compress_segmentby` on high-cardinality/unused column  | Too many tiny segments, no scan-skipping benefit, worse compression ratio     | Segment by actual equality-filter columns                            |
| Using `DELETE` for retention instead of `drop_chunks`   | Row-by-row deletes bloat tables, need VACUUM, don't reclaim disk immediately  | Use `add_retention_policy` (chunk-level drop)                        |
| Forgetting `WHERE` bounds with `time_bucket_gapfill`    | Function errors — it needs explicit start/end to know what to fill            | Always bound the query range explicitly                              |
| Not indexing hypertables like normal Postgres tables    | Assuming Timescale "just handles" all query patterns via chunking alone       | Still add B-tree/GIN indexes as needed — chunking ≠ indexing         |
| Running `UPDATE`/heavy write patterns on compressed chunks | Each touched row decompresses/recompresses its whole segment — slow          | Keep the compression boundary (`add_compression_policy` interval) past the point where data stabilizes |

---

### 33. **How does TimescaleDB fit into a high-availability / scaling architecture?**

**Answer:** Because TimescaleDB is a Postgres extension, it inherits Postgres's HA and scaling tools directly — no separate ecosystem to learn.

```
┌──────────────┐   Streaming Replication   ┌───────────────┐
│   Primary    │ ────────────────────────→ │    Replica    │
│ (hypertable  │                           │  (read-only    │
│  writes)     │                           │   queries)     │
└──────────────┘                           └───────────────┘
```

| Concern             | Approach                                                                 |
| -------------------- | --------------------------------------------------------------------------- |
| High availability     | Standard Postgres streaming replication, Patroni, or **Timescale's managed HA replicas** |
| Read scaling          | Read replicas — continuous aggregates + compressed chunks make replicas cheap to keep in sync |
| Write scaling          | Space partitioning (multi-dimensional hypertables) to spread ingest; or **Timescale Multi-Node** (distributed hypertables) for horizontal write scale in self-managed deployments |
| Backup/restore        | `pg_dump`/`pg_basebackup` work as usual; Timescale Cloud offers continuous backup |
| Storage cost at scale | Compression (90%+ reduction) + tiered storage to object storage for cold chunks |

> **Interview soundbite:** "TimescaleDB doesn't reinvent HA — it inherits 25+ years of PostgreSQL replication and tooling, then adds time-series-specific scaling levers (compression, chunk exclusion, space partitioning) on top."

---
