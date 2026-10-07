# SAA Quest Glossary

A quick reference for every service and concept in the SAA Quest labs. Keep it open in another tab. The **Lab** column tells you where you used it hands-on.

---

## AWS Services

| Service | Plain English | Lab |
|---------|---------------|-----|
| **IAM** | Who can do what in your account (users, roles, policies) | 1A |
| **AWS STS** | Issues temporary credentials when you assume a role | 1A |
| **IAM Access Analyzer** | Validates policies and finds resources shared outside your account | 1C |
| **AWS Organizations** | Groups accounts; enables SCPs and consolidated billing | 1C |
| **IAM Identity Center** | Single sign-on for your workforce across many accounts | 1B (concept) |
| **Amazon VPC** | Your private network in AWS | 2A |
| **Internet Gateway** | Connects a VPC to the internet | 2A |
| **NAT Gateway** | Outbound-only internet for private subnets | 2A (concept), 8C |
| **VPC Endpoint (Gateway)** | Free private route to S3/DynamoDB | 2B |
| **VPC Endpoint (Interface / PrivateLink)** | Private IP (ENI) for reaching AWS services without internet | 2B |
| **VPC Peering** | Private one-to-one link between two VPCs | 2C |
| **VPC Flow Logs** | Metadata log of accepted/rejected network traffic | 2C |
| **Transit Gateway** | Hub that connects many VPCs and on-prem networks | 2C (Boss) |
| **AWS Systems Manager (Session Manager / Run Command / Parameter Store)** | Shell access without SSH, remote commands, config storage | 2B, 3C, 4C |
| **AWS KMS** | Creates and controls encryption keys | 3A |
| **Amazon S3** | Object storage | 1A, 3B, 7A |
| **AWS Secrets Manager** | Stores and **rotates** secrets | 3C, 6A |
| **AWS Lambda** | Runs code without servers | 3C |
| **Amazon EC2** | Virtual servers | 2B, 4A |
| **EC2 Auto Scaling** | Keeps the right number of instances running, and replaces failed ones | 4A, 4C |
| **Elastic Load Balancing (ALB / NLB / GWLB)** | Distributes traffic across targets | 4B |
| **Amazon CloudWatch** | Metrics, logs and alarms | 2C, 4C |
| **Amazon SQS** | Message queue (buffer between components) | 5A |
| **Amazon SNS** | Pub/sub notifications (one-to-many push) | 5B |
| **Amazon EventBridge** | Event bus that routes events by content; scheduler | 5C |
| **AWS Step Functions** | Visual workflows with branching, retries and waits | 5C |
| **Amazon DynamoDB** | Serverless key-value / NoSQL database | 5C, 6B, 6C |
| **Amazon RDS** | Managed relational databases (MySQL, PostgreSQL…) | 6A |
| **Amazon Aurora** | AWS's cloud-native MySQL/PostgreSQL-compatible database | 6A (concept) |
| **AWS Backup** | Central, policy-based backups across services, Regions and accounts | 6C |
| **Amazon EBS** | Block storage volumes for EC2 (single AZ) | 7B |
| **Amazon EFS** | Shared NFS file system across AZs (Linux) | 7B |
| **Amazon FSx** | Managed Windows, Lustre, ONTAP and OpenZFS file systems | 7B (concept) |
| **Amazon Route 53** | DNS, health checks, routing policies | 7C |
| **AWS Global Accelerator** | Static anycast IPs that route over the AWS backbone | 7C (Boss) |
| **Amazon CloudFront** | Content delivery network (edge caching) | 8C |
| **AWS Cost Explorer** | Analyze and forecast spending | 8A |
| **AWS Budgets** | Cost/usage alerts and automated actions | 8A |
| **Cost Anomaly Detection** | ML alerts for unusual spending | 8A |
| **AWS Compute Optimizer** | Right-sizing recommendations | 8A |
| **AWS Pricing Calculator** | Free web tool for cost estimates | 8C |

---

## Key Concepts

| Concept | Meaning | Lab |
|---------|---------|-----|
| **Implicit deny** | Everything is denied unless something allows it | 1A |
| **Explicit deny** | A `Deny` statement, which always wins | 1A |
| **Identity-based policy** | Attached to a user/role; says what it can do | 1A |
| **Resource-based policy** | Attached to a resource; has a `Principal` (bucket policy, queue policy, key policy) | 1A, 5B |
| **Trust policy** | The resource policy on a role saying who can assume it | 1B |
| **External ID** | Trust-policy condition that prevents the confused-deputy problem | 1B |
| **Permissions boundary** | Maximum permissions for a user/role (intersection with identity policy) | 1B |
| **SCP** | Organization guardrail limiting what accounts can do (never grants) | 1C |
| **CIDR** | IP range notation (`10.0.0.0/16`) | 2A |
| **Public subnet** | A subnet with a route `0.0.0.0/0 → internet gateway` | 2A |
| **Security group** | Stateful, allow-only firewall on ENIs | 2B |
| **Network ACL** | Stateless subnet firewall with allow and deny rules | 2B |
| **Ephemeral ports** | High ports (1024–65535) used for return traffic | 2B |
| **Envelope encryption** | Encrypt data with a data key, and encrypt the data key with a KMS key | 3A |
| **SSE-S3 / SSE-KMS / SSE-C** | S3 server-side encryption with S3-managed, KMS or customer-provided keys | 3A, 3B |
| **Presigned URL** | Time-limited URL granting access to one S3 object | 3B |
| **Object Lock (Governance / Compliance)** | WORM protection for S3 objects | 3B |
| **Launch template** | Versioned instance blueprint | 4A |
| **IMDSv2** | Token-based instance metadata service (blocks SSRF credential theft) | 4A |
| **Health check grace period** | Time before a new instance's health checks count | 4B |
| **Instance refresh** | Rolling replacement of ASG instances | 4B |
| **Target tracking** | Scaling policy that holds a metric at a target value | 4C |
| **Visibility timeout** | How long a received SQS message stays hidden | 5A |
| **Dead-letter queue (DLQ)** | Where messages go after too many failed attempts | 5A |
| **FIFO / message group** | Ordered, deduplicated queue; ordering per group | 5A |
| **Fan-out** | One message delivered to many subscribers | 5B |
| **Filter policy** | SNS subscription rule selecting which messages to deliver | 5B |
| **Event pattern** | EventBridge rule's JSON matcher | 5C |
| **Multi-AZ** | Synchronous standby with automatic failover (availability) | 6A |
| **Read replica** | Asynchronous readable copy (read scaling, DR) | 6A |
| **Partition key / sort key** | DynamoDB primary key parts | 6B |
| **GSI / LSI** | DynamoDB secondary indexes (global / local) | 6B |
| **TTL** | Automatic expiry of DynamoDB items | 6B |
| **PITR** | Point-in-time recovery (to any second in 35 days) | 6B, 6C |
| **RPO / RTO** | Max acceptable data loss / max acceptable downtime | 6C |
| **Backup & Restore / Pilot Light / Warm Standby / Active-Active** | The four DR strategies, cheapest to most expensive | 6C |
| **Storage class** | S3 tier with its own price, durability and retrieval profile | 7A |
| **Lifecycle rule** | Automatically transitions or expires S3 objects | 7A |
| **Elastic Volumes** | Change EBS size/type/IOPS while in use | 7B |
| **Snapshot** | Point-in-time, regional backup of an EBS volume | 7B |
| **Private hosted zone** | Route 53 zone visible only inside associated VPCs | 7C |
| **Routing policy** | Simple, weighted, failover, latency, geolocation, geoproximity, multivalue, IP-based | 7C |
| **Alias record** | Route 53 pointer to AWS resources; works at the zone apex | 7C |
| **Spot Instance** | Spare EC2 capacity, up to ~90% off, interruptible with a 2-minute warning | 8B |
| **Savings Plan** | Discount for committing to $/hr of usage for 1–3 years | 8B |
| **Reserved Instance** | Discount for committing to a specific configuration for 1–3 years | 8B |
| **Cost allocation tag** | A tag activated so it appears in cost reports | 8A |

---

## Exam Vocabulary

| Phrase | What the Exam Means |
|--------|---------------------|
| **Highly available** | Keeps working when a component or AZ fails (brief failover OK) |
| **Fault tolerant** | Keeps working *without degradation* during a failure |
| **Decoupled / loosely coupled** | Components communicate through queues/events, not direct calls |
| **Least operational overhead** | Managed or serverless; fewest things for you to run |
| **Durable** | Data won't be lost (S3: 11 nines) |
| **Elastic** | Scales out and in automatically with demand |
| **Idempotent** | Processing the same message twice has the same effect as once |
