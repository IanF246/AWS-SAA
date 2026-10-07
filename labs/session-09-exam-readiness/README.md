# 🏆 Session 9: Exam Readiness Capstone

**Track:** SAA Quest finale  
**Covers:** All four SAA-C03 domains  
**Format:** No new AWS resources. Design practice, a timed mock exam and an exam-day game plan.

---

## What This Session Is

Sessions 1–8 gave you hands-on instincts. Session 9 turns those instincts into **exam performance**: reading long scenarios quickly, spotting the one requirement that decides the answer, and managing 130 minutes.

| Part | File | Time | XP |
|------|------|------|----|
| 1 | [🧠 Keyword → Service Cheat Sheet](keyword-cheat-sheet.md) | 30 min (and the night before the exam) | +100 for completing the self-quiz |
| 2 | [🏗️ Design Scenarios](design-scenarios.md): 5 whiteboard challenges | 90–120 min | Up to +250 each |
| 3 | [📝 30-Question Mock Exam](mock-exam.md) | 60 min, timed | +500 for ≥ 75%, +250 bonus for ≥ 90% |
| 4 | Exam-day plan (below) | 15 min | +50 |

---

## ✅ Readiness Checklist

Tick each box honestly. **Book the exam when every box is ticked.**

**Hands-on**
- [ ] All 24 labs completed, including cleanup
- [ ] At least 5 of 8 ⚔️ Boss Challenges beaten

**Knowledge checks**
- [ ] Every mini exam passed at ≥ 80%
- [ ] Mini exams 1–4 retaken **cold** (two or more weeks later) at ≥ 80%
- [ ] Mock exam (Part 3) at ≥ 75%
- [ ] **One full-length official practice exam** (AWS Skill Builder) at ≥ 75%

**Weak spots**
- [ ] Every item in the weak-spot log ([README](../../README.md#-progress-tracker)) reviewed and marked fixed

**Logistics**
- [ ] Exam booked (Pearson VUE: test center or online proctored)
- [ ] For online exams: system test passed, clean desk, government ID ready
- [ ] Non-native English speakers: request the **+30 minute ESL accommodation** *before* booking

---

## 🎯 Exam-Day Game Plan

### The 3-Pass Strategy (130 minutes, 65 questions)

| Pass | Time Budget | What to Do |
|------|-------------|-----------|
| **1. Sweep** | ~75 min | Answer everything you're confident about in **≤ 90 seconds**. Anything harder: make your best guess, **flag** it, move on. **Never leave a blank** (there's no penalty for guessing). |
| **2. Flags** | ~40 min | Return to flagged questions with fresh eyes. Eliminate, then decide. |
| **3. Sanity** | ~15 min | Re-read questions with words like **NOT**, **LEAST** and **MOST**, and "select TWO" questions, to check you chose the right *number* of answers. |

### Reading a Scenario (The 4-Step Decoder)

1. **Read the LAST sentence first.** It names the optimization target: *most cost-effective*, *least operational overhead*, *most secure*, *highest availability*, *lowest latency*.
2. **Underline the hard constraints** in the scenario: *must not be interrupted*, *within milliseconds*, *on-premises*, *Windows*, *existing Active Directory*, *without code changes*, *in another Region*.
3. **Eliminate the impossible** (gateway endpoint for SQS, SG deny rule, SCP on the management account).
4. **Choose by the optimization target** among what's left. For "least operational overhead," prefer **managed / serverless** options.

### The Qualifier Translator

| The Question Says | It Usually Means Pick… |
|-------------------|------------------------|
| "LEAST operational overhead" | Managed / serverless / native feature over custom scripts |
| "MOST cost-effective" | The cheapest option that still meets **every** requirement |
| "MOST secure" | Least privilege, encryption, private connectivity, no long-lived keys |
| "Highly available" | Multi-AZ, auto-replacement, no single point of failure |
| "Fault tolerant" | Keeps working **during** a failure, with no degradation (more capacity than HA) |
| "Decouple" | SQS / SNS / EventBridge |
| "Minimal code changes" | Drop-in options (DAX, RDS Proxy, Amazon MQ, lift-and-shift) |
| "Near real-time" | Kinesis / streams, not batch |

---

## 🗡️ Final Boss: The 10-Minute Whiteboard

When you're done with Parts 1–3, set a 10-minute timer and, **from memory**, draw a three-tier web application that is:
- Multi-AZ, auto-scaling, behind an ALB, with CloudFront in front
- With private subnets, NAT per AZ, and gateway endpoints for S3
- Using Aurora Multi-AZ with a read replica and Secrets Manager credentials
- Decoupled order processing with SQS + a DLQ
- With encryption everywhere (KMS), WAF on CloudFront, and CloudTrail + GuardDuty on
- With a DR copy in a second Region (pilot light)

Then label **which session taught each piece**. If you can do it, you're ready. **+300 XP** and the 🏆 **Exam Ready** rank.

---

## After You Pass 🎉

- Add the badge to LinkedIn (Credly).
- Natural next steps: **AWS Certified Developer – Associate**, **SysOps Administrator – Associate**, **Security – Specialty**, or the **Solutions Architect – Professional**.
- Keep building: the [AI Cloud Fusion capstone](https://github.com/IanF246/AICloudFusion/blob/main/labs/session-13-capstone/README.md) is a great place to apply SAA design skills to a real project.
