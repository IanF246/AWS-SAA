# 📝 Mini Exam 6: Databases & Disaster Recovery

**Covers:** Labs 6A, 6B, 6C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: availability ≠ performance ≠ DR.** Before choosing, decide which problem the scenario describes. **Availability** → Multi-AZ. **Read performance** → read replicas / caching. **Region-level recovery** → cross-Region replicas, backups, Global Database. Most wrong answers solve the wrong one of the three.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

A company runs a production RDS for MySQL database and needs it to **automatically fail over** if the primary instance or its AZ fails, without changing the application's connection string. **What should it enable?**

- **A.** A read replica in another AZ
- **B.** Multi-AZ deployment
- **C.** Automated backups with a 35-day retention
- **D.** RDS Proxy

<details>
<summary>Reveal answer</summary>

**✅ B.** Multi-AZ uses a synchronous standby with **automatic** failover behind the **same endpoint**. Read replicas need manual promotion and have a different endpoint.

🔁 **Review:** [Lab 6A, Step 5](lab-6a-rds-multi-az-read-replica.md#step-5-pull-the-plug--forced-failover-)

</details>

---

### Question 2

A reporting tool runs long analytical queries against a production RDS PostgreSQL database during business hours, slowing down the customer-facing app. **What is the BEST solution?**

- **A.** Enable Multi-AZ and point the reporting tool at the standby
- **B.** Create a read replica and point the reporting tool at the replica endpoint
- **C.** Increase the storage size
- **D.** Enable Performance Insights

<details>
<summary>Reveal answer</summary>

**✅ B.** Offload reads to a replica. The classic Multi-AZ standby **can't serve reads** (A).

🔁 **Review:** [Lab 6A, Step 3 checkpoint](lab-6a-rds-multi-az-read-replica.md#step-3-add-a-read-replica)

</details>

---

### Question 3

A global application needs a relational database with **cross-Region replication under 1 second** and the ability to **fail over to another Region in about a minute**. **Which solution fits BEST?**

- **A.** RDS MySQL with a cross-Region read replica
- **B.** Amazon Aurora Global Database
- **C.** DynamoDB global tables
- **D.** RDS Multi-AZ

<details>
<summary>Reveal answer</summary>

**✅ B.** Aurora Global Database is built for typically sub-second cross-Region replication and fast regional failover. C isn't relational. A works, but with higher lag and a slower, more manual promotion.

🔁 **Review:** [Lab 6A, Concepts](lab-6a-rds-multi-az-read-replica.md#concepts)

</details>

---

### Question 4

A DynamoDB table stores orders. A new feature needs to **look up orders by customer email**, which isn't part of the primary key. The table has existed for a year. **What should be done?**

- **A.** Scan the table with a filter expression on email
- **B.** Add a global secondary index with email as the partition key
- **C.** Add a local secondary index with email as the sort key
- **D.** Create a new table and migrate the data

<details>
<summary>Reveal answer</summary>

**✅ B.** GSIs can be added to existing tables and support a different partition key. LSIs (C) can only be created **at table creation** and keep the same partition key. A Scan (A) is costly and slow at scale.

🔁 **Review:** [Lab 6B, Steps 5–6](lab-6b-dynamodb-design.md#step-5-access-pattern-2--top-3-scores-for-meteorblasters-query-on-the-gsi)

</details>

---

### Question 5

A DynamoDB-backed application needs **microsecond** read latency for a read-heavy workload, with **minimal code changes**. **What should be added?**

- **A.** ElastiCache for Redis with custom caching logic
- **B.** DynamoDB Accelerator (DAX)
- **C.** A read replica
- **D.** DynamoDB Streams

<details>
<summary>Reveal answer</summary>

**✅ B.** DAX is a DynamoDB-compatible in-memory cache. You swap the client and keep the API. ElastiCache would work but needs custom cache logic ("minimal code changes" points to DAX).

🔁 **Review:** [Lab 6B, Exam Corner](lab-6b-dynamodb-design.md#-exam-corner)

</details>

---

### Question 6

User session data in DynamoDB must be **automatically removed 24 hours after creation**, with no extra cost or code to run deletions. **What should a solutions architect use?**

- **A.** A Lambda function on an EventBridge schedule that scans and deletes old items
- **B.** DynamoDB Time to Live (TTL) on an expiration attribute
- **C.** DynamoDB Streams
- **D.** S3 Lifecycle rules

<details>
<summary>Reveal answer</summary>

**✅ B.** TTL deletes expired items in the background at no cost.

🔁 **Review:** [Lab 6B, Step 8](lab-6b-dynamodb-design.md#step-8-ttl-and-point-in-time-recovery)

</details>

---

### Question 7

A developer accidentally runs a script that corrupts data in a DynamoDB table at 14:32. The team must restore the table to its state at **14:31**. **What makes this possible?**

- **A.** An on-demand backup taken that morning
- **B.** Point-in-time recovery (PITR), enabled beforehand
- **C.** DynamoDB global tables
- **D.** DynamoDB Streams

<details>
<summary>Reveal answer</summary>

**✅ B.** PITR restores to any second in the last 35 days, **into a new table**. Global tables (C) would replicate the corruption instantly.

🔁 **Review:** [Lab 6C, Step 5 checkpoint](lab-6c-backup-dr-strategies.md#step-5-restore)

</details>

---

### Question 8

A company must back up EBS volumes, RDS databases and DynamoDB tables across **30 accounts**, with consistent schedules, retention, and **copies to a second Region**, all managed centrally. **What is the solution with the LEAST operational overhead?**

- **A.** Lambda functions in each account calling each service's snapshot API
- **B.** AWS Backup with backup policies from AWS Organizations and cross-Region copy rules
- **C.** Data Lifecycle Manager for EBS and manual snapshots for the rest
- **D.** AWS DataSync

<details>
<summary>Reveal answer</summary>

**✅ B.** AWS Backup covers many services, tag-based selection, copy actions and org-wide policies.

🔁 **Review:** [Lab 6C, Part 2](lab-6c-backup-dr-strategies.md#part-2--centralized-cross-region-backups-with-aws-backup)

</details>

---

### Question 9

A company requires **RTO of 1 hour and RPO of 5 minutes** for a critical application, at the **lowest possible cost**. Data must be continuously replicated to the DR Region, but application servers don't need to run there until a disaster. **Which DR strategy is this?**

- **A.** Backup and restore
- **B.** Pilot light
- **C.** Warm standby
- **D.** Multi-site active/active

<details>
<summary>Reveal answer</summary>

**✅ B.** Live data replication with compute **off** until needed = **pilot light**. Backup & restore couldn't hit a 5-minute RPO. Warm standby keeps compute running, which costs more.

🔁 **Review:** [Lab 6C, Part 3](lab-6c-backup-dr-strategies.md#part-3---the-dr-strategy-game-10-xp-per-correct-answer)

</details>

---

### Question 10 *(Select TWO)*

A Lambda-based application opens a **new database connection on every invocation** to an RDS PostgreSQL instance. During traffic spikes, the database hits **"too many connections"** errors. **Which TWO actions help?**

- **A.** Put Amazon RDS Proxy in front of the database and connect through it
- **B.** Enable Multi-AZ
- **C.** Initialize the database connection outside the Lambda handler so warm invocations reuse it
- **D.** Add a read replica in another Region
- **E.** Increase the Lambda timeout

<details>
<summary>Reveal answer</summary>

**✅ A and C.** RDS Proxy pools and shares connections. Reusing connections across warm invocations cuts how many get opened. Multi-AZ (B) doesn't add connection capacity.

🔁 **Review:** [Lab 6A, Exam Corner](lab-6a-rds-multi-az-read-replica.md#-exam-corner)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | 🎉 **Domain 2 (Resilient) complete!** |
| **8–9** | ✅ Pass | +200 | Log misses, move on |
| **6–7** | 🔁 Almost | +0 | Redo 🔁 links, retake in 48 h |
| **≤ 5** | 📚 Rebuild | +0 | Re-read the Lab 6A comparison table and Lab 6C DR table |

### 🗄️ Database Picker Speed Round (+5 XP each)

| Need | Database |
|------|----------|
| Relational, managed, MySQL/PostgreSQL | RDS / Aurora |
| Key-value, single-digit ms at any scale | DynamoDB |
| In-memory cache / session store | ElastiCache (Redis OSS / Valkey / Memcached) |
| Data warehouse, analytics SQL on PBs | Redshift |
| Graph relationships (social, fraud rings) | Neptune |
| MongoDB-compatible documents | DocumentDB |
| Time-series (IoT metrics) | Timestream |
| Immutable, cryptographically verifiable ledger | QLDB (retired in 2025, but older exam questions may still name it) |
| Wide-column, Cassandra-compatible | Keyspaces |

---

**Next:** [Session 7 — Storage & DNS, Lab 7A →](../session-07-storage-dns/lab-7a-s3-storage-classes-lifecycle.md)
