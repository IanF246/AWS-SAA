# Lab 8C: Architecture Cost Showdown — Napkin Math Like a Solutions Architect

**Session:** 8 — Cost Optimization  
**Exam Domain:** Domain 4 — Design Cost-Optimized Architectures (20%)  
**Difficulty:** Advanced  
**Estimated Time:** 45–60 minutes  
**AWS resources created:** None. This lab is all reasoning, math and the AWS Pricing Calculator.

---

## Overview

The exam never asks you to compute a bill to the cent, but it constantly asks *"which option is MOST cost-effective?"* Architects who can do **napkin math** answer those in seconds, because they know which line items dominate.

This lab is a **game in six rounds**. In each round, two architectures face off. You:
1. 🔮 **Predict** the winner on instinct (+10 XP if right)
2. 🧮 **Calculate** both with the unit prices given (+20 XP if within 15% of the answer key)
3. 📖 **Reveal** the answer key and the exam lesson

Then you'll rebuild one round in the official **AWS Pricing Calculator**, and finish with a **Boss Challenge**: cut a real-looking $13,560/month bill.

---

## Prerequisites

- ✅ Labs 8A and 8B complete
- ✅ A calculator (or a spreadsheet)
- ✅ Browser access to [calculator.aws](https://calculator.aws)

---

## Cost Notice

**$0.00.** No AWS resources are created.

---

## 📋 Unit Price Sheet (us-east-1, approximate)

> ⚠️ Prices change. These are rounded, illustrative figures for practicing the math. **Always verify current prices** on the AWS pricing pages or [calculator.aws](https://calculator.aws). For this lab, use these numbers so your answers match the key.

| Item | Price |
|------|-------|
| Hours in a month | **730** |
| NAT gateway | $0.045/hr + **$0.045/GB processed** |
| S3 gateway endpoint | **$0** |
| Interface VPC endpoint | $0.01/hr per AZ + $0.01/GB |
| EC2 m5.large On-Demand | $0.096/hr |
| EC2 m5.large, 1-yr Compute Savings Plan (no upfront) | ~$0.067/hr |
| EC2 m5.large Spot (typical) | ~$0.035/hr |
| Lambda | $0.20 per 1M requests + $0.0000166667 per GB-second |
| API Gateway HTTP API | $1.00 per 1M requests |
| ALB | ~$0.0225/hr + ~$6/month in LCUs at moderate load (use **$22.50/month** total) |
| Data transfer S3/EC2 → internet | $0.09/GB (first 10 TB), $0.085/GB (next 40 TB) |
| CloudFront → internet (North America) | $0.085/GB (first 10 TB), $0.080/GB (next 40 TB) |
| S3 → CloudFront (origin fetch) | **$0** |
| RDS db.m5.large MySQL Single-AZ | $0.171/hr |
| RDS db.m5.large MySQL Multi-AZ | $0.342/hr |
| EBS gp2 | $0.10/GB-month |
| EBS gp3 | $0.08/GB-month (3,000 IOPS + 125 MB/s included) |

---

## Round 1 — 🥊 NAT Gateway vs. Gateway Endpoint

**Scenario:** Private EC2 instances download **10 TB/month** (use 10,240 GB) from S3 in the same Region.

| Architecture A | Architecture B |
|----------------|----------------|
| Traffic goes through **one NAT gateway** | Traffic goes through an **S3 gateway endpoint** (NAT stays for other traffic) |

🔮 Winner? ________ 🧮 Monthly S3-path cost: A = $______ B = $______

<details>
<summary>📖 Answer key</summary>

- **A:** 730 × $0.045 = $32.85 hourly + 10,240 × $0.045 = $460.80 processing → **≈ $494/month**
- **B:** the S3 path costs **$0** (the NAT's hourly charge would remain for any other traffic, but S3 bytes no longer flow through it)

**Lesson:** NAT gateway **data processing** is one of the most common surprise bills in AWS. Any time a scenario mentions *private instances + S3/DynamoDB + cost*, think **gateway endpoint**. (Lab 2B)

</details>

---

## Round 2 — 🥊 Paying for a Batch Job

**Scenario:** A fault-tolerant analytics batch job runs on **10 × m5.large for 4 hours every day** (≈ 120 hours/month each). It checkpoints and can be retried.

| A: On-Demand | B: 1-yr Compute Savings Plan sized for 10 instances | C: Spot |
|--------------|------------------------------------------------------|---------|

🔮 Winner? ________ 🧮 A = $______ B = $______ C = $______

<details>
<summary>📖 Answer key</summary>

- **A:** 10 × 120 h × $0.096 = **$115.20/month**
- **B:** a Savings Plan is a **24/7 hourly commitment**. To cover 10 × m5.large you commit ~10 × $0.067 = $0.67/hr × 730 h = **$489.10/month**, even though you use it only 4 h/day. ❌ **Worse than On-Demand!**
- **C:** 10 × 120 h × $0.035 = **$42/month** ✅

**Lesson:** Commitments (SPs/RIs) reward **steady 24/7 usage**. Part-time, interruptible work belongs on **Spot**. (Lab 8B)

</details>

---

## Round 3 — 🥊 Serverless vs. Servers (Two Traffic Levels)

**Scenario:** An API averages **200 ms** per request with **512 MB** memory (0.5 GB × 0.2 s = **0.1 GB-s per request**).

| A: API Gateway HTTP API + Lambda | B: ALB + 2 × m5.large On-Demand (enough for both traffic levels) |
|----------------------------------|------------------------------------------------------------------|

Compute both at **3a) 1 million requests/month** and **3b) 300 million requests/month**.

🔮 Winner at 3a? ______ at 3b? ______

<details>
<summary>📖 Answer key</summary>

**B (fixed):** 2 × 730 × $0.096 = $140.16 + ALB $22.50 = **≈ $163/month at any volume**

**A at 1M:** Lambda requests $0.20 + compute 100,000 GB-s × $0.0000166667 = $1.67 + API GW $1.00 → **≈ $2.87** ✅ (often fully covered by the free tier)

**A at 300M:** Lambda requests 300 × $0.20 = $60 + compute 30,000,000 GB-s × $0.0000166667 = $500 + API GW $300 → **≈ $860** ❌ (servers win at ~$163)

**Lesson:** Serverless wins at **low or spiky** traffic (you pay nothing when idle). At **high, steady** volume, provisioned compute (better still with Savings Plans) is often cheaper. The exam's "most cost-effective for **unpredictable / infrequent** traffic" → Lambda.

</details>

---

## Round 4 — 🥊 Serving 50 TB of Downloads

**Scenario:** A software company serves **50 TB/month** (use 51,200 GB) of installers to North American users.

| A: Directly from S3 | B: CloudFront in front of S3 |
|---------------------|------------------------------|

🔮 Winner? ________ 🧮 A = $______ B = $______ (data transfer only)

<details>
<summary>📖 Answer key</summary>

- **A:** 10,240 × $0.09 = $921.60 + 40,960 × $0.085 = $3,481.60 → **≈ $4,403**
- **B:** 10,240 × $0.085 = $870.40 + 40,960 × $0.080 = $3,276.80 → **≈ $4,147**, and **S3 → CloudFront origin transfer is free**, plus caching cuts S3 GET requests ✅

**Lesson:** CloudFront is **cheaper per GB than S3/EC2 direct-to-internet**, *and* faster for users. Two wins at once, which is why it appears in so many "cost + performance" answers.

</details>

---

## Round 5 — 🥊 The Dev Database

**Scenario:** A **dev/test** MySQL database (db.m5.large) is used **weekdays 08:00–20:00** (12 h × 22 days = 264 h/month). Today it runs **Multi-AZ, 24/7**.

| A: Keep Multi-AZ 24/7 | B: Single-AZ, stopped outside working hours (EventBridge Scheduler / Instance Scheduler) |
|-----------------------|------------------------------------------------------------------------------------------|

🔮 Winner? ________ 🧮 A = $______ B = $______ (instance hours only)

<details>
<summary>📖 Answer key</summary>

- **A:** 730 × $0.342 = **$249.66**
- **B:** 264 × $0.171 = **$45.14** ✅ (**−82%**)

**Lesson:** **Non-production doesn't need production resilience.** Scheduling start/stop is one of the biggest easy wins. (Watch out: a stopped RDS instance **automatically restarts after 7 days**, so your scheduler must keep stopping it.)

</details>

---

## Round 6 — 🥊 Storage Volumes

**Scenario:** 50 database servers each have a **1 TB gp2** volume (use 1,024 GB). They need ≤ 3,000 IOPS.

| A: Keep gp2 | B: Migrate to gp3 (Elastic Volumes, no downtime) |
|-------------|--------------------------------------------------|

🔮 Winner? ________ 🧮 A = $______ B = $______

<details>
<summary>📖 Answer key</summary>

- **A:** 50 × 1,024 × $0.10 = **$5,120/month**
- **B:** 50 × 1,024 × $0.08 = **$4,096/month** ✅ (−20%, ~$12k/year, zero downtime)

**Lesson:** **gp2 → gp3** is a classic "most cost-effective with no downtime" answer. (Lab 7B)

</details>

---

## 🧮 Tally Your Score

| Round | 🔮 Prediction (+10) | 🧮 Calculation (+20) |
|-------|--------------------|---------------------|
| 1 | | |
| 2 | | |
| 3 (both parts) | | |
| 4 | | |
| 5 | | |
| 6 | | |
| **Total** | | **/ 180 XP** |

---

## Hands-On: Rebuild Round 1 in the AWS Pricing Calculator

1. Open **[calculator.aws](https://calculator.aws)** → **Create estimate**.
2. Search **"Amazon VPC"** → **Configure**. Region: US East (N. Virginia).
3. Under **NAT Gateway**: 1 gateway, **10 TB** data processed per month.
4. Click **Save and add service**, and look at the monthly cost.
5. Add a second group named **"With gateway endpoint"** and model the same scenario with **0 GB** through NAT.
6. **Share** the estimate and paste the link into your study notes.

**✅ Checkpoint:** Does the calculator's NAT figure roughly match your Round 1 answer? Any difference is the current real price vs. the lab's rounded sheet.

> 💡 The Pricing Calculator is the tool architects use to put cost estimates into design documents. Knowing it exists (and that it's free) is fair game for exam questions.

---

## ⚔️ Boss Challenge: Cut This Bill (+250 XP)

You've joined a startup. Here's last month's bill, **$13,560**. The CFO wants it **down by at least 40%** with **no loss of availability for production**. Find the savings and estimate each one.

| # | Line Item | Monthly Cost | Notes From the Team |
|---|-----------|--------------|---------------------|
| 1 | 30 × m5.xlarge On-Demand (prod web), 24/7 | $4,205 | Steady traffic for 2+ years; may move to Graviton |
| 2 | NAT gateways (3 AZs), mostly S3 + DynamoDB traffic | $2,300 | ~45 TB/month processed |
| 3 | Data transfer out: website images served from EC2 | $2,100 | Global users |
| 4 | RDS Multi-AZ db.r5.2xlarge, **dev** environment, 24/7 | $1,460 | Used office hours only |
| 5 | EBS: 15 TB gp2 | $1,500 | Typical IOPS needs well under 3,000 per volume |
| 6 | S3 Standard: 80 TB of logs | $1,840 | Rarely read after 30 days; 1-year retention |
| 7 | 12 unattached EBS volumes + 8 idle Elastic IPs | $155 | "Forgot about those" |

<details>
<summary>🏆 Answer key</summary>

| # | Fix | Est. Saving |
|---|-----|-------------|
| 1 | **Compute Savings Plan** (1–3 yr) for the steady baseline. It's flexible enough to move to Graviton (which itself is ~20% cheaper per instance). | ~$1,250–2,000 |
| 2 | **S3 + DynamoDB gateway endpoints** (free) → most NAT processing disappears | ~$2,000 |
| 3 | **CloudFront** for images (cheaper per GB, cached, faster) + S3 as the origin | ~$500–900 |
| 4 | Dev → **Single-AZ, scheduled stop** outside office hours (and maybe a smaller class) | ~$1,200 |
| 5 | **gp2 → gp3** (−20%) | ~$300 |
| 6 | **Lifecycle:** Standard → Standard-IA/Glacier IR at 30 days → expire at 365 | ~$1,000–1,200 |
| 7 | **Delete** unattached volumes (snapshot first if unsure) and release idle EIPs. Find them with **Trusted Advisor / Compute Optimizer / Cost Explorer**. | ~$155 |
| | **Total** | **≈ $6,400–7,750 (≈ 47–57%)** ✅ |

**Scoring:** 6–7 correct fixes = 250 XP · 4–5 = 175 XP · 2–3 = 100 XP. **Bonus +50 XP** if you also proposed *preventing* this next time: tagging + Budgets + Anomaly Detection (Lab 8A).

</details>

---

## What You Just Did

1. Ran napkin math on six classic cost trade-offs
2. Learned which line items dominate: NAT processing, data transfer, idle compute, wrong storage tier
3. Used the **AWS Pricing Calculator**
4. Cut a realistic bill by ~50% with exam-standard techniques

---

## 🎯 Exam Corner — The Cost-Optimization Reflexes

| If You See… | Reach For… |
|-------------|-----------|
| Private subnet + S3/DynamoDB + cost | Gateway endpoint |
| Steady 24/7 compute | Savings Plans / RIs |
| Interruptible / batch | Spot |
| Unpredictable or low traffic | Serverless (Lambda, Fargate, DynamoDB on-demand, Aurora Serverless v2) |
| Global content delivery | CloudFront |
| Old/infrequent data | Lifecycle → IA / Glacier |
| Non-prod 24/7 | Schedule stop/start |
| gp2 | gp3 |
| "Which resources are oversized?" | Compute Optimizer |
| Idle / unattached resources | Trusted Advisor |

**🚨 Exam traps**
- The *cheapest* option that **fails a requirement** (availability, latency, durability) is never correct. Check requirements first, then cost.
- Cross-AZ data transfer costs money (~$0.01/GB each way). Chatty microservices spread across AZs can add up.
- Data transfer **into** AWS is free; **out to the internet** is not.

---

## 🧹 Cleanup

Nothing was created in AWS in this lab. 🎉

**Delete the Session 8 local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-session-8`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-session-8`

---

**🏁 Lab complete: +100 XP.** Session 8 done! **[📝 Mini Exam 8 →](mini-exam-08.md)**
