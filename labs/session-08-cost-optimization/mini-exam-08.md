# 📝 Mini Exam 8: Cost Optimization

**Covers:** Labs 8A, 8B, 8C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: requirements first, then price.** A "MOST cost-effective" question still has requirements (availability, no interruption, latency). Eliminate every option that breaks a requirement, and only *then* pick the cheapest of what's left. The cheapest option overall is often a trap.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

A company runs a fleet of EC2 instances **24/7 for the foreseeable future**. It plans to migrate parts of the workload to **AWS Fargate and Lambda** over the next year and may change instance families. **Which purchasing option gives the biggest discount while keeping that flexibility?**

- **A.** Standard Reserved Instances, 3-year
- **B.** Compute Savings Plans
- **C.** EC2 Instance Savings Plans
- **D.** Spot Instances

<details>
<summary>Reveal answer</summary>

**✅ B.** Compute Savings Plans apply across **EC2 (any family/Region), Fargate and Lambda**. EC2 Instance SPs and Standard RIs lock you to a family/configuration.

🔁 **Review:** [Lab 8B, Concepts](lab-8b-purchasing-options-spot.md#concepts)

</details>

---

### Question 2

A genomics company runs **batch jobs** that can be **stopped and restarted** at any point and have flexible completion times. **What is the MOST cost-effective compute option?**

- **A.** On-Demand Instances
- **B.** Reserved Instances
- **C.** Spot Instances
- **D.** Dedicated Hosts

<details>
<summary>Reveal answer</summary>

**✅ C.** Interruptible + flexible = Spot (up to ~90% off).

🔁 **Review:** [Lab 8B, Step 3](lab-8b-purchasing-options-spot.md#step-3-launch-a-single-spot-instance)

</details>

---

### Question 3

A web application must **always** have at least 4 instances that are never interrupted, and can use extra capacity during peaks at the **lowest cost**. **Which configuration is BEST?**

- **A.** An ASG with all Spot Instances
- **B.** An ASG mixed-instances policy with an On-Demand base capacity of 4 and Spot above the base, diversified across instance types
- **C.** An ASG with all On-Demand Instances
- **D.** Dedicated Instances

<details>
<summary>Reveal answer</summary>

**✅ B.** The On-Demand base guarantees the floor, and diversified Spot keeps peak capacity cheap.

🔁 **Review:** [Lab 8B, Step 4](lab-8b-purchasing-options-spot.md#step-4-production-pattern--mixed-instances-auto-scaling-group)

</details>

---

### Question 4

A company must run software licensed **per physical CPU core** on EC2 and needs visibility into the underlying sockets and cores. **Which option?**

- **A.** Dedicated Instances
- **B.** Dedicated Hosts
- **C.** Spot Instances
- **D.** Compute Savings Plans

<details>
<summary>Reveal answer</summary>

**✅ B.** Only Dedicated **Hosts** expose socket/core details for BYOL. Dedicated **Instances** give hardware isolation but not that visibility.

🔁 **Review:** [Lab 8B, Step 6](lab-8b-purchasing-options-spot.md#step-6--purchase-option-picker-10-xp-each)

</details>

---

### Question 5

Private EC2 instances transfer **several TB per month** to and from **Amazon S3 and DynamoDB** through a NAT gateway. The NAT gateway is the largest item on the bill. **What reduces cost the MOST?**

- **A.** Replace the NAT gateway with a NAT instance on a larger instance type
- **B.** Create gateway VPC endpoints for S3 and DynamoDB and add them to the private route tables
- **C.** Create interface endpoints for S3 and DynamoDB
- **D.** Move the instances to public subnets

<details>
<summary>Reveal answer</summary>

**✅ B.** Gateway endpoints are free and remove NAT processing charges for that traffic.

🔁 **Review:** [Lab 8C, Round 1](lab-8c-architecture-cost-showdown.md#round-1---nat-gateway-vs-gateway-endpoint)

</details>

---

### Question 6

An API receives **a few thousand requests per day**, with occasional unpredictable bursts. Today it runs on two always-on EC2 instances behind an ALB. **Which change is MOST cost-effective?**

- **A.** Buy a Savings Plan for the instances
- **B.** Re-architect to API Gateway + Lambda
- **C.** Use larger instances to handle bursts
- **D.** Add a third instance

<details>
<summary>Reveal answer</summary>

**✅ B.** Low, spiky traffic means paying only per request. Always-on servers sit idle most of the time.

🔁 **Review:** [Lab 8C, Round 3](lab-8c-architecture-cost-showdown.md#round-3---serverless-vs-servers-two-traffic-levels)

</details>

---

### Question 7

A finance team wants to see **costs per project** across a multi-account organization. Resources are already tagged with a `Project` key. Cost Explorer doesn't show the tag. **What must be done?**

- **A.** Enable AWS Config
- **B.** Activate `Project` as a cost allocation tag in the Billing console of the management account
- **C.** Enable detailed monitoring
- **D.** Create a new Cost Explorer report

<details>
<summary>Reveal answer</summary>

**✅ B.** Tags must be **activated** for cost allocation, and they apply going forward, not retroactively.

🔁 **Review:** [Lab 8A, Step 4](lab-8a-cost-visibility.md#step-4-who-spent-it-tag-resources)

</details>

---

### Question 8

A company wants to be **automatically notified of unusual spending**, such as a sudden spike in one service, **without defining fixed thresholds** for every service. **Which tool?**

- **A.** AWS Budgets
- **B.** AWS Cost Anomaly Detection
- **C.** AWS Trusted Advisor
- **D.** Cost and Usage Report

<details>
<summary>Reveal answer</summary>

**✅ B.** ML-based detection of deviations from your normal pattern. Budgets need thresholds you set yourself.

🔁 **Review:** [Lab 8A, Step 5](lab-8a-cost-visibility.md#step-5-will-i-find-out-if-it-spikes-cost-anomaly-detection)

</details>

---

### Question 9

A software company distributes large downloads to customers across North America directly from an S3 bucket, and data transfer is its largest cost. **What reduces cost AND improves download speed?**

- **A.** Enable S3 Transfer Acceleration
- **B.** Serve downloads through Amazon CloudFront with the bucket as the origin
- **C.** Move the files to S3 One Zone-IA
- **D.** Use S3 Cross-Region Replication to a second bucket

<details>
<summary>Reveal answer</summary>

**✅ B.** CloudFront's per-GB price is lower than S3 direct-to-internet, origin fetches from S3 are free, and edge caching speeds up delivery. Transfer Acceleration (A) **adds** cost and speeds up *uploads* to S3.

🔁 **Review:** [Lab 8C, Round 4](lab-8c-architecture-cost-showdown.md#round-4---serving-50-tb-of-downloads)

</details>

---

### Question 10 *(Select TWO)*

A development RDS database (Multi-AZ, 24/7) and a set of development EC2 instances are only used **on weekdays during office hours**. **Which TWO actions reduce cost the MOST without affecting production?**

- **A.** Convert the development database to Single-AZ
- **B.** Purchase 3-year Reserved Instances for the development database
- **C.** Stop the development EC2 and RDS instances outside office hours using a scheduler
- **D.** Enable Multi-AZ on the development EC2 instances
- **E.** Move development to a larger instance class

<details>
<summary>Reveal answer</summary>

**✅ A and C.** Dev doesn't need Multi-AZ, and scheduling avoids paying for idle nights and weekends (~65–75% fewer hours). B commits you to paying 24/7 for something used ~35% of the time.

🔁 **Review:** [Lab 8C, Round 5](lab-8c-architecture-cost-showdown.md#round-5---the-dev-database)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | 🎉 **All four domains covered!** On to the capstone |
| **8–9** | ✅ Pass | +200 | Log misses, then Session 9 |
| **6–7** | 🔁 Almost | +0 | Redo 🔁 links, retake in 48 h |
| **≤ 5** | 📚 Rebuild | +0 | Replay all six rounds of Lab 8C |

### 💰 Cost Reflex Drill (+5 XP each)

| Clue | Reflex |
|------|--------|
| Steady 24/7, flexible future | Compute Savings Plan |
| Steady RDS | RDS Reserved Instances |
| Interruptible | Spot |
| Idle nights/weekends | Scheduler |
| Old logs | Lifecycle → Glacier |
| Private → S3 | Gateway endpoint |
| gp2 | gp3 |
| Global downloads | CloudFront |
| "Oversized?" | Compute Optimizer |

---

**Next:** [🏆 Session 9 — Exam Readiness Capstone →](../session-09-exam-readiness/README.md)
