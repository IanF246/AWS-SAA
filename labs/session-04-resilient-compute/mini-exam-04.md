# 📝 Mini Exam 4: Resilient Compute

**Covers:** Labs 4A, 4B, 4C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: the resilience checklist.** For any "highly available" question, run through: **Multi-AZ?** **Health checks that match the failure?** **Automatic replacement?** **No single point of failure** (one NAT, one instance, one AZ)? The right answer usually fixes the one item missing from the scenario.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

A web application runs on **four EC2 instances in a single Availability Zone** behind an ALB. **What change MOST improves availability?**

- **A.** Increase the instance size
- **B.** Configure an Auto Scaling group that spans at least two AZs, and enable the ALB in those AZs
- **C.** Add two more instances in the same AZ
- **D.** Enable detailed monitoring

<details>
<summary>Reveal answer</summary>

**✅ B.** The single AZ is the single point of failure. More or bigger instances in the same AZ don't survive an AZ outage.

🔁 **Review:** [Lab 4A, Step 6](lab-4a-launch-template-asg.md#step-6-create-the-auto-scaling-group)

</details>

---

### Question 2

An ASG behind an ALB uses default settings. A bad deployment causes the application to return HTTP 500 errors, but the instances' **EC2 status checks pass**. The ASG doesn't replace them. **What should be changed?**

- **A.** Enable detailed monitoring on the instances
- **B.** Change the ASG health check type to ELB
- **C.** Increase the health check grace period
- **D.** Use a Network Load Balancer

<details>
<summary>Reveal answer</summary>

**✅ B.** The default **EC2** health check only sees infrastructure failures. **ELB** health checks test the app endpoint, and the ASG then replaces failing instances.

🔁 **Review:** [Lab 4B, Step 7](lab-4b-application-load-balancer.md#step-7-the-silent-failure-why-elb-health-checks-matter)

</details>

---

### Question 3

Web servers behind an ALB must accept HTTP traffic **only from the load balancer**. Instances are replaced frequently by Auto Scaling. **What is the BEST security group configuration for the instances?**

- **A.** Allow port 80 from the ALB's private IP addresses
- **B.** Allow port 80 from the ALB's security group ID
- **C.** Allow port 80 from 0.0.0.0/0 and rely on the NACL
- **D.** Allow port 80 from the VPC CIDR

<details>
<summary>Reveal answer</summary>

**✅ B.** Referencing the **security group** stays correct no matter how IPs change. ALB node IPs change too, which rules out A. D is broader than needed.

🔁 **Review:** [Lab 4B, Step 6](lab-4b-application-load-balancer.md#step-6-lock-down-the-instances-sg-chaining)

</details>

---

### Question 4

A company's partners must allow-list the **IP addresses** of a public TCP application in their firewalls. The application must be highly available across AZs. **Which load balancer should be used?**

- **A.** Application Load Balancer
- **B.** Network Load Balancer with an Elastic IP in each AZ
- **C.** Classic Load Balancer
- **D.** Gateway Load Balancer

<details>
<summary>Reveal answer</summary>

**✅ B.** NLBs support **one static/Elastic IP per AZ**. ALBs don't have static IPs. (AWS Global Accelerator in front of an ALB is another valid "static IP" pattern.)

🔁 **Review:** [Lab 4B, Concepts](lab-4b-application-load-balancer.md#concepts)

</details>

---

### Question 5

An application needs to send requests for `/api/*` to one group of instances and `/images/*` to another, through **one** load balancer. **Which solution meets this?**

- **A.** NLB with two listeners
- **B.** ALB with path-based routing rules to two target groups
- **C.** Two Route 53 records with weighted routing
- **D.** Gateway Load Balancer

<details>
<summary>Reveal answer</summary>

**✅ B.** Path-based (and host- or header-based) routing is a **Layer 7** ALB feature.

🔁 **Review:** [Lab 4B, Exam Corner](lab-4b-application-load-balancer.md#-exam-corner)

</details>

---

### Question 6

Traffic to an application **predictably** increases every weekday at 9:00 AM. Users experience slowness for the first 10 minutes while the ASG scales. **What is the MOST effective solution?**

- **A.** Lower the target tracking target value to 10%
- **B.** Configure a scheduled scaling action to increase capacity shortly before 9:00 AM on weekdays
- **C.** Switch to simple scaling
- **D.** Use larger instances permanently

<details>
<summary>Reveal answer</summary>

**✅ B.** **Predictable** load → **scheduled** (or predictive) scaling, so capacity is ready *before* the spike. A and D waste money all day.

🔁 **Review:** [Lab 4C, Step 5](lab-4c-scaling-under-load.md#step-5-add-a-scheduled-action-for-a-known-peak)

</details>

---

### Question 7

A team wants their ASG to scale based on **memory utilization**. They can't find a memory metric in CloudWatch. **Why?**

- **A.** Memory metrics are only available with Dedicated Hosts
- **B.** EC2 doesn't publish memory metrics by default. Install the CloudWatch agent to publish them as custom metrics.
- **C.** Memory metrics require Enhanced Networking
- **D.** They must enable detailed monitoring

<details>
<summary>Reveal answer</summary>

**✅ B.** The hypervisor can't see inside the guest OS's memory. Detailed monitoring (D) only makes the **default** metrics (CPU, network, disk ops) arrive every minute instead of every 5.

🔁 **Review:** [Lab 4C, Exam Corner](lab-4c-scaling-under-load.md#-exam-corner)

</details>

---

### Question 8

A company must deploy a new AMI to a 20-instance ASG **without downtime**, replacing instances gradually while keeping at least 90% of capacity healthy. **What should it do?**

- **A.** Delete the ASG and create a new one
- **B.** Create a new launch template version and start an instance refresh with a minimum healthy percentage of 90
- **C.** Stop all instances, change the AMI, start them again
- **D.** Update the AMI on each running instance

<details>
<summary>Reveal answer</summary>

**✅ B.** **Instance refresh** does a rolling replacement, respecting `MinHealthyPercentage`. (Blue/green with a second ASG/target group is also valid when instant rollback is needed.)

🔁 **Review:** [Lab 4B, Step 8](lab-4b-application-load-balancer.md#step-8-zero-downtime-deploy-with-an-instance-refresh)

</details>

---

### Question 9

Before an instance is terminated by scale-in, the application must **upload its local log files to S3**. **What should a solutions architect configure?**

- **A.** A CloudWatch alarm
- **B.** An Auto Scaling lifecycle hook for the terminating state
- **C.** Scale-in protection on all instances
- **D.** A longer health check grace period

<details>
<summary>Reveal answer</summary>

**✅ B.** Lifecycle hooks pause the instance in `Terminating:Wait` so custom actions can run first. C would stop scale-in altogether.

🔁 **Review:** [Lab 4C, Step 6](lab-4c-scaling-under-load.md#step-6-watch-scale-in-optional-1520-min)

</details>

---

### Question 10 *(Select TWO)*

New instances in an ASG take **8 minutes** to install software and become healthy, causing errors during sudden spikes. **Which TWO actions reduce this time?**

- **A.** Create a custom AMI with the software preinstalled
- **B.** Configure a warm pool for the ASG
- **C.** Increase the health check grace period to 15 minutes
- **D.** Switch the ASG health check type to EC2
- **E.** Reduce the ASG's maximum size

<details>
<summary>Reveal answer</summary>

**✅ A and B.** A golden AMI removes install time. A warm pool keeps pre-initialized instances ready. C only delays health checking; it doesn't make instances ready sooner.

🔁 **Review:** [Lab 4C, Step 4 checkpoint](lab-4c-scaling-under-load.md#step-4-unleash-the-load-)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | Session 5 |
| **8–9** | ✅ Pass | +200 | Log misses, move on |
| **6–7** | 🔁 Almost | +0 | Redo 🔁 links, retake in 48 h |
| **≤ 5** | 📚 Rebuild | +0 | Redo Lab 4B Step 7. It's the most-tested idea in this session. |

### 🗂️ Match-Up (+5 XP each)

Cover the right column:

| Requirement | Load Balancer / Feature |
|-------------|------------------------|
| Path-based routing | ALB |
| Static IP per AZ | NLB |
| UDP traffic | NLB |
| Inline firewall appliances | GWLB |
| Gradual AMI rollout | Instance refresh |
| Script before termination | Lifecycle hook |
| Known 9 AM peak | Scheduled scaling |
| Boot faster | Golden AMI / warm pool |

---

**Next:** [Session 5 — Decoupled Architectures, Lab 5A →](../session-05-decoupling/lab-5a-sqs-queues-dlq-fifo.md)
