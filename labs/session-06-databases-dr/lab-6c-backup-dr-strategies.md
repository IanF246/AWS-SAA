# Lab 6C: Backup & Disaster Recovery — RPO, RTO and Cross-Region Copies

**Session:** 6 — Databases & Disaster Recovery  
**Exam Domain:** Domain 2 — Design Resilient Architectures (26%)  
**Difficulty:** Advanced  
**Estimated Time:** 50–60 minutes

---

## Overview

High availability (Multi-AZ, ASGs) keeps you running when **a piece** fails. **Disaster recovery** is about what happens when **a whole Region** has a problem, or when someone deletes the production table at 3 AM.

In this lab you'll:
1. **Recover from a human error**: back up a DynamoDB table, "accidentally" delete data, and restore it.
2. **Build centralized, cross-Region backups** with **AWS Backup**: a backup plan, tag-based selection and a **copy to us-west-2**.
3. **Play the DR Strategy Game**: match business requirements to the four AWS DR strategies.

---

## Prerequisites

- ✅ **Labs 6A and 6B** complete
- ✅ The `saa-session-6` folder from Lab 6B

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| DynamoDB on-demand + on-demand backup | Small table + backup | ~$0.00 |
| EBS gp3 volume, 1 GiB, ~1 hour | Backup source | ~$0.0001 |
| AWS Backup (EBS snapshot, 2 Regions) | Recovery points | ~$0.05/GB-month, prorated (≈ $0) |
| Cross-Region data transfer | Copy to us-west-2 | $0.02/GB (an empty 1 GiB volume ≈ $0) |

**Estimated cost for this lab: ~$0.01**

---

## Concepts

**RPO and RTO:**

```
   last good backup              DISASTER                  back in business
         │◀──────── RPO ────────▶│◀────────── RTO ──────────▶│
         │   data you LOSE       │   time you're DOWN         │
```

- **RPO (Recovery Point Objective):** how much data loss is acceptable, measured in time.
- **RTO (Recovery Time Objective):** how long the business can be down.

**The four AWS DR strategies** (memorize this table):

| Strategy | What Runs in the DR Region | RPO / RTO | Cost |
|----------|---------------------------|-----------|------|
| **Backup & Restore** | Nothing, just backups | Hours / hours–day | 💲 |
| **Pilot Light** | Core **data** replicated live (DB replica); compute **off** (AMIs/IaC ready) | Minutes / tens of minutes | 💲💲 |
| **Warm Standby** | A **scaled-down but running** copy of the full stack | Seconds–minutes / minutes | 💲💲💲 |
| **Multi-Site Active/Active** | **Full** production in 2+ Regions, all serving traffic | Near zero / near zero | 💲💲💲💲 |

> 🧠 **Memory hook:** *Pilot light* = only the flame (data) is lit. *Warm standby* = the engine is idling. *Active/active* = both engines at full throttle.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<ACCOUNT_ID>` | Your account ID |
| `<DDB_BACKUP_ARN>` | DynamoDB backup ARN (Step 3) |
| `<VOLUME_ID>` | EBS volume ID (Step 7) |
| `<PLAN_ID>` | Backup plan ID (Step 8) |
| `<SELECTION_ID>` | Backup selection ID (Step 8) |
| `<BACKUP_JOB_ID>`, `<RP_ARN_EAST>` | Backup job and recovery point (Step 9) |
| `<COPY_JOB_ID>`, `<RP_ARN_WEST>` | Copy job and copied recovery point (Step 10) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-6`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-6`

---

## Part 1 — Recover from "Oops, I Deleted Production"

### Step 2: Create and Fill an Inventory Table

```
aws dynamodb create-table --table-name saa-inventory --attribute-definitions AttributeName=sku,AttributeType=S --key-schema AttributeName=sku,KeyType=HASH --billing-mode PAY_PER_REQUEST
```
```
aws dynamodb wait table-exists --table-name saa-inventory
```

**Create `inventory.json`:**
```json
{
  "saa-inventory": [
    { "PutRequest": { "Item": { "sku": {"S": "SKU-001"}, "name": {"S": "Mechanical keyboard"}, "stock": {"N": "42"} } } },
    { "PutRequest": { "Item": { "sku": {"S": "SKU-002"}, "name": {"S": "4K monitor"},          "stock": {"N": "7"} } } },
    { "PutRequest": { "Item": { "sku": {"S": "SKU-003"}, "name": {"S": "USB-C dock"},          "stock": {"N": "19"} } } }
  ]
}
```
```
aws dynamodb batch-write-item --request-items file://inventory.json
```

---

### Step 3: Take an On-Demand Backup

```
aws dynamodb create-backup --table-name saa-inventory --backup-name saa-inventory-before-deploy --query "BackupDetails.[BackupArn,BackupStatus]" --output text
```

> **📝 Save as `<DDB_BACKUP_ARN>`**
>
> 💡 DynamoDB backups are **instant** and have **no performance impact**. Good practice: take one before every risky deployment.

---

### Step 4: The Disaster 😱

A buggy script runs in production. **Create `oops.json`:**
```json
{
  "saa-inventory": [
    { "DeleteRequest": { "Key": { "sku": {"S": "SKU-001"} } } },
    { "DeleteRequest": { "Key": { "sku": {"S": "SKU-002"} } } }
  ]
}
```
```
aws dynamodb batch-write-item --request-items file://oops.json
```
```
aws dynamodb scan --table-name saa-inventory --query "Items[].sku.S"
```

**✅ You should see** only `SKU-003` left. Two products are gone.

🔮 **Predict:** When you restore the backup, does DynamoDB overwrite the existing `saa-inventory` table?

---

### Step 5: Restore

```
aws dynamodb restore-table-from-backup --target-table-name saa-inventory-restored --backup-arn <DDB_BACKUP_ARN> --query "TableDescription.TableStatus" --output text
```
```
aws dynamodb wait table-exists --table-name saa-inventory-restored
```
```
aws dynamodb scan --table-name saa-inventory-restored --query "Items[].[sku.S,name.S,stock.N]" --output table
```

<details>
<summary>🔮 Reveal</summary>

**No.** DynamoDB (and RDS) **always restore into a NEW table/instance**. You then either repoint the app or copy the missing items back. All 3 products are in `saa-inventory-restored`. Restore takes a few minutes, which is part of your **RTO**.

</details>

### 🧩 Checkpoint

The backup was taken at 09:00. The bad script ran at 14:30. With **on-demand backups only**, what's your data loss (RPO)? How would PITR change that?

<details>
<summary>Answer (+10 XP)</summary>

You lose everything written between 09:00 and 14:30, an **RPO of 5.5 hours**. With **PITR** you restore to **14:29:59**, an RPO of about a second, with up to 35 days of history.

</details>

---

## Part 2 — Centralized, Cross-Region Backups with AWS Backup

### Step 6: Role and Vaults

**Create `backup-trust.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "backup.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```
aws iam create-role --role-name saa-backup-role --assume-role-policy-document file://backup-trust.json
```
```
aws iam attach-role-policy --role-name saa-backup-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup
```
```
aws iam attach-role-policy --role-name saa-backup-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores
```

📋 One vault in each Region:
```
aws backup create-backup-vault --backup-vault-name saa-vault --region us-east-1
```
```
aws backup create-backup-vault --backup-vault-name saa-dr-vault --region us-west-2
```

> 💡 A **backup vault** is an encrypted container for recovery points, with its own access policy. Add **Vault Lock** (WORM, like S3 Object Lock in Lab 3B) to stop even admins from deleting backups, which is a major ransomware defense.

---

### Step 7: Create a Resource to Protect

A tiny EBS volume, tagged for backup:
```
aws ec2 create-volume --availability-zone us-east-1a --size 1 --volume-type gp3 --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=saa-app-data},{Key=backup,Value=daily}]" --query VolumeId --output text
```

> **📝 Save as `<VOLUME_ID>`**

---

### Step 8: Backup Plan + Tag-Based Selection

**Create `plan.json`**, **replacing `<ACCOUNT_ID>`**:
```json
{
  "BackupPlanName": "saa-daily-with-dr-copy",
  "Rules": [
    {
      "RuleName": "daily-5am-utc",
      "TargetBackupVaultName": "saa-vault",
      "ScheduleExpression": "cron(0 5 ? * * *)",
      "StartWindowMinutes": 60,
      "Lifecycle": { "DeleteAfterDays": 7 },
      "CopyActions": [
        {
          "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:<ACCOUNT_ID>:backup-vault:saa-dr-vault",
          "Lifecycle": { "DeleteAfterDays": 7 }
        }
      ]
    }
  ]
}
```

```
aws backup create-backup-plan --backup-plan file://plan.json --query BackupPlanId --output text
```

> **📝 Save as `<PLAN_ID>`**

**Create `selection.json`**, **replacing `<ACCOUNT_ID>`**. It means "back up anything tagged `backup=daily`":
```json
{
  "SelectionName": "tagged-daily",
  "IamRoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/saa-backup-role",
  "ListOfTags": [
    { "ConditionType": "STRINGEQUALS", "ConditionKey": "backup", "ConditionValue": "daily" }
  ]
}
```
```
aws backup create-backup-selection --backup-plan-id <PLAN_ID> --backup-selection file://selection.json --query SelectionId --output text
```

> **📝 Save as `<SELECTION_ID>`**
>
> 💡 **Tag-based selection** = governance at scale. Any new resource tagged `backup=daily` (EBS, RDS, DynamoDB, EFS, EC2 and more) is protected automatically, with no per-resource setup.

---

### Step 9: Don't Wait Until 5 AM — Run an On-Demand Backup Job

```
aws backup start-backup-job --backup-vault-name saa-vault --resource-arn arn:aws:ec2:us-east-1:<ACCOUNT_ID>:volume/<VOLUME_ID> --iam-role-arn arn:aws:iam::<ACCOUNT_ID>:role/saa-backup-role --query BackupJobId --output text
```

> **📝 Save as `<BACKUP_JOB_ID>`**

📋 Poll until `COMPLETED` (usually 2–5 minutes):
```
aws backup describe-backup-job --backup-job-id <BACKUP_JOB_ID> --query "[State,RecoveryPointArn]" --output text
```

> **📝 Save the recovery point ARN as `<RP_ARN_EAST>`** (it looks like `arn:aws:ec2:us-east-1::snapshot/snap-…`)

---

### Step 10: Copy It to Another Region

🔮 **Predict:** A whole-Region outage hits us-east-1. Is your backup in `saa-vault` still usable?

<details>
<summary>🔮 Reveal</summary>

**Not while the Region is down.** The recovery point lives *in* us-east-1. That's why the plan has a **CopyAction**, and why you'll copy this one right now.

</details>

```
aws backup start-copy-job --recovery-point-arn <RP_ARN_EAST> --source-backup-vault-name saa-vault --destination-backup-vault-arn arn:aws:backup:us-west-2:<ACCOUNT_ID>:backup-vault:saa-dr-vault --iam-role-arn arn:aws:iam::<ACCOUNT_ID>:role/saa-backup-role --query CopyJobId --output text
```
```
aws backup describe-copy-job --copy-job-id <COPY_JOB_ID> --query "CopyJob.[State,DestinationRecoveryPointArn]" --output text
```

Repeat until `COMPLETED` (~3–10 min). **📝 Save the destination ARN as `<RP_ARN_WEST>`.**

📋 Confirm it's in Oregon:
```
aws backup list-recovery-points-by-backup-vault --backup-vault-name saa-dr-vault --region us-west-2 --query "RecoveryPoints[].[RecoveryPointArn,Status,ResourceType]" --output table
```

**✅ You should see** one `COMPLETED` EBS recovery point in **us-west-2**. Your data now survives the loss of us-east-1. 🎉

---

## Part 3 — 🎮 The DR Strategy Game (+10 XP per correct answer)

For each company, choose **Backup & Restore**, **Pilot Light**, **Warm Standby** or **Multi-Site Active/Active**. Write your answers down first, then reveal.

| # | Company | Requirement |
|---|---------|-------------|
| 1 | Internal HR wiki | RTO 24 h, RPO 24 h, minimize cost |
| 2 | Online bank | RTO ≈ 0, RPO ≈ 0, global customers, budget is not a concern |
| 3 | SaaS ticketing app | RTO 15 min, RPO 1 min; must handle reduced traffic immediately while scaling up |
| 4 | E-commerce back office | RTO 1 h, RPO 5 min; keep DR cost low, but the database must be current |
| 5 | Media archive (30 TB) | RTO 48 h, RPO 24 h |
| 6 | Trading platform | Must serve users from two Regions simultaneously for latency *and* resilience |

<details>
<summary>🎮 Reveal answers</summary>

1. **Backup & Restore**: generous objectives, lowest cost.
2. **Multi-Site Active/Active**: near-zero objectives (e.g. Aurora Global Database or DynamoDB global tables, Route 53 latency/failover routing).
3. **Warm Standby**: a scaled-down stack already **running** can take traffic immediately, then scale out.
4. **Pilot Light**: the DB is replicated live (low RPO), and compute starts from AMIs/IaC on demand (1 h RTO is OK).
5. **Backup & Restore**: large and slow-changing, with long objectives. Copy S3 cross-Region, or use Glacier.
6. **Multi-Site Active/Active**: "serve from two Regions simultaneously" is the definition.

</details>

---

## ⚔️ Boss Challenge: DR Runbook (+250 XP)

**Scenario:** A web app in **us-east-1** runs on ALB + ASG (EC2) + **Aurora MySQL** + S3 (user uploads). The business sets **RTO 30 minutes, RPO 1 minute**, and asks for the **lowest-cost** design that meets them. Write: (a) the strategy, (b) what you'd deploy in **us-west-2** per component, (c) how traffic fails over.

<details>
<summary>🏆 Answer key</summary>

(a) **Pilot Light**, or a minimal **Warm Standby** if 30 min is too tight to boot from scratch. (+50)

(b) Per component:
- **Aurora:** **Aurora Global Database** with a secondary cluster in us-west-2 (typically < 1 s replication lag, meets RPO 1 min; managed failover in minutes). (+60)
- **S3:** **Cross-Region Replication** to a us-west-2 bucket (optionally S3 RTC for a 15-min SLA). (+40)
- **Compute:** AMIs copied to us-west-2 + **launch template + ASG with desired = 0** (pilot light) or 1–2 (warm standby), ideally defined in **CloudFormation/Terraform**. (+40)
- **Backups:** AWS Backup with a cross-Region copy as the last line of defense. (+20)

(c) **Route 53 failover routing** with a **health check** on the primary ALB. On failure: promote the Aurora secondary, scale the DR ASG, and DNS fails over. Test it with game days! (+40)

</details>

---

## What You Just Did

1. Recovered deleted data from a **DynamoDB backup**, which always restores into a new table
2. Built an **AWS Backup plan** with lifecycle, **tag-based selection** and a **cross-Region copy**
3. Ran a backup **and** copied it to us-west-2 on demand
4. Mapped business RTO/RPO to the **four DR strategies**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Centrally manage and automate backups across services and accounts" | **AWS Backup** (+ Organizations backup policies) |
| "Backups must be immutable, even from administrators" | **AWS Backup Vault Lock** (compliance mode) |
| "Restore DynamoDB to any second in the last 35 days" | **PITR** |
| "Copy backups to another Region / account automatically" | **AWS Backup copy actions** |
| "Lowest-cost DR, hours of downtime acceptable" | **Backup & Restore** |
| "Minimal running infra in DR, DB kept in sync" | **Pilot Light** |
| "Scaled-down full copy running in DR" | **Warm Standby** |
| "Near-zero RTO/RPO" | **Multi-Site Active/Active** |
| "Replicate on-prem servers to AWS for DR with minimal downtime" | **AWS Elastic Disaster Recovery (DRS)** |

**🚨 Exam traps**
- **Multi-AZ is not DR** for a Region-level event. It protects against AZ failures only.
- Snapshots and recovery points are **regional**. Without a copy, a Region outage takes them offline too.
- RPO is about **data**; RTO is about **time**. The exam swaps them in distractors.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Backup job `FAILED`: role can't be assumed | Role propagation | Wait 30 s, re-run Step 9 |
| Copy job `FAILED` with access denied | The us-west-2 vault doesn't exist or the ARN is wrong | `aws backup list-backup-vaults --region us-west-2` |
| `start-backup-job` says the resource type isn't opted in | EBS isn't enabled in AWS Backup settings | `aws backup update-region-settings --resource-type-opt-in-preference EBS=true` |

---

## 🧹 Cleanup — All of Session 6

**1. AWS Backup recovery points** (must go before the vaults):
```
aws backup delete-recovery-point --backup-vault-name saa-dr-vault --recovery-point-arn <RP_ARN_WEST> --region us-west-2
```
```
aws backup delete-recovery-point --backup-vault-name saa-vault --recovery-point-arn <RP_ARN_EAST>
```
Wait ~1 minute, then:
```
aws backup delete-backup-vault --backup-vault-name saa-dr-vault --region us-west-2
aws backup delete-backup-selection --backup-plan-id <PLAN_ID> --selection-id <SELECTION_ID>
aws backup delete-backup-plan --backup-plan-id <PLAN_ID>
aws backup delete-backup-vault --backup-vault-name saa-vault
```

**2. EBS volume:**
```
aws ec2 delete-volume --volume-id <VOLUME_ID>
```

**3. DynamoDB:**
```
aws dynamodb delete-table --table-name saa-inventory
aws dynamodb delete-table --table-name saa-inventory-restored
aws dynamodb delete-backup --backup-arn <DDB_BACKUP_ARN>
```

**4. IAM:**
```
aws iam detach-role-policy --role-name saa-backup-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup
aws iam detach-role-policy --role-name saa-backup-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores
aws iam delete-role --role-name saa-backup-role
```

**5. Local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-session-6`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-session-6`

**✅ Checkpoint:** **AWS Backup → Backup vaults** is empty in **both** us-east-1 and us-west-2 (switch Region in the console!). **EC2 → Volumes** and **Snapshots** show no `saa-*` items.

---

**🏁 Lab complete: +100 XP.** Session 6 done! **[📝 Mini Exam 6 →](mini-exam-06.md)**
