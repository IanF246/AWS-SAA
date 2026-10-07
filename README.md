# SAA Quest: Hands-On Labs for the AWS Certified Solutions Architect – Associate

A self-paced, hands-on lab track that gets you ready for the **AWS Certified Solutions Architect – Associate (SAA-C03)** exam. You build every architecture the exam asks you to reason about, break it on purpose, fix it, and then prove what you learned with a mini exam at the end of each session.

It uses the same format as the [AI Cloud Fusion labs](https://github.com/IanF246/AICloudFusion): three labs per session (A → B → C), copy-and-paste commands, console checkpoints and a cleanup at the end of every lab. Two things are added for exam prep: **mini exams** and **game mechanics**, so studying feels like progress and not just reading.

---

## 🎯 The Exam at a Glance

| Item | Detail |
|------|--------|
| Exam code | SAA-C03 |
| Questions | 65 (50 scored + 15 unscored, and you can't tell which are which) |
| Question types | Multiple choice (1 correct of 4) and multiple response (2+ correct of 5+) |
| Time | 130 minutes |
| Passing score | 720 / 1000 (compensatory: you don't need to pass each domain on its own) |
| Price | $150 USD |

| Domain | Weight | Sessions That Cover It |
|--------|--------|------------------------|
| **1. Design Secure Architectures** | 30% | 1, 2, 3 |
| **2. Design Resilient Architectures** | 26% | 4, 5, 6 |
| **3. Design High-Performing Architectures** | 24% | 6, 7 |
| **4. Design Cost-Optimized Architectures** | 20% | 7, 8 |

> ⚠️ Always confirm the current exam version, price and domain weights on the [official SAA-C03 exam guide](https://docs.aws.amazon.com/aws-certification/latest/examguides/solutions-architect-associate-03.html) before you book. AWS revises exams from time to time.

---

## 🕹️ How This Track Works

### The session loop

```
 ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────────┐
 │  Lab A   │ →  │  Lab B   │ →  │  Lab C   │ →  │ ⚔️ Boss       │ →  │ 📝 Mini Exam  │
 │ Beginner │    │ Intermed.│    │ Advanced │    │ (optional)   │    │ score ≥ 80%  │
 └──────────┘    └──────────┘    └──────────┘    └──────────────┘    └──────────────┘
                                                                           │
                                         below 80%? → revisit the linked lab steps ↩
```

### What you'll find inside every lab

| Icon | Meaning |
|------|---------|
| 📋 | Copy-and-paste command |
| 🔄 | Placeholder you need to replace, like `<YOUR_PROFILE_NAME>` |
| 🔮 | **Predict first.** Guess the result *before* you run the command, then check yourself. This is how lab work turns into exam instincts. |
| 🧩 | **Checkpoint question.** A quick question in the middle of a lab, with the answer hidden in a fold-out. |
| ✅ | Console checkpoint: confirm what you built, visually |
| 💡 | Why this matters: the concept behind the step |
| 🚨 | **Exam trap.** A wrong answer the exam *wants* you to pick |
| ⚔️ | **Boss Challenge.** An optional build with no copy-paste help, just requirements |
| 🧹 | Cleanup, so you don't pay for anything after you finish |

### Earning XP

Keep score in the tracker below. XP doesn't mean anything to AWS, but it makes your progress visible.

| Action | XP |
|--------|----|
| Complete a lab, including cleanup | **+100** |
| Get a 🔮 prediction right before running the command | **+10** each |
| Get a 🧩 checkpoint right on the first try | **+10** each |
| Beat a ⚔️ Boss Challenge | **+250** |
| Score ≥ 80% on a mini exam | **+200** |
| Score 100% on a mini exam | **+100 bonus** |
| Retake a mini exam you failed and pass it | **+150** (redemption arc) |

**Ranks:** 🥉 Cloud Cadet (0) → 🥈 VPC Voyager (1,500) → 🥇 Resilience Ranger (3,500) → 💎 Well-Architected Warrior (6,000) → 🏆 **Exam Ready** (all mini exams ≥ 80% + Session 9 mock exam ≥ 75%)

---

## 🗺️ Session Labs

| # | Session | Domain | Labs | Mini Exam |
|---|---------|--------|------|-----------|
| 1 | Identity & Access Control | Secure | [1A: Policy Evaluation Logic](labs/session-01-identity-access/lab-1a-policy-evaluation-logic.md) · [1B: Trust Policies, External ID & Boundaries](labs/session-01-identity-access/lab-1b-trust-policies-boundaries.md) · [1C: Multi-Account Guardrails with SCPs](labs/session-01-identity-access/lab-1c-scp-guardrails.md) | [📝 Exam 1](labs/session-01-identity-access/mini-exam-01.md) |
| 2 | VPC Networking | Secure | [2A: Build a VPC from Scratch](labs/session-02-vpc-networking/lab-2a-build-a-vpc.md) · [2B: Private Subnets, Endpoints, SGs & NACLs](labs/session-02-vpc-networking/lab-2b-private-subnet-endpoints.md) · [2C: VPC Peering & Flow Logs](labs/session-02-vpc-networking/lab-2c-peering-flow-logs.md) | [📝 Exam 2](labs/session-02-vpc-networking/mini-exam-02.md) |
| 3 | Data Protection | Secure | [3A: KMS & Envelope Encryption](labs/session-03-data-protection/lab-3a-kms-envelope-encryption.md) · [3B: Locking Down S3](labs/session-03-data-protection/lab-3b-s3-lockdown.md) · [3C: Secrets Manager vs Parameter Store](labs/session-03-data-protection/lab-3c-secrets-and-parameters.md) | [📝 Exam 3](labs/session-03-data-protection/mini-exam-03.md) |
| 4 | Resilient Compute | Resilient | [4A: Launch Templates & Self-Healing ASGs](labs/session-04-resilient-compute/lab-4a-launch-template-asg.md) · [4B: Application Load Balancer](labs/session-04-resilient-compute/lab-4b-application-load-balancer.md) · [4C: Auto Scaling Under Load](labs/session-04-resilient-compute/lab-4c-scaling-under-load.md) | [📝 Exam 4](labs/session-04-resilient-compute/mini-exam-04.md) |
| 5 | Decoupled Architectures | Resilient | [5A: SQS Queues, DLQs & FIFO](labs/session-05-decoupling/lab-5a-sqs-queues-dlq-fifo.md) · [5B: SNS Fan-Out with Filtering](labs/session-05-decoupling/lab-5b-sns-fanout-filtering.md) · [5C: EventBridge + Step Functions](labs/session-05-decoupling/lab-5c-eventbridge-step-functions.md) | [📝 Exam 5](labs/session-05-decoupling/mini-exam-05.md) |
| 6 | Databases & Disaster Recovery | Resilient / Perf. | [6A: RDS Multi-AZ vs Read Replicas](labs/session-06-databases-dr/lab-6a-rds-multi-az-read-replica.md) · [6B: DynamoDB Design Patterns](labs/session-06-databases-dr/lab-6b-dynamodb-design.md) · [6C: Backup & DR Strategies](labs/session-06-databases-dr/lab-6c-backup-dr-strategies.md) | [📝 Exam 6](labs/session-06-databases-dr/mini-exam-06.md) |
| 7 | Storage & DNS Performance | Perf. / Cost | [7A: S3 Storage Classes & Lifecycle](labs/session-07-storage-dns/lab-7a-s3-storage-classes-lifecycle.md) · [7B: EBS vs EFS vs Instance Store](labs/session-07-storage-dns/lab-7b-ebs-vs-efs.md) · [7C: Route 53 Routing Policies](labs/session-07-storage-dns/lab-7c-route53-routing-policies.md) | [📝 Exam 7](labs/session-07-storage-dns/mini-exam-07.md) |
| 8 | Cost Optimization | Cost | [8A: Cost Visibility & Anomaly Detection](labs/session-08-cost-optimization/lab-8a-cost-visibility.md) · [8B: Spot, Savings Plans & Purchasing Options](labs/session-08-cost-optimization/lab-8b-purchasing-options-spot.md) · [8C: Architecture Cost Showdown](labs/session-08-cost-optimization/lab-8c-architecture-cost-showdown.md) | [📝 Exam 8](labs/session-08-cost-optimization/mini-exam-08.md) |
| 9 | 🏆 Exam Readiness Capstone | All | [Capstone Guide](labs/session-09-exam-readiness/README.md) · [Design Scenarios](labs/session-09-exam-readiness/design-scenarios.md) · [Keyword → Service Cheat Sheet](labs/session-09-exam-readiness/keyword-cheat-sheet.md) | [📝 30-Question Mock Exam](labs/session-09-exam-readiness/mock-exam.md) |

Keep the [Glossary](labs/GLOSSARY.md) open in another tab.

---

## 📈 Progress Tracker

Tick the boxes as you go (`[ ]` → `[x]`). On GitHub and in VS Code's Markdown preview, they render as checkboxes.

| Session | Lab A | Lab B | Lab C | ⚔️ Boss | Mini Exam Score | XP Earned |
|---------|-------|-------|-------|---------|-----------------|-----------|
| 1 Identity | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 2 VPC | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 3 Data Protection | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 4 Resilient Compute | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 5 Decoupling | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 6 Databases & DR | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 7 Storage & DNS | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 8 Cost | [ ] | [ ] | [ ] | [ ] | __ / 10 | |
| 9 Mock Exam | — | — | — | — | __ / 30 | |
| **Total** | | | | | | **____ XP** |

**Weak-spot log.** Every time you miss a mini-exam question, write the topic here. Read this list again the night before the exam.

| Date | Topic I missed | Why I got it wrong | Fixed? |
|------|----------------|--------------------|--------|
| | | | [ ] |
| | | | [ ] |
| | | | [ ] |

---

## 📅 Suggested Study Plan (8 weeks, ~6 hrs/week)

| Week | Do This |
|------|---------|
| 1 | Session 1 + Mini Exam 1 |
| 2 | Session 2 + Mini Exam 2 |
| 3 | Session 3 + Mini Exam 3 · re-take Exam 1 cold (spaced repetition) |
| 4 | Session 4 + Session 5 |
| 5 | Session 6 + re-take Exams 2 & 3 cold |
| 6 | Session 7 + Session 8 |
| 7 | Session 9: design scenarios + mock exam · go through your weak-spot log |
| 8 | A full-length practice test (AWS Skill Builder official practice exam) · book the exam · 🏆 |

> 💡 **Spaced repetition:** Retake each mini exam **cold** about two weeks after you first passed it. If you score lower the second time, that topic belongs in your weak-spot log.

---

## Prerequisites

- An AWS account with the AWS CLI v2 configured using IAM Identity Center (SSO). If you haven't done this yet, follow [AI Cloud Fusion Lab 1A](https://github.com/IanF246/AICloudFusion/blob/main/labs/session-01-cloud-concepts/lab-1a-aws-cli-setup.md) first.
- An **AWS Budget alert**, so you hear about it if something gets left running. See [AI Cloud Fusion Lab 1B](https://github.com/IanF246/AICloudFusion/blob/main/labs/session-01-cloud-concepts/lab-1b-cost-budget-sns-alert.md).
- VS Code with the `code` command on your PATH
- Comfort with the basics: what EC2, S3, IAM and Lambda are. AI Cloud Fusion Sessions 1–3 cover them.

All labs use **us-east-1** unless the lab says otherwise.

## Cost

Most labs cost **$0.00–$0.10** if you clean up right away. The few that cost more (load balancers, RDS Multi-AZ, NAT-style networking) say so clearly in their Cost Notice and are designed to finish within about an hour. **Expected total for the whole track: under $3.** That assumes you finish every 🧹 Cleanup section, so always do it.

## Ground Rules

1. **Never skip Cleanup.** It's the difference between $0.05 and $50.
2. **Do the 🔮 predictions honestly.** Write your guess down before you run the command. Getting one wrong in the lab costs nothing. Getting it wrong on the exam costs points.
3. **Use the mini exams to diagnose, not to judge yourself.** Every wrong answer links back to the exact lab step that teaches the concept.
