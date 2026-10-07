# Lab 7A: S3 Storage Classes & Lifecycle Policies — Paying Only for What Data Is Worth

**Session:** 7 — Storage & DNS Performance  
**Exam Domain:** Domain 4 — Cost-Optimized (20%) · Domain 3 — High-Performing (24%)  
**Difficulty:** Beginner  
**Estimated Time:** 35–45 minutes

---

## Overview

The same 1 TB costs **~$23/month** in S3 Standard and **~$1/month** in Glacier Deep Archive. The SAA exam rewards you for knowing which class fits which **access pattern** and penalizes you for missing the catch: minimum storage durations, retrieval fees and retrieval times.

In this lab you'll put objects in six storage classes, **try to read a Glacier object and get refused**, start a restore, and write a lifecycle policy that moves data down the cost ladder automatically. You'll also hit an invalid rule on purpose.

---

## Prerequisites

- ✅ Session 3 complete (versioning)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| S3 (all classes) | A handful of 1 KB objects | Fractions of a cent, even with minimum durations |
| Glacier restore (Bulk) | One tiny object | ~$0.00 |

**Estimated cost for this lab: $0.00**

> 💡 Archive classes bill a **minimum duration** (e.g. 180 days for Deep Archive) and a **40 KB minimum billable size per object**, even if you delete early. For a few tiny lab objects that's a fraction of a cent. For millions of small files it's a real exam trap.

---

## Concepts

**The S3 storage class ladder:**

| Class | Access Pattern | Min Duration | Retrieval | AZs | ~$/GB-mo* |
|-------|---------------|--------------|-----------|-----|-----------|
| **Standard** | Frequent | — | Instant, free | ≥3 | 0.023 |
| **Intelligent-Tiering** | **Unknown / changing** | — | Instant (archive tiers optional) | ≥3 | 0.023 → 0.0125 → 0.004, auto |
| **Standard-IA** | Infrequent, needs instant access | 30 days | Instant, **per-GB fee** | ≥3 | 0.0125 |
| **One Zone-IA** | Infrequent, **re-creatable** data | 30 days | Instant, per-GB fee | **1** | 0.01 |
| **Glacier Instant Retrieval** | Quarterly access, needs ms retrieval | 90 days | **Milliseconds**, per-GB fee | ≥3 | 0.004 |
| **Glacier Flexible Retrieval** | 1–2×/year | 90 days | **Minutes–12 h** (Expedited 1–5 min, Standard 3–5 h, Bulk 5–12 h) | ≥3 | 0.0036 |
| **Glacier Deep Archive** | Compliance archives, rarely accessed | 180 days | **12–48 h** | ≥3 | 0.00099 |

<sub>*us-east-1, approximate. Check the S3 pricing page for current numbers.</sub>

**Intelligent-Tiering** moves objects between tiers based on access, for a small monitoring fee per object. There are **no retrieval fees**. It's the right answer when the access pattern is **unknown or unpredictable**.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<ACCOUNT_ID>` | Your account ID |

---

## Lab Steps

### Step 1: Profile, Folder, Bucket

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-session-7; cd ~\Desktop\saa-session-7; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-session-7 && cd ~/Desktop/saa-session-7 && code .
```

```
aws s3 mb s3://saa-lab7a-<ACCOUNT_ID> --region us-east-1
```

Create a file `doc.txt` containing `Important business document` and save it.

---

### Step 2: Upload the Same File to Six Classes

📋 One per class:
```
aws s3 cp doc.txt s3://saa-lab7a-<ACCOUNT_ID>/standard/doc.txt
```
```
aws s3 cp doc.txt s3://saa-lab7a-<ACCOUNT_ID>/it/doc.txt --storage-class INTELLIGENT_TIERING
```
```
aws s3 cp doc.txt s3://saa-lab7a-<ACCOUNT_ID>/ia/doc.txt --storage-class STANDARD_IA
```
```
aws s3 cp doc.txt s3://saa-lab7a-<ACCOUNT_ID>/onezone/doc.txt --storage-class ONEZONE_IA
```
```
aws s3 cp doc.txt s3://saa-lab7a-<ACCOUNT_ID>/glacier-ir/doc.txt --storage-class GLACIER_IR
```
```
aws s3 cp doc.txt s3://saa-lab7a-<ACCOUNT_ID>/glacier/doc.txt --storage-class GLACIER
```

📋 See them all:
```
aws s3api list-objects-v2 --bucket saa-lab7a-<ACCOUNT_ID> --query "Contents[].[Key,StorageClass,Size]" --output table
```

---

### Step 3: Try to Read Each One

🔮 **Predict** which downloads succeed **instantly**:

| Object | Prediction |
|--------|-----------|
| `ia/doc.txt` | |
| `glacier-ir/doc.txt` | |
| `glacier/doc.txt` | |

```
aws s3 cp s3://saa-lab7a-<ACCOUNT_ID>/ia/doc.txt -
```
```
aws s3 cp s3://saa-lab7a-<ACCOUNT_ID>/glacier-ir/doc.txt -
```
```
aws s3 cp s3://saa-lab7a-<ACCOUNT_ID>/glacier/doc.txt -
```

<details>
<summary>🔮 Reveal</summary>

- **IA** ✅ instant (with a small per-GB retrieval fee).
- **Glacier Instant Retrieval** ✅ instant. That's the whole point of the class: archive pricing, millisecond access.
- **Glacier Flexible Retrieval** ❌ `InvalidObjectState`: the object is archived. You must **restore** it first, which creates a temporary readable copy.

</details>

---

### Step 4: Restore from Glacier Flexible Retrieval

**Create `restore.json`:**
```json
{
  "Days": 1,
  "GlacierJobParameters": { "Tier": "Bulk" }
}
```
```
aws s3api restore-object --bucket saa-lab7a-<ACCOUNT_ID> --key glacier/doc.txt --restore-request file://restore.json
```
```
aws s3api head-object --bucket saa-lab7a-<ACCOUNT_ID> --key glacier/doc.txt --query "[StorageClass,Restore]"
```

**✅ You should see** `ongoing-request="true"`. A **Bulk** restore takes **5–12 hours**, so don't wait. Check back tomorrow if you're curious: it'll show `ongoing-request="false", expiry-date=…`, and the download will work for 1 day.

### 🧩 Checkpoint

Auditors give you 24 hours' notice before they need documents. Documents are retrieved about twice a year. Which class is cheapest while meeting that requirement?

<details>
<summary>Answer (+10 XP)</summary>

**Glacier Deep Archive.** Standard retrieval is within 12 hours, so it meets a 24-hour notice, and it's the lowest-cost class. If the notice were "within minutes," it would be Glacier Flexible (Expedited) or Glacier Instant Retrieval.

</details>

---

### Step 5: Write a Lifecycle Policy — and Break It First

🔮 **Predict:** Will S3 accept a rule that moves objects to **Standard-IA after 10 days**?

**Create `lifecycle-bad.json`:**
```json
{
  "Rules": [
    {
      "ID": "too-eager",
      "Filter": { "Prefix": "logs/" },
      "Status": "Enabled",
      "Transitions": [ { "Days": 10, "StorageClass": "STANDARD_IA" } ]
    }
  ]
}
```
```
aws s3api put-bucket-lifecycle-configuration --bucket saa-lab7a-<ACCOUNT_ID> --lifecycle-configuration file://lifecycle-bad.json
```

<details>
<summary>🔮 Reveal</summary>

❌ **`InvalidArgument`**: objects must be stored **at least 30 days** before a lifecycle transition to Standard-IA or One Zone-IA. 🚨 Exam distractors use exactly this ("transition to IA after 7 days").

</details>

**Now the real one.** Create `lifecycle.json`:
```json
{
  "Rules": [
    {
      "ID": "logs-cost-ladder",
      "Filter": { "Prefix": "logs/" },
      "Status": "Enabled",
      "Transitions": [
        { "Days": 30,  "StorageClass": "STANDARD_IA" },
        { "Days": 90,  "StorageClass": "GLACIER_IR" },
        { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
      ],
      "Expiration": { "Days": 2555 }
    },
    {
      "ID": "versioning-hygiene",
      "Filter": {},
      "Status": "Enabled",
      "NoncurrentVersionExpiration": { "NoncurrentDays": 30 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }
  ]
}
```
```
aws s3api put-bucket-lifecycle-configuration --bucket saa-lab7a-<ACCOUNT_ID> --lifecycle-configuration file://lifecycle.json
```
```
aws s3api get-bucket-lifecycle-configuration --bucket saa-lab7a-<ACCOUNT_ID> --query "Rules[].ID"
```

**✅ You should see** both rule IDs.

> 💡 **What each rule does:**
> - **logs-cost-ladder:** logs step down Standard → IA (30 d) → Glacier IR (90 d) → Deep Archive (1 yr) → deleted after 7 years (a common retention requirement).
> - **versioning-hygiene:** old versions pile up silently and cost money, so they expire 30 days after being replaced. Incomplete multipart uploads are invisible in normal listings but **billed**, so they're aborted after 7 days.

---

### Step 6: 🎮 Storage Class Match-Up (+10 XP each)

Write your answer, then reveal.

| # | Data | Your Class |
|---|------|-----------|
| 1 | Website images, accessed constantly | |
| 2 | Data-lake files; some hot, some untouched for months; pattern unknown | |
| 3 | Thumbnails that can be regenerated from originals, accessed monthly | |
| 4 | Medical images: rarely viewed, but doctors need them **in milliseconds** | |
| 5 | Financial records kept 10 years for regulators, retrieval within 48 h OK | |
| 6 | Disaster-recovery copy of backups, restored maybe once a year, within hours | |

<details>
<summary>🎮 Reveal</summary>

1. **S3 Standard**
2. **S3 Intelligent-Tiering** (unknown/changing pattern)
3. **S3 One Zone-IA** (infrequent and re-creatable, so losing an AZ is acceptable)
4. **S3 Glacier Instant Retrieval** (archive price, millisecond access)
5. **S3 Glacier Deep Archive**
6. **S3 Glacier Flexible Retrieval**

</details>

---

### Step 7: Console Checkpoint

**✅ Checkpoint:**
1. **S3 → saa-lab7a-… → Objects**: the **Storage class** column shows all six classes.
2. **Management → Lifecycle rules**: both rules. Click one to see the **timeline visualization**.
3. *(Bonus)* **S3 → Storage Lens → default dashboard**: account-wide storage by class (data refreshes daily).

---

## What You Just Did

1. Stored data in **six storage classes** and compared retrieval behavior
2. Hit `InvalidObjectState` on Glacier Flexible and started a **restore**
3. Discovered the **30-day minimum** before IA transitions
4. Wrote a production-grade **lifecycle policy**, including noncurrent-version and multipart cleanup

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Unknown or changing access patterns" | **Intelligent-Tiering** |
| "Infrequent access, data can be re-created" | **One Zone-IA** |
| "Archive, but retrieve in milliseconds" | **Glacier Instant Retrieval** |
| "Lowest cost archive, 12–48 h retrieval OK" | **Glacier Deep Archive** |
| "Automatically move objects to cheaper storage over time" | **Lifecycle rules** |
| "Speed up uploads from distant users" | **S3 Transfer Acceleration** / **multipart upload** |
| "Download part of a large object, parallel downloads" | **Byte-range fetches** |
| "Analyze access patterns to choose lifecycle timing" | **S3 Storage Class Analysis** / **Storage Lens** |

**🚨 Exam traps**
- **Minimum durations:** IA 30 days, Glacier IR/Flexible 90 days, Deep Archive 180 days. Delete early and you still pay for the minimum.
- **IA classes charge retrieval fees.** For frequently read data they can cost **more** than Standard.
- **Small objects** (< 128 KB) aren't auto-tiered by Intelligent-Tiering and are billed as 128 KB in IA classes.
- One Zone-IA data is **lost** if that AZ is destroyed.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `RestoreAlreadyInProgress` | You already started a restore | That's fine; check `head-object` |
| `MalformedXML` on lifecycle | JSON structure error | Every rule needs `Filter` (can be `{}`), `Status` and `ID` |

---

## 🧹 Cleanup

```
aws s3 rm s3://saa-lab7a-<ACCOUNT_ID> --recursive
```
```
aws s3 rb s3://saa-lab7a-<ACCOUNT_ID>
```

> Keep the `saa-session-7` folder for Labs 7B and 7C.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 7B — EBS vs EFS vs Instance Store](lab-7b-ebs-vs-efs.md)
