# MongoDB Interview Questions & Answers (Basic to Advanced)

A comprehensive set of MongoDB interview questions covering schema design, indexing, aggregation, sharding, replication, security, and modern features like vector search and time-series collections.

### 1. What is MongoDB and when would you use it?

**Answer:** MongoDB is a NoSQL, document-oriented database that stores data in flexible, JSON-like documents. Use it when dealing with semi-structured data, rapid prototyping, schema evolution, or when horizontal scalability is a priority.

### 2. How is data stored in MongoDB?

**Answer:** Data is stored in BSON (Binary JSON) format inside collections. Each document can have a dynamic schema, allowing flexibility in data structure.

### 3. What is the difference between `db.collection.find()` and `db.collection.aggregate()`?

**Answer:** `find()` is used for simple queries and projections, while `aggregate()` supports a powerful pipeline to process and transform data (e.g., grouping, joining, filtering, reshaping).

### 4. How do you implement pagination in MongoDB?

**Answer:** Using `.skip()` and `.limit()`, or with a range-based query using the `_id` or a timestamp field to avoid performance issues with large skips.

### 5. What are MongoDB indexes? How do they impact performance?

**Answer:** Indexes improve query speed by avoiding full collection scans. Types include single-field, compound, multikey, text, hashed, and geospatial. Poorly designed indexes can slow down writes and consume memory.

### 6. What is a compound index and when should you use it?

**Answer:** An index on multiple fields. Use it when queries filter or sort by multiple fields. The order of fields in the index matters for optimization.

### 7. What is the difference between `covered` and `non-covered` queries?

**Answer:** Covered queries are resolved entirely using the index without scanning any documents. Improves performance significantly.

### 8. How would you analyze and improve a slow MongoDB query?

**Answer:** Use `.explain("executionStats")` to inspect index usage, scanned docs vs returned docs, and winning plans. Optimize by creating/selecting better indexes or restructuring the query.

### 9. What is a multikey index? Can you index arrays in MongoDB?

**Answer:** Yes. When you index a field that holds an array, MongoDB creates a multikey index to index each element in the array.

### 10. How would you design a schema for a messaging application?

**Answer:** Store messages in a separate `messages` collection, referencing `user_id` and `thread_id`. Denormalize sender/receiver info selectively for read optimization. Use indexes on `thread_id`, `timestamp`.

### 11. How would you store user preferences (key-value pairs) efficiently?

**Answer:** Use a single document per user with preferences as a nested object, or use a flat structure if querying specific keys is common. Use schema validation for standardization.

### 12. What is write concern and read concern?

**Answer:**

- Write concern defines the level of acknowledgment from MongoDB before considering a write successful (e.g., `w:1`, `w:majority`).
- Read concern defines the isolation level for reading data (`local`, `majority`, `linearizable`).

### 13. What happens during a replica set election?

**Answer:** When the primary goes down, secondaries elect a new primary using a voting algorithm. The election considers priority, oplog lag, and heartbeat failures.

### 14. What is the oplog and how does it work?

**Answer:** The oplog is a capped collection on the primary that records all operations that modify data. Secondaries apply these in the same order for replication.

### 15. What is a rollback in a replica set?

**Answer:** Rollback occurs when a primary steps down, and its writes are not present in the majority. MongoDB discards these writes to sync with the new primary.

### 16. How does sharding work internally?

**Answer:** Sharding distributes data across multiple shards. A shard key determines the partitioning. `mongos` is the router that directs queries to the appropriate shard(s), and config servers maintain metadata.

### 17. What are good and bad shard key choices?

**Answer:**

- **Good:** High cardinality, evenly distributed, and present in most queries.
- **Bad:** Low cardinality, monotonically increasing (e.g., timestamp), or not present in query filters.

### 18. What are change streams in MongoDB?

**Answer:** They allow real-time data change tracking using the oplog. Useful for event-driven architectures and reactive systems.

### 19. How do you prevent write conflicts in a high-concurrency environment?

**Answer:** Use optimistic concurrency control with versioning (`$inc` version), or transactions with proper isolation in MongoDB 4.0+.

### 20. How would you store time-series data in MongoDB?

**Answer:** Use time-series collections introduced in MongoDB 5.0. Provide `timeField`, `metaField`, and optionally, bucketing granularity for optimized performance and compression.

### 21. How does MongoDB handle memory and working sets?

**Answer:** MongoDB loads frequently accessed data (working set) into RAM. If the working set exceeds available memory, performance drops due to disk I/O.

### 22. What is aggregation pipeline optimization?

**Answer:** Place `$match` and `$project` early in the pipeline. Use `$merge`, `$out`, `$facet`, and `$group` wisely to avoid in-memory spills.

### 23. How do you monitor MongoDB performance?

**Answer:** Tools: `mongotop`, `mongostat`, `Atlas monitoring`, custom Prometheus exporters. Metrics: connection count, slow queries, page faults, replication lag.

### 24. What is MongoDB Atlas and how is it different from self-hosted MongoDB?

**Answer:** Atlas is a fully managed cloud MongoDB service with automatic scaling, backups, monitoring, and built-in security features like encryption and IP whitelisting.

### 25. How do you secure MongoDB deployments?

**Answer:** Enable authentication, use TLS/SSL, disable external network exposure, enforce IP whitelisting, and use role-based access control.

### 26. What is the impact of large documents in MongoDB?

**Answer:** MongoDB has a 16MB document limit. Large documents can cause performance issues in reads/writes and affect index efficiency.

### 27. How do you handle schema migrations in MongoDB?

**Answer:** Use background scripts or migration tools like `migrate-mongo`, handle versioning inside documents, and apply changes incrementally.

### 28. What happens if a document exceeds the 16MB limit?

**Answer:** MongoDB will throw a `DocumentTooLarge` error. Split large objects across multiple documents or collections.

### 29. Can you run transactions in sharded clusters?

**Answer:** Yes, since MongoDB 4.2. Transactions across shards are supported but can impact performance and increase complexity.

### 30. How do you avoid hotspotting in sharded clusters?

**Answer:** Use shard keys with high cardinality and randomness. Avoid monotonically increasing keys like timestamps.

### 31. How do you tune MongoDB for write-heavy workloads?

**Answer:**

- Use write concern `w:1`
- Use bulk operations
- Avoid unnecessary indexes
- Use capped collections if possible

### 32. What are capped collections?

**Answer:** Fixed-size collections that overwrite oldest entries when full. Fast insertions; ideal for logs, event storage.

### 33. How do you detect and fix index bloat?

**Answer:** Monitor index sizes via `db.collection.stats()`. Drop unused indexes using `db.collection.dropIndex()`, consolidate similar indexes.

### 34. Can indexes affect write performance?

**Answer:** Yes. Every insert/update must update all relevant indexes, so excessive or large indexes can slow down writes.

### 35. How do you implement optimistic concurrency control in MongoDB?

**Answer:** Include a `version` field in the document. Before update, match both `_id` and `version`, then increment `version`.

### 36. What is `$merge` and how is it different from `$out`?

**Answer:**

- `$out` replaces the entire target collection.
- `$merge` merges into the target with flexible behavior (`merge`, `replace`, `fail`, `keepExisting`).

### 37. How does vector search work in MongoDB Atlas?

**Answer:** MongoDB Atlas supports vector search through Atlas Vector Search, enabling approximate nearest neighbor (ANN) queries for use cases like semantic search and AI embeddings. You create a vector search index through Atlas UI or API and query using the `$vectorSearch` aggregation stage.

```js
// Create a vector search index via Atlas (JSON definition)
// {
//   "type": "vectorSearch",
//   "fields": [{ "type": "vector", "path": "embedding", "numDimensions": 512, "similarity": "cosine" }]
// }

// Query using $vectorSearch aggregation stage
db.images.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: [0.2, 0.3, 0.1 /* ...512 dimensions */],
      numCandidates: 100,
      limit: 10,
    },
  },
]);
```

> **Note:** Vector search requires MongoDB Atlas. It is not available in self-hosted MongoDB.

### 38. What is queryable encryption in MongoDB?

**Answer:** Queryable Encryption allows applications to encrypt sensitive fields client-side while still being able to query them using equality and range searches. The server never sees the plaintext data. It is generally available in MongoDB 7.0+ and supported through the MongoDB drivers.

```js
// Queryable encryption is configured at the driver level
// using AutoEncryptionOpts or explicit encryption APIs.
// The encrypted fields can be queried normally:
db.patients.find({ ssn: "123-45-6789" });
// The server processes this on encrypted data without decrypting it.
```

### 39. What are wildcard indexes and when would you use them?

**Answer:** Indexes that cover unknown or dynamic fields, useful for logging, analytics, or schemaless data where field names vary.

### 40. How does MongoDB handle durability and crash recovery?

**Answer:** MongoDB uses journaling to ensure data durability. On crash, the journal is replayed to bring the DB to a consistent state.

### 41. What is a use case where MongoDB would not be ideal?

**Answer:** Systems requiring complex transactions with strong ACID guarantees across many entities (e.g., financial ledgers) may be better served with RDBMS.

### 42. What is `dbHash` and how is it used?

**Answer:** `dbHash` returns MD5 hashes of collections. Used for comparing data consistency between replica set nodes.

### 43. How do you rollback a migration or change made to documents?

**Answer:** Keep a backup before migration, or implement reversible migrations with timestamp/version fields to identify and revert data.

### 44. How do you handle real-time notifications using MongoDB?

**Answer:** Use Change Streams to listen to insert/update/delete events in real time and trigger downstream actions via queues, WebSockets, or functions.

### 45. How do you perform a full-text search in MongoDB?

**Answer:** Create a text index and use `$text` query:

```js
db.articles.createIndex({ content: "text" });
db.articles.find({ $text: { $search: "mongodb performance" } });
```

### 46. Can a document belong to two shards?

**Answer:** No. A document belongs to exactly one shard, determined by the shard key.

### 47. What is bucket pattern in schema design?

**Answer:** Aggregating related records (e.g., log entries) into a single document (bucket) to reduce number of writes and improve read efficiency.

### 48. How would you migrate a MongoDB replica set with minimal downtime?

**Answer:** Add new nodes to the replica set, allow them to sync, promote them, and decommission old ones. Use maintenance mode and monitor replication lag.

### 49. What is `$facet` stage used for?

**Answer:** Allows multi-dimensional aggregation pipelines in parallel. E.g., compute multiple aggregates in a single query.

### 50. How do you implement rate limiting using MongoDB?

**Answer:** Use a TTL index on a collection storing API hits with timestamps. Count hits in the last window using aggregation.

Here are questions **51 to 66** from your MongoDB advanced interview set, fully expanded with detailed answers and shell commands:

---

### 51. How do you perform a text search with scoring in MongoDB?

**Answer:** MongoDB supports full-text search using a text index. You can also retrieve relevance scores using the `$meta` projection.

```sh
db.articles.createIndex({ content: "text" })
db.articles.find(
  { $text: { $search: "mongodb optimization" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

---

### 52. What is schema validation in MongoDB and how do you enforce it?

**Answer:** MongoDB supports JSON Schema validation from version 3.6+. You can define rules while creating a collection or modify existing ones with `collMod`.

```sh
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "age"],
      properties: {
        email: { bsonType: "string" },
        age: { bsonType: "int", minimum: 18 }
      }
    }
  }
})
```

---

### 53. How can you project specific fields in MongoDB queries?

**Answer:** Use the projection option in `find()` to include or exclude fields.

```sh
db.users.find({}, { name: 1, email: 1, _id: 0 })
```

---

### 54. How do you implement caching with MongoDB for repeated queries?

**Answer:** While MongoDB doesn't provide a built-in cache, you can use external cache systems (e.g., Redis) or utilize the in-memory WiredTiger cache. Check if queries are being cached by analyzing execution time and `executionStats`.

```sh
db.products.find({ category: "Books" }).explain("executionStats")
```

---

### 55. What is Change Data Capture (CDC) and how does MongoDB support it?

**Answer:** MongoDB supports CDC through **Change Streams**, which let you listen to real-time changes on collections or databases.

```sh
db.orders.watch()
```

---

### 56. How do you sort results in MongoDB?

**Answer:** Use `.sort()` on a cursor to sort documents.

```sh
db.products.find().sort({ price: -1 })
```

---

### 57. How do you back up and restore data in MongoDB?

**Answer:** Use `mongodump` to create a binary backup and `mongorestore` to restore it.

```sh
mongodump --db=mydb --out=/backups/may
mongorestore --db=mydb /backups/may/mydb
```

---

### 58. What is a compound text and regular index and how do you create it?

**Answer:** A compound index can combine regular and text fields but text must be last.

```sh
db.articles.createIndex({ status: 1, title: "text" })
```

---

### 59. How do you view current index usage in MongoDB?

**Answer:** Use the `aggregate()` pipeline with `$indexStats`.

```sh
db.orders.aggregate([{ $indexStats: {} }])
```

---

### 60. What is `explain()` in MongoDB and how do you use it?

**Answer:** `explain()` shows how MongoDB executes a query, including which indexes were used and how many documents were examined.

```sh
db.users.find({ age: { $gt: 25 } }).explain("executionStats")
```

---

### 61. How can you detect slow queries in MongoDB?

**Answer:** Enable the profiler or check `system.profile`.

```sh
db.setProfilingLevel(1, { slowms: 50 })
db.system.profile.find({ millis: { $gt: 50 } })
```

---

### 62. How do you analyze and troubleshoot lock issues?

**Answer:** Use `serverStatus` and look for locks section or use `currentOp`.

```sh
db.currentOp({ "locks": { $exists: true }, "secs_running": { $gt: 5 } })
```

---

### 63. What is Atlas Vector Search and how is it used for AI applications?

**Answer:** Atlas Vector Search enables similarity search on vector embeddings stored in MongoDB Atlas. It supports cosine, euclidean, and dot-product similarity functions, making it suitable for RAG (Retrieval-Augmented Generation), semantic search, and recommendation systems.

```js
// Query with $vectorSearch in an aggregation pipeline
db.products.aggregate([
  {
    $vectorSearch: {
      index: "product_embeddings",
      path: "embedding",
      queryVector: [0.2, 0.3, 0.1 /* ... */],
      numCandidates: 150,
      limit: 10,
    },
  },
  { $project: { name: 1, score: { $meta: "vectorSearchScore" } } },
]);
```

---

### 64. How can you enforce sorting on an index?

**Answer:** MongoDB uses indexes to optimize sorting. Create a compound index matching your sort pattern.

```sh
db.products.createIndex({ category: 1, price: -1 })
db.products.find({ category: "Books" }).sort({ price: -1 })
```

---

### 65. How do you run aggregation pipelines on a sharded cluster?

**Answer:** Aggregation pipelines are supported on sharded clusters. Make sure the first stage uses the shard key if possible.

```sh
db.orders.aggregate([
  { $match: { region: "APAC" } },
  { $group: { _id: "$region", total: { $sum: "$amount" } } }
])
```

---

### 66. How can you monitor replica set status?

**Answer:** Use `rs.status()` to check the health of each member in the replica set.

```sh
rs.status()
```

This provides information such as member state, health, uptime, and replication lag.

### 67. How do you troubleshoot replication lag?

**Answer:** Check the `optimeDate` using `rs.printSecondaryReplicationInfo()` and compare it to the primary.

```sh
rs.printSecondaryReplicationInfo()
```

Also inspect oplog window using:

```sh
db.getReplicationInfo()
```

> **Note:** `rs.printSlaveReplicationInfo()` was deprecated in MongoDB 4.4. Use `rs.printSecondaryReplicationInfo()` instead.

### 68. How can you identify unused indexes?

**Answer:** Use the `indexStats` aggregation to check access frequency.

```sh
db.collection.aggregate([
  { $indexStats: {} }
])
```

Look for indexes with `accesses.ops` = 0.

### 69. What is a TTL index and how is it used?

**Answer:** TTL (Time-To-Live) indexes automatically delete documents after a set time.

```sh
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })
```

This deletes documents 1 hour after the `createdAt` field.

### 70. How do you convert a replica set member to a hidden member?

**Answer:** Reconfigure the replica set and set `hidden: true`.

```sh
cfg = rs.conf()
cfg.members[1].hidden = true
cfg.members[1].priority = 0
rs.reconfig(cfg)
```

### 71. How can you trigger a manual failover in a replica set?

**Answer:** Use `rs.stepDown()` on the current primary.

```sh
rs.stepDown(60)
```

This forces the primary to step down for 60 seconds, promoting another member.

### 72. How to perform a rolling upgrade in a replica set?

**Answer:** Upgrade one secondary at a time. Once done, step down the primary and upgrade it last.

```sh
sudo apt install -y mongodb-org=VERSION
```

Repeat for each member.

### 73. How to encrypt data at rest in MongoDB?

**Answer:** Enable encryption using the WiredTiger engine with a key file.

```sh
mongod --enableEncryption --encryptionKeyFile /etc/mongodb-keyfile
```

### 74. How do you enable authentication in MongoDB?

**Answer:** Add a user with roles, then enable auth in the config.

```sh
db.createUser({ user: "admin", pwd: "pass", roles: ["root"] })
```

Enable in config:

```yaml
auth: true
```

### 75. How do you inspect the oplog?

**Answer:** Connect to the `local` database and query `oplog.rs`.

```sh
use local
db.oplog.rs.find().sort({ $natural: -1 }).limit(5)
```

### 76. What is a hot collection and how to detect it?

**Answer:** A hot collection receives frequent read/write activity. Use metrics or `mongostat`.

```sh
mongostat --discover
```

Check for collections with high ops/sec.

### 77. How do you run a repair operation on a corrupted MongoDB database?

**Answer:** Use `--repair` option during startup.

```sh
mongod --repair --dbpath /var/lib/mongodb
```

Make sure the instance is stopped before running.

### 78. How do you export and import large datasets?

**Answer:** Use `mongoexport` and `mongoimport` with `--batchSize` or `--numInsertionWorkers`.

```sh
mongoexport --db=mydb --collection=logs --out=logs.json
mongoimport --db=mydb --collection=logs --file=logs.json
```

### 79. What is a compound index and when should you use it?

**Answer:** A compound index is on multiple fields and improves queries filtering/sorting on those fields.

```sh
db.users.createIndex({ name: 1, age: -1 })
```

Useful when query uses both `name` and `age`.

### 80. How can you detect and avoid write conflicts?

**Answer:** Use `writeConcern` and handle duplicate key errors.

```sh
db.collection.insertOne({ _id: 1, name: "A" }, { writeConcern: { w: 1 } })
```

Use retries with backoff logic in your application.

### 81. How to shard a collection?

**Answer:** Enable sharding on the DB, then shard the collection.

```sh
sh.enableSharding("mydb")
sh.shardCollection("mydb.orders", { orderId: "hashed" })
```

### 82. How can you move a chunk manually in a sharded cluster?

**Answer:** Use `moveChunk` to redistribute.

```sh
sh.moveChunk("mydb.orders", { orderId: 123 }, "shard0001")
```

### 83. How do you monitor chunk distribution in a sharded cluster?

**Answer:** Use `sh.status()`.

```sh
sh.status()
```

This shows number of chunks per shard.

### 84. How do you detect a missing shard key in an insert?

**Answer:** The insert will fail with an error if no shard key is provided on a sharded collection.

```sh
db.orders.insert({ item: "pen" })
```

This fails if `orders` is sharded on `orderId`.

### 85. How does MongoDB Atlas support vector search for AI/ML?

**Answer:** MongoDB Atlas Vector Search supports similarity search on embedding vectors. Define a vector search index through Atlas, then query using `$vectorSearch` aggregation stage.

```js
// Query example using aggregation
db.items.aggregate([
  {
    $vectorSearch: {
      index: "embedding_index",
      path: "embedding",
      queryVector: [0.2, 0.3, 0.1 /* ...dimensions */],
      numCandidates: 100,
      limit: 10,
    },
  },
]);
```

> **Note:** Vector search is an Atlas-only feature. Self-managed deployments should use external vector databases or search engines.

### 86. How can you handle large file uploads in MongoDB?

**Answer:** Use GridFS for files >16MB.

```sh
mongofiles -d=mydb put largefile.pdf
```

Stores file in `fs.chunks` and `fs.files`.

### 87. What is `$merge` stage in aggregation?

**Answer:** It writes the output of an aggregation to a collection.

```sh
db.orders.aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } },
  { $merge: "orderStats" }
])
```

### 88. How to rotate logs in MongoDB?

**Answer:** Use `logRotate` command.

```sh
db.adminCommand({ logRotate: 1 })
```

Useful for truncating the logfile.

### 89. How to safely shutdown a replica set node?

**Answer:** Step down if primary, then shut down.

```sh
rs.stepDown()
db.adminCommand({ shutdown: 1 })
```

### 90. How can you list all databases and their sizes?

**Answer:** Use `db.adminCommand('listDatabases')`.

```sh
db.adminCommand({ listDatabases: 1 })
```

### 91. How can you drop all indexes on a collection?

**Answer:** Use `dropIndexes`.

```sh
db.users.dropIndexes()
```

### 92. How do you truncate a collection in MongoDB?

**Answer:** Use `deleteMany({})` or `drop()`.

```sh
db.logs.deleteMany({})
```

### 93. How can you set a read-only user?

**Answer:** Assign `read` role to user.

```sh
db.createUser({ user: "readonly", pwd: "pass", roles: [ { role: "read", db: "mydb" } ] })
```

### 94. What does `mongotop` show?

**Answer:** It shows read/write activity by collection.

```sh
mongotop
```

### 95. How can you check index usage per query?

**Answer:** Use `explain("executionStats")`.

```sh
db.users.find({ age: { $gt: 30 } }).explain("executionStats")
```

### 96. How can you analyze and reduce large documents?

**Answer:** Use schema analysis tools or check with BSON size.

```sh
Object.bsonsize(db.collection.findOne())
```

### 97. How can you find duplicate documents?

**Answer:** Use aggregation with `$group` and `$match`.

```sh
db.users.aggregate([
  { $group: { _id: "$email", count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
])
```

### 98. How do you enforce uniqueness on a nested field?

**Answer:** Create a unique index.

```sh
db.orders.createIndex({ "customer.email": 1 }, { unique: true })
```

### 99. What is `$out` vs `$merge`?

**Answer:** `$out` replaces entire collection, `$merge` can merge/upsert.

```sh
db.data.aggregate([ ..., { $out: "backupData" } ])
```

### 100. How to monitor connections in MongoDB?

**Answer:** Use server status.

```sh
db.serverStatus().connections
```

Shows current, available, and total created connections.
