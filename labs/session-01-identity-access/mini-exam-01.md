# 📝 Mini Exam 1: Identity & Access Control

**Covers:** Labs 1A, 1B, 1C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes (the real exam gives ~2 min per question)  
**Pass mark:** 8/10 (80%) · **Reward:** +200 XP for passing, +100 bonus for a perfect score

---

## How to Take This Exam

1. ⏱️ **Start a 15-minute timer.**
2. Read each scenario and write your answer in the **Answer Sheet** below *before* you open any answer.
3. When time's up (or you're done), open each **Reveal** and mark yourself.
4. For every miss, follow the **🔁 Review** link and add the topic to the weak-spot log in the [README](../../README.md#-progress-tracker).

> 💡 **Exam technique:** Read the **last sentence first**. It tells you what's being optimized ("most secure," "least operational overhead," "most cost-effective"). Then read the scenario looking for the constraint that rules answers out.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

An application on Amazon EC2 needs to read objects from an S3 bucket in the same account. A developer proposes storing an IAM user's access keys in the application's config file. **What is the MOST secure solution?**

- **A.** Store the access keys in AWS Systems Manager Parameter Store as a SecureString
- **B.** Create an IAM role with S3 read permissions and attach it to the instance through an instance profile
- **C.** Encrypt the config file with KMS and decrypt the keys at boot
- **D.** Make the bucket public but restrict it by the instance's public IP with a bucket policy

<details>
<summary>Reveal answer</summary>

**✅ B.** Roles give the instance **temporary, automatically rotated** credentials. Nothing long-lived ever touches the server.

- A and C still rely on **long-lived** access keys; they just hide them better.
- D exposes the bucket publicly, and public IPs can change.

🔁 **Review:** [Lab 1A, Step 5](lab-1a-policy-evaluation-logic.md#step-5-create-a-cli-profile-that-assumes-the-role)

</details>

---

### Question 2

A role's identity policy allows `s3:*` on `arn:aws:s3:::reports/*`. The `reports` bucket policy (same account) contains a statement that **denies** `s3:DeleteObject` to all principals. The role tries to delete an object. **What happens?**

- **A.** The delete succeeds because the identity policy is evaluated first
- **B.** The delete succeeds because `s3:*` is more specific than a bucket policy
- **C.** The delete is denied because an explicit deny in any applicable policy overrides allows
- **D.** The delete is denied because identity policies can't grant delete permissions

<details>
<summary>Reveal answer</summary>

**✅ C.** Explicit deny always wins, regardless of which policy type contains it or the order of evaluation.

🔁 **Review:** [Lab 1A, Step 8](lab-1a-policy-evaluation-logic.md#step-8-round-3--the-explicit-deny)

</details>

---

### Question 3

Account A has an S3 bucket. A role in Account B needs to read from it. Account A's bucket policy grants the Account B role `s3:GetObject`. Requests still fail with Access Denied. **What is the MOST likely cause?**

- **A.** S3 doesn't support cross-account access
- **B.** The role in Account B doesn't have an identity policy that allows `s3:GetObject` on the bucket
- **C.** The bucket must be made public for cross-account access
- **D.** Account B must be in the same AWS Region as Account A

<details>
<summary>Reveal answer</summary>

**✅ B.** For **cross-account** access, **both** sides must allow: the resource policy in Account A **and** the identity policy in Account B. In the same account, either one alone is enough. That's the difference you saw in Lab 1A.

🔁 **Review:** [Lab 1A, Concepts](lab-1a-policy-evaluation-logic.md#concepts) and [Step 7 reveal](lab-1a-policy-evaluation-logic.md#step-7-round-2--add-a-resource-based-policy)

</details>

---

### Question 4

A company uses a third-party SaaS monitoring tool that needs read access to its AWS account. The vendor serves thousands of customers from a single AWS account. **Which approach follows AWS best practices?**

- **A.** Create an IAM user for the vendor with `ReadOnlyAccess` and share the access keys over a secure channel
- **B.** Create an IAM role that trusts the vendor's AWS account, requires an External ID in the trust policy, and has `ReadOnlyAccess`
- **C.** Share the root user's credentials with MFA enabled
- **D.** Create an IAM role that trusts `"Principal": "*"` with `ReadOnlyAccess`

<details>
<summary>Reveal answer</summary>

**✅ B.** A cross-account role plus an **External ID** prevents the **confused deputy** problem, where another of the vendor's customers tricks the vendor into accessing your account.

- A uses long-lived keys. C is never acceptable. D lets *anyone in any AWS account* assume the role.

🔁 **Review:** [Lab 1B, Part 1](lab-1b-trust-policies-boundaries.md#part-1--external-id-third-party-access)

</details>

---

### Question 5

A team lead wants developers to create IAM roles for their own Lambda functions, but developers must **never** be able to create a role with more permissions than they have themselves. **Which solution meets this requirement?**

- **A.** Attach `IAMFullAccess` to developers and review CloudTrail weekly
- **B.** Use a permissions boundary, and allow `iam:CreateRole` only when the `iam:PermissionsBoundary` condition key equals that boundary
- **C.** Use an SCP that denies `iam:CreateRole` in the account
- **D.** Require MFA for `iam:CreateRole`

<details>
<summary>Reveal answer</summary>

**✅ B.** This is the textbook **delegated administration** pattern. Any role a developer creates is capped by the boundary.

- A only *detects* escalation after it happens. C blocks the requirement entirely. D adds authentication strength but no limit on permissions.

🔁 **Review:** [Lab 1B, Step 7](lab-1b-trust-policies-boundaries.md#step-7-delegate-role-creation-safely)

</details>

---

### Question 6

A role has `AmazonS3FullAccess` attached **and** a permissions boundary that allows only `s3:GetObject` and `s3:ListBucket`. **Which action can the role perform?**

- **A.** `s3:PutObject`
- **B.** `s3:DeleteBucket`
- **C.** `s3:GetObject`
- **D.** `ec2:DescribeInstances`

<details>
<summary>Reveal answer</summary>

**✅ C.** Effective permissions = identity policy **∩** boundary. Only `GetObject` and `ListBucket` are in both. D isn't in either policy.

🔁 **Review:** [Lab 1B, Step 6](lab-1b-trust-policies-boundaries.md#step-6-create-the-developer-role-generous-identity--boundary)

</details>

---

### Question 7

A company with 40 AWS accounts in AWS Organizations must ensure that **no one in any member account, including root users**, can disable AWS CloudTrail. **What is the solution with the LEAST operational overhead?**

- **A.** Deploy an IAM policy denying `cloudtrail:StopLogging` to every IAM role in every account
- **B.** Create an EventBridge rule in each account that re-enables CloudTrail when it's stopped
- **C.** Attach an SCP that denies `cloudtrail:StopLogging` and `cloudtrail:DeleteTrail` to the root of the organization
- **D.** Enable MFA Delete on the CloudTrail S3 bucket

<details>
<summary>Reveal answer</summary>

**✅ C.** SCPs apply to every principal in member accounts, **including the root user**, and are managed centrally.

- A: IAM policies can't restrict the root user and would need upkeep in 40 accounts.
- B: fixes the problem *after* the fact, with gaps.
- D: protects log *objects*, not the trail itself.

🔁 **Review:** [Lab 1C, Step 5 (SCP 2)](lab-1c-scp-guardrails.md#step-5-write-three-scps)

</details>

---

### Question 8

An SCP that denies all actions outside `us-east-1` is attached to the **root** of an organization. An administrator signed in to the **management account** launches an EC2 instance in `eu-west-1`. **What happens?**

- **A.** The launch is denied by the SCP
- **B.** The launch succeeds because SCPs don't affect the management account
- **C.** The launch succeeds only if the administrator uses MFA
- **D.** The launch is denied unless the SCP lists `eu-west-1` under `NotAction`

<details>
<summary>Reveal answer</summary>

**✅ B.** SCPs never restrict the management account. That's why best practice is to keep workloads **out of** the management account.

🔁 **Review:** [Lab 1C, Step 8](lab-1c-scp-guardrails.md#step-8-prove-the-management-account-is-unaffected)

</details>

---

### Question 9 *(Select TWO)*

An S3 bucket policy must allow access only to principals from the company's AWS Organization and must reject any request not sent over HTTPS. **Which two condition keys should the policy use?**

- **A.** `aws:PrincipalOrgID`
- **B.** `aws:SourceVpc`
- **C.** `aws:SecureTransport`
- **D.** `aws:RequestedRegion`
- **E.** `s3:x-amz-acl`

<details>
<summary>Reveal answer</summary>

**✅ A and C.** `aws:PrincipalOrgID` matches every account in the org without listing account IDs. `aws:SecureTransport = false` in a Deny statement blocks plain HTTP.

🔁 **Review:** [Lab 1A, Exam Corner](lab-1a-policy-evaluation-logic.md#-exam-corner). You'll build the TLS policy in Lab 3B.

</details>

---

### Question 10

A company wants to give 2,000 employees access to multiple AWS accounts using their existing corporate identity provider (Microsoft Entra ID), with **centralized** permission management. **Which service should a solutions architect recommend?**

- **A.** Create an IAM user for each employee in each account
- **B.** AWS IAM Identity Center connected to the external identity provider, with permission sets assigned to accounts
- **C.** Amazon Cognito user pools
- **D.** AWS Directory Service Simple AD

<details>
<summary>Reveal answer</summary>

**✅ B.** IAM Identity Center is the workforce single sign-on (SSO) service for multi-account organizations. **Permission sets** become roles in each account automatically.

- C, Cognito, is for **customer/app users**, not the workforce.
- A doesn't scale.
- D doesn't federate Entra ID.

🔁 **Review:** [Lab 1B, Exam Corner](lab-1b-trust-policies-boundaries.md#-exam-corner)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | Move on to Session 2 |
| **8–9** | ✅ Pass | +200 | Log your misses, then move on |
| **6–7** | 🔁 Almost | +0 | Redo the 🔁 Review steps, retake in 48 hours (+150 XP when you pass) |
| **≤ 5** | 📚 Rebuild | +0 | Redo Lab 1A end to end, then retake |

### 🗂️ Flashcard Round (Optional, +5 XP each)

Cover the right column and say the answer out loud:

| Prompt | Answer |
|--------|--------|
| Policy type with a `Principal` element | Resource-based policy |
| The only thing that beats an Allow | An explicit Deny |
| Effective permissions with a boundary | Identity ∩ Boundary |
| Stops the confused deputy problem | External ID |
| Restricts a member account's root user | SCP |
| Accounts SCPs don't affect | The management account |
| Workforce SSO across accounts | IAM Identity Center |
| Customer sign-in for mobile/web apps | Amazon Cognito |

---

**Next:** [Session 2 — VPC Networking, Lab 2A →](../session-02-vpc-networking/lab-2a-build-a-vpc.md)
