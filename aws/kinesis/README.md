# Amazon Kinesis Interview Questions & Answers

A curated list of **Amazon Kinesis interview questions** with practical, production-grade answers. Covers Data Streams, Firehose, shard management, consumer patterns, CDK/TypeScript examples, and real-world system design decisions.

---

### 1. **What is Amazon Kinesis and its components?**

**Answer:** Kinesis is a platform for **real-time streaming data** at any scale. It has four components:

| Component | Purpose | Use Case |
| --- | --- | --- |
| **Kinesis Data Streams (KDS)** | Real-time data ingestion and processing | Custom consumers, sub-second processing |
| **Kinesis Data Firehose** | Fully managed delivery to destinations | S3, Redshift, OpenSearch, HTTP endpoints |
| **Kinesis Data Analytics** | SQL/Flink on streaming data | Real-time aggregations, anomaly detection |
| **Kinesis Video Streams** | Ingest and process video streams | ML on video, live playback |

```ts
import { KinesisClient, CreateStreamCommand } from "@aws-sdk/client-kinesis";

const kinesis = new KinesisClient({});

await kinesis.send(
  new CreateStreamCommand({
    StreamName: "order-events",
    ShardCount: 4,
  }),
);
```

---

### 2. **When should you use Kinesis vs when NOT to?**

**Answer:**

| Use Kinesis When | Don't Use Kinesis When |
| --- | --- |
| Real-time processing (sub-second) | Batch processing with hourly/daily jobs (use S3 + Glue) |
| Event ordering matters (per-partition) | Simple fan-out messaging (use SNS) |
| Multiple consumers need same data | Point-to-point messaging (use SQS) |
| Replay/reprocess is required | Data doesn't need ordering or replay |
| High-throughput streaming (millions of events/sec) | Low volume, intermittent events (use EventBridge) |
| Log/metric aggregation pipeline | Simple event routing (use EventBridge) |

> **System Design Tip:** Use **Kinesis Data Streams** when you need real-time processing with custom consumer logic. Use **Firehose** when you just need to deliver data to S3/Redshift/OpenSearch with minimal code.

**Decision tree:**

```
Need sub-second processing? → Kinesis Data Streams
Just delivering to S3/Redshift? → Kinesis Firehose
Need SQL on stream? → Kinesis Data Analytics (or Managed Flink)
Pub/sub without ordering? → SNS
Queue with retry? → SQS
Event routing with filtering? → EventBridge
```

---

### 3. **What is a shard and how does it affect performance?**

**Answer:** A shard is the **base throughput unit** of a Kinesis stream:

| Metric | Per Shard |
| --- | --- |
| **Write** | 1 MB/s or 1,000 records/s |
| **Read (shared)** | 2 MB/s across all consumers |
| **Read (enhanced fan-out)** | 2 MB/s per consumer |
| **Data retention** | 24 hours (default) up to 365 days |

**Shard calculation:**

```
Required shards = MAX(
  incoming_data_MB_per_sec / 1,
  incoming_records_per_sec / 1000
)
```

```ts
// Example: 5 MB/s incoming → need 5 shards minimum
await kinesis.send(
  new CreateStreamCommand({
    StreamName: "clickstream",
    ShardCount: 8, // Over-provision slightly for bursts
  }),
);
```

> **Production Tip:** Always over-provision by 20–30% for burst handling. Shard splitting during a spike takes minutes and doesn't help immediately.

---

### 4. **How do you produce records to Kinesis from TypeScript?**

**Answer:**

```ts
import { KinesisClient, PutRecordCommand, PutRecordsCommand } from "@aws-sdk/client-kinesis";

const kinesis = new KinesisClient({});

// Single record
await kinesis.send(
  new PutRecordCommand({
    StreamName: "order-events",
    PartitionKey: "customer-123", // Determines shard placement
    Data: Buffer.from(JSON.stringify({ orderId: "ord-1", amount: 99.99 })),
  }),
);

// ✅ Batch: Always use PutRecords for throughput (up to 500 records/call)
const records = orders.map((order) => ({
  PartitionKey: order.customerId,
  Data: Buffer.from(JSON.stringify(order)),
}));

// Split into batches of 500
for (let i = 0; i < records.length; i += 500) {
  const batch = records.slice(i, i + 500);
  const result = await kinesis.send(
    new PutRecordsCommand({ StreamName: "order-events", Records: batch }),
  );

  // ❌ Bad: Ignoring FailedRecordCount
  // ✅ Good: Retry failed records with backoff
  if (result.FailedRecordCount && result.FailedRecordCount > 0) {
    const failedRecords = result.Records!
      .map((r, idx) => (r.ErrorCode ? batch[idx] : null))
      .filter(Boolean);
    // Retry failedRecords with exponential backoff
  }
}
```

> **Hidden Gem:** The `PartitionKey` determines which shard receives the record (via MD5 hash). Use a high-cardinality key (like `userId`) to distribute evenly. A low-cardinality key (like `country`) causes **hot shards**.

---

### 5. **What are the consumer patterns and how do they differ?**

**Answer:**

| Pattern | How It Works | Latency | Throughput | Cost |
| --- | --- | --- | --- | --- |
| **GetRecords (polling)** | Consumer polls shards via API | ~200ms–1s | 2 MB/s shared per shard | Lowest |
| **KCL (Kinesis Client Library)** | Managed consumer with checkpointing | ~200ms | 2 MB/s shared per shard | Low |
| **Enhanced Fan-Out (EFO)** | Push-based, dedicated throughput | ~70ms | 2 MB/s per consumer per shard | Higher |
| **Lambda (event source)** | Lambda triggered per batch | ~1–5s | Configurable batch size | Pay per invoke |

```ts
// CDK: Lambda consumer with Kinesis event source
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as eventsources from "aws-cdk-lib/aws-lambda-event-sources";

const processor = new lambda.Function(this, "StreamProcessor", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("lambda"),
  timeout: cdk.Duration.minutes(5),
});

processor.addEventSource(
  new eventsources.KinesisEventSource(stream, {
    startingPosition: lambda.StartingPosition.TRIM_HORIZON,
    batchSize: 100,
    maxBatchingWindow: cdk.Duration.seconds(5),  // Wait up to 5s to fill batch
    bisectBatchOnError: true,                     // Split batch on failure
    retryAttempts: 3,
    parallelizationFactor: 10,                    // 10 concurrent batches per shard
    onFailure: new eventsources.SqsDestination(dlq),
    reportBatchItemFailures: true,                // Partial batch failure reporting
  }),
);
```

---

### 6. **How do you handle partial batch failures with Lambda?**

**Answer:** Without proper handling, one bad record blocks the entire shard.

```ts
// Lambda handler with partial batch failure reporting
import { KinesisStreamEvent, KinesisStreamBatchResponse } from "aws-lambda";

export const handler = async (
  event: KinesisStreamEvent,
): Promise<KinesisStreamBatchResponse> => {
  const failures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    try {
      const payload = JSON.parse(
        Buffer.from(record.kinesis.data, "base64").toString(),
      );
      await processRecord(payload);
    } catch (err) {
      failures.push({ itemIdentifier: record.kinesis.sequenceNumber });
    }
  }

  return { batchItemFailures: failures };
};
```

> **Critical:** You MUST enable `reportBatchItemFailures: true` in the event source mapping AND return `batchItemFailures` from Lambda. Without both, Lambda retries the entire batch.

---

### 7. **What are Kinesis's important config parameters?**

**Answer:**

| Parameter | Description | Production Recommendation |
| --- | --- | --- |
| `ShardCount` | Number of shards (throughput units) | Over-provision 20–30% for bursts |
| `RetentionPeriodHours` | How long data stays (24h–8760h) | 24h for real-time; 168h if replay needed |
| `StreamModeDetails` | ON_DEMAND or PROVISIONED | ON_DEMAND for unpredictable traffic |
| `PartitionKey` | Determines shard routing | High-cardinality key (userId, sessionId) |
| `EnhancedMonitoring` | Per-shard CloudWatch metrics | Enable for production troubleshooting |
| `EncryptionType` | KMS encryption at rest | Always use KMS in production |

```ts
// CDK: Production-ready Kinesis stream
import * as kinesis from "aws-cdk-lib/aws-kinesis";

const stream = new kinesis.Stream(this, "EventStream", {
  streamName: "order-events",
  streamMode: kinesis.StreamMode.ON_DEMAND,  // Auto-scales shards
  retentionPeriod: cdk.Duration.hours(168),  // 7 days for replay
  encryption: kinesis.StreamEncryption.KMS,
});

// ✅ Enhanced monitoring for per-shard metrics
const cfnStream = stream.node.defaultChild as kinesis.CfnStream;
cfnStream.addPropertyOverride("StreamModeDetails", {
  StreamMode: "ON_DEMAND",
});
```

> **Hidden Gem:** `ON_DEMAND` mode auto-scales up to 200 MB/s write and 400 MB/s read without any shard management. You only pay for actual throughput used.

---

### 8. **How does Kinesis Firehose work and when to use it?**

**Answer:** Firehose is a **fully managed delivery stream** — no shards, no consumers, no code needed for basic delivery.

```ts
// CDK: Firehose delivery to S3 with transformation
import * as firehose from "aws-cdk-lib/aws-kinesisfirehose";
import * as destinations from "@aws-cdk/aws-kinesisfirehose-destinations-alpha";

const deliveryStream = new firehose.CfnDeliveryStream(this, "LogDelivery", {
  deliveryStreamName: "log-delivery",
  deliveryStreamType: "DirectPut",
  extendedS3DestinationConfiguration: {
    bucketArn: dataBucket.bucketArn,
    roleArn: firehoseRole.roleArn,
    prefix: "logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/",
    errorOutputPrefix: "errors/",
    bufferingHints: {
      sizeInMBs: 128,         // Buffer up to 128 MB
      intervalInSeconds: 300,  // Or flush every 5 minutes
    },
    compressionFormat: "GZIP",
    dataFormatConversionConfiguration: {
      enabled: true,
      inputFormatConfiguration: { deserializer: { openXJsonSerDe: {} } },
      outputFormatConfiguration: { serializer: { parquetSerDe: { compression: "SNAPPY" } } },
      schemaConfiguration: {
        databaseName: "analytics",
        tableName: "logs",
        roleArn: firehoseRole.roleArn,
      },
    },
  },
});
```

**Firehose vs Data Streams:**

| Feature | Firehose | Data Streams |
| --- | --- | --- |
| Management | Fully managed | You manage consumers |
| Latency | 60s–900s buffer | Sub-second |
| Destinations | S3, Redshift, OpenSearch, HTTP | Anything (custom consumer) |
| Transformation | Lambda (optional) | Full custom processing |
| Scaling | Automatic | Manual (PROVISIONED) or ON_DEMAND |
| Cost | Per GB ingested | Per shard-hour + per GB |

---

### 9. **What are Kinesis's resiliency and robustness patterns?**

**Answer:**

| Concern | Solution |
| --- | --- |
| Producer failures | Retry with `PutRecords` + backoff on `FailedRecordCount` |
| Consumer failures | KCL checkpointing; Lambda `bisectBatchOnError` |
| Hot shards | Use high-cardinality partition keys; add random suffix |
| Data loss | Multi-AZ replication (built-in); extend retention period |
| Consumer lag | Monitor `IteratorAge` metric; scale consumers |
| Poison records | `bisectBatchOnError` + DLQ for failed records |
| Shard split during processing | KCL handles shard lineage automatically |

```ts
// ❌ Bad: Single low-cardinality partition key → hot shard
await kinesis.send(
  new PutRecordCommand({
    StreamName: "events",
    PartitionKey: "US",  // All US traffic on one shard
    Data: Buffer.from(payload),
  }),
);

// ✅ Good: Add random suffix for distribution
await kinesis.send(
  new PutRecordCommand({
    StreamName: "events",
    PartitionKey: `US-${Math.floor(Math.random() * 100)}`,
    Data: Buffer.from(payload),
  }),
);
```

**Critical CloudWatch alarms:**

```ts
// CDK: Monitor consumer lag
new cloudwatch.Alarm(this, "ConsumerLagAlarm", {
  metric: new cloudwatch.Metric({
    namespace: "AWS/Kinesis",
    metricName: "GetRecords.IteratorAgeMilliseconds",
    dimensionsMap: { StreamName: "order-events" },
    statistic: "Maximum",
    period: cdk.Duration.minutes(1),
  }),
  threshold: 60_000,  // Alert if consumer is 60s behind
  evaluationPeriods: 3,
  alarmDescription: "Kinesis consumer falling behind",
});

// Monitor write throttling
new cloudwatch.Alarm(this, "WriteThrottleAlarm", {
  metric: new cloudwatch.Metric({
    namespace: "AWS/Kinesis",
    metricName: "WriteProvisionedThroughputExceeded",
    dimensionsMap: { StreamName: "order-events" },
    statistic: "Sum",
    period: cdk.Duration.minutes(1),
  }),
  threshold: 1,
  evaluationPeriods: 3,
});
```

---

### 10. **How does Enhanced Fan-Out work and when to use it?**

**Answer:** EFO gives each consumer a **dedicated 2 MB/s pipe per shard** via HTTP/2 push (SubscribeToShard API). Without EFO, all consumers share 2 MB/s per shard.

```
Without EFO (shared): Shard → [Consumer A, B, C] share 2 MB/s
With EFO (dedicated):  Shard → Consumer A: 2 MB/s
                        Shard → Consumer B: 2 MB/s
                        Shard → Consumer C: 2 MB/s
```

**When to use EFO:**

| Use EFO | Skip EFO |
| --- | --- |
| 3+ consumers on same stream | 1–2 consumers |
| Sub-100ms latency required | Seconds of latency acceptable |
| Consumers can't keep up (lag) | Consumers easily keep pace |
| Budget allows ($0.015/shard-hour/consumer) | Cost-sensitive workload |

```ts
import {
  KinesisClient,
  RegisterStreamConsumerCommand,
} from "@aws-sdk/client-kinesis";

// Register an enhanced fan-out consumer
await kinesis.send(
  new RegisterStreamConsumerCommand({
    StreamARN: "arn:aws:kinesis:us-east-1:123456789:stream/order-events",
    ConsumerName: "analytics-consumer",
  }),
);
```

---

### 11. **What are Kinesis cost optimization strategies?**

**Answer:**

| Strategy | How | Impact |
| --- | --- | --- |
| Use ON_DEMAND mode | Auto-scales, pay for actual use | Best for variable traffic |
| Batch with PutRecords | Up to 500 records per call | Reduces API calls |
| Use Firehose for S3 delivery | No shard management overhead | Simpler + often cheaper |
| Minimize retention period | 24h vs 168h vs 365 days | 24h is cheapest |
| Avoid Enhanced Fan-Out unless needed | EFO adds per-consumer cost | Save $0.015/shard-hour/consumer |
| Right-size shards (PROVISIONED) | Monitor `IncomingBytes` metric | Avoid paying for idle shards |
| Aggregate records client-side | Kinesis Producer Library (KPL) aggregation | More records per PUT |

**Cost comparison (us-east-1):**

| Component | Cost |
| --- | --- |
| Shard-hour (PROVISIONED) | $0.015/hr ($10.80/month) |
| ON_DEMAND write | $0.08 per GB |
| ON_DEMAND read | $0.04 per GB |
| PUT payload unit (25KB) | $0.014 per million |
| Extended retention (>24h) | $0.023 per shard-hour |
| Long-term retention (>7d) | $0.013 per GB/month |
| Enhanced Fan-Out | $0.015/shard-hour + $0.013 per GB |

---

### 12. **How do you implement exactly-once processing with Kinesis?**

**Answer:** Kinesis provides **at-least-once** delivery. For exactly-once, implement **idempotency** on the consumer side.

```ts
import { DynamoDBClient, PutItemCommand } from "@aws-sdk/client-dynamodb";

const dynamo = new DynamoDBClient({});

export const handler = async (event: KinesisStreamEvent) => {
  const failures: { itemIdentifier: string }[] = [];

  for (const record of event.Records) {
    const seqNum = record.kinesis.sequenceNumber;
    const payload = JSON.parse(
      Buffer.from(record.kinesis.data, "base64").toString(),
    );

    try {
      // Idempotency: Use conditional write
      await dynamo.send(
        new PutItemCommand({
          TableName: "ProcessedRecords",
          Item: {
            pk: { S: `ORDER#${payload.orderId}` },
            sequenceNumber: { S: seqNum },
            processedAt: { S: new Date().toISOString() },
          },
          ConditionExpression: "attribute_not_exists(pk)", // Skip if already processed
        }),
      );

      await processOrder(payload);
    } catch (err: any) {
      if (err.name === "ConditionalCheckFailedException") {
        continue; // Already processed, skip
      }
      failures.push({ itemIdentifier: seqNum });
    }
  }

  return { batchItemFailures: failures };
};
```

---

### 13. **What hidden/lesser-known Kinesis features are useful?**

**Answer:**

| Feature | What It Does | Why It Matters |
| --- | --- | --- |
| **Server-side timestamps** | `ApproximateArrivalTimestamp` on each record | Debug latency, order by ingestion time |
| **KPL aggregation** | Pack multiple user records into one Kinesis record | 10x+ throughput, lower costs |
| **Shard-level metrics** | Per-shard CloudWatch metrics via Enhanced Monitoring | Identify hot shards by name |
| **ON_DEMAND auto-scaling** | Scales to 200 MB/s write without config | Zero capacity planning |
| **SubscribeToShard** | HTTP/2 push for sub-70ms latency | Real-time dashboards |
| **Cross-account access** | Resource-based policy on stream | Central data lake ingestion |
| **ListShards pagination** | Get shard map with filters | Identify shard lineage after splits |

> **Hidden Gem:** `ApproximateArrivalTimestamp` is set server-side when Kinesis receives the record. Use it (not client timestamps) for reliable latency calculations and event ordering.

---

### 14. **How do you design a real-time analytics pipeline with Kinesis?**

**Answer:**

```
Producers → Kinesis Data Streams → [Fan-out to multiple consumers]
                                      ├── Lambda → DynamoDB (real-time dashboard)
                                      ├── Firehose → S3 (data lake, Parquet)
                                      ├── Managed Flink → Aggregations → Kinesis out
                                      └── EFO Consumer → OpenSearch (search/alerting)
```

```ts
// CDK: Complete pipeline
const eventStream = new kinesis.Stream(this, "Events", {
  streamMode: kinesis.StreamMode.ON_DEMAND,
  retentionPeriod: cdk.Duration.hours(168),
  encryption: kinesis.StreamEncryption.KMS,
});

// Real-time processor
const realtimeProcessor = new lambda.Function(this, "RealtimeProcessor", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "realtime.handler",
  code: lambda.Code.fromAsset("lambda"),
  timeout: cdk.Duration.minutes(1),
  memorySize: 256,
});

realtimeProcessor.addEventSource(
  new eventsources.KinesisEventSource(eventStream, {
    startingPosition: lambda.StartingPosition.LATEST,
    batchSize: 100,
    maxBatchingWindow: cdk.Duration.seconds(1),
    parallelizationFactor: 10,
    bisectBatchOnError: true,
    reportBatchItemFailures: true,
    onFailure: new eventsources.SqsDestination(dlq),
  }),
);

// S3 archival via Firehose
new firehose.CfnDeliveryStream(this, "Archive", {
  deliveryStreamType: "KinesisStreamAsSource",
  kinesisStreamSourceConfiguration: {
    kinesisStreamArn: eventStream.streamArn,
    roleArn: firehoseRole.roleArn,
  },
  extendedS3DestinationConfiguration: {
    bucketArn: archiveBucket.bucketArn,
    roleArn: firehoseRole.roleArn,
    prefix: "events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
    bufferingHints: { sizeInMBs: 128, intervalInSeconds: 300 },
    compressionFormat: "GZIP",
  },
});
```

---

### 15. **What are Kinesis scalability limits?**

**Answer:**

| Limit | Value | Workaround |
| --- | --- | --- |
| Shards per stream (PROVISIONED) | 500 (soft limit, can increase) | Request limit increase |
| ON_DEMAND write throughput | 200 MB/s (auto) | Request limit increase |
| Record size | 1 MB | Compress or use S3 reference |
| PutRecords batch size | 500 records / 5 MB | Split into multiple calls |
| Consumers per stream (EFO) | 20 | Use shared consumers for low-priority |
| Lambda parallelization | 10 per shard | Use more shards for higher parallelism |
| Data retention | 24h–365 days | Export to S3 for permanent storage |
| GetRecords per shard | 5 calls/sec | Use EFO for high-read scenarios |

> **Production Tip:** When hitting `ReadProvisionedThroughputExceeded`, either add EFO for the bottleneck consumer, or increase shard count. Don't just add retry — fix the root cause.
