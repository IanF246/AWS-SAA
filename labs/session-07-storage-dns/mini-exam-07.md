# 📝 Mini Exam 7: Storage & DNS Performance

**Covers:** Labs 7A, 7B, 7C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: look for the access-pattern clue.** Storage questions almost always hide one key phrase: *"accessed frequently for 30 days, then rarely,"* *"shared across AZs,"* *"Windows,"* *"milliseconds,"* *"12 hours is acceptable."* Circle it, and the answer usually follows.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

A company stores application logs in S3. Logs are queried often for **30 days**, occasionally for the next 60 days, and must be **kept for 7 years** for compliance, with retrieval within 48 hours acceptable after the first year. **What is the MOST cost-effective approach?**

- **A.** Keep everything in S3 Standard
- **B.** Lifecycle rule: Standard → Standard-IA at 30 days → Glacier Deep Archive at 365 days → expire at 7 years
- **C.** Store everything in S3 One Zone-IA
- **D.** Lifecycle rule: Standard → Glacier Deep Archive at 7 days

<details>
<summary>Reveal answer</summary>

**✅ B.** It matches each phase to the cheapest class that meets its access needs. D would break the "queried often for 30 days" requirement and make occasional access slow and expensive.

🔁 **Review:** [Lab 7A, Step 5](lab-7a-s3-storage-classes-lifecycle.md#step-5-write-a-lifecycle-policy--and-break-it-first)

</details>

---

### Question 2

A data lake has objects with **unpredictable access patterns**: some files are hot for a week, others are never read again, and nobody knows which in advance. The team wants to **optimize cost automatically without retrieval fees**. **Which storage class?**

- **A.** S3 Standard-IA
- **B.** S3 Intelligent-Tiering
- **C.** S3 Glacier Flexible Retrieval
- **D.** S3 One Zone-IA

<details>
<summary>Reveal answer</summary>

**✅ B.** "Unknown/changing patterns" + "automatic" + "no retrieval fees" = Intelligent-Tiering.

🔁 **Review:** [Lab 7A, Concepts](lab-7a-s3-storage-classes-lifecycle.md#concepts)

</details>

---

### Question 3

A hospital archives X-ray images that are **rarely accessed**, but when a doctor requests one it must be returned **within milliseconds**. **Which is the MOST cost-effective class?**

- **A.** S3 Glacier Deep Archive
- **B.** S3 Glacier Flexible Retrieval with Expedited retrieval
- **C.** S3 Glacier Instant Retrieval
- **D.** S3 Standard

<details>
<summary>Reveal answer</summary>

**✅ C.** Archive pricing with **millisecond** access. Expedited retrievals (B) take 1–5 minutes.

🔁 **Review:** [Lab 7A, Step 3](lab-7a-s3-storage-classes-lifecycle.md#step-3-try-to-read-each-one)

</details>

---

### Question 4

A content management system runs on **Linux EC2 instances across three AZs** in an ASG. All instances need **read/write access to the same set of files**. **Which storage should be used?**

- **A.** An EBS volume attached to all instances
- **B.** Amazon EFS
- **C.** Instance store volumes, synchronized with rsync
- **D.** S3 mounted with a custom script on each boot

<details>
<summary>Reveal answer</summary>

**✅ B.** EFS is a regional, shared NFS file system. EBS (A) is single-AZ and generally single-instance.

🔁 **Review:** [Lab 7B, Step 6](lab-7b-ebs-vs-efs.md#step-6-mount-on-both-nodes)

</details>

---

### Question 5

A Windows-based application on EC2 needs a **shared file system** that supports **SMB** and integrates with **Microsoft Active Directory**. **Which service?**

- **A.** Amazon EFS
- **B.** Amazon FSx for Windows File Server
- **C.** Amazon FSx for Lustre
- **D.** Amazon S3

<details>
<summary>Reveal answer</summary>

**✅ B.** EFS is NFS/Linux only. Lustre is for HPC.

🔁 **Review:** [Lab 7B, Exam Corner](lab-7b-ebs-vs-efs.md#-exam-corner)

</details>

---

### Question 6

An EBS volume attached to an instance in `us-east-1a` must be used by a new instance in `us-east-1c`. **What should be done?**

- **A.** Detach the volume and attach it to the new instance
- **B.** Create a snapshot, create a new volume from it in us-east-1c, and attach that
- **C.** Enable Multi-Attach
- **D.** Modify the volume's Availability Zone

<details>
<summary>Reveal answer</summary>

**✅ B.** EBS volumes are AZ-bound. Snapshots are regional. Multi-Attach (C) works only within one AZ, and only for io1/io2.

🔁 **Review:** [Lab 7B, Steps 8–9](lab-7b-ebs-vs-efs.md#step-8-try-to-move-the-volume-to-node-b)

</details>

---

### Question 7

A database on a **gp2** EBS volume needs more IOPS. The volume has plenty of free space. **What is the MOST cost-effective change with no downtime?**

- **A.** Double the gp2 volume size to raise its baseline IOPS
- **B.** Modify the volume to gp3 and provision the required IOPS
- **C.** Migrate to an instance-store-backed instance
- **D.** Stop the instance and change to io2

<details>
<summary>Reveal answer</summary>

**✅ B.** gp3 decouples IOPS from size and is cheaper per GB. **Elastic Volumes** change type and IOPS online. A works, but you pay for storage you don't need.

🔁 **Review:** [Lab 7B, Step 10](lab-7b-ebs-vs-efs.md#step-10-elastic-volumes--more-iops-no-downtime)

</details>

---

### Question 8

A company is releasing a new application version and wants to send **10% of users** to it while monitoring errors, then gradually increase the share. **Which Route 53 routing policy should be used?**

- **A.** Latency-based
- **B.** Weighted
- **C.** Geolocation
- **D.** Multivalue answer

<details>
<summary>Reveal answer</summary>

**✅ B.** Weighted routing is the DNS canary.

🔁 **Review:** [Lab 7C, Part 1](lab-7c-route53-routing-policies.md#part-1--weighted-routing-canary-release)

</details>

---

### Question 9

A company must ensure users in **France** see only content hosted for France, to comply with local regulations. Users elsewhere should go to a global site. **Which routing policy?**

- **A.** Latency-based routing
- **B.** Geolocation routing with a France record and a default record
- **C.** Geoproximity routing with a bias
- **D.** Weighted routing

<details>
<summary>Reveal answer</summary>

**✅ B.** Geolocation routes by **where the user is**. The **default** record catches everyone else. Latency (A) could send French users to a non-French Region if it happened to be faster.

🔁 **Review:** [Lab 7C, Step 9](lab-7c-route53-routing-policies.md#step-9--policy-picker-10-xp-each)

</details>

---

### Question 10 *(Select TWO)*

A company wants its domain apex `example.com` to point to an Application Load Balancer, and wants traffic to **automatically fail over** to a static "sorry" site in S3 if the ALB is unhealthy. **Which TWO configurations are needed?**

- **A.** A CNAME record at `example.com` pointing to the ALB
- **B.** Alias records with failover routing: primary → ALB (evaluate target health), secondary → S3 website endpoint
- **C.** A weighted routing policy with 50/50 weights
- **D.** A health check (or Evaluate Target Health) associated with the primary record
- **E.** A geolocation record for each country

<details>
<summary>Reveal answer</summary>

**✅ B and D.** An Alias works at the apex (a CNAME can't). Failover routing needs health evaluation on the primary.

🔁 **Review:** [Lab 7C, Part 2](lab-7c-route53-routing-policies.md#part-2--failover-routing-with-a-health-check)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | Session 8 |
| **8–9** | ✅ Pass | +200 | Log misses, move on |
| **6–7** | 🔁 Almost | +0 | Redo 🔁 links, retake in 48 h |
| **≤ 5** | 📚 Rebuild | +0 | Re-read the Lab 7A class table and the Lab 7B comparison table |

### 🚀 Performance Lightning Round (+5 XP each)

| Need | Answer |
|------|--------|
| Speed up global uploads to one S3 bucket | S3 Transfer Acceleration |
| Upload a 5 GB file reliably | Multipart upload (required above 5 GB, recommended above 100 MB) |
| Cache static content worldwide | CloudFront |
| 2 static anycast IPs, global failover | Global Accelerator |
| Cache DB query results in memory | ElastiCache |
| Microsecond DynamoDB reads | DAX |
| HPC scratch file system linked to S3 | FSx for Lustre |
| Move 80 TB offline to AWS | Snowball Edge |
| Ongoing online transfer from on-prem NFS to EFS/S3 | DataSync |

---

**Next:** [Session 8 — Cost Optimization, Lab 8A →](../session-08-cost-optimization/lab-8a-cost-visibility.md)
