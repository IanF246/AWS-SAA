# Lab 3C: Secrets Manager vs. Parameter Store — Least-Privilege Secrets for Lambda

**Session:** 3 — Data Protection  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%)  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes

---

## Overview

"The application's database password is hard-coded in the source." Every SAA candidate sees this scenario. There are two AWS answers, **Secrets Manager** and **Systems Manager Parameter Store**, and the exam expects you to pick the right one based on a single keyword.

In this lab you'll store configuration and secrets in **both**, then build a Lambda function that reads them with a **least-privilege** role, rotate a secret version by hand, and prove the function **can't** read anything it wasn't granted.

**What you will build:**

```
 ┌────────────────────────┐   ssm:GetParameter (1 ARN)    ┌───────────────────────────┐
 │ Lambda: saa-config-    │ ────────────────────────────▶ │ Parameter Store           │
 │ reader                 │                               │  /saa/app/feature-flag    │
 │ (role: exactly 2 ARNs) │                               │  /saa/app/api-key 🔒      │
 │                        │   secretsmanager:GetSecret…   ├───────────────────────────┤
 │                        │ ────────────────────────────▶ │ Secrets Manager           │
 └────────────────────────┘                               │  saa/app/db-credentials 🔒│
                                                          └───────────────────────────┘
```

---

## Prerequisites

- ✅ **Labs 3A and 3B** complete
- ✅ Python is **not** required locally; the Lambda runtime runs the code

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| SSM Parameter Store (Standard) | Config + SecureString values | Always Free |
| AWS Secrets Manager | Managed secrets with rotation | $0.40/secret/month, **prorated** (~$0.01 for this lab) |
| AWS Lambda | Runs the reader function | Free tier |
| KMS (AWS managed keys) | Encrypts SecureStrings/secrets | Free |

**Estimated cost for this lab: ~$0.01**

---

## Concepts

| | **Parameter Store** | **Secrets Manager** |
|-|--------------------|---------------------|
| Built for | Config values + simple secrets | Secrets with a **lifecycle** |
| Encryption | Optional (`SecureString` via KMS) | Always (KMS) |
| **Automatic rotation** | ❌ Not built in | ✅ **Built in** (native for RDS, Aurora, Redshift, DocumentDB; Lambda for anything else) |
| Cross-account access | ❌ (Advanced tier can be shared via RAM) | ✅ Resource policies |
| Cross-Region replication | ❌ | ✅ |
| Cost | Standard tier **free** | $0.40/secret/month + API calls |
| Hierarchies / paths | ✅ `/app/prod/db/url` | Names can include `/` |

> 🧠 **The one-keyword rule:** if the question says **"rotate automatically"**, the answer is **Secrets Manager**. If it says **"cheapest"** or **"configuration values"** with no rotation requirement, the answer is **Parameter Store**.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With |
|-------------|------------------------|
| `<YOUR_PROFILE_NAME>` | Your CLI profile |
| `<ACCOUNT_ID>` | Your account ID |
| `<SECRET_ARN>` | Full secret ARN (Step 4), including the random 6-character suffix |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-lab-3c; cd ~\Desktop\saa-lab-3c; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-lab-3c && cd ~/Desktop/saa-lab-3c && code .
```

---

### Step 2: Store Values in Parameter Store

📋 A plain config value:
```
aws ssm put-parameter --name /saa/app/feature-flag --type String --value "dark-mode=on"
```

📋 An encrypted value (uses the AWS managed `aws/ssm` key):
```
aws ssm put-parameter --name /saa/app/api-key --type SecureString --value "sk-live-THIS-IS-A-FAKE-KEY-123"
```

📋 A value the Lambda function must **not** be able to read:
```
aws ssm put-parameter --name /saa/admin/root-password --type SecureString --value "never-give-this-out"
```

🔮 **Predict:** What does this return?
```
aws ssm get-parameter --name /saa/app/api-key --query Parameter.Value --output text
```

<details>
<summary>🔮 Reveal</summary>

A long **encrypted blob**, not the key. SecureStrings stay encrypted unless you add `--with-decryption` (which also requires KMS decrypt permission).

</details>

```
aws ssm get-parameter --name /saa/app/api-key --with-decryption --query Parameter.Value --output text
```

**✅ You should see** `sk-live-THIS-IS-A-FAKE-KEY-123`.

📋 Get a whole **hierarchy** at once:
```
aws ssm get-parameters-by-path --path /saa/app --with-decryption --query "Parameters[].[Name,Value]" --output table
```

---

### Step 3: Store a Database Credential in Secrets Manager

**Create `db-secret.json`:**
```json
{
  "username": "app_user",
  "password": "Initial-Pa55word!",
  "engine": "mysql",
  "host": "saa-db.example.internal",
  "port": 3306
}
```

📋 Create the secret:
```
aws secretsmanager create-secret --name saa/app/db-credentials --description "SAA Lab 3C fake DB creds" --secret-string file://db-secret.json --query ARN --output text
```

> **📝 Save as `<SECRET_ARN>`.** Notice the random `-AbCdEf` suffix AWS added to the end of the ARN.

---

### Step 4: Simulate a Rotation

Real rotation is done by a Lambda function on a schedule (Secrets Manager provides ready-made ones for RDS). Let's do what that function does, by hand.

Change the password in `db-secret.json` to `Rotated-Pa55word!` and save. 📋 Put the new version:
```
aws secretsmanager put-secret-value --secret-id saa/app/db-credentials --secret-string file://db-secret.json
```

📋 Inspect the versions:
```
aws secretsmanager list-secret-version-ids --secret-id saa/app/db-credentials --query "Versions[].[VersionId,VersionStages]" --output json
```

**✅ You should see** two versions: one labeled `AWSCURRENT` (new) and one `AWSPREVIOUS` (old).

> 💡 **Staging labels** are how rotation works without downtime. Apps always ask for `AWSCURRENT`. During rotation there's a brief `AWSPENDING` stage while the new password is set on the database and tested. Only then does the label move.

---

### Step 5: Create a Least-Privilege Role for Lambda

**Create `lambda-trust.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Create `lambda-secrets-policy.json`**, **replacing `<ACCOUNT_ID>` and `<SECRET_ARN>`**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyTheAppParameters",
      "Effect": "Allow",
      "Action": ["ssm:GetParameter", "ssm:GetParametersByPath"],
      "Resource": [
        "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/saa/app",
        "arn:aws:ssm:us-east-1:<ACCOUNT_ID>:parameter/saa/app/*"
      ]
    },
    {
      "Sid": "ReadOnlyTheDbSecret",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "<SECRET_ARN>"
    }
  ]
}
```

📋 Create the role:
```
aws iam create-role --role-name saa-lab3c-lambda-role --assume-role-policy-document file://lambda-trust.json
```
```
aws iam attach-role-policy --role-name saa-lab3c-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```
```
aws iam put-role-policy --role-name saa-lab3c-lambda-role --policy-name read-app-secrets --policy-document file://lambda-secrets-policy.json
```

> 💡 **Why no `kms:Decrypt`?** The secrets use **AWS managed keys** (`aws/ssm`, `aws/secretsmanager`), whose key policies let principals in your account use them **through that service**. With a **customer managed** key, you'd have to add `kms:Decrypt` for that key ARN too. That's a common exam "why does this fail?" answer.

---

### Step 6: Write and Deploy the Function

**Create `lambda_function.py`:**
```python
import json
import boto3

ssm = boto3.client("ssm")
secrets = boto3.client("secretsmanager")


def mask(value):
    """Show only the first 4 characters - never log full secrets!"""
    return value[:4] + "*" * max(len(value) - 4, 0)


def try_read(label, fn):
    try:
        return {label: fn()}
    except Exception as e:
        return {label: f"DENIED -> {type(e).__name__}"}


def lambda_handler(event, context):
    results = {}
    results.update(try_read("feature_flag", lambda: ssm.get_parameter(
        Name="/saa/app/feature-flag")["Parameter"]["Value"]))
    results.update(try_read("api_key", lambda: mask(ssm.get_parameter(
        Name="/saa/app/api-key", WithDecryption=True)["Parameter"]["Value"])))
    results.update(try_read("db_password", lambda: mask(json.loads(
        secrets.get_secret_value(SecretId="saa/app/db-credentials")["SecretString"])["password"])))
    results.update(try_read("admin_root_password", lambda: ssm.get_parameter(
        Name="/saa/admin/root-password", WithDecryption=True)["Parameter"]["Value"]))
    print(json.dumps(results))
    return results
```

📋 Zip it:

**macOS / Linux:**
```bash
zip function.zip lambda_function.py
```

**Windows (PowerShell):**
```powershell
Compress-Archive -Path lambda_function.py -DestinationPath function.zip -Force
```

Wait 10 seconds for the role to propagate, then 📋 **replacing `<ACCOUNT_ID>`**:
```
aws lambda create-function --function-name saa-config-reader --runtime python3.12 --handler lambda_function.lambda_handler --role arn:aws:iam::<ACCOUNT_ID>:role/saa-lab3c-lambda-role --zip-file fileb://function.zip --timeout 15
```

---

### Step 7: Invoke and Predict

🔮 **Predict** the result for each of the four keys before invoking: value returned or `DENIED`?

| Key | Prediction |
|-----|-----------|
| `feature_flag` | |
| `api_key` | |
| `db_password` | |
| `admin_root_password` | |

📋 Invoke:
```
aws lambda invoke --function-name saa-config-reader out.json
```

Open `out.json` in VS Code.

<details>
<summary>🔮 Reveal</summary>

```json
{
  "feature_flag": "dark-mode=on",
  "api_key": "sk-l**************************",
  "db_password": "Rota*************",
  "admin_root_password": "DENIED -> ClientError"
}
```

- `db_password` starts with **Rota**: the function automatically got the **rotated** (`AWSCURRENT`) value. No redeploy.
- `admin_root_password` is denied: `/saa/admin/*` isn't in the role's resources. **Least privilege works.**

</details>

### 🧩 Checkpoint

Your team wants Lambda to stop calling Secrets Manager on *every* invocation, to cut latency and API cost. What's the AWS-recommended approach?

<details>
<summary>Answer (+10 XP)</summary>

**Cache the secret**: store it in a variable outside the handler with a TTL, or use the **AWS Parameters and Secrets Lambda Extension**, which caches values locally for the function. Warm invocations reuse the cached value; rotation is picked up when the TTL expires.

</details>

---

### Step 8: Console Checkpoint

**✅ Checkpoint:**
1. **Systems Manager → Parameter Store** shows three parameters. SecureStrings show their KMS key.
2. **Secrets Manager → saa/app/db-credentials**: click **Retrieve secret value**, and see the **Rotation configuration** section (disabled; you'd enable it here for a real RDS database).
3. **Lambda → saa-config-reader → Monitor → View CloudWatch logs**: the masked output is logged, with no full secrets in the logs.

---

## ⚔️ Boss Challenge: Secure the Credentials Pipeline (+250 XP)

**Scenario:** An application on ECS Fargate in **Account A (prod)** connects to an **Amazon RDS for MySQL** database. Requirements:
1. The DB password must **rotate every 30 days automatically** with **no application downtime**.
2. A reporting Lambda function in **Account B (analytics)** must read the same credential.
3. Security requires that the secret is encrypted with a key **the company controls** and that **every decryption is audited**.
4. The password must never appear in task definitions or environment variables in plain text.

Write down your design: services, policy types and where each permission lives.

<details>
<summary>🏆 Answer key</summary>

1. **Secrets Manager** with **managed rotation** for RDS (or `--manage-master-user-password` on RDS itself) on a 30-day schedule. The alternating-users rotation strategy avoids downtime.
2. **Cross-account:** a **secret resource policy** allowing Account B's Lambda role `secretsmanager:GetSecretValue` **plus** an identity policy in Account B (cross-account = both sides!) **plus** the KMS key policy granting `kms:Decrypt` to that role.
3. Encrypt the secret with a **customer managed KMS key**. The default `aws/secretsmanager` key **can't be used cross-account** and isn't under your control. CloudTrail logs every `Decrypt`.
4. ECS task definitions reference the secret ARN in the **`secrets`** block (`valueFrom`). The **task execution role** fetches it at container start. Nothing is in plain text.

**Scoring:** rotation (+60) · resource policy + identity policy (+70) · CMK for cross-account (+70) · ECS `secrets` integration (+50)

</details>

---

## What You Just Did

1. Stored config in **Parameter Store** (String + SecureString + hierarchy)
2. Stored DB credentials in **Secrets Manager** and simulated a **rotation** with staging labels
3. Built a Lambda role scoped to **exact ARNs**, and proved it can't read anything else
4. Saw why customer managed keys change the KMS permission story

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Rotate database credentials automatically" | **Secrets Manager** |
| "Store configuration / license keys, lowest cost, no rotation" | **Parameter Store** (Standard) |
| "Hard-coded credentials in application code" | Move to **Secrets Manager** (rotation) or **Parameter Store** and retrieve at runtime via an IAM role |
| "Secret needed in multiple Regions for DR" | **Secrets Manager replication** |
| "Provision and auto-renew public TLS certificates for ALB/CloudFront" | **AWS Certificate Manager (ACM)** |
| "Containers on ECS need DB passwords" | Task definition `secrets` → Secrets Manager / Parameter Store |

**🚨 Exam traps**
- Parameter Store **can't rotate automatically** by itself. Any answer claiming it can is wrong.
- Environment variables with plain-text secrets are never the "most secure" answer.
- ACM certificates for **CloudFront** must be in **us-east-1**.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InvalidParameterValueException: The role defined for the function cannot be assumed` | The role hasn't propagated yet | Wait 15 seconds and retry `create-function` |
| Every key shows `DENIED` | Inline policy not attached, or the secret ARN is missing its suffix | `aws iam get-role-policy --role-name saa-lab3c-lambda-role --policy-name read-app-secrets` |
| `ResourceExistsException` on create-secret | A secret with that name is pending deletion | Use a new name, or `aws secretsmanager restore-secret --secret-id saa/app/db-credentials` |

---

## 🧹 Cleanup

```
aws lambda delete-function --function-name saa-config-reader
```
```
aws logs delete-log-group --log-group-name /aws/lambda/saa-config-reader
```
```
aws secretsmanager delete-secret --secret-id saa/app/db-credentials --force-delete-without-recovery
```
```
aws ssm delete-parameters --names /saa/app/feature-flag /saa/app/api-key /saa/admin/root-password
```
```
aws iam delete-role-policy --role-name saa-lab3c-lambda-role --policy-name read-app-secrets
```
```
aws iam detach-role-policy --role-name saa-lab3c-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```
```
aws iam delete-role --role-name saa-lab3c-lambda-role
```

> 💡 `--force-delete-without-recovery` skips the default 7–30 day recovery window. Fine for a lab; in production you'd want that window.

**Delete the local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-lab-3c`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-lab-3c`

---

**🏁 Lab complete: +100 XP.** Session 3 done! **[📝 Mini Exam 3 →](mini-exam-03.md)**
