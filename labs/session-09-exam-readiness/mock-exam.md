# 📝 SAA Quest Mock Exam (30 Questions)

**Covers:** Everything · **Time limit:** ⏱️ **60 minutes** (the real exam's pace: 2 min/question)  
**Pass mark:** 23/30 (≈ 75%) · **Reward:** +500 XP for passing, +250 bonus for ≥ 27/30

All questions are **new**: none are repeated from the mini exams. The mix follows the real exam's weighting:

| Domain | Questions |
|--------|-----------|
| D1 Secure (30%) | Q1–Q9 |
| D2 Resilient (26%) | Q10–Q17 |
| D3 High-Performing (24%) | Q18–Q24 |
| D4 Cost-Optimized (20%) | Q25–Q30 |

### Rules
1. ⏱️ Timer on. No notes, no console, no searching.
2. Use the **3-pass strategy** from the [Capstone Guide](README.md#-exam-day-game-plan): sweep, then flags, then sanity check.
3. Fill in the answer sheet. Open **no** reveals until time is up.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
|---|---|---|---|---|---|---|---|---|---|----|----|----|----|----|----|
| Answer | | | | | | | | | | | | | | | |
| Flag 🚩 | | | | | | | | | | | | | | | |

| Q | 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 | 25 | 26 | 27 | 28 | 29 | 30 |
|---|----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
| Answer | | | | | | | | | | | | | | | |
| Flag 🚩 | | | | | | | | | | | | | | | |

---

## Domain 1: Design Secure Architectures

**Q1.** A company's S3 bucket must only be accessible from instances in a specific VPC. Requests from anywhere else, including the console, must be denied. **What should be used?**
- A. A bucket policy that denies requests unless `aws:SourceVpce` matches the VPC's S3 gateway endpoint
- B. A security group on the bucket
- C. A NACL on the bucket
- D. S3 Block Public Access

**Q2.** A web application behind an ALB is receiving a **SQL injection** attack. **What provides protection with the LEAST operational overhead?**
- A. Install a host-based firewall on each instance
- B. Associate an AWS WAF web ACL with AWS managed rules for SQL injection with the ALB
- C. Enable GuardDuty
- D. Use Shield Standard

**Q3.** An organization needs to **centrally view security findings** from GuardDuty, Inspector and Macie across 20 accounts and check them against CIS benchmarks. **Which service?**
- A. AWS Config
- B. AWS Security Hub
- C. Amazon Detective
- D. CloudTrail Lake

**Q4.** A mobile application needs to let users sign in with **Google or Apple accounts** and then upload files directly to their own prefix in S3. **What should be used?**
- A. IAM users for each customer
- B. Amazon Cognito user pools with identity pools that issue temporary credentials scoped to the user's prefix
- C. IAM Identity Center
- D. A shared access key embedded in the app

**Q5.** A company must ensure that **any new S3 bucket in any account** automatically has default encryption and **any non-compliant bucket is flagged and remediated** automatically. **What is the BEST approach?**
- A. A weekly script that checks buckets
- B. AWS Config rules with automatic remediation via SSM Automation, deployed org-wide with conformance packs
- C. Amazon Macie
- D. CloudTrail alerts

**Q6.** EC2 instances in a private subnet must download patches from the internet, but **must not be reachable from the internet**. **What should be configured?**
- A. An internet gateway route in the private subnet
- B. A NAT gateway in a public subnet and a default route from the private subnet to it
- C. Elastic IPs on the instances
- D. A gateway VPC endpoint

**Q7.** An application uses an **ALB with HTTPS**. The company wants to **offload TLS** and use a certificate that **renews automatically**. **What should be done?**
- A. Install certificates on each EC2 instance
- B. Request a public certificate in ACM and attach it to the ALB's HTTPS listener
- C. Use a self-signed certificate in CloudHSM
- D. Use an NLB with TCP passthrough

**Q8.** A security audit finds that an EC2 instance's IAM role has `AdministratorAccess`, though the app only reads from one DynamoDB table. **Which action follows best practice?**
- A. Leave it, since the instance is in a private subnet
- B. Replace the policy with one allowing only `dynamodb:GetItem`/`Query` on that table's ARN
- C. Add an MFA requirement to the role
- D. Rotate the role's access keys

**Q9. (Select TWO)** A company wants to protect a public website on CloudFront against **large DDoS attacks** and wants **financial protection against scaling charges** caused by an attack. **Which TWO?**
- A. AWS Shield Advanced
- B. AWS WAF rate-based rules
- C. Amazon Inspector
- D. VPC Flow Logs
- E. AWS Config

---

## Domain 2: Design Resilient Architectures

**Q10.** An application stores **user session state in memory** on EC2 instances behind an ALB. Users are logged out whenever Auto Scaling terminates an instance. **What is the MOST resilient fix?**
- A. Enable sticky sessions
- B. Store session state in ElastiCache or DynamoDB
- C. Disable scale-in
- D. Use larger instances

**Q11.** A company's critical application uses a **single NAT gateway**. **What should be done to make outbound connectivity highly available?**
- A. Add a second NAT gateway in the same subnet
- B. Deploy a NAT gateway in each AZ and configure each private subnet's route table to use the NAT gateway in its own AZ
- C. Replace the NAT gateway with an internet gateway
- D. Use a NAT instance with auto recovery

**Q12.** Orders are sent to an SQS queue and processed by EC2 workers. The team wants the worker fleet to **scale based on the backlog**. **Which metric approach is recommended?**
- A. Scale on worker CPU utilization
- B. Target tracking on a custom metric: `ApproximateNumberOfMessagesVisible` ÷ number of running instances
- C. Scale on the ALB request count
- D. Scheduled scaling every hour

**Q13.** An S3 bucket in us-east-1 holds critical data. The company needs a copy in **eu-west-1** within minutes of every write, for DR. **What should be configured?**
- A. S3 Lifecycle rules
- B. S3 Cross-Region Replication (with versioning enabled on both buckets)
- C. S3 Transfer Acceleration
- D. A daily AWS Backup job

**Q14.** A stateless web application must keep running **if an entire AWS Region fails**, with **near-zero RTO**. **Which design?**
- A. Multi-AZ ASG in one Region
- B. Active/active deployments in two Regions with Route 53 latency or failover routing and health checks, and a multi-Region data layer
- C. Daily AMI copies to another Region
- D. Larger instances in one AZ

**Q15.** Lambda functions process images uploaded to S3. Occasionally, a malformed image makes the function fail repeatedly. **How should the failures be captured for later analysis without blocking other events?**
- A. Increase the Lambda timeout
- B. Configure an on-failure destination or a dead-letter queue (SQS/SNS) for the asynchronous invocation
- C. Enable S3 versioning
- D. Use provisioned concurrency

**Q16.** A database team needs an **RPO of seconds** for an Aurora MySQL cluster, against **accidental data deletion by a user**. **What provides this?**
- A. Aurora Replicas
- B. Aurora Backtrack, or point-in-time restore from continuous backups
- C. Multi-AZ
- D. Aurora Global Database

**Q17. (Select TWO)** An application on an ASG behind an ALB fails health checks for 5 minutes after every scale-out because initialization takes 4 minutes. Instances are terminated before they finish starting. **Which TWO actions fix this?**
- A. Increase the ASG health check grace period
- B. Pre-bake dependencies into a custom AMI
- C. Change the health check type to EC2 only, permanently
- D. Reduce the ASG maximum capacity
- E. Remove the ALB

---

## Domain 3: Design High-Performing Architectures

**Q18.** A news website has **read-heavy** traffic on an RDS MySQL database, and the **same articles are requested repeatedly**. **What reduces database load the MOST?**
- A. Multi-AZ
- B. An ElastiCache cache in front of the database (cache-aside)
- C. Larger storage
- D. Provisioned IOPS

**Q19.** Users around the world upload large video files (several GB) to a **single S3 bucket in us-east-1**, and uploads from Asia are slow. **What improves upload performance?**
- A. CloudFront signed URLs
- B. S3 Transfer Acceleration with multipart uploads
- C. S3 Cross-Region Replication
- D. A larger S3 storage class

**Q20.** An HPC application needs the **lowest possible network latency and highest throughput between instances**. **What should be used?**
- A. A spread placement group
- B. A cluster placement group in a single AZ, with enhanced networking/EFA
- C. Instances in multiple Regions
- D. A partition placement group across AZs

**Q21.** A company needs to run SQL queries **occasionally** on **CSV and Parquet files in S3** without loading them into a database or managing servers. **Which service?**
- A. Amazon Redshift provisioned cluster
- B. Amazon Athena
- C. Amazon RDS
- D. Amazon EMR on EC2

**Q22.** A gaming company needs a leaderboard with **sub-millisecond reads** and **sorted rankings** that update in real time. **Which is MOST suitable?**
- A. RDS with an index on score
- B. ElastiCache for Redis OSS / Valkey, using sorted sets
- C. S3 Select
- D. Athena

**Q23.** A Lambda function behind API Gateway has **high latency on the first requests after idle periods**, which is unacceptable for a latency-sensitive API. **What fixes this?**
- A. Increase the function's timeout
- B. Configure provisioned concurrency (or SnapStart, where supported)
- C. Move the function into a VPC
- D. Set reserved concurrency to 0

**Q24. (Select TWO)** A global application needs to improve performance for **dynamic API calls (TCP and UDP)** from users worldwide to an NLB in us-east-1, with **fast failover** to us-west-2. **Which TWO?**
- A. AWS Global Accelerator with endpoint groups in both Regions
- B. CloudFront with caching for API responses
- C. Health checks on the endpoint groups to fail over automatically
- D. S3 Transfer Acceleration
- E. A larger NLB

---

## Domain 4: Design Cost-Optimized Architectures

**Q25.** A data pipeline writes **temporary intermediate files** to S3. Each file is read heavily for **2 days**, rarely afterward, and **must be deleted after 14 days**. **What is the MOST cost-effective lifecycle configuration?**
- A. Transition to S3 Standard-IA after 2 days, expire after 14 days
- B. Transition to S3 Glacier Flexible Retrieval after 2 days, expire after 14 days
- C. Keep in S3 Standard and expire after 14 days
- D. Transition to S3 One Zone-IA after 3 days, expire after 14 days

**Q26.** A company's EC2 fleet has steady utilization of **15% CPU** on m5.2xlarge instances 24/7. **What should be done FIRST to reduce cost?**
- A. Purchase 3-year RIs for m5.2xlarge
- B. Right-size the instances using Compute Optimizer recommendations, then commit with Savings Plans
- C. Switch to Spot
- D. Add more instances

**Q27.** An application runs **container workloads** for 2 hours a day with unpredictable start times. The team doesn't want to manage servers. **What is MOST cost-effective?**
- A. An EKS cluster on always-on EC2 nodes
- B. Amazon ECS on AWS Fargate (consider Fargate Spot if interruptions are OK)
- C. Dedicated Hosts
- D. EC2 Reserved Instances

**Q28.** A company transfers **large amounts of data between EC2 instances in different AZs** within the same VPC (a chatty microservice). The bill shows high **inter-AZ data transfer**. The workload is **not critical** and can tolerate an AZ outage. **What reduces cost?**
- A. Use a VPC endpoint
- B. Place the communicating instances in the same AZ
- C. Use a NAT gateway
- D. Enable enhanced networking

**Q29.** A DynamoDB table has **steady, predictable traffic** around the clock and currently uses **on-demand** capacity. **What lowers cost?**
- A. Switch to provisioned capacity with auto scaling (and consider reserved capacity)
- B. Enable DAX
- C. Enable global tables
- D. Enable PITR

**Q30. (Select TWO)** A startup wants to be alerted when monthly spend is **forecast to exceed $500**, and **automatically apply a restrictive IAM policy** to developer roles if actual spend exceeds $700. **Which TWO?**
- A. AWS Budgets with a forecasted-cost alert at $500
- B. A Budgets action that applies an IAM policy (or SCP) when actual cost exceeds $700
- C. Cost Explorer reports
- D. Trusted Advisor
- E. CloudWatch billing alarm on EstimatedCharges with SNS email only

---

## 📖 Answer Key

<details>
<summary>Open only after the timer ends</summary>

| Q | Ans | Why | Lab |
|---|-----|-----|-----|
| 1 | **A** | `aws:SourceVpce`/`aws:SourceVpc` conditions restrict a bucket to endpoint traffic. Buckets don't have SGs/NACLs. | 2B, 3B |
| 2 | **B** | WAF managed rule groups block SQLi at the ALB. Shield is DDoS; GuardDuty detects threats but doesn't block them. | Mini Exam 3 |
| 3 | **B** | Security Hub aggregates findings and runs standards checks (CIS, AWS FSBP). | Mini Exam 3 |
| 4 | **B** | Cognito user pools (social sign-in) + identity pools → temporary, scoped credentials. | Cheat sheet |
| 5 | **B** | AWS Config rules + auto-remediation, deployed org-wide. | Mini Exam 3 |
| 6 | **B** | NAT gateway = outbound-only internet for private subnets. | 2A |
| 7 | **B** | ACM public certs are free and auto-renew on ALB/CloudFront. | 3C |
| 8 | **B** | Least privilege: scope actions **and** resource ARNs. (Roles have no access keys to rotate.) | 1A, 3C |
| 9 | **A, B** | Shield Advanced = large-DDoS protection + **cost protection**. WAF rate-based rules handle L7 floods. | Cheat sheet |
| 10 | **B** | Externalize state. Sticky sessions still lose state when the instance dies. | 4B |
| 11 | **B** | One NAT gateway per AZ, with AZ-local routing. | 2A |
| 12 | **B** | "Backlog per instance" is AWS's recommended SQS scaling metric. | 4C |
| 13 | **B** | CRR replicates new objects asynchronously (minutes; RTC for a 15-min SLA). Requires versioning. | 3B, 7A |
| 14 | **B** | A Region-level failure with near-zero RTO needs active/active multi-Region. | 6C, 7C |
| 15 | **B** | Async Lambda: on-failure destination / DLQ captures failed events. | 5A |
| 16 | **B** | Backtrack/PITR recover from logical errors. Replicas and Global DB replicate the deletion too. | 6C |
| 17 | **A, B** | Grace period gives boot time, and a golden AMI shortens boot. | 4B, 4C |
| 18 | **B** | Repeated identical reads → cache. | 6A |
| 19 | **B** | Transfer Acceleration uses edge locations for uploads, and multipart parallelizes them. | Mini Exam 7 |
| 20 | **B** | Cluster placement group = low latency, high throughput within one AZ. | Cheat sheet |
| 21 | **B** | Athena = serverless SQL on S3, pay per query. | Cheat sheet |
| 22 | **B** | Redis/Valkey sorted sets are made for leaderboards. | Mini Exam 6 |
| 23 | **B** | Provisioned concurrency keeps execution environments warm, removing cold starts. | — |
| 24 | **A, C** | Global Accelerator handles TCP/UDP over the AWS backbone with health-based regional failover. CloudFront doesn't support UDP. | 7C |
| 25 | **C** | Trap! A and D are **invalid** (IA transitions need 30+ days). B is allowed, but Glacier Flexible bills a **90-day minimum** plus transition fees, which costs more than 12 more days in Standard. Short-lived data → Standard + expiration. | 7A |
| 26 | **B** | Right-size **before** committing, or you lock in waste. | 8A, 8B |
| 27 | **B** | Fargate = pay per task-second, no idle nodes. | 8C |
| 28 | **B** | Inter-AZ transfer costs ~$0.01/GB each way. Co-locating is acceptable *because* the workload isn't critical. | 8C |
| 29 | **A** | Steady traffic → provisioned (+ reserved) is cheaper than on-demand. | 6B |
| 30 | **A, B** | Budgets support forecast alerts **and** actions (IAM policy / SCP / stop instances). | 8A |

</details>

---

## 🧮 Score by Domain

| Domain | Questions | My Score | % | Status |
|--------|-----------|----------|---|--------|
| D1 Secure | Q1–9 | / 9 | | |
| D2 Resilient | Q10–17 | / 8 | | |
| D3 High-Performing | Q18–24 | / 7 | | |
| D4 Cost | Q25–30 | / 6 | | |
| **Total** | | **/ 30** | | |

**Any domain below 70%?** Go back through that domain's mini exams and Exam Corners before booking.

| Total | Verdict |
|-------|---------|
| **27–30** | 🏆 Book the exam. +750 XP |
| **23–26** | ✅ Ready. Take one official practice exam, then book. +500 XP |
| **18–22** | 🔁 Close. Review your weakest domain for a week, then retake |
| **≤ 17** | 📚 Revisit Sessions 1–8 mini exams first |

---

**Back to:** [🏆 Capstone Guide](README.md) · [Main README](../../README.md)
