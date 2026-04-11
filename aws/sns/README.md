# Amazon SNS Interview Questions & Answers

A curated list of **Amazon SNS (Simple Notification Service) interview questions** with practical, production-grade answers. Covers topics, subscriptions, fan-out, filtering, FIFO, delivery, security, and real-world messaging patterns.

---

### 1. **What is Amazon SNS?**

**Answer:** SNS is a fully managed **pub/sub** messaging service. Publishers send messages to a **topic**, and SNS delivers them to all **subscribers** (fan-out). It supports push-based delivery — subscribers don't poll.

**Supported subscriber protocols:**

- SQS, Lambda, HTTP/HTTPS, Email, SMS, Kinesis Data Firehose, Platform application (mobile push)

---

### 2. **What is the difference between SNS Standard and FIFO topics?**

**Answer:**

| Feature       | Standard topic                 | FIFO topic                               |
| ------------- | ------------------------------ | ---------------------------------------- |
| Throughput    | Nearly unlimited               | 300 msg/s (batching: 3,000)              |
| Ordering      | Best-effort                    | Strict per message group                 |
| Deduplication | No                             | Yes (content or ID-based)                |
| Subscribers   | All protocols                  | **SQS FIFO only**                        |
| Use case      | Notifications, alerts, fan-out | Financial transactions, order processing |

```ts
import { SNSClient, CreateTopicCommand } from "@aws-sdk/client-sns";

const sns = new SNSClient({});

// FIFO topic (name must end with .fifo)
await sns.send(
  new CreateTopicCommand({
    Name: "order-events.fifo",
    Attributes: {
      FifoTopic: "true",
      ContentBasedDeduplication: "true",
    },
  }),
);
```

---

### 3. **How does SNS message filtering work?**

**Answer:** Filter policies let subscribers receive only a subset of messages, avoiding unnecessary processing. Filters are applied at the **subscription** level.

```ts
import { SubscribeCommand } from "@aws-sdk/client-sns";

// ❌ Bad: All subscribers get all messages, filter in application code
await sns.send(
  new SubscribeCommand({
    TopicArn: "arn:aws:sns:us-east-1:123456789:order-events",
    Protocol: "sqs",
    Endpoint: "arn:aws:sqs:us-east-1:123456789:payment-queue",
  }),
);

// ✅ Good: SNS filters at the broker — only matching messages are delivered
await sns.send(
  new SubscribeCommand({
    TopicArn: "arn:aws:sns:us-east-1:123456789:order-events",
    Protocol: "sqs",
    Endpoint: "arn:aws:sqs:us-east-1:123456789:payment-queue",
    Attributes: {
      FilterPolicy: JSON.stringify({
        eventType: ["order.paid", "order.refunded"],
        amount: [{ numeric: [">=", 100] }],
      }),
      FilterPolicyScope: "MessageBody", // Filter on body (not just attributes)
    },
  }),
);
```

**Filter operators:**

- Exact match: `["value1", "value2"]`
- Prefix: `[{"prefix": "order."}]`
- Numeric: `[{"numeric": [">=", 100, "<=", 1000]}]`
- Exists/not exists: `[{"exists": true}]`
- OR logic: `["a", "b"]` — matches either value
- Suffix (2023+): `[{"suffix": ".jpg"}]`

---

### 4. **What is the SNS fan-out pattern?**

**Answer:** Fan-out sends a single message to multiple consumers simultaneously via multiple subscriptions. The classic pattern is **SNS + SQS fan-out**.

```
                    ┌── SQS: order-processing
                    │
  Publisher → SNS ──┼── SQS: inventory-update
                    │
                    ├── SQS: analytics-pipeline
                    │
                    └── Lambda: send-notification
```

```ts
// CDK: Fan-out with SNS → multiple SQS queues
const topic = new sns.Topic(this, "OrderEvents");

const orderQueue = new sqs.Queue(this, "OrderProcessing");
const inventoryQueue = new sqs.Queue(this, "InventoryUpdate");
const analyticsQueue = new sqs.Queue(this, "Analytics");

topic.addSubscription(
  new subscriptions.SqsSubscription(orderQueue, {
    filterPolicy: {
      eventType: sns.SubscriptionFilter.stringFilter({
        allowlist: ["order.created"],
      }),
    },
  }),
);
topic.addSubscription(new subscriptions.SqsSubscription(inventoryQueue));
topic.addSubscription(new subscriptions.SqsSubscription(analyticsQueue));
```

> **Why SNS + SQS instead of SNS alone?** SQS provides buffering, retry, and decoupling. If a subscriber is down, messages wait in the queue instead of being lost.

---

### 5. **How do you handle SNS delivery failures?**

**Answer:** Delivery behavior depends on the subscriber protocol:

| Protocol  | On failure                                   | Retry policy                    |
| --------- | -------------------------------------------- | ------------------------------- |
| SQS       | Moves to SQS DLQ (configure on queue)        | SNS retries per delivery policy |
| Lambda    | Retries (2 attempts), then SNS DLQ           | Exponential backoff             |
| HTTP/S    | Retries with backoff (up to 23 days default) | Configurable delivery policy    |
| Email/SMS | Best-effort, no retry                        | No DLQ support                  |

```ts
// Set a DLQ for failed Lambda/SQS deliveries
await sns.send(
  new SubscribeCommand({
    TopicArn: topicArn,
    Protocol: "lambda",
    Endpoint: lambdaArn,
    Attributes: {
      RedrivePolicy: JSON.stringify({
        deadLetterTargetArn: "arn:aws:sqs:us-east-1:123456789:sns-dlq",
      }),
    },
  }),
);
```

---

### 6. **How does SNS message delivery work with HTTP/HTTPS endpoints?**

**Answer:**

1. **Subscription confirmation** — SNS sends a `SubscribeURL` to the endpoint; the endpoint must confirm by visiting the URL
2. **Message delivery** — SNS POSTs a JSON payload with `Message`, `MessageId`, `TopicArn`, `Signature`
3. **Signature verification** — Always verify the SNS signature to prevent spoofing

```ts
// ❌ Bad: Accepting messages without verification
app.post("/sns-webhook", (req, res) => {
  const message = JSON.parse(req.body.Message);
  processOrder(message); // Could be spoofed!
  res.sendStatus(200);
});

// ✅ Good: Verify SNS message signature
import { MessageValidator } from "sns-validator";
const validator = new MessageValidator();

app.post("/sns-webhook", async (req, res) => {
  try {
    const validated = await validator.validate(req.body);

    if (validated.Type === "SubscriptionConfirmation") {
      // Confirm by fetching the SubscribeURL
      await fetch(validated.SubscribeURL);
    } else if (validated.Type === "Notification") {
      await processOrder(JSON.parse(validated.Message));
    }
    res.sendStatus(200);
  } catch {
    res.sendStatus(403);
  }
});
```

---

### 7. **What is the maximum SNS message size and how do you handle large payloads?**

**Answer:** SNS message size limit is **256 KB**. For larger payloads, use the **extended client library** pattern:

1. Store the payload in S3
2. Send a reference (S3 pointer) as the SNS message
3. Consumer retrieves the full payload from S3

```ts
// Large payload pattern
const key = `events/${crypto.randomUUID()}.json`;

// Store payload in S3
await s3.send(
  new PutObjectCommand({
    Bucket: "event-payloads",
    Key: key,
    Body: JSON.stringify(largePayload),
  }),
);

// Send reference via SNS
await sns.send(
  new PublishCommand({
    TopicArn: topicArn,
    Message: JSON.stringify({
      bucket: "event-payloads",
      key,
      size: JSON.stringify(largePayload).length,
    }),
    MessageAttributes: {
      payloadLocation: { DataType: "String", StringValue: "s3" },
    },
  }),
);
```

---

### 8. **What is SNS message deduplication in FIFO topics?**

**Answer:** FIFO topics prevent duplicate message delivery within a 5-minute deduplication window.

**Two methods:**

1. **Content-based** — SNS generates a hash of the message body
2. **MessageDeduplicationId** — You provide an explicit ID

```ts
// Content-based deduplication (enabled on topic)
await sns.send(
  new PublishCommand({
    TopicArn: "arn:aws:sns:us-east-1:123456789:orders.fifo",
    Message: JSON.stringify({ orderId: "ORD-123", status: "paid" }),
    MessageGroupId: "order-ORD-123", // Required for FIFO
    // Dedup automatically based on message body hash
  }),
);

// Explicit deduplication ID
await sns.send(
  new PublishCommand({
    TopicArn: "arn:aws:sns:us-east-1:123456789:orders.fifo",
    Message: JSON.stringify({ orderId: "ORD-123", status: "paid" }),
    MessageGroupId: "order-ORD-123",
    MessageDeduplicationId: "ORD-123-paid-v1", // Your own ID
  }),
);
```

---

### 9. **How do you secure SNS topics?**

**Answer:**

| Control               | Mechanism                                        |
| --------------------- | ------------------------------------------------ |
| Who can publish       | Topic policy (resource-based) or IAM policy      |
| Who can subscribe     | Topic policy `sns:Subscribe`                     |
| Encryption at rest    | SSE with KMS (`KmsMasterKeyId`)                  |
| Encryption in transit | HTTPS endpoints enforced                         |
| Cross-account         | Topic policy with `aws:PrincipalOrgID` condition |
| VPC                   | VPC endpoint for SNS (PrivateLink)               |

```json
// Topic policy: only allow publishing from specific account within org
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "*" },
      "Action": "SNS:Publish",
      "Resource": "arn:aws:sns:us-east-1:123456789:order-events",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-abc123def4"
        }
      }
    }
  ]
}
```

---

### 10. **How does SNS integrate with EventBridge?**

**Answer:** SNS and EventBridge are complementary:

| Feature    | SNS                           | EventBridge                                     |
| ---------- | ----------------------------- | ----------------------------------------------- |
| Pattern    | Fan-out (pub/sub)             | Event routing (event bus)                       |
| Filtering  | Attribute/body filter         | Rich content-based rules                        |
| Targets    | SQS, Lambda, HTTP, Email, SMS | 20+ targets (Step Functions, API Gateway, etc.) |
| Replay     | ❌ No                         | ✅ Archive & replay                             |
| Schema     | ❌ No                         | ✅ Schema Registry                              |
| Throughput | Very high                     | Lower (soft limits)                             |

**Use SNS when:** High-throughput fan-out to SQS/Lambda, mobile push, SMS/email.
**Use EventBridge when:** Complex routing rules, multiple AWS service integrations, event replay.

---

### 11. **What are SNS message attributes?**

**Answer:** Message attributes are key-value metadata sent alongside the message body. They're used for filter policies and pass-through metadata without requiring subscribers to parse the body.

```ts
await sns.send(new PublishCommand({
  TopicArn: topicArn,
  Message: JSON.stringify({ orderId: 'ORD-456', items: [...] }),
  MessageAttributes: {
    eventType: { DataType: 'String', StringValue: 'order.created' },
    priority: { DataType: 'Number', StringValue: '1' },
    region: { DataType: 'String', StringValue: 'us-east' },
  },
}));
```

> **Max 10 attributes** per message. Attribute names, types, and values count toward the 256 KB message size limit.

---

### 12. **How do you implement SNS mobile push notifications?**

**Answer:** SNS supports push notifications to iOS (APNs), Android (FCM), and more via **platform applications**.

**Flow:**

1. Create a platform application in SNS (provide FCM/APNs credentials)
2. Register each device as a **platform endpoint**
3. Publish to the endpoint (direct) or a topic (broadcast)

```ts
import {
  CreatePlatformEndpointCommand,
  PublishCommand,
} from "@aws-sdk/client-sns";

// Register a device
const { EndpointArn } = await sns.send(
  new CreatePlatformEndpointCommand({
    PlatformApplicationArn: "arn:aws:sns:us-east-1:123456789:app/GCM/MyApp",
    Token: deviceFcmToken, // From Firebase
    CustomUserData: userId,
  }),
);

// Send push notification
await sns.send(
  new PublishCommand({
    TargetArn: EndpointArn,
    Message: JSON.stringify({
      GCM: JSON.stringify({
        notification: {
          title: "Order Shipped",
          body: "Your order #123 is on the way!",
        },
        data: { orderId: "ORD-123", screen: "order-tracking" },
      }),
    }),
    MessageStructure: "json", // Required for platform-specific payloads
  }),
);
```

---

### 13. **What is the difference between SNS publish and PublishBatch?**

**Answer:** `PublishBatch` sends up to **10 messages** in a single API call, reducing costs and latency.

```ts
import { PublishBatchCommand } from "@aws-sdk/client-sns";

// ✅ Batch publish (fewer API calls, lower cost)
await sns.send(
  new PublishBatchCommand({
    TopicArn: topicArn,
    PublishBatchRequestEntries: events.map((event, i) => ({
      Id: `msg-${i}`,
      Message: JSON.stringify(event),
      MessageAttributes: {
        eventType: { DataType: "String", StringValue: event.type },
      },
      // For FIFO topics:
      // MessageGroupId: event.groupId,
      // MessageDeduplicationId: event.dedupId,
    })),
  }),
);
```

---

### 14. **How do you monitor SNS?**

**Answer:**

Key CloudWatch metrics:

| Metric                           | What it tells you                            |
| -------------------------------- | -------------------------------------------- |
| `NumberOfMessagesPublished`      | Publish throughput                           |
| `NumberOfNotificationsDelivered` | Successful deliveries                        |
| `NumberOfNotificationsFailed`    | Delivery failures (alarm on this!)           |
| `PublishSize`                    | Message sizes (watch for approaching 256 KB) |
| `SMSSuccessRate`                 | SMS-specific delivery rate                   |

```ts
// Alarm on failed deliveries
new cloudwatch.Alarm(this, "SNSFailures", {
  metric: topic.metricNumberOfNotificationsFailed(),
  threshold: 10,
  evaluationPeriods: 1,
  alarmDescription: "More than 10 SNS delivery failures in 5 minutes",
});
```

---

### 15. **How do you implement a real-world event-driven architecture with SNS?**

**Answer:** A common pattern for an e-commerce platform:

```
Order Service
    │
    ▼
SNS: order-events (topic)
    │
    ├── [filter: order.created] → SQS → Payment Service
    ├── [filter: order.paid]    → SQS → Fulfillment Service
    ├── [filter: order.paid]    → SQS → Inventory Service
    ├── [filter: order.*]       → SQS → Analytics Pipeline
    └── [filter: order.shipped] → Lambda → Email Notification
```

```ts
// Publisher: Order Service
async function publishOrderEvent(order: Order, eventType: string) {
  await sns.send(
    new PublishCommand({
      TopicArn: process.env.ORDER_TOPIC_ARN,
      Message: JSON.stringify({
        eventType,
        orderId: order.id,
        customerId: order.customerId,
        amount: order.total,
        items: order.items,
        timestamp: new Date().toISOString(),
      }),
      MessageAttributes: {
        eventType: { DataType: "String", StringValue: eventType },
        customerId: { DataType: "String", StringValue: order.customerId },
      },
    }),
  );
}

// Usage
await publishOrderEvent(order, "order.created");
// After payment confirmation:
await publishOrderEvent(order, "order.paid");
```

> **Key benefit:** The Order Service doesn't know or care about its consumers. New subscribers (e.g., a loyalty points service) can be added without changing the publisher.

---

## Practical System Design Supplement

---

### 16. **What are SNS's cost considerations?**

**Answer:**

| Component | Cost (us-east-1) |
| --- | --- |
| Publish requests | $0.50 per million (first 1M free) |
| HTTP/S deliveries | $0.60 per million |
| SQS deliveries | Free |
| Lambda deliveries | Free (Lambda invocation cost applies) |
| Email deliveries | $2.00 per 100,000 |
| SMS deliveries | $0.00645 per msg (US) — varies by country |
| Mobile push (APNs/FCM) | $0.50 per million |
| Data transfer | Standard AWS data transfer rates |
| FIFO topics | $0.50 per million publishes |

**Cost optimization tips:**

| Strategy | Savings |
| --- | --- |
| Use filter policies | Avoid unnecessary Lambda/HTTP invocations |
| Batch publish (up to 10 messages) | Reduce API calls |
| Use SQS subscriptions (free delivery) | Cheapest target type |
| Large payload via S3 reference | Stay under 256KB limit, avoid oversized charges |
| Use SNS FIFO only when ordering needed | Standard is simpler and cheaper |

> **Production Tip:** SNS filtering is processed broker-side for free. Moving filter logic from Lambda to SNS subscription filter policies saves both Lambda invocation cost and latency.

---

### 17. **What hidden/lesser-known SNS features are useful?**

**Answer:**

| Feature | What It Does | Why It Matters |
| --- | --- | --- |
| **Message filtering (advanced)** | Filter on nested attributes, prefix, numeric range | Precise subscriber targeting |
| **FIFO topics** | Ordering + deduplication (with SQS FIFO) | Financial, transactional use cases |
| **Delivery status logging** | Log success/failure per endpoint | Debug delivery issues |
| **Raw message delivery** | Skip SNS envelope wrapper for SQS/HTTP | Cleaner payloads, smaller messages |
| **Cross-region subscriptions** | Subscriber in different region | Multi-region architectures |
| **Message archiving** | SNS → Kinesis Firehose → S3 | Audit trail without custom code |
| **Redrive policy** | DLQ for failed HTTP/Lambda deliveries | Don't lose messages on transient failures |
| **Subscription filter policy scope** | Filter on `MessageBody` (not just attributes) | Filter on payload content directly |

```ts
// Message filtering on body (not just attributes)
new sns.Subscription(this, "HighValueOrders", {
  topic: orderTopic,
  protocol: sns.SubscriptionProtocol.SQS,
  endpoint: highValueQueue.queueArn,
  filterPolicyWithMessageBody: {
    amount: sns.FilterOrPolicy.filter(
      sns.SubscriptionFilter.numericFilter({ greaterThan: 1000 }),
    ),
    region: sns.FilterOrPolicy.filter(
      sns.SubscriptionFilter.stringFilter({ allowlist: ["us-east-1", "eu-west-1"] }),
    ),
  },
});
```

---

### 18. **What are SNS's resiliency patterns?**

**Answer:**

| Concern | Solution |
| --- | --- |
| Target temporarily down | Built-in retry with backoff (HTTP: up to 23 days) |
| Lambda invocation failure | DLQ on subscription |
| Message loss | SNS is multi-AZ durable; add DLQ for delivery failures |
| Fan-out ordering | Use FIFO topic + FIFO SQS subscriptions |
| Cross-region failure | Publish to topics in multiple regions |
| Subscriber overwhelmed | Buffer with SQS subscription |

> **Production Tip:** Always set a DLQ (redrive policy) on HTTP/Lambda subscriptions. Without it, messages that fail all retries are silently dropped.
