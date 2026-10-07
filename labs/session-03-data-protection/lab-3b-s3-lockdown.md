# Lab 3B: Locking Down S3 — TLS, Encryption Policies, Presigned URLs, Versioning & Object Lock

**Session:** 3 — Data Protection  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%)  
**Difficulty:** Intermediate  
**Estimated Time:** 45–55 minutes

---

## Overview

S3 is the most-tested service on the SAA exam, and a large share of those questions are about **protecting** S3 data from four threats:

| Threat | Control You'll Build |
|--------|---------------------|
| Data intercepted in transit | Bucket policy that **denies non-HTTPS** requests |
| Uploads that skip your encryption standard | Bucket policy that **denies uploads without SSE-KMS** |
| Sharing a private file with an outsider | **Presigned URL** that expires |
| Accidental deletes, overwrites and ransomware | **Versioning** + **Object Lock** (WORM) |

---

## Prerequisites

- ✅ **Lab 3A** complete (you understand SSE-KMS)
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon S3 | Two buckets, a few tiny objects | ~$0.00 |
| KMS (AWS managed `aws/s3` key) | Encrypts uploads | Free key, requests within free tier |

**Estimated cost for this lab: $0.00**

> ⚠️ Object Lock in this lab uses **GOVERNANCE** mode with a **1-day** retention, which an admin can bypass. **Never use COMPLIANCE mode for practice.** Nobody, including the root user, can delete those objects until retention expires.

---

## Concepts

| Feature | One-Liner |
|---------|-----------|
| **Block Public Access** | Account- or bucket-level override that blocks public ACLs and policies. **On by default** for new buckets. |
| **`aws:SecureTransport`** | `false` when a request arrives over plain HTTP |
| **Presigned URL** | A URL signed with *your* credentials that grants temporary access to one object. Valid up to 7 days with IAM user keys; shorter with role/STS credentials. |
| **Versioning** | Keeps every version of every object. A "delete" adds a **delete marker** instead of erasing data. |
| **Object Lock** | **WORM** (write once, read many). Must be enabled at bucket creation (or later via support/API). **Governance** mode can be bypassed with a special permission; **Compliance** mode can't, by anyone. |
| **Legal hold** | An Object Lock flag with no expiry date, which stays until removed |
| **MFA Delete** | Requires MFA to permanently delete versions or suspend versioning. Root user only, CLI only. |

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With |
|-------------|------------------------|
| `<YOUR_PROFILE_NAME>` | Your CLI profile |
| `<ACCOUNT_ID>` | Your account ID |
| `<VERSION_ID_…>` | Version IDs you'll copy from command output |

---

## Lab Steps

### Step 1: Profile, Folder and Bucket

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-lab-3b; cd ~\Desktop\saa-lab-3b; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-lab-3b && cd ~/Desktop/saa-lab-3b && code .
```

📋 Create the bucket and check Block Public Access:
```
aws s3 mb s3://saa-lab3b-<ACCOUNT_ID> --region us-east-1
```
```
aws s3api get-public-access-block --bucket saa-lab3b-<ACCOUNT_ID>
```

**✅ You should see** all four settings `true`. That's the default since April 2023.

Create a file `report.txt` containing `Q3 revenue: confidential` and save it.

---

## Part 1 — Enforce TLS and Encryption with a Bucket Policy

### Step 2: Write the Policy

**Create `secure-bucket-policy.json`**, **replacing `<ACCOUNT_ID>`** (4 places):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::saa-lab3b-<ACCOUNT_ID>",
        "arn:aws:s3:::saa-lab3b-<ACCOUNT_ID>/*"
      ],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    },
    {
      "Sid": "DenyUploadsWithoutKMS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::saa-lab3b-<ACCOUNT_ID>/*",
      "Condition": {
        "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" }
      }
    }
  ]
}
```

```
aws s3api put-bucket-policy --bucket saa-lab3b-<ACCOUNT_ID> --policy file://secure-bucket-policy.json
```

---

### Step 3: Test the Policy

🔮 **Predict each** before running:

| # | Command | Prediction |
|---|---------|-----------|
| 1 | `aws s3 cp report.txt s3://saa-lab3b-<ACCOUNT_ID>/report.txt` | |
| 2 | `aws s3 cp report.txt s3://saa-lab3b-<ACCOUNT_ID>/report.txt --sse aws:kms` | |
| 3 | `aws s3 ls s3://saa-lab3b-<ACCOUNT_ID>/ --endpoint-url http://s3.us-east-1.amazonaws.com` | |
| 4 | `aws s3 ls s3://saa-lab3b-<ACCOUNT_ID>/` | |

<details>
<summary>🔮 Reveal</summary>

1. ❌ **AccessDenied.** No `x-amz-server-side-encryption: aws:kms` header in the request, so the second Deny matches. (Default bucket encryption would have encrypted it anyway, but the policy checks the *request header*.)
2. ✅ **Uploads.** `--sse aws:kms` sends the header.
3. ❌ **AccessDenied.** Plain `http://`, so `aws:SecureTransport = false`. Even you, the admin, are denied: explicit deny beats your admin allow (Lab 1A).
4. ✅ **Lists.** HTTPS is the CLI default.

</details>

> 💡 **Modern note:** Since 2023, S3 encrypts *every* new object with SSE-S3 by default, so the "deny unencrypted uploads" policy is now mostly about **forcing a specific encryption type** (SSE-KMS). The exam still uses this pattern, so recognize it.

---

## Part 2 — Presigned URLs

### Step 4: Share a Private Object for 2 Minutes

🔮 **Predict:** open this object's normal URL in a browser: `https://saa-lab3b-<ACCOUNT_ID>.s3.amazonaws.com/report.txt`. What do you see?

<details>
<summary>🔮 Reveal</summary>

An XML **AccessDenied** error. The bucket is private and Block Public Access is on.

</details>

📋 Generate a presigned URL valid for **120 seconds**:
```
aws s3 presign s3://saa-lab3b-<ACCOUNT_ID>/report.txt --expires-in 120
```

Copy the long URL into a browser (or a private window). **✅ You should see** `Q3 revenue: confidential`.

Wait 2+ minutes and refresh. **✅ You should see** `Request has expired`.

### 🧩 Checkpoint

Who is the request "from" when someone uses your presigned URL, and what happens if the IAM role that signed it loses access to the object?

<details>
<summary>Answer (+10 XP)</summary>

The request is made **with your permissions**. The URL embeds your signature. If the signer loses permission (or the temporary credentials used to sign expire), the URL stops working even before its own expiry. That's why URLs signed with short-lived role credentials can't outlive the session.

</details>

> 💡 **Exam pattern:** "Let users upload directly to S3 from the browser without making the bucket public or proxying through servers" → **presigned URLs** (you can presign `PUT` too, through the SDK).

---

## Part 3 — Versioning: Undo for S3

### Step 5: Enable Versioning and Make Mistakes

```
aws s3api put-bucket-versioning --bucket saa-lab3b-<ACCOUNT_ID> --versioning-configuration Status=Enabled
```

Change `report.txt` to `Q3 revenue: OVERWRITTEN BY MISTAKE` and save. 📋 Upload it again (with KMS, because your policy requires it):
```
aws s3 cp report.txt s3://saa-lab3b-<ACCOUNT_ID>/report.txt --sse aws:kms
```

📋 Now "delete" it:
```
aws s3 rm s3://saa-lab3b-<ACCOUNT_ID>/report.txt
```

🔮 **Predict:** Is the data gone?

📋 Look:
```
aws s3api list-object-versions --bucket saa-lab3b-<ACCOUNT_ID> --prefix report.txt --query "{Versions:Versions[].[VersionId,IsLatest,LastModified],DeleteMarkers:DeleteMarkers[].[VersionId,IsLatest]}"
```

<details>
<summary>🔮 Reveal</summary>

**Nothing is gone.** You'll see:
- **Two versions.** The original upload from Part 1 was made *before* versioning, so its version ID is `null`. The overwrite has a real version ID.
- **One delete marker** with `IsLatest: true`. It just *hides* the object.

</details>

---

### Step 6: Recover

📋 Delete the **delete marker** (copy its version ID from the output above):
```
aws s3api delete-object --bucket saa-lab3b-<ACCOUNT_ID> --key report.txt --version-id <VERSION_ID_DELETE_MARKER>
```

📋 The object is back, but it's the *overwritten* version. Fetch the original:
```
aws s3api get-object --bucket saa-lab3b-<ACCOUNT_ID> --key report.txt --version-id null original.txt
```

Open `original.txt`. **✅ You should see** `Q3 revenue: confidential`. Recovered! 🎉

---

## Part 4 — Object Lock (Ransomware Protection)

### Step 7: Create a Locked Bucket

Object Lock is enabled at creation (it automatically turns on versioning):
```
aws s3api create-bucket --bucket saa-lab3b-vault-<ACCOUNT_ID> --object-lock-enabled-for-bucket
```

**Create `lock-config.json`**: a default 1-day GOVERNANCE retention.
```json
{
  "ObjectLockEnabled": "Enabled",
  "Rule": {
    "DefaultRetention": { "Mode": "GOVERNANCE", "Days": 1 }
  }
}
```

```
aws s3api put-object-lock-configuration --bucket saa-lab3b-vault-<ACCOUNT_ID> --object-lock-configuration file://lock-config.json
```

📋 Upload a "backup":
```
aws s3 cp original.txt s3://saa-lab3b-vault-<ACCOUNT_ID>/backup.txt
```

📋 Get its version ID and retention:
```
aws s3api head-object --bucket saa-lab3b-vault-<ACCOUNT_ID> --key backup.txt --query "[VersionId,ObjectLockMode,ObjectLockRetainUntilDate]"
```

> **📝 Save the version ID as `<VERSION_ID_BACKUP>`**

---

### Step 8: Play the Ransomware Attacker

🔮 **Predict each:**

| # | Attack | Prediction |
|---|--------|-----------|
| 1 | `aws s3 rm s3://saa-lab3b-vault-<ACCOUNT_ID>/backup.txt` | |
| 2 | `aws s3api delete-object --bucket saa-lab3b-vault-<ACCOUNT_ID> --key backup.txt --version-id <VERSION_ID_BACKUP>` | |

<details>
<summary>🔮 Reveal</summary>

1. ✅ **"Succeeds"**, but it only adds a **delete marker**. The locked version is untouched. An attacker thinks they won; you remove the marker and the data's back.
2. ❌ **AccessDenied.** Permanently deleting a locked version is blocked until the retain-until date. That's WORM in action.

</details>

### 🧩 Checkpoint

The attacker has stolen admin credentials, which include `s3:BypassGovernanceRetention`. Is GOVERNANCE mode enough to protect your backups? What should you use?

<details>
<summary>Answer (+10 XP)</summary>

**No.** Governance mode can be bypassed by anyone with `s3:BypassGovernanceRetention`. For protection even against compromised admins and the root user (or for regulations like SEC 17a-4), use **COMPLIANCE mode**. Also consider **AWS Backup Vault Lock** and replicating backups to a **separate account**.

</details>

---

### Step 9: Console Checkpoint

**✅ Checkpoint:**
1. **S3 → saa-lab3b-… → Permissions → Bucket policy** shows both Deny statements.
2. **S3 → saa-lab3b-… → Objects**, toggle **Show versions**: every version is visible.
3. **S3 → saa-lab3b-vault-… → Properties → Object Lock** shows Enabled, Governance, 1 day.

---

## What You Just Did

1. Forced **HTTPS-only** access and **SSE-KMS uploads** with a bucket policy
2. Shared a private object temporarily with a **presigned URL**
3. Recovered from an overwrite **and** a delete with **versioning**
4. Defeated a simulated ransomware delete with **Object Lock**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Enforce encryption in transit to S3" | Bucket policy: Deny when `aws:SecureTransport = false` |
| "Temporary access to a private object for an external user" | **Presigned URL** |
| "Protect against accidental deletion" | **Versioning** (+ **MFA Delete** for extra protection) |
| "Data must be immutable / WORM / regulatory retention" | **Object Lock, Compliance mode** |
| "Prevent any bucket in the account from ever going public" | **Account-level Block Public Access** (+ SCP to stop it being changed) |
| "Serve private S3 content through CloudFront only" | **Origin Access Control (OAC)** |
| "Discover sensitive data (PII) in S3" | **Amazon Macie** |

**🚨 Exam traps**
- Versioning **can't be disabled** once enabled, only **suspended**.
- **Compliance** mode: nobody can shorten retention or delete, not even root. **Governance**: users with the bypass permission can.
- Presigned URLs work **with** Block Public Access on. They're authenticated requests, not public access.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Locked yourself out of the bucket policy (can't even read/delete it) | A Deny that's too broad | Use the **root user** to delete the bucket policy, or fix it from the console as root |
| Presigned URL says `SignatureDoesNotMatch` | URL got truncated when copying | Copy the *entire* URL, including everything after `?` |
| `InvalidRequest` on upload to the vault bucket | Your CLI is too old to send the required checksum | Update AWS CLI v2 |

---

## 🧹 Cleanup

**Bucket 1** (versioned, so every version must be deleted). 📋 List the versions and delete markers:
```
aws s3api list-object-versions --bucket saa-lab3b-<ACCOUNT_ID> --query "[Versions[].[Key,VersionId], DeleteMarkers[].[Key,VersionId]]" --output text
```
For **each** version ID printed (including `null`):
```
aws s3api delete-object --bucket saa-lab3b-<ACCOUNT_ID> --key report.txt --version-id <VERSION_ID>
```
```
aws s3api delete-bucket-policy --bucket saa-lab3b-<ACCOUNT_ID>
```
```
aws s3 rb s3://saa-lab3b-<ACCOUNT_ID>
```

> 💡 Shortcut: **S3 console → select bucket → Empty** deletes all versions for you. Then **Delete**.

**Bucket 2** (the vault). As admin you can **bypass governance**. 📋 List versions:
```
aws s3api list-object-versions --bucket saa-lab3b-vault-<ACCOUNT_ID> --query "[Versions[].[Key,VersionId], DeleteMarkers[].[Key,VersionId]]" --output text
```
For **each** version ID printed:
```
aws s3api delete-object --bucket saa-lab3b-vault-<ACCOUNT_ID> --key backup.txt --version-id <VERSION_ID> --bypass-governance-retention
```
```
aws s3 rb s3://saa-lab3b-vault-<ACCOUNT_ID>
```

> If the console's **Empty** button fails on the vault, that's Object Lock working. Use the CLI with `--bypass-governance-retention`, or wait 1 day.

**Delete the local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-lab-3b`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-lab-3b`

---

**🏁 Lab complete: +100 XP.** Next: [Lab 3C — Secrets Manager vs Parameter Store](lab-3c-secrets-and-parameters.md)
