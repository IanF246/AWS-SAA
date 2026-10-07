# 📝 Mini Exam 2: VPC Networking

**Covers:** Labs 2A, 2B, 2C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: eliminate first.** In networking questions, two options are usually *technically impossible* (a security group "deny" rule, a gateway endpoint for SQS). Cross those out first and you're left with a 50/50 on the trade-off.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

An EC2 instance is in a subnet whose route table has `0.0.0.0/0 → igw-123`. The instance has **no public IP** and no Elastic IP. **Can it reach the internet?**

- **A.** Yes, because the route table points to an internet gateway
- **B.** No. It needs a public or Elastic IP address as well as the route
- **C.** Yes, but only for outbound HTTPS
- **D.** No. Internet gateways only work in the default VPC

<details>
<summary>Reveal answer</summary>

**✅ B.** Direct internet access needs **both** a route to the IGW **and** a public IPv4 address. The IGW performs 1:1 NAT between the private and public IP; without a public IP there's nothing to translate to.

🔁 **Review:** [Lab 2A, Step 5](lab-2a-build-a-vpc.md#step-5-build-the-public-route-table)

</details>

---

### Question 2

A company's application runs on EC2 instances in **private subnets** and processes **20 TB per month** of data from Amazon S3 in the same Region. Today the traffic flows through a NAT gateway. **Which change reduces cost the MOST?**

- **A.** Replace the NAT gateway with a NAT instance
- **B.** Add an S3 gateway VPC endpoint and update the private route tables
- **C.** Add an S3 interface VPC endpoint
- **D.** Move the instances to public subnets

<details>
<summary>Reveal answer</summary>

**✅ B.** Gateway endpoints are **free**, with no hourly or per-GB charges. NAT gateway data processing at ~$0.045/GB × 20,000 GB ≈ **$900/month** goes away.

- C works but has hourly and per-GB charges.
- D weakens security.
- A still processes traffic and adds management overhead.

🔁 **Review:** [Lab 2B, Step 7](lab-2b-private-subnet-endpoints.md#step-7-connect-and-test)

</details>

---

### Question 3

Instances in private subnets must call **AWS Secrets Manager** without traversing the internet. **What should a solutions architect create?**

- **A.** A gateway VPC endpoint for Secrets Manager
- **B.** An interface VPC endpoint for Secrets Manager
- **C.** A NAT gateway in the private subnet
- **D.** A VPC peering connection to the Secrets Manager VPC

<details>
<summary>Reveal answer</summary>

**✅ B.** Gateway endpoints exist **only for S3 and DynamoDB**, which rules out A. Every other service uses **interface endpoints** (PrivateLink).

🔁 **Review:** [Lab 2B, Concepts](lab-2b-private-subnet-endpoints.md#concepts)

</details>

---

### Question 4

A security team needs to **block traffic from a specific IP range** (`198.51.100.0/24`) to all instances in a subnet. **What should they do?**

- **A.** Add a deny rule for the range to the instances' security group
- **B.** Add a deny rule for the range to the subnet's network ACL, with a rule number lower than any allow rules
- **C.** Remove the internet gateway
- **D.** Create a VPC endpoint policy that denies the range

<details>
<summary>Reveal answer</summary>

**✅ B.** NACLs support **explicit deny** and are evaluated **lowest rule number first**. Security groups are allow-only.

🔁 **Review:** [Lab 2B, Step 9 checkpoint](lab-2b-private-subnet-endpoints.md#step-9-test-the-stateless-firewall)

</details>

---

### Question 5

After a custom network ACL is applied, instances can send HTTPS requests to an external API but **never receive responses**. Security groups allow all outbound traffic. **What is the MOST likely cause?**

- **A.** Security groups are stateless and need an inbound rule
- **B.** The NACL lacks an inbound rule allowing ephemeral ports (1024–65535)
- **C.** The route table is missing a local route
- **D.** HTTPS isn't supported through NACLs

<details>
<summary>Reveal answer</summary>

**✅ B.** NACLs are **stateless**, so replies arrive inbound on ephemeral ports and must be explicitly allowed. A is backwards: security groups are **stateful**.

🔁 **Review:** [Lab 2B, Step 9](lab-2b-private-subnet-endpoints.md#step-9-test-the-stateless-firewall)

</details>

---

### Question 6

A NAT gateway is deployed in `public-a` and serves private subnets in **two** AZs. **What happens if us-east-1a fails, and what's the fix?**

- **A.** Nothing. NAT gateways are multi-AZ by default.
- **B.** Instances in the us-east-1b private subnet lose internet access. Deploy a NAT gateway in each AZ and route each AZ's private subnet to its local NAT gateway.
- **C.** Instances lose access. Replace the NAT gateway with an internet gateway.
- **D.** Traffic fails over automatically to the internet gateway.

<details>
<summary>Reveal answer</summary>

**✅ B.** A NAT gateway is redundant **within one AZ** only. For AZ-independent resilience, use **one NAT gateway per AZ**.

🔁 **Review:** [Lab 2A, Exam Corner](lab-2a-build-a-vpc.md#-exam-corner)

</details>

---

### Question 7

VPC A is peered with VPC B. VPC B is peered with VPC C. **Applications in VPC A cannot reach VPC C. Why?**

- **A.** The peering connections are in different AZs
- **B.** VPC peering is not transitive
- **C.** Security groups can't reference peered VPCs
- **D.** VPC C needs an internet gateway

<details>
<summary>Reveal answer</summary>

**✅ B.** Each pair of VPCs that need to talk must be **directly** peered, or connected through a **Transit Gateway**, which supports transitive routing.

🔁 **Review:** [Lab 2C, transitivity checkpoint](lab-2c-peering-flow-logs.md#-checkpoint--transitivity)

</details>

---

### Question 8

A company is adding its **30th VPC** and needs all VPCs plus an on-premises network to communicate. Management complains about the operational overhead of maintaining peering connections. **What is the BEST solution?**

- **A.** More VPC peering connections, managed with AWS CloudFormation
- **B.** AWS Transit Gateway with VPC attachments and a VPN or Direct Connect attachment
- **C.** AWS PrivateLink between every pair of VPCs
- **D.** A single large VPC with all workloads

<details>
<summary>Reveal answer</summary>

**✅ B.** A Transit Gateway is a regional **hub**: one attachment per VPC, transitive routing, plus on-prem attachments. Full-mesh peering for 30 VPCs would need 435 connections.

🔁 **Review:** [Lab 2C, Boss Challenge](lab-2c-peering-flow-logs.md#️-boss-challenge-design-the-network-250-xp)

</details>

---

### Question 9

Users report intermittent failures connecting to an application. The operations team wants to determine whether connections are being **rejected by security groups or network ACLs**. **Which service provides this information?**

- **A.** AWS CloudTrail
- **B.** VPC Flow Logs
- **C.** Amazon Inspector
- **D.** AWS Config

<details>
<summary>Reveal answer</summary>

**✅ B.** Flow logs record each flow with an `ACCEPT` or `REJECT` action. CloudTrail logs **API calls** (like who *changed* the security group), not network packets. A favorite distractor.

🔁 **Review:** [Lab 2C, Step 8](lab-2c-peering-flow-logs.md#step-8-break-it-and-investigate-detective-mode-️)

</details>

---

### Question 10 *(Select TWO)*

A company wants to manage EC2 instances in private subnets **without** opening inbound ports or using bastion hosts, and the instances **have no internet access**. **Which TWO steps are required?**

- **A.** Attach an IAM instance profile that includes `AmazonSSMManagedInstanceCore`
- **B.** Open port 22 in the security group from the corporate IP range
- **C.** Create interface VPC endpoints for `ssm`, `ssmmessages` and `ec2messages`
- **D.** Assign Elastic IP addresses to the instances
- **E.** Create a gateway endpoint for Systems Manager

<details>
<summary>Reveal answer</summary>

**✅ A and C.** The role lets the SSM agent authenticate, and the interface endpoints give it a private path to Systems Manager. E doesn't exist (gateway endpoints are S3/DynamoDB only). B and D defeat the purpose.

🔁 **Review:** [Lab 2B, Steps 2–6](lab-2b-private-subnet-endpoints.md#part-1--endpoints)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | Session 3 |
| **8–9** | ✅ Pass | +200 | Log misses, move on |
| **6–7** | 🔁 Almost | +0 | Redo the 🔁 links, retake in 48 h (+150 on pass) |
| **≤ 5** | 📚 Rebuild | +0 | Rebuild Lab 2B: it's the densest exam content in the track |

### 🗂️ Speed Round (+5 XP each, 5 seconds per answer)

| Prompt | Answer |
|--------|--------|
| Usable IPs in a /24 | 251 |
| Stateful firewall | Security group |
| Firewall with deny rules | NACL |
| Free endpoint type | Gateway (S3, DynamoDB) |
| Where a NAT gateway lives | Public subnet |
| IPv6-only outbound gateway | Egress-only internet gateway |
| Hub for many VPCs | Transit Gateway |
| Shows ACCEPT/REJECT records | VPC Flow Logs |

---

**Next:** [Session 3 — Data Protection, Lab 3A →](../session-03-data-protection/lab-3a-kms-envelope-encryption.md)
