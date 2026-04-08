# AWS IAM Interview Questions & Answers

A curated list of **AWS IAM (Identity and Access Management) interview questions** with practical, production-grade answers. Covers policies, roles, least privilege, cross-account access, federation, and security best practices.

---

### 1. **What is AWS IAM?**

**Answer:** IAM is the service that controls **who** (authentication) can do **what** (authorization) on **which** AWS resources. It's global — IAM entities are not region-specific.

Core components:

- **Users** — Long-term credentials for people or applications
- **Groups** — Collection of users with shared permissions
- **Roles** — Temporary credentials assumed by users, services, or external identities
- **Policies** — JSON documents defining permissions

---

### 2. **What is the IAM policy evaluation logic?**

**Answer:** AWS evaluates policies in this order:

1. **Explicit Deny** → If any policy says Deny, access is denied (always wins)
2. **SCPs (Organization)** → Must allow the action (if AWS Organizations is used)
3. **Resource-based policy** → Bucket policy, KMS key policy, etc.
4. **Permission boundary** → Sets maximum permissions for the entity
5. **Session policy** → For assumed roles or federated sessions
6. **Identity-based policy** → Must explicitly Allow

> **Default:** Everything is **denied by default**. You must explicitly grant permissions.

```
Is there an explicit Deny?  → YES → ❌ DENIED
Is there an SCP Allow?      → NO  → ❌ DENIED
Is there an Allow?          → NO  → ❌ DENIED (implicit deny)
                            → YES → ✅ ALLOWED
```

---

### 3. **What are the different IAM policy types?**

**Answer:**

| Policy type             | Attached to                 | Use case                                             |
| ----------------------- | --------------------------- | ---------------------------------------------------- |
| **AWS Managed**         | IAM entities                | Pre-built by AWS (e.g., `ReadOnlyAccess`)            |
| **Customer Managed**    | IAM entities                | Your custom policies, reusable                       |
| **Inline**              | Single entity               | Tightly coupled to one user/role (avoid if possible) |
| **Resource-based**      | AWS resource                | Cross-account access (S3, SQS, Lambda, KMS)          |
| **Permission Boundary** | IAM user/role               | Maximum permission ceiling                           |
| **SCP**                 | AWS Organization OU/account | Guardrails across accounts                           |
| **Session policy**      | STS session                 | Further restrict assumed role                        |

---

### 4. **What is the difference between IAM roles and IAM users?**

**Answer:**

| Feature       | IAM User                          | IAM Role                                    |
| ------------- | --------------------------------- | ------------------------------------------- |
| Credentials   | Long-term (access keys, password) | Temporary (STS tokens, 1-12 hrs)            |
| Used by       | People, CI/CD (legacy)            | Services, cross-account, federated users    |
| Best practice | Avoid for applications            | ✅ Prefer for everything                    |
| Rotation      | Manual key rotation needed        | Automatic (new credentials each assumption) |

```ts
// ❌ Bad: Hardcoded access keys (security risk)
const s3 = new S3Client({
  credentials: {
    accessKeyId: "AKIAIOSFODNN7EXAMPLE",
    secretAccessKey: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
  },
});

// ✅ Good: Use IAM role (SDK auto-discovers credentials)
const s3 = new S3Client({});
// On EC2: uses instance profile role
// On Lambda: uses execution role
// On ECS: uses task role
// Locally: uses ~/.aws/credentials or SSO
```

---

### 5. **What is the principle of least privilege and how do you implement it?**

**Answer:** Grant only the minimum permissions needed to accomplish a task.

```json
// ❌ Bad: Overly permissive
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}

// ✅ Good: Specific actions, specific resources
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::my-app-uploads/user-files/*"
}

// ✅ Better: Add conditions
{
  "Effect": "Allow",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-app-uploads/user-files/*",
  "Condition": {
    "StringEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    },
    "StringLike": {
      "s3:prefix": ["${aws:PrincipalTag/team}/*"]
    }
  }
}
```

**Tools to achieve least privilege:**

- **IAM Access Analyzer** — Analyzes resource policies for external access
- **Access Analyzer policy generation** — Generates policies from CloudTrail activity
- **IAM last accessed** — Shows when permissions were last used (remove unused)

---

### 6. **What is cross-account access and how do you set it up?**

**Answer:** Cross-account access lets a principal in Account A access resources in Account B.

**Method 1: Cross-account role assumption (recommended)**

```json
// Account B (resource owner): Trust policy on the role
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111111111111:role/AppRole" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:ExternalId": "unique-external-id-12345" }
      }
    }
  ]
}
```

```ts
// Account A (caller): Assume the role in Account B
import { STSClient, AssumeRoleCommand } from "@aws-sdk/client-sts";

const sts = new STSClient({});
const { Credentials } = await sts.send(
  new AssumeRoleCommand({
    RoleArn: "arn:aws:iam::222222222222:role/CrossAccountRole",
    RoleSessionName: "my-app-session",
    ExternalId: "unique-external-id-12345",
  }),
);

// Use temporary credentials
const s3InAccountB = new S3Client({
  credentials: {
    accessKeyId: Credentials!.AccessKeyId!,
    secretAccessKey: Credentials!.SecretAccessKey!,
    sessionToken: Credentials!.SessionToken!,
  },
});
```

> **ExternalId:** Always use when granting cross-account access to third parties to prevent the **confused deputy problem**.

---

### 7. **What are IAM permission boundaries?**

**Answer:** Permission boundaries set the **maximum** permissions an IAM entity can have. The effective permissions are the **intersection** of identity-based policies and the boundary.

**Use case:** Allow developers to create IAM roles for their Lambda functions, but restrict what those roles can do.

```json
// Permission boundary: max permissions for developer-created roles
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "logs:*",
        "xray:*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": ["iam:*", "organizations:*", "sts:*"],
      "Resource": "*"
    }
  ]
}
```

```json
// Allow developers to create roles, but only with the boundary attached
{
  "Effect": "Allow",
  "Action": "iam:CreateRole",
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "iam:PermissionsBoundary": "arn:aws:iam::123456789:policy/DeveloperBoundary"
    }
  }
}
```

---

### 8. **What are Service Control Policies (SCPs)?**

**Answer:** SCPs are guardrails in **AWS Organizations** that set maximum permissions for all accounts in an OU. They don't grant permissions — they restrict what identity policies can allow.

```json
// SCP: Deny actions outside approved regions
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/OrganizationAdmin"
        }
      }
    }
  ]
}
```

```json
// SCP: Prevent disabling CloudTrail or GuardDuty
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### 9. **What is IAM Identity Center (formerly AWS SSO)?**

**Answer:** IAM Identity Center provides centralized workforce access to multiple AWS accounts and business applications using a single sign-on experience.

**Key features:**

- Integration with external IdPs (Okta, Azure AD, Google Workspace)
- Multi-account access via **permission sets**
- Automatic credential rotation
- SAML 2.0 and OIDC support

**Flow:**

1. User authenticates with corporate IdP
2. Identity Center assigns temporary credentials
3. User accesses AWS accounts via permission sets
4. CloudTrail logs the federated identity

> **Best practice:** Use IAM Identity Center instead of creating individual IAM users across accounts.

---

### 10. **What are IAM condition keys and how do you use them?**

**Answer:** Conditions add fine-grained control to policy statements. Common condition keys:

| Condition key                | Purpose                                   |
| ---------------------------- | ----------------------------------------- |
| `aws:SourceIp`               | Restrict by IP range                      |
| `aws:PrincipalOrgID`         | Restrict to your organization             |
| `aws:RequestedRegion`        | Restrict by region                        |
| `aws:PrincipalTag/key`       | ABAC — attribute-based access             |
| `aws:MultiFactorAuthPresent` | Require MFA                               |
| `aws:CalledVia`              | Must be called through a specific service |
| `ec2:ResourceTag/key`        | Tag-based resource access                 |

```json
// Require MFA for sensitive operations
{
  "Effect": "Deny",
  "Action": ["iam:DeactivateMFADevice", "iam:DeleteUser", "s3:DeleteBucket"],
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```

```json
// ABAC: Users can only access resources tagged with their team
{
  "Effect": "Allow",
  "Action": ["ec2:StartInstances", "ec2:StopInstances"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "ec2:ResourceTag/team": "${aws:PrincipalTag/team}"
    }
  }
}
```

---

### 11. **What is ABAC (Attribute-Based Access Control)?**

**Answer:** ABAC uses tags on principals and resources to control access, instead of defining separate policies per resource.

**Advantages over RBAC:**

- Scales without creating new policies per resource
- New resources automatically accessible if tags match
- Fewer policies to manage

```json
// Single policy that works for all teams
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::shared-data/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/project": "${aws:PrincipalTag/project}"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::shared-data/*",
      "Condition": {
        "StringEquals": {
          "s3:RequestObjectTag/project": "${aws:PrincipalTag/project}"
        }
      }
    }
  ]
}
```

---

### 12. **What are service-linked roles?**

**Answer:** Service-linked roles are pre-defined IAM roles that AWS services create and manage. You cannot modify their permissions.

**Examples:**

- `AWSServiceRoleForElasticLoadBalancing` — ELB managing ENIs
- `AWSServiceRoleForAutoScaling` — ASG launching instances
- `AWSServiceRoleForAmazonGuardDuty` — GuardDuty reading logs

```ts
// Some services create them automatically; others need explicit creation
import { IAMClient, CreateServiceLinkedRoleCommand } from "@aws-sdk/client-iam";

await iam.send(
  new CreateServiceLinkedRoleCommand({
    AWSServiceName: "elasticloadbalancing.amazonaws.com",
  }),
);
```

> You can delete service-linked roles, but only if the service is no longer using them.

---

### 13. **How do you audit IAM permissions and detect over-privileged identities?**

**Answer:**

| Tool                    | What it does                                                                              |
| ----------------------- | ----------------------------------------------------------------------------------------- |
| **IAM Access Analyzer** | Finds resources shared externally, validates policies, generates least-privilege policies |
| **Credential Report**   | CSV of all users — passwords, keys, MFA, last used                                        |
| **Access Advisor**      | Shows last-accessed time for each service permission                                      |
| **CloudTrail**          | Logs every API call with caller identity                                                  |
| **Config Rules**        | `iam-user-no-policies-check`, `mfa-enabled-for-iam-console-access`                        |

```bash
# Generate credential report
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 -d > creds.csv

# Find unused access keys (not used in 90+ days)
aws iam list-users --query 'Users[].UserName' --output text | while read user; do
  aws iam list-access-keys --user-name "$user" \
    --query "AccessKeyMetadata[?Status=='Active'].AccessKeyId" --output text
done
```

```ts
// Access Analyzer: generate policy from CloudTrail activity
import {
  AccessAnalyzerClient,
  StartPolicyGenerationCommand,
} from "@aws-sdk/client-accessanalyzer";

const analyzer = new AccessAnalyzerClient({});
await analyzer.send(
  new StartPolicyGenerationCommand({
    policyGenerationDetails: {
      principalArn: "arn:aws:iam::123456789:role/MyAppRole",
    },
    cloudTrailDetails: {
      trails: [
        {
          cloudTrailArn: "arn:aws:cloudtrail:us-east-1:123456789:trail/main",
          allRegions: true,
        },
      ],
      accessRole: "arn:aws:iam::123456789:role/AccessAnalyzerRole",
      startTime: new Date("2024-01-01"),
      endTime: new Date(),
    },
  }),
);
```

---

### 14. **How do you secure the root account?**

**Answer:**

1. **Enable MFA** — Use a hardware security key (FIDO2) or virtual MFA
2. **Delete access keys** — Root should never have programmatic access
3. **Use SCPs** — Deny root actions at the organization level (except for root-only tasks)
4. **Don't use root** — Use IAM Identity Center for daily operations
5. **Enable CloudTrail** — Monitor any root usage
6. **Set up alerting** — CloudWatch alarm on root API calls

```json
// CloudWatch metric filter: alert on root account usage
{
  "filterPattern": "{ $.userIdentity.type = \"Root\" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != \"AwsServiceEvent\" }"
}
```

> **Root-only tasks** (can't be delegated): Changing account settings, restoring IAM permissions, closing the account, enabling S3 MFA Delete, certain billing operations.

---

### 15. **What is the confused deputy problem?**

**Answer:** A confused deputy attack occurs when a less-privileged entity tricks a more-privileged service into misusing its permissions.

**Example:** An attacker uses a third-party SaaS service to access your S3 bucket by guessing your role ARN.

**Solution:** Use `ExternalId` and `aws:SourceArn` / `aws:SourceAccount` conditions.

```json
// ❌ Bad: Any account could assume this role if they know the ARN
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::999999999999:root" },
  "Action": "sts:AssumeRole"
}

// ✅ Good: ExternalId prevents confused deputy
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::999999999999:root" },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "customer-unique-secret-abc123"
    }
  }
}
```

```json
// For AWS services: use SourceArn/SourceAccount
{
  "Effect": "Allow",
  "Principal": { "Service": "sns.amazonaws.com" },
  "Action": "sqs:SendMessage",
  "Resource": "arn:aws:sqs:us-east-1:123456789:my-queue",
  "Condition": {
    "ArnEquals": {
      "aws:SourceArn": "arn:aws:sns:us-east-1:123456789:my-topic"
    }
  }
}
```

---

### 16. **How does IAM work with temporary credentials (STS)?**

**Answer:** AWS Security Token Service (STS) issues temporary credentials (access key + secret key + session token) that expire automatically.

| STS API                     | Use case                                    |
| --------------------------- | ------------------------------------------- |
| `AssumeRole`                | Cross-account access, service assume        |
| `AssumeRoleWithSAML`        | SAML federation (enterprise IdP)            |
| `AssumeRoleWithWebIdentity` | Web/mobile apps (Cognito, Google, Facebook) |
| `GetSessionToken`           | MFA-protected API access                    |
| `GetFederationToken`        | Custom federation broker                    |

```ts
// Common pattern: assume role for cross-account DynamoDB access
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { fromTemporaryCredentials } from "@aws-sdk/credential-providers";

const dynamodb = new DynamoDBClient({
  credentials: fromTemporaryCredentials({
    params: {
      RoleArn: "arn:aws:iam::222222222222:role/DynamoDBReadRole",
      RoleSessionName: "cross-account-read",
      DurationSeconds: 3600,
    },
  }),
});
```

---

### 17. **What are IAM Roles Anywhere?**

**Answer:** IAM Roles Anywhere lets **on-premises workloads** (servers, containers) obtain temporary AWS credentials using X.509 certificates, without long-term access keys.

**Flow:**

1. Register a Certificate Authority (CA) as a trust anchor
2. Workload presents its certificate
3. IAM Roles Anywhere validates and returns temporary credentials
4. Workload uses credentials like any IAM role

**Use case:** Hybrid cloud — on-prem servers accessing S3, DynamoDB, etc., without storing AWS access keys.

---

### 18. **How do you implement security guardrails for a multi-account organization?**

**Answer:**

```
AWS Organization
├── Root (SCP: deny leaving org, deny disabling security)
│   ├── Security OU
│   │   ├── Audit account (CloudTrail, Config, GuardDuty delegated admin)
│   │   └── Log archive account (centralized logging, S3 Object Lock)
│   ├── Sandbox OU (SCP: limited services, limited regions)
│   ├── Dev OU (SCP: no production data access)
│   ├── Staging OU
│   └── Production OU (SCP: strict, deny delete operations without MFA)
```

**Layered controls:**

1. **SCPs** — Preventive guardrails (deny regions, deny certain services)
2. **Permission Boundaries** — Cap permissions for delegated role creation
3. **AWS Config Rules** — Detective guardrails (flag non-compliance)
4. **IAM Access Analyzer** — Find unintended external access
5. **CloudTrail** — Audit trail for all API calls

---

### 19. **What are some IAM security best practices?**

**Answer:**

| Practice                             | Why                                                |
| ------------------------------------ | -------------------------------------------------- |
| Use IAM roles, not users/access keys | Temporary credentials, auto-rotation               |
| Enable MFA everywhere                | Protects against credential theft                  |
| Use permission boundaries            | Safe delegation of IAM management                  |
| Apply SCPs at org level              | Prevent risky actions across all accounts          |
| Use IAM Access Analyzer              | Detect external access and unused permissions      |
| Tag IAM roles                        | Enable ABAC, cost tracking, organization           |
| Rotate credentials                   | If access keys are necessary, rotate every 90 days |
| Separate environments                | Different accounts for dev/staging/prod            |
| Use VPC endpoints                    | Keep API calls within AWS network                  |
| Monitor with CloudTrail              | Audit all IAM and STS activity                     |

---

### 20. **How do you implement identity federation for a web application?**

**Answer:** Use **Amazon Cognito** for customer-facing apps or **IAM Identity Center** for workforce access.

```ts
// Web app with Cognito User Pool + Identity Pool
// 1. User signs in → receives JWT tokens from Cognito User Pool
// 2. Exchange token for temporary AWS credentials via Identity Pool

import {
  CognitoIdentityClient,
  GetIdCommand,
  GetCredentialsForIdentityCommand,
} from "@aws-sdk/client-cognito-identity";

const cognito = new CognitoIdentityClient({});

// Exchange Cognito token for AWS credentials
const { IdentityId } = await cognito.send(
  new GetIdCommand({
    IdentityPoolId: "us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    Logins: {
      "cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123": idToken,
    },
  }),
);

const { Credentials } = await cognito.send(
  new GetCredentialsForIdentityCommand({
    IdentityId: IdentityId!,
    Logins: {
      "cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123": idToken,
    },
  }),
);

// User now has scoped AWS credentials (e.g., access their own S3 prefix)
```

```json
// Cognito Identity Pool: authenticated role policy
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::user-data/${cognito-identity.amazonaws.com:sub}/*"
}
```

> This scopes each user to their own S3 prefix using the Cognito identity ID as a dynamic variable.
