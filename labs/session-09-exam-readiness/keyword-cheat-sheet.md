# 🧠 Keyword → Service Cheat Sheet

SAA questions are long, but most hinge on a **trigger phrase** that points almost directly at a service. Learn these mappings and you'll cut your reading time in half.

**How to use this page:**
1. Read it once fully.
2. Then do the **🎴 Self-Quiz** at the bottom with the right-hand columns covered.
3. Re-read it the night before the exam.

---

## 🔐 Security & Identity

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Application on EC2/Lambda needs AWS access" | **IAM role** (never access keys) | 1A |
| "Third-party / vendor access to our account" | Cross-account role + **External ID** | 1B |
| "Developers create roles without escalating privileges" | **Permissions boundary** | 1B |
| "Restrict all accounts / even root of member accounts" | **SCP** | 1C |
| "Workforce SSO across many accounts" | **IAM Identity Center** | 1B |
| "Sign-up / sign-in for mobile or web app users" | **Cognito user pools** (+ identity pools for AWS credentials) | — |
| "Rotate database credentials automatically" | **Secrets Manager** | 3C |
| "Store config values cheaply" | **SSM Parameter Store** | 3C |
| "Encryption key usage audit / control" | **KMS customer managed key** | 3A |
| "Dedicated HSM, FIPS 140-3 Level 3" | **CloudHSM** | 3A |
| "Free TLS certificates, auto-renewal" | **ACM** | 3C |
| "Protect from SQL injection / XSS / bad bots / rate limiting" | **AWS WAF** | — |
| "DDoS protection with 24/7 response team + cost protection" | **Shield Advanced** | — |
| "Detect compromised instances / unusual API calls" | **GuardDuty** | — |
| "Find PII in S3" | **Macie** | 3B |
| "Scan EC2/ECR/Lambda for vulnerabilities" | **Inspector** | — |
| "Track config changes, check compliance rules" | **AWS Config** | — |
| "Who did what, when (API audit)" | **CloudTrail** | 1C |
| "Central security findings dashboard" | **Security Hub** | — |
| "Immutable / WORM storage" | **S3 Object Lock (Compliance)** / **Backup Vault Lock** | 3B, 6C |

## 🌐 Networking

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Private subnet → internet (IPv4)" | **NAT gateway** (one per AZ) | 2A |
| "Private subnet → internet (IPv6 only, outbound)" | **Egress-only internet gateway** | 2A |
| "Private access to S3/DynamoDB, free" | **Gateway endpoint** | 2B |
| "Private access to other AWS services" | **Interface endpoint (PrivateLink)** | 2B |
| "Expose a service to other VPCs/accounts privately" | **PrivateLink endpoint service** (NLB) | 2B |
| "Block a specific IP" | **NACL deny** (or WAF) | 2B |
| "Connect 2 VPCs" | **VPC peering** | 2C |
| "Connect many VPCs + on-prem, transitive" | **Transit Gateway** | 2C |
| "Dedicated private link to on-prem, consistent performance" | **Direct Connect** | 2C |
| "Encrypted on-prem link quickly / DX backup" | **Site-to-Site VPN** | 2C |
| "Troubleshoot rejected traffic" | **VPC Flow Logs** | 2C |
| "Static IPs, global anycast, fast regional failover" | **Global Accelerator** | 7C |
| "Cache content at the edge" | **CloudFront** | 8C |
| "Restrict S3 to CloudFront only" | **Origin Access Control (OAC)** | — |
| "Hybrid DNS resolution" | **Route 53 Resolver endpoints** | 7C |

## ⚙️ Compute & Scaling

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Highly available EC2 app" | **ASG across ≥2 AZs + ALB** | 4A, 4B |
| "Path/host-based routing, HTTP" | **ALB** | 4B |
| "TCP/UDP, extreme performance, static IP" | **NLB** | 4B |
| "Third-party virtual appliances inline" | **Gateway Load Balancer** | 4B |
| "Replace instances failing app health" | **ELB health checks on ASG** | 4B |
| "Predictable daily peak" | **Scheduled / predictive scaling** | 4C |
| "Keep CPU at X%" | **Target tracking** | 4C |
| "Run code before termination" | **Lifecycle hook** | 4C |
| "Faster instance boot" | **Golden AMI / warm pool** | 4C |
| "Run containers without managing servers" | **ECS/EKS on Fargate** | — |
| "Event-driven code, short (≤ 15 min)" | **Lambda** | 3C |
| "Long-running batch jobs, managed queueing of jobs" | **AWS Batch** | — |
| "Low-latency between instances (HPC)" | **Cluster placement group** | — |
| "Spread instances across hardware to avoid correlated failure" | **Spread placement group** | — |

## 📨 Integration

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Decouple / buffer / don't lose requests" | **SQS** | 5A |
| "Strict order + exactly-once" | **SQS FIFO** | 5A |
| "Failed messages set aside" | **DLQ** | 5A |
| "One message → many subscribers" | **SNS** (fan-out to SQS) | 5B |
| "Content-based routing, SaaS events, schedules" | **EventBridge** | 5C |
| "Workflow with steps, retries, human approval" | **Step Functions** | 5C |
| "Real-time streaming, multiple consumers, replay" | **Kinesis Data Streams** | — |
| "Load streaming data into S3/Redshift/OpenSearch, no code" | **Amazon Data Firehose** | — |
| "Migrate JMS/AMQP/MQTT app" | **Amazon MQ** | — |
| "Managed Kafka" | **Amazon MSK** | — |

## 🗄️ Databases

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Automatic failover, RDS" | **Multi-AZ** | 6A |
| "Offload reads" | **Read replicas** | 6A |
| "Global relational, < 1 s replication" | **Aurora Global Database** | 6A |
| "Variable / unpredictable relational load" | **Aurora Serverless v2** | 6A |
| "Too many DB connections from Lambda" | **RDS Proxy** | 6A |
| "Key-value, single-digit ms, any scale" | **DynamoDB** | 6B |
| "Microsecond DynamoDB reads" | **DAX** | 6B |
| "Multi-Region active-active NoSQL" | **DynamoDB global tables** | 6B |
| "Cache / session store" | **ElastiCache** | — |
| "Data warehouse, analytics" | **Redshift** | — |
| "Query S3 data with SQL, serverless" | **Athena** | — |
| "Graph" / "MongoDB" / "time-series" / "Cassandra" | **Neptune / DocumentDB / Timestream / Keyspaces** | — |
| "Migrate a database with minimal downtime" | **AWS DMS** (+ **SCT** for engine changes) | — |

## 💾 Storage

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Unknown access pattern" | **S3 Intelligent-Tiering** | 7A |
| "Infrequent, re-creatable" | **S3 One Zone-IA** | 7A |
| "Archive, millisecond retrieval" | **Glacier Instant Retrieval** | 7A |
| "Cheapest archive, 12–48 h OK" | **Glacier Deep Archive** | 7A |
| "Move data between classes over time" | **Lifecycle rules** | 7A |
| "Replicate S3 to another Region" | **S3 Cross-Region Replication** (needs versioning) | — |
| "Shared Linux file system across AZs" | **EFS** | 7B |
| "Windows file shares / SMB / AD" | **FSx for Windows** | 7B |
| "HPC file system linked to S3" | **FSx for Lustre** | 7B |
| "Highest-IOPS block storage" | **EBS io2 Block Express** | 7B |
| "Temporary, fastest local disk" | **Instance store** | 7B |
| "On-prem apps using cloud storage (NFS/SMB/iSCSI/tape)" | **Storage Gateway** (File / Volume / Tape) | — |
| "Online data transfer from on-prem" | **DataSync** | — |
| "Petabytes offline / limited bandwidth" | **Snowball Edge** | — |
| "SFTP into S3" | **AWS Transfer Family** | — |

## 💰 Cost

| Trigger Phrase | Service / Feature | Lab |
|----------------|-------------------|-----|
| "Steady, flexible across EC2/Fargate/Lambda" | **Compute Savings Plan** | 8B |
| "Interruptible / fault-tolerant / batch" | **Spot** | 8B |
| "Per-socket licensing" | **Dedicated Host** | 8B |
| "Guaranteed capacity, no commitment" | **On-Demand Capacity Reservation** | 8B |
| "Alert at threshold" | **Budgets** | 8A |
| "Detect unusual spend" | **Cost Anomaly Detection** | 8A |
| "Right-sizing recommendations" | **Compute Optimizer** | 8A |
| "Cost per team/project" | **Cost allocation tags** | 8A |

---

## 🎴 Self-Quiz (+100 XP for 25/30 or better)

Cover the answers. Say the service out loud. Then reveal.

| # | Trigger | | # | Trigger |
|---|---------|-|---|---------|
| 1 | Vendor access, confused deputy | | 16 | Windows SMB shared storage |
| 2 | Restrict member-account root | | 17 | Archive, ms retrieval |
| 3 | Rotate DB password automatically | | 18 | Unknown access pattern |
| 4 | Block one IP at subnet level | | 19 | Steady RDS, cheapest |
| 5 | Free private S3 access | | 20 | Interruptible batch |
| 6 | 30 VPCs + on-prem, transitive | | 21 | Global static IPs |
| 7 | Rejected-traffic troubleshooting | | 22 | Hybrid DNS |
| 8 | Path-based routing | | 23 | Find PII in S3 |
| 9 | Static IP load balancer | | 24 | SQL injection protection |
| 10 | App-level health replacement | | 25 | WORM, nobody can delete |
| 11 | Ordered, exactly-once queue | | 26 | Migrate DB, minimal downtime |
| 12 | One event → many consumers | | 27 | Query S3 with SQL, serverless |
| 13 | Workflow + human approval | | 28 | Too many Lambda DB connections |
| 14 | Microsecond DynamoDB | | 29 | 80 TB offline transfer |
| 15 | Shared Linux files across AZs | | 30 | Detect unusual spend |

<details>
<summary>🎴 Answers</summary>

1 External ID · 2 SCP · 3 Secrets Manager · 4 NACL · 5 Gateway endpoint · 6 Transit Gateway · 7 VPC Flow Logs · 8 ALB · 9 NLB · 10 ELB health checks on the ASG · 11 SQS FIFO · 12 SNS fan-out · 13 Step Functions Standard (task token) · 14 DAX · 15 EFS · 16 FSx for Windows · 17 Glacier Instant Retrieval · 18 Intelligent-Tiering · 19 RDS Reserved Instances · 20 Spot · 21 Global Accelerator · 22 Route 53 Resolver endpoints · 23 Macie · 24 WAF · 25 Object Lock Compliance · 26 DMS · 27 Athena · 28 RDS Proxy · 29 Snowball Edge · 30 Cost Anomaly Detection

</details>

---

**Next:** [🏗️ Design Scenarios →](design-scenarios.md)
