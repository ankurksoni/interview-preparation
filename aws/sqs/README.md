# Amazon SQS Interview Questions & Answers

A curated list of **Amazon SQS (Simple Queue Service) interview questions** with practical, production-grade answers. Covers Standard vs FIFO queues, visibility timeout, DLQ, long polling, batching, and real-world messaging patterns.

---

### 1. **What is Amazon SQS?**

**Answer:** SQS is a fully managed **message queue** service for decoupling producers and consumers. It's pull-based — consumers poll the queue for messages.

**Key guarantees:**

- **Durability:** Messages stored redundantly across multiple AZs
- **Retention:** 1 minute to 14 days (default: 4 days)
- **Max message size:** 256 KB (use S3 for larger payloads)
- **Unlimited throughput** (Standard) / 300–3,000 msg/s (FIFO)

---

### 2. **What is the difference between Standard and FIFO queues?**

**Answer:**

| Feature    | Standard                            | FIFO                                                           |
| ---------- | ----------------------------------- | -------------------------------------------------------------- |
| Throughput | Unlimited                           | 300 msg/s (3,000 with batching, 70K with high throughput mode) |
| Ordering   | Best-effort                         | Strict per message group                                       |
| Delivery   | At-least-once (possible duplicates) | Exactly-once (5-min dedup window)                              |
| Queue name | Any                                 | Must end with `.fifo`                                          |
| Use case   | High volume, out-of-order OK        | Financial txns, order processing                               |

```ts
import { SQSClient, CreateQueueCommand } from "@aws-sdk/client-sqs";

const sqs = new SQSClient({});

// Standard queue
await sqs.send(
  new CreateQueueCommand({
    QueueName: "image-processing",
  }),
);

// FIFO queue with high throughput mode
await sqs.send(
  new CreateQueueCommand({
    QueueName: "order-processing.fifo",
    Attributes: {
      FifoQueue: "true",
      ContentBasedDeduplication: "true",
      DeduplicationScope: "messageGroup", // High throughput mode
      FifoThroughputLimit: "perMessageGroupId", // High throughput mode
    },
  }),
);
```

---

### 3. **What is visibility timeout?**

**Answer:** When a consumer receives a message, SQS hides it from other consumers for the **visibility timeout** period. If the consumer processes it and deletes it within that time, it's done. If not, the message reappears for another consumer.

```
Consumer A receives msg → msg hidden (30s default)
  ├── Processes and deletes → ✅ Done
  └── Crashes/timeout → msg reappears → Consumer B picks it up
```

```ts
// ❌ Bad: Visibility timeout too short for your processing time
await sqs.send(
  new CreateQueueCommand({
    QueueName: "video-transcode",
    Attributes: { VisibilityTimeout: "30" }, // 30s, but transcoding takes 5 min!
  }),
);

// ✅ Good: Set timeout > max processing time
await sqs.send(
  new CreateQueueCommand({
    QueueName: "video-transcode",
    Attributes: { VisibilityTimeout: "600" }, // 10 minutes
  }),
);

// ✅ Better: Extend dynamically during processing
import { ChangeMessageVisibilityCommand } from "@aws-sdk/client-sqs";

async function processWithHeartbeat(message: Message) {
  const heartbeat = setInterval(async () => {
    await sqs.send(
      new ChangeMessageVisibilityCommand({
        QueueUrl: queueUrl,
        ReceiptHandle: message.ReceiptHandle!,
        VisibilityTimeout: 120, // Extend by 2 more minutes
      }),
    );
  }, 60_000); // Every 60 seconds

  try {
    await transcodeVideo(message);
    await sqs.send(
      new DeleteMessageCommand({
        QueueUrl: queueUrl,
        ReceiptHandle: message.ReceiptHandle!,
      }),
    );
  } finally {
    clearInterval(heartbeat);
  }
}
```

---

### 4. **What is a Dead Letter Queue (DLQ)?**

**Answer:** A DLQ is a separate queue that receives messages that failed processing after a configured number of attempts (`maxReceiveCount`). This prevents poison messages from blocking the queue.

```ts
// CDK: Queue with DLQ
const dlq = new sqs.Queue(this, "DLQ", {
  retentionPeriod: cdk.Duration.days(14),
});

const mainQueue = new sqs.Queue(this, "OrderQueue", {
  deadLetterQueue: {
    queue: dlq,
    maxReceiveCount: 3, // After 3 failed attempts → move to DLQ
  },
});
```

**DLQ processing pattern:**

```ts
// ❌ Bad: Ignoring DLQ messages
// They sit there until retention expires, you lose visibility into failures

// ✅ Good: Monitor DLQ and process/alert
new cloudwatch.Alarm(this, "DLQAlarm", {
  metric: dlq.metricApproximateNumberOfMessagesVisible(),
  threshold: 1,
  evaluationPeriods: 1,
  alarmDescription: "Messages in DLQ — investigate failed processing",
});

// DLQ redrive: move messages back to source queue for reprocessing
// Available via console or StartMessageMoveTask API
```

```ts
import { StartMessageMoveTaskCommand } from "@aws-sdk/client-sqs";

// Redrive messages from DLQ back to source queue
await sqs.send(
  new StartMessageMoveTaskCommand({
    SourceArn: "arn:aws:sqs:us-east-1:123456789:OrderQueue-DLQ",
    DestinationArn: "arn:aws:sqs:us-east-1:123456789:OrderQueue",
    MaxNumberOfMessagesPerSecond: 50, // Throttle to avoid overload
  }),
);
```

---

### 5. **What is long polling vs short polling?**

**Answer:**

| Polling type            | Behavior                                | Cost                        |
| ----------------------- | --------------------------------------- | --------------------------- |
| Short polling (default) | Returns immediately (possibly empty)    | More API calls, higher cost |
| Long polling            | Waits up to 20s for a message to arrive | Fewer API calls, lower cost |

```ts
// ❌ Bad: Short polling in a tight loop (expensive, wasteful)
while (true) {
  const { Messages } = await sqs.send(
    new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      WaitTimeSeconds: 0, // Short poll — returns immediately
    }),
  );
  if (!Messages?.length) continue; // Empty response, wasted API call
}

// ✅ Good: Long polling (set at queue or receive level)
while (true) {
  const { Messages } = await sqs.send(
    new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      WaitTimeSeconds: 20, // Long poll — waits up to 20s
      MaxNumberOfMessages: 10, // Batch receive (max 10)
    }),
  );
  if (Messages?.length) {
    await Promise.all(Messages.map(processMessage));
  }
}
```

> **Best practice:** Set `ReceiveMessageWaitTimeSeconds` on the queue itself so all consumers automatically use long polling.

---

### 6. **How do you handle message ordering in FIFO queues?**

**Answer:** FIFO queues guarantee ordering **per message group**. Messages in different groups are processed independently and in parallel.

```ts
// Order updates — group by orderId so each order's events are ordered
await sqs.send(
  new SendMessageCommand({
    QueueUrl: fifoQueueUrl,
    MessageBody: JSON.stringify({ orderId: "ORD-123", status: "paid" }),
    MessageGroupId: "order-ORD-123", // All ORD-123 events ordered
    MessageDeduplicationId: "ORD-123-paid-v1",
  }),
);

await sqs.send(
  new SendMessageCommand({
    QueueUrl: fifoQueueUrl,
    MessageBody: JSON.stringify({ orderId: "ORD-123", status: "shipped" }),
    MessageGroupId: "order-ORD-123", // Same group → processed after 'paid'
    MessageDeduplicationId: "ORD-123-shipped-v1",
  }),
);

// Different order — independent group, can be processed in parallel
await sqs.send(
  new SendMessageCommand({
    QueueUrl: fifoQueueUrl,
    MessageBody: JSON.stringify({ orderId: "ORD-456", status: "paid" }),
    MessageGroupId: "order-ORD-456", // Different group → no ordering constraint with ORD-123
    MessageDeduplicationId: "ORD-456-paid-v1",
  }),
);
```

> **Pitfall:** Using a single `MessageGroupId` for all messages serializes everything. Use granular group IDs (e.g., per customer, per order) to maximize parallelism.

---

### 7. **How does SQS integrate with Lambda?**

**Answer:** Lambda polls SQS using an **event source mapping**. Lambda manages polling, batching, and scaling automatically.

```ts
// CDK: Lambda triggered by SQS
const fn = new lambda.Function(this, "Processor", {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("lambda"),
  timeout: cdk.Duration.seconds(30),
});

fn.addEventSource(
  new lambdaEventSources.SqsEventSource(queue, {
    batchSize: 10, // Up to 10 messages per invocation
    maxBatchingWindow: cdk.Duration.seconds(5), // Wait up to 5s to fill batch
    reportBatchItemFailures: true, // ✅ Critical: partial batch failure
  }),
);
```

**Handling partial batch failures:**

```ts
// ❌ Bad: Entire batch retries on any failure
export const handler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    await processMessage(record); // If one fails, ALL messages retry
  }
};

// ✅ Good: Report individual failures (requires reportBatchItemFailures: true)
import { SQSEvent, SQSBatchResponse } from "aws-lambda";

export const handler = async (event: SQSEvent): Promise<SQSBatchResponse> => {
  const failures: { itemIdentifier: string }[] = [];

  await Promise.all(
    event.Records.map(async (record) => {
      try {
        await processMessage(record);
      } catch (error) {
        failures.push({ itemIdentifier: record.messageId });
      }
    }),
  );

  return { batchItemFailures: failures };
  // Only failed messages return to queue; successful ones are deleted
};
```

---

### 8. **What is the SQS extended client library pattern?**

**Answer:** For messages larger than 256 KB, store the payload in S3 and send a reference via SQS. The `@aws-sdk/client-sqs` doesn't include this natively, but you can implement it:

```ts
// Producer: store large payload in S3
async function sendLargeMessage(body: object) {
  const payload = JSON.stringify(body);

  if (Buffer.byteLength(payload) > 200_000) {
    // ~200 KB threshold
    const key = `sqs-payloads/${crypto.randomUUID()}.json`;
    await s3.send(
      new PutObjectCommand({
        Bucket: "message-payloads",
        Key: key,
        Body: payload,
      }),
    );

    await sqs.send(
      new SendMessageCommand({
        QueueUrl: queueUrl,
        MessageBody: JSON.stringify({
          s3Bucket: "message-payloads",
          s3Key: key,
        }),
        MessageAttributes: {
          payloadType: { DataType: "String", StringValue: "s3" },
        },
      }),
    );
  } else {
    await sqs.send(
      new SendMessageCommand({
        QueueUrl: queueUrl,
        MessageBody: payload,
      }),
    );
  }
}

// Consumer: retrieve from S3 if needed
async function receiveLargeMessage(record: SQSRecord): Promise<object> {
  const attr = record.messageAttributes?.payloadType?.stringValue;
  if (attr === "s3") {
    const ref = JSON.parse(record.body);
    const obj = await s3.send(
      new GetObjectCommand({
        Bucket: ref.s3Bucket,
        Key: ref.s3Key,
      }),
    );
    return JSON.parse(await obj.Body!.transformToString());
  }
  return JSON.parse(record.body);
}
```

---

### 9. **How do you batch messages for cost efficiency?**

**Answer:** SQS charges per API request. Batching up to 10 messages per API call reduces costs by up to 10x.

```ts
import { SendMessageBatchCommand } from "@aws-sdk/client-sqs";

// ✅ Batch send (up to 10 messages per call)
const entries = messages.map((msg, i) => ({
  Id: `msg-${i}`,
  MessageBody: JSON.stringify(msg),
}));

// Split into chunks of 10
for (let i = 0; i < entries.length; i += 10) {
  const batch = entries.slice(i, i + 10);
  const result = await sqs.send(
    new SendMessageBatchCommand({
      QueueUrl: queueUrl,
      Entries: batch,
    }),
  );

  // Check for failures
  if (result.Failed?.length) {
    console.error("Failed messages:", result.Failed);
  }
}
```

```ts
// ✅ Batch delete after processing
import { DeleteMessageBatchCommand } from "@aws-sdk/client-sqs";

await sqs.send(
  new DeleteMessageBatchCommand({
    QueueUrl: queueUrl,
    Entries: messages.map((msg, i) => ({
      Id: `msg-${i}`,
      ReceiptHandle: msg.ReceiptHandle!,
    })),
  }),
);
```

---

### 10. **How do you implement idempotent message processing?**

**Answer:** With Standard queues (at-least-once delivery), your consumer must handle duplicate messages.

```ts
// ✅ Idempotent consumer using DynamoDB conditional write
import { ConditionalCheckFailedException } from "@aws-sdk/client-dynamodb";

async function processIdempotent(record: SQSRecord) {
  const message = JSON.parse(record.body);

  try {
    // Try to claim this messageId (idempotency key)
    await dynamodb.send(
      new PutCommand({
        TableName: "processed-messages",
        Item: {
          messageId: record.messageId,
          processedAt: new Date().toISOString(),
          ttl: Math.floor(Date.now() / 1000) + 86400 * 7, // 7-day TTL
        },
        ConditionExpression: "attribute_not_exists(messageId)",
      }),
    );

    // First time seeing this message — process it
    await handleOrder(message);
  } catch (error) {
    if (error instanceof ConditionalCheckFailedException) {
      console.log(`Duplicate message ${record.messageId}, skipping`);
      return; // Already processed, skip
    }
    throw error; // Unexpected error, let it retry
  }
}
```

---

### 11. **How do you secure SQS queues?**

**Answer:**

| Control               | How                                                |
| --------------------- | -------------------------------------------------- |
| Access control        | Queue policy (resource-based) + IAM policies       |
| Encryption at rest    | SSE-SQS (default) or SSE-KMS (custom key)          |
| Encryption in transit | HTTPS enforced via `aws:SecureTransport` condition |
| Cross-account         | Queue policy with specific account principals      |
| VPC-only access       | VPC endpoint + queue policy condition              |

```json
// Queue policy: enforce HTTPS and restrict to VPC endpoint
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "sqs:*",
      "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue",
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "sqs:*",
      "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue",
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "vpce-1a2b3c4d"
        }
      }
    }
  ]
}
```

---

### 12. **What is the difference between SQS and SNS?**

**Answer:**

| Feature     | SQS                                     | SNS                               |
| ----------- | --------------------------------------- | --------------------------------- |
| Model       | Queue (point-to-point)                  | Pub/Sub (fan-out)                 |
| Delivery    | Pull (consumer polls)                   | Push (SNS delivers)               |
| Consumers   | Single consumer group                   | Multiple subscribers              |
| Persistence | Messages retained until deleted/expired | No persistence (deliver or fail)  |
| Use case    | Work queues, task processing            | Event broadcasting, notifications |

**Common pattern:** Use them together — **SNS → SQS fan-out**:

```
                    ┌── SQS: order-service (processes orders)
SNS: order-topic ──┼── SQS: email-service (sends confirmations)
                    └── SQS: analytics (tracks metrics)
```

Each SQS queue processes independently at its own pace.

---

### 13. **How does SQS handle delayed messages?**

**Answer:** Two levels of delay:

| Method                     | Scope                 | Max delay              |
| -------------------------- | --------------------- | ---------------------- |
| Queue-level `DelaySeconds` | All messages in queue | 0–900 seconds (15 min) |
| Per-message `DelaySeconds` | Individual message    | 0–900 seconds (15 min) |

```ts
// Delay a specific message (e.g., retry after 5 minutes)
await sqs.send(
  new SendMessageCommand({
    QueueUrl: queueUrl,
    MessageBody: JSON.stringify({ action: "send-reminder", userId: "u-123" }),
    DelaySeconds: 300, // Available to consumers after 5 minutes
  }),
);
```

> **For delays > 15 minutes**, use Step Functions Wait state, EventBridge Scheduler, or a "delay" DynamoDB table with TTL + DynamoDB Streams.

---

### 14. **How do you monitor SQS?**

**Answer:**

Key CloudWatch metrics:

| Metric                                  | Meaning                     | Alert when                            |
| --------------------------------------- | --------------------------- | ------------------------------------- |
| `ApproximateNumberOfMessagesVisible`    | Messages waiting            | Growing → consumers too slow          |
| `ApproximateAgeOfOldestMessage`         | Oldest message age          | High → processing lag                 |
| `ApproximateNumberOfMessagesNotVisible` | Messages in-flight          | Near limit (120K Standard / 20K FIFO) |
| `NumberOfEmptyReceives`                 | Polls that returned nothing | High → switch to long polling         |
| `NumberOfMessagesSent`                  | Inflow rate                 | Spike detection                       |

```ts
// CDK: Alarm on processing lag
new cloudwatch.Alarm(this, "QueueLag", {
  metric: queue.metricApproximateAgeOfOldestMessage(),
  threshold: 300, // 5 minutes
  evaluationPeriods: 3,
  alarmDescription: "Messages waiting more than 5 minutes — scale consumers",
});

// Alarm on DLQ receiving messages
new cloudwatch.Alarm(this, "DLQMessages", {
  metric: dlq.metricApproximateNumberOfMessagesVisible(),
  threshold: 1,
  evaluationPeriods: 1,
  treatMissingData: cloudwatch.TreatMissingData.NOT_BREACHING,
});
```

---

### 15. **How do you scale SQS consumers?**

**Answer:**

**Lambda (automatic):**

- Lambda scales based on queue depth — up to 1,000 concurrent executions for Standard, 300 for FIFO (scales by message groups)
- No configuration needed beyond the event source mapping

**EC2/ECS (manual or auto-scaled):**

```ts
// CDK: Auto Scale ECS tasks based on SQS queue depth
const scaling = service.autoScaleTaskCount({ minCapacity: 1, maxCapacity: 20 });

scaling.scaleOnMetric("QueueDepthScaling", {
  metric: queue.metricApproximateNumberOfMessagesVisible(),
  scalingSteps: [
    { upper: 0, change: -1 }, // 0 messages → scale in
    { lower: 100, change: +1 }, // 100+ → add 1 task
    { lower: 500, change: +3 }, // 500+ → add 3 tasks
    { lower: 2000, change: +5 }, // 2000+ → add 5 tasks
  ],
  adjustmentType: autoscaling.AdjustmentType.CHANGE_IN_CAPACITY,
});
```

---

### 16. **What is the request-response pattern with SQS?**

**Answer:** SQS supports a temporary reply queue pattern using `ReplyTo` and correlation IDs:

```ts
// Requester: send request and wait for response
const responseQueueUrl = "https://sqs.../response-queue";
const correlationId = crypto.randomUUID();

await sqs.send(
  new SendMessageCommand({
    QueueUrl: requestQueueUrl,
    MessageBody: JSON.stringify({ action: "getPrice", productId: "P-100" }),
    MessageAttributes: {
      correlationId: { DataType: "String", StringValue: correlationId },
      replyTo: { DataType: "String", StringValue: responseQueueUrl },
    },
  }),
);

// Poll response queue for matching correlation ID
// (For production, consider API Gateway + Step Functions instead
//  of SQS for request-response patterns)
```

> **Note:** SQS is best for **asynchronous** processing. For synchronous request-response, prefer API Gateway, AppSync, or direct Lambda invocation.

---

### 17. **What are SQS temporary queues?**

**Answer:** The SQS Temporary Queue Client creates lightweight, automatically-deleted queues for request-reply patterns. Under the hood, it multiplexes virtual queues onto a single SQS queue.

**Use case:** Microservice A needs a one-time response from Microservice B. Instead of creating/deleting real queues, use virtual temporary queues.

---

### 18. **How do you implement a reliable processing pipeline with SQS?**

**Answer:**

```
API Gateway → Lambda (validate) → SQS (buffer) → Lambda (process) → DynamoDB
                                    │
                                    └── DLQ → CloudWatch Alarm → SNS → PagerDuty
```

Key reliability patterns:

1. **Idempotent consumers** — Handle duplicates (Standard queues)
2. **Visibility timeout heartbeat** — Extend for long-running tasks
3. **DLQ with monitoring** — Catch poison messages
4. **Partial batch failure reporting** — Don't retry entire batches
5. **Graceful shutdown** — Stop receiving, finish in-flight, then exit

```ts
// Graceful shutdown for ECS/EC2 worker
let isShuttingDown = false;

process.on("SIGTERM", () => {
  console.log("Received SIGTERM, finishing current messages...");
  isShuttingDown = true;
});

while (!isShuttingDown) {
  const { Messages } = await sqs.send(
    new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      WaitTimeSeconds: 20,
      MaxNumberOfMessages: 10,
    }),
  );

  if (Messages?.length) {
    await Promise.all(
      Messages.map(async (msg) => {
        await processMessage(msg);
        await sqs.send(
          new DeleteMessageCommand({
            QueueUrl: queueUrl,
            ReceiptHandle: msg.ReceiptHandle!,
          }),
        );
      }),
    );
  }
}

console.log("Shutdown complete");
process.exit(0);
```

---

### 19. **What are SQS access patterns for cross-account and cross-region?**

**Answer:**

**Cross-account:** Use a queue policy granting `sqs:SendMessage` to the producing account.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111111111111:role/ProducerRole" },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:222222222222:shared-queue"
    }
  ]
}
```

**Cross-region:** SQS queues are regional. For cross-region patterns:

- **SNS → SQS** — SNS can deliver cross-region to SQS
- **EventBridge** — Route events across regions with event bus rules
- **Application-level** — Producer sends to a queue in the target region

---

### 20. **How do you choose between SQS, SNS, EventBridge, and Kinesis?**

**Answer:**

| Service         | Model            | Best for                                             |
| --------------- | ---------------- | ---------------------------------------------------- |
| **SQS**         | Queue (pull)     | Work queues, decoupling, buffering                   |
| **SNS**         | Pub/Sub (push)   | Fan-out, notifications, broadcasting                 |
| **EventBridge** | Event bus (push) | Event routing, AWS service integration, scheduling   |
| **Kinesis**     | Stream (pull)    | Real-time analytics, ordered high-throughput, replay |

**Decision flow:**

1. Need to **buffer work** for async processing? → **SQS**
2. Need to **broadcast** to multiple consumers? → **SNS** (+ SQS for buffering)
3. Need **complex routing rules** or AWS service events? → **EventBridge**
4. Need **ordered, high-throughput** real-time data? → **Kinesis Data Streams**
5. Need fan-out + buffering + filtering? → **SNS → SQS** (fan-out pattern)

```
Use case examples:
─────────────────
Image processing queue          → SQS Standard
Order events to multiple teams  → SNS → SQS fan-out
React to S3/EC2/CloudTrail      → EventBridge
Real-time clickstream analytics → Kinesis Data Streams
Scheduled jobs                  → EventBridge Scheduler
```
