# Elasticsearch Interview Questions & Answers

A comprehensive collection of **easy to medium** Elasticsearch interview Q&A with practical examples and theoretical refreshers. Covers architecture, indexing, querying, scalability, resiliency, performance, and real-world patterns.

---

## Fundamentals

### 1. **What is Elasticsearch and how does it differ from a traditional database?**

**Answer:** Elasticsearch is a **distributed, RESTful search and analytics engine** built on top of Apache Lucene. Unlike traditional databases optimized for CRUD by primary key, Elasticsearch is optimized for **full-text search, filtering, and aggregations** across large volumes of unstructured or semi-structured data.

| Feature         | Traditional RDBMS     | Elasticsearch                  |
| --------------- | --------------------- | ------------------------------ |
| Primary purpose | Transactional CRUD    | Search & analytics             |
| Query language  | SQL                   | Query DSL (JSON) + SQL         |
| Schema          | Strict (DDL)          | Dynamic or explicit mapping    |
| Best at         | JOINs, transactions   | Full-text search, aggregations |
| Data model      | Tables, rows, columns | Indices, documents (JSON)      |
| Normalization   | Encouraged            | Denormalization encouraged     |
| ACID            | ✅ Full               | ❌ Near-real-time, eventual    |

---

### 2. **Explain the core terminology of Elasticsearch.**

**Answer:**

| Elasticsearch Term | Relational DB Analogy | Description                                          |
| ------------------ | --------------------- | ---------------------------------------------------- |
| **Index**          | Database/Table        | Collection of documents with similar characteristics |
| **Document**       | Row                   | A single JSON record                                 |
| **Field**          | Column                | A key-value pair in a document                       |
| **Mapping**        | Schema                | Defines field types and how they're indexed          |
| **Shard**          | Partition             | A subdivision of an index (a Lucene index)           |
| **Replica**        | Read replica          | Copy of a primary shard for HA and read throughput   |
| **Node**           | Server                | A single Elasticsearch instance                      |
| **Cluster**        | Cluster               | A group of nodes                                     |

```json
// A document in the "products" index
{
  "_index": "products",
  "_id": "prod-001",
  "_source": {
    "name": "Wireless Headphones",
    "description": "Noise-cancelling Bluetooth headphones with 30hr battery",
    "price": 79.99,
    "category": "electronics",
    "tags": ["bluetooth", "wireless", "noise-cancelling"],
    "created_at": "2026-01-15T10:30:00Z"
  }
}
```

---

### 3. **What is an inverted index and why is it fundamental to Elasticsearch?**

**Answer:** An inverted index maps **terms to the documents** that contain them — the opposite of a forward index (document → terms). This makes full-text search **O(1)** per term lookup instead of scanning every document.

```
Documents:
  doc1: "quick brown fox"
  doc2: "quick red car"
  doc3: "brown car drives fast"

Inverted Index:
  "quick"  → [doc1, doc2]
  "brown"  → [doc1, doc3]
  "fox"    → [doc1]
  "red"    → [doc2]
  "car"    → [doc2, doc3]
  "drives" → [doc3]
  "fast"   → [doc3]

Search "brown car" → intersect [doc1, doc3] ∩ [doc2, doc3] = [doc3]
```

> **Key insight:** Every field in Elasticsearch has its own inverted index. `keyword` fields store exact values; `text` fields are analyzed (tokenized, lowercased, stemmed) before indexing.

---

### 4. **What is the difference between `text` and `keyword` field types?**

**Answer:**

| Aspect         | `text`                                        | `keyword`                          |
| -------------- | --------------------------------------------- | ---------------------------------- |
| Analyzed?      | ✅ Tokenized, lowercased, stemmed             | ❌ Stored as-is                    |
| Use case       | Full-text search                              | Exact match, sorting, aggregations |
| Example value  | "Quick Brown Fox" → ["quick", "brown", "fox"] | "Quick Brown Fox" (unchanged)      |
| Can sort?      | ❌ (use `.keyword` sub-field)                 | ✅                                 |
| Can aggregate? | ❌                                            | ✅                                 |

```json
// Mapping with both
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "status": { "type": "keyword" },
      "description": { "type": "text", "analyzer": "english" }
    }
  }
}
```

```json
// Search on text field (full-text, analyzed)
{ "query": { "match": { "title": "wireless headphone" } } }

// Filter on keyword (exact match, fast)
{ "query": { "term": { "status": "active" } } }

// Sort by keyword sub-field
{ "sort": [{ "title.keyword": "asc" }] }
```

---

### 5. **What are analyzers and how do they work?**

**Answer:** An analyzer processes text during indexing and search. It has 3 stages:

```
Input: "The Quick BROWN Fox's jumped!"
          │
          ▼
  ┌──────────────────┐
  │ Character Filter  │  → Strips HTML, replaces characters
  │ (html_strip)      │  → "The Quick BROWN Fox's jumped!"
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │    Tokenizer      │  → Splits into tokens
  │   (standard)      │  → ["The", "Quick", "BROWN", "Fox's", "jumped"]
  └────────┬─────────┘
           ▼
  ┌──────────────────┐
  │   Token Filters   │  → Transforms tokens
  │ (lowercase, stop, │  → ["quick", "brown", "fox", "jump"]
  │  stemmer)         │
  └──────────────────┘
```

| Built-in Analyzer | Behavior                                    | Best For          |
| ----------------- | ------------------------------------------- | ----------------- |
| `standard`        | Unicode tokenization + lowercase            | General purpose   |
| `english`         | Standard + English stemming + stop words    | English content   |
| `keyword`         | No tokenization (entire value as one token) | IDs, codes        |
| `whitespace`      | Split on whitespace only                    | Log lines         |
| `pattern`         | Split on regex pattern                      | Custom delimiters |

```json
// Test an analyzer without indexing
POST _analyze
{
  "analyzer": "english",
  "text": "The foxes were running quickly"
}
// Tokens: ["fox", "run", "quick"]
```

---

### 6. **What is the difference between `match`, `term`, and `bool` queries?**

**Answer:**

```json
// match — full-text search (analyzes the query string)
{ "query": { "match": { "description": "noise cancelling headphones" } } }
// Finds: "Best noise-cancelling headphones", "Wireless headphones with cancellation"

// term — exact match (NO analysis, use with keyword fields)
{ "query": { "term": { "status": "active" } } }
// ❌ DON'T use term on text fields — "Active" won't match "active"

// bool — combines multiple conditions
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "headphones" } }
      ],
      "filter": [
        { "term": { "category": "electronics" } },
        { "range": { "price": { "gte": 50, "lte": 200 } } }
      ],
      "should": [
        { "match": { "tags": "wireless" } }
      ],
      "must_not": [
        { "term": { "status": "discontinued" } }
      ]
    }
  }
}
```

| Clause     | Affects Score? | Purpose                                          |
| ---------- | :------------: | ------------------------------------------------ |
| `must`     |       ✅       | Required conditions (like AND)                   |
| `filter`   |       ❌       | Required conditions, cached, faster (no scoring) |
| `should`   |       ✅       | Optional boost (like OR)                         |
| `must_not` |       ❌       | Exclusion (like NOT)                             |

> **Performance tip:** Use `filter` instead of `must` when you don't need relevance scoring. Filters are cached and reused.

---

### 7. **How does relevance scoring work in Elasticsearch?**

**Answer:** Elasticsearch uses **BM25** (Best Matching 25) by default to rank documents by relevance.

```
BM25 Score = IDF × (TF × (k1 + 1)) / (TF + k1 × (1 - b + b × fieldLength/avgFieldLength))

Where:
  TF  = Term Frequency — how often the term appears in the document
  IDF = Inverse Document Frequency — rare terms score higher
  k1  = Term frequency saturation (default: 1.2)
  b   = Field length normalization (default: 0.75)
```

```json
// Explain a score
GET products/_search
{
  "explain": true,
  "query": { "match": { "description": "wireless headphones" } }
}

// Boost specific fields
{
  "query": {
    "multi_match": {
      "query": "wireless headphones",
      "fields": ["title^3", "description^1", "tags^2"],
      "type": "best_fields"
    }
  }
}
// title matches are 3× more important than description
```

---

## Indexing & Mappings

### 8. **How do you design an index mapping for a product catalog?**

**Answer:**

```json
PUT products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "english_stemmer", "synonym_filter"]
        }
      },
      "filter": {
        "english_stemmer": { "type": "stemmer", "language": "english" },
        "synonym_filter": {
          "type": "synonym",
          "synonyms": ["headphone,earphone,earbuds", "laptop,notebook"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name":        { "type": "text", "analyzer": "product_analyzer", "fields": { "keyword": { "type": "keyword" } } },
      "description": { "type": "text", "analyzer": "product_analyzer" },
      "price":       { "type": "float" },
      "category":    { "type": "keyword" },
      "tags":        { "type": "keyword" },
      "brand":       { "type": "keyword" },
      "rating":      { "type": "float" },
      "in_stock":    { "type": "boolean" },
      "created_at":  { "type": "date" },
      "location":    { "type": "geo_point" }
    }
  }
}
```

> **Mapping rules:**
>
> - Once a field is mapped, you **cannot change its type** (must reindex)
> - Set `"dynamic": "strict"` to reject unexpected fields
> - Use `"enabled": false` for fields you store but never search

---

### 9. **What is dynamic mapping and when should you disable it?**

**Answer:** Dynamic mapping auto-detects field types when you index a document without pre-defining a mapping.

| Detected JSON  | Elasticsearch Type | Problem?                          |
| -------------- | ------------------ | --------------------------------- |
| `"hello"`      | `text` + `keyword` | Usually OK                        |
| `"2026-01-15"` | `date`             | Can break if later value is "N/A" |
| `42`           | `long`             | Wastes space if values are small  |
| `3.14`         | `float`            | OK                                |
| `true`         | `boolean`          | OK                                |

```json
// Strict mode — reject unknown fields
PUT products
{
  "mappings": {
    "dynamic": "strict",
    "properties": { ... }
  }
}
// Indexing a doc with an unmapped field → 400 error

// Runtime mode — detect but don't add to mapping
PUT products
{
  "mappings": {
    "dynamic": "runtime",
    "properties": { ... }
  }
}
// Unknown fields queryable but not physically indexed (slower)
```

> **Best practice:** Use `"dynamic": "strict"` in production. Uncontrolled dynamic mapping leads to mapping explosion (too many fields), index bloat, and cluster instability.

---

### 10. **What is the difference between indexing and reindexing?**

**Answer:**

```json
// Indexing — add or update a single document
PUT products/_doc/prod-001
{
  "name": "Wireless Headphones",
  "price": 79.99
}

// Bulk indexing — efficient for large batches
POST _bulk
{ "index": { "_index": "products", "_id": "prod-001" } }
{ "name": "Wireless Headphones", "price": 79.99 }
{ "index": { "_index": "products", "_id": "prod-002" } }
{ "name": "USB Cable", "price": 9.99 }

// Reindex — copy documents from one index to another (used for mapping changes)
POST _reindex
{
  "source": { "index": "products_v1" },
  "dest": { "index": "products_v2" }
}
```

| Operation       | When to use                                         |
| --------------- | --------------------------------------------------- |
| Index (single)  | Real-time writes, updates                           |
| Bulk index      | Initial data load, batch imports                    |
| Reindex         | Mapping change, shard count change, version upgrade |
| Update by query | Modify existing docs in place                       |
| Delete by query | Remove matching docs                                |

> **Zero-downtime reindex pattern:** Use an alias (`products` → `products_v1`). Reindex to `products_v2`. Swap alias atomically: `products` → `products_v2`.

---

## Search Patterns

### 11. **How do you implement autocomplete / search-as-you-type?**

**Answer:** Use the `search_as_you_type` field type or `edge_ngram` tokenizer.

```json
// Option 1: search_as_you_type (built-in)
PUT products
{
  "mappings": {
    "properties": {
      "name": { "type": "search_as_you_type" }
    }
  }
}

// Query
GET products/_search
{
  "query": {
    "multi_match": {
      "query": "wire head",
      "type": "bool_prefix",
      "fields": ["name", "name._2gram", "name._3gram"]
    }
  }
}
// Matches: "Wireless Headphones", "Wired Headset"

// Option 2: Completion Suggester (fastest, in-memory FST)
PUT products
{
  "mappings": {
    "properties": {
      "suggest": { "type": "completion" }
    }
  }
}

POST products/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "wire",
      "completion": { "field": "suggest", "size": 5, "fuzzy": { "fuzziness": 1 } }
    }
  }
}
```

---

### 12. **How do aggregations work in Elasticsearch?**

**Answer:** Aggregations are Elasticsearch's analytics framework — like SQL GROUP BY on steroids.

```json
GET products/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 20 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "price_ranges": {
          "range": {
            "field": "price",
            "ranges": [
              { "to": 50, "key": "budget" },
              { "from": 50, "to": 150, "key": "mid-range" },
              { "from": 150, "key": "premium" }
            ]
          }
        }
      }
    },
    "monthly_sales": {
      "date_histogram": {
        "field": "created_at",
        "calendar_interval": "month"
      }
    }
  }
}
```

| Aggregation Type | Examples                                                 | Purpose                   |
| ---------------- | -------------------------------------------------------- | ------------------------- |
| **Bucket**       | `terms`, `range`, `date_histogram`, `filters`            | Group documents           |
| **Metric**       | `avg`, `sum`, `min`, `max`, `cardinality`, `percentiles` | Compute stats             |
| **Pipeline**     | `moving_avg`, `derivative`, `cumulative_sum`             | Aggregate on aggregations |

---

### 13. **How do you handle typos and fuzzy matching?**

**Answer:** Elasticsearch supports fuzzy matching based on **Levenshtein edit distance**.

```json
// Fuzzy match — tolerates typos
{
  "query": {
    "match": {
      "name": {
        "query": "headphnes",
        "fuzziness": "AUTO"
      }
    }
  }
}
// AUTO fuzziness: 0 edits for 1-2 chars, 1 edit for 3-5 chars, 2 edits for 6+ chars
// "headphnes" → matches "headphones" (1 edit: insert 'o')

// Phonetic matching (sounds-like)
// Requires analysis-phonetic plugin
PUT products
{
  "settings": {
    "analysis": {
      "filter": {
        "phonetic_filter": { "type": "phonetic", "encoder": "double_metaphone" }
      },
      "analyzer": {
        "phonetic_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "phonetic_filter"]
        }
      }
    }
  }
}
```

---

### 14. **How do you implement faceted search (e-commerce style filters)?**

**Answer:**

```json
// Search with faceted filters
GET products/_search
{
  "query": {
    "bool": {
      "must": [{ "match": { "name": "headphones" } }],
      "filter": [
        { "term": { "brand": "Sony" } },
        { "range": { "price": { "gte": 50, "lte": 200 } } }
      ]
    }
  },
  "aggs": {
    "brands":     { "terms": { "field": "brand", "size": 20 } },
    "categories": { "terms": { "field": "category", "size": 20 } },
    "price_stats": { "stats": { "field": "price" } },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 50 }, { "from": 50, "to": 100 },
          { "from": 100, "to": 200 }, { "from": 200 }
        ]
      }
    }
  }
}
// Response: search results + counts per brand, category, price range
// Like Amazon's left sidebar filters with counts
```

---

## Distributed Architecture & Sharding

### 15. **How does sharding work in Elasticsearch?**

**Answer:** An index is split into **shards** (each is a self-contained Lucene index). Shards are distributed across cluster nodes.

```
Index: products (3 primary shards, 1 replica each)

Node 1:  [P0] [R1] [R2]
Node 2:  [P1] [R0] [R2']  ← Not shown: replicas are spread for HA
Node 3:  [P2] [R0'] [R1']

When you search:
1. Query sent to coordinating node
2. Coordinating node fans out to ALL primary shards (or replicas)
3. Each shard searches locally (Lucene)
4. Results merged (scatter-gather) by coordinating node
5. Top N returned to client
```

| Setting              | Guideline                  | Why                                           |
| -------------------- | -------------------------- | --------------------------------------------- |
| Shard size           | 10–50 GB each              | Too small: overhead. Too large: slow recovery |
| Shards per index     | Start with # of data nodes | 1 shard per node for balanced distribution    |
| Replicas             | At least 1                 | Fault tolerance + read throughput             |
| Max shards per node  | ~1,000                     | Avoid cluster instability                     |
| Total cluster shards | Monitor (< 10,000 ideal)   | JVM heap per shard overhead                   |

---

### 16. **How do you choose the number of shards for an index?**

**Answer:** You **cannot change primary shard count** after index creation (must reindex). Choose wisely.

```
Estimation formula:
  Expected data = 500 GB
  Target shard size = 30 GB
  Primary shards = 500 / 30 ≈ 17 shards

  With 1 replica: 17 × 2 = 34 total shards
  Need at least 2 nodes (replicas can't be on same node as primary)
```

| Scenario                    | Recommended Shards          | Why                        |
| --------------------------- | --------------------------- | -------------------------- |
| Small index (< 10 GB)       | 1 primary, 1 replica        | Overhead outweighs benefit |
| Medium (10–200 GB)          | 3–5 primaries               | Balanced parallelism       |
| Large (200 GB – 1 TB)       | 10–20 primaries             | Keep shards 20–50 GB       |
| Time-series (daily indices) | 1–3 per daily index         | ILM handles rollover       |
| Single search node          | 1 primary, 0 replicas (dev) | No HA needed               |

> **Over-sharding is the #1 operational mistake.** Each shard consumes heap (memory), file descriptors, and thread pool resources. 1,000 shards of 100 MB each is far worse than 10 shards of 10 GB.

---

### 17. **What are the different node roles in an Elasticsearch cluster?**

**Answer:**

| Role             | Responsibility                                             |       Scale Separately?       |
| ---------------- | ---------------------------------------------------------- | :---------------------------: |
| **Master**       | Cluster state management, index creation, shard allocation | ✅ (dedicated, odd number: 3) |
| **Data**         | Stores data, executes searches and aggregations            |     ✅ (add for capacity)     |
| **Ingest**       | Pre-process documents (pipelines)                          |   ✅ (if heavy transforms)    |
| **Coordinating** | Routes requests, merges results (no data)                  |  ✅ (for heavy search load)   |
| **ML**           | Machine learning jobs                                      |         ✅ (optional)         |
| **Transform**    | Continuous aggregation jobs                                |         ✅ (optional)         |

```yaml
# Dedicated master node (elasticsearch.yml)
node.roles: [master]

# Dedicated data node
node.roles: [data_content, data_hot, data_warm]

# Coordinating-only node (no roles = coordinating)
node.roles: []
```

> **Minimum production cluster:** 3 dedicated master nodes + 2+ data nodes. Never run master and data on the same node at scale.

---

### 18. **How does Elasticsearch handle node failures?**

**Answer:**

```
Scenario: Data node 2 fails (holds P1, R0)

Before failure:
  Node 1: [P0] [R1]
  Node 2: [P1] [R0]  ← dies
  Node 3: [R0'] [R1']

After detection (default ~1 min):
  1. Master detects node 2 is gone
  2. R1' on Node 3 promoted to P1 (was a replica)
  3. Master creates new R0 on Node 1 or Node 3
  4. Cluster is GREEN again (all primaries + replicas allocated)

Timeline:
  0s     — Node 2 fails
  ~30s   — Master detects (ping timeout)
  ~60s   — Shard reallocation starts
  ~5min  — New replica fully synced (depends on data size)
```

| Setting                                          | Default | Purpose                                     |
| ------------------------------------------------ | ------- | ------------------------------------------- |
| `cluster.fault_detection.leader_check.timeout`   | 10s     | Timeout for checking master                 |
| `cluster.fault_detection.follower_check.timeout` | 10s     | Timeout for checking data nodes             |
| `index.unassigned.node_left.delayed_timeout`     | 1m      | Wait before reallocating (handles restarts) |
| `cluster.routing.allocation.enable`              | all     | Can disable during maintenance              |

---

## Scalability & Performance

### 19. **How do you scale Elasticsearch for high read throughput?**

**Answer:**

| Strategy               | Impact                                 | When to use                       |
| ---------------------- | -------------------------------------- | --------------------------------- |
| Add replicas           | More shards serve search in parallel   | Read-heavy workload               |
| Add data nodes         | Distribute shards across more machines | Shards are clustered on few nodes |
| Use coordinating nodes | Offload merge/sort from data nodes     | Complex aggregations              |
| Query caching          | Repeated filter results cached         | Repeated queries (facets)         |
| `_source` filtering    | Return only needed fields              | Large documents                   |
| `search_after`         | Efficient deep pagination              | Paginating beyond 10,000          |

```json
// Increase replicas for read throughput
PUT products/_settings
{ "number_of_replicas": 2 }
// Now each shard has 2 replicas = 3× read capacity

// Use search_after instead of from/size for deep pagination
GET products/_search
{
  "size": 20,
  "sort": [{ "created_at": "desc" }, { "_id": "asc" }],
  "search_after": ["2026-03-01T00:00:00Z", "prod-500"]
}
```

---

### 20. **How do you optimize indexing (write) performance?**

**Answer:**

| Setting/Technique                     | Default | Optimized (bulk)    | Impact                          |
| ------------------------------------- | ------- | ------------------- | ------------------------------- |
| `refresh_interval`                    | 1s      | 30s or -1 (disable) | Less segment creation           |
| `number_of_replicas`                  | 1       | 0 (during bulk)     | No replica sync overhead        |
| Bulk size                             | —       | 5–15 MB per request | Optimal network + memory        |
| `index.translog.durability`           | request | async               | Less fsync (risk: lose last 5s) |
| `index.translog.flush_threshold_size` | 512 MB  | 1 GB                | Less frequent flushes           |

```json
// Optimize settings for initial bulk load
PUT products/_settings
{
  "refresh_interval": "-1",
  "number_of_replicas": 0
}

// Bulk index millions of documents...
POST _bulk
{ "index": { "_index": "products" } }
{ "name": "Product 1", "price": 10 }
...

// Restore settings after bulk load
PUT products/_settings
{
  "refresh_interval": "1s",
  "number_of_replicas": 1
}

// Force merge (reduce segments for faster search)
POST products/_forcemerge?max_num_segments=1
```

> **`refresh_interval`** controls when indexed documents become searchable (near-real-time). Default 1s means a 1-second delay. During bulk loads, disable it completely.

---

### 21. **What is Index Lifecycle Management (ILM)?**

**Answer:** ILM automates the lifecycle of time-series indices (logs, metrics, events) across hot → warm → cold → frozen → delete tiers.

```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": { "snapshot_repository": "my-repo" },
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

| Tier       | Storage                 | Performance | Cost | Data           |
| ---------- | ----------------------- | ----------- | ---- | -------------- |
| **Hot**    | Fast SSD                | Highest     | $$$$ | Today's data   |
| **Warm**   | SSD/HDD                 | Medium      | $$$  | Last 7–30 days |
| **Cold**   | Searchable snapshots    | Lower       | $$   | 30–90 days     |
| **Frozen** | Fully in object storage | Lowest      | $    | Archived       |
| **Delete** | —                       | —           | Free | Purged         |

---

### 22. **What are the key performance factors to monitor?**

**Answer:**

| Metric                | Healthy      | Warning                      | Critical                   |
| --------------------- | ------------ | ---------------------------- | -------------------------- |
| Cluster status        | 🟢 GREEN     | 🟡 YELLOW (missing replicas) | 🔴 RED (missing primaries) |
| JVM heap usage        | < 70%        | 70–85%                       | > 85% (GC storms)          |
| Search latency (p99)  | < 200ms      | 200–500ms                    | > 1s                       |
| Indexing rate         | Stable       | Dropping                     | Rejection errors           |
| Disk usage            | < 80%        | 80–90%                       | > 90% (read-only mode!)    |
| Pending tasks         | 0            | 1–5                          | > 10                       |
| Circuit breaker trips | 0            | Occasional                   | Frequent (OOM risk)        |
| Shard count           | < 1,000/node | 1,000–2,000                  | > 2,000                    |

```json
// Check cluster health
GET _cluster/health

// Check node stats
GET _nodes/stats/jvm,os,indices

// Find slow queries
GET _nodes/stats/indices/search

// Hot threads (debug CPU)
GET _nodes/hot_threads

// Shard allocation explanation
GET _cluster/allocation/explain
```

---

## Error Handling & Resiliency

### 23. **What errors should you handle when writing to Elasticsearch?**

**Answer:**

| Error                | HTTP Code |   Retryable?   | Cause                          | Action                                     |
| -------------------- | :-------: | :------------: | ------------------------------ | ------------------------------------------ |
| Version conflict     |    409    |    Depends     | Concurrent update to same doc  | Use `retry_on_conflict` or re-read + retry |
| Mapper parsing       |    400    |       ❌       | Document doesn't match mapping | Fix document or mapping                    |
| Circuit breaker      |    429    |       ✅       | JVM memory pressure            | Back off, reduce request size              |
| Bulk partial failure |    200    |    Partial     | Some docs in bulk failed       | Check `items[].error` in response          |
| Index read-only      |    403    | ✅ (after fix) | Disk > 95% full                | Free disk, reset read-only block           |
| Timeout              |    408    |       ✅       | Cluster overloaded             | Retry with backoff                         |
| Unavailable shards   |    503    |       ✅       | Node down, shard relocating    | Wait for recovery                          |

```ts
// Handle bulk indexing errors properly
const body = await client.bulk({ body: bulkPayload });

if (body.errors) {
  const failedItems = body.items.filter((item: any) => {
    const op = item.index || item.update || item.delete;
    return op?.error;
  });

  for (const item of failedItems) {
    const op = item.index || item.update || item.delete;
    if (op.status === 429) {
      // Rate limited — add to retry queue with backoff
      retryQueue.push(op);
    } else if (op.status === 409) {
      // Version conflict — may need re-read
      console.warn(`Version conflict for ${op._id}`);
    } else {
      // Non-retryable — log for investigation
      console.error(`Failed to index ${op._id}:`, op.error);
    }
  }
}
```

---

### 24. **How do you ensure Elasticsearch cluster resiliency?**

**Answer:**

```
Production cluster topology:

┌─────────────────────────────────┐
│          Load Balancer          │
└──────────┬──────────────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
  [Coord] [Coord] [Coord]    ← Coordinating nodes (stateless)
    │      │      │
    ├──────┼──────┤
    ▼      ▼      ▼
  [Master][Master][Master]    ← 3 dedicated masters (quorum)
    │      │      │
    ├──────┼──────┤
    ▼      ▼      ▼
  [Data-1][Data-2][Data-3]    ← Data nodes (across AZs)
  [Data-4][Data-5][Data-6]
```

| Resiliency Measure         | How                                    |
| -------------------------- | -------------------------------------- |
| No single point of failure | 3 master-eligible nodes (quorum = 2)   |
| Zone awareness             | Spread primaries + replicas across AZs |
| Snapshot & restore         | Regular snapshots to S3 / GCS          |
| Index aliases              | Zero-downtime reindexing               |
| Circuit breakers           | Prevent OOM crashes                    |
| Slow log                   | Detect problematic queries             |
| Watermark thresholds       | Stop allocation before disk full       |

```json
// Enable shard allocation awareness (cross-AZ)
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.awareness.attributes": "zone",
    "cluster.routing.allocation.awareness.force.zone.values": ["az-1", "az-2", "az-3"]
  }
}

// Snapshot policy
PUT _slm/policy/daily-snapshots
{
  "schedule": "0 30 2 * * ?",
  "name": "<daily-snap-{now/d}>",
  "repository": "s3-backups",
  "config": {
    "indices": ["*"],
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}
```

---

### 25. **How does Elasticsearch achieve high availability?**

**Answer:**

| Layer       | HA Mechanism                                               |
| ----------- | ---------------------------------------------------------- |
| **Cluster** | Master election quorum (3 masters), split-brain prevention |
| **Index**   | Replica shards on different nodes (automatic failover)     |
| **Network** | Cross-AZ shard allocation awareness                        |
| **Storage** | Translog (WAL equivalent) per shard                        |
| **API**     | Any node can serve as coordinator (no single entry point)  |

**SLA comparison:**

| Deployment                | Availability | Recovery                   |
| ------------------------- | :----------: | -------------------------- |
| Single node (dev)         |     ~99%     | Manual restart             |
| 3-node cluster            |    ~99.9%    | Auto shard reallocation    |
| Multi-AZ (3+ AZ)          |   ~99.95%    | Survive AZ failure         |
| Cross-region (CCR)        |   ~99.99%    | Read from surviving region |
| AWS OpenSearch Serverless |  99.99% SLA  | Fully managed              |

```json
// Cross-Cluster Replication for disaster recovery
PUT _ccr/auto_follow/dr-pattern
{
  "remote_cluster": "cluster-eu-west",
  "leader_index_patterns": ["products*", "orders*"],
  "follow_index_pattern": "{{leader_index}}-replica"
}
```

---

## Real-World Patterns

### 26. **When should you use Elasticsearch vs a database?**

**Answer:**

| Use Case                            |     Elasticsearch     |    PostgreSQL / MongoDB    |
| ----------------------------------- | :-------------------: | :------------------------: |
| Full-text search                    |    ✅ Best choice     | ⚠️ Basic (tsvector, $text) |
| Product search (e-commerce)         |          ✅           |    ❌ Too slow at scale    |
| Log aggregation                     |    ✅ (ELK stack)     |  ❌ Not designed for this  |
| Autocomplete                        |          ✅           |             ❌             |
| Analytics dashboards                |      ✅ (Kibana)      |   ⚠️ Possible but slower   |
| ACID transactions                   |          ❌           |             ✅             |
| Primary data store                  | ❌ (use as secondary) |             ✅             |
| Relationships / JOINs               |   ❌ (denormalize)    |             ✅             |
| Real-time writes + read consistency |  ❌ (near-real-time)  |             ✅             |

> **Golden rule:** Elasticsearch is a **secondary index**, not your primary database. Write to PostgreSQL/MongoDB first, then sync to Elasticsearch for search.

---

### 27. **How do you keep Elasticsearch in sync with your primary database?**

**Answer:**

| Method                                      | Latency       | Complexity |        Reliability        |
| ------------------------------------------- | ------------- | :--------: | :-----------------------: |
| Application dual-write                      | ~0ms          |    Low     | ⚠️ Can diverge on failure |
| Change Data Capture (Debezium → Kafka → ES) | Seconds       |   Medium   |          ✅ High          |
| Database triggers → queue → ES              | Seconds       |   Medium   |          ✅ High          |
| Periodic batch reindex                      | Minutes–hours |    Low     |      ✅ (eventually)      |

```ts
// CDC pipeline: PostgreSQL → Debezium → Kafka → Elasticsearch
// Debezium captures every INSERT/UPDATE/DELETE from PostgreSQL WAL
// Kafka Connect Elasticsearch Sink writes to ES

// Application-level dual-write (simpler but less reliable)
async function createProduct(product: Product) {
  // 1. Write to primary database (source of truth)
  const saved = await db.products.create(product);

  // 2. Index in Elasticsearch (best-effort, async)
  try {
    await esClient.index({
      index: "products",
      id: saved.id,
      body: toSearchDocument(saved),
    });
  } catch (error) {
    // Don't fail the request — queue for retry
    await retryQueue.add("es-index", { id: saved.id });
    console.error("ES index failed, queued for retry:", error.message);
  }

  return saved;
}
```

---

### 28. **How do you handle Elasticsearch in a microservices architecture?**

**Answer:**

```
Architecture: Search as a Service

┌──────────┐   ┌──────────┐   ┌──────────┐
│ Product  │   │  Order   │   │   User   │
│ Service  │   │ Service  │   │ Service  │
└────┬─────┘   └────┬─────┘   └────┬─────┘
     │              │              │
     ▼              ▼              ▼
  ┌─────────────────────────────────┐
  │         Kafka / Event Bus       │
  └───────────────┬─────────────────┘
                  │
                  ▼
  ┌─────────────────────────────────┐
  │        Search Service           │  ← Owns the ES cluster
  │  - Consumes events              │
  │  - Builds search indices        │
  │  - Exposes search API           │
  └───────────────┬─────────────────┘
                  │
                  ▼
  ┌─────────────────────────────────┐
  │       Elasticsearch Cluster     │
  └─────────────────────────────────┘
```

| Principle              | Implementation                                          |
| ---------------------- | ------------------------------------------------------- |
| Single ownership       | One service owns the ES cluster                         |
| Event-driven sync      | Services publish events, Search service consumes        |
| Denormalized documents | Combine data from multiple services into one search doc |
| Graceful degradation   | If ES is down, show cached results or fallback          |
| Read-only API          | Other services query Search service, never ES directly  |

---

### 29. **What is the ELK/Elastic Stack and how do the pieces fit together?**

**Answer:**

```
ELK Stack (now Elastic Stack):

  Beats (lightweight shippers)
    ├── Filebeat  → Log files
    ├── Metricbeat → System/app metrics
    ├── Packetbeat → Network data
    └── Heartbeat → Uptime monitoring
          │
          ▼
  Logstash (optional, heavy transforms)
    - Parse, filter, enrich
    - Grok patterns for unstructured logs
          │
          ▼
  Elasticsearch (store + search)
    - Index logs, metrics, traces
    - Full-text search + aggregations
          │
          ▼
  Kibana (visualize)
    - Dashboards, visualizations
    - Discover (explore logs)
    - Alerting
    - APM (Application Performance Monitoring)
```

| Component      | Alternatives                   |
| -------------- | ------------------------------ |
| Beats/Logstash | Fluentd, Fluent Bit, Vector    |
| Elasticsearch  | OpenSearch (AWS fork)          |
| Kibana         | Grafana, OpenSearch Dashboards |

---

### 30. **How does AWS OpenSearch compare to self-managed Elasticsearch?**

**Answer:** AWS OpenSearch Service is a fork of Elasticsearch 7.10 (before license change), fully managed by AWS.

| Feature           | Self-managed Elasticsearch | AWS OpenSearch Service                      |
| ----------------- | -------------------------- | ------------------------------------------- |
| Version           | Latest (8.x+)              | OpenSearch 2.x (based on ES 7.10 fork)      |
| License           | SSPL / Elastic License 2.0 | Apache 2.0                                  |
| Management        | You manage everything      | AWS manages infra, upgrades, backups        |
| Scaling           | Manual                     | Easy (API/console)                          |
| Multi-AZ          | Manual config              | ✅ Built-in (3-AZ recommended)              |
| Serverless option | ❌                         | ✅ OpenSearch Serverless                    |
| Security          | Self-managed (X-Pack)      | IAM, Cognito, fine-grained access built-in  |
| Cost              | EC2/hardware + ops time    | Per-instance-hour + storage + data transfer |
| Kibana            | Kibana                     | OpenSearch Dashboards                       |
| Plugins           | Full control               | Curated list only                           |

> **When to use OpenSearch Serverless:** Variable workloads, don't want to manage capacity. It auto-scales but costs more per query than provisioned.
