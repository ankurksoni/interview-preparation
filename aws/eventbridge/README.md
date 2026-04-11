# Amazon EventBridge Interview Questions & Answers

A curated list of **Amazon EventBridge interview questions** with practical, real-world answers. Covers event-driven architecture, event buses, rules, scheduling, schema registry, and integration patterns used in production systems.

---

### 1. **What is Amazon EventBridge?**

**Answer:** Amazon EventBridge is a serverless event bus service that enables event-driven architectures. It routes events from AWS services, SaaS applications, and custom sources to targets like Lambda, SQS, Step Functions, and more. It is the successor to CloudWatch Events.

---

### 2. **What is the difference between EventBridge and SNS?**

**Answer:**

| Feature    | EventBridge                                        | SNS                              |
| ---------- | -------------------------------------------------- | -------------------------------- |
| Filtering  | Content-based (JSON pattern matching on any field) | Message attribute filtering only |
| Schema     | Schema registry & discovery                        | No schema support                |
| Sources    | AWS, SaaS, custom apps                             | AWS, custom publishers           |
| Targets    | 20+ AWS service targets                            | Lambda, SQS, HTTP, Email, SMS    |
| Throughput | Lower default quota (adjustable)                   | Very high throughput             |
| Replay     | Event archive & replay                             | No replay                        |

Use EventBridge for complex routing/filtering. Use SNS for high-throughput fan-out.

---

### 3. **What is an Event Bus in EventBridge?**

**Answer:** An event bus receives events and routes them to rules. There are three types:

- **Default event bus** — receives events from AWS services automatically
- **Custom event bus** — for your application events
- **Partner event bus** — for SaaS provider events (e.g., Shopify, Zendesk, Auth0)

```ts
import {
  EventBridgeClient,
  PutEventsCommand,
} from "@aws-sdk/client-eventbridge";

const client = new EventBridgeClient({});

await client.send(
  new PutEventsCommand({
    Entries: [
      {
        Source: "com.myapp.orders",
        DetailType: "OrderPlaced",
        Detail: JSON.stringify({
          orderId: "ORD-123",
          userId: "USR-456",
          amount: 99.99,
        }),
        EventBusName: "my-custom-bus",
      },
    ],
  }),
);
```

---

### 4. **How do EventBridge rules work?**

**Answer:** Rules match incoming events using event patterns and route them to one or more targets. A rule can have up to 5 targets.

```json
{
  "source": ["com.myapp.orders"],
  "detail-type": ["OrderPlaced"],
  "detail": {
    "amount": [{ "numeric": [">", 100] }]
  }
}
```

This rule matches only orders with amount > 100.

---

### 5. **What event pattern matching capabilities does EventBridge support?**

**Answer:** EventBridge supports:

- **Exact match**: `"status": ["COMPLETED"]`
- **Prefix match**: `"sku": [{"prefix": "PROD-"}]`
- **Suffix match**: `"email": [{"suffix": "@company.com"}]`
- **Numeric comparison**: `"amount": [{"numeric": [">", 0, "<=", 1000]}]`
- **Exists check**: `"error": [{"exists": true}]`
- **Anything-but**: `"status": [{"anything-but": ["CANCELLED"]}]`
- **Wildcard**: `"path": [{"wildcard": "orders/*/items"}]`

```json
{
  "source": ["com.myapp.payments"],
  "detail": {
    "status": [{ "anything-but": ["PENDING"] }],
    "amount": [{ "numeric": [">=", 50] }],
    "currency": ["USD", "EUR"]
  }
}
```

---

### 6. **How do you implement event replay for debugging production issues?**

**Answer:** EventBridge Archives store events for later replay. This is critical for debugging and reprocessing failed events.

```ts
// Create an archive
import {
  EventBridgeClient,
  CreateArchiveCommand,
} from "@aws-sdk/client-eventbridge";

const client = new EventBridgeClient({});

await client.send(
  new CreateArchiveCommand({
    ArchiveName: "order-events-archive",
    EventSourceArn:
      "arn:aws:events:us-east-1:123456789:event-bus/my-custom-bus",
    EventPattern: JSON.stringify({
      source: ["com.myapp.orders"],
    }),
    RetentionDays: 30,
  }),
);

// Replay events from a time window
import { StartReplayCommand } from "@aws-sdk/client-eventbridge";

await client.send(
  new StartReplayCommand({
    ReplayName: "debug-replay-2026-04",
    EventSourceArn:
      "arn:aws:events:us-east-1:123456789:event-bus/my-custom-bus/archive/order-events-archive",
    Destination: {
      Arn: "arn:aws:events:us-east-1:123456789:event-bus/my-custom-bus",
    },
    EventStartTime: new Date("2026-04-01"),
    EventEndTime: new Date("2026-04-02"),
  }),
);
```

---

### 7. **How does EventBridge Scheduler differ from EventBridge Rules with schedules?**

**Answer:** EventBridge Scheduler is a dedicated scheduling service, separate from rule-based scheduling.

| Feature           | EventBridge Rules (Schedule) | EventBridge Scheduler            |
| ----------------- | ---------------------------- | -------------------------------- |
| One-time schedule | ❌ Not supported             | ✅ Supported                     |
| Time zones        | ❌ UTC only                  | ✅ Any time zone                 |
| Flexible windows  | ❌ No                        | ✅ Yes (spread invocations)      |
| Scale             | 300 rules per bus            | Millions of schedules            |
| Retry policy      | Limited                      | Configurable (up to 185 retries) |

```ts
import {
  SchedulerClient,
  CreateScheduleCommand,
} from "@aws-sdk/client-scheduler";

const scheduler = new SchedulerClient({});

// One-time schedule: send reminder email in 24 hours
await scheduler.send(
  new CreateScheduleCommand({
    Name: "order-reminder-ORD-123",
    ScheduleExpression: "at(2026-04-09T14:00:00)",
    ScheduleExpressionTimezone: "Asia/Kolkata",
    FlexibleTimeWindow: { Mode: "OFF" },
    Target: {
      Arn: "arn:aws:lambda:us-east-1:123456789:function:sendReminder",
      RoleArn: "arn:aws:iam::123456789:role/scheduler-role",
      Input: JSON.stringify({ orderId: "ORD-123" }),
    },
  }),
);
```

---

### 8. **How do you handle cross-account event routing?**

**Answer:** EventBridge supports sending events across AWS accounts using resource-based policies on the target event bus.

**Account A (sender):**

```ts
await client.send(
  new PutEventsCommand({
    Entries: [
      {
        Source: "com.myapp.orders",
        DetailType: "OrderShipped",
        Detail: JSON.stringify({ orderId: "ORD-789" }),
        EventBusName:
          "arn:aws:events:us-east-1:ACCOUNT_B_ID:event-bus/shared-bus",
      },
    ],
  }),
);
```

**Account B (receiver) — resource policy:**

```json
{
  "Statement": [
    {
      "Sid": "AllowAccountAPutEvents",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::ACCOUNT_A_ID:root" },
      "Action": "events:PutEvents",
      "Resource": "arn:aws:events:us-east-1:ACCOUNT_B_ID:event-bus/shared-bus"
    }
  ]
}
```

---

### 9. **How do you handle idempotency with EventBridge?**

**Answer:** EventBridge guarantees at-least-once delivery, so consumers must be idempotent. Common strategies:

```ts
// ❌ Bad: No idempotency — duplicate events cause duplicate charges
async function handlePayment(event: any) {
  await chargeCustomer(event.detail.orderId, event.detail.amount);
}

// ✅ Good: Use idempotency key with conditional write
async function handlePayment(event: any) {
  const { orderId, amount } = event.detail;

  try {
    await dynamodb.send(
      new PutCommand({
        TableName: "Payments",
        Item: {
          orderId,
          amount,
          processedAt: new Date().toISOString(),
        },
        ConditionExpression: "attribute_not_exists(orderId)",
      }),
    );

    await chargeCustomer(orderId, amount);
  } catch (err: any) {
    if (err.name === "ConditionalCheckFailedException") {
      console.log(`Payment for ${orderId} already processed. Skipping.`);
      return;
    }
    throw err;
  }
}
```

---

### 10. **What is the EventBridge Schema Registry?**

**Answer:** The Schema Registry automatically discovers and stores event schemas from your event bus. You can generate code bindings for TypeScript, Python, and Java to ensure type-safe event handling.

```ts
// Auto-generated type from Schema Registry
interface OrderPlacedEvent {
  version: string;
  id: string;
  source: string;
  "detail-type": string;
  detail: {
    orderId: string;
    userId: string;
    amount: number;
    items: Array<{ sku: string; quantity: number }>;
  };
}

// Type-safe handler
export const handler = async (event: OrderPlacedEvent) => {
  const { orderId, amount } = event.detail;
  // TypeScript catches any typos or missing fields at compile time
};
```

---

### 11. **How do you implement the Saga pattern using EventBridge?**

**Answer:** EventBridge is ideal for orchestrating distributed transactions across microservices using the choreography-based Saga pattern.

```
OrderService → [OrderPlaced] → EventBridge
    → PaymentService → [PaymentSucceeded] → EventBridge
        → InventoryService → [InventoryReserved] → EventBridge
            → ShippingService → [ShipmentCreated]

    → PaymentService → [PaymentFailed] → EventBridge
        → OrderService → [OrderCancelled] (compensating action)
```

```ts
// Payment service listens for OrderPlaced, emits PaymentSucceeded or PaymentFailed
export const handler = async (event: any) => {
  const { orderId, amount, userId } = event.detail;

  try {
    await processPayment(userId, amount);

    await eventbridge.send(
      new PutEventsCommand({
        Entries: [
          {
            Source: "com.myapp.payments",
            DetailType: "PaymentSucceeded",
            Detail: JSON.stringify({ orderId, transactionId: "TXN-001" }),
            EventBusName: "order-bus",
          },
        ],
      }),
    );
  } catch (err) {
    await eventbridge.send(
      new PutEventsCommand({
        Entries: [
          {
            Source: "com.myapp.payments",
            DetailType: "PaymentFailed",
            Detail: JSON.stringify({ orderId, reason: (err as Error).message }),
            EventBusName: "order-bus",
          },
        ],
      }),
    );
  }
};
```

---

### 12. **What is the maximum event size in EventBridge?**

**Answer:** The maximum event size is **256 KB**. For larger payloads, store the data in S3 and pass the S3 reference in the event.

```ts
// ❌ Bad: Putting large payload directly in the event
await eventbridge.send(
  new PutEventsCommand({
    Entries: [
      {
        Detail: JSON.stringify({ data: hugePayload }), // May exceed 256 KB
      },
    ],
  }),
);

// ✅ Good: Store in S3, reference in event
const key = `events/${orderId}.json`;
await s3.send(
  new PutObjectCommand({
    Bucket: "my-events-bucket",
    Key: key,
    Body: JSON.stringify(hugePayload),
  }),
);

await eventbridge.send(
  new PutEventsCommand({
    Entries: [
      {
        Source: "com.myapp.reports",
        DetailType: "ReportGenerated",
        Detail: JSON.stringify({
          orderId,
          payloadRef: { bucket: "my-events-bucket", key },
        }),
        EventBusName: "my-bus",
      },
    ],
  }),
);
```

---

### 13. **How do you test EventBridge rules locally?**

**Answer:** Use the `TestEventPattern` API to validate that your patterns match expected events without deploying.

```ts
import {
  EventBridgeClient,
  TestEventPatternCommand,
} from "@aws-sdk/client-eventbridge";

const client = new EventBridgeClient({});

const result = await client.send(
  new TestEventPatternCommand({
    EventPattern: JSON.stringify({
      source: ["com.myapp.orders"],
      detail: {
        amount: [{ numeric: [">", 100] }],
      },
    }),
    Event: JSON.stringify({
      source: "com.myapp.orders",
      "detail-type": "OrderPlaced",
      detail: { orderId: "ORD-1", amount: 150 },
    }),
  }),
);

console.log(result.Result); // true — pattern matches the event
```

---

### 14. **How do you implement dead-letter queues with EventBridge?**

**Answer:** Configure a DLQ (SQS queue) on a rule target. If EventBridge cannot deliver an event to the target after retries, it sends the event to the DLQ.

```ts
// CDK example
const rule = new events.Rule(this, "OrderRule", {
  eventBus: customBus,
  eventPattern: {
    source: ["com.myapp.orders"],
    detailType: ["OrderPlaced"],
  },
});

const dlq = new sqs.Queue(this, "OrderDLQ", {
  retentionPeriod: cdk.Duration.days(14),
});

rule.addTarget(
  new targets.LambdaFunction(orderHandler, {
    deadLetterQueue: dlq,
    retryAttempts: 3,
    maxEventAge: cdk.Duration.hours(1),
  }),
);
```

---

### 15. **What are EventBridge Pipes and when should you use them?**

**Answer:** EventBridge Pipes provide point-to-point integrations between a source and a target, with optional filtering, enrichment, and transformation — without writing custom Lambda glue code.

**Sources:** SQS, Kinesis, DynamoDB Streams, Kafka, MQ
**Targets:** Lambda, Step Functions, SQS, SNS, EventBridge, API Gateway, and more

```
SQS Queue → [Filter] → [Enrich via Lambda] → [Transform] → Step Functions
```

Use Pipes when you need a direct source-to-target flow. Use event bus rules when you need fan-out to multiple targets.

---

### 16. **How do you batch events to reduce Lambda invocations?**

**Answer:** Use SQS as a buffer between EventBridge and Lambda. EventBridge sends events to an SQS queue, and Lambda polls the queue with batch settings.

```ts
// CDK: EventBridge → SQS → Lambda (batched)
rule.addTarget(new targets.SqsQueue(queue));

new lambda.EventSourceMapping(this, "BatchProcessor", {
  target: processorFn,
  eventSourceArn: queue.queueArn,
  batchSize: 10,
  maxBatchingWindow: cdk.Duration.seconds(30),
});
```

---

### 17. **What are the default quotas for EventBridge?**

**Answer:**

| Quota                   | Default Limit                      |
| ----------------------- | ---------------------------------- |
| PutEvents throughput    | 10,000 entries/second (soft limit) |
| Rules per event bus     | 300 (can request increase)         |
| Targets per rule        | 5                                  |
| Event size              | 256 KB                             |
| Event buses per account | 100                                |
| Archives per account    | 20                                 |

---

### 18. **How do you handle event ordering in EventBridge?**

**Answer:** EventBridge does **not** guarantee ordering. If your application needs ordered processing:

1. Include a sequence number or timestamp in the event detail
2. Use SQS FIFO as the target (with MessageGroupId for per-entity ordering)
3. Handle out-of-order events in consumers using version checks

```ts
// ❌ Bad: Assuming events arrive in order
async function handleStatusChange(event: any) {
  await updateOrderStatus(event.detail.orderId, event.detail.status);
}

// ✅ Good: Check version before applying
async function handleStatusChange(event: any) {
  const { orderId, status, version } = event.detail;

  await dynamodb.send(
    new UpdateCommand({
      TableName: "Orders",
      Key: { orderId },
      UpdateExpression: "SET #s = :status, #v = :newVersion",
      ConditionExpression: "#v < :newVersion",
      ExpressionAttributeNames: { "#s": "status", "#v": "version" },
      ExpressionAttributeValues: { ":status": status, ":newVersion": version },
    }),
  );
}
```

---

### 19. **How do you monitor EventBridge in production?**

**Answer:** Key CloudWatch metrics to monitor:

- `FailedInvocations` — target invocation failures
- `ThrottledRules` — rules being throttled
- `MatchedEvents` — events that matched a rule
- `DeadLetterInvocations` — events sent to DLQ
- `IngestionToInvocationStartLatency` — time from event arrival to target invocation

Set alarms on `FailedInvocations` and `DeadLetterInvocations` for production systems.

---

### 20. **How do you implement event versioning for backward compatibility?**

**Answer:** Include a version field in your events and handle multiple versions in consumers.

```ts
// Producer: always include version
await eventbridge.send(
  new PutEventsCommand({
    Entries: [
      {
        Source: "com.myapp.orders",
        DetailType: "OrderPlaced",
        Detail: JSON.stringify({
          version: "2.0",
          orderId: "ORD-123",
          userId: "USR-456",
          amount: { value: 99.99, currency: "USD" }, // v2: structured amount
        }),
      },
    ],
  }),
);

// Consumer: handle both v1 and v2
export const handler = async (event: any) => {
  const detail = event.detail;

  const amount = detail.version === "2.0" ? detail.amount.value : detail.amount; // v1 was a plain number

  await processOrder(detail.orderId, amount);
};
```

---

## Practical System Design Supplement

---

### 21. **What are EventBridge's cost considerations?**

**Answer:**

| Component                        | Cost (us-east-1)                |
| -------------------------------- | ------------------------------- |
| Custom events published          | $1.00 per million events        |
| Default bus (AWS service events) | Free                            |
| Third-party events (SaaS)        | $1.00 per million               |
| Archive & replay                 | $0.023 per GB stored            |
| Schema discovery                 | Free                            |
| Pipes (filtering, enrichment)    | $0.40 per million invocations   |
| Scheduler                        | Free (first 14M), $1.00/M after |
| Cross-region event bus           | $1.00/M + data transfer         |

**Cost optimization tips:**

| Strategy                                            | Savings                                    |
| --------------------------------------------------- | ------------------------------------------ |
| Use filter patterns on rules                        | Reduce target invocations (free filtering) |
| Batch events into fewer PutEvents calls             | Up to 10 entries/call                      |
| Use Scheduler instead of CloudWatch Events for cron | Free tier is generous                      |
| Avoid archiving all events                          | Archive only what you'd replay             |
| Use default bus for AWS events                      | Free vs custom bus                         |

> **Production Tip:** EventBridge filtering is free — filter at the rule level instead of invoking Lambda to filter. Every Lambda invocation you avoid saves money.

---

### 22. **What are EventBridge Pipes and when to use them?**

**Answer:** Pipes connect sources to targets with optional filtering, enrichment, and transformation — without writing Lambda glue code.

```
Source (SQS/Kinesis/DynamoDB) → Filter → Enrich (Lambda/API GW/Step Functions) → Target
```

```ts
import * as pipes from "aws-cdk-lib/aws-pipes";

new pipes.CfnPipe(this, "OrderPipe", {
  name: "order-processing-pipe",
  source: queue.queueArn,
  sourceParameters: {
    sqsQueueParameters: { batchSize: 10 },
    filterCriteria: {
      filters: [{ pattern: '{"body":{"eventType":["ORDER_CREATED"]}}' }],
    },
  },
  enrichment: enrichmentFn.functionArn,
  target: stepFunction.stateMachineArn,
  targetParameters: {
    stepFunctionStateMachineParameters: {
      invocationType: "FIRE_AND_FORGET",
    },
  },
  roleArn: pipeRole.roleArn,
});
```

| Use Pipes When                        | Use Rules + Targets When                      |
| ------------------------------------- | --------------------------------------------- |
| Source is SQS/Kinesis/DynamoDB Stream | Source is custom events or AWS service events |
| Need point-to-point processing        | Need fan-out to multiple targets              |
| Want to eliminate glue Lambda         | Complex routing logic needed                  |
| Enrichment step is simple             | Multiple rules with different patterns        |

---

### 23. **What hidden/lesser-known EventBridge features matter in production?**

**Answer:**

| Feature                     | What It Does                                  | Why It Matters                      |
| --------------------------- | --------------------------------------------- | ----------------------------------- |
| **Content-based filtering** | Match on nested JSON fields, arrays, prefixes | Reduce target invocations by 90%+   |
| **Input transformers**      | Reshape event before delivery                 | Send only needed fields to target   |
| **Archive & replay**        | Store events and replay to bus                | Disaster recovery, reprocessing     |
| **Global endpoints**        | Active-active multi-region                    | Automatic failover in <1min         |
| **Schema registry**         | Auto-discover event schemas                   | Code generation for type safety     |
| **DLQ per rule**            | Failed deliveries go to SQS                   | Don't lose events on target failure |

> **Hidden Gem:** **Global endpoints** with health checks enable automatic failover between regions. Events are replicated and your producers don't need to change endpoints — EventBridge handles routing.
