# AWS CloudFormation & CDK Interview Questions & Answers

A curated list of **AWS CloudFormation and CDK (Cloud Development Kit) interview questions** with practical, production-grade answers. Covers IaC fundamentals, stacks, constructs, patterns, testing, CI/CD pipelines, and real-world deployment strategies.

---

### 1. **What is AWS CloudFormation?**

**Answer:** CloudFormation is AWS's native **Infrastructure as Code (IaC)** service. You define AWS resources in a template (JSON/YAML), and CloudFormation provisions and manages them as a **stack**. It handles dependency ordering, rollback on failure, and drift detection.

---

### 2. **What is AWS CDK and how does it relate to CloudFormation?**

**Answer:** CDK (Cloud Development Kit) is a framework that lets you define cloud infrastructure using programming languages (TypeScript, Python, Java, Go, C#) instead of writing raw YAML/JSON. CDK **synthesizes** your code into a CloudFormation template, which is then deployed.

```
CDK Code (TypeScript) → cdk synth → CloudFormation Template (YAML) → cdk deploy → AWS Resources
```

**Why CDK over raw CloudFormation:**

- Loops, conditionals, and abstractions via real code
- Type safety and IDE auto-complete
- Higher-level constructs (L2/L3) reduce boilerplate
- Unit testable infrastructure
- Share constructs as libraries (npm, PyPI)

```ts
// Raw CloudFormation YAML (verbose)
// Resources:
//   MyBucket:
//     Type: AWS::S3::Bucket
//     Properties:
//       BucketEncryption:
//         ServerSideEncryptionConfiguration:
//           - ServerSideEncryptionByDefault:
//               SSEAlgorithm: aws:kms

// CDK equivalent (concise, type-safe)
import * as s3 from "aws-cdk-lib/aws-s3";

new s3.Bucket(this, "MyBucket", {
  encryption: s3.BucketEncryption.KMS_MANAGED,
});
```

---

### 3. **What are CDK construct levels (L1, L2, L3)?**

**Answer:**

| Level | Name               | Description                                             | Example                                                  |
| ----- | ------------------ | ------------------------------------------------------- | -------------------------------------------------------- |
| L1    | CFN Resources      | 1:1 mapping to CloudFormation resources. Prefix: `Cfn*` | `CfnBucket`, `CfnFunction`                               |
| L2    | Curated Constructs | Higher-level with sensible defaults, helper methods     | `s3.Bucket`, `lambda.Function`                           |
| L3    | Patterns           | Multi-resource architectures in a single construct      | `LambdaRestApi`, `ApplicationLoadBalancedFargateService` |

```ts
// L1: Verbose, mirrors CloudFormation exactly
new s3.CfnBucket(this, "L1Bucket", {
  bucketEncryption: {
    serverSideEncryptionConfiguration: [
      {
        serverSideEncryptionByDefault: { sseAlgorithm: "aws:kms" },
      },
    ],
  },
});

// L2: Concise, with helper methods and defaults
const bucket = new s3.Bucket(this, "L2Bucket", {
  encryption: s3.BucketEncryption.KMS_MANAGED,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
});
bucket.grantRead(myLambda); // Auto-generates IAM policy

// L3: Full architecture in one construct
new ecsPatterns.ApplicationLoadBalancedFargateService(this, "L3Service", {
  taskImageOptions: { image: ecs.ContainerImage.fromAsset("./app") },
  publicLoadBalancer: true,
});
```

> **Best practice:** Use L2 constructs whenever possible. Fall back to L1 only when L2 doesn't expose a specific CloudFormation property (use `addPropertyOverride` or escape hatches).

---

### 4. **What is a CDK Stack vs a CDK App?**

**Answer:**

- **App** — The root of your CDK application. Contains one or more stacks.
- **Stack** — Maps to a single CloudFormation stack. Contains constructs (resources). Each stack is independently deployable.

```ts
import * as cdk from "aws-cdk-lib";

const app = new cdk.App();

// Multiple stacks in one app
new NetworkStack(app, "Network", {
  env: { account: "123456789", region: "us-east-1" },
});
new DatabaseStack(app, "Database", {
  env: { account: "123456789", region: "us-east-1" },
});
new AppStack(app, "App", {
  env: { account: "123456789", region: "us-east-1" },
});

app.synth();
```

**Stack limits:**

- 500 resources per stack (soft limit, request increase)
- 51,200 bytes template size (S3-uploaded)
- 200 outputs, 200 parameters, 200 mappings

> **When to split stacks:** Separate by lifecycle (networking rarely changes, app code changes often), team ownership, or deployment frequency.

---

### 5. **How do you pass data between CDK stacks?**

**Answer:** Use stack-to-stack references. CDK automatically creates CloudFormation exports/imports.

```ts
// ❌ Bad: Hardcoding ARNs across stacks
class AppStack extends cdk.Stack {
  constructor(scope: Construct, id: string) {
    super(scope, id);
    const bucket = s3.Bucket.fromBucketName(
      this,
      "Bucket",
      "my-hardcoded-bucket",
    );
  }
}

// ✅ Good: Pass references via constructor props
interface AppStackProps extends cdk.StackProps {
  vpc: ec2.IVpc;
  database: rds.IDatabaseCluster;
}

class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;

  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);
    this.vpc = new ec2.Vpc(this, "VPC", { maxAzs: 2 });
  }
}

class AppStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: AppStackProps) {
    super(scope, id, props);
    // Use the VPC from NetworkStack
    new ecs.Cluster(this, "Cluster", { vpc: props.vpc });
  }
}

// Wire them together
const network = new NetworkStack(app, "Network");
new AppStack(app, "App", { vpc: network.vpc });
```

> **Warning:** Cross-stack references create CloudFormation exports. Deleting or renaming the exporting stack will fail if the importing stack still references it.

---

### 6. **What are CDK Aspects and how do you use them?**

**Answer:** Aspects are a visitor pattern that applies an operation to **every construct** in a scope. They're used for compliance, tagging, and validation.

```ts
import { Aspects, IAspect } from "aws-cdk-lib";
import { CfnBucket } from "aws-cdk-lib/aws-s3";

// Aspect: Enforce encryption on all S3 buckets
class BucketEncryptionChecker implements IAspect {
  visit(node: IConstruct) {
    if (node instanceof CfnBucket) {
      if (!node.bucketEncryption) {
        Annotations.of(node).addError(
          "S3 buckets must have encryption enabled",
        );
      }
    }
  }
}

// Aspect: Enforce tags on all resources
class RequiredTags implements IAspect {
  visit(node: IConstruct) {
    if (cdk.TagManager.isTaggable(node)) {
      cdk.Tags.of(node).add("Environment", "production");
      cdk.Tags.of(node).add("ManagedBy", "cdk");
    }
  }
}

// Apply aspects to the entire app
Aspects.of(app).add(new BucketEncryptionChecker());
Aspects.of(app).add(new RequiredTags());
```

---

### 7. **How do you handle secrets and configuration in CDK?**

**Answer:**

```ts
// ❌ Bad: Hardcoding secrets in CDK code
new lambda.Function(this, "Fn", {
  environment: {
    DB_PASSWORD: "my-secret-password", // Exposed in CloudFormation template!
  },
});

// ✅ Good: Reference from Secrets Manager (resolved at runtime)
const dbSecret = secretsmanager.Secret.fromSecretNameV2(
  this,
  "DBSecret",
  "prod/db/password",
);

new lambda.Function(this, "Fn", {
  environment: {
    DB_SECRET_ARN: dbSecret.secretArn, // Only the ARN, not the value
  },
});
dbSecret.grantRead(fn);

// ✅ Good: Use SSM Parameter Store for non-secret config
const config = ssm.StringParameter.valueForStringParameter(
  this,
  "/prod/app/feature-flags",
);

// ✅ Good: Use CDK context for deployment-time config
const stage = this.node.tryGetContext("stage") || "dev";
const instanceSize =
  stage === "prod" ? ec2.InstanceSize.XLARGE : ec2.InstanceSize.SMALL;
```

---

### 8. **How do you test CDK infrastructure?**

**Answer:** CDK supports three levels of testing:

**1. Snapshot tests** — Detect unintended changes:

```ts
import { Template } from "aws-cdk-lib/assertions";

test("snapshot test", () => {
  const app = new cdk.App();
  const stack = new MyStack(app, "Test");
  const template = Template.fromStack(stack);
  expect(template.toJSON()).toMatchSnapshot();
});
```

**2. Fine-grained assertions** — Verify specific resources:

```ts
test("creates encrypted S3 bucket", () => {
  const app = new cdk.App();
  const stack = new MyStack(app, "Test");
  const template = Template.fromStack(stack);

  template.hasResourceProperties("AWS::S3::Bucket", {
    BucketEncryption: {
      ServerSideEncryptionConfiguration: [
        {
          ServerSideEncryptionByDefault: {
            SSEAlgorithm: "aws:kms",
          },
        },
      ],
    },
  });

  // Count resources
  template.resourceCountIs("AWS::Lambda::Function", 3);

  // Check outputs exist
  template.hasOutput("ApiUrl", {});
});
```

**3. Validation tests** — Test Aspects and custom validation:

```ts
test("fails if bucket is not encrypted", () => {
  const app = new cdk.App();
  const stack = new cdk.Stack(app, "Test");

  new s3.Bucket(stack, "BadBucket"); // No encryption
  Aspects.of(stack).add(new BucketEncryptionChecker());

  const annotations = Annotations.fromStack(stack);
  annotations.hasError(
    "/Test/BadBucket/Resource",
    "S3 buckets must have encryption enabled",
  );
});
```

---

### 9. **What is CDK Pipelines and how do you set up CI/CD?**

**Answer:** CDK Pipelines is an L3 construct that creates a self-mutating CI/CD pipeline. The pipeline updates itself when you change the pipeline definition.

```ts
import {
  CodePipeline,
  CodePipelineSource,
  ShellStep,
} from "aws-cdk-lib/pipelines";

class PipelineStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const pipeline = new CodePipeline(this, "Pipeline", {
      pipelineName: "MyAppPipeline",
      synth: new ShellStep("Synth", {
        input: CodePipelineSource.gitHub("myorg/myrepo", "main", {
          authentication: cdk.SecretValue.secretsManager("github-token"),
        }),
        commands: ["npm ci", "npm run build", "npm test", "npx cdk synth"],
      }),
    });

    // Deploy to staging
    const staging = pipeline.addStage(
      new MyAppStage(this, "Staging", {
        env: { account: "111111111111", region: "us-east-1" },
      }),
    );

    staging.addPost(
      new ShellStep("IntegrationTests", {
        commands: ["npm run test:integration"],
      }),
    );

    // Deploy to production (with manual approval)
    const prod = pipeline.addStage(
      new MyAppStage(this, "Production", {
        env: { account: "222222222222", region: "us-east-1" },
      }),
    );

    prod.addPre(new pipelines.ManualApprovalStep("PromoteToProd"));
  }
}

// Stage: a set of stacks representing one environment
class MyAppStage extends cdk.Stage {
  constructor(scope: Construct, id: string, props?: cdk.StageProps) {
    super(scope, id, props);
    new NetworkStack(this, "Network");
    new AppStack(this, "App");
  }
}
```

---

### 10. **What are CloudFormation change sets and how does CDK use them?**

**Answer:** A change set previews what CloudFormation will modify **before** executing. CDK uses change sets by default during `cdk deploy`.

```bash
# Preview changes without deploying
cdk diff

# Example output:
# [-] AWS::S3::Bucket OldBucket destroy
# [+] AWS::S3::Bucket NewBucket
# [~] AWS::Lambda::Function MyFunc
#  └─ [~] Runtime
#      ├─ [-] nodejs18.x
#      └─ [+] nodejs22.x
```

```bash
# Deploy creates a change set, shows it, and asks for confirmation
cdk deploy --require-approval broadening
# Options: never | anyChange | broadening (new IAM/security changes)
```

> **Important:** Always run `cdk diff` before deploying to production. Some changes cause **resource replacement** (delete-then-create), which can cause downtime.

---

### 11. **What is CloudFormation drift detection?**

**Answer:** Drift occurs when actual resource configuration diverges from the CloudFormation template (e.g., someone manually changes a security group via console).

```bash
# Detect drift
aws cloudformation detect-stack-drift --stack-name MyStack
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id <id>

# View drifted resources
aws cloudformation describe-stack-resource-drifts \
  --stack-name MyStack \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

**Prevention:**

- Use SCPs to deny console changes to CDK-managed resources
- Tag resources with `ManagedBy: cdk` and restrict manual edits
- Run drift detection on a schedule via AWS Config rule `cloudformation-stack-drift-detection-check`

---

### 12. **How do you handle stateful resources (databases, S3) during stack updates?**

**Answer:** CloudFormation may **replace** resources on certain property changes, which deletes the original. Protect stateful resources:

```ts
// 1. Removal Policy: RETAIN prevents deletion even if stack is deleted
const bucket = new s3.Bucket(this, "DataBucket", {
  removalPolicy: cdk.RemovalPolicy.RETAIN, // ✅ Keep on stack delete
  autoDeleteObjects: false,
});

const database = new rds.DatabaseCluster(this, "DB", {
  removalPolicy: cdk.RemovalPolicy.SNAPSHOT, // ✅ Take final snapshot
  deletionProtection: true, // ✅ Prevent accidental deletes
  // ...
});

// 2. Stack-level termination protection
class DataStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, {
      ...props,
      terminationProtection: true, // ✅ Prevents stack deletion
    });
  }
}
```

```ts
// ❌ Dangerous: Changing properties that trigger replacement
// Changing a DynamoDB table name → DELETE old table + CREATE new table = DATA LOSS
new dynamodb.Table(this, "Users", {
  tableName: "users-v2", // Changing from 'users' → triggers replacement!
  partitionKey: { name: "pk", type: dynamodb.AttributeType.STRING },
});

// ✅ Safe: Check cdk diff for "replace" before deploying
// Resources
// [~] AWS::DynamoDB::Table Users REPLACE  ← ⚠️ DANGER
```

---

### 13. **What are CloudFormation custom resources and when do you use them?**

**Answer:** Custom resources let you execute custom logic during stack create/update/delete when CloudFormation doesn't natively support a resource.

```ts
import { custom_resources as cr } from "aws-cdk-lib";

// AwsCustomResource: make SDK calls without writing Lambda code
new cr.AwsCustomResource(this, "EmptyBucketOnDelete", {
  onDelete: {
    service: "S3",
    action: "listObjectsV2",
    parameters: { Bucket: bucket.bucketName },
    physicalResourceId: cr.PhysicalResourceId.of("empty-bucket"),
  },
  policy: cr.AwsCustomResourcePolicy.fromSdkCalls({
    resources: [bucket.bucketArn],
  }),
});

// Provider framework: full custom resource with Lambda
const onEvent = new lambda.Function(this, "OnEvent", {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("custom-resource"),
});

const provider = new cr.Provider(this, "Provider", {
  onEventHandler: onEvent,
  isCompleteHandler: isComplete, // Optional: for async operations
});

new cdk.CustomResource(this, "MyCustomResource", {
  serviceToken: provider.serviceToken,
  properties: {
    ConfigValue: "my-config",
  },
});
```

**Common use cases:**

- Seed a database on first deployment
- Configure third-party resources (Datadog, PagerDuty)
- Look up existing resources (AMI IDs, VPC info)
- Run database migrations as part of deployment

---

### 14. **What is the CDK bootstrap and why is it needed?**

**Answer:** Bootstrapping creates the resources CDK needs to deploy into an AWS account/region — an S3 bucket for assets, an ECR repository for Docker images, and IAM roles for deployment.

```bash
# Bootstrap an account/region
cdk bootstrap aws://123456789012/us-east-1

# Bootstrap with custom qualifier (for multiple CDK apps in one account)
cdk bootstrap aws://123456789012/us-east-1 \
  --qualifier myapp \
  --toolkit-stack-name CDKToolkit-MyApp

# Bootstrap with trust for cross-account deployment
cdk bootstrap aws://222222222222/us-east-1 \
  --trust 111111111111 \
  --cloudformation-execution-policies "arn:aws:iam::aws:policy/AdministratorAccess"
```

**What bootstrapping creates:**

- `CDKToolkit` CloudFormation stack containing:
  - S3 bucket for file assets (Lambda code, etc.)
  - ECR repository for Docker assets
  - IAM roles: deploy role, CFN execution role, file/image publish roles
  - SSM parameter with bootstrap version

> **Important:** Bootstrap each account/region combination you deploy to. For CI/CD pipelines deploying cross-account, bootstrap target accounts with `--trust`.

---

### 15. **How do you manage multiple environments (dev/staging/prod) in CDK?**

**Answer:**

```ts
// Environment configuration
interface EnvironmentConfig {
  account: string;
  region: string;
  instanceSize: ec2.InstanceSize;
  minCapacity: number;
  maxCapacity: number;
  domainName?: string;
}

const environments: Record<string, EnvironmentConfig> = {
  dev: {
    account: "111111111111",
    region: "us-east-1",
    instanceSize: ec2.InstanceSize.SMALL,
    minCapacity: 1,
    maxCapacity: 2,
  },
  staging: {
    account: "222222222222",
    region: "us-east-1",
    instanceSize: ec2.InstanceSize.MEDIUM,
    minCapacity: 2,
    maxCapacity: 4,
  },
  prod: {
    account: "333333333333",
    region: "us-east-1",
    instanceSize: ec2.InstanceSize.XLARGE,
    minCapacity: 3,
    maxCapacity: 20,
    domainName: "api.myapp.com",
  },
};

// Use Stage for each environment
const stage = app.node.tryGetContext("stage") as string;
const config = environments[stage];

new MyAppStage(app, `MyApp-${stage}`, {
  env: { account: config.account, region: config.region },
  config,
});
```

```bash
# Deploy to specific environment
cdk deploy --context stage=dev
cdk deploy --context stage=prod
```

---

### 16. **What are escape hatches in CDK and when do you need them?**

**Answer:** Escape hatches let you access the underlying L1 (CloudFormation) resource from an L2 construct to set properties that L2 doesn't expose.

```ts
// L2 Bucket doesn't expose AnalyticsConfiguration
const bucket = new s3.Bucket(this, "DataBucket", {
  encryption: s3.BucketEncryption.S3_MANAGED,
});

// Escape hatch: access the underlying CfnBucket
const cfnBucket = bucket.node.defaultChild as s3.CfnBucket;

cfnBucket.analyticsConfigurations = [
  {
    id: "FullAnalytics",
    storageClassAnalysis: {
      dataExport: {
        destination: {
          bucketArn: analyticsBucket.bucketArn,
          format: "CSV",
          prefix: "analytics/",
        },
        outputSchemaVersion: "V_1",
      },
    },
  },
];

// Override any CFN property
cfnBucket.addPropertyOverride(
  "AccelerateConfiguration.AccelerationStatus",
  "Enabled",
);

// Remove a property
cfnBucket.addPropertyDeletionOverride("Tags");
```

---

### 17. **How do you deploy Lambda functions with CDK?**

**Answer:**

```ts
// Inline code (small functions only)
new lambda.Function(this, "SimpleHandler", {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: "index.handler",
  code: lambda.Code.fromInline(`
    exports.handler = async (event) => ({ statusCode: 200, body: 'Hello' });
  `),
});

// From local directory (bundled as zip)
new lambda.Function(this, "ApiHandler", {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: "index.handler",
  code: lambda.Code.fromAsset("lambda/api"),
  timeout: cdk.Duration.seconds(30),
  memorySize: 256,
  environment: {
    TABLE_NAME: table.tableName,
  },
});

// ✅ Best: NodejsFunction — auto-bundles with esbuild
import { NodejsFunction } from "aws-cdk-lib/aws-lambda-nodejs";

const fn = new NodejsFunction(this, "Handler", {
  entry: "src/handlers/orders.ts", // TypeScript source
  handler: "handler",
  runtime: lambda.Runtime.NODEJS_22_X,
  bundling: {
    minify: true,
    sourceMap: true,
    externalModules: ["@aws-sdk/*"], // Use Lambda-provided SDK
  },
  environment: { TABLE_NAME: table.tableName },
});

table.grantReadWriteData(fn); // CDK auto-generates IAM policy

// Docker-based Lambda
new lambda.DockerImageFunction(this, "MLInference", {
  code: lambda.DockerImageCode.fromImageAsset("./ml-model"),
  memorySize: 3008,
  timeout: cdk.Duration.minutes(5),
});
```

---

### 18. **How do you build a REST API with CDK?**

**Answer:**

```ts
import * as apigateway from "aws-cdk-lib/aws-apigateway";

// L3 pattern: Lambda-backed REST API
const api = new apigateway.LambdaRestApi(this, "Api", {
  handler: defaultHandler,
  proxy: false,
});

// Define routes
const orders = api.root.addResource("orders");
orders.addMethod("GET", new apigateway.LambdaIntegration(listOrdersFn));
orders.addMethod("POST", new apigateway.LambdaIntegration(createOrderFn));

const order = orders.addResource("{orderId}");
order.addMethod("GET", new apigateway.LambdaIntegration(getOrderFn));
order.addMethod("PUT", new apigateway.LambdaIntegration(updateOrderFn));

// Add API key and usage plan
const key = api.addApiKey("ApiKey");
const plan = api.addUsagePlan("UsagePlan", {
  throttle: { rateLimit: 100, burstLimit: 200 },
  quota: { limit: 10000, period: apigateway.Period.DAY },
});
plan.addApiKey(key);
plan.addApiStage({ stage: api.deploymentStage });

// Custom domain
new apigateway.DomainName(this, "Domain", {
  domainName: "api.myapp.com",
  certificate: cert,
  mapping: api,
});
```

---

### 19. **What are CloudFormation stack policies and rollback triggers?**

**Answer:**

**Stack policies** prevent unintentional updates to critical resources:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "Update:Replace",
      "Principal": "*",
      "Resource": "LogicalResourceId/ProductionDatabase"
    },
    {
      "Effect": "Allow",
      "Action": "Update:*",
      "Principal": "*",
      "Resource": "*"
    }
  ]
}
```

**Rollback triggers** — automatically roll back if a CloudWatch alarm fires during deployment:

```ts
new cdk.Stack(this, "AppStack", {
  // Roll back if error rate spikes during deploy
  // (monitor for 5 minutes after deployment completes)
});

// Can be set via CloudFormation CLI:
// aws cloudformation update-stack \
//   --rollback-configuration \
//   'RollbackTriggers=[{Arn=arn:aws:cloudwatch:...:alarm:HighErrors,Type=AWS::CloudWatch::Alarm}]'
```

---

### 20. **What are CDK best practices for production?**

**Answer:**

| Practice                                         | Why                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------- |
| **Explicit `env`** on every stack                | Prevents deploying to the wrong account/region                       |
| **`RemovalPolicy.RETAIN`** on stateful resources | Protects databases and buckets from accidental deletion              |
| **`terminationProtection: true`** on prod stacks | Prevents accidental stack deletion                                   |
| **Fine-grained stacks**                          | Separate stateful (DB) from stateless (Lambda) for safe updates      |
| **`cdk diff` before `cdk deploy`**               | Review changes, watch for REPLACE operations                         |
| **Use `cdk.context.json`** in version control    | Ensures deterministic deployments (cached lookups)                   |
| **Grant least-privilege IAM** via CDK methods    | `table.grantReadData(fn)` instead of `iam.PolicyStatement` wildcards |
| **Unit test your stacks**                        | Catch misconfigurations before deployment                            |
| **Use CDK Pipelines** for CI/CD                  | Self-mutating pipeline, multi-account deployments                    |
| **Pin CDK version**                              | Avoid breaking changes from surprise upgrades                        |

```ts
// ✅ Production-ready stack template
class ProductionStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props: cdk.StackProps) {
    super(scope, id, {
      ...props,
      terminationProtection: true, // Prevent accidental deletion
    });

    // Stateful resource with protection
    const table = new dynamodb.Table(this, "Orders", {
      partitionKey: { name: "pk", type: dynamodb.AttributeType.STRING },
      sortKey: { name: "sk", type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      pointInTimeRecovery: true,
      removalPolicy: cdk.RemovalPolicy.RETAIN,
    });

    // Lambda with least-privilege
    const fn = new NodejsFunction(this, "Handler", {
      entry: "src/handlers/orders.ts",
      runtime: lambda.Runtime.NODEJS_22_X,
    });
    table.grantReadWriteData(fn); // Not table.grant(fn, 'dynamodb:*')

    // Enforce security via Aspects
    Aspects.of(this).add(new RequiredTags());
    Aspects.of(this).add(new BucketEncryptionChecker());
  }
}
```
