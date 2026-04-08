# Amazon EC2 Interview Questions & Answers

A curated list of **Amazon EC2 interview questions** with practical, production-grade answers. Covers instance types, networking, storage, Auto Scaling, security, pricing, and real-world deployment patterns.

---

### 1. **What is Amazon EC2?**

**Answer:** EC2 (Elastic Compute Cloud) provides resizable virtual machines (instances) in the cloud. You choose the OS, instance type (CPU/RAM/GPU), storage, and networking. It's the foundational compute service of AWS.

---

### 2. **What are the different EC2 instance families and when do you use each?**

**Answer:**

| Family                | Prefix                | Optimized for        | Example use case                       |
| --------------------- | --------------------- | -------------------- | -------------------------------------- |
| General Purpose       | `t3`, `m7i`, `m7g`    | Balanced CPU/memory  | Web servers, APIs, microservices       |
| Compute Optimized     | `c7i`, `c7g`          | High-performance CPU | Batch processing, ML inference, gaming |
| Memory Optimized      | `r7i`, `r7g`, `x2idn` | Large memory         | In-memory databases, Redis, SAP        |
| Storage Optimized     | `i4i`, `d3`           | High sequential I/O  | Data warehouses, HDFS, Cassandra       |
| Accelerated Computing | `p5`, `g5`, `inf2`    | GPU/AI accelerators  | ML training, video rendering, GenAI    |

> **Note:** The `g` suffix (e.g., `m7g`, `c7g`) means Graviton (ARM) — up to 40% better price-performance than x86.

---

### 3. **What are the EC2 pricing models?**

**Answer:**

| Model              | Discount          | Commitment                | Best for                                |
| ------------------ | ----------------- | ------------------------- | --------------------------------------- |
| On-Demand          | None (base price) | None                      | Unpredictable workloads, testing        |
| Savings Plans      | Up to 72%         | 1 or 3 year               | Steady-state baseline workloads         |
| Reserved Instances | Up to 72%         | 1 or 3 year               | Fixed instance type/region              |
| Spot Instances     | Up to 90%         | None (can be interrupted) | Batch processing, CI/CD, fault-tolerant |
| Dedicated Hosts    | Varies            | Optional                  | License compliance, regulated workloads |

```ts
// Request Spot Instances via SDK
import { EC2Client, RequestSpotInstancesCommand } from "@aws-sdk/client-ec2";

const ec2 = new EC2Client({});

await ec2.send(
  new RequestSpotInstancesCommand({
    SpotPrice: "0.05",
    InstanceCount: 5,
    LaunchSpecification: {
      ImageId: "ami-0abcdef1234567890",
      InstanceType: "c7g.large",
      SecurityGroupIds: ["sg-12345"],
    },
  }),
);
```

---

### 4. **What is the difference between stopping and terminating an EC2 instance?**

**Answer:**

| Action        | EBS root volume              | Instance store | Public IP                    | Billing                                 |
| ------------- | ---------------------------- | -------------- | ---------------------------- | --------------------------------------- |
| **Stop**      | Preserved                    | Lost           | Released (unless Elastic IP) | No compute charge; EBS charges continue |
| **Terminate** | Deleted (by default)         | Lost           | Released                     | Stops all charges                       |
| **Hibernate** | Preserved (RAM saved to EBS) | Lost           | Released                     | No compute charge                       |

```ts
// ❌ Bad: Terminating when you meant to stop (data loss!)
await ec2.send(new TerminateInstancesCommand({ InstanceIds: ["i-1234"] }));

// ✅ Good: Enable termination protection for production instances
await ec2.send(
  new ModifyInstanceAttributeCommand({
    InstanceId: "i-1234",
    DisableApiTermination: { Value: true },
  }),
);
```

---

### 5. **What is an AMI (Amazon Machine Image)?**

**Answer:** An AMI is a template containing the OS, application code, libraries, and configuration needed to launch an instance. It includes:

- Root volume snapshot (OS + apps)
- Launch permissions
- Block device mapping

**Golden AMI pattern:** Pre-bake your application, dependencies, and configs into a custom AMI. This reduces startup time from minutes to seconds.

```bash
# Create a custom AMI from a running instance
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "myapp-v2.1.0-$(date +%Y%m%d)" \
  --description "MyApp v2.1.0 with Node.js 22 and dependencies" \
  --no-reboot
```

---

### 6. **What are security groups vs NACLs?**

**Answer:**

| Feature    | Security Group                         | NACL                                   |
| ---------- | -------------------------------------- | -------------------------------------- |
| Level      | Instance (ENI)                         | Subnet                                 |
| State      | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules      | Allow only                             | Allow and Deny                         |
| Evaluation | All rules evaluated                    | Rules evaluated in order (by number)   |
| Default    | Deny all inbound, allow all outbound   | Allow all inbound and outbound         |

```ts
// ❌ Bad: Overly permissive security group
{
  IpProtocol: '-1',             // All traffic
  CidrIp: '0.0.0.0/0',         // From anywhere
}

// ✅ Good: Principle of least privilege
{
  IpProtocol: 'tcp',
  FromPort: 443,
  ToPort: 443,
  CidrIp: '10.0.0.0/16',       // Only from VPC CIDR
}

// ✅ Better: Reference another security group
{
  IpProtocol: 'tcp',
  FromPort: 5432,
  ToPort: 5432,
  SourceSecurityGroupId: 'sg-app-servers',  // Only from app tier
}
```

---

### 7. **What is an Elastic IP and when should you use it?**

**Answer:** An Elastic IP (EIP) is a static public IPv4 address. Unlike default public IPs, it persists when you stop/start an instance.

**Use when:** You need a fixed public IP (e.g., DNS A record, firewall allow-listing).
**Avoid when:** You use a load balancer (ALB/NLB handles public-facing traffic).

> **Note:** AWS charges for unused Elastic IPs and for each additional EIP beyond one per running instance. As of Feb 2024, AWS also charges $0.005/hr for all public IPv4 addresses (even in-use ones).

---

### 8. **How does EC2 Auto Scaling work?**

**Answer:** Auto Scaling adjusts the number of EC2 instances based on demand.

Components:

- **Launch Template** — defines instance configuration (AMI, type, security groups)
- **Auto Scaling Group (ASG)** — manages min/max/desired instance count
- **Scaling Policies** — rules for when to scale

```ts
// CDK: Auto Scaling Group with target tracking
const asg = new autoscaling.AutoScalingGroup(this, "ASG", {
  vpc,
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.C7G,
    ec2.InstanceSize.LARGE,
  ),
  machineImage: ec2.MachineImage.latestAmazonLinux2023({
    cpuType: ec2.AmazonLinuxCpuType.ARM_64,
  }),
  minCapacity: 2,
  maxCapacity: 20,
  healthCheck: autoscaling.HealthCheck.elb({ grace: cdk.Duration.seconds(60) }),
});

// Target tracking: maintain 60% average CPU
asg.scaleOnCpuUtilization("CpuScaling", {
  targetUtilizationPercent: 60,
  cooldown: cdk.Duration.seconds(300),
});

// Scheduled scaling: scale up before peak hours
asg.scaleOnSchedule("PeakHours", {
  schedule: autoscaling.Schedule.cron({ hour: "8", minute: "0" }),
  minCapacity: 10,
});

asg.scaleOnSchedule("OffPeak", {
  schedule: autoscaling.Schedule.cron({ hour: "20", minute: "0" }),
  minCapacity: 2,
});
```

---

### 9. **What is a Launch Template vs Launch Configuration?**

**Answer:** Launch Templates are the newer, recommended approach. Launch Configurations are **legacy and cannot be modified**.

| Feature            | Launch Template    | Launch Configuration    |
| ------------------ | ------------------ | ----------------------- |
| Versioning         | ✅ Yes             | ❌ No (create new one)  |
| Spot support       | ✅ Mixed instances | Limited                 |
| Network interfaces | ✅ Multiple        | Single                  |
| T2/T3 Unlimited    | ✅ Yes             | ❌ No                   |
| Status             | **Current**        | **Legacy** (do not use) |

---

### 10. **What is the difference between EBS, Instance Store, and EFS?**

**Answer:**

| Feature     | EBS                                       | Instance Store             | EFS                         |
| ----------- | ----------------------------------------- | -------------------------- | --------------------------- |
| Persistence | Persists independently                    | Lost on stop/terminate     | Persists independently      |
| Performance | Up to 256K IOPS (io2)                     | Very high (NVMe local)     | Scalable throughput         |
| Sharing     | Single instance (except io2 multi-attach) | Single instance            | Multi-instance (NFS)        |
| Snapshots   | ✅ Yes (to S3)                            | ❌ No                      | ✅ Backup to AWS Backup     |
| Use case    | Boot volumes, databases                   | Caches, temp data, buffers | Shared config, CMS, ML data |

---

### 11. **What are EBS volume types and how do you choose?**

**Answer:**

| Type                | IOPS                 | Throughput              | Use case                         |
| ------------------- | -------------------- | ----------------------- | -------------------------------- |
| `gp3`               | 3,000 (up to 16,000) | 125 MiB/s (up to 1,000) | General purpose (default choice) |
| `io2 Block Express` | Up to 256,000        | Up to 4,000 MiB/s       | High-performance databases       |
| `st1`               | Throughput-focused   | Up to 500 MiB/s         | Big data, log processing         |
| `sc1`               | Lowest cost          | Up to 250 MiB/s         | Infrequently accessed archives   |

> **Note:** `gp2` is the previous generation. `gp3` is cheaper and gives better baseline performance — always use `gp3` for new volumes.

---

### 12. **How do you handle EC2 instance failures in production?**

**Answer:**

1. **Health checks** — ALB/NLB health checks detect unhealthy instances
2. **Auto Scaling** — Automatically replaces failed instances
3. **Multi-AZ** — Spread ASG across availability zones
4. **Status checks** — EC2 system and instance status checks trigger auto-recovery

```ts
// CDK: Auto-recovery for a critical single instance
const instance = new ec2.Instance(this, "CriticalDB", {
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.R7G,
    ec2.InstanceSize.XLARGE,
  ),
  machineImage: ec2.MachineImage.latestAmazonLinux2023(),
  vpc,
});

// CloudWatch alarm to auto-recover on system check failure
new cloudwatch.Alarm(this, "AutoRecover", {
  metric: new cloudwatch.Metric({
    namespace: "AWS/EC2",
    metricName: "StatusCheckFailed_System",
    dimensionsMap: { InstanceId: instance.instanceId },
    period: cdk.Duration.minutes(1),
  }),
  threshold: 1,
  evaluationPeriods: 2,
  alarmActions: [
    // EC2 auto-recover action
    {
      bind: () => ({
        alarmActionArn: `arn:aws:automate:${this.region}:ec2:recover`,
      }),
    } as any,
  ],
});
```

---

### 13. **What is EC2 Instance Metadata Service (IMDS)?**

**Answer:** IMDS provides instance information (instance ID, IAM role credentials, user data) accessible from within the instance at `http://169.254.169.254`.

```bash
# IMDSv2 (recommended — requires session token)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

# Get IAM role credentials
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/my-role
```

> **Important:** Always enforce **IMDSv2** (token-required) to prevent SSRF attacks from extracting credentials. IMDSv1 (no token) is a major security risk.

```ts
// Enforce IMDSv2
new ec2.Instance(this, "SecureInstance", {
  requireImdsv2: true, // Blocks IMDSv1
  // ...
});
```

---

### 14. **What is User Data and how is it used?**

**Answer:** User Data is a script that runs on first boot of an EC2 instance. Used for automated configuration.

```bash
#!/bin/bash
# User Data script
set -e

# Install dependencies
yum update -y
yum install -y docker
systemctl enable docker && systemctl start docker

# Pull and run application
docker pull my-registry/myapp:latest
docker run -d -p 80:3000 \
  --restart always \
  -e NODE_ENV=production \
  my-registry/myapp:latest

# Signal CloudFormation that setup is complete
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource ASG --region ${AWS::Region}
```

---

### 15. **What are Placement Groups?**

**Answer:**

| Type          | Behavior                                            | Use case                                    |
| ------------- | --------------------------------------------------- | ------------------------------------------- |
| **Cluster**   | All instances in same rack/AZ                       | Low-latency HPC, tightly coupled nodes      |
| **Spread**    | Each instance on distinct hardware                  | Critical instances that must be isolated    |
| **Partition** | Instances grouped into partitions on separate racks | Large distributed systems (HDFS, Cassandra) |

```ts
// Cluster placement group for HPC workload
const pg = new ec2.PlacementGroup(this, "HPC-PG", {
  strategy: ec2.PlacementGroupStrategy.CLUSTER,
});
```

---

### 16. **How do you connect to EC2 instances securely?**

**Answer:**

| Method               | Key pair needed? | Public IP needed? | Best for                                |
| -------------------- | ---------------- | ----------------- | --------------------------------------- |
| SSH with key pair    | Yes              | Yes (or VPN)      | Traditional Linux access                |
| EC2 Instance Connect | No (temp key)    | Yes               | Quick browser-based SSH                 |
| SSM Session Manager  | No               | No                | **Recommended** — no SSH, no open ports |

```bash
# ✅ Recommended: SSM Session Manager (no SSH key, no port 22)
aws ssm start-session --target i-1234567890abcdef0

# Forward a port (e.g., access RDS through bastion)
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.cluster-xxx.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

---

### 17. **What is Nitro System?**

**Answer:** Nitro is AWS's custom hardware and hypervisor platform that powers all modern EC2 instances. It offloads networking, storage, and security to dedicated hardware, giving nearly all host resources to the guest VM. Benefits:

- Better performance and security isolation
- Enables bare-metal instances
- Hardware-enforced encryption (EBS, NVMe)
- Required for features like Nitro Enclaves (secure compute)

---

### 18. **How do you right-size EC2 instances?**

**Answer:** Use AWS Compute Optimizer and CloudWatch metrics to identify over/under-provisioned instances.

```bash
# Check Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --instance-arns arn:aws:ec2:us-east-1:123456789:instance/i-1234
```

**Process:**

1. Enable Compute Optimizer (at least 14 days of CloudWatch data)
2. Review recommendations: right-size or change instance family
3. Check CPU, memory (requires CloudWatch Agent), and network utilization
4. Consider Graviton (`m7g`, `c7g`) — often 40% cheaper for same workload

---

### 19. **What is an Elastic Load Balancer and how do the types differ?**

**Answer:**

| Type              | Layer   | Protocol                    | Use case                                       |
| ----------------- | ------- | --------------------------- | ---------------------------------------------- |
| ALB (Application) | Layer 7 | HTTP/HTTPS, gRPC, WebSocket | Web apps, APIs, path-based routing             |
| NLB (Network)     | Layer 4 | TCP, UDP, TLS               | Ultra-low latency, static IPs, millions of RPS |
| GLB (Gateway)     | Layer 3 | IP                          | Third-party appliances (firewalls, IDS)        |

> **Note:** Classic Load Balancer (CLB) is **legacy**. Do not use for new deployments.

---

### 20. **How do you design a highly available EC2 architecture?**

**Answer:**

```
                    Route 53 (DNS)
                        │
                    ALB (Multi-AZ)
                   ┌────┴────┐
                AZ-a        AZ-b
              ┌──┴──┐    ┌──┴──┐
             EC2   EC2  EC2   EC2  ← Auto Scaling Group (min: 2, desired: 4)
              │     │    │     │
             EBS   EBS  EBS   EBS
                   │
               RDS Multi-AZ
              (Primary + Standby)
```

Key principles:

1. **Multi-AZ** — Spread instances across at least 2 AZs
2. **Auto Scaling** — Automatically replace failed instances
3. **Health checks** — ALB removes unhealthy targets
4. **Stateless design** — Store sessions in ElastiCache/DynamoDB, not on instance
5. **Immutable deployments** — Replace instances with new AMIs, don't patch in-place
