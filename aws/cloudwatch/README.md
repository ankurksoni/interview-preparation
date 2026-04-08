# Amazon CloudWatch Interview Questions & Answers

A curated list of **Amazon CloudWatch interview questions** with practical, production-grade answers. Covers metrics, alarms, logs, dashboards, anomaly detection, Container Insights, and real-world observability patterns.

---

### 1. **What is Amazon CloudWatch?**

**Answer:** CloudWatch is AWS's monitoring and observability service. It collects metrics, logs, and traces from AWS resources and applications, enabling alarms, dashboards, automated actions, and anomaly detection. It is the central pillar of AWS observability.

---

### 2. **What is the difference between CloudWatch Metrics, Logs, and Traces?**

**Answer:**

| Component              | Purpose                      | Example                                 |
| ---------------------- | ---------------------------- | --------------------------------------- |
| **Metrics**            | Numeric time-series data     | CPU utilization, request count          |
| **Logs**               | Text-based event records     | Application logs, Lambda output         |
| **Traces** (via X-Ray) | Request flow across services | API Gateway → Lambda → DynamoDB latency |

---

### 3. **What are CloudWatch namespaces, dimensions, and metric resolution?**

**Answer:**

- **Namespace** — Logical grouping (e.g., `AWS/EC2`, `MyApp/Payments`)
- **Dimensions** — Key-value pairs to filter metrics (e.g., `InstanceId`, `FunctionName`)
- **Resolution** — Standard metrics: 1-minute intervals (60s). High-resolution: up to 1-second intervals.

```ts
import {
  CloudWatchClient,
  PutMetricDataCommand,
} from "@aws-sdk/client-cloudwatch";

const cw = new CloudWatchClient({});

await cw.send(
  new PutMetricDataCommand({
    Namespace: "MyApp/Payments",
    MetricData: [
      {
        MetricName: "PaymentProcessed",
        Value: 1,
        Unit: "Count",
        Dimensions: [
          { Name: "Environment", Value: "production" },
          { Name: "PaymentMethod", Value: "credit_card" },
        ],
        StorageResolution: 1, // High-resolution (1 second)
      },
    ],
  }),
);
```

---

### 4. **How do you publish custom metrics from your application?**

**Answer:**

```ts
// ❌ Bad: Publishing one metric per API call (expensive, slow)
for (const order of orders) {
  await cw.send(
    new PutMetricDataCommand({
      Namespace: "MyApp/Orders",
      MetricData: [
        { MetricName: "OrderValue", Value: order.amount, Unit: "None" },
      ],
    }),
  );
}

// ✅ Good: Batch metrics (up to 1000 metric data points per call)
const metricData = orders.map((order) => ({
  MetricName: "OrderValue",
  Value: order.amount,
  Unit: "None",
  Timestamp: new Date(),
  Dimensions: [{ Name: "Region", Value: order.region }],
}));

// PutMetricData supports max 1000 values per call
for (let i = 0; i < metricData.length; i += 1000) {
  await cw.send(
    new PutMetricDataCommand({
      Namespace: "MyApp/Orders",
      MetricData: metricData.slice(i, i + 1000),
    }),
  );
}
```

---

### 5. **How do CloudWatch Alarms work?**

**Answer:** Alarms monitor a metric and transition between three states: `OK`, `ALARM`, and `INSUFFICIENT_DATA`. They can trigger actions like SNS notifications, Auto Scaling, or Lambda.

```ts
import {
  CloudWatchClient,
  PutMetricAlarmCommand,
} from "@aws-sdk/client-cloudwatch";

await cw.send(
  new PutMetricAlarmCommand({
    AlarmName: "HighErrorRate-PaymentService",
    Namespace: "MyApp/Payments",
    MetricName: "ErrorCount",
    Statistic: "Sum",
    Period: 300, // 5 minutes
    EvaluationPeriods: 2, // Must breach for 2 consecutive periods
    Threshold: 10,
    ComparisonOperator: "GreaterThanThreshold",
    TreatMissingData: "notBreaching",
    AlarmActions: [
      "arn:aws:sns:us-east-1:123456789:ops-alerts", // Notify on-call team
    ],
    OKActions: [
      "arn:aws:sns:us-east-1:123456789:ops-resolved", // Notify when resolved
    ],
  }),
);
```

---

### 6. **What is the difference between Standard and High-Resolution metrics?**

**Answer:**

| Feature          | Standard                  | High-Resolution                     |
| ---------------- | ------------------------- | ----------------------------------- |
| Resolution       | 60 seconds                | 1 second                            |
| Cost             | Included for AWS services | Higher cost for custom metrics      |
| Alarm evaluation | Minimum 1 minute          | Minimum 10 seconds                  |
| Use case         | General monitoring        | Real-time alerting, trading systems |

High-resolution metrics are set by specifying `StorageResolution: 1` in `PutMetricData`.

---

### 7. **How do you use CloudWatch Logs Insights for querying logs?**

**Answer:** Logs Insights is a query language for searching and analyzing log data.

```
# Find the top 10 most expensive Lambda invocations
filter @type = "REPORT"
| stats max(@duration) as maxDuration,
        avg(@duration) as avgDuration,
        max(@memorySize / 1000 / 1000) as memoryMB,
        max(@maxMemoryUsed / 1000 / 1000) as maxMemoryUsedMB
  by @logStream
| sort maxDuration desc
| limit 10
```

```
# Find all errors with context
fields @timestamp, @message
| filter @message like /ERROR|Exception/
| sort @timestamp desc
| limit 50
```

```
# Aggregate API response times by endpoint
filter @message like /API response/
| parse @message "* * * *ms" as method, endpoint, status, duration
| stats avg(duration) as avgMs, p99(duration) as p99Ms, count() as total
  by endpoint
| sort p99Ms desc
```

---

### 8. **How do you create CloudWatch metric filters from log data?**

**Answer:** Metric filters extract numeric values from log entries and publish them as CloudWatch metrics.

```ts
import {
  CloudWatchLogsClient,
  PutMetricFilterCommand,
} from "@aws-sdk/client-cloudwatch-logs";

const logs = new CloudWatchLogsClient({});

// Create a metric from JSON logs: count payment failures
await logs.send(
  new PutMetricFilterCommand({
    LogGroupName: "/aws/lambda/payment-handler",
    FilterName: "PaymentFailures",
    FilterPattern: '{ $.status = "FAILED" && $.service = "payment" }',
    MetricTransformations: [
      {
        MetricNamespace: "MyApp/Payments",
        MetricName: "PaymentFailureCount",
        MetricValue: "1",
        DefaultValue: 0,
      },
    ],
  }),
);
```

Now you can create alarms on `PaymentFailureCount` just like any other metric.

---

### 9. **What is CloudWatch Anomaly Detection and when should you use it?**

**Answer:** Anomaly Detection uses machine learning to automatically determine the expected range for a metric. Instead of setting static thresholds, the alarm triggers when the metric deviates from its learned pattern.

```ts
await cw.send(
  new PutMetricAlarmCommand({
    AlarmName: "AnomalousLatency-API",
    MetricName: "Latency",
    Namespace: "AWS/ApiGateway",
    Dimensions: [{ Name: "ApiName", Value: "my-api" }],
    ComparisonOperator: "GreaterThanUpperThreshold",
    ThresholdMetricId: "ad1",
    EvaluationPeriods: 3,
    Metrics: [
      {
        Id: "m1",
        MetricStat: {
          Metric: {
            MetricName: "Latency",
            Namespace: "AWS/ApiGateway",
            Dimensions: [{ Name: "ApiName", Value: "my-api" }],
          },
          Period: 300,
          Stat: "p99",
        },
      },
      {
        Id: "ad1",
        Expression: "ANOMALY_DETECTION_BAND(m1, 2)", // 2 standard deviations
      },
    ],
  }),
);
```

Best for: metrics with seasonal patterns (e.g., traffic that peaks during business hours).

---

### 10. **What are Composite Alarms?**

**Answer:** Composite alarms combine multiple alarms using AND/OR logic to reduce alert noise.

```ts
import { PutCompositeAlarmCommand } from "@aws-sdk/client-cloudwatch";

// Only alert if BOTH high error rate AND high latency
await cw.send(
  new PutCompositeAlarmCommand({
    AlarmName: "CriticalPaymentIssue",
    AlarmRule:
      'ALARM("HighErrorRate-PaymentService") AND ALARM("HighLatency-PaymentService")',
    AlarmActions: ["arn:aws:sns:us-east-1:123456789:pager-duty-critical"],
  }),
);
```

This prevents false positives where a transient spike in one metric triggers unnecessary pages.

---

### 11. **How do you implement structured logging for CloudWatch?**

**Answer:**

```ts
// ❌ Bad: Unstructured logs — hard to query
console.log("Payment failed for user 123, order ORD-456, amount 99.99");

// ✅ Good: Structured JSON logs — queryable with Logs Insights
console.log(
  JSON.stringify({
    level: "ERROR",
    message: "Payment failed",
    userId: "123",
    orderId: "ORD-456",
    amount: 99.99,
    errorCode: "CARD_DECLINED",
    timestamp: new Date().toISOString(),
  }),
);
```

Query structured logs:

```
fields @timestamp, orderId, errorCode, amount
| filter level = "ERROR" and errorCode = "CARD_DECLINED"
| stats count() as failures, sum(amount) as lostRevenue by bin(1h)
```

---

### 12. **What is CloudWatch Embedded Metric Format (EMF)?**

**Answer:** EMF lets you embed custom metrics directly in log output, avoiding separate `PutMetricData` API calls. The metrics are extracted automatically by CloudWatch.

```ts
import { createMetricsLogger, Unit } from "aws-embedded-metrics";

export const handler = async (event: any) => {
  const metrics = createMetricsLogger();

  metrics.setNamespace("MyApp/Orders");
  metrics.putDimensions({ Environment: "production", Region: "us-east-1" });
  metrics.putMetric("OrderProcessingTime", 245, Unit.Milliseconds);
  metrics.putMetric("OrderValue", 99.99, Unit.None);
  metrics.setProperty("orderId", "ORD-123"); // Searchable but not a metric

  await metrics.flush();
};
```

Benefits: no extra API calls, lower cost, metrics + logs in one place.

---

### 13. **How do you set up cross-account CloudWatch monitoring?**

**Answer:** Use **CloudWatch cross-account observability** to aggregate metrics, logs, and traces from multiple accounts into a central monitoring account.

1. **Monitoring account** — creates a sink (the central hub)
2. **Source accounts** — create links to the monitoring account's sink

This is configured via CloudWatch → Settings → Cross-account observability in the console, or via `oam:CreateSink` and `oam:CreateLink` APIs.

---

### 14. **What are CloudWatch Dashboards best practices?**

**Answer:**

- **Use automatic dashboards** — AWS auto-creates resource-specific dashboards
- **Design for incidents** — Top row: key alarms. Middle: key metrics. Bottom: supporting data
- **Use dashboard variables** — Parameterize by environment, region, service
- **Set appropriate time ranges** — 1h for incident response, 1w for trends
- **Include metric math** — Calculate error rates: `100 * m1 / m2` (errors / total)

```ts
// Metric math: Error rate percentage
{
  Id: 'errorRate',
  Expression: '100 * errors / requests',
  Label: 'Error Rate %',
}
```

---

### 15. **How do CloudWatch alarms integrate with Auto Scaling?**

**Answer:** CloudWatch alarms trigger Auto Scaling policies to add/remove EC2 instances based on metrics.

```ts
// CDK: Scale out when CPU > 70% for 2 consecutive periods
const cpuAlarm = new cloudwatch.Alarm(this, "HighCPU", {
  metric: asg.metricCPUUtilization(),
  threshold: 70,
  evaluationPeriods: 2,
  comparisonOperator: cloudwatch.ComparisonOperator.GREATER_THAN_THRESHOLD,
});

asg.scaleOnMetric("ScaleOnCPU", {
  metric: asg.metricCPUUtilization(),
  scalingSteps: [
    { upper: 50, change: -1 }, // Scale in below 50%
    { lower: 70, change: +1 }, // Scale out above 70%
    { lower: 90, change: +3 }, // Scale out aggressively above 90%
  ],
});
```

---

### 16. **What is the default retention for CloudWatch Logs?**

**Answer:** Logs are retained **indefinitely** by default (never expire). This can be very costly. Always set a retention policy.

```ts
// ❌ Bad: Default — logs stored forever, increasing costs
new logs.LogGroup(this, "AppLogs");

// ✅ Good: Set explicit retention
new logs.LogGroup(this, "AppLogs", {
  logGroupName: "/myapp/api",
  retention: logs.RetentionDays.THIRTY_DAYS, // or 14, 90, 365
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

---

### 17. **How do you monitor Lambda functions effectively with CloudWatch?**

**Answer:** Key Lambda metrics to monitor:

| Metric                 | What it tells you                        |
| ---------------------- | ---------------------------------------- |
| `Invocations`          | How often the function runs              |
| `Errors`               | Unhandled exceptions                     |
| `Duration`             | Execution time (watch p99)               |
| `Throttles`            | Concurrency limit hit                    |
| `ConcurrentExecutions` | Current parallel invocations             |
| `IteratorAge`          | Stream processing lag (Kinesis/DynamoDB) |

```
# Logs Insights: Find cold starts
filter @type = "REPORT"
| fields @duration, @initDuration, @memorySize, @maxMemoryUsed
| filter ispresent(@initDuration)
| stats count() as coldStarts, avg(@initDuration) as avgColdStart
  by bin(5m)
```

---

### 18. **What is CloudWatch Contributor Insights?**

**Answer:** Contributor Insights analyzes log data to identify the top contributors to a metric. Useful for finding top talkers, noisy tenants, or high-error endpoints.

```ts
// Find top 10 IP addresses causing 5xx errors
const rule = new cloudwatch.CfnInsightRule(this, "TopErrorIPs", {
  ruleName: "Top5xxIPs",
  ruleState: "ENABLED",
  ruleBody: JSON.stringify({
    Schema: { Name: "CloudWatchLogRule", Version: 1 },
    LogGroupNames: ["/myapp/api"],
    LogFormat: "JSON",
    Contribution: {
      Keys: ["$.sourceIp"],
      Filters: [{ Match: "$.statusCode", GreaterThan: 499 }],
    },
    AggregateOn: "Count",
  }),
});
```

---

### 19. **How do you reduce CloudWatch costs?**

**Answer:**

1. **Set log retention** — Don't keep logs forever
2. **Use EMF** — Embed metrics in logs instead of separate PutMetricData calls
3. **Filter before ingestion** — Use subscription filters to send only relevant logs
4. **Use metric math** — Derive metrics from existing ones instead of publishing new ones
5. **Reduce dimensions** — Each unique dimension combination creates a separate metric (billable)
6. **Use standard resolution** — Only use high-resolution (1s) when truly needed

---

### 20. **How do you implement alerting on application-level SLOs?**

**Answer:** Use metric math to compute SLO metrics like availability and latency percentiles.

```ts
// Alarm: Availability drops below 99.9% over 1 hour
await cw.send(
  new PutMetricAlarmCommand({
    AlarmName: "SLO-Availability-99.9",
    EvaluationPeriods: 1,
    Metrics: [
      {
        Id: "success",
        MetricStat: {
          Metric: { Namespace: "MyApp/API", MetricName: "SuccessCount" },
          Period: 3600,
          Stat: "Sum",
        },
        ReturnData: false,
      },
      {
        Id: "total",
        MetricStat: {
          Metric: { Namespace: "MyApp/API", MetricName: "RequestCount" },
          Period: 3600,
          Stat: "Sum",
        },
        ReturnData: false,
      },
      {
        Id: "availability",
        Expression: "100 * success / total",
        Label: "Availability %",
      },
    ],
    Threshold: 99.9,
    ComparisonOperator: "LessThanThreshold",
    AlarmActions: ["arn:aws:sns:us-east-1:123456789:slo-breach-alerts"],
  }),
);
```
