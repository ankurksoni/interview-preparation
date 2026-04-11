# Amazon API Gateway Interview Questions & Answers

A curated list of **Amazon API Gateway interview questions** with practical, production-grade answers. Covers REST vs HTTP vs WebSocket APIs, throttling, caching, authorization, CDK/TypeScript examples, and real-world system design decisions.

---

### 1. **What is Amazon API Gateway?**

**Answer:** API Gateway is a **fully managed service** for creating, publishing, and managing APIs at any scale. It acts as a "front door" between clients and backend services (Lambda, ECS, HTTP endpoints, AWS services).

**Three API types:**

| Type | Protocol | Use Case | Pricing |
| --- | --- | --- | --- |
| **REST API** | REST | Full-featured (caching, WAF, resource policies, API keys) | $3.50/million requests |
| **HTTP API** | REST | Lower cost, simpler, JWT auth, faster | $1.00/million requests |
| **WebSocket API** | WebSocket | Real-time bidirectional (chat, gaming, live updates) | $1.00/million msgs + $0.25/million connection-min |

> **System Design Tip:** Default to **HTTP API** unless you specifically need REST API features (caching, WAF integration, request validation, API key usage plans). HTTP API is 71% cheaper and has lower latency (~10ms less).

---

### 2. **When should you use API Gateway vs when NOT to?**

**Answer:**

| Use API Gateway When | Don't Use API Gateway When |
| --- | --- |
| Serverless APIs (Lambda backend) | Ultra-low-latency APIs (<10ms overhead) |
| Rate limiting and throttling needed | gRPC services (use ALB or NLB) |
| Multiple auth strategies (IAM, JWT, Cognito) | Internal service-to-service calls (use ALB/NLB) |
| API versioning and staged rollouts | Large file uploads >10MB payload (use S3 presigned URLs) |
| Request/response transformation | GraphQL APIs (use AppSync) |
| WebSocket real-time communication | Very high throughput (>10K RPS sustained, consider ALB) |
| Public-facing APIs | Simple static content (use CloudFront) |

**Payload limits:**

| Limit | REST API | HTTP API | WebSocket |
| --- | --- | --- | --- |
| Max payload | 10 MB | 10 MB | 128 KB per frame |
| Max timeout | 29 seconds | 29 seconds | 2 hours idle, 10 min message |

---

### 3. **How do you create a production API with CDK?**

**Answer:**

```ts
import * as cdk from "aws-cdk-lib";
import * as apigateway from "aws-cdk-lib/aws-apigateway";
import * as lambda from "aws-cdk-lib/aws-lambda";

// Lambda handler
const handler = new lambda.Function(this, "ApiHandler", {
  runtime: lambda.Runtime.NODEJS_20_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("lambda"),
  memorySize: 256,
  timeout: cdk.Duration.seconds(10),
  environment: { TABLE_NAME: table.tableName },
});

// REST API with production settings
const api = new apigateway.RestApi(this, "ProductApi", {
  restApiName: "Product Service",
  description: "Product catalog API",
  deployOptions: {
    stageName: "prod",
    throttlingRateLimit: 1000,      // 1000 requests/sec
    throttlingBurstLimit: 2000,     // Burst up to 2000
    loggingLevel: apigateway.MethodLoggingLevel.INFO,
    dataTraceEnabled: false,        // Don't log request/response bodies in prod
    metricsEnabled: true,
    tracingEnabled: true,           // X-Ray tracing
    cachingEnabled: true,
    cacheClusterEnabled: true,
    cacheClusterSize: "0.5",        // GB
    cacheTtl: cdk.Duration.minutes(5),
  },
  defaultCorsPreflightOptions: {
    allowOrigins: ["https://myapp.com"],
    allowMethods: ["GET", "POST", "PUT", "DELETE"],
    allowHeaders: ["Content-Type", "Authorization"],
    maxAge: cdk.Duration.hours(1),
  },
});

// Resources and methods
const products = api.root.addResource("products");
products.addMethod("GET", new apigateway.LambdaIntegration(handler), {
  authorizationType: apigateway.AuthorizationType.IAM,
});

const product = products.addResource("{productId}");
product.addMethod("GET", new apigateway.LambdaIntegration(handler));
product.addMethod("PUT", new apigateway.LambdaIntegration(handler), {
  authorizationType: apigateway.AuthorizationType.COGNITO,
  authorizer: cognitoAuthorizer,
  requestValidator: bodyValidator,
  requestModels: { "application/json": productModel },
});
```

---

### 4. **How do HTTP APIs differ from REST APIs in CDK?**

**Answer:**

```ts
import * as apigwv2 from "aws-cdk-lib/aws-apigatewayv2";
import * as integrations from "aws-cdk-lib/aws-apigatewayv2-integrations";
import * as authorizers from "aws-cdk-lib/aws-apigatewayv2-authorizers";

// HTTP API (cheaper, simpler, faster)
const httpApi = new apigwv2.HttpApi(this, "HttpApi", {
  apiName: "product-api",
  corsPreflight: {
    allowOrigins: ["https://myapp.com"],
    allowMethods: [apigwv2.CorsHttpMethod.GET, apigwv2.CorsHttpMethod.POST],
    allowHeaders: ["Content-Type", "Authorization"],
    maxAge: cdk.Duration.hours(1),
  },
  defaultAuthorizer: new authorizers.HttpJwtAuthorizer("JwtAuth", "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_abc123", {
    jwtAudience: ["my-api-client-id"],
  }),
});

httpApi.addRoutes({
  path: "/products",
  methods: [apigwv2.HttpMethod.GET],
  integration: new integrations.HttpLambdaIntegration("GetProducts", handler),
});

httpApi.addRoutes({
  path: "/products/{productId}",
  methods: [apigwv2.HttpMethod.GET, apigwv2.HttpMethod.PUT],
  integration: new integrations.HttpLambdaIntegration("ProductById", handler),
});
```

**Feature comparison:**

| Feature | REST API | HTTP API |
| --- | --- | --- |
| Price | $3.50/M requests | $1.00/M requests |
| Latency | ~15ms overhead | ~5ms overhead |
| Caching | Built-in (0.5–237 GB) | No (use CloudFront) |
| WAF | Yes | No |
| API Keys / Usage Plans | Yes | No |
| Request validation | Yes | No (validate in Lambda) |
| Request/response transforms | VTL templates | No |
| JWT authorizer | Via Lambda | Native |
| IAM auth | Yes | Yes |
| Resource policies | Yes | No |
| Private integrations | Yes | Yes |

---

### 5. **What are the important config parameters?**

**Answer:**

| Parameter | Description | Production Recommendation |
| --- | --- | --- |
| `throttlingRateLimit` | Steady-state requests/second | Set per stage and per method |
| `throttlingBurstLimit` | Max concurrent requests | 2× rate limit |
| `cacheClusterSize` | Cache size in GB (REST only) | 0.5–6.1 GB based on response sizes |
| `cacheTtl` | Cache duration per method | 60–300s for read-heavy; 0 for writes |
| `loggingLevel` | CloudWatch log verbosity | INFO in prod, ERROR minimum |
| `dataTraceEnabled` | Log full request/response | **false in prod** (PII risk, cost) |
| `tracingEnabled` | X-Ray tracing | Enable for debugging latency |
| `minimumCompressionSize` | Gzip responses above size | 1000 bytes (saves bandwidth) |
| `timeout` | Backend integration timeout | Set close to Lambda timeout |

```ts
// Per-method throttling (important for write protection)
products.addMethod("POST", new apigateway.LambdaIntegration(handler), {
  methodResponses: [{ statusCode: "201" }],
});

// Usage plans for API Key-based throttling
const plan = api.addUsagePlan("BasicPlan", {
  name: "Basic",
  throttle: { rateLimit: 100, burstLimit: 200 },
  quota: { limit: 10_000, period: apigateway.Period.DAY },
});

const apiKey = api.addApiKey("CustomerKey", { apiKeyName: "customer-abc" });
plan.addApiKey(apiKey);
plan.addApiStage({ stage: api.deploymentStage });
```

> **Hidden Property:** `minimumCompressionSize` is often overlooked. Set it to ~1000 bytes to automatically gzip responses, reducing transfer time by 60–80% for JSON payloads.

---

### 6. **How do you implement authorization patterns?**

**Answer:**

**Four authorization strategies:**

```ts
// 1. IAM Authorization (service-to-service, AWS SigV4)
products.addMethod("GET", integration, {
  authorizationType: apigateway.AuthorizationType.IAM,
});

// 2. Cognito Authorizer (user pools)
const cognitoAuth = new apigateway.CognitoUserPoolsAuthorizer(this, "CognitoAuth", {
  cognitoUserPools: [userPool],
  resultsCacheTtl: cdk.Duration.minutes(5),
});

products.addMethod("POST", integration, {
  authorizer: cognitoAuth,
  authorizationScopes: ["products/write"],
});

// 3. Lambda Authorizer (custom logic — API keys, JWTs, custom tokens)
const tokenAuth = new apigateway.TokenAuthorizer(this, "TokenAuth", {
  handler: authFunction,
  resultsCacheTtl: cdk.Duration.minutes(5),
  identitySource: "method.request.header.Authorization",
});

products.addMethod("DELETE", integration, {
  authorizer: tokenAuth,
});

// 4. HTTP API JWT Authorizer (simplest for Cognito/Auth0/Okta)
const jwtAuth = new authorizers.HttpJwtAuthorizer("JwtAuth", issuerUrl, {
  jwtAudience: [clientId],
});
```

**Lambda authorizer handler:**

```ts
import { APIGatewayTokenAuthorizerEvent, APIGatewayAuthorizerResult } from "aws-lambda";

export const handler = async (
  event: APIGatewayTokenAuthorizerEvent,
): Promise<APIGatewayAuthorizerResult> => {
  const token = event.authorizationToken.replace("Bearer ", "");

  try {
    const decoded = await verifyJwt(token); // Your JWT verification

    return {
      principalId: decoded.sub,
      policyDocument: {
        Version: "2012-10-17",
        Statement: [
          {
            Action: "execute-api:Invoke",
            Effect: "Allow",
            Resource: event.methodArn,
          },
        ],
      },
      context: { userId: decoded.sub, role: decoded.role },
    };
  } catch {
    throw new Error("Unauthorized"); // Returns 401
  }
};
```

> **Production Tip:** Always set `resultsCacheTtl` on authorizers (5–300s). Without caching, every request invokes the authorizer Lambda, adding latency and cost.

---

### 7. **How does API Gateway caching work?**

**Answer:** REST API has built-in caching. Cache is per-stage, keyed by URL + query params + headers.

```ts
// ✅ Good: Cache GET requests, bypass for writes
const getMethod = products.addMethod("GET", integration, {
  methodResponses: [{ statusCode: "200" }],
});

// Per-method cache override
const cfnMethod = getMethod.node.defaultChild as apigateway.CfnMethod;
cfnMethod.addPropertyOverride("Integration.CacheKeyParameters", [
  "method.request.querystring.category",
  "method.request.querystring.page",
]);

// Cache invalidation via header
// Client sends: Cache-Control: max-age=0
// API Gateway: Must enable "Require authorization for cache invalidation"
```

**Cache strategies:**

| Strategy | Use Case | TTL |
| --- | --- | --- |
| Full caching | Static catalog data | 300s |
| Short cache | Frequently changing lists | 30–60s |
| No cache | User-specific data, writes | 0 |
| Cache key customization | Different cache per query params | Varies |

> **Hidden Gem:** For HTTP APIs (which lack built-in caching), put **CloudFront in front** with cache behaviors. This gives you caching + WAF + global distribution at minimal extra cost.

```ts
// CloudFront + HTTP API (caching for HTTP APIs)
import * as cloudfront from "aws-cdk-lib/aws-cloudfront";
import * as origins from "aws-cdk-lib/aws-cloudfront-origins";

const distribution = new cloudfront.Distribution(this, "ApiCdn", {
  defaultBehavior: {
    origin: new origins.HttpOrigin(`${httpApi.httpApiId}.execute-api.${cdk.Aws.REGION}.amazonaws.com`),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.HTTPS_ONLY,
    cachePolicy: new cloudfront.CachePolicy(this, "ApiCachePolicy", {
      defaultTtl: cdk.Duration.minutes(5),
      maxTtl: cdk.Duration.hours(1),
      queryStringBehavior: cloudfront.CacheQueryStringBehavior.all(),
      headerBehavior: cloudfront.CacheHeaderBehavior.allowList("Authorization"),
    }),
    originRequestPolicy: cloudfront.OriginRequestPolicy.ALL_VIEWER_EXCEPT_HOST_HEADER,
  },
});
```

---

### 8. **How do you handle errors and request validation?**

**Answer:**

```ts
// Request validation (REST API only — catches bad requests before Lambda)
const validator = new apigateway.RequestValidator(this, "BodyValidator", {
  restApi: api,
  validateRequestBody: true,
  validateRequestParameters: true,
});

const productModel = api.addModel("ProductModel", {
  contentType: "application/json",
  schema: {
    type: apigateway.JsonSchemaType.OBJECT,
    required: ["name", "price"],
    properties: {
      name: { type: apigateway.JsonSchemaType.STRING, minLength: 1, maxLength: 200 },
      price: { type: apigateway.JsonSchemaType.NUMBER, minimum: 0 },
      category: { type: apigateway.JsonSchemaType.STRING },
    },
  },
});

products.addMethod("POST", integration, {
  requestValidator: validator,
  requestModels: { "application/json": productModel },
});

// Custom Gateway Responses (global error formatting)
api.addGatewayResponse("Throttled", {
  type: apigateway.ResponseType.THROTTLED,
  statusCode: "429",
  responseHeaders: { "Access-Control-Allow-Origin": "'*'" },
  templates: {
    "application/json": '{"message":"Rate limit exceeded. Try again later.","requestId":"$context.requestId"}',
  },
});

api.addGatewayResponse("Unauthorized", {
  type: apigateway.ResponseType.UNAUTHORIZED,
  statusCode: "401",
  responseHeaders: { "Access-Control-Allow-Origin": "'*'" },
  templates: {
    "application/json": '{"message":"Invalid or missing authentication token."}',
  },
});
```

> **Production Tip:** Always customize Gateway Responses. The default error format is XML-like and confusing for API consumers.

---

### 9. **What are API Gateway's resiliency patterns?**

**Answer:**

| Concern | Solution |
| --- | --- |
| Backend overload | Stage/method-level throttling |
| Lambda cold starts | Provisioned concurrency on critical APIs |
| Single region failure | Multi-region with Route 53 failover |
| DDoS | WAF + rate limiting (REST API) |
| Cascading failures | Set integration timeout < 29s |
| Canary deployments | Stage canary settings (% traffic split) |
| Breaking API changes | Use stage variables and versioning |

```ts
// Canary deployment: Route 10% traffic to new version
const deployment = new apigateway.Deployment(this, "Deployment", { api });

const stage = new apigateway.Stage(this, "ProdStage", {
  deployment,
  stageName: "prod",
  // Canary settings
  canarySetting: {
    percentTraffic: 10,       // 10% to canary
    useStageCache: false,     // Don't corrupt main cache
  },
});

// Multi-region with custom domain
const customDomain = new apigateway.DomainName(this, "CustomDomain", {
  domainName: "api.myapp.com",
  certificate: cert,
  endpointType: apigateway.EndpointType.REGIONAL,
});

customDomain.addBasePathMapping(api, { basePath: "v1" });
```

---

### 10. **How do WebSocket APIs work?**

**Answer:** WebSocket APIs maintain persistent bidirectional connections, ideal for real-time features.

```ts
import * as apigwv2 from "aws-cdk-lib/aws-apigatewayv2";

// WebSocket API
const wsApi = new apigwv2.WebSocketApi(this, "ChatApi", {
  connectRouteOptions: {
    integration: new integrations.WebSocketLambdaIntegration("ConnectIntegration", connectHandler),
    authorizer: wsAuthorizer,
  },
  disconnectRouteOptions: {
    integration: new integrations.WebSocketLambdaIntegration("DisconnectIntegration", disconnectHandler),
  },
  defaultRouteOptions: {
    integration: new integrations.WebSocketLambdaIntegration("DefaultIntegration", messageHandler),
  },
});

const wsStage = new apigwv2.WebSocketStage(this, "ProdStage", {
  webSocketApi: wsApi,
  stageName: "prod",
  autoDeploy: true,
});
```

**Send message back to connected client:**

```ts
import { ApiGatewayManagementApiClient, PostToConnectionCommand } from "@aws-sdk/client-apigatewaymanagementapi";

export const handler = async (event: any) => {
  const { connectionId, body } = event.requestContext;
  const message = JSON.parse(body);

  const api = new ApiGatewayManagementApiClient({
    endpoint: `https://${event.requestContext.domainName}/${event.requestContext.stage}`,
  });

  // Send to specific connection
  await api.send(
    new PostToConnectionCommand({
      ConnectionId: connectionId,
      Data: Buffer.from(JSON.stringify({ type: "ack", data: "received" })),
    }),
  );

  // Broadcast to all connections (from DynamoDB connection store)
  const connections = await getActiveConnections(); // Your DynamoDB query
  await Promise.allSettled(
    connections.map((connId) =>
      api.send(
        new PostToConnectionCommand({
          ConnectionId: connId,
          Data: Buffer.from(JSON.stringify(message)),
        }),
      ).catch(async (err) => {
        if (err.statusCode === 410) {
          await removeConnection(connId); // Stale connection
        }
      }),
    ),
  );

  return { statusCode: 200 };
};
```

---

### 11. **What are API Gateway cost optimization strategies?**

**Answer:**

| Strategy | Savings | Effort |
| --- | --- | --- |
| Use HTTP API instead of REST API | 71% per request | Low (migration) |
| Enable caching (REST) or CloudFront (HTTP) | Up to 90% on cache hits | Low |
| Compress responses (`minimumCompressionSize`) | 60–80% data transfer cost | Low |
| Cache Lambda authorizer results | Reduces authorizer invocations | Low |
| Use direct AWS service integrations | Eliminate Lambda costs | Medium |
| Regional endpoint (not Edge) | Lower data transfer | Low |
| Request validation at API GW level | Avoid Lambda invocations for bad requests | Low |

**Direct service integration (skip Lambda):**

```ts
// ✅ API Gateway → DynamoDB directly (no Lambda needed)
const dynamoIntegration = new apigateway.AwsIntegration({
  service: "dynamodb",
  action: "GetItem",
  options: {
    credentialsRole: apiGwRole,
    requestTemplates: {
      "application/json": JSON.stringify({
        TableName: "Products",
        Key: { pk: { S: "$input.params('productId')" } },
      }),
    },
    integrationResponses: [
      {
        statusCode: "200",
        responseTemplates: {
          "application/json": `
            #set($item = $input.path('$.Item'))
            {"id":"$item.pk.S","name":"$item.name.S","price":$item.price.N}
          `,
        },
      },
    ],
  },
});

product.addMethod("GET", dynamoIntegration, {
  methodResponses: [{ statusCode: "200" }],
});
```

> **Hidden Gem:** Direct integrations with DynamoDB, SQS, Step Functions, and S3 skip Lambda entirely — lower latency, lower cost, and fewer failure points.

---

### 12. **What are API Gateway scalability limits?**

**Answer:**

| Limit | Value | Workaround |
| --- | --- | --- |
| Requests/sec (account) | 10,000 (soft limit) | Request increase; use multiple accounts |
| Burst limit | 5,000 | Throttle at usage plan level |
| Payload size | 10 MB | Use S3 presigned URLs for large files |
| Integration timeout | 29 seconds | Async pattern: API GW → SQS → Lambda |
| WebSocket connections | 500/sec connect rate | Connection pooling on client |
| APIs per account | 600 | Organize into fewer APIs with more resources |
| Routes per API (HTTP) | 300 | Split into multiple APIs by domain |
| Stage variables | 100 per stage | Use SSM Parameter Store for overflow |

**Async pattern for long operations:**

```ts
// ❌ Bad: Synchronous call that might timeout (>29s)
// POST /reports → Lambda (runs 5 min) → 504 Gateway Timeout

// ✅ Good: Async pattern
// POST /reports → Lambda → SQS → Worker Lambda (5 min)
//              → Returns 202 Accepted { jobId: "abc" }
// GET /reports/abc → Check status in DynamoDB

const submitJob = new lambda.Function(this, "SubmitJob", {
  handler: "submit.handler",
  timeout: cdk.Duration.seconds(5), // Very fast
});

const workerQueue = new sqs.Queue(this, "WorkerQueue", {
  visibilityTimeout: cdk.Duration.minutes(15),
});

// Submit just enqueues the work
// Client polls GET /reports/{id} for status
```

---

### 13. **What hidden/lesser-known API Gateway features are useful?**

**Answer:**

| Feature | What It Does | Why It Matters |
| --- | --- | --- |
| **$context variables** | Access request metadata in VTL/logs | `$context.requestId` for tracing, `$context.identity.sourceIp` for logging |
| **Gateway responses** | Customize error formats globally | Consistent error responses for 4xx/5xx |
| **Canary releases** | % traffic routing to new deployment | Safe rollouts |
| **Mock integrations** | Return hardcoded responses | API-first development, testing |
| **VPC Links** | Private integrations to NLB/ALB | Access internal services |
| **Stage variables** | Per-stage config (Lambda alias, URLs) | Same API code, different backends |
| **Access logging** | Custom log format with $context vars | Detailed API analytics |
| **Mutual TLS** | Client certificate authentication | B2B API security |

```ts
// Custom access logging for API analytics
const logGroup = new logs.LogGroup(this, "ApiAccessLogs", {
  retention: logs.RetentionDays.ONE_MONTH,
});

const stage = api.deploymentStage;
const cfnStage = stage.node.defaultChild as apigateway.CfnStage;
cfnStage.accessLogSetting = {
  destinationArn: logGroup.logGroupArn,
  format: JSON.stringify({
    requestId: "$context.requestId",
    ip: "$context.identity.sourceIp",
    method: "$context.httpMethod",
    path: "$context.path",
    status: "$context.status",
    latency: "$context.responseLatency",
    integrationLatency: "$context.integrationLatency",
    userAgent: "$context.identity.userAgent",
  }),
};
```

> **Hidden Gem:** `$context.integrationLatency` vs `$context.responseLatency` tells you exactly how much time is API Gateway overhead vs your backend. Essential for performance debugging.

---

### 14. **How do you implement API versioning?**

**Answer:**

```ts
// Strategy 1: Path-based versioning (most common)
const v1 = api.root.addResource("v1");
const v2 = api.root.addResource("v2");

v1.addResource("products").addMethod("GET", v1Handler);
v2.addResource("products").addMethod("GET", v2Handler);

// Strategy 2: Stage-based with stage variables
const deployment = new apigateway.Deployment(this, "Deployment", { api });

// Each stage points to different Lambda alias
const prodStage = new apigateway.Stage(this, "Prod", {
  deployment,
  stageName: "prod",
  variables: { lambdaAlias: "stable" },
});

const betaStage = new apigateway.Stage(this, "Beta", {
  deployment,
  stageName: "beta",
  variables: { lambdaAlias: "beta" },
});

// Strategy 3: Header-based (via Lambda authorizer or request mapping)
// Client sends: Accept: application/vnd.myapi.v2+json
// Lambda checks header and routes accordingly
```

---

### 15. **How do you monitor API Gateway effectively?**

**Answer:**

```ts
import * as cloudwatch from "aws-cdk-lib/aws-cloudwatch";

// 5xx error rate alarm
new cloudwatch.Alarm(this, "Api5xxAlarm", {
  metric: api.metricServerError({
    statistic: "Sum",
    period: cdk.Duration.minutes(1),
  }),
  threshold: 10,
  evaluationPeriods: 2,
  alarmDescription: "API returning 5xx errors",
});

// Latency alarm (p99)
new cloudwatch.Alarm(this, "LatencyAlarm", {
  metric: api.metricLatency({
    statistic: "p99",
    period: cdk.Duration.minutes(5),
  }),
  threshold: 5000, // 5 seconds
  evaluationPeriods: 3,
  alarmDescription: "API p99 latency exceeds 5s",
});

// 4xx spike alarm (might indicate bad deployment or attack)
new cloudwatch.Alarm(this, "Api4xxAlarm", {
  metric: api.metricClientError({
    statistic: "Sum",
    period: cdk.Duration.minutes(5),
  }),
  threshold: 100,
  evaluationPeriods: 2,
});
```

**Key metrics to monitor:**

| Metric | Healthy | Alert When |
| --- | --- | --- |
| `5XXError` | 0 | > 0 sustained |
| `4XXError` | Low, stable | Sudden spike |
| `Latency` (p99) | < 1s | > 3s |
| `IntegrationLatency` | < 500ms | > 2s |
| `Count` | Stable pattern | Unexpected drop or spike |
| `CacheHitCount/CacheMissCount` | High hit ratio | Ratio drops below 50% |
