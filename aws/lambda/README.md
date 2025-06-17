# AWS Lambda Interview Questions and Answers

A curated list of **AWS Lambda interview questions**, with clear and concise answers. The first 40 questions cover foundational and intermediate topics. The final 10 questions are **advanced and high-difficulty**, designed for expert-level interviews.

---

### 1. **What is AWS Lambda?**

**Answer:** AWS Lambda is a serverless compute service that runs code in response to events and automatically manages the compute resources. It eliminates the need to provision or manage servers.

---

### 2. **What languages does AWS Lambda support?**

**Answer:** AWS Lambda natively supports Node.js, Python, Java, Ruby, Go, .NET Core, and custom runtimes via the AWS Lambda Runtime API (e.g., Rust, PHP).

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

* **Synchronous**: Caller waits for the result (e.g., API Gateway).
* **Asynchronous**: Lambda queues the event and processes it later (e.g., S3, SNS).

---

### 8. **How can you handle errors in Lambda?**

**Answer:**

* **Synchronous**: Return proper status codes.
* **Asynchronous**: Use Dead Letter Queues (DLQs) or EventBridge retry policies.
* **All**: Wrap code in try-catch and monitor via CloudWatch logs.

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

* **Direct upload**: 50 MB (zipped)
* **Via S3**: 250 MB (unzipped)

---

### 18. **How do you secure Lambda functions?**

**Answer:**

* Use IAM roles with least privilege
* Use environment variable encryption (KMS)
* Enable VPC if needed
* Apply resource-based policies for triggers

---

### 19. **What are custom runtimes in Lambda?**

**Answer:** Custom runtimes allow you to use programming languages not natively supported by AWS Lambda by implementing the Lambda Runtime API in your code.

---

### 20. **How can you reduce Lambda cold starts?**

**Answer:**

* Use provisioned concurrency
* Minimize package size
* Avoid VPC or optimize VPC settings (e.g., use private subnets with NAT Gateway)

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

**Answer:** EventBridge is an evolution of CloudWatch Events with support for SaaS apps and advanced filtering. Both can trigger Lambda functions.

---

### 25. **Can Lambda functions invoke other Lambda functions?**

**Answer:** Yes. One Lambda can invoke another using the AWS SDK (e.g., `lambda.invoke`) or through event sources like SNS.

---

### 26. **What are some limitations of AWS Lambda?**

**Answer:**

* 15-minute execution limit
* Package size limits
* Execution environment is ephemeral
* Limited in high-performance compute or long-running tasks

---

### 27. **How can you reduce the size of your Lambda deployment package?**

**Answer:**

* Use Lambda layers
* Bundle only required dependencies
* Use tree-shaking tools (Webpack, esbuild, etc.)

---

### 28. **How can you test Lambda functions locally?**

**Answer:** Using tools like AWS SAM CLI (`sam local invoke`), `serverless-offline`, or writing unit tests that mock event and context objects.

---

### 29. **How do you optimize cold start performance in a VPC-attached Lambda?**

**Answer:**

* Place Lambda in subnets with minimal route tables
* Use VPC endpoints
* Reduce security group rules
* Use provisioned concurrency

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

* **/tmp (512MB or 10GB)**: Ephemeral storage per invocation
* **EFS**: Persistent network storage shared across Lambdas

---

### 37. **Can Lambda access internet when in a VPC?**

**Answer:** Only if it’s in a public subnet or a private subnet with a NAT Gateway attached.

---

### 38. **What are some best practices for writing Lambda functions?**

**Answer:**

* Keep functions small and focused
* Handle errors and timeouts gracefully
* Use environment variables for configuration
* Monitor with metrics and logs

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

**Answer:** Lambda: best for short-lived, event-driven tasks. Fargate: container-based, mid-range tasks. EC2: long-running, full control workloads. Choose based on control, cost, and runtime.

---

### 48. **How would you mitigate noisy-neighbor performance issues in Lambda?**

**Answer:** Use provisioned concurrency, break workloads into smaller functions, and monitor duration metrics. If consistent latency is required, move to containers or EC2.

---

### 49. **Describe a use case where Lambda is not the right choice.**

**Answer:** Real-time multiplayer games, long-running compute-heavy tasks (like video rendering), and apps needing GPU access or persistent connections (e.g., WebSockets at scale).

---

### 50. **Explain the lifecycle of a Lambda function in detail.**

**Answer:** Lifecycle has **Init** (cold start – run once), **Invoke** (request handling), and **Shutdown** (cleanup). Init is reused with provisioned concurrency. State outside handler persists within same container.
