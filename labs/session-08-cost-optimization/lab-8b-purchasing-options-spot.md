# Lab 8B: EC2 Purchasing Options — Spot, Savings Plans, RIs & Mixed-Instance Fleets

**Session:** 8 — Cost Optimization  
**Exam Domain:** Domain 4 — Design Cost-Optimized Architectures (20%)  
**Difficulty:** Intermediate  
**Estimated Time:** 35–45 minutes

---

## Overview

The same `m5.large` can cost **$0.096/hr** (On-Demand), **~$0.06/hr** (Savings Plan) or **~$0.03/hr** (Spot). The exam gives you a workload and asks which purchasing option fits. The answer comes down to **how long it runs**, **how predictable it is**, and **whether it can be interrupted**.

In this lab you'll read live Spot prices, launch a Spot instance, then build the production pattern: an **Auto Scaling group with a mixed-instances policy**, an On-Demand base plus Spot capacity spread across several instance types.

---

## Prerequisites

- ✅ **Session 4** complete (launch templates, ASGs)
- ✅ Default VPC in us-east-1

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| EC2 Spot t3.micro ×3 (briefly) | Spot capacity | ~$0.003–0.004/hr each |
| EC2 On-Demand t3.micro ×1 | Base capacity | ~$0.0104/hr |
| Public IPv4 ×4 | Instance IPs | $0.005/hr each |
| Cost Explorer API | Savings Plans recommendation | $0.01 |

**Estimated cost for this lab: ~$0.03**

---

## Concepts

| Option | Discount vs. On-Demand | Commitment | Flexibility | Use When |
|--------|-----------------------|------------|-------------|----------|
| **On-Demand** | — | None | Total | Short, spiky, unpredictable; testing |
| **Compute Savings Plans** | Up to ~66% | $/hr for 1 or 3 yrs | **Any** instance family, size, Region, OS; also **Fargate and Lambda** | Steady baseline, and you want flexibility |
| **EC2 Instance Savings Plans** | Up to ~72% | $/hr for 1 or 3 yrs | One family in one Region (any size/OS/AZ) | Steady and you know the family |
| **Standard Reserved Instances** | Up to ~72% | 1 or 3 yrs, specific attributes | Low (can sell on the RI Marketplace) | Steady, fixed config; also **RDS, ElastiCache, Redshift, OpenSearch** |
| **Convertible RIs** | Up to ~66% | 1 or 3 yrs | Can exchange for a different family | Steady but may change |
| **Spot** | **Up to ~90%** | None | **Can be interrupted with a 2-minute warning** | **Fault-tolerant**, stateless, flexible: batch, CI, big data, containers |
| **Dedicated Hosts** | — | On-Demand or reserved | A physical server for you | **BYOL** (per-socket/core licenses), compliance |
| **Dedicated Instances** | — | — | Single-tenant hardware | Compliance needing isolated hardware |
| **On-Demand Capacity Reservations** | None (combine with SPs) | None | Reserve capacity in an AZ | **Guaranteed capacity** for a known event, no long-term commitment |

> 🧠 **Memory hook:** *Savings Plans = commit to **spend**. RIs = commit to **configuration**. Spot = commit to **nothing**, and so does AWS.*

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<SUBNET_1A>`, `<SUBNET_1B>` | Default subnets |
| `<AMI_ID>` | Amazon Linux 2023 |
| `<SPOT_INSTANCE_ID>` | Single Spot instance (Step 3) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-8`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-8`

```
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text
```
```
aws ec2 describe-subnets --filters Name=default-for-az,Values=true Name=availability-zone,Values=us-east-1a,us-east-1b --query "Subnets[].[AvailabilityZone,SubnetId]" --output table
```

---

### Step 2: Read the Spot Market

🔮 **Predict:** The On-Demand price of `m5.large` in us-east-1 is about **$0.096/hr**. What's today's Spot price, as a percentage of that?

```
aws ec2 describe-spot-price-history --instance-types m5.large --product-descriptions "Linux/UNIX" --max-items 6 --query "SpotPriceHistory[].[AvailabilityZone,SpotPrice,Timestamp]" --output table
```

<details>
<summary>🔮 Reveal</summary>

Typically **$0.03–0.045/hr**, i.e. **55–70% off**, and it **varies by AZ**. Spot prices move slowly with long-term supply and demand (there's no bidding war anymore). You pay the current Spot price, never more than On-Demand.

</details>

📋 Compare several sizes at once:
```
aws ec2 describe-spot-price-history --instance-types t3.micro t3a.micro m5.large c5.xlarge --product-descriptions "Linux/UNIX" --availability-zone us-east-1a --max-items 4 --query "SpotPriceHistory[].[InstanceType,SpotPrice]" --output table
```

---

### Step 3: Launch a Single Spot Instance

```
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro --subnet-id <SUBNET_1A> --instance-market-options "MarketType=spot" --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=saa-spot-single}]" --query "Instances[0].[InstanceId,InstanceLifecycle,SpotInstanceRequestId]" --output text
```

**✅ You should see** an instance ID, `spot` and a Spot request ID (`sir-…`).

> **📝 Save as `<SPOT_INSTANCE_ID>`**

📋 The Spot request behind it:
```
aws ec2 describe-spot-instance-requests --filters Name=instance-id,Values=<SPOT_INSTANCE_ID> --query "SpotInstanceRequests[0].[State,Status.Code,Type,InstanceInterruptionBehavior]" --output text
```

**✅ You should see** `active  fulfilled  one-time  terminate`.

> 💡 **The 2-minute warning.** When AWS needs the capacity back, the instance gets an **interruption notice** (via instance metadata and an **EventBridge** event) 2 minutes before it's terminated (or stopped/hibernated, if configured). Well-built Spot workloads **checkpoint** progress and drain work when they see it, and AWS also sends a **rebalance recommendation** even earlier when risk rises.

📋 Done with it:
```
aws ec2 terminate-instances --instance-ids <SPOT_INSTANCE_ID>
```

### 🧩 Checkpoint

A nightly ETL job takes 3 hours, can be **restarted from checkpoints**, and must finish by 6 AM. Which purchasing option?

<details>
<summary>Answer (+10 XP)</summary>

**Spot Instances**, diversified across several instance types and AZs, with checkpointing. Optionally keep a small On-Demand base, or fall back to On-Demand if the deadline is at risk. Interruptions only cost a retry from the last checkpoint.

</details>

---

### Step 4: Production Pattern — Mixed-Instances Auto Scaling Group

Real Spot fleets are built from **many instance types** (so one type's interruption doesn't take everything down) and an **On-Demand base** for the must-not-fail minimum.

**Create `spot-lt.json`**, **replacing `<AMI_ID>`**:
```json
{
  "ImageId": "<AMI_ID>",
  "MetadataOptions": { "HttpTokens": "required" },
  "TagSpecifications": [
    { "ResourceType": "instance", "Tags": [ { "Key": "Name", "Value": "saa-mixed-fleet" } ] }
  ]
}
```
```
aws ec2 create-launch-template --launch-template-name saa-spot-lt --launch-template-data file://spot-lt.json
```

**Create `mixed.json`:**
```json
{
  "LaunchTemplate": {
    "LaunchTemplateSpecification": { "LaunchTemplateName": "saa-spot-lt", "Version": "$Latest" },
    "Overrides": [
      { "InstanceType": "t3.micro" },
      { "InstanceType": "t3a.micro" },
      { "InstanceType": "t2.micro" }
    ]
  },
  "InstancesDistribution": {
    "OnDemandBaseCapacity": 1,
    "OnDemandPercentageAboveBaseCapacity": 0,
    "SpotAllocationStrategy": "price-capacity-optimized"
  }
}
```

> 💡 **Read it as:** "The first **1** instance is On-Demand. **0%** of everything above that is On-Demand, so it's **all Spot**. Choose Spot pools with the best mix of **low price and deep capacity** (`price-capacity-optimized`, AWS's recommended strategy)."

🔮 **Predict:** With desired capacity **3**, how many On-Demand and how many Spot instances?

📋 Create the ASG, **replacing the subnets**:
```
aws autoscaling create-auto-scaling-group --auto-scaling-group-name saa-mixed-asg --mixed-instances-policy file://mixed.json --min-size 0 --max-size 3 --desired-capacity 3 --vpc-zone-identifier "<SUBNET_1A>,<SUBNET_1B>"
```

Wait ~60 seconds, then:
```
aws ec2 describe-instances --filters Name=tag:aws:autoscaling:groupName,Values=saa-mixed-asg Name=instance-state-name,Values=pending,running --query "Reservations[].Instances[].[InstanceId,InstanceType,Placement.AvailabilityZone,InstanceLifecycle]" --output table
```

<details>
<summary>🔮 Reveal</summary>

**1 On-Demand + 2 Spot.** In the output, `InstanceLifecycle` is `spot` for Spot instances and **empty/None** for On-Demand. You'll likely also see **different instance types and AZs**: diversification in action.

</details>

---

### Step 5: Ask AWS for a Savings Plans Recommendation

```
aws ce get-savings-plans-purchase-recommendation --savings-plans-type COMPUTE_SP --term-in-years ONE_YEAR --payment-option NO_UPFRONT --lookback-period-in-days THIRTY_DAYS --query "SavingsPlansPurchaseRecommendation.SavingsPlansPurchaseRecommendationSummary"
```

🔮 **Predict:** Will AWS recommend buying a Savings Plan for your study account?

<details>
<summary>🔮 Reveal</summary>

Almost certainly **no recommendation** (an empty or null summary). Your usage is short and bursty, with no steady baseline to commit to. That's the right outcome: **commitments only pay off for steady, predictable usage**. Buying one for spiky lab usage would *waste* money.

</details>

---

### Step 6: 🎮 Purchase-Option Picker (+10 XP each)

| # | Workload | Best Option |
|---|----------|------------|
| 1 | Production web tier: 10 instances 24/7 for the next 3 years; may move from x86 to Graviton next year | |
| 2 | Production **RDS** database, steady for 3 years | |
| 3 | CI/CD build agents, short jobs, retry is fine | |
| 4 | Software licensed **per physical CPU socket** that must run on EC2 | |
| 5 | A 2-week product launch in November needing guaranteed capacity in a specific AZ, no long-term commitment | |
| 6 | Steady Lambda + Fargate + EC2 spend of ~$50/hr, likely to shift between them | |
| 7 | A dev environment used weekdays 9–5, unpredictable sizes | |

<details>
<summary>🎮 Reveal</summary>

1. **Compute Savings Plan (3-yr)**: steady, but the family will change, so you want maximum flexibility.
2. **RDS Reserved Instances**: Compute Savings Plans don't cover RDS.
3. **Spot**
4. **Dedicated Host**: per-socket BYOL needs visibility into the physical server.
5. **On-Demand Capacity Reservation**: capacity without a 1–3 year commitment.
6. **Compute Savings Plan**: the only commitment that spans EC2, Fargate *and* Lambda.
7. **On-Demand**, plus **scheduling** (stop nights and weekends, saving ~70%). Commitments don't fit part-time, unpredictable usage.

</details>

---

### Step 7: Console Checkpoint

**✅ Checkpoint:**
1. **EC2 → Spot Requests**: your one-time request shows `closed` / `instance-terminated-by-user`.
2. **EC2 → Auto Scaling Groups → saa-mixed-asg → Details → Instance type requirements / Purchase options**: base 1 On-Demand, 100% Spot above.
3. **EC2 → Spot Requests → Savings summary**: how much Spot saved you vs. On-Demand.

---

## What You Just Did

1. Read live **Spot prices** and compared them to On-Demand
2. Launched a **Spot instance** and saw the interruption model
3. Built a **mixed-instances ASG** (On-Demand base + diversified Spot)
4. Saw why commitment-based discounts need **steady usage**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Steady-state, 1–3 years, maximum flexibility across families/Regions/Lambda/Fargate" | **Compute Savings Plan** |
| "Steady, fixed instance family in one Region, max discount" | **EC2 Instance Savings Plan** or **Standard RI** |
| "Interruptible / fault-tolerant / batch / stateless / flexible start time" | **Spot** |
| "Must not be interrupted, short-term or unpredictable" | **On-Demand** |
| "BYOL per socket/core, server-bound licenses" | **Dedicated Hosts** |
| "Reserve capacity for an event without a long commitment" | **On-Demand Capacity Reservation** |
| "Reduce cost of steady RDS/ElastiCache/Redshift" | **Reserved Instances / reserved nodes** for that service |

**🚨 Exam traps**
- **Spot for databases** or anything stateful that can't handle interruption = wrong.
- **Compute/EC2 Savings Plans don't cover RDS.** For RDS, the classic exam answer is **Reserved Instances**. AWS has since announced Database Savings Plans, so check the current docs, but expect the exam to say RIs.
- Dedicated **Instances** ≠ Dedicated **Hosts**. Only Hosts give you socket/core visibility for licensing.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InsufficientInstanceCapacity` / Spot not fulfilled | No Spot capacity for that type/AZ | Add more `Overrides` or AZs. That's exactly why you diversify. |
| `MaxSpotInstanceCountExceeded` | Account Spot quota is low (new accounts) | Use fewer instances, or request a quota increase |
| All instances show On-Demand | Spot couldn't be fulfilled | Check the ASG **Activity** tab for the reason |

---

## 🧹 Cleanup

```
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name saa-mixed-asg --force-delete
```
```
aws ec2 delete-launch-template --launch-template-name saa-spot-lt
```

**✅ Checkpoint:** **EC2 → Instances**: all `saa-mixed-fleet` and `saa-spot-single` instances are **terminated** (allow ~2 minutes).

---

**🏁 Lab complete: +100 XP.** Next: [Lab 8C — Architecture Cost Showdown](lab-8c-architecture-cost-showdown.md)
