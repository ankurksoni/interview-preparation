# DynamoDB Deep Dive — Indexing, Design, and Querying Patterns

This document captures real-world knowledge and implementation details around Amazon DynamoDB. It’s structured for engineers and architects looking to truly understand how DynamoDB behaves under the hood — especially around **partitioning**, **LSI**, **GSI**, **data modeling**, and **performance tuning**.

---

## 📘 Table of Contents

1. [DynamoDB vs RDBMS Concepts](#-dynamodb-vs-rdbms-concepts)
2. [Table Design: Books Table Example](#-table-design-books-table-example)
3. [LSI vs GSI: Internals and Behavior](#-lsi-vs-gsi-internals-and-behavior)
4. [Querying with AWS SDK](#-querying-with-aws-sdk)
5. [Behind the Scenes: How DynamoDB Stores Indexes](#-behind-the-scenes-how-dynamodb-stores-indexes)
6. [Limits: Item Size and Index Duplication](#-limits-item-size-and-index-duplication)
7. [Consistency and Query Behavior](#-consistency-and-query-behavior)
8. [10 Interview Questions](#-10-advanced-dynamodb-interview-questions)
9. [FAQ](#-faq)

---

## 🔄 DynamoDB vs RDBMS Concepts

| RDBMS Concept | DynamoDB Equivalent                                  |
| ------------- | ---------------------------------------------------- |
| Database      | Region + AWS Account                                 |
| Table         | Table                                                |
| Row           | Item                                                 |
| Column        | Attribute                                            |
| Primary Key   | Partition Key (and optional Sort Key)                |
| Index         | LSI / GSI                                            |
| Foreign Key   | ❌ Not Supported (Use Denormalization)               |
| Transactions  | ✅ Supported (ACID, up to 100 items per transaction) |

---

## 🧱 Table Design: Books Table Example

A DynamoDB table `books` with:

- **Partition Key**: `bookId`
- **Sort Key**: `edition`

Includes:

- **LSI** on `publishedYear`
- **GSI** on `author + genre`

```ts
booksTable.addLocalSecondaryIndex({
  indexName: "PublishedYearIndex",
  sortKey: { name: "publishedYear", type: dynamodb.AttributeType.NUMBER },
  projectionType: dynamodb.ProjectionType.ALL,
});

booksTable.addGlobalSecondaryIndex({
  indexName: "AuthorGenreIndex",
  partitionKey: { name: "author", type: dynamodb.AttributeType.STRING },
  sortKey: { name: "genre", type: dynamodb.AttributeType.STRING },
  projectionType: dynamodb.ProjectionType.ALL,
});
```

---

## ⚙️ LSI vs GSI: Internals and Behavior

### 📘 Main Table (Partitioned by bookId, sorted by edition)

```
Partition: "B001"
edition: 1 → publishedYear: 2008
edition: 2 → publishedYear: 2021
```

---

### 📂 Local Secondary Index (LSI: PublishedYearIndex)

```
Partition: "B001"
Sorted by: publishedYear
  → edition: 1, year: 2008
  → edition: 2, year: 2021
```

✅ Shares the same partition key as main table.

---

### 🌐 Global Secondary Index (GSI: AuthorGenreIndex)

```
Partition: "Robert C. Martin"
Sorted by: genre
  → bookId: B001, edition: 1
  → bookId: B001, edition: 2
```

✅ Stored independently, supports new access pattern.

---

## ⚡ Querying with AWS SDK

```ts
await docClient.send(
  new QueryCommand({
    TableName: "books",
    KeyConditionExpression: "bookId = :id AND edition = :ed",
    ExpressionAttributeValues: {
      ":id": "B001",
      ":ed": 2,
    },
  }),
);

await docClient.send(
  new QueryCommand({
    TableName: "books",
    IndexName: "PublishedYearIndex",
    KeyConditionExpression: "bookId = :id AND publishedYear = :year",
    ExpressionAttributeValues: {
      ":id": "B001",
      ":year": 2021,
    },
  }),
);

await docClient.send(
  new QueryCommand({
    TableName: "books",
    IndexName: "AuthorGenreIndex",
    KeyConditionExpression: "author = :author AND genre = :genre",
    ExpressionAttributeValues: {
      ":author": "Robert C. Martin",
      ":genre": "Software",
    },
  }),
);
```

---

## 📦 Limits: Item Size and Index Duplication

- ✅ **Max item size**: 400 KB
- ✅ Applies to each **GSI/LSI projection** independently
- ✅ GSI/LSI store **copies of projected data**
- ⚠️ Writes fail if projected index entry exceeds 400 KB

---

## 🔁 Consistency and Query Behavior

| Feature         | Base Table      | LSI                             | GSI                      |
| --------------- | --------------- | ------------------------------- | ------------------------ |
| Consistent Read | ✅ Yes          | ✅ Yes                          | ❌ No (eventual)         |
| Update Delay    | ❌ Minimal      | ❌ Minimal                      | ✅ May lag               |
| Use Case        | Primary lookups | Alternate sort within partition | Alternate access pattern |

---

## 🎯 10 Advanced DynamoDB Interview Questions

- 🏢 How would you model a multi-tenant application in DynamoDB while maintaining tenant isolation and performance?
  → Use composite keys like `tenantId#entityId`.

- 📏 If your item size is close to 400 KB and projected into 3 GSIs, what are the risks and how would you mitigate them?
  → Writes can fail — reduce projection size or use S3 for overflow.

- 🕒 Why might your GSI not return recently inserted data even though the main table does?
  → GSIs are eventually consistent.

- 📊 What’s the impact of a high-cardinality vs low-cardinality partition key on read/write throughput distribution?
  → Low-cardinality can cause hot partitions.

- 🎯 How do you handle querying a list of books filtered by rating range if rating is not part of any index?
  → Pre-index or scan + filter (inefficient).

- ⏳ Why is storing timestamps as sort keys a potential anti-pattern in high-throughput systems?
  → Leads to hot partitions due to write clustering.

- 🚀 How do you avoid hot partitions when millions of users write to the same partition key in real-time?
  → Use write-sharding by suffixing partition keys.

- 🔍 How would you redesign your access patterns if your application needs ad-hoc filtering and searching across multiple fields?
  → Use a search engine like OpenSearch or Typesense alongside DynamoDB.

- ⚖️ What are the trade-offs between using a GSI with `ProjectionType.ALL` vs `ProjectionType.KEYS_ONLY`?
  → `ALL` uses more storage but gives rich queries; `KEYS_ONLY` is lean.

- 🔒 You need strong consistency guarantees but also want to query by a non-key attribute. How do you design for it?
  → Store precomputed query views in a strongly consistent table.

---

## ❓ FAQ

- **Is data duplicated in GSIs?**
  ✅ Yes, projected attributes are copied into each index.

- **Is the 400 KB limit shared across table and indexes?**
  ❌ No, each item in the table and each index entry has its own 400 KB limit.

- **Are GSIs updated immediately?**
  ❌ No, they’re eventually consistent and may lag.

- **Can you force strong consistency on GSI queries?**
  ❌ Not supported — only base table and LSI support consistent reads.

- **When should I use GSI vs LSI?**
  - LSI = same partition, alternate sort
  - GSI = completely new access path

---

## 📌 Summary

DynamoDB is powerful — but only if you design with its **partitioning, consistency model, and indexing rules** in mind. Understand your access patterns, leverage indexes carefully, and monitor size constraints to build scalable apps.

---

## Resiliency, Availability, Error Handling & Distributed Patterns

### 11. **How does DynamoDB achieve high availability and fault tolerance?**

**Answer:** DynamoDB automatically replicates data across **3 Availability Zones** within a region. There's no single point of failure.

| Layer                | Redundancy                     |   You manage?    |
| -------------------- | ------------------------------ | :--------------: |
| Storage              | 3 AZ replication (synchronous) |   ❌ Automatic   |
| Request routing      | Multiple internal routers      |   ❌ Automatic   |
| Partition management | Auto-split, auto-migrate       |   ❌ Automatic   |
| Global Tables        | Active-active across regions   | ✅ You enable it |

```
Region: us-east-1
┌─────────┐    ┌─────────┐    ┌─────────┐
│  AZ-1a  │    │  AZ-1b  │    │  AZ-1c  │
│ Replica │ ←→ │ Replica │ ←→ │ Replica │
└─────────┘    └─────────┘    └─────────┘
     Write acknowledged after 2 of 3 replicas confirm.
     Read (strong consistency) routes to leader replica.
     Read (eventual consistency) can use any replica.
```

> **SLA:** 99.999% for Global Tables, 99.99% for single-region tables.

---

### 12. **How does DynamoDB handle throttling and what errors should you expect?**

**Answer:** DynamoDB returns `ProvisionedThroughputExceededException` when you exceed your table's capacity, or `ThrottlingException` for API-level throttling.

```ts
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const client = new DynamoDBClient({
  maxAttempts: 5, // SDK retries with exponential backoff built-in
});

try {
  await client.send(
    new PutItemCommand({
      TableName: "Orders",
      Item: { orderId: { S: "o-123" }, status: { S: "pending" } },
    }),
  );
} catch (error) {
  if (error.name === "ProvisionedThroughputExceededException") {
    // SDK already retried 5 times — this is a sustained capacity issue
    console.error("Throughput exceeded. Consider:");
    console.error("1. Switch to on-demand mode");
    console.error("2. Increase provisioned WCU");
    console.error("3. Check for hot partitions");
  }
  if (error.name === "ConditionalCheckFailedException") {
    // Expected in optimistic locking patterns — not an actual error
    console.log("Item was modified by another request");
  }
  if (error.name === "TransactionCanceledException") {
    console.error("Transaction failed. Reasons:", error.CancellationReasons);
  }
}
```

| Error                                    |  Retryable?   | Common Cause                          | Fix                                |
| ---------------------------------------- | :-----------: | ------------------------------------- | ---------------------------------- |
| ProvisionedThroughputExceededException   | ✅ (SDK auto) | Burst exceeds capacity                | On-demand mode or increase RCU/WCU |
| ThrottlingException                      | ✅ (SDK auto) | Too many control plane API calls      | Back off, reduce API calls         |
| ConditionalCheckFailedException          |      ❌       | Condition expression failed           | Expected — handle in logic         |
| TransactionCanceledException             |    Depends    | Conflict, condition failure, capacity | Check CancellationReasons array    |
| ValidationException                      |      ❌       | Invalid request (bad key, wrong type) | Fix request                        |
| ItemCollectionSizeLimitExceededException |      ❌       | LSI item collection exceeded 10 GB    | Redesign partition key             |

---

### 13. **How does DynamoDB partition data and what is a hot partition?**

**Answer:** DynamoDB hashes the partition key to distribute items across internal partitions. Each partition supports up to **3,000 RCU** and **1,000 WCU**.

```
Table: UserSessions (partition key = userId)

Good distribution:              Bad distribution (hot partition):
┌──────────┐                    ┌──────────┐
│ Part 1   │ userId: A-D  25%  │ Part 1   │ userId: "system"  90% ← HOT!
│ Part 2   │ userId: E-K  25%  │ Part 2   │ userId: others     5%
│ Part 3   │ userId: L-R  25%  │ Part 3   │ userId: others     5%
│ Part 4   │ userId: S-Z  25%  │ Part 4   │ (empty)            0%
└──────────┘                    └──────────┘
```

**Strategies to avoid hot partitions:**

| Strategy                       | Example                          | When to use              |
| ------------------------------ | -------------------------------- | ------------------------ |
| High-cardinality partition key | `userId`, `deviceId`             | Always                   |
| Write sharding (suffix)        | `date#0`, `date#1`, `date#2`     | Time-series writes       |
| Composite key                  | `userId#orderDate`               | Multi-access patterns    |
| Scatter-gather                 | Write to N shards, read from all | Extreme write throughput |

```ts
// Write sharding for a high-traffic counter
const SHARD_COUNT = 10;

async function incrementCounter(counterId: string) {
  const shard = Math.floor(Math.random() * SHARD_COUNT);
  await client.send(
    new UpdateItemCommand({
      TableName: "Counters",
      Key: { pk: { S: `${counterId}#${shard}` } },
      UpdateExpression: "ADD #count :inc",
      ExpressionAttributeNames: { "#count": "count" },
      ExpressionAttributeValues: { ":inc": { N: "1" } },
    }),
  );
}

// Read: aggregate across all shards
async function getCounter(counterId: string) {
  const queries = Array.from({ length: SHARD_COUNT }, (_, i) =>
    client.send(
      new GetItemCommand({
        TableName: "Counters",
        Key: { pk: { S: `${counterId}#${i}` } },
      }),
    ),
  );
  const results = await Promise.all(queries);
  return results.reduce((sum, r) => sum + parseInt(r.Item?.count?.N || "0"), 0);
}
```

---

### 14. **How do Global Tables provide distributed, multi-region availability?**

**Answer:** Global Tables are **active-active**: every region can accept reads AND writes. Replication is automatic and typically under 1 second.

```ts
// Write in us-east-1
await clientUSEast.send(
  new PutItemCommand({
    TableName: "Users",
    Item: { userId: { S: "u123" }, name: { S: "Alice" } },
  }),
);

// Read in eu-west-1 — available within ~1 second
const user = await clientEUWest.send(
  new GetItemCommand({
    TableName: "Users",
    Key: { userId: { S: "u123" } },
  }),
);
```

| Feature             | Single-region table     | Global Table                                  |
| ------------------- | ----------------------- | --------------------------------------------- |
| Availability SLA    | 99.99%                  | 99.999%                                       |
| Write location      | One region              | Any region (active-active)                    |
| Conflict resolution | N/A                     | Last writer wins (timestamp)                  |
| Read consistency    | Strong or eventual      | Eventual (cross-region), Strong (same region) |
| Cost                | Base price              | Base + replication WCU                        |
| Disaster recovery   | Manual (backup/restore) | Automatic (another region is already live)    |

> **Conflict resolution:** If two regions write to the same item within the replication window, **last-writer-wins** (based on timestamp). Design for idempotency to handle this.

---

### 15. **What performance factors matter most in DynamoDB?**

**Answer:**

| Factor               | Impact                               | Optimization                                           |
| -------------------- | ------------------------------------ | ------------------------------------------------------ |
| Partition key design | Determines distribution, throughput  | High cardinality, avoid hot keys                       |
| Item size            | Larger items = more RCU/WCU consumed | Project only needed attributes                         |
| Read consistency     | Strong reads cost 2× eventual reads  | Use eventual reads when possible                       |
| Query vs Scan        | Scan reads entire table              | Always Query with partition key                        |
| GSI projections      | ALL = double storage cost            | Project only attributes you need                       |
| Transaction overhead | 2× capacity for transactional ops    | Batch normal writes when possible                      |
| DAX (caching)        | Microsecond reads, reduces RCU       | Enable for read-heavy, eventually consistent workloads |

```ts
// ❌ Expensive: Scan entire table with filter (reads everything, discards most)
const expensive = await client.send(
  new ScanCommand({
    TableName: "Orders",
    FilterExpression: "userId = :uid",
    ExpressionAttributeValues: { ":uid": { S: "u123" } },
  }),
);

// ✅ Efficient: Query with partition key (reads only matching items)
const efficient = await client.send(
  new QueryCommand({
    TableName: "Orders",
    KeyConditionExpression: "userId = :uid",
    ExpressionAttributeValues: { ":uid": { S: "u123" } },
    ProjectionExpression: "orderId, #s, total", // only fetch needed fields
    ExpressionAttributeNames: { "#s": "status" },
  }),
);
```
