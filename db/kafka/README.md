# Apache Kafka Interview Questions & Answers

A curated list of **Apache Kafka interview questions** — easy to medium difficulty with practical examples. Covers architecture, topics, partitions, consumer groups, exactly-once semantics, and real-world streaming patterns.

---

## Fundamentals

### 1. **What is Apache Kafka?**

**Answer:** Kafka is a distributed **event streaming platform** used for building real-time data pipelines and streaming applications. It acts as a durable, high-throughput, fault-tolerant **commit log**.

Core use cases:

- **Messaging** — Decouple producers and consumers
- **Activity tracking** — Clickstreams, page views, logs
- **Event sourcing** — Store state changes as immutable events
- **Stream processing** — Transform, aggregate, enrich in real-time
- **Data integration** — Connect databases, services, and data lakes

---

### 2. **What are the core components of Kafka?**

**Answer:**

| Component             | Role                                                                       |
| --------------------- | -------------------------------------------------------------------------- |
| **Broker**            | A Kafka server that stores data and serves clients                         |
| **Topic**             | A named feed/category for messages (like a table)                          |
| **Partition**         | A topic is split into partitions for parallelism                           |
| **Producer**          | Publishes messages to topics                                               |
| **Consumer**          | Reads messages from topics                                                 |
| **Consumer Group**    | Group of consumers sharing work on a topic                                 |
| **ZooKeeper / KRaft** | Cluster coordination and metadata (KRaft replaces ZooKeeper in Kafka 3.3+) |
| **Offset**            | Position of a message within a partition                                   |

```
Producer ──▶ Topic (3 partitions) ──▶ Consumer Group
             ├── Partition 0 ──▶ Consumer A
             ├── Partition 1 ──▶ Consumer B
             └── Partition 2 ──▶ Consumer C
```

---

### 3. **What is a Kafka topic?**

**Answer:** A topic is a logical category/feed name. Messages published to a topic are stored in **partitions** — the unit of parallelism and ordering.

Key properties:

- Topics can have 1 to thousands of partitions
- Messages are **immutable** — once written, they cannot be changed
- Messages are retained based on **time** (default: 7 days) or **size**
- Each message within a partition has a unique, sequential **offset**

```bash
# Create a topic with 6 partitions and replication factor 3
kafka-topics.sh --create \
  --topic order-events \
  --partitions 6 \
  --replication-factor 3 \
  --bootstrap-server localhost:9092
```

---

### 4. **What is a partition and why does it matter?**

**Answer:** Partitions are how Kafka achieves **parallelism** and **ordering**.

- Messages within a partition are **strictly ordered** (by offset)
- Messages across partitions have **no ordering guarantee**
- Each partition is stored on one broker (leader) with replicas on others
- More partitions = more parallelism = higher throughput

```
Topic: order-events (3 partitions)

Partition 0: [msg0] [msg1] [msg2] [msg3] → ordered within partition
Partition 1: [msg0] [msg1] [msg2]         → ordered within partition
Partition 2: [msg0] [msg1]                → ordered within partition
                                          → NO ordering across partitions
```

**How messages are assigned to partitions:**

1. **Explicit partition** — Producer specifies partition number
2. **Key-based** — Hash of the key determines partition (same key → same partition → ordered)
3. **Round-robin** — No key → distributed evenly

```ts
// Key-based: all events for order ORD-123 go to the same partition
await producer.send({
  topic: "order-events",
  messages: [
    {
      key: "ORD-123", // Partition = hash(key) % numPartitions
      value: JSON.stringify({ status: "created", amount: 99.99 }),
    },
  ],
});
```

---

### 5. **What is an offset?**

**Answer:** An offset is a unique, sequential ID assigned to each message within a partition. It acts as the **position marker** for consumers.

```
Partition 0:  [0] [1] [2] [3] [4] [5] [6]
                         ↑
                    Consumer position (committed offset = 2)
                    Next read starts at offset 3
```

- Offsets are **per-partition**, not per-topic
- Consumers track their position by **committing offsets**
- If a consumer crashes and restarts, it resumes from the last committed offset
- Offsets are stored in a special internal topic: `__consumer_offsets`

---

### 6. **What is a consumer group?**

**Answer:** A consumer group is a set of consumers that cooperate to consume a topic. Each partition is assigned to **exactly one** consumer in the group.

```
Topic: orders (4 partitions)

Consumer Group A (3 consumers):
  Consumer 1 → Partition 0, Partition 1
  Consumer 2 → Partition 2
  Consumer 3 → Partition 3

Consumer Group B (2 consumers):    ← Independent group, sees ALL messages too
  Consumer 1 → Partition 0, Partition 1
  Consumer 2 → Partition 2, Partition 3
```

**Key rules:**

- Max useful consumers in a group = number of partitions
- If consumers > partitions, some consumers sit idle
- If a consumer fails, its partitions are **rebalanced** to remaining consumers
- Different consumer groups are independent — each sees all messages

---

### 7. **What is the difference between Kafka and a traditional message queue?**

**Answer:**

| Feature         | Kafka                            | Traditional Queue (RabbitMQ/SQS)   |
| --------------- | -------------------------------- | ---------------------------------- |
| Model           | Commit log (pull)                | Message queue (push/pull)          |
| Retention       | Time/size based (keeps messages) | Deleted after consumption          |
| Consumer groups | Multiple groups read same data   | One consumer gets each message     |
| Replay          | ✅ Read from any offset          | ❌ Once consumed, gone             |
| Ordering        | Per-partition                    | Per-queue (FIFO mode)              |
| Throughput      | Millions of msgs/sec             | Thousands to hundreds of thousands |
| Routing         | Topic + partition key            | Exchanges, queues, routing keys    |

---

### 8. **What is a Kafka broker?**

**Answer:** A broker is a single Kafka server. A Kafka **cluster** is a group of brokers.

Broker responsibilities:

- Receive and store messages from producers
- Serve messages to consumers
- Host partition replicas
- One broker per partition is the **leader** (handles reads/writes); others are **followers** (replicate data)

```
Cluster (3 brokers):
  Broker 0: [Topic A, Partition 0 (leader)] [Topic A, Partition 1 (follower)]
  Broker 1: [Topic A, Partition 0 (follower)] [Topic A, Partition 1 (leader)]
  Broker 2: [Topic A, Partition 0 (follower)] [Topic A, Partition 2 (leader)]
```

> **Replication factor 3** means every partition exists on 3 brokers. If a broker fails, another broker's replica becomes the new leader — **zero downtime**.

---

### 9. **What is the role of ZooKeeper and what is KRaft?**

**Answer:**

**ZooKeeper (legacy):**

- Managed cluster metadata (broker list, topic configs, partition leaders)
- Handled leader election for partitions
- Tracked consumer group offsets (very old versions)

**KRaft (Kafka Raft — the replacement):**

- Introduced in Kafka 3.3 (production-ready in 3.5)
- Metadata managed by Kafka brokers themselves using Raft consensus
- No external dependency — simpler operations, faster failover
- ZooKeeper is **deprecated** and will be removed in Kafka 4.0

---

## Producers

### 10. **How does the Kafka producer work?**

**Answer:**

```
Application → Producer → Serializer → Partitioner → RecordAccumulator → Sender → Broker
```

1. **Serialize** — Convert key/value to bytes
2. **Partition** — Determine target partition (by key hash or round-robin)
3. **Batch** — Accumulate messages in memory (per partition)
4. **Send** — Background thread sends batches to brokers
5. **Acknowledge** — Broker responds with success or error

```ts
import { Kafka } from "kafkajs";

const kafka = new Kafka({ brokers: ["broker1:9092", "broker2:9092"] });
const producer = kafka.producer();
await producer.connect();

await producer.send({
  topic: "user-events",
  messages: [
    {
      key: "user-123",
      value: JSON.stringify({ event: "login", ts: Date.now() }),
    },
    {
      key: "user-456",
      value: JSON.stringify({ event: "signup", ts: Date.now() }),
    },
  ],
});
```

---

### 11. **What are producer acknowledgment levels (acks)?**

**Answer:**

| Setting         | Behavior                      | Durability                                     | Performance |
| --------------- | ----------------------------- | ---------------------------------------------- | ----------- |
| `acks=0`        | Don't wait for any broker     | Possible data loss                             | Fastest     |
| `acks=1`        | Wait for leader ACK only      | Data loss if leader crashes before replication | Balanced    |
| `acks=all` (-1) | Wait for all in-sync replicas | **No data loss**                               | Slowest     |

```ts
const producer = kafka.producer({
  // acks: -1 is default in KafkaJS — safest
});

// Combined with min.insync.replicas=2 on the topic/broker,
// this guarantees at least 2 replicas have the data before ACK
```

> **Production recommendation:** `acks=all` + `min.insync.replicas=2` + `replication.factor=3`. This allows one broker to fail without data loss or unavailability.

---

### 12. **What is idempotent producer?**

**Answer:** An idempotent producer prevents **duplicate messages** caused by network retries. Kafka assigns each producer a ProducerId and sequence number, so the broker detects and deduplicates retries.

```ts
const producer = kafka.producer({
  idempotent: true, // Enable exactly-once per partition
  maxInFlightRequests: 5, // Can be > 1 with idempotent producer
});
```

- Enabled by default in Kafka 3.0+ (`enable.idempotence=true`)
- Guarantees **exactly-once** within a single producer session and partition
- No duplicate messages from network retries

---

## Consumers

### 13. **What is the difference between auto-commit and manual commit?**

**Answer:**

```ts
// ❌ Risky: Auto-commit (default in some clients)
// Offsets committed on a timer — message may be lost if crash happens after commit but before processing
const consumer = kafka.consumer({
  groupId: "my-group",
  // autoCommit defaults vary by client
});

// ✅ Better: Manual commit after successful processing
await consumer.run({
  autoCommit: false,
  eachMessage: async ({ topic, partition, message }) => {
    await processMessage(message);

    // Commit only after successful processing
    await consumer.commitOffsets([
      {
        topic,
        partition,
        offset: (parseInt(message.offset) + 1).toString(),
      },
    ]);
  },
});
```

| Mode                         | Risk                                       | Throughput |
| ---------------------------- | ------------------------------------------ | ---------- |
| Auto-commit                  | Message loss (committed but not processed) | Higher     |
| Manual commit (each message) | At-least-once (safe)                       | Lower      |
| Manual commit (batch)        | At-least-once (balanced)                   | Medium     |

---

### 14. **What is a consumer rebalance?**

**Answer:** A rebalance is the process of redistributing partitions among consumers in a group. It happens when:

1. A consumer **joins** the group
2. A consumer **leaves** (crashes, network issue, or graceful shutdown)
3. Topic partitions change (new partitions added)

**Problem:** During a rebalance, the group **stops consuming** temporarily.

**Strategies:**
| Strategy | Behavior |
|----------|----------|
| Eager (default) | Revoke all partitions, then reassign. Full stop-the-world. |
| Cooperative Sticky | Only revoke partitions that need to move. Minimal pause. |

> Use **CooperativeStickyAssignor** in production to minimize rebalance disruption.

---

### 15. **How do you reset consumer offsets?**

**Answer:**

```bash
# Reset to earliest (reprocess all messages)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events \
  --reset-offsets --to-earliest --execute

# Reset to a specific timestamp
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events \
  --reset-offsets --to-datetime '2024-04-01T00:00:00.000' --execute

# Reset to specific offset
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events \
  --reset-offsets --to-offset 1000 --execute

# Shift by N (forward or backward)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic order-events \
  --reset-offsets --shift-by -100 --execute
```

> **Important:** Stop all consumers in the group before resetting offsets.

---

## Reliability & Performance

### 16. **What is the ISR (In-Sync Replicas)?**

**Answer:** ISR is the set of replicas that are fully caught up with the leader. Only ISR members can be elected as new leader.

```
Partition 0:
  Leader: Broker 0     (offset 100)
  ISR:    Broker 0, Broker 1 (offset 100), Broker 2 (offset 98 - slightly behind)

If Broker 2 falls too far behind (replica.lag.time.max.ms), it's removed from ISR.
```

**`min.insync.replicas`** — Minimum ISR size required to accept writes:

- Set to `2` with replication factor `3` → tolerates 1 broker failure
- If ISR drops below this, producers with `acks=all` get errors (not-enough-replicas)

---

### 17. **How do you handle exactly-once semantics in Kafka?**

**Answer:** Kafka supports exactly-once in two scopes:

**1. Within Kafka (transactions):**

```ts
// Transactional producer: atomic writes across multiple partitions
const producer = kafka.producer({
  idempotent: true,
  transactionalId: "order-processor-1",
});

await producer.connect();

const transaction = await producer.transaction();
try {
  await transaction.send({
    topic: "processed-orders",
    messages: [{ key: orderId, value: JSON.stringify(result) }],
  });

  // Commit consumer offset as part of the transaction
  await transaction.sendOffsets({
    consumerGroupId: "order-processors",
    topics: [
      { topic: "raw-orders", partitions: [{ partition: 0, offset: "42" }] },
    ],
  });

  await transaction.commit();
} catch (error) {
  await transaction.abort();
}
```

**2. End-to-end (Kafka → external system):**

- Kafka guarantees exactly-once internally
- For external systems (databases), use **idempotent consumers** (check before write)

---

### 18. **What are the key producer performance tuning parameters?**

**Answer:**

| Parameter                | Default | Tuning advice                              |
| ------------------------ | ------- | ------------------------------------------ |
| `batch.size`             | 16 KB   | Increase to 64–256 KB for throughput       |
| `linger.ms`              | 0       | Set to 5–50 ms (wait to fill batches)      |
| `compression.type`       | none    | Use `lz4` or `zstd` (70%+ smaller)         |
| `buffer.memory`          | 32 MB   | Increase for high-throughput producers     |
| `max.in.flight.requests` | 5       | 1 for strict ordering (without idempotent) |

```ts
const producer = kafka.producer({
  allowAutoTopicCreation: false,
  idempotent: true,
  // KafkaJS applies batching internally
});
```

> **Compression** is almost always worth enabling. `lz4` for speed, `zstd` for best ratio.

---

### 19. **What is log compaction?**

**Answer:** Instead of deleting old messages by time, log compaction keeps the **latest value for each key**. Earlier messages with the same key are garbage collected.

```
Before compaction:
  [key=A, v=1] [key=B, v=1] [key=A, v=2] [key=B, v=2] [key=A, v=3]

After compaction:
  [key=B, v=2] [key=A, v=3]    ← Only latest per key
```

**Use cases:**

- Database changelog (CDC) — latest state per record
- Configuration snapshots
- User profiles — always keep latest version

```bash
# Enable compaction on a topic
kafka-configs.sh --alter \
  --entity-type topics --entity-name user-profiles \
  --add-config cleanup.policy=compact \
  --bootstrap-server localhost:9092
```

> **Tombstones:** A message with a key and `null` value signals deletion. After a configurable period, the tombstone and old values are removed.

---

### 20. **How do you choose the number of partitions?**

**Answer:**

**Formula:** `Partitions = max(T/Pt, T/Ct)`

- T = Target throughput (MB/s)
- Pt = Throughput per partition for producer
- Ct = Throughput per partition for consumer

**Rules of thumb:**

- Start with **number of consumers you want to run in parallel**
- If each consumer processes 10 MB/s and you need 60 MB/s → 6 partitions minimum
- More partitions = more parallelism, but also more overhead
- **You can add partitions later but cannot reduce them**
- Under 10 partitions for low-throughput topics, 30–100 for high-throughput

> **Ordering caveat:** Adding partitions changes key-to-partition mapping. Existing consumers may see out-of-order messages for existing keys.

---

## Architecture Patterns

### 21. **What is Kafka Connect?**

**Answer:** Kafka Connect is a framework for streaming data **between Kafka and external systems** without writing code.

| Type                 | Direction        | Examples                             |
| -------------------- | ---------------- | ------------------------------------ |
| **Source connector** | External → Kafka | Debezium (DB CDC), JDBC, S3, MongoDB |
| **Sink connector**   | Kafka → External | Elasticsearch, S3, JDBC, Snowflake   |

```json
// Debezium PostgreSQL CDC source connector
{
  "name": "postgres-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-host",
    "database.port": "5432",
    "database.user": "debezium",
    "database.dbname": "orders_db",
    "topic.prefix": "cdc",
    "table.include.list": "public.orders,public.customers",
    "plugin.name": "pgoutput"
  }
}
// → Creates topics: cdc.public.orders, cdc.public.customers
```

---

### 22. **What is the event sourcing pattern with Kafka?**

**Answer:** Store all state changes as immutable events. Current state is derived by replaying events.

```
Topic: account-events (compacted)

[key=ACC-1] { "type": "AccountCreated", "balance": 0 }
[key=ACC-1] { "type": "MoneyDeposited", "amount": 1000 }
[key=ACC-1] { "type": "MoneyWithdrawn", "amount": 200 }
[key=ACC-1] { "type": "MoneyDeposited", "amount": 500 }

Current state of ACC-1: balance = 0 + 1000 - 200 + 500 = $1300
```

**Benefits:**

- Complete audit trail
- Replay events to rebuild state or debug
- Multiple consumers can derive different views from the same events
- Time travel — what was the state at any point in time?

---

### 23. **What is the difference between Kafka and Amazon Kinesis?**

**Answer:**

| Feature         | Kafka                                        | Kinesis Data Streams               |
| --------------- | -------------------------------------------- | ---------------------------------- |
| Hosting         | Self-managed or Confluent Cloud              | Fully managed (AWS)                |
| Throughput      | Millions msg/s                               | 1 MB/s per shard (2 MB/s enhanced) |
| Retention       | Configurable (unlimited with tiered storage) | 24 hrs – 365 days                  |
| Consumer groups | Native                                       | KCL library                        |
| Exactly-once    | ✅ Transactions                              | ❌ At-least-once                   |
| Ecosystem       | Connect, Streams, ksqlDB, Schema Registry    | Firehose, Analytics, Lambda        |
| Cost            | Infrastructure + ops                         | Per-shard hourly + per-GB          |

**Choose Kafka:** Large scale, complex stream processing, multi-datacenter, need exactly-once.
**Choose Kinesis:** AWS-native, no ops overhead, moderate scale.

---

### 24. **What is Schema Registry?**

**Answer:** Schema Registry stores and enforces schemas (Avro, Protobuf, JSON Schema) for Kafka messages. It prevents producers from sending incompatible data.

```
Producer → Schema Registry (validate schema) → Kafka
                                                 ↓
Consumer ← Schema Registry (deserialize with schema) ←
```

**Compatibility modes:**
| Mode | Rule |
|------|------|
| BACKWARD | New schema can read old data |
| FORWARD | Old schema can read new data |
| FULL | Both backward and forward compatible |
| NONE | No compatibility check |

> **Production best practice:** Use Avro or Protobuf with BACKWARD compatibility. This allows consumers to be deployed before producers.

---

### 25. **How do you monitor a Kafka cluster?**

**Answer:**

Key metrics to watch:

| Metric                          | Healthy value  | Alert when                              |
| ------------------------------- | -------------- | --------------------------------------- |
| **Consumer lag**                | Low and stable | Growing — consumers can't keep up       |
| **Under-replicated partitions** | 0              | > 0 — broker replication issue          |
| **ISR shrinks**                 | 0              | > 0 — replicas falling behind           |
| **Request latency (p99)**       | < 100 ms       | Spikes — disk or network issues         |
| **Disk usage per broker**       | < 70%          | > 80% — add storage or reduce retention |
| **Active controller count**     | 1 per cluster  | 0 — no controller, cluster issues       |

```bash
# Check consumer group lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-group

# Output:
# TOPIC      PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders     0          45023           45030           7
# orders     1          38901           38901           0
# orders     2          42156           42200           44
```

> **Tools:** Prometheus + Grafana (JMX metrics), Confluent Control Center, Burrow (lag monitoring), AKHQ (web UI).

---

## Advanced Error Handling, Scalability & Performance

### 26. **How do you handle poison pill messages (messages that always fail)?**

**Answer:** A poison pill is a message that causes a consumer to crash or throw every time it's processed. Without handling, it blocks the entire partition.

```js
async function processWithDLQ(message) {
  const MAX_RETRIES = 3;
  const retryCount = parseInt(
    message.headers?.["retry-count"]?.toString() || "0",
  );

  try {
    const value = JSON.parse(message.value.toString());
    await handleEvent(value);
  } catch (error) {
    if (retryCount >= MAX_RETRIES) {
      // Send to Dead Letter Topic (DLT) — never retry again
      await producer.send({
        topic: `${message.topic}.DLT`,
        messages: [
          {
            key: message.key,
            value: message.value,
            headers: {
              ...message.headers,
              "original-topic": message.topic,
              "original-partition": String(message.partition),
              "original-offset": String(message.offset),
              "error-message": error.message,
              "failed-at": new Date().toISOString(),
            },
          },
        ],
      });
      console.error(
        `Message sent to DLT after ${MAX_RETRIES} retries:`,
        error.message,
      );
      return; // commit offset, move on
    }

    // Retry: republish with incremented retry count
    await producer.send({
      topic: `${message.topic}.retry`,
      messages: [
        {
          key: message.key,
          value: message.value,
          headers: {
            ...message.headers,
            "retry-count": String(retryCount + 1),
          },
        },
      ],
    });
  }
}
```

| Strategy          | How it works                  | Pros                    | Cons               |
| ----------------- | ----------------------------- | ----------------------- | ------------------ |
| Skip & log        | Log error, commit offset      | Simple                  | Data loss          |
| Retry topic       | Republish to `.retry` topic   | Doesn't block partition | Extra topics       |
| Dead letter topic | After N retries → `.DLT`      | Full audit trail        | Needs DLT consumer |
| Pause partition   | Stop consuming that partition | No data loss            | Reduces throughput |

---

### 27. **How do you scale Kafka consumers and what are the limits?**

**Answer:** Consumer scaling is bounded by the **number of partitions**. You can never have more active consumers in a group than partitions.

```
Topic: orders (6 partitions)

1 consumer  → reads all 6 partitions (max throughput per consumer)
3 consumers → each reads 2 partitions (balanced)
6 consumers → each reads 1 partition (maximum parallelism)
7 consumers → 1 idle consumer (wasted) ⚠️

Scaling rule: consumers in group ≤ partition count
```

```sh
# Increase partitions (CAUTION: can't decrease, and key ordering changes)
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic orders --partitions 12

# Check consumer group lag — high lag means consumers can't keep up
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
```

| Scaling Lever                         | When to Use                          | Trade-off                             |
| ------------------------------------- | ------------------------------------ | ------------------------------------- |
| Add consumers (up to partition count) | Lag growing, consumers underutilized | More instances to manage              |
| Increase partitions                   | Consumers maxed out                  | Breaks key ordering during transition |
| Increase `fetch.min.bytes`            | Low throughput, wasteful polling     | Higher latency                        |
| Increase batch size                   | Producer bottleneck                  | More memory per batch                 |
| Tune `max.poll.records`               | Consumer processing time varies      | Too high = rebalance timeouts         |

---

### 28. **What happens during a Kafka broker failure and how does the cluster recover?**

**Answer:**

```
Scenario: Broker 2 fails (hosts partitions P1-leader, P3-replica, P5-replica)

Before failure:
  P1: Leader=B2, ISR=[B2, B1, B3]  ← B2 is leader
  P3: Leader=B1, ISR=[B1, B2, B3]  ← B2 is replica
  P5: Leader=B3, ISR=[B3, B2]      ← B2 is replica

After B2 fails:
  P1: Leader=B1, ISR=[B1, B3]      ← B1 elected new leader (was in ISR)
  P3: Leader=B1, ISR=[B1, B3]      ← B2 removed from ISR
  P5: Leader=B3, ISR=[B3]          ← B2 removed from ISR

Recovery when B2 comes back:
  P1: Leader=B1, ISR=[B1, B3, B2]  ← B2 catches up, rejoins ISR
  (optional) Preferred replica election → B2 becomes leader again
```

| Config                                 | Impact                                        | Recommended                |
| -------------------------------------- | --------------------------------------------- | -------------------------- |
| `min.insync.replicas=2`                | Writes need 2+ ISR members to succeed         | Yes for critical data      |
| `unclean.leader.election.enable=false` | Never elect non-ISR replica as leader         | Yes (prevents data loss)   |
| `replica.lag.time.max.ms=30000`        | How long before slow replica removed from ISR | 10000–30000                |
| `num.replica.fetchers=4`               | Parallel replication threads                  | Increase for fast catch-up |

---

### 29. **What are the key performance factors for Kafka producers and consumers?**

**Answer:**

**Producer tuning:**

| Parameter                               | Default | Tuned (throughput) | Tuned (latency) |
| --------------------------------------- | ------- | ------------------ | --------------- |
| `batch.size`                            | 16 KB   | 64–128 KB          | 16 KB           |
| `linger.ms`                             | 0       | 10–50 ms           | 0               |
| `compression.type`                      | none    | lz4 or zstd        | lz4             |
| `buffer.memory`                         | 32 MB   | 64–128 MB          | 32 MB           |
| `acks`                                  | all     | all                | 1 (if loss OK)  |
| `max.in.flight.requests.per.connection` | 5       | 5                  | 5               |

**Consumer tuning:**

| Parameter              | Default | Impact                                          |
| ---------------------- | ------- | ----------------------------------------------- |
| `fetch.min.bytes`      | 1       | Higher = fewer requests, more latency           |
| `fetch.max.wait.ms`    | 500     | How long broker waits if min.bytes not met      |
| `max.poll.records`     | 500     | Records per poll — set based on processing time |
| `max.poll.interval.ms` | 300000  | Exceeded = consumer removed from group          |
| `session.timeout.ms`   | 45000   | Heartbeat timeout                               |
| `auto.offset.reset`    | latest  | earliest for reprocessing, latest for live      |

```sh
# Producer performance test
kafka-producer-perf-test.sh --topic test-perf \
  --num-records 1000000 --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092 \
    acks=all compression.type=lz4 batch.size=65536 linger.ms=20

# Consumer performance test
kafka-consumer-perf-test.sh --topic test-perf \
  --messages 1000000 \
  --broker-list localhost:9092 \
  --group perf-test
```

---

### 30. **How do you handle schema evolution in Kafka without breaking consumers?**

**Answer:** Use **Schema Registry** with compatibility rules to evolve schemas safely across services.

| Compatibility      |  Can Add Fields   | Can Remove Fields | Can Change Types |
| ------------------ | :---------------: | :---------------: | :--------------: |
| BACKWARD (default) | ✅ (with default) |        ✅         |        ❌        |
| FORWARD            |        ✅         | ✅ (with default) |        ❌        |
| FULL               | ✅ (with default) | ✅ (with default) |        ❌        |
| NONE               |        ✅         |        ✅         |  ✅ (dangerous)  |

```json
// Avro schema v1
{
  "type": "record",
  "name": "Order",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "amount", "type": "double" },
    { "name": "currency", "type": "string" }
  ]
}

// Avro schema v2 — BACKWARD compatible (old consumers can read new data)
{
  "type": "record",
  "name": "Order",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "amount", "type": "double" },
    { "name": "currency", "type": "string" },
    { "name": "discount", "type": "double", "default": 0.0 }
  ]
}
```

> **Rule:** BACKWARD compatibility is the safest default — deploy new consumers first, then new producers. Old consumers ignore the new `discount` field, and new consumers get a default value of `0.0` for old messages.
