# Amazon OpenSearch Service Interview Questions & Answers

A curated list of **Amazon OpenSearch Service interview questions** with practical, production-grade answers. Covers cluster design, indexing, search patterns, performance tuning, CDK/TypeScript examples, and real-world system design decisions.

---

### 1. **What is Amazon OpenSearch Service?**

**Answer:** Amazon OpenSearch Service (successor to Amazon Elasticsearch Service) is a **managed search and analytics engine** based on OpenSearch (Apache 2.0 fork of Elasticsearch). It handles cluster provisioning, patching, backups, and monitoring.

**Key characteristics:**

- **Engine:** OpenSearch (or legacy Elasticsearch 7.x)
- **Use cases:** Full-text search, log analytics, SIEM, observability
- **Includes:** OpenSearch Dashboards (Kibana fork) for visualization
- **Deployment:** Multi-AZ with dedicated master nodes
- **Serverless option:** OpenSearch Serverless for zero-management workloads

---

### 2. **When should you use OpenSearch vs when NOT to?**

**Answer:**

| Use OpenSearch When | Don't Use OpenSearch When |
| --- | --- |
| Full-text search with relevance ranking | Simple key-value lookups (use DynamoDB) |
| Log analytics and observability | Relational queries with joins (use RDS/Aurora) |
| Fuzzy/typo-tolerant search | Primary database / source of truth |
| Faceted search and aggregations | Simple filtering on structured data (use DynamoDB + GSI) |
| Real-time dashboards on time-series data | Long-term archival (use S3 + Athena) |
| Geospatial search | Write-heavy OLTP (use DynamoDB/RDS) |
| SIEM / security analytics | Small dataset under 100K records (overkill) |

> **System Design Tip:** OpenSearch is a **secondary index**, not a primary database. Always have a source of truth (DynamoDB, RDS, S3) and replicate to OpenSearch for search functionality.

**Architecture pattern:**

```
Primary DB (DynamoDB/RDS) → DynamoDB Streams / CDC → Lambda → OpenSearch
                                                              ↓
                                                      API → Search queries
```

---

### 3. **How do you connect to OpenSearch from TypeScript?**

**Answer:**

```ts
import { Client } from "@opensearch-project/opensearch";
import { AwsSigv4Signer } from "@opensearch-project/opensearch/aws";
import { defaultProvider } from "@aws-sdk/credential-provider-node";

// ✅ Production: Use IAM auth with SigV4
const client = new Client({
  ...AwsSigv4Signer({
    region: "us-east-1",
    service: "es",
    getCredentials: () => defaultProvider()(),
  }),
  node: "https://my-domain.us-east-1.es.amazonaws.com",
});

// Index a document
await client.index({
  index: "products",
  id: "prod-123",
  body: {
    name: "Wireless Mouse",
    description: "Ergonomic wireless mouse with USB-C",
    price: 29.99,
    category: "electronics",
    tags: ["wireless", "mouse", "ergonomic"],
    created_at: new Date().toISOString(),
  },
});

// Search with full-text + filters
const result = await client.search({
  index: "products",
  body: {
    query: {
      bool: {
        must: [
          { multi_match: { query: "wireless mouse", fields: ["name^3", "description"] } },
        ],
        filter: [
          { range: { price: { lte: 50 } } },
          { term: { category: "electronics" } },
        ],
      },
    },
    highlight: { fields: { name: {}, description: {} } },
    size: 20,
  },
});
```

---

### 4. **What are the important cluster sizing and config parameters?**

**Answer:**

| Parameter | Description | Production Recommendation |
| --- | --- | --- |
| **Instance type** | Compute for data nodes | `r6g.xlarge` for search; `r6g.2xlarge` for log analytics |
| **Data nodes** | Store and serve data | Minimum 2 (multi-AZ); 3+ for production |
| **Dedicated master nodes** | Cluster coordination | Always 3 for production (odd number) |
| **EBS volume** | Storage per node | gp3; size = data × (1 + replicas) / nodes × 1.45 |
| **Replica count** | Copies of each shard | 1 in multi-AZ (2 for critical data) |
| **Shard count** | Partitions per index | Target 10–50 GB per shard |
| **Zone awareness** | Multi-AZ deployment | Always enable for production |
| **UltraWarm** | Warm storage tier | Use for logs >30 days (90% cheaper) |

```ts
// CDK: Production OpenSearch cluster
import * as opensearch from "aws-cdk-lib/aws-opensearchservice";
import * as ec2 from "aws-cdk-lib/aws-ec2";

const domain = new opensearch.Domain(this, "SearchDomain", {
  version: opensearch.EngineVersion.OPENSEARCH_2_11,
  domainName: "product-search",

  // Data nodes
  capacity: {
    dataNodes: 3,
    dataNodeInstanceType: "r6g.xlarge.search",
    multiAzWithStandbyEnabled: false, // true = 3-AZ with standby (higher cost)
    masterNodes: 3,
    masterNodeInstanceType: "m6g.large.search",
  },

  // Storage
  ebs: {
    volumeSize: 100,    // GB per node
    volumeType: ec2.EbsDeviceVolumeType.GP3,
    throughput: 250,     // MB/s for gp3
    iops: 3000,
  },

  // Networking
  vpc,
  vpcSubnets: [{ subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS }],
  zoneAwareness: { enabled: true, availabilityZoneCount: 3 },

  // Security
  enforceHttps: true,
  nodeToNodeEncryption: true,
  encryptionAtRest: { enabled: true },
  fineGrainedAccessControl: {
    masterUserArn: masterUserRole.roleArn,
  },

  // Performance
  logging: {
    slowSearchLogEnabled: true,
    slowIndexLogEnabled: true,
    appLogEnabled: true,
  },
});
```

---

### 5. **How do you design index mappings for performance?**

**Answer:** Mapping defines field types and how data is indexed. Poor mappings = poor performance.

```ts
// Create index with explicit mapping
await client.indices.create({
  index: "products",
  body: {
    settings: {
      number_of_shards: 3,
      number_of_replicas: 1,
      "index.refresh_interval": "5s",     // Balance between freshness and perf
      "index.translog.durability": "async", // Faster indexing (slight data risk)
      "index.translog.sync_interval": "5s",
    },
    mappings: {
      properties: {
        name: {
          type: "text",
          analyzer: "standard",
          fields: { keyword: { type: "keyword" } }, // For exact match + aggregations
        },
        description: { type: "text" },
        price: { type: "float" },
        category: { type: "keyword" },   // keyword = exact match, no analysis
        tags: { type: "keyword" },
        location: { type: "geo_point" },  // Geospatial search
        created_at: { type: "date" },
        metadata: { type: "object", enabled: false }, // Store but don't index
      },
    },
  },
});
```

**Key mapping decisions:**

| Field Type | Use | Don't Use |
| --- | --- | --- |
| `text` | Full-text search (analyzed) | Exact matching, aggregations |
| `keyword` | Filtering, sorting, aggregations | Full-text search |
| `text` + `keyword` subfield | Searching AND filtering same field | Simple use cases |
| `enabled: false` | Store without indexing | If you ever need to search/filter |
| `date` | Time-series, range queries | String dates (no range queries) |
| `nested` | Array of objects with independent matching | Simple arrays (use `keyword`) |

> **Hidden Property:** Use `"index": false` on fields you never search/filter. Saves CPU and disk. Use `"doc_values": false` on text fields you never sort/aggregate.

---

### 6. **What are OpenSearch's performance tuning strategies?**

**Answer:**

**Indexing performance:**

```ts
// ✅ Bulk indexing: Always use _bulk API (not single index calls)
const body = products.flatMap((product) => [
  { index: { _index: "products", _id: product.id } },
  product,
]);

await client.bulk({ body, refresh: false }); // Don't refresh after every bulk

// ❌ Bad: Index one at a time
for (const product of products) {
  await client.index({ index: "products", body: product }); // N API calls!
}
```

**Search performance:**

| Strategy | Impact | How |
| --- | --- | --- |
| Use `filter` context | 30–50% faster (cached, no scoring) | Move non-scoring clauses to `filter` |
| Limit `_source` fields | Reduce network I/O | `_source: ["name", "price"]` |
| Use `keyword` for exact match | 10x faster than `text` for filtering | Map filterable fields as `keyword` |
| Avoid deep pagination | Memory explosion | Use `search_after` instead of `from/size` |
| Warm up frequently used indices | Faster first queries | `POST /index/_forcemerge?max_num_segments=1` |
| Right-size shards | 10–50 GB per shard | Too many small shards = overhead |

```ts
// ✅ Use search_after for deep pagination (not from/size)
let searchAfter: any[] | undefined;
const allResults = [];

do {
  const response = await client.search({
    index: "products",
    body: {
      query: { match: { category: "electronics" } },
      sort: [{ created_at: "desc" }, { _id: "asc" }], // Tiebreaker required
      size: 100,
      ...(searchAfter && { search_after: searchAfter }),
    },
  });

  const hits = response.body.hits.hits;
  allResults.push(...hits);
  searchAfter = hits.length ? hits[hits.length - 1].sort : undefined;
} while (searchAfter);
```

---

### 7. **How do you handle resiliency in OpenSearch?**

**Answer:**

| Concern | Solution |
| --- | --- |
| Node failure | Multi-AZ deployment with replicas |
| Index corruption | Automated daily snapshots (14-day retention) |
| Data loss | Replicas + manual snapshots to S3 |
| Split brain | 3 dedicated master nodes (quorum-based) |
| Traffic spike | Auto-Tune (adjusts JVM, queue sizes) |
| Index mapping conflicts | Use index templates with strict mappings |
| Disk full | Set watermark alerts at 75% disk usage |
| Blue/green deploy failure | Automated by AWS during config changes |

```ts
// CDK: Snapshot configuration + alarms
domain.addAccessPolicies(
  new iam.PolicyStatement({
    actions: ["es:ESHttp*"],
    principals: [new iam.ArnPrincipal(lambdaRole.roleArn)],
    resources: [`${domain.domainArn}/*`],
  }),
);

// Disk usage alarm
new cloudwatch.Alarm(this, "DiskAlarm", {
  metric: domain.metricFreeStorageSpace(),
  threshold: 20_000, // 20 GB free remaining
  evaluationPeriods: 1,
  comparisonOperator: cloudwatch.ComparisonOperator.LESS_THAN_OR_EQUAL_TO_THRESHOLD,
  alarmDescription: "OpenSearch cluster running low on disk",
});

// Cluster health alarm
new cloudwatch.Alarm(this, "ClusterRedAlarm", {
  metric: domain.metricClusterStatusRed(),
  threshold: 1,
  evaluationPeriods: 1,
  alarmDescription: "OpenSearch cluster RED — data loss possible",
});
```

---

### 8. **What is the Index Lifecycle pattern for time-series data?**

**Answer:** Use **Index State Management (ISM)** to automatically rotate, warm, and delete indices.

```ts
// Create ISM policy via API
await client.transport.request({
  method: "PUT",
  path: "/_plugins/_ism/policies/log-rotation",
  body: {
    policy: {
      description: "Rotate logs: hot → warm → delete",
      default_state: "hot",
      states: [
        {
          name: "hot",
          actions: [{ rollover: { min_index_age: "1d", min_size: "50gb" } }],
          transitions: [{ state_name: "warm", conditions: { min_index_age: "7d" } }],
        },
        {
          name: "warm",
          actions: [
            { replica_count: { number_of_replicas: 0 } },
            { force_merge: { max_num_segments: 1 } },    // Optimize storage
          ],
          transitions: [{ state_name: "delete", conditions: { min_index_age: "90d" } }],
        },
        {
          name: "delete",
          actions: [{ delete: {} }],
        },
      ],
      ism_template: [{ index_patterns: ["logs-*"], priority: 100 }],
    },
  },
});

// Index template for new indices
await client.indices.putIndexTemplate({
  name: "logs-template",
  body: {
    index_patterns: ["logs-*"],
    template: {
      settings: {
        number_of_shards: 3,
        number_of_replicas: 1,
        "plugins.index_state_management.rollover_alias": "logs",
      },
    },
  },
});
```

> **Production Tip:** Always use ISM for time-series data. Without it, indices grow indefinitely, cluster performance degrades, and costs spiral.

---

### 9. **How does OpenSearch Serverless work?**

**Answer:** OpenSearch Serverless removes cluster management entirely — no nodes, shards, or capacity planning.

| Feature | Managed OpenSearch | OpenSearch Serverless |
| --- | --- | --- |
| **Management** | You size/manage cluster | Fully managed |
| **Scaling** | Manual or Auto-Tune | Automatic |
| **Pricing** | Per instance-hour + storage | OCU-hours ($0.24/OCU-hr) + storage |
| **Min cost** | ~$50/month (t3.small) | ~$350/month (min 4 OCUs) |
| **Use case** | Predictable workloads | Variable/unpredictable workloads |
| **APIs** | Full OpenSearch API | Subset (no ISM, limited admin) |
| **Collection types** | N/A | Search, Time-series, Vector search |

```ts
// CDK: OpenSearch Serverless collection
import * as opensearchserverless from "aws-cdk-lib/aws-opensearchserverless";

const collection = new opensearchserverless.CfnCollection(this, "SearchCollection", {
  name: "product-search",
  type: "SEARCH",
});

// Security policy (required)
new opensearchserverless.CfnSecurityPolicy(this, "EncPolicy", {
  name: "product-search-enc",
  type: "encryption",
  policy: JSON.stringify({
    Rules: [{ ResourceType: "collection", Resource: ["collection/product-search"] }],
    AWSOwnedKey: true,
  }),
});

new opensearchserverless.CfnSecurityPolicy(this, "NetPolicy", {
  name: "product-search-net",
  type: "network",
  policy: JSON.stringify([{
    Rules: [{ ResourceType: "collection", Resource: ["collection/product-search"] }],
    AllowFromPublic: true,
  }]),
});
```

> **Hidden Gem:** OpenSearch Serverless supports **vector search** natively — great for semantic search, RAG (Retrieval-Augmented Generation), and ML embedding workflows without managing any infrastructure.

---

### 10. **What are OpenSearch cost optimization strategies?**

**Answer:**

| Strategy | Savings | Effort |
| --- | --- | --- |
| Use UltraWarm for logs >30 days | ~90% vs hot storage | Low (ISM policy) |
| Cold storage for >90 days | ~95% vs hot storage | Low (ISM policy) |
| Right-size instances | 20–50% | Medium (load testing) |
| Reduce replica count on non-critical indices | 50% storage | Low |
| Use `_source` filtering on large docs | Reduces I/O costs | Low |
| gp3 volumes (not gp2) | 20% cheaper + better IOPS | Low |
| Reserved instances (1yr/3yr) | 30–50% | Commit required |
| Disable unused features (slow logs, etc.) | Minor savings | Low |

**Cost comparison (3-node production cluster, us-east-1):**

| Tier | Instance | Storage | Monthly Cost |
| --- | --- | --- | --- |
| Dev/Test | `t3.medium.search` × 2 | 50 GB gp3 | ~$120 |
| Production | `r6g.xlarge.search` × 3 + 3 masters | 200 GB gp3 | ~$900 |
| Large-scale | `r6g.2xlarge.search` × 6 + 3 masters | 500 GB gp3 | ~$2,500 |

---

### 11. **How do you implement search-as-you-type (autocomplete)?**

**Answer:**

```ts
// Create index with autocomplete mapping
await client.indices.create({
  index: "products-autocomplete",
  body: {
    settings: {
      analysis: {
        analyzer: {
          autocomplete_analyzer: {
            type: "custom",
            tokenizer: "autocomplete_tokenizer",
            filter: ["lowercase"],
          },
        },
        tokenizer: {
          autocomplete_tokenizer: {
            type: "edge_ngram",
            min_gram: 2,
            max_gram: 15,
            token_chars: ["letter", "digit"],
          },
        },
      },
    },
    mappings: {
      properties: {
        name: {
          type: "text",
          analyzer: "autocomplete_analyzer",
          search_analyzer: "standard", // Don't ngram the search query
        },
        suggest: {
          type: "completion", // Completion suggester (fastest)
          contexts: [{ name: "category", type: "category" }],
        },
      },
    },
  },
});

// Search-as-you-type query
const result = await client.search({
  index: "products-autocomplete",
  body: {
    suggest: {
      product_suggest: {
        prefix: "wire",
        completion: {
          field: "suggest",
          size: 5,
          contexts: { category: [{ context: "electronics" }] },
          fuzzy: { fuzziness: "AUTO" }, // Handle typos
        },
      },
    },
  },
});
```

---

### 12. **What are hidden/lesser-known OpenSearch features useful in production?**

**Answer:**

| Feature | What It Does | Why It Matters |
| --- | --- | --- |
| **Auto-Tune** | Automatically adjusts JVM heap, queue sizes | Prevents OOM and performance degradation |
| **Slow log queries** | Log queries exceeding threshold | Identify and fix performance bottlenecks |
| **Index templates** | Auto-apply settings to matching indices | Consistent mappings across time-series indices |
| **Alias routing** | Route writes to one index, reads to many | Zero-downtime reindexing |
| **Field caps API** | Discover field types across indices | Useful for dynamic dashboards |
| **Profile API** | Detailed query execution breakdown | Debug why a specific query is slow |
| **Anomaly detection** | ML-based anomaly detection on metrics | Alerting without manual thresholds |
| **Cross-cluster search** | Query across multiple domains | Multi-region search architecture |
| **Painless scripting** | In-query transformations | Custom scoring, field computation |

```ts
// Alias-based zero-downtime reindexing
// 1. Create new index with updated mappings
await client.indices.create({ index: "products-v2", body: { /* new mapping */ } });

// 2. Reindex data
await client.reindex({
  body: {
    source: { index: "products-v1" },
    dest: { index: "products-v2" },
  },
});

// 3. Swap alias atomically — zero downtime
await client.indices.updateAliases({
  body: {
    actions: [
      { remove: { index: "products-v1", alias: "products" } },
      { add: { index: "products-v2", alias: "products" } },
    ],
  },
});
```

---

### 13. **How do you integrate OpenSearch with DynamoDB for real-time search?**

**Answer:**

```ts
// CDK: DynamoDB → Streams → Lambda → OpenSearch
const table = new dynamodb.Table(this, "Products", {
  partitionKey: { name: "pk", type: dynamodb.AttributeType.STRING },
  stream: dynamodb.StreamViewType.NEW_AND_OLD_IMAGES,
});

const indexer = new lambda.Function(this, "Indexer", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "indexer.handler",
  code: lambda.Code.fromAsset("lambda"),
  environment: {
    OPENSEARCH_ENDPOINT: domain.domainEndpoint,
  },
});

indexer.addEventSource(
  new eventsources.DynamoEventSource(table, {
    startingPosition: lambda.StartingPosition.TRIM_HORIZON,
    batchSize: 100,
    maxBatchingWindow: cdk.Duration.seconds(5),
    bisectBatchOnError: true,
    retryAttempts: 3,
    onFailure: new eventsources.SqsDestination(dlq),
    reportBatchItemFailures: true,
  }),
);

domain.grantWrite(indexer);
```

```ts
// Lambda: DynamoDB Stream → OpenSearch indexer
import { DynamoDBStreamEvent } from "aws-lambda";
import { unmarshall } from "@aws-sdk/util-dynamodb";

export const handler = async (event: DynamoDBStreamEvent) => {
  const bulkBody: any[] = [];

  for (const record of event.Records) {
    const id = record.dynamodb?.Keys?.pk?.S;

    if (record.eventName === "REMOVE") {
      bulkBody.push({ delete: { _index: "products", _id: id } });
    } else {
      const item = unmarshall(record.dynamodb?.NewImage as any);
      bulkBody.push(
        { index: { _index: "products", _id: id } },
        { name: item.name, price: item.price, category: item.category },
      );
    }
  }

  if (bulkBody.length > 0) {
    const result = await client.bulk({ body: bulkBody });
    if (result.body.errors) {
      const failed = result.body.items.filter((i: any) =>
        Object.values(i)[0].error,
      );
      console.error("Bulk indexing failures:", JSON.stringify(failed));
    }
  }
};
```

---

### 14. **What are key production monitoring metrics?**

**Answer:**

| Metric | Healthy Value | Alert Threshold |
| --- | --- | --- |
| `ClusterStatus.red` | 0 | ≥ 1 (immediate) |
| `ClusterStatus.yellow` | 0 | ≥ 1 (investigate) |
| `FreeStorageSpace` | >20% of total | < 25% remaining |
| `JVMMemoryPressure` | < 80% | > 80% (GC issues) |
| `CPUUtilization` | < 70% | > 80% sustained |
| `SearchLatency` | < 100ms (p99) | > 500ms |
| `IndexingLatency` | < 50ms (p99) | > 200ms |
| `ThreadpoolSearchRejected` | 0 | > 0 (queue full) |
| `AutomatedSnapshotFailure` | 0 | > 0 |

> **Hidden Gem:** `ThreadpoolSearchRejected` and `ThreadpoolWriteRejected` are early indicators of an overloaded cluster. If these spike, scale up before users notice latency.
