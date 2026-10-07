# Lab 3A: AWS KMS & Envelope Encryption

**Session:** 3 — Data Protection  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%)  
**Difficulty:** Beginner  
**Estimated Time:** 35–45 minutes

---

## Overview

"Encrypt data at rest" shows up in a huge share of SAA questions. The answer is almost always **AWS KMS**, but the *right* KMS answer depends on details: who manages the key, whether you need an audit trail, whether rotation matters, whether data is larger than 4 KB.

In this lab you'll create your own KMS key, encrypt data directly, hit the 4 KB limit on purpose, learn **envelope encryption** (how AWS really encrypts gigabytes), turn on S3 encryption with your key, and then lock yourself out by disabling the key.

**What you will build:**
- A customer managed KMS key with an alias and automatic rotation
- An S3 bucket with default **SSE-KMS** encryption and an **S3 Bucket Key**
- A lock-out demo: disable the key → the data becomes unreadable

---

## Prerequisites

- ✅ Session 1 complete (key policies are resource-based policies)
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS KMS customer managed key | Your encryption key | $1/month, **prorated hourly** (~$0.03 for this lab) |
| KMS API requests | Encrypt/decrypt calls | 20,000 free/month |
| Amazon S3 | A few test objects | ~$0.00 |

**Estimated cost for this lab: ~$0.03.** The key is scheduled for deletion in Cleanup; keys pending deletion aren't billed.

---

## Concepts

**Three kinds of KMS keys:**

| Key Type | Who Manages It | You Control the Key Policy? | Rotation | Cost |
|----------|---------------|------------------------------|----------|------|
| **AWS owned** | AWS, shared across accounts | ❌ | AWS decides | Free, invisible |
| **AWS managed** (`aws/s3`, `aws/ebs`…) | AWS, in your account | ❌ (view only) | Automatic, yearly | Free (pay per request) |
| **Customer managed** | **You** | ✅ | Optional, configurable | $1/month + requests |

**Envelope encryption.** KMS can only encrypt **up to 4 KB** directly. For anything bigger:
1. Ask KMS for a **data key**. You get it twice: in **plaintext** and **encrypted under your KMS key**.
2. Encrypt your data locally with the plaintext data key, then **throw the plaintext key away**.
3. Store the **encrypted data key** next to the encrypted data.
4. To decrypt: send the encrypted data key to KMS → get the plaintext key back → decrypt locally.

```
     KMS key (never leaves KMS)
          │ encrypts
          ▼
   🔑 data key ──encrypts──▶ 📦 your 5 GB file
   (stored encrypted next to the file)
```

This is exactly what S3, EBS and RDS do behind the scenes.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With |
|-------------|------------------------|
| `<YOUR_PROFILE_NAME>` | Your CLI profile |
| `<ACCOUNT_ID>` | Your account ID |
| `<KEY_ID>` | Your KMS key ID (Step 3) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-lab-3a; cd ~\Desktop\saa-lab-3a; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-lab-3a && cd ~/Desktop/saa-lab-3a && code .
```

---

### Step 2: Look at the AWS Managed Keys You Already Have

```
aws kms list-aliases --query "Aliases[?starts_with(AliasName,'alias/aws/')].AliasName" --output table
```

**✅ You'll likely see** `alias/aws/s3`, `alias/aws/ebs`, `alias/aws/ssm` and others. AWS created these the first time a service needed them.

---

### Step 3: Create a Customer Managed Key

```
aws kms create-key --description "SAA Lab 3A key" --query KeyMetadata.KeyId --output text
```

> **📝 Save as `<KEY_ID>`**

📋 Give it a friendly alias:
```
aws kms create-alias --alias-name alias/saa-lab3a --target-key-id <KEY_ID>
```

📋 Look at the **default key policy** AWS attached:
```
aws kms get-key-policy --key-id <KEY_ID> --policy-name default --output text
```

**✅ You should see** a statement with `"Principal": {"AWS": "arn:aws:iam::<ACCOUNT_ID>:root"}` and `"Action": "kms:*"`.

> 💡 **Why this matters:** Unlike most AWS resources, **a KMS key is only usable if its key policy allows it**. This default statement delegates control to IAM, so IAM policies in this account *can* grant access. Remove it, and even an admin can be locked out of the key forever.

---

### Step 4: Encrypt and Decrypt a Small Secret

**Create `secret.txt`** in VS Code containing: `The vault code is 4-8-15-16-23-42`

📋 Encrypt it:

**macOS / Linux:**
```bash
aws kms encrypt --key-id alias/saa-lab3a --plaintext fileb://secret.txt --query CiphertextBlob --output text | base64 --decode > secret.enc
```

**Windows (PowerShell):**
```powershell
$b64 = aws kms encrypt --key-id alias/saa-lab3a --plaintext fileb://secret.txt --query CiphertextBlob --output text
[IO.File]::WriteAllBytes("$PWD\secret.enc", [Convert]::FromBase64String($b64))
```

Open `secret.enc` in VS Code. It's unreadable binary. ✅

🔮 **Predict:** To decrypt, do you need to tell KMS which key to use?

📋 Decrypt (notice: **no `--key-id`**):

**macOS / Linux:**
```bash
aws kms decrypt --ciphertext-blob fileb://secret.enc --query Plaintext --output text | base64 --decode
```

**Windows (PowerShell):**
```powershell
$p = aws kms decrypt --ciphertext-blob fileb://secret.enc --query Plaintext --output text
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($p))
```

<details>
<summary>🔮 Reveal</summary>

**No key ID needed** for symmetric keys. The ciphertext includes metadata identifying which KMS key encrypted it. You'll see your vault code printed. +10 XP if you predicted it.

</details>

---

### Step 5: Hit the 4 KB Wall

📋 Create a 5,000-byte file:

**macOS / Linux:**
```bash
head -c 5000 /dev/urandom > big.bin
```

**Windows (PowerShell):**
```powershell
[IO.File]::WriteAllBytes("$PWD\big.bin", (New-Object byte[] 5000))
```

🔮 **Predict**, then run:
```
aws kms encrypt --key-id alias/saa-lab3a --plaintext fileb://big.bin
```

<details>
<summary>🔮 Reveal</summary>

❌ **ValidationException**: the plaintext must be **at most 4096 bytes**. This is *why* envelope encryption exists.

</details>

---

### Step 6: Generate a Data Key (Envelope Encryption)

```
aws kms generate-data-key --key-id alias/saa-lab3a --key-spec AES_256
```

**✅ You should see three fields:**

| Field | What It Is | What You Do With It |
|-------|-----------|---------------------|
| `Plaintext` | The 256-bit data key, in the clear | Encrypt your big file locally, then **erase it from memory** |
| `CiphertextBlob` | The same data key, encrypted under your KMS key | **Store it** alongside the encrypted file |
| `KeyId` | Which KMS key wrapped it | Informational |

### 🧩 Checkpoint

Why does envelope encryption scale to petabytes, when KMS itself only handles 4 KB per call?

<details>
<summary>Answer (+10 XP)</summary>

The bulk data **never travels to KMS**. Only the tiny data key (32 bytes) does. Encryption of the actual data happens locally (or inside S3/EBS) at full speed. KMS just protects the keys, which also keeps request costs and latency low.

</details>

---

### Step 7: Turn On Key Rotation

```
aws kms enable-key-rotation --key-id <KEY_ID>
```
```
aws kms get-key-rotation-status --key-id <KEY_ID>
```

**✅ You should see** `"KeyRotationEnabled": true` and a rotation period (365 days by default).

> 💡 Rotation creates new **backing key material** but keeps the **same key ID and ARN**. Old ciphertext still decrypts because KMS keeps every previous version. Nothing to re-encrypt, nothing to update in your apps.

---

### Step 8: Default SSE-KMS Encryption on S3 with a Bucket Key

```
aws s3 mb s3://saa-lab3a-<ACCOUNT_ID> --region us-east-1
```

**Create `bucket-encryption.json`**, **replacing `<ACCOUNT_ID>` and `<KEY_ID>`**:
```json
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:<ACCOUNT_ID>:key/<KEY_ID>"
      },
      "BucketKeyEnabled": true
    }
  ]
}
```

```
aws s3api put-bucket-encryption --bucket saa-lab3a-<ACCOUNT_ID> --server-side-encryption-configuration file://bucket-encryption.json
```

📋 Upload and inspect:
```
aws s3 cp secret.txt s3://saa-lab3a-<ACCOUNT_ID>/secret.txt
```
```
aws s3api head-object --bucket saa-lab3a-<ACCOUNT_ID> --key secret.txt --query "[ServerSideEncryption,SSEKMSKeyId,BucketKeyEnabled]"
```

**✅ You should see** `"aws:kms"`, your key ARN and `true`.

> 💡 **S3 Bucket Key** = S3 gets a short-lived bucket-level key from KMS and derives per-object keys from it. That cuts KMS requests (and cost) **by up to 99%** for busy buckets. 🚨 The exam answer to "SSE-KMS is causing KMS throttling or high KMS costs" is **enable S3 Bucket Keys**.

---

### Step 9: The Lock-Out Demo

🔮 **Predict:** You're an **administrator**. You disable the KMS key. Can you still download `secret.txt`?

```
aws kms disable-key --key-id <KEY_ID>
```
```
aws s3 cp s3://saa-lab3a-<ACCOUNT_ID>/secret.txt -
```

<details>
<summary>🔮 Reveal</summary>

❌ **Fails with a KMS error** (`KMS.DisabledException` / AccessDenied). S3 permissions aren't enough: reading an SSE-KMS object also requires `kms:Decrypt` on a **usable** key. This is **crypto-shredding**: destroy or disable the key and the data is gone, even if the bytes still exist.

It's also why SSE-KMS gives you **two layers of access control** (S3 policy + key policy) and an **audit trail** (every decrypt is a CloudTrail event), while SSE-S3 gives you neither.

</details>

📋 Re-enable it:
```
aws kms enable-key --key-id <KEY_ID>
```
```
aws s3 cp s3://saa-lab3a-<ACCOUNT_ID>/secret.txt -
```

**✅ Works again.**

---

### Step 10: Console Checkpoint

**✅ Checkpoint:**
1. **KMS → Customer managed keys → saa-lab3a**: see **Key policy**, **Key rotation** and **Aliases**.
2. **S3 → saa-lab3a-… → Properties → Default encryption** shows SSE-KMS, your key and **Bucket Key: Enabled**.
3. *(Bonus)* **CloudTrail → Event history**, filter **Event source = kms.amazonaws.com**: your `Decrypt`, `GenerateDataKey` and `DisableKey` calls are all there.

---

## What You Just Did

1. Compared AWS owned, AWS managed and customer managed keys
2. Encrypted directly with KMS and hit the **4 KB limit**
3. Generated a data key and understood **envelope encryption**
4. Enabled **automatic rotation** (same key ID, new material)
5. Applied **SSE-KMS + Bucket Key** to S3
6. Locked yourself out by disabling the key: crypto-shredding

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Encrypt at rest, **least operational overhead**" | **SSE-S3** (default on all new objects) |
| "Encrypt at rest **with an audit trail of key usage** / control who can decrypt" | **SSE-KMS** with a customer managed key |
| "Company must **manage its own keys outside AWS** / keys never stored in AWS" | **SSE-C** (customer-provided keys) or client-side encryption |
| "Dedicated, single-tenant HSM, FIPS 140-3 Level 3, full control" | **AWS CloudHSM** |
| "Encrypted data must be usable in **another Region**" | KMS **multi-Region keys** |
| "SSE-KMS throttling / high KMS cost on S3" | **S3 Bucket Keys** |
| "Encrypt an existing **unencrypted** EBS volume or RDS DB" | Snapshot → **copy snapshot with encryption** → restore |

**🚨 Exam traps**
- You **can't** encrypt an existing unencrypted RDS instance in place. Snapshot, copy encrypted, restore.
- KMS keys are **regional**. A snapshot encrypted in us-east-1 needs a key in the target Region when you copy it across Regions.
- Deleting a KMS key has a **7–30 day** waiting period. It can't be immediate, by design.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InvalidCiphertextException` on decrypt | The file got corrupted (often a base64/encoding issue) | Re-run Step 4 exactly; on Windows use the PowerShell version |
| `AlreadyExistsException` on alias | Alias left over from a previous run | `aws kms delete-alias --alias-name alias/saa-lab3a` and retry |
| `put-bucket-encryption` error | Typo in the key ARN | Check `aws kms describe-key --key-id alias/saa-lab3a --query KeyMetadata.Arn` |

---

## 🧹 Cleanup

```
aws s3 rm s3://saa-lab3a-<ACCOUNT_ID> --recursive
```
```
aws s3 rb s3://saa-lab3a-<ACCOUNT_ID>
```
```
aws kms delete-alias --alias-name alias/saa-lab3a
```
```
aws kms schedule-key-deletion --key-id <KEY_ID> --pending-window-in-days 7
```

**✅ You should see** a `DeletionDate` 7 days from now. 🔮 Why can't you delete it immediately? (Answer: protection against accidental crypto-shredding. You can still `cancel-key-deletion` during the window.)

**Delete the local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-lab-3a`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-lab-3a`

---

**🏁 Lab complete: +100 XP.** Next: [Lab 3B — Locking Down S3](lab-3b-s3-lockdown.md)
