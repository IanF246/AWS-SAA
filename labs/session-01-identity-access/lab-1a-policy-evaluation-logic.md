# Lab 1A: IAM Policy Evaluation Logic — Who Wins When Policies Disagree?

**Session:** 1 — Identity & Access Control  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%)  
**Difficulty:** Beginner  
**Estimated Time:** 30–40 minutes

---

## Overview

The SAA exam loves questions where several policies apply at once: an identity policy says *allow*, a bucket policy says *deny*, and you have to say what happens. In this lab you **stop memorizing the evaluation rules and watch them happen**.

**What you will build:**
- An S3 bucket with three "folders": `public/`, `private/` and a `top-secret` file
- An IAM role that *you* assume from the CLI, with a deliberately limited identity policy
- A bucket policy (resource-based) that grants extra access
- An explicit deny that overrides everything

Then you'll use the **IAM Policy Simulator** from the CLI to test condition keys without creating any resources.

---

## Prerequisites

- ✅ AWS account with CLI v2 + IAM Identity Center (SSO) profile ([AI Cloud Fusion Lab 1A](https://github.com/IanF246/AICloudFusion/blob/main/labs/session-01-cloud-concepts/lab-1a-aws-cli-setup.md))
- ✅ VS Code with the `code` command

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| IAM / STS | Identities, roles, temporary credentials | Always Free |
| Amazon S3 | A few bytes of test files | ~$0.00 |
| IAM Policy Simulator | Policy testing API | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**The evaluation flow (single account).** AWS evaluates every request like this:

```
                  ┌───────────────────────────────┐
  Request ──────▶ │ Any EXPLICIT DENY anywhere?   │── yes ──▶ ❌ DENY (game over)
                  └───────────────┬───────────────┘
                                  │ no
                  ┌───────────────▼───────────────┐
                  │ SCPs / permission boundaries  │── don't allow ──▶ ❌ DENY
                  │ (if present) allow it?        │
                  └───────────────┬───────────────┘
                                  │ yes
                  ┌───────────────▼───────────────┐
                  │ Identity policy OR resource   │── yes ──▶ ✅ ALLOW
                  │ policy allows it?             │
                  └───────────────┬───────────────┘
                                  │ no
                                  ▼
                         ❌ DENY (implicit — the default)
```

**Implicit deny:** Everything is denied unless something allows it.  
**Explicit deny:** A `"Effect": "Deny"` statement. It **always wins**, no matter how many allows exist.  
**Identity-based policy:** Attached to a user, group or role. Says what *that identity* can do.  
**Resource-based policy:** Attached to a resource (S3 bucket, SQS queue, KMS key, Lambda function). Says *who* can access *this resource*. It has a `Principal` element; identity policies don't.

> 💡 **Same account vs. cross-account:** Within one account, an allow in **either** the identity policy **or** the resource policy is enough. Across accounts, you need an allow on **both** sides. You'll see the same-account rule in action in this lab, and the cross-account rule in Lab 1B.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI SSO profile name | `AdministratorAccess-123456789012` |
| `<ACCOUNT_ID>` | Your 12-digit AWS account ID (Step 1) | `123456789012` |

---

## Lab Steps

### Step 1: Set Your Profile and Get Your Account ID

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

📋 Copy and paste:
```
aws sts get-caller-identity --query Account --output text
```

> **📝 Write down your Account ID:** ______________________________

If you get an expired-token error, run `aws sso login --profile <YOUR_PROFILE_NAME>`.

---

### Step 2: Create Your Project Folder

**Windows (PowerShell):**
```powershell
mkdir ~\Desktop\saa-lab-1a; cd ~\Desktop\saa-lab-1a; code .
```

**macOS / Linux:**
```bash
mkdir -p ~/Desktop/saa-lab-1a && cd ~/Desktop/saa-lab-1a && code .
```

> 💡 Save every file you create in this lab into this folder. Your terminal looks for `file://` paths here.

---

### Step 3: Create the Bucket and Test Files

📋 Copy and paste, **replacing `<ACCOUNT_ID>`**:
```
aws s3 mb s3://saa-lab1a-<ACCOUNT_ID> --region us-east-1
```

Create a tiny file and upload it to three locations:
```
echo hello > note.txt
```
```
aws s3 cp note.txt s3://saa-lab1a-<ACCOUNT_ID>/public/note.txt
```
```
aws s3 cp note.txt s3://saa-lab1a-<ACCOUNT_ID>/private/note.txt
```
```
aws s3 cp note.txt s3://saa-lab1a-<ACCOUNT_ID>/private/top-secret.txt
```

**✅ You should see** three `upload:` lines.

---

### Step 4: Create a Role You Can Assume

You're going to *become* a less-privileged role so you can feel the policies take effect.

**Create `trust-policy.json`** in VS Code, **replacing `<ACCOUNT_ID>`**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

> 💡 **What does `:root` mean here?** It doesn't mean "only the root user." It means "**this account**": any IAM principal in it whose *own* permissions allow `sts:AssumeRole` on this role. Your admin SSO role has those permissions.

📋 Create the role:
```
aws iam create-role --role-name saa-lab1a-role --assume-role-policy-document file://trust-policy.json
```

**Create `identity-v1.json`**, **replacing `<ACCOUNT_ID>`** in both places. It allows listing the bucket and reading **only** the `public/` prefix:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTheBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::saa-lab1a-<ACCOUNT_ID>"
    },
    {
      "Sid": "ReadPublicOnly",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::saa-lab1a-<ACCOUNT_ID>/public/*"
    }
  ]
}
```

📋 Attach it as an inline policy:
```
aws iam put-role-policy --role-name saa-lab1a-role --policy-name identity --policy-document file://identity-v1.json
```

> 🚨 **Exam trap:** `s3:ListBucket` applies to the **bucket** ARN (`arn:aws:s3:::bucket`). `s3:GetObject` and `s3:PutObject` apply to the **object** ARN (`arn:aws:s3:::bucket/*`). Mix them up and you get Access Denied, and the exam will show you exactly this broken policy.

---

### Step 5: Create a CLI Profile That Assumes the Role

The CLI can assume a role automatically for you. 📋 Run both, **replacing the placeholders**:

```
aws configure set profile.saa-lab1a.role_arn arn:aws:iam::<ACCOUNT_ID>:role/saa-lab1a-role
```
```
aws configure set profile.saa-lab1a.source_profile <YOUR_PROFILE_NAME>
```

Wait about 10 seconds for IAM to propagate, then confirm who you are:

```
aws sts get-caller-identity --profile saa-lab1a
```

**✅ You should see** an `Arn` like `arn:aws:sts::<ACCOUNT_ID>:assumed-role/saa-lab1a-role/botocore-session-...`

> 💡 You are now using **temporary credentials** from STS. Nothing long-lived was created. The exam's answer to "how should X get credentials?" is almost always "a role."

---

### Step 6: Round 1 — Identity Policy Only

For each command below, **🔮 predict first**, *then* run it.

> 🎯 **How to play:** for each prediction, tick **one** box by changing `[ ]` to `[x]` in the file (VS Code makes this quick: put your cursor in the brackets and type `x`). Commit to your answer *before* you run the command, then open that prediction's **Reveal**. +10 XP for every correct tick.

> 💡 The `-` at the end of a `cp` means "print to the screen" instead of saving a file.

#### 🔮 Prediction 1: list the bucket

```
aws s3 ls s3://saa-lab1a-<ACCOUNT_ID>/ --recursive --profile saa-lab1a
```

- [ ] ✅ Allowed
- [ ] ❌ AccessDenied: **implicit** deny (nothing allows it)
- [ ] ❌ AccessDenied: **explicit** deny (a Deny statement blocks it)

<details>
<summary>🔮 Reveal #1</summary>

✅ **Allowed.** The identity policy grants `s3:ListBucket` on the **bucket** ARN.

</details>

#### 🔮 Prediction 2: read a public file

```
aws s3 cp s3://saa-lab1a-<ACCOUNT_ID>/public/note.txt - --profile saa-lab1a
```

- [ ] ✅ Allowed
- [ ] ❌ AccessDenied: **implicit** deny (nothing allows it)
- [ ] ❌ AccessDenied: **explicit** deny (a Deny statement blocks it)

<details>
<summary>🔮 Reveal #2</summary>

✅ **Allowed.** `s3:GetObject` on `public/*` matches.

</details>

#### 🔮 Prediction 3: read a private file

```
aws s3 cp s3://saa-lab1a-<ACCOUNT_ID>/private/note.txt - --profile saa-lab1a
```

- [ ] ✅ Allowed
- [ ] ❌ AccessDenied: **implicit** deny (nothing allows it)
- [ ] ❌ AccessDenied: **explicit** deny (a Deny statement blocks it)

<details>
<summary>🔮 Reveal #3</summary>

❌ **Implicit deny.** No policy *denies* it, but nothing *allows* `private/*`, so the default "no" applies. If you ticked "explicit," look again: there's no `Deny` statement anywhere yet.

</details>

#### 🔮 Prediction 4: upload a file

```
aws s3 cp note.txt s3://saa-lab1a-<ACCOUNT_ID>/public/new.txt --profile saa-lab1a
```

- [ ] ✅ Allowed
- [ ] ❌ AccessDenied: **implicit** deny (nothing allows it)
- [ ] ❌ AccessDenied: **explicit** deny (a Deny statement blocks it)

<details>
<summary>🔮 Reveal #4</summary>

❌ **Implicit deny.** The role can *read* `public/*` but nothing allows `s3:PutObject`. Being allowed to read a path doesn't mean you can write to it.

</details>

### 📊 Round 1 Score

| Prediction | 1 | 2 | 3 | 4 | Total |
|------------|---|---|---|---|-------|
| Correct? (✅/❌) | | | | | __ / 4 → **+__ XP** |

---

### Step 7: Round 2 — Add a Resource-Based Policy

Now grant the role access to `private/*` through the **bucket policy**, without touching the identity policy.

**Create `bucket-policy.json`**, **replacing `<ACCOUNT_ID>`** (three places):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LetLabRoleReadPrivate",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/saa-lab1a-role" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::saa-lab1a-<ACCOUNT_ID>/private/*"
    }
  ]
}
```

📋 Apply it with your **admin** profile (no `--profile saa-lab1a`):
```
aws s3api put-bucket-policy --bucket saa-lab1a-<ACCOUNT_ID> --policy file://bucket-policy.json
```

#### 🔮 Prediction 5: read the private file again

Will Prediction 3's command work now? The role's identity policy still doesn't mention `private/`.

- [ ] ✅ Allowed: the bucket policy alone is enough
- [ ] ❌ Still denied: the identity policy must allow it too
- [ ] ❌ Denied: the bucket policy creates an explicit deny for everyone else

📋 Run it:
```
aws s3 cp s3://saa-lab1a-<ACCOUNT_ID>/private/note.txt - --profile saa-lab1a
```

<details>
<summary>🔮 Reveal #5</summary>

✅ **Allowed: the bucket policy alone is enough.** In the **same account**, an allow in the identity policy **OR** the resource policy is enough. The bucket policy names the role as a `Principal`, so the request is allowed.

If this were **cross-account**, the role's own identity policy would *also* need to allow it.

</details>

---

### Step 8: Round 3 — The Explicit Deny

The security team says nobody may read `top-secret.txt`. Add an **explicit deny** to the role's identity policy.

**Create `identity-v2.json`**, **replacing `<ACCOUNT_ID>`** (three places):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTheBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::saa-lab1a-<ACCOUNT_ID>"
    },
    {
      "Sid": "ReadPublicOnly",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::saa-lab1a-<ACCOUNT_ID>/public/*"
    },
    {
      "Sid": "NeverTopSecret",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::saa-lab1a-<ACCOUNT_ID>/private/top-secret.txt"
    }
  ]
}
```

📋 Replace the inline policy:
```
aws iam put-role-policy --role-name saa-lab1a-role --policy-name identity --policy-document file://identity-v2.json
```

Wait about 10 seconds, then predict both results before running them.

#### 🔮 Prediction 6: `private/note.txt`

```
aws s3 cp s3://saa-lab1a-<ACCOUNT_ID>/private/note.txt - --profile saa-lab1a
```

- [ ] ✅ Allowed
- [ ] ❌ AccessDenied: **implicit** deny (nothing allows it)
- [ ] ❌ AccessDenied: **explicit** deny (a Deny statement blocks it)

#### 🔮 Prediction 7: `private/top-secret.txt`

The bucket policy **allows** `private/*`. The identity policy **denies** `top-secret.txt`.

```
aws s3 cp s3://saa-lab1a-<ACCOUNT_ID>/private/top-secret.txt - --profile saa-lab1a
```

- [ ] ✅ Allowed: the bucket policy's allow wins because it's more specific to the bucket
- [ ] ✅ Allowed: allows and denies cancel out, and the resource policy decides
- [ ] ❌ AccessDenied: the explicit deny wins

<details>
<summary>🔮 Reveal #6 and #7</summary>

- **#6** `private/note.txt` → ✅ still allowed by the bucket policy. The deny only targets one file.
- **#7** `private/top-secret.txt` → ❌ **AccessDenied.** The bucket policy *allows* `private/*`, but the identity policy has an **explicit deny**, and an explicit deny always wins, wherever it's written.

</details>

---

### 🧩 Checkpoint

A bucket policy has `"Effect": "Deny", "Principal": "*", "Action": "s3:*"` with a condition that the request is **not** over HTTPS. An admin with `AdministratorAccess` sends a request over plain HTTP. What happens?

<details>
<summary>Answer (+10 XP)</summary>

❌ **Denied.** `AdministratorAccess` is just an allow. The bucket policy's explicit deny (its condition is met because the request isn't HTTPS) overrides it. This "enforce TLS" bucket policy is a classic exam pattern, and you'll build it in Lab 3B.

</details>

---

### Step 9: Test Conditions with the Policy Simulator

Condition keys narrow *when* a statement applies. Let's test a **region restriction** without creating anything.

**Create `region-guard.json`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["us-east-1", "us-west-2"] }
      }
    }
  ]
}
```

#### 🔮 Prediction 8: `ec2:RunInstances` in **us-east-1**

- [ ] `allowed`
- [ ] `implicitDeny`
- [ ] `explicitDeny`

#### 🔮 Prediction 9: `ec2:RunInstances` in **eu-west-1**

- [ ] `allowed`
- [ ] `implicitDeny`
- [ ] `explicitDeny`

📋 Simulate us-east-1:
```
aws iam simulate-custom-policy --policy-input-list file://region-guard.json --action-names ec2:RunInstances --context-entries "ContextKeyName=aws:RequestedRegion,ContextKeyValues=us-east-1,ContextKeyType=string" --query "EvaluationResults[0].EvalDecision" --output text
```

📋 Simulate eu-west-1:
```
aws iam simulate-custom-policy --policy-input-list file://region-guard.json --action-names ec2:RunInstances --context-entries "ContextKeyName=aws:RequestedRegion,ContextKeyValues=eu-west-1,ContextKeyType=string" --query "EvaluationResults[0].EvalDecision" --output text
```

<details>
<summary>🔮 Reveal #8 and #9</summary>

- **#8 us-east-1** → `allowed`. It's in the approved list, so the Deny's `StringNotEquals` condition is **false** and the Deny doesn't apply. The Allow stands.
- **#9 eu-west-1** → `explicitDeny`. Not in the list, so the condition is **true** and the Deny applies. It isn't `implicitDeny`, because a Deny statement matched.

</details>

### 📊 Lab 1A Prediction Scorecard

| # | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | Total |
|---|---|---|---|---|---|---|---|---|---|-------|
| ✅/❌ | | | | | | | | | | __ / 9 → **+__ XP** |

**9/9:** 🧠 Policy Whisperer · **7–8:** solid · **≤ 6:** reread the flowchart in **Concepts** before Lab 1B.

> 💡 The simulator returns three possible decisions: `allowed`, `explicitDeny` and `implicitDeny`. Those are the three outcomes from the flowchart in **Concepts**.

---

### Step 10: Console Checkpoint

**✅ Checkpoint:**
1. Open **IAM → Roles → saa-lab1a-role → Permissions**. You'll see the inline policy `identity`.
2. Open **S3 → saa-lab1a-… → Permissions → Bucket policy**. You'll see your resource-based policy with its `Principal`.
3. *(Bonus)* Open the **IAM Policy Simulator** in the console (search "policy simulator"), pick the role, and test `s3:GetObject` on `private/top-secret.txt`. It shows which statement caused the deny.

---

## What You Just Did

1. Assumed a role with temporary STS credentials, the right way to grant access
2. Watched an **implicit deny** block access nothing had allowed
3. Saw a **resource-based policy** grant access by itself (same-account union)
4. Watched an **explicit deny** override an allow from another policy
5. Tested **condition keys** with the IAM Policy Simulator

---

## 🎯 Exam Corner

| Fact | Remember It As |
|------|---------------|
| Explicit deny always wins | "Deny beats everything, everywhere" |
| Default is implicit deny | "Silence = no" |
| Same account: identity **OR** resource policy allows | "Either key opens the door" |
| Cross-account: identity **AND** resource policy must allow | "Both sides must agree" |
| Resource policies have `Principal`; identity policies don't | "Resource policies name *who*" |
| `aws:RequestedRegion`, `aws:SourceIp`, `aws:SecureTransport`, `aws:PrincipalOrgID` | The four condition keys you'll see most often on the exam |

**🚨 Exam traps**
- An answer that attaches **access keys** to an EC2 instance or Lambda function is always wrong. Use a role.
- "Grant the least privilege" means scoping **actions and resources**, not attaching `AdministratorAccess` "temporarily."
- `aws:PrincipalOrgID` in a bucket policy is the cleanest way to say "anyone in my AWS Organization." That's a common correct answer.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `is not authorized to perform: sts:AssumeRole` | The trust policy has the wrong account ID, or IAM hasn't propagated yet | Check `trust-policy.json`, wait 15 seconds and retry |
| `MalformedPolicy` on the bucket policy | The role doesn't exist yet, or there's an ARN typo | Confirm the role with `aws iam get-role --role-name saa-lab1a-role` |
| Step 8 deny doesn't apply | Propagation delay | Wait 15–30 seconds and retry |
| `Error when retrieving token from sso` | Your SSO session expired | `aws sso login --profile <YOUR_PROFILE_NAME>` |

---

## 🧹 Cleanup

📋 Run these with your **admin** profile, **replacing `<ACCOUNT_ID>`**:

```
aws s3 rm s3://saa-lab1a-<ACCOUNT_ID> --recursive
```
```
aws s3 rb s3://saa-lab1a-<ACCOUNT_ID>
```
```
aws iam delete-role-policy --role-name saa-lab1a-role --policy-name identity
```
```
aws iam delete-role --role-name saa-lab1a-role
```

**Remove the `saa-lab1a` CLI profile.** Open `~/.aws/config` (Windows: `%USERPROFILE%\.aws\config`) in VS Code and delete the `[profile saa-lab1a]` section (3 lines).

**Delete the local folder** (close VS Code first):

**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-lab-1a`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-lab-1a`

**✅ Checkpoint:** S3 shows no `saa-lab1a-…` bucket. IAM → Roles shows no `saa-lab1a-role`.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 1B — Trust Policies, External ID & Permissions Boundaries](lab-1b-trust-policies-boundaries.md)
