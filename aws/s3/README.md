# Amazon S3 Interview Questions & Answers

A curated list of **Amazon S3 interview questions** with practical, production-grade answers. Covers storage classes, security, performance, replication, lifecycle management, and real-world patterns.

---

### 1. **What is Amazon S3?**

**Answer:** S3 (Simple Storage Service) is an object storage service offering virtually unlimited storage with 99.999999999% (11 nines) durability. Objects are stored in **buckets** and accessed via unique keys.

- **Max object size:** 5 TB
- **Max single PUT:** 5 GB (use multipart upload for larger objects)
- **Namespace:** Bucket names are globally unique across all AWS accounts

---

### 2. **What are S3 storage classes and how do you choose?**

**Answer:**

| Storage class           | Availability | Min duration | Retrieval fee    | Use case                                  |
| ----------------------- | ------------ | ------------ | ---------------- | ----------------------------------------- |
| S3 Standard             | 99.99%       | None         | None             | Frequently accessed data                  |
| S3 Intelligent-Tiering  | 99.9%        | None         | None             | Unknown access patterns                   |
| S3 Standard-IA          | 99.9%        | 30 days      | Per-GB fee       | Infrequently accessed (backups)           |
| S3 One Zone-IA          | 99.5%        | 30 days      | Per-GB fee       | Recreatable data, secondary backups       |
| S3 Glacier Instant      | 99.9%        | 90 days      | Per-GB fee       | Archive with instant access               |
| S3 Glacier Flexible     | 99.99%       | 90 days      | Per-GB + request | Archives (minutes to hours retrieval)     |
| S3 Glacier Deep Archive | 99.99%       | 180 days     | Per-GB + request | Long-term compliance (12-48 hr retrieval) |

```ts
// Upload with storage class
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({});

await s3.send(
  new PutObjectCommand({
    Bucket: "my-data-lake",
    Key: "logs/2024/01/access.log.gz",
    Body: logBuffer,
    StorageClass: "STANDARD_IA", // Infrequent access
  }),
);
```

---

### 3. **How do S3 lifecycle policies work?**

**Answer:** Lifecycle rules automatically transition objects between storage classes or expire (delete) them based on age or other criteria.

```json
{
  "Rules": [
    {
      "ID": "LogRetention",
      "Filter": { "Prefix": "logs/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER_IR" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 2555 },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 }
    }
  ]
}
```

> **Tip:** Always add `NoncurrentVersionExpiration` on versioned buckets to avoid unbounded storage costs from old versions.

---

### 4. **How does S3 versioning work?**

**Answer:** Versioning keeps all versions of an object (including deletes) in the same bucket. Once enabled, it can only be **suspended**, not disabled.

```ts
// ❌ Bad: Permanently deleting the wrong object
await s3.send(
  new DeleteObjectCommand({
    Bucket: "my-bucket",
    Key: "important-data.json",
    // This puts a delete marker — data is still there!
  }),
);

// ✅ Good: Restore a specific version
await s3.send(
  new CopyObjectCommand({
    Bucket: "my-bucket",
    CopySource: "my-bucket/important-data.json?versionId=abc123",
    Key: "important-data.json",
  }),
);
```

> **MFA Delete:** For extra protection, enable MFA Delete so that permanently deleting versions requires an MFA token.

---

### 5. **What are presigned URLs and when should you use them?**

**Answer:** Presigned URLs grant temporary, time-limited access to private S3 objects without making them public.

**Use cases:** File downloads for authenticated users, direct browser uploads, sharing temporary links.

```ts
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { GetObjectCommand, PutObjectCommand } from "@aws-sdk/client-s3";

// Generate a download URL (valid 15 minutes)
const downloadUrl = await getSignedUrl(
  s3,
  new GetObjectCommand({
    Bucket: "my-uploads",
    Key: `invoices/${userId}/invoice-2024.pdf`,
  }),
  { expiresIn: 900 },
);

// Generate an upload URL (direct browser upload)
const uploadUrl = await getSignedUrl(
  s3,
  new PutObjectCommand({
    Bucket: "my-uploads",
    Key: `uploads/${userId}/${crypto.randomUUID()}.jpg`,
    ContentType: "image/jpeg",
  }),
  { expiresIn: 300 },
);
```

```ts
// ❌ Bad: Making the bucket public for file sharing
{
  Effect: 'Allow',
  Principal: '*',
  Action: 's3:GetObject',
  Resource: 'arn:aws:s3:::my-bucket/*'
}

// ✅ Good: Use presigned URLs with short expiry
const url = await getSignedUrl(s3, new GetObjectCommand({
  Bucket: 'my-bucket', Key: 'file.pdf',
}), { expiresIn: 3600 });
```

---

### 6. **How does S3 encryption work?**

**Answer:**

| Type        | Key management    | Description                                                  |
| ----------- | ----------------- | ------------------------------------------------------------ |
| SSE-S3      | AWS-managed       | Default. S3 manages everything. Zero config.                 |
| SSE-KMS     | AWS KMS           | You control key policy, rotation, audit trail via CloudTrail |
| SSE-C       | Customer-provided | You send the key with each request                           |
| Client-side | Application       | Encrypt before upload. S3 stores ciphertext.                 |

> **Default:** Since Jan 2023, **all new S3 objects are encrypted with SSE-S3 by default**. No action needed for basic encryption.

```ts
// SSE-KMS encryption with custom key
await s3.send(
  new PutObjectCommand({
    Bucket: "sensitive-data",
    Key: "pii/customer-records.json",
    Body: JSON.stringify(records),
    ServerSideEncryption: "aws:kms",
    SSEKMSKeyId: "arn:aws:kms:us-east-1:123456789:key/my-key-id",
    BucketKeyEnabled: true, // Reduces KMS API calls (cost savings)
  }),
);
```

---

### 7. **What is the difference between bucket policies and IAM policies for S3?**

**Answer:**

| Feature       | Bucket policy                         | IAM policy                      |
| ------------- | ------------------------------------- | ------------------------------- |
| Attached to   | S3 bucket (resource-based)            | IAM user/role (identity-based)  |
| Cross-account | ✅ Can grant access to other accounts | Needs role assumption           |
| Public access | ✅ Can grant anonymous access         | ❌ No                           |
| Size limit    | 20 KB                                 | 6 KB (inline) / 10 KB (managed) |

**Best practice:** Use IAM policies for your own account's access control. Use bucket policies for cross-account access and bucket-level rules.

```json
// Bucket policy: Allow CloudFront to read objects (OAC)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-website/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789:distribution/E1ABC2DEF3"
        }
      }
    }
  ]
}
```

---

### 8. **How do you optimize S3 performance for high-throughput workloads?**

**Answer:**

S3 supports **5,500 GET/HEAD and 3,500 PUT/POST/DELETE requests per second per prefix**. Strategies:

1. **Parallelize requests** — Use multiple prefixes to distribute load
2. **Multipart upload** — Required for >5 GB, recommended for >100 MB
3. **S3 Transfer Acceleration** — Use CloudFront edge locations for faster uploads
4. **Byte-range fetches** — Download parts in parallel

```ts
import { Upload } from "@aws-sdk/lib-storage";

// Multipart upload with parallelism
const upload = new Upload({
  client: s3,
  params: {
    Bucket: "data-lake",
    Key: "large-dataset.parquet",
    Body: readStream,
  },
  queueSize: 4, // 4 concurrent parts
  partSize: 64 * 1024 * 1024, // 64 MB parts
});

upload.on("httpUploadProgress", (progress) => {
  console.log(`Uploaded: ${progress.loaded}/${progress.total}`);
});

await upload.done();
```

```ts
// ❌ Bad: Sequential key pattern (causes hot partitions, pre-2018 issue)
`logs/2024-01-15/file001.gz``logs/2024-01-15/file002.gz`
// ✅ Good: S3 now auto-partitions, but multiple prefixes still help at extreme scale
`logs/shard-01/2024-01-15/file001.gz``logs/shard-02/2024-01-15/file002.gz`;
```

---

### 9. **What is S3 Event Notifications?**

**Answer:** S3 can trigger notifications when objects are created, deleted, or restored. Targets include Lambda, SQS, SNS, and EventBridge.

```ts
// CDK: Trigger Lambda on new uploads
import * as s3 from "aws-cdk-lib/aws-s3";
import * as s3n from "aws-cdk-lib/aws-s3-notifications";

const bucket = new s3.Bucket(this, "Uploads");

bucket.addEventNotification(
  s3.EventType.OBJECT_CREATED,
  new s3n.LambdaDestination(processImageFn),
  { prefix: "images/", suffix: ".jpg" },
);
```

> **EventBridge integration:** Since 2021, you can route S3 events to EventBridge for advanced filtering, multiple targets, and replay. Enable it on the bucket.

---

### 10. **What is S3 Select and when should you use it?**

**Answer:** S3 Select lets you query CSV, JSON, and Parquet files using SQL **without downloading the entire object**. This reduces data transfer and speeds up queries.

```ts
import { SelectObjectContentCommand } from "@aws-sdk/client-s3";

const response = await s3.send(
  new SelectObjectContentCommand({
    Bucket: "data-lake",
    Key: "sales/2024-Q1.csv",
    ExpressionType: "SQL",
    Expression:
      "SELECT s.product, s.revenue FROM s3object s WHERE s.revenue > '10000'",
    InputSerialization: { CSV: { FileHeaderInfo: "USE" } },
    OutputSerialization: { JSON: {} },
  }),
);
```

> **For large-scale analytics**, prefer **Athena** (SQL over S3 at scale) or **S3 Object Lambda** (transform objects on-the-fly during GET requests).

---

### 11. **How does S3 replication work?**

**Answer:**

| Type                           | Scope             | Use case                                              |
| ------------------------------ | ----------------- | ----------------------------------------------------- |
| Cross-Region Replication (CRR) | Different regions | Disaster recovery, compliance, lower latency          |
| Same-Region Replication (SRR)  | Same region       | Log aggregation, prod-to-test sync, compliance copies |

**Requirements:** Versioning must be enabled on both source and destination buckets.

```ts
// CDK: Cross-region replication
const destBucket = new s3.Bucket(this, "DestBucket", {
  versioned: true,
});

const sourceBucket = new s3.Bucket(this, "SourceBucket", {
  versioned: true,
  replication: {
    rules: [
      {
        destination: { bucket: destBucket },
        filter: { prefix: "critical/" },
      },
    ],
  },
});
```

> **Note:** Replication does NOT replicate existing objects (only new objects after the rule is created). Use **S3 Batch Replication** for existing objects.

---

### 12. **What are S3 Access Points?**

**Answer:** Access Points are named network endpoints attached to a bucket, each with its own permissions policy. They simplify managing access to shared datasets.

**Why use them:**

- Replace complex bucket policies (one per access point instead of one giant policy)
- Restrict VPC-only access per access point
- Each application/team gets its own access point

```json
// Access point policy: data-science team can only read analytics/ prefix
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::123456789:role/DataScienceRole" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:us-east-1:123456789:accesspoint/analytics-ap/object/analytics/*",
        "arn:aws:s3:us-east-1:123456789:accesspoint/analytics-ap"
      ]
    }
  ]
}
```

---

### 13. **How do you block public access on S3?**

**Answer:** S3 Block Public Access provides four settings that override bucket policies and ACLs:

1. **BlockPublicAcls** — Block new public ACLs
2. **IgnorePublicAcls** — Ignore existing public ACLs
3. **BlockPublicPolicy** — Block new public bucket policies
4. **RestrictPublicBuckets** — Restrict access via public policies to AWS service principals only

```ts
// ✅ Set at the account level (recommended)
import {
  S3ControlClient,
  PutPublicAccessBlockCommand,
} from "@aws-sdk/client-s3-control";

await s3Control.send(
  new PutPublicAccessBlockCommand({
    AccountId: "123456789012",
    PublicAccessBlockConfiguration: {
      BlockPublicAcls: true,
      IgnorePublicAcls: true,
      BlockPublicPolicy: true,
      RestrictPublicBuckets: true,
    },
  }),
);
```

> **Best practice:** Enable all four Block Public Access settings at the **account level**. Only disable on specific buckets that genuinely need public access (e.g., static website hosting without CloudFront).

---

### 14. **How do you set up S3 static website hosting?**

**Answer:**

```ts
// ❌ Less secure: Direct S3 website hosting
const bucket = new s3.Bucket(this, "Website", {
  websiteIndexDocument: "index.html",
  websiteErrorDocument: "error.html",
  publicReadAccess: true, // Requires disabling Block Public Access
});

// ✅ Better: CloudFront + S3 with OAC (Origin Access Control)
const bucket = new s3.Bucket(this, "Website", {
  blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
});

const distribution = new cloudfront.Distribution(this, "CDN", {
  defaultBehavior: {
    origin: origins.S3BucketOrigin.withOriginAccessControl(bucket),
    viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
  },
  defaultRootObject: "index.html",
  errorResponses: [
    {
      httpStatus: 403,
      responsePagePath: "/index.html",
      responseHttpStatus: 200, // SPA routing
    },
  ],
});
```

---

### 15. **What is S3 Object Lock and when do you use it?**

**Answer:** Object Lock enforces **WORM (Write Once Read Many)** — prevents objects from being deleted or overwritten for a set retention period.

| Mode       | Can be overridden?                                           | Use case                           |
| ---------- | ------------------------------------------------------------ | ---------------------------------- |
| Governance | Yes (with special permission `s3:BypassGovernanceRetention`) | Internal data protection           |
| Compliance | ❌ No — not even root account                                | Regulatory compliance (SEC, HIPAA) |
| Legal Hold | Indefinite hold until removed                                | E-discovery, litigation            |

> **Note:** Object Lock requires versioning. Must be enabled at bucket creation.

---

### 16. **How do you monitor S3 costs and optimize spending?**

**Answer:**

1. **S3 Storage Lens** — Dashboard showing usage, activity, and cost optimization recommendations across all buckets
2. **Intelligent-Tiering** — Automatically moves objects between tiers (no retrieval fees)
3. **Lifecycle policies** — Transition to cheaper storage classes
4. **Incomplete multipart cleanup** — Delete abandoned multipart uploads (common hidden cost!)

```json
// Lifecycle rule: clean up incomplete multipart uploads
{
  "Rules": [
    {
      "ID": "CleanupMultipart",
      "Filter": {},
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

---

### 17. **How do you securely share S3 data across AWS accounts?**

**Answer:**

| Method                        | Complexity | Best for                                |
| ----------------------------- | ---------- | --------------------------------------- |
| Bucket policy                 | Low        | Simple cross-account read/write         |
| Access Points                 | Medium     | Multi-team shared data lakes            |
| S3 Access Grants              | Medium     | Map identities from corporate directory |
| AWS RAM (S3 Multi-Region AP)  | High       | Multi-region access patterns            |
| Cross-account role assumption | Medium     | Full control, temporary credentials     |

```json
// Bucket policy: Allow another account's role to read
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::987654321:role/DataConsumerRole" },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::shared-data-bucket",
        "arn:aws:s3:::shared-data-bucket/shared/*"
      ]
    }
  ]
}
```

---

### 18. **What is S3 Batch Operations?**

**Answer:** S3 Batch Operations performs large-scale actions on billions of objects with a single API call. Provides progress tracking, completion reports, and retries.

**Supported operations:**

- Copy objects
- Invoke Lambda per object
- Replace tags or ACLs
- Restore from Glacier
- S3 Object Lock retention changes

```bash
# Generate inventory, then create a batch job
aws s3control create-job \
  --account-id 123456789012 \
  --operation '{"S3PutObjectTagging":{"TagSet":[{"Key":"classified","Value":"true"}]}}' \
  --manifest '{"Spec":{"Format":"S3InventoryReport_CSV_20211130"},"Location":{"ObjectArn":"arn:aws:s3:::inventory-bucket/manifest.json","ETag":"abc123"}}' \
  --report '{"Bucket":"arn:aws:s3:::report-bucket","Prefix":"batch-reports/","Format":"Report_CSV_20180820","Enabled":true,"ReportScope":"AllTasks"}' \
  --priority 1 \
  --role-arn arn:aws:iam::123456789012:role/BatchOperationsRole
```

---

### 19. **How do you implement a secure data lake on S3?**

**Answer:**

Key architecture decisions:

```
Raw Zone (landing)     → Curated Zone (transformed)  → Analytics Zone (aggregated)
 s3://datalake-raw/     s3://datalake-curated/         s3://datalake-analytics/
 ├── Parquet/JSON       ├── Parquet (columnar)         ├── Pre-computed aggregates
 ├── SSE-KMS            ├── SSE-KMS                    ├── SSE-KMS
 └── Lifecycle: 90d→IA  └── Lifecycle: 1yr→Glacier     └── Access via Athena/Redshift
```

**Security layers:**

1. Block Public Access on all buckets
2. SSE-KMS for encryption (separate keys per zone)
3. S3 Access Points per team
4. Lake Formation for fine-grained column/row-level access
5. VPC endpoints (Gateway) — all S3 traffic stays in AWS network
6. CloudTrail data events for audit trail

---

### 20. **What are the S3 consistency guarantees?**

**Answer:** Since December 2020, S3 provides **strong read-after-write consistency** for all operations:

- PUT a new object → immediately readable
- Overwrite an object → immediately returns the new version
- DELETE an object → immediately returns 404
- LIST after PUT → immediately includes the new object

This eliminated the previous eventual consistency behavior for overwrite PUTs and DELETEs. No application changes or extra cost needed.
