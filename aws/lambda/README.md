# AWS Lambda Interview Questions & Answers

A curated list of **AWS Lambda interview questions**, with clear and concise answers. The first 40 questions cover foundational and intermediate topics. The final 10 questions are **advanced and high-difficulty**, designed for expert-level interviews.

---

### 1. **What is AWS Lambda?**

**Answer:** AWS Lambda is a serverless compute service that runs code in response to events and automatically manages the compute resources. It eliminates the need to provision or manage servers.

---

### 2. **What languages does AWS Lambda support?**

**Answer:** AWS Lambda natively supports Node.js (18.x, 20.x, 22.x), Python (3.9–3.13), Java (11, 17, 21), Ruby (3.3, 3.4), .NET (8), and custom runtimes via the AWS Lambda Runtime API (e.g., Rust, Go, PHP). Previously supported runtimes like Node.js 14/16, Python 3.7/3.8, and Go 1.x (managed) have been deprecated.

> **Note:** Go is supported but as a custom runtime (via `provided.al2023`), not as a managed runtime since the deprecation of the `go1.x` runtime in early 2024.

---

### 3. **What is the maximum execution timeout for a Lambda function?**

**Answer:** The maximum timeout for a Lambda function is **15 minutes (900 seconds)**.

---

### 4. **What triggers can invoke a Lambda function?**

**Answer:** Lambda functions can be triggered by various AWS services like API Gateway, S3, DynamoDB, SNS, SQS, EventBridge, CloudWatch Events, CloudFormation, and others.

---

### 5. **How does AWS Lambda scale?**

**Answer:** Lambda scales automatically and concurrently. Each event triggers a new instance of the function, with limits based on region concurrency quotas.

---

### 6. **What is cold start in AWS Lambda?**

**Answer:** A cold start occurs when a new container is initialized to run a Lambda function. This adds latency due to initialization overhead, especially for VPC-attached or large package functions.

---

### 7. **What is the difference between synchronous and asynchronous invocation in Lambda?**

**Answer:**

- **Synchronous**: Caller waits for the result (e.g., API Gateway).
- **Asynchronous**: Lambda queues the event and processes it later (e.g., S3, SNS).

---

### 8. **How can you handle errors in Lambda?**

**Answer:**

- **Synchronous**: Return proper status codes.
- **Asynchronous**: Use Dead Letter Queues (DLQs) or EventBridge retry policies.
- **All**: Wrap code in try-catch and monitor via CloudWatch logs.

---

### 9. **What is a Dead Letter Queue (DLQ) in Lambda?**

**Answer:** DLQ captures failed asynchronous Lambda invocations using SQS or SNS, allowing retry analysis or manual handling of failed events.

---

### 10. **How do environment variables work in Lambda?**

**Answer:** Environment variables are key-value pairs passed to the function code during execution. They are useful for storing config data securely.

---

### 11. **What are Lambda layers?**

**Answer:** Lambda Layers are a distribution mechanism for libraries, custom runtimes, and other dependencies to be shared across functions without bundling them into deployment packages.

---

### 12. **What is the maximum memory you can allocate to a Lambda function?**

**Answer:** Currently, the maximum memory allocation is **10,240 MB (10 GB)**, with proportional CPU power.

---

### 13. **Can Lambda functions run in a VPC?**

**Answer:** Yes. You can configure a Lambda to run inside a VPC to access resources like RDS, but it may introduce cold start latency unless optimized.

---

### 14. **How do you deploy AWS Lambda functions?**

**Answer:** Via AWS Console, AWS CLI, AWS SDK, SAM (Serverless Application Model), CDK, or frameworks like Serverless Framework.

---

### 15. **What are provisioned concurrency and how is it different from standard concurrency?**

**Answer:** Provisioned Concurrency pre-initializes Lambda instances to avoid cold starts. Standard concurrency dynamically scales without pre-initialization.

---

### 16. **How can you monitor Lambda functions?**

**Answer:** AWS provides **CloudWatch Logs**, **CloudWatch Metrics**, and **AWS X-Ray** for tracing performance and debugging.

---

### 17. **What is the deployment package size limit for Lambda?**

**Answer:**

- **Direct upload (zipped)**: 50 MB
- **Unzipped (via S3)**: 250 MB
- **Container image**: Up to 10 GB (Lambda supports container images since 2020)

---

### 18. **How do you secure Lambda functions?**

**Answer:**

- Use IAM roles with least privilege
- Use environment variable encryption (KMS)
- Enable VPC if needed
- Apply resource-based policies for triggers

---

### 19. **What are custom runtimes in Lambda?**

**Answer:** Custom runtimes allow you to use programming languages not natively supported by AWS Lambda by implementing the Lambda Runtime API in your code.

---

### 20. **How can you reduce Lambda cold starts?**

**Answer:**

- Use Provisioned Concurrency
- Use **Lambda SnapStart** (for Java — pre-initializes and caches execution environments)
- Minimize package size and dependencies
- Use `provided.al2023` runtime for faster startup
- Avoid VPC unless required; if needed, use VPC endpoints
- Keep initialization code outside the handler

---

### 21. **How does AWS Lambda handle concurrency limits?**

**Answer:** Lambda has a regional concurrency limit. If that limit is reached, additional requests are throttled unless reserved or provisioned concurrency is configured.

---

### 22. **How do you version and alias a Lambda function?**

**Answer:** Versions are immutable snapshots. Aliases point to specific versions and can be used for blue-green or canary deployments.

---

### 23. **How can you implement a Lambda function with retry logic?**

**Answer:** AWS retries asynchronous invocations twice by default. You can also implement retries in your code or configure them using EventBridge or SQS.

---

### 24. **What is the difference between Amazon EventBridge and CloudWatch Events for Lambda?**

**Answer:** EventBridge is the successor to CloudWatch Events. It offers the same core functionality plus support for SaaS integrations, custom event buses, schema discovery, and advanced content-based filtering. All new projects should use EventBridge. CloudWatch Events is in maintenance mode.

---

### 25. **Can Lambda functions invoke other Lambda functions?**

**Answer:** Yes. One Lambda can invoke another using the AWS SDK (e.g., `lambda.invoke`) or through event sources like SNS.

---

### 26. **What are some limitations of AWS Lambda?**

**Answer:**

- 15-minute execution limit
- Package size limits
- Execution environment is ephemeral
- Limited in high-performance compute or long-running tasks

---

### 27. **How can you reduce the size of your Lambda deployment package?**

**Answer:**

- Use Lambda layers
- Bundle only required dependencies
- Use tree-shaking tools (Webpack, esbuild, etc.)

---

### 28. **How can you test Lambda functions locally?**

**Answer:** Using tools like AWS SAM CLI (`sam local invoke`), `serverless-offline`, or writing unit tests that mock event and context objects.

---

### 29. **How do you optimize cold start performance in a VPC-attached Lambda?**

**Answer:**

- Place Lambda in subnets with minimal route tables
- Use VPC endpoints
- Reduce security group rules
- Use provisioned concurrency

---

### 30. **How do you manage secrets in Lambda?**

**Answer:** Use AWS Secrets Manager or Parameter Store (with encryption), accessed securely from within your Lambda function code.

---

### 31. **What is Lambda Destinations?**

**Answer:** Lambda Destinations allows you to route the result of asynchronous invocations to another Lambda, SQS, SNS, or EventBridge depending on success or failure.

---

### 32. **What is the Init phase in Lambda lifecycle?**

**Answer:** Init phase runs during cold start and includes loading external libraries and initializing SDK clients. Code in this phase persists across invocations.

---

### 33. **How can Lambda be used in CI/CD pipelines?**

**Answer:** Lambda can be deployed using tools like AWS CodePipeline, CodeDeploy, GitHub Actions, or Terraform, ensuring automated testing and deployment.

---

### 34. **How does Lambda pricing work?**

**Answer:** Charged based on number of requests and execution duration (GB-seconds). Provisioned concurrency is charged separately.

---

### 35. **Can Lambda be used for real-time streaming?**

**Answer:** Yes, Lambda can process streaming data from sources like Kinesis or DynamoDB Streams in near real-time.

---

### 36. **What are ephemeral and persistent storage options in Lambda?**

**Answer:**

- **/tmp**: Ephemeral storage per execution environment, configurable from 512 MB to 10 GB
- **EFS**: Persistent, shared network storage mountable across Lambda invocations
- **S3**: Object storage for larger or permanent data

---

### 37. **Can Lambda access internet when in a VPC?**

**Answer:** Lambda functions in a VPC can only access the internet if they are in a **private subnet** with a route to a **NAT Gateway** (or NAT Instance) in a public subnet. Lambda functions cannot be placed directly in a public subnet with an Internet Gateway.

---

### 38. **What are some best practices for writing Lambda functions?**

**Answer:**

- Keep functions small and focused
- Handle errors and timeouts gracefully
- Use environment variables for configuration
- Monitor with metrics and logs

---

### 39. **How does Lambda handle file uploads?**

**Answer:** Use S3 for file storage. Lambda can be triggered on upload (S3 event), process the file, and store results back in S3 or another service.

---

### 40. **How do you protect Lambda from abuse or excessive calls?**

**Answer:** Use API Gateway throttling, Lambda concurrency limits, and WAF rules to control and rate-limit incoming requests.

---

## Advanced & Difficult-Level AWS Lambda Interview Questions

### 41. **Design a fault-tolerant serverless application using Lambda, DynamoDB, and SQS.**

**Answer:** Use API Gateway to trigger Lambda. The function writes to SQS. A second Lambda polls the queue and stores processed data in DynamoDB. Use DLQ and retries for fault tolerance.

---

### 42. **Explain how you would build a multi-region active-active architecture using AWS Lambda.**

**Answer:** Deploy Lambda and supporting services (API Gateway, DynamoDB Global Tables) in multiple regions. Use Route 53 with latency-based routing and failover mechanisms.

---

### 43. **How do you handle transaction consistency in a serverless microservice architecture with Lambda?**

**Answer:** Use a combination of idempotent functions, DynamoDB transactions, and event sourcing or Saga patterns to manage distributed state transitions.

---

### 44. **Describe the security implications of Lambda’s shared execution environment.**

**Answer:** While each invocation runs in a separate container, the underlying hardware is shared. Apply strong isolation practices, avoid writing to shared `/tmp`, and encrypt all data.

---

### 45. **Can you explain how Lambda integrates with Step Functions and the trade-offs?**

**Answer:** Lambda can be used as tasks in Step Functions to orchestrate workflows. Pros: serverless orchestration, error handling. Cons: state transition cost, limited express state timeout.

---

### 46. **How do you achieve zero-downtime deployments with Lambda?**

**Answer:** Use versions and aliases. Deploy new version, shift traffic gradually using weighted aliases or CodeDeploy blue/green deployments.

---

### 47. **Compare Lambda with Fargate and EC2 for backend workloads.**

**Answer:**

| Feature     | Lambda                 | Fargate              | EC2               |
| ----------- | ---------------------- | -------------------- | ----------------- |
| Max runtime | 15 min                 | Unlimited            | Unlimited         |
| Scaling     | Auto (per-request)     | Auto (task-based)    | Manual/ASG        |
| Pricing     | Per-request + duration | Per vCPU/memory/hour | Per instance/hour |
| Startup     | Cold start (ms-sec)    | ~30s–1min            | Minutes           |
| Control     | Minimal                | Container-level      | Full OS           |

Choose Lambda for short-lived, event-driven tasks. Fargate for containerized workloads. EC2 for full control or GPU access.

---

### 48. **How would you mitigate noisy-neighbor performance issues in Lambda?**

**Answer:** Use provisioned concurrency, break workloads into smaller functions, and monitor duration metrics. If consistent latency is required, move to containers or EC2.

---

### 49. **Describe a use case where Lambda is not the right choice.**

**Answer:** Real-time multiplayer games, long-running compute-heavy tasks (like video rendering), and apps needing GPU access or persistent connections (e.g., WebSockets at scale).

---

### 50. **Explain the lifecycle of a Lambda function in detail.**

**Answer:** Lifecycle has **Init** (cold start – run once), **Invoke** (request handling), and **Shutdown** (cleanup). Init is reused with provisioned concurrency. State outside handler persists within same container.

---

## Practical System Design Supplement

---

### 51. **How do you deploy a Lambda function with CDK (production-grade)?**

**Answer:**

```ts
import * as cdk from "aws-cdk-lib";
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as logs from "aws-cdk-lib/aws-logs";
import * as sqs from "aws-cdk-lib/aws-sqs";

const dlq = new sqs.Queue(this, "DLQ", {
  retentionPeriod: cdk.Duration.days(14),
});

const fn = new lambda.Function(this, "OrderProcessor", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("lambda/order-processor"),
  memorySize: 512,
  timeout: cdk.Duration.seconds(30),
  architecture: lambda.Architecture.ARM_64,    // 20% cheaper than x86
  environment: {
    TABLE_NAME: table.tableName,
    POWERTOOLS_SERVICE_NAME: "order-service",
    LOG_LEVEL: "INFO",
  },
  tracing: lambda.Tracing.ACTIVE,              // X-Ray
  insightsVersion: lambda.LambdaInsightsVersion.VERSION_1_0_229_0,
  logRetention: logs.RetentionDays.ONE_MONTH,
  deadLetterQueue: dlq,                        // DLQ for async invocations
  retryAttempts: 2,
  reservedConcurrentExecutions: 100,           // Protect downstream services
});

// Provisioned concurrency via alias (for latency-critical paths)
const alias = new lambda.Alias(this, "ProdAlias", {
  aliasName: "prod",
  version: fn.currentVersion,
  provisionedConcurrentExecutions: 5,
});

// Auto-scaling provisioned concurrency
const scaling = alias.addAutoScaling({ maxCapacity: 50, minCapacity: 5 });
scaling.scaleOnUtilization({ utilizationTarget: 0.7 });

// Grant least-privilege access
table.grantReadWriteData(fn);
```

---

### 52. **How do you invoke Lambda programmatically from TypeScript?**

**Answer:**

```ts
import { LambdaClient, InvokeCommand } from "@aws-sdk/client-lambda";

const lambda = new LambdaClient({});

// Synchronous invocation (wait for response)
const syncResult = await lambda.send(
  new InvokeCommand({
    FunctionName: "order-processor",
    InvocationType: "RequestResponse",
    Payload: Buffer.from(JSON.stringify({ orderId: "ord-123" })),
  }),
);
const response = JSON.parse(Buffer.from(syncResult.Payload!).toString());

// Asynchronous invocation (fire-and-forget)
await lambda.send(
  new InvokeCommand({
    FunctionName: "email-sender",
    InvocationType: "Event", // Returns 202 immediately
    Payload: Buffer.from(JSON.stringify({ to: "user@example.com" })),
  }),
);
```

> **Production Tip:** Use `Event` invocation type for non-critical tasks. Lambda auto-retries twice and sends failures to DLQ. This decouples the caller from processing time.

---

### 53. **What are Lambda's key cost optimization strategies?**

**Answer:**

| Strategy | Savings | Effort |
| --- | --- | --- |
| ARM64 (Graviton) architecture | 20% cheaper, 34% better perf | Low (set `architecture`) |
| Right-size memory (use Power Tuning) | 10–50% | Medium |
| Minimize cold starts (small bundles, lazy init) | Reduces billed init time | Medium |
| Use provisioned concurrency only where needed | Avoid paying for idle concurrency | Low |
| Batch processing (SQS, Kinesis) | Fewer invocations | Low |
| Avoid VPC unless required | Eliminates ENI cold start overhead | Low |
| Use Lambda Layers for shared deps | Smaller deployment, faster deploys | Medium |

```ts
// ❌ Bad: 1024 MB for a simple API handler
new lambda.Function(this, "Simple", { memorySize: 1024 });

// ✅ Good: Use AWS Lambda Power Tuning to find optimal
// Usually 256–512 MB for API handlers, 128 MB for simple transforms
new lambda.Function(this, "Optimized", {
  memorySize: 256,
  architecture: lambda.Architecture.ARM_64,
});
```

**Cost estimate (us-east-1):**

| Memory | Duration | Invocations/month | Monthly Cost |
| --- | --- | --- | --- |
| 128 MB | 100ms | 1M | ~$0.21 |
| 256 MB | 100ms | 1M | ~$0.42 |
| 512 MB | 200ms | 10M | ~$16.70 |
| 1024 MB | 500ms | 10M | ~$83.40 |

---

### 54. **What hidden/lesser-known Lambda features are useful?**

**Answer:**

| Feature | What It Does | Why It Matters |
| --- | --- | --- |
| **Response streaming** | Stream response progressively | Long responses without timeout |
| **SnapStart** | Snapshot init phase (Java) | Near-zero cold starts for Java |
| **Function URLs** | Built-in HTTPS endpoint | Skip API Gateway for simple use cases |
| **Recursive loop detection** | Auto-detects Lambda→SQS→Lambda loops | Prevents infinite invocation costs |
| **`/tmp` storage** | 512 MB–10 GB ephemeral storage | Cache files between invocations (same container) |
| **Provisioned concurrency auto-scaling** | Scale warm instances by schedule/utilization | Match traffic patterns without over-provisioning |
| **Lambda Insights** | Enhanced monitoring (memory, CPU, network) | Find memory leaks, optimize sizing |
| **Event filtering** | Filter at event source level | Reduce invocations for SQS/Kinesis/DynamoDB triggers |

```ts
// Event source filtering: Only invoke Lambda for specific events
import * as eventsources from "aws-cdk-lib/aws-lambda-event-sources";

fn.addEventSource(
  new eventsources.SqsEventSource(queue, {
    batchSize: 10,
    filters: [
      lambda.FilterCriteria.filter({
        body: {
          eventType: lambda.FilterRule.isEqual("ORDER_CREATED"),
          amount: lambda.FilterRule.between(100, 10000),
        },
      }),
    ],
  }),
);

// Function URL (skip API Gateway for internal/simple endpoints)
const fnUrl = fn.addFunctionUrl({
  authType: lambda.FunctionUrlAuthType.AWS_IAM,
  cors: {
    allowedOrigins: ["https://myapp.com"],
    allowedMethods: [lambda.HttpMethod.GET, lambda.HttpMethod.POST],
  },
});
```

---

### 55. **What are Lambda's resiliency and scalability patterns?**

**Answer:**

| Concern | Solution |
| --- | --- |
| Cold start latency | Provisioned concurrency + ARM64 + small bundles |
| Downstream overload | Reserved concurrency to cap parallel invocations |
| Transient failures | Built-in retries (async: 2, stream: configurable) |
| Poison messages | DLQ (async) or `bisectBatchOnError` (streams) |
| Timeout | Set timeout < downstream timeout; use heartbeats |
| Region failure | Multi-region with Route 53 health checks |
| Throttling | Reserved concurrency + SQS buffer |
| Memory leaks | Monitor with Lambda Insights; container reuse is time-boxed |

```ts
// CDK: Lambda with SQS buffer for resiliency
const buffer = new sqs.Queue(this, "Buffer", {
  visibilityTimeout: cdk.Duration.minutes(6), // 6× Lambda timeout
  deadLetterQueue: { queue: dlq, maxReceiveCount: 3 },
});

fn.addEventSource(
  new eventsources.SqsEventSource(buffer, {
    batchSize: 10,
    maxBatchingWindow: cdk.Duration.seconds(5),
    reportBatchItemFailures: true,
  }),
);
```
