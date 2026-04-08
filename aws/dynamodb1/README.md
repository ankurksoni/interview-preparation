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
