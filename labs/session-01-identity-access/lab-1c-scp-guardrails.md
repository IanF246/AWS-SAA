# Lab 1C: Multi-Account Guardrails with Service Control Policies

**Session:** 1 — Identity & Access Control  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%)  
**Difficulty:** Advanced  
**Estimated Time:** 40–50 minutes

---

## Overview

Permissions boundaries limit **one identity**. **Service Control Policies (SCPs)** limit **entire AWS accounts**: every user and role in them, including the account's root user. The SAA exam uses SCPs whenever a scenario says *"across all accounts in the organization"* or *"even administrators must not be able to…"*.

In this lab you'll act as the cloud platform team. You'll write three production-grade SCPs, **validate them with IAM Access Analyzer**, and attach them to a sandbox Organizational Unit (OU), all **without risking the account you work in**.

**What you will build:**
- A `saa-lab-sandbox` OU in your AWS Organization
- Three SCPs: **region lockdown**, **protect the audit trail** and **don't leave the organization**
- An automated policy-linting step with Access Analyzer

---

## Prerequisites

- ✅ Completed **Labs 1A and 1B**
- ✅ Your account is the **management account** of an AWS Organization. If you set up IAM Identity Center with AI Cloud Fusion Lab 1A, it is.
- ✅ AWS CLI authenticated with an admin profile

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| AWS Organizations | Multi-account management, OUs, SCPs | Always Free |
| IAM Access Analyzer — policy validation | Lints policies for errors and risks | Always Free |

**Estimated cost for this lab: $0.00**

> 🛡️ **Safety note:** SCPs **never affect the management account**. You'll attach them to an *empty* OU, so nothing can lock you out. That fact is also an exam question.

---

## Concepts

```
                     ┌──────────────────────────┐
                     │   Root (FullAWSAccess)   │
                     └────────────┬─────────────┘
              ┌───────────────────┼────────────────────┐
     ┌────────▼────────┐  ┌───────▼────────┐  ┌────────▼────────┐
     │ Management acct │  │  OU: Prod      │  │ OU: Sandbox     │
     │ ⚠️ SCPs ignored  │  │  SCP: deny X   │  │ SCP: regions    │
     └─────────────────┘  │  ├ acct A      │  │  └ (empty)      │
                          │  └ acct B      │  └─────────────────┘
                          └────────────────┘
```

| SCP Fact | Why It Matters on the Exam |
|----------|---------------------------|
| SCPs **never grant** permissions | They are filters. Identities still need IAM allows. |
| SCPs **don't apply to the management account** | "Restrict the management account" + SCP = wrong answer |
| SCPs **do** apply to the **root user** of member accounts | The only way to restrict a member account's root user |
| SCPs **don't** restrict **service-linked roles** | AWS services can still do their work |
| Inherited down the OU tree | An action must be allowed at **every** level from root → OU → account |

**Deny-list vs. allow-list strategy.** By default, AWS attaches `FullAWSAccess` everywhere, and you add **Deny** SCPs (deny-list). The alternative, removing `FullAWSAccess` and allowing only specific services (allow-list), is stricter but higher-maintenance.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI SSO profile name | `AdministratorAccess-123456789012` |
| `<ROOT_ID>` | Your organization's root ID (Step 2) | `r-ab12` |
| `<OU_ID>` | The sandbox OU ID (Step 4) | `ou-ab12-cd34ef56` |
| `<POLICY_ID_1/2/3>` | SCP IDs (Step 6) | `p-abc123de` |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-lab-1c; cd ~\Desktop\saa-lab-1c; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-lab-1c && cd ~/Desktop/saa-lab-1c && code .
```

---

### Step 2: Inspect Your Organization

📋 Confirm you're in the management account:
```
aws organizations describe-organization --query "Organization.[Id,MasterAccountId,FeatureSet]" --output table
```

**✅ You should see** the org ID, a management account ID that **matches your account ID**, and `ALL` (all features are needed for SCPs).

📋 Get the root ID and check which policy types are enabled:
```
aws organizations list-roots --query "Roots[0].[Id,PolicyTypes]" --output json
```

> **📝 Write down your Root ID:** ______________________________
>
> **📝 Is `SERVICE_CONTROL_POLICY` listed as `ENABLED`?** ☐ Yes ☐ No

---

### Step 3: Enable SCPs (Only If Needed)

**Skip this step if** SCPs were already `ENABLED` in Step 2.

📋 Otherwise, **replacing `<ROOT_ID>`**:
```
aws organizations enable-policy-type --root-id <ROOT_ID> --policy-type SERVICE_CONTROL_POLICY
```

> 💡 Enabling SCPs automatically attaches the AWS-managed `FullAWSAccess` SCP to the root and every OU and account. Nothing changes for anyone until you add a Deny.

---

### Step 4: Create the Sandbox OU

📋 **Replacing `<ROOT_ID>`**:
```
aws organizations create-organizational-unit --parent-id <ROOT_ID> --name saa-lab-sandbox --query "OrganizationalUnit.Id" --output text
```

> **📝 Write down your OU ID:** ______________________________

---

### Step 5: Write Three SCPs

**SCP 1: `scp-region-lockdown.json`.** Deny all actions outside the two approved regions, except for **global services** (which are served from us-east-1 and would break otherwise) and except for the platform team's break-glass role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*",
        "organizations:*",
        "sts:*",
        "route53:*",
        "cloudfront:*",
        "budgets:*",
        "ce:*",
        "globalaccelerator:*",
        "support:*",
        "waf:*",
        "health:*",
        "sso:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/PlatformBreakGlass"
        }
      }
    }
  ]
}
```

> 💡 **`NotAction` + `Deny`** means "deny everything *except* these." It's not the same as `Allow` + `Action`. 🚨 The exam may show both and ask which one restricts regions correctly.

**SCP 2: `scp-protect-audit.json`.** Nobody in the account, not even its root user, may switch off CloudTrail or GuardDuty.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectCloudTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail",
        "cloudtrail:PutEventSelectors"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ProtectGuardDuty",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "guardduty:UpdateDetector"
      ],
      "Resource": "*"
    }
  ]
}
```

**SCP 3: `scp-stay-in-org.json`.** Accounts can't leave the organization, which would escape every guardrail at once.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLeavingOrg",
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

---

### Step 6: Lint the Policies with Access Analyzer

Before attaching guardrails to production, platform teams validate them automatically. 📋 Run once per file:

```
aws accessanalyzer validate-policy --policy-type SERVICE_CONTROL_POLICY --policy-document file://scp-region-lockdown.json --query "findings[].[findingType,issueCode]" --output table
```
```
aws accessanalyzer validate-policy --policy-type SERVICE_CONTROL_POLICY --policy-document file://scp-protect-audit.json --query "findings[].[findingType,issueCode]" --output table
```
```
aws accessanalyzer validate-policy --policy-type SERVICE_CONTROL_POLICY --policy-document file://scp-stay-in-org.json --query "findings[].[findingType,issueCode]" --output table
```

**✅ You should see** empty output (no findings) for each.

🔮 **Break it on purpose.** In `scp-stay-in-org.json`, change `"organizations:LeaveOrganization"` to `"organizations:LeaveOrganisation"` (British spelling). Predict the result, then re-run the validation.

<details>
<summary>🔮 Reveal</summary>

You'll get a finding like `ERROR | INVALID_ACTION`. Without validation, this typo would ship silently, and the guardrail would protect **nothing**. **Change the spelling back** before continuing.

</details>

---

### Step 7: Create and Attach the SCPs

📋 Create each policy. Each command prints a policy ID; write them down.

```
aws organizations create-policy --name saa-lab-region-lockdown --type SERVICE_CONTROL_POLICY --description "Approved regions only" --content file://scp-region-lockdown.json --query "Policy.PolicySummary.Id" --output text
```
```
aws organizations create-policy --name saa-lab-protect-audit --type SERVICE_CONTROL_POLICY --description "Protect CloudTrail and GuardDuty" --content file://scp-protect-audit.json --query "Policy.PolicySummary.Id" --output text
```
```
aws organizations create-policy --name saa-lab-stay-in-org --type SERVICE_CONTROL_POLICY --description "Deny leaving the organization" --content file://scp-stay-in-org.json --query "Policy.PolicySummary.Id" --output text
```

> **📝 Policy IDs:** 1 ________ 2 ________ 3 ________

📋 Attach each to the sandbox OU, **replacing `<POLICY_ID_n>` and `<OU_ID>`**:
```
aws organizations attach-policy --policy-id <POLICY_ID_1> --target-id <OU_ID>
```
```
aws organizations attach-policy --policy-id <POLICY_ID_2> --target-id <OU_ID>
```
```
aws organizations attach-policy --policy-id <POLICY_ID_3> --target-id <OU_ID>
```

📋 Confirm:
```
aws organizations list-policies-for-target --target-id <OU_ID> --filter SERVICE_CONTROL_POLICY --query "Policies[].Name" --output table
```

**✅ You should see** four policies: `FullAWSAccess` (inherited automatically) plus your three.

---

### Step 8: Prove the Management Account Is Unaffected

🔮 **Predict:** You attached a region lockdown. Can *you* still call EC2 in `eu-west-1`?

```
aws ec2 describe-availability-zones --region eu-west-1 --query "AvailabilityZones[0].ZoneName" --output text
```

<details>
<summary>🔮 Reveal</summary>

✅ **Yes.** Two reasons: (1) the SCP is on the sandbox OU, not on your account, and (2) even if it were attached to the root, **SCPs never apply to the management account**. That's why AWS's best practice is to run **no workloads** in the management account.

</details>

---

### 🧩 Checkpoint

A member account's **root user** tries to call `cloudtrail:StopLogging`. The account sits in the sandbox OU. What happens, and could an IAM policy in that account change the outcome?

<details>
<summary>Answer (+10 XP)</summary>

❌ **Denied**, and **no**, nothing inside the account can change that. SCPs apply to member-account root users, and an explicit deny in an SCP overrides any IAM allow. Only the management account can detach the SCP.

</details>

---

### Step 9: Console Checkpoint

**✅ Checkpoint:**
1. **AWS Organizations → AWS accounts** shows `saa-lab-sandbox` under Root.
2. Click the OU → **Policies** tab shows your three SCPs plus `FullAWSAccess`.
3. **Organizations → Policies → Service control policies → saa-lab-region-lockdown** shows the JSON and its **Targets**.

---

## ⚔️ Boss Challenge: Design the Guardrails (+250 XP)

No copy-paste help this time. Write the SCP(s) yourself, validate them with Access Analyzer, and compare with the answer key.

**Scenario:** A fintech company has a `Workloads` OU. Requirements:
1. No one may launch EC2 instances larger than `*.large` (to cap costs).
2. S3 buckets must never have their Block Public Access settings removed.
3. Requirement 1 must **not** apply to the role `arn:aws:iam::*:role/HPC-Approved`.

<details>
<summary>🏆 Answer key — open only after you've written yours</summary>

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LimitInstanceSize",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotLike": {
          "ec2:InstanceType": ["*.nano", "*.micro", "*.small", "*.medium", "*.large"]
        },
        "ArnNotLike": {
          "aws:PrincipalARN": "arn:aws:iam::*:role/HPC-Approved"
        }
      }
    },
    {
      "Sid": "KeepBlockPublicAccess",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:PutAccountPublicAccessBlock"
      ],
      "Resource": "*"
    }
  ]
}
```

**What makes it work:**
- `Resource` is scoped to `instance/*`. `RunInstances` also touches volumes, ENIs and AMIs, which don't carry an `ec2:InstanceType` key.
- Two conditions in one statement are **AND-ed**: deny only if the size is big **and** the caller isn't the HPC role.
- Denying `PutBucketPublicAccessBlock` stops anyone from *changing* the setting at all. Real platforms often exempt the platform role the same way as in requirement 3.

**Self-score:** both requirements correct plus the exemption = full XP. Missing the `Resource` scoping = 150 XP.

</details>

---

## What You Just Did

1. Built an OU structure and enabled SCPs
2. Wrote the three SCPs nearly every real AWS organization runs
3. Used **Access Analyzer** to catch a silent typo that would have disabled a guardrail
4. Proved that SCPs don't reach the management account

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Prevent **all accounts** from using unapproved regions" | **SCP** with `aws:RequestedRegion` |
| "Even the **root user** of member accounts must not…" | **SCP** |
| "Centrally manage **many accounts** with guardrails + account vending" | **AWS Control Tower** (it builds on Organizations + SCPs) |
| "Share a resource (subnets, Transit Gateway) across accounts" | **AWS RAM** (Resource Access Manager) |
| "Consolidated billing + volume discounts" | **AWS Organizations** |
| "Enforce tags with standard keys/values across accounts" | **Tag policies** (Organizations) |

**🚨 Exam traps**
- "Attach an SCP to grant developers S3 access" → wrong. SCPs never grant.
- "Use an SCP to restrict the management account" → wrong. They don't apply there.
- "Use IAM policies in each account" when the question says *centrally* or *all accounts* → usually wrong. Use SCPs.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `AWSOrganizationsNotInUseException` | Your account isn't in an organization | Create one: `aws organizations create-organization --feature-set ALL` |
| `AccessDeniedException` on Organizations calls | You're in a member account, not the management account | Use a profile for the management account |
| `PolicyTypeNotEnabledException` | SCPs aren't enabled | Do Step 3 |
| `DuplicatePolicyException` | A policy with that name exists from a previous attempt | Reuse it: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY` |

---

## 🧹 Cleanup

📋 Detach and delete each policy, **replacing the IDs**:
```
aws organizations detach-policy --policy-id <POLICY_ID_1> --target-id <OU_ID>
```
```
aws organizations detach-policy --policy-id <POLICY_ID_2> --target-id <OU_ID>
```
```
aws organizations detach-policy --policy-id <POLICY_ID_3> --target-id <OU_ID>
```
```
aws organizations delete-policy --policy-id <POLICY_ID_1>
```
```
aws organizations delete-policy --policy-id <POLICY_ID_2>
```
```
aws organizations delete-policy --policy-id <POLICY_ID_3>
```
```
aws organizations delete-organizational-unit --organizational-unit-id <OU_ID>
```

*(Optional)* If SCPs were **not** enabled before Step 3 and you want to restore that state:
```
aws organizations disable-policy-type --root-id <ROOT_ID> --policy-type SERVICE_CONTROL_POLICY
```

**Delete the local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-lab-1c`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-lab-1c`

**✅ Checkpoint:** Organizations → AWS accounts no longer shows `saa-lab-sandbox`.

---

**🏁 Lab complete: +100 XP.** Session 1 done! Now prove it: **[📝 Mini Exam 1 →](mini-exam-01.md)**
