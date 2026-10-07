# 🏗️ Design Scenarios: Five Whiteboard Challenges

Each scenario is a mini case study, like a long SAA question but **open-ended**. For each one:

1. ⏱️ **Set a 15-minute timer.**
2. ✏️ **Draw the architecture** (paper, a whiteboard or [draw.io](https://app.diagrams.net)) and list the services.
3. 📝 Answer the **follow-up questions**.
4. 📖 Open the **model answer** and score yourself with the **rubric**.

Each scenario is worth up to **250 XP**.

---

## Scenario 1: 📸 "SnapShare": Photo Sharing Startup

**The brief:** A startup lets users upload photos (up to 20 MB) from a mobile app. Each upload must generate three thumbnail sizes and run content moderation. Traffic is **unpredictable**: near zero at night, huge spikes when an influencer posts. The team is **three developers**, with no ops staff. Users worldwide must see photos quickly. Budget is tight.

**Follow-up questions:**
1. How do users upload without the photo passing through your servers?
2. What triggers thumbnail generation, and how do you avoid losing work during spikes?
3. How do you serve photos fast worldwide, while keeping the bucket private?
4. Where do photo metadata (owner, tags, likes) live?

<details>
<summary>📖 Model answer + rubric</summary>

```
Mobile app ──(Cognito auth)──▶ API Gateway ──▶ Lambda: "get presigned PUT URL"
     │
     └──── PUT photo directly ────▶ S3 (uploads/, private, SSE-S3)
                                       │ S3 Event Notification
                                       ▼
                                    SQS queue (+ DLQ) ──▶ Lambda: thumbnails ──▶ S3 (thumbs/)
                                       │                     └▶ Rekognition (moderation)
                                       ▼
                                    DynamoDB (metadata, on-demand)
Users ──▶ CloudFront (OAC) ──▶ S3 thumbs/ & photos/
```

| Element | Points |
|---------|--------|
| **Presigned URLs**: direct-to-S3 upload, no servers in the path | 40 |
| **S3 event → SQS → Lambda** (the queue absorbs spikes; DLQ for failures) | 50 |
| **Serverless everywhere** (Lambda, API Gateway, DynamoDB on-demand), fitting "no ops staff" + unpredictable traffic | 40 |
| **CloudFront + OAC**: global speed with a private bucket | 40 |
| **DynamoDB** for metadata | 30 |
| **Cognito** for user auth | 25 |
| Lifecycle: originals → Intelligent-Tiering / IA | 25 |

</details>

---

## Scenario 2: 🏦 "LedgerCo": Regulated Financial Platform

**The brief:** A fintech runs a transaction-processing app on EC2 with a PostgreSQL database. Regulators require: all data **encrypted with keys the company controls**, every key use **audited**, transaction records **immutable for 7 years**, **no internet access** from application servers, and the ability to recover from a **Region outage within 1 hour, losing at most 5 minutes of data**.

**Follow-up questions:**
1. Which database service and DR setup meet RTO 1 h / RPO 5 min?
2. How do app servers reach S3, KMS and Secrets Manager with no internet?
3. How do you make the 7-year records immutable?
4. How do you stop anyone from disabling CloudTrail?

<details>
<summary>📖 Model answer + rubric</summary>

- **Aurora PostgreSQL** Multi-AZ, plus **Aurora Global Database** (or a cross-Region read replica) to the DR Region: **pilot light**, with compute pre-defined in IaC (AMIs copied, ASG desired = 0). **Route 53 failover** routing. *(60)*
- **Customer managed KMS keys** (multi-Region keys for DR) for Aurora, EBS, S3 and Secrets Manager. CloudTrail logs every key use. *(40)*
- **Private subnets only**, **no NAT**. **Gateway endpoint** for S3, and **interface endpoints** for KMS, Secrets Manager, SSM and CloudWatch Logs. Session Manager for admin access. *(50)*
- Transaction exports in S3 with **Object Lock, Compliance mode, 7 years**, replicated cross-Region. **AWS Backup Vault Lock** for backups. *(40)*
- **Organization CloudTrail** to a locked log-archive account. **SCP** denying `cloudtrail:StopLogging/DeleteTrail`. GuardDuty + Security Hub. *(40)*
- DB credentials in **Secrets Manager** with automatic rotation. *(20)*

</details>

---

## Scenario 3: 🛒 "MegaMart": Lift-and-Shift Then Modernize

**The brief:** A retailer runs a monolithic Java app on 6 on-prem servers with a MySQL database (2 TB), and a Windows file share for product images (5 TB). They want to move to AWS **within 3 months** with **minimal code changes**, then modernize. Their data-center link is 500 Mbps and already busy. During Black Friday, traffic is 10× normal.

**Follow-up questions:**
1. How do you migrate the database with minimal downtime?
2. How do you move 5 TB of images given the busy network link?
3. What's the target compute architecture for phase 1?
4. Where does the Windows file share go?

<details>
<summary>📖 Model answer + rubric</summary>

- **AWS Application Migration Service (MGN)** to rehost the servers onto EC2 (lift-and-shift). *(40)*
- **AWS DMS** with full load + **CDC** (ongoing replication) into **RDS for MySQL / Aurora MySQL** (Multi-AZ). Cut over with minutes of downtime. *(50)*
- 5 TB on a busy 500 Mbps link would take ~1 day at 100% utilization, unrealistic during business hours. Use a **Snowball Edge** for the bulk copy, then **DataSync** for the delta. *(Accepting "DataSync with bandwidth throttling over a few weeks" earns partial credit.)* *(40)*
- Phase 1 compute: **ALB + ASG across 2 AZs**, golden AMI, **scheduled scaling** for Black Friday. *(40)*
- Images: **FSx for Windows File Server** (Multi-AZ) for the app as-is. Phase 2: move images to **S3 + CloudFront**. *(40)*
- Phase 2 modernization: break out services with **SQS / EventBridge**, sessions to **ElastiCache**, containers on **ECS Fargate**. *(40)*

</details>

---

## Scenario 4: 📡 "SensorStream": IoT Telemetry

**The brief:** 50,000 devices each send a small reading **every second**. The company needs (a) a **real-time dashboard** with < 5 s delay, (b) **anomaly alerts** within seconds, (c) all raw data kept **cheaply for 5 years** for occasional analysis with SQL, and (d) the ability to **replay the last 24 hours** of data if a processing bug is found.

**Follow-up questions:**
1. What ingests 50,000 records/second with replay?
2. How do you store 5 years cheaply and query with SQL?
3. Where does the real-time dashboard data live?

<details>
<summary>📖 Model answer + rubric</summary>

- Devices → **AWS IoT Core** → **Kinesis Data Streams** (provisioned or on-demand mode, ≥ 24 h retention for **replay**). *(60)*
- **Lambda consumer** (or Managed Service for Apache Flink) for anomaly detection → **SNS / EventBridge** alerts. *(50)*
- **Amazon Data Firehose** → **S3** (Parquet, partitioned by date) → lifecycle to **Glacier Instant Retrieval / Deep Archive**. Query with **Athena** (Glue Data Catalog). *(60)*
- Dashboard data in **Timestream** (time-series) or DynamoDB, visualized with **Managed Grafana** / QuickSight. *(40)*
- Explaining **why not SQS**: SQS doesn't support multiple independent consumers replaying the same data. *(40)*

</details>

---

## Scenario 5: 🌍 "GlobalLearn": Multi-Region E-Learning

**The brief:** An e-learning platform serves students in North America, Europe and Asia. Requirements: **lowest latency** for each student; video lessons streamed **globally**; user progress data writable **in every Region** (students travel); **no single Region failure** should take the platform down; EU student data must be **served from the EU only** for compliance.

**Follow-up questions:**
1. Which Route 53 routing policy (or policies)?
2. Which database supports multi-Region writes?
3. How do you satisfy the EU-only requirement *and* latency for everyone else?

<details>
<summary>📖 Model answer + rubric</summary>

- **Active/active multi-Region**: ALB + ASG/ECS in 3 Regions. *(40)*
- **Route 53 geolocation** for Europe → EU Region (compliance), with **latency-based routing** for everyone else, plus health checks. Since geolocation and latency can't be mixed in one record set, use **nested records** (or Traffic Flow policies). *(60)*
- **DynamoDB global tables** for progress (multi-Region active-active writes). EU student PII in a **separate EU-only table** to respect data residency. *(60)*
- **CloudFront** for video (plus S3 origins, signed URLs/cookies for paid content). *(40)*
- **Global Accelerator** as an alternative front door for the APIs (static IPs, fast failover). *(25)*
- Identity: **Cognito** with tokens valid across Regions. *(25)*

</details>

---

## 🧮 Scenario Score Card

| Scenario | My Score / 250 | Biggest Miss |
|----------|----------------|--------------|
| 1 SnapShare | | |
| 2 LedgerCo | | |
| 3 MegaMart | | |
| 4 SensorStream | | |
| 5 GlobalLearn | | |
| **Total** | **/ 1250** | |

> Add each "biggest miss" to your weak-spot log in the [README](../../README.md#-progress-tracker).

---

**Next:** [📝 30-Question Mock Exam →](mock-exam.md)
