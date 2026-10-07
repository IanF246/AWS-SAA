# 📝 Mini Exam 3: Data Protection

**Covers:** Labs 3A, 3B, 3C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: spot the qualifier.** Encryption questions hinge on one phrase: *"least operational overhead"* (SSE-S3), *"audit key usage"* (SSE-KMS), *"company manages its own keys"* (SSE-C or client-side), *"dedicated hardware"* (CloudHSM). Underline it before you look at the options.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

A company must encrypt objects in S3 and needs an **audit trail showing who used the encryption key and when**. The company wants AWS to manage the key's storage and durability. **Which option meets these requirements?**

- **A.** SSE-S3
- **B.** SSE-KMS with a customer managed key
- **C.** SSE-C
- **D.** Client-side encryption with a key stored on an EC2 instance

<details>
<summary>Reveal answer</summary>

**✅ B.** Every use of a KMS key is logged in **CloudTrail**, and a customer managed key gives you a key policy you control. SSE-S3 provides no key-usage audit. SSE-C means *you* store and manage the keys.

🔁 **Review:** [Lab 3A, Step 9](lab-3a-kms-envelope-encryption.md#step-9-the-lock-out-demo)

</details>

---

### Question 2

An application uploads millions of small objects per hour to a bucket using SSE-KMS. The team sees **KMS `ThrottlingException` errors** and a large KMS bill. **What is the MOST operationally efficient fix?**

- **A.** Request a KMS quota increase and keep the design
- **B.** Enable S3 Bucket Keys on the bucket
- **C.** Switch to SSE-C
- **D.** Encrypt objects client-side with a hard-coded key

<details>
<summary>Reveal answer</summary>

**✅ B.** S3 Bucket Keys reduce KMS calls by **up to 99%** while keeping SSE-KMS's controls.

🔁 **Review:** [Lab 3A, Step 8](lab-3a-kms-envelope-encryption.md#step-8-default-sse-kms-encryption-on-s3-with-a-bucket-key)

</details>

---

### Question 3

A company has an **unencrypted** Amazon RDS for PostgreSQL DB instance and must encrypt it at rest. **How?**

- **A.** Modify the DB instance and enable encryption
- **B.** Take a snapshot, copy the snapshot with encryption enabled, and restore a new DB instance from the encrypted copy
- **C.** Enable SSE-KMS on the RDS storage volume
- **D.** Enable encryption on the read replica, then promote it

<details>
<summary>Reveal answer</summary>

**✅ B.** You can't enable encryption on an existing RDS instance, and a read replica of an unencrypted DB is also unencrypted (D). The **snapshot → encrypted copy → restore** path is the standard answer, and it's the same for EBS volumes.

🔁 **Review:** [Lab 3A, Exam Corner](lab-3a-kms-envelope-encryption.md#-exam-corner)

</details>

---

### Question 4

Security policy requires that **all requests to an S3 bucket use HTTPS**. **What should a solutions architect do?**

- **A.** Enable default encryption with SSE-S3
- **B.** Add a bucket policy that denies all actions when `aws:SecureTransport` is `false`
- **C.** Enable S3 Block Public Access
- **D.** Enable S3 Transfer Acceleration

<details>
<summary>Reveal answer</summary>

**✅ B.** That's encryption **in transit**. A is encryption **at rest**, a common mix-up the exam relies on.

🔁 **Review:** [Lab 3B, Part 1](lab-3b-s3-lockdown.md#part-1--enforce-tls-and-encryption-with-a-bucket-policy)

</details>

---

### Question 5

A media company wants to let **customers download purchased videos** from a private S3 bucket. Access must expire after **1 hour**, and the bucket must stay private. **What is the simplest solution?**

- **A.** Make the objects public and rotate their names hourly
- **B.** Generate presigned URLs with a 1-hour expiry from the application backend
- **C.** Create an IAM user for each customer
- **D.** Use S3 Object Lock with a 1-hour retention

<details>
<summary>Reveal answer</summary>

**✅ B.** Presigned URLs grant time-limited access to a specific object using the signer's permissions. (At scale, with CloudFront in front, **CloudFront signed URLs/cookies** are the equivalent.)

🔁 **Review:** [Lab 3B, Part 2](lab-3b-s3-lockdown.md#part-2--presigned-urls)

</details>

---

### Question 6

A financial firm must store trade records so that **no user, including the root user, can delete or modify them for 7 years**, to meet SEC regulations. **What should it use?**

- **A.** S3 Versioning with MFA Delete
- **B.** S3 Object Lock in governance mode
- **C.** S3 Object Lock in compliance mode with a 7-year retention period
- **D.** An S3 Lifecycle rule to Glacier Deep Archive

<details>
<summary>Reveal answer</summary>

**✅ C.** **Compliance mode** can't be bypassed by anyone. Governance mode (B) can be bypassed with `s3:BypassGovernanceRetention`, and MFA Delete (A) still allows deletion by the root user with MFA.

🔁 **Review:** [Lab 3B, Step 8 checkpoint](lab-3b-s3-lockdown.md#step-8-play-the-ransomware-attacker)

</details>

---

### Question 7

A user deletes an object in a **versioning-enabled** bucket with a simple `DELETE` request (no version ID). **What happens?**

- **A.** All versions of the object are permanently deleted
- **B.** S3 inserts a delete marker; previous versions remain and can be restored
- **C.** The request fails because versioned objects can't be deleted
- **D.** Only the oldest version is deleted

<details>
<summary>Reveal answer</summary>

**✅ B.** Delete the delete marker and the object is back.

🔁 **Review:** [Lab 3B, Steps 5–6](lab-3b-s3-lockdown.md#step-5-enable-versioning-and-make-mistakes)

</details>

---

### Question 8

An application on EC2 uses an **Amazon Aurora** database. The credentials must be **rotated automatically every 30 days** with minimal operational effort. **Which solution meets this requirement?**

- **A.** Store the credentials in Parameter Store as a SecureString and rotate them with a cron job on the instance
- **B.** Store the credentials in AWS Secrets Manager and enable automatic rotation
- **C.** Store the credentials in an encrypted S3 object and update it monthly
- **D.** Hard-code the credentials and redeploy monthly

<details>
<summary>Reveal answer</summary>

**✅ B.** "Rotate automatically" + database = **Secrets Manager**. It has native rotation for Aurora/RDS. Parameter Store has no built-in rotation.

🔁 **Review:** [Lab 3C, Concepts](lab-3c-secrets-and-parameters.md#concepts)

</details>

---

### Question 9

A Lambda function reads a SecureString parameter encrypted with a **customer managed KMS key**. The role has `ssm:GetParameter` on the parameter, but invocations fail with **AccessDeniedException**. **What is missing?**

- **A.** `ssm:DescribeParameters` permission
- **B.** `kms:Decrypt` permission on the customer managed key (in the role's policy and/or key policy)
- **C.** A VPC endpoint for KMS
- **D.** The `WithDecryption` flag must be set to `false`

<details>
<summary>Reveal answer</summary>

**✅ B.** AWS managed keys implicitly allow use through the service. **Customer managed keys require explicit `kms:Decrypt`** for the caller.

🔁 **Review:** [Lab 3C, Step 5](lab-3c-secrets-and-parameters.md#step-5-create-a-least-privilege-role-for-lambda)

</details>

---

### Question 10 *(Select TWO)*

A company needs to automatically **discover and classify PII** stored across hundreds of S3 buckets and **prevent any bucket from becoming public**. **Which TWO services or features should be used?**

- **A.** Amazon Macie
- **B.** Amazon GuardDuty
- **C.** S3 Block Public Access at the account level
- **D.** Amazon Inspector
- **E.** AWS Shield

<details>
<summary>Reveal answer</summary>

**✅ A and C.** **Macie** discovers sensitive data in S3. **Account-level Block Public Access** stops public buckets. GuardDuty is threat detection, Inspector is vulnerability scanning for EC2/ECR/Lambda, and Shield is DDoS protection.

🔁 **Review:** [Lab 3B, Exam Corner](lab-3b-s3-lockdown.md#-exam-corner)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | 🎉 **Domain 1 (30% of the exam) complete!** |
| **8–9** | ✅ Pass | +200 | Log misses, move on |
| **6–7** | 🔁 Almost | +0 | Redo 🔁 links, retake in 48 h |
| **≤ 5** | 📚 Rebuild | +0 | Redo Lab 3A Steps 8–9 and Lab 3B Part 1 |

### 🎲 Domain 1 Lightning Round: Which Service? (+5 XP each)

| Need | Service |
|------|---------|
| DDoS protection (L3/L4), always on, free | Shield Standard |
| Block SQL injection / XSS at the ALB or CloudFront | AWS WAF |
| Threat detection from CloudTrail, VPC Flow Logs, DNS logs | GuardDuty |
| Vulnerability scanning of EC2 & container images | Inspector |
| Find PII in S3 | Macie |
| Aggregate security findings across accounts | Security Hub |
| Track resource configuration changes & compliance rules | AWS Config |
| Who made this API call? | CloudTrail |

---

**Next:** [Session 4 — Resilient Compute, Lab 4A →](../session-04-resilient-compute/lab-4a-launch-template-asg.md)
