# Lab 1B: Trust Policies, External ID & Permissions Boundaries

**Session:** 1 — Identity & Access Control  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%)  
**Difficulty:** Intermediate  
**Estimated Time:** 35–45 minutes

---

## Overview

Every IAM role has **two** policies that answer two different questions:

| Policy | Question It Answers |
|--------|--------------------|
| **Trust policy** | *Who* is allowed to become this role? |
| **Permissions policy** | *What* can the role do once someone becomes it? |

In this lab you'll lock a role's trust policy with an **External ID**, the exam's standard answer for "a third-party vendor needs access to our account." Then you'll cap a developer's power with a **permissions boundary**, the exam's standard answer for "let developers create IAM roles without letting them escalate their own privileges."

**What you will build:**
- A "vendor" role that can only be assumed when the caller supplies the correct External ID
- A "developer" role whose broad S3 permissions are trimmed down by a boundary
- A delegation rule: the developer can create new roles **only if** they attach the same boundary

---

## Prerequisites

- ✅ Completed **Lab 1A** (you understand implicit deny, explicit deny and resource vs. identity policies)
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| IAM / STS | Roles, policies, temporary credentials | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Confused deputy problem.** A SaaS vendor uses *one* AWS account to assume roles in *thousands* of customer accounts. If another customer learns your role ARN, they could trick the vendor into reaching into *your* account. The fix is an **External ID**: a secret value the vendor assigns to *you*, which your role's trust policy requires.

**Permissions boundary.** A managed policy attached to a user or role that sets the **maximum** permissions it can ever have. Effective permissions = **identity policy ∩ boundary**. A boundary never grants anything by itself.

```
  Identity policy: ████████████████████   (S3 full access)
  Boundary:        ██████                 (S3 read only)
  Effective:       ██████                 ← only the overlap
```

**Delegated administration.** You want developers to create roles for their Lambda functions, but if they could create a role with `AdministratorAccess` and assume it, they'd escalate their own privileges. The fix: allow `iam:CreateRole` **only when** the new role carries the boundary (`iam:PermissionsBoundary` condition key).

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI SSO profile name | `AdministratorAccess-123456789012` |
| `<ACCOUNT_ID>` | Your 12-digit account ID | `123456789012` |

---

## Lab Steps

### Step 1: Set Your Profile and Project Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-lab-1b; cd ~\Desktop\saa-lab-1b; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-lab-1b && cd ~/Desktop/saa-lab-1b && code .
```

📋 Confirm your identity and note the account ID:
```
aws sts get-caller-identity --query Account --output text
```

---

## Part 1 — External ID (Third-Party Access)

### Step 2: Create the Vendor Role

Pretend "Acme Monitoring" is a SaaS vendor that has asked for read-only access to your account. They gave you the External ID `acme-7Q2x-customer-0042`.

**Create `vendor-trust.json`**, **replacing `<ACCOUNT_ID>`**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "sts:ExternalId": "acme-7Q2x-customer-0042" }
      }
    }
  ]
}
```

> 💡 In real life, `Principal` would be **the vendor's** account ID, not yours. You only have one account, so you're playing both sides. The trust logic is identical.

📋 Create the role and give it read-only access:
```
aws iam create-role --role-name saa-lab1b-vendor-role --assume-role-policy-document file://vendor-trust.json
```
```
aws iam attach-role-policy --role-name saa-lab1b-vendor-role --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```

---

### Step 3: Try to Assume It Without the External ID

Create a CLI profile for the role, **without** an External ID:

```
aws configure set profile.saa-vendor.role_arn arn:aws:iam::<ACCOUNT_ID>:role/saa-lab1b-vendor-role
```
```
aws configure set profile.saa-vendor.source_profile <YOUR_PROFILE_NAME>
```

🔮 **Predict:** Will this succeed?
```
aws sts get-caller-identity --profile saa-vendor
```

<details>
<summary>🔮 Reveal</summary>

❌ **AccessDenied** when calling AssumeRole. The trust policy's `Condition` isn't met, so nobody may become this role, not even an admin of the same account.

</details>

---

### Step 4: Assume It With the External ID

📋 Add the External ID to the profile:
```
aws configure set profile.saa-vendor.external_id acme-7Q2x-customer-0042
```
```
aws sts get-caller-identity --profile saa-vendor
```

**✅ You should see** `assumed-role/saa-lab1b-vendor-role/...`

🔮 **Predict:** The role has `ReadOnlyAccess`. Will this work?
```
aws s3 mb s3://saa-lab1b-should-fail-<ACCOUNT_ID> --profile saa-vendor
```

<details>
<summary>🔮 Reveal</summary>

❌ **AccessDenied.** The trust policy decided *who* can become the role; the permissions policy (`ReadOnlyAccess`) decides *what* they can do. Creating a bucket is a write.

</details>

### 🧩 Checkpoint

Is the External ID a password that protects you if your role ARN leaks?

<details>
<summary>Answer (+10 XP)</summary>

**Not really.** The External ID isn't treated as a secret. Its job is to make sure the **vendor** only uses *your* External ID when acting for *you*, which stops another customer from pointing the vendor at your role (the confused deputy). The real protection is that `Principal` only trusts the vendor's account.

</details>

---

## Part 2 — Permissions Boundaries

### Step 5: Create the Boundary Policy

The boundary is the **ceiling**. Developers may read S3, and may create and manage roles, but nothing else.

**Create `dev-boundary.json`:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadOnly",
      "Effect": "Allow",
      "Action": ["s3:ListAllMyBuckets", "s3:ListBucket", "s3:GetObject"],
      "Resource": "*"
    },
    {
      "Sid": "RoleManagement",
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:GetRole", "iam:ListRoles"],
      "Resource": "*"
    }
  ]
}
```

📋 Create it as a **managed** policy (boundaries must be managed policies, not inline):
```
aws iam create-policy --policy-name saa-lab1b-dev-boundary --policy-document file://dev-boundary.json
```

---

### Step 6: Create the Developer Role, Generous Identity + Boundary

**Create `dev-trust.json`**, **replacing `<ACCOUNT_ID>`**:

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

📋 Create the role **with the boundary attached**:
```
aws iam create-role --role-name saa-lab1b-dev-role --assume-role-policy-document file://dev-trust.json --permissions-boundary arn:aws:iam::<ACCOUNT_ID>:policy/saa-lab1b-dev-boundary
```

📋 Give the developer a *generous* identity policy (full S3):
```
aws iam attach-role-policy --role-name saa-lab1b-dev-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

📋 Set up the profile:
```
aws configure set profile.saa-dev.role_arn arn:aws:iam::<ACCOUNT_ID>:role/saa-lab1b-dev-role
```
```
aws configure set profile.saa-dev.source_profile <YOUR_PROFILE_NAME>
```

Wait 10 seconds. 🔮 **Predict each**, then run:

| # | Command | Identity allows? | Boundary allows? | Your Prediction |
|---|---------|-----------------|------------------|-----------------|
| 1 | `aws s3 ls --profile saa-dev` | ✅ | ✅ | |
| 2 | `aws s3 mb s3://saa-lab1b-dev-<ACCOUNT_ID> --profile saa-dev` | ✅ | ❌ | |

<details>
<summary>🔮 Reveal</summary>

1. ✅ **Allowed.** Both policies allow it.
2. ❌ **AccessDenied.** `AmazonS3FullAccess` allows `CreateBucket`, but the boundary doesn't. Effective permissions are the **intersection**.

</details>

---

### Step 7: Delegate Role Creation Safely

Now let the developer create roles, **but only if** each new role also gets the boundary. That makes privilege escalation impossible.

**Create `dev-delegation.json`**, **replacing `<ACCOUNT_ID>`**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateRolesOnlyWithBoundary",
      "Effect": "Allow",
      "Action": "iam:CreateRole",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::<ACCOUNT_ID>:policy/saa-lab1b-dev-boundary"
        }
      }
    },
    {
      "Sid": "ReadRoles",
      "Effect": "Allow",
      "Action": ["iam:GetRole", "iam:ListRoles"],
      "Resource": "*"
    }
  ]
}
```

📋 Attach it inline to the dev role (with your **admin** profile):
```
aws iam put-role-policy --role-name saa-lab1b-dev-role --policy-name delegation --policy-document file://dev-delegation.json
```

Wait 10 seconds. 🔮 **Predict:** the developer tries to create a role with **no** boundary:

```
aws iam create-role --role-name saa-lab1b-sneaky-role --assume-role-policy-document file://dev-trust.json --profile saa-dev
```

<details>
<summary>🔮 Reveal</summary>

❌ **AccessDenied.** The condition `iam:PermissionsBoundary` wasn't satisfied. Without this rule, the developer could create an unrestricted role and assume it, a classic privilege-escalation path.

</details>

📋 Now **with** the boundary:
```
aws iam create-role --role-name saa-lab1b-lambda-role --assume-role-policy-document file://dev-trust.json --permissions-boundary arn:aws:iam::<ACCOUNT_ID>:policy/saa-lab1b-dev-boundary --profile saa-dev
```

**✅ You should see** the new role's JSON. The developer created a role, and it can never exceed the boundary.

---

### Step 8: Console Checkpoint

**✅ Checkpoint:**
1. **IAM → Roles → saa-lab1b-vendor-role → Trust relationships** shows the `sts:ExternalId` condition.
2. **IAM → Roles → saa-lab1b-dev-role → Permissions** shows the **Permissions boundary** section, separate from the permission policies.
3. **IAM → Roles → saa-lab1b-lambda-role** also shows the boundary.

---

## What You Just Did

1. Required an **External ID** in a trust policy to block the confused-deputy attack
2. Separated *who can assume* (trust) from *what they can do* (permissions)
3. Capped a role's permissions with a **boundary**, where effective permissions are the intersection
4. Built **safe delegation**: developers can create roles only inside the boundary

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Third-party / vendor / SaaS needs access to our account" | Cross-account **IAM role** + **External ID** |
| "Developers must create IAM roles but must not escalate privileges" | **Permissions boundary** + `iam:PermissionsBoundary` condition |
| "Users from another AWS account need access to our S3 bucket" | Bucket policy **and** their identity policy (cross-account = both sides) **or** a role they assume |
| "Federate corporate Active Directory users into AWS" | **IAM Identity Center** (or SAML 2.0 federation) |
| "Mobile app users need temporary AWS credentials" | **Amazon Cognito** identity pools |

**🚨 Exam traps**
- A boundary **does not grant** permissions. An answer that says "add a permissions boundary to give the user access to X" is wrong.
- Creating IAM **users with access keys** for a vendor is always the wrong answer when a role is an option.
- Boundaries limit **identities**. To limit **whole accounts**, use SCPs. That's the next lab.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Step 4 still denied | The profile didn't save the external ID, or it has a typo | Open `~/.aws/config` and check `external_id = acme-7Q2x-customer-0042` |
| `create-policy` says `EntityAlreadyExists` | A previous attempt left the policy behind | Skip creation and reuse it |
| Step 7 succeeds without a boundary | Your *admin* profile ran it | Make sure the command ends with `--profile saa-dev` |

---

## 🧹 Cleanup

📋 With your **admin** profile, **replacing `<ACCOUNT_ID>`**. Boundaries must be removed before the policy can be deleted:

```
aws iam delete-role-permissions-boundary --role-name saa-lab1b-lambda-role
```
```
aws iam delete-role --role-name saa-lab1b-lambda-role
```
```
aws iam delete-role-policy --role-name saa-lab1b-dev-role --policy-name delegation
```
```
aws iam detach-role-policy --role-name saa-lab1b-dev-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```
```
aws iam delete-role-permissions-boundary --role-name saa-lab1b-dev-role
```
```
aws iam delete-role --role-name saa-lab1b-dev-role
```
```
aws iam delete-policy --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/saa-lab1b-dev-boundary
```
```
aws iam detach-role-policy --role-name saa-lab1b-vendor-role --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess
```
```
aws iam delete-role --role-name saa-lab1b-vendor-role
```

**Remove the CLI profiles:** in `~/.aws/config`, delete the `[profile saa-vendor]` and `[profile saa-dev]` sections.

**Delete the local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-lab-1b`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-lab-1b`

**✅ Checkpoint:** IAM → Roles shows no `saa-lab1b-*` roles. IAM → Policies (filter "Customer managed") shows no `saa-lab1b-dev-boundary`.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 1C — Multi-Account Guardrails with SCPs](lab-1c-scp-guardrails.md)
