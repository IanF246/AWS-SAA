# Lab 8A: Cost Visibility — Cost Explorer, Allocation Tags & Anomaly Detection

**Session:** 8 — Cost Optimization  
**Exam Domain:** Domain 4 — Design Cost-Optimized Architectures (20%)  
**Difficulty:** Beginner  
**Estimated Time:** 30–40 minutes

---

## Overview

You can't optimize what you can't see. Before any purchasing trick or architecture change, a cost-aware architect answers three questions: **Where is the money going? Who is spending it? Will I find out quickly if it spikes?**

In this lab you'll query your own bill from the CLI with **Cost Explorer**, forecast the month, tag resources so costs can be **allocated to teams/projects**, and set up **Cost Anomaly Detection** to email you when spending looks unusual. Your study account has been through 7 sessions of labs, so there's real data to look at.

---

## Prerequisites

- ✅ Sessions 1–7 complete (so your bill has some history)
- ✅ An email address for alerts

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Cost Explorer (console) | Interactive bill analysis | Free |
| Cost Explorer **API** | `aws ce …` CLI calls | **$0.01 per request** (this lab: ~6 calls ≈ $0.06) |
| Cost Anomaly Detection | ML-based spend monitoring | Free |
| S3 bucket with tags | Tagging target | $0.00 |

**Estimated cost for this lab: ~$0.06**

---

## Concepts

| Tool | Answers |
|------|---------|
| **Cost Explorer** | "What did I spend, on what, and when?" Up to 13 months of history (opt-in for 38), with forecasts |
| **AWS Budgets** | "Alert me (or **take an action**) when cost/usage crosses X." You built one in AI Cloud Fusion Lab 1B. |
| **Cost Anomaly Detection** | "Tell me when spend is **unusual**," with no threshold to guess |
| **Cost allocation tags** | "Which team/project/env spent it?" Tags must be **activated** in Billing before they show up in cost reports |
| **Cost and Usage Report (CUR 2.0 / Data Exports)** | The most detailed line-item billing data, delivered to S3 for Athena/QuickSight |
| **AWS Compute Optimizer** | "Which instances, volumes and Lambdas are over- or under-provisioned?" |
| **Trusted Advisor** | Checks for idle resources, unassociated EIPs, low-utilization instances (full checks need Business Support or higher) |

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is | Example |
|-------------|-----------|---------|
| `<MONTH_START>` | First day of the current month | `2026-10-01` |
| `<TOMORROW>` | Tomorrow's date (Cost Explorer's End date is exclusive) | `2026-10-08` |
| `<NEXT_MONTH_START>` | First day of next month | `2026-11-01` |
| `<SIX_MONTHS_AGO>` | First day of the month 6 months ago | `2026-04-01` |
| `<ACCOUNT_ID>` | Your account ID | |
| `<YOUR_EMAIL>` | Where anomaly alerts go | |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-session-8; cd ~\Desktop\saa-session-8; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-session-8 && cd ~/Desktop/saa-session-8 && code .
```

---

### Step 2: 🔮 Guess Your Bill First

Before you look: write down your guesses.

| Question | Your Guess |
|----------|-----------|
| Total spend this month so far | $ |
| The single most expensive service | |
| End-of-month forecast | $ |

---

### Step 3: Where Is the Money Going? (Cost Explorer API)

📋 This month, grouped by service (**replace the dates**):
```
aws ce get-cost-and-usage --time-period Start=<MONTH_START>,End=<TOMORROW> --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE --query "ResultsByTime[0].Groups[].[Keys[0],Metrics.UnblendedCost.Amount]" --output table
```

📋 Daily totals, so you can spot which lab days cost the most:
```
aws ce get-cost-and-usage --time-period Start=<MONTH_START>,End=<TOMORROW> --granularity DAILY --metrics UnblendedCost --query "ResultsByTime[].[TimePeriod.Start,Total.UnblendedCost.Amount]" --output table
```

📋 Forecast to the end of the month:
```
aws ce get-cost-forecast --time-period Start=<TOMORROW>,End=<NEXT_MONTH_START> --metric UNBLENDED_COST --granularity MONTHLY --query "Total.Amount" --output text
```

> ⚠️ The forecast needs some history. A brand-new account may return `DataUnavailableException`. That's fine; skip it.

**Score your guesses** from Step 2: +10 XP for each within 25% (or for naming the top service correctly).

### 🧩 Checkpoint

Your top service is **"EC2 - Other"**, even though your instances were tiny. What is "EC2 - Other" usually made of?

<details>
<summary>Answer (+10 XP)</summary>

**EBS volumes and snapshots, NAT gateway hours and data processing, Elastic/public IPv4 addresses, and data transfer.** It's where the "hidden" networking and storage costs live, and where forgotten resources (unattached volumes, idle NAT gateways, unused Elastic IPs) quietly add up.

</details>

---

### Step 4: Who Spent It? Tag Resources

📋 Create a tagged resource:
```
aws s3 mb s3://saa-lab8a-<ACCOUNT_ID> --region us-east-1
```
```
aws s3api put-bucket-tagging --bucket saa-lab8a-<ACCOUNT_ID> --tagging "TagSet=[{Key=Project,Value=saa-quest},{Key=Environment,Value=study},{Key=Owner,Value=me}]"
```

📋 Find every resource in the Region with a given tag, across services:
```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Project,Values=saa-quest --query "ResourceTagMappingList[].ResourceARN"
```

📋 Try to **activate** `Project` as a **cost allocation tag**:
```
aws ce update-cost-allocation-tags-status --cost-allocation-tags-status TagKey=Project,Status=Active
```

🔮 **Predict:** Will this succeed right now?

<details>
<summary>🔮 Reveal</summary>

Probably ❌ **not yet**. A tag key only becomes available for activation **after AWS's billing pipeline has seen it**, which takes up to 24 hours. Retry tomorrow. Once it's active, tag-based cost data appears **from that point forward**, never retroactively. 🚨 That's why a **tagging strategy must exist before resources are created**, ideally enforced with **tag policies** and SCPs (Lab 1C).

</details>

> 💡 **AWS-generated tags** like `aws:createdBy` can also be activated, and they show who created each resource.

---

### Step 5: Will I Find Out If It Spikes? Cost Anomaly Detection

📋 Check whether AWS already created a default monitor for you:
```
aws ce get-anomaly-monitors --query "AnomalyMonitors[].[MonitorName,MonitorType,MonitorDimension,MonitorArn]" --output table
```

**If you see a monitor with dimension `SERVICE`**, copy its ARN as `<MONITOR_ARN>` and **skip to the subscription** below. (Only one service monitor is allowed per account.)

**Otherwise**, create one:
```
aws ce create-anomaly-monitor --anomaly-monitor MonitorName=saa-services-monitor,MonitorType=DIMENSIONAL,MonitorDimension=SERVICE --query MonitorArn --output text
```

> **📝 Save as `<MONITOR_ARN>`**

**Create `anomaly-sub.json`**, **replacing `<MONITOR_ARN>` and `<YOUR_EMAIL>`**. It means "email me daily about anomalies with ≥ $5 total impact":
```json
{
  "SubscriptionName": "saa-anomaly-alerts",
  "MonitorArnList": ["<MONITOR_ARN>"],
  "Subscribers": [ { "Address": "<YOUR_EMAIL>", "Type": "EMAIL" } ],
  "Frequency": "DAILY",
  "ThresholdExpression": {
    "Dimensions": {
      "Key": "ANOMALY_TOTAL_IMPACT_ABSOLUTE",
      "Values": ["5"],
      "MatchOptions": ["GREATER_THAN_OR_EQUAL"]
    }
  }
}
```
```
aws ce create-anomaly-subscription --anomaly-subscription file://anomaly-sub.json --query SubscriptionArn --output text
```

> **📝 Save as `<SUBSCRIPTION_ARN>`.** Keep this one running after the lab. It's free, and it's your safety net for the rest of your studies.

---

### Step 6: Console Exploration (Interactive)

Spend 5–10 minutes in the console. It's where the exam's "which tool?" intuition comes from.

1. **Billing and Cost Management → Cost Explorer**: switch to **Group by: Service**, then **Usage type**. Find your NAT/endpoint/EIP line items from Session 2.
2. **Cost Explorer → Reports → Savings Plans recommendations**: probably empty (no steady usage). Note where it lives.
3. **Billing → Cost allocation tags**: search for `Project`. Is it there yet?
4. **Trusted Advisor → Cost optimization**: which checks are available on your support plan?
5. **Compute Optimizer**: click **Opt in** (free). Recommendations need ~12+ hours of metrics.

---

## What You Just Did

1. Queried spend by **service** and by **day**, and forecast the month, from the CLI
2. Tagged resources and learned the **activation** delay for cost allocation tags
3. Set up free, ML-based **Cost Anomaly Detection** with email alerts
4. Toured the cost tools the exam names

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Visualize and analyze costs over time, forecast" | **Cost Explorer** |
| "Alert when spending exceeds a threshold" | **AWS Budgets** |
| "Automatically stop resources or apply an SCP when the budget is exceeded" | **Budgets actions** |
| "Detect unusual spending patterns automatically" | **Cost Anomaly Detection** |
| "Allocate costs to departments/projects" | **Cost allocation tags** (activated) + Cost Categories |
| "Most granular billing data for custom analysis in Athena" | **Cost and Usage Report (Data Exports)** |
| "Right-size over-provisioned EC2 instances" | **Compute Optimizer** |
| "Consolidated billing, volume discounts across accounts" | **AWS Organizations** |

**🚨 Exam traps**
- Cost allocation tags are **not retroactive** and must be **activated** in the management (payer) account.
- **Budgets** alert on thresholds you set; **Anomaly Detection** learns your normal pattern. Pick by the scenario's wording.
- Cost Explorer API calls **cost money** ($0.01 each); the console is free.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `AccessDeniedException` on `ce` calls | IAM Identity Center permission set lacks billing access, or billing access for IAM is disabled | Use the admin permission set; or root → Account → *IAM user and role access to billing* → Activate |
| `ValidationException: Start date must be before end date` | Dates wrong or End = Start | End is **exclusive**; use tomorrow's date |
| `LimitExceededException` creating a monitor | A SERVICE monitor already exists | Reuse its ARN (Step 5) |
| `Tag keys not found` | Billing hasn't seen the tag yet | Retry in 24 hours |

---

## 🧹 Cleanup

```
aws s3 rb s3://saa-lab8a-<ACCOUNT_ID> --force
```

**Keep** the anomaly monitor and subscription; they're free and protect you. If you really want them gone:
```
aws ce delete-anomaly-subscription --subscription-arn <SUBSCRIPTION_ARN>
```
```
aws ce delete-anomaly-monitor --monitor-arn <MONITOR_ARN>
```

> Keep the `saa-session-8` folder for Labs 8B and 8C.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 8B — Spot, Savings Plans & Purchasing Options](lab-8b-purchasing-options-spot.md)
