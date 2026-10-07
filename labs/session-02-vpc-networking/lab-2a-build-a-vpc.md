# Lab 2A: Build a Production-Shaped VPC from Scratch

**Session:** 2 — VPC Networking  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%) · Domain 2 — Resilient (26%)  
**Difficulty:** Beginner  
**Estimated Time:** 35–45 minutes

---

## Overview

The console's "VPC and more" wizard builds a VPC in one click, which is great at work and terrible for learning. On the exam you need to know **what every piece does** and **what breaks when it's missing**. So you'll build it by hand, piece by piece.

**What you will build:**

```
 VPC  saa-vpc  10.0.0.0/16
 ┌───────────────────────────────────────────────────────────────┐
 │            us-east-1a                     us-east-1b          │
 │   ┌───────────────────────┐    ┌───────────────────────┐      │
 │   │ public-a 10.0.1.0/24  │    │ public-b 10.0.2.0/24  │ ──┐  │
 │   └───────────────────────┘    └───────────────────────┘   │  │    ┌─────┐
 │   ┌───────────────────────┐    ┌───────────────────────┐   ├──┼──▶ │ IGW │──▶ Internet
 │   │ private-a 10.0.11.0/24│    │ private-b 10.0.12.0/24│   │  │    └─────┘
 │   └───────────────────────┘    └───────────────────────┘   │  │
 │   public-rt:  10.0.0.0/16 → local, 0.0.0.0/0 → IGW ────────┘  │
 │   private-rt: 10.0.0.0/16 → local   (no internet route)       │
 └───────────────────────────────────────────────────────────────┘
```

This VPC is reused in **Labs 2B and 2C**, so don't clean it up until the end of 2C (it costs $0.00 while it sits there).

---

## Prerequisites

- ✅ Session 1 complete (or comfortable with the CLI and IAM)
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon VPC, subnets, route tables | Your private network | Always Free |
| Internet Gateway | Connects the VPC to the internet | Always Free (you pay for data, not the gateway) |

**Estimated cost for this lab: $0.00**

---

## Concepts

| Component | What It Does | Exam Hook |
|-----------|-------------|-----------|
| **VPC** | Your isolated network, defined by a CIDR block (e.g. `/16` = 65,536 IPs) | Regional; spans all AZs |
| **Subnet** | A slice of the VPC CIDR that lives in **exactly one AZ** | High availability = subnets in **2+ AZs** |
| **Route table** | Rules for where traffic goes. Each subnet uses exactly one. | The `local` route can't be deleted |
| **Internet Gateway (IGW)** | Horizontally scaled, HA gateway to the internet. One per VPC. | No bandwidth limit, no single point of failure |
| **Public subnet** | A subnet whose route table has `0.0.0.0/0 → IGW` | That route is **the only** thing that makes it "public" |
| **Private subnet** | No route to an IGW | Reaches the internet only through a NAT gateway |

> 💡 **5 reserved IPs per subnet.** AWS keeps the first four addresses and the last one in every subnet: network address, VPC router, DNS, future use and broadcast. A `/24` gives you **251** usable IPs, not 256. Yes, the exam asks this.

---

## 📝 ID Tracker

You'll create a lot of IDs. Create a file `ids.txt` in your project folder and paste each ID in as you go, or fill in this table:

| Resource | ID |
|----------|----|
| `<VPC_ID>` | |
| `<PUB_A_ID>` | |
| `<PUB_B_ID>` | |
| `<PRIV_A_ID>` | |
| `<PRIV_B_ID>` | |
| `<IGW_ID>` | |
| `<PUBLIC_RT_ID>` | |
| `<PRIVATE_RT_ID>` | |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-session-2; cd ~\Desktop\saa-session-2; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-session-2 && cd ~/Desktop/saa-session-2 && code .
```

> 💡 This folder is shared by Labs 2A, 2B and 2C.

---

### Step 2: Create the VPC

📋 Copy and paste:
```
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region us-east-1 --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=saa-vpc}]" --query Vpc.VpcId --output text
```

> **📝 Save as `<VPC_ID>`**

📋 Turn on DNS hostnames (needed for VPC endpoints in Lab 2B), **replacing `<VPC_ID>`**:
```
aws ec2 modify-vpc-attribute --vpc-id <VPC_ID> --enable-dns-hostnames Value=true
```

---

### Step 3: Create Four Subnets Across Two AZs

📋 Run each, **replacing `<VPC_ID>`**, and save each subnet ID:

```
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=saa-public-a}]" --query Subnet.SubnetId --output text
```
```
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=saa-public-b}]" --query Subnet.SubnetId --output text
```
```
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.11.0/24 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=saa-private-a}]" --query Subnet.SubnetId --output text
```
```
aws ec2 create-subnet --vpc-id <VPC_ID> --cidr-block 10.0.12.0/24 --availability-zone us-east-1b --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=saa-private-b}]" --query Subnet.SubnetId --output text
```

🔮 **Predict:** How many available IP addresses does AWS report for each `/24` subnet?

📋 Check:
```
aws ec2 describe-subnets --filters Name=vpc-id,Values=<VPC_ID> --query "Subnets[].[Tags[?Key=='Name']|[0].Value,CidrBlock,AvailabilityZone,AvailableIpAddressCount]" --output table
```

<details>
<summary>🔮 Reveal</summary>

**251.** 256 − 5 reserved. +10 XP if you said 251.

</details>

### 🧩 Checkpoint

At this moment, are the `public-a` and `public-b` subnets "public"?

<details>
<summary>Answer (+10 XP)</summary>

**No.** Right now all four subnets are identical. They use the VPC's **main route table**, which only has the `local` route. The name tag means nothing to AWS. A subnet becomes public only when its route table sends `0.0.0.0/0` to an Internet Gateway, and you'll do that next.

</details>

---

### Step 4: Create and Attach the Internet Gateway

```
aws ec2 create-internet-gateway --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=saa-igw}]" --query InternetGateway.InternetGatewayId --output text
```

> **📝 Save as `<IGW_ID>`**

```
aws ec2 attach-internet-gateway --internet-gateway-id <IGW_ID> --vpc-id <VPC_ID>
```

---

### Step 5: Build the Public Route Table

```
aws ec2 create-route-table --vpc-id <VPC_ID> --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=saa-public-rt}]" --query RouteTable.RouteTableId --output text
```

> **📝 Save as `<PUBLIC_RT_ID>`**

📋 The route that makes a subnet public:
```
aws ec2 create-route --route-table-id <PUBLIC_RT_ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <IGW_ID>
```

**✅ You should see** `"Return": true`

📋 Associate both public subnets:
```
aws ec2 associate-route-table --route-table-id <PUBLIC_RT_ID> --subnet-id <PUB_A_ID>
```
```
aws ec2 associate-route-table --route-table-id <PUBLIC_RT_ID> --subnet-id <PUB_B_ID>
```

📋 Auto-assign public IPs to instances launched in the public subnets:
```
aws ec2 modify-subnet-attribute --subnet-id <PUB_A_ID> --map-public-ip-on-launch
```
```
aws ec2 modify-subnet-attribute --subnet-id <PUB_B_ID> --map-public-ip-on-launch
```

> 💡 An instance needs **both** a route to the IGW **and** a public IP (or Elastic IP) to reach the internet directly. Missing either one = no internet. Exam questions about "the instance can't reach the internet" usually come down to one of these two.

---

### Step 6: Build the Private Route Table

Private subnets get their **own** explicit route table, which is safer than relying on the main table.

```
aws ec2 create-route-table --vpc-id <VPC_ID> --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=saa-private-rt}]" --query RouteTable.RouteTableId --output text
```

> **📝 Save as `<PRIVATE_RT_ID>`**

```
aws ec2 associate-route-table --route-table-id <PRIVATE_RT_ID> --subnet-id <PRIV_A_ID>
```
```
aws ec2 associate-route-table --route-table-id <PRIVATE_RT_ID> --subnet-id <PRIV_B_ID>
```

> 💡 **Why not just leave private subnets on the main route table?** If someone later adds an IGW route to the main table "for testing," every subnet using it silently becomes public. Explicit associations protect you from that.

---

### Step 7: Verify the Routes

📋 Copy and paste:
```
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<VPC_ID> --query "RouteTables[].{Name:Tags[?Key=='Name']|[0].Value,Routes:Routes[].[DestinationCidrBlock,GatewayId]}" --output json
```

**✅ You should see:**
- `saa-public-rt`: `10.0.0.0/16 → local` **and** `0.0.0.0/0 → igw-…`
- `saa-private-rt`: only `10.0.0.0/16 → local`
- One table with no name: the **main** route table, created automatically with the VPC

---

### Step 8: Console Checkpoint — the Resource Map

**✅ Checkpoint:**
1. Open **VPC → Your VPCs → saa-vpc**.
2. Click the **Resource map** tab.
3. You should see 4 subnets → 2 named route tables → the IGW connected to the public route table only.

Compare it with the diagram at the top of this lab. If they match, you've built it right.

---

### 🧩 Checkpoint

A web tier must survive the loss of an entire Availability Zone. Which subnets should the load balancer and instances use?

<details>
<summary>Answer (+10 XP)</summary>

**At least two subnets in different AZs**: the load balancer in `public-a` + `public-b`, the instances in `private-a` + `private-b`. A subnet can never span AZs, so "multi-AZ" always means "multiple subnets."

</details>

---

## What You Just Did

1. Created a `/16` VPC and carved it into four `/24` subnets across **two AZs**
2. Proved that AWS reserves 5 IPs per subnet
3. Made subnets public with **one route** (`0.0.0.0/0 → IGW`) plus public IP assignment
4. Isolated private subnets with an explicit route table

---

## 🎯 Exam Corner

| Fact | Detail |
|------|--------|
| VPC CIDR size | `/16` (largest) down to `/28` (smallest) |
| Reserved IPs | 5 per subnet |
| Subnet scope | One AZ |
| IGWs per VPC | One |
| What makes a subnet public | Route table: `0.0.0.0/0 → IGW` |
| Private-subnet internet egress (IPv4) | **NAT gateway** in a **public** subnet (one per AZ for HA) |
| Private-subnet egress (IPv6) | **Egress-only internet gateway** |
| Overlapping CIDRs | Can't be peered, so plan non-overlapping ranges up front |

**🚨 Exam traps**
- "Place the NAT gateway in the private subnet" → wrong. It goes in a **public** subnet.
- "One NAT gateway is highly available across AZs" → wrong. A NAT gateway is HA **within** its AZ. For AZ resilience, deploy one per AZ.
- "Attach a security group to the subnet" → wrong. SGs attach to **ENIs/instances**. NACLs attach to **subnets**.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InvalidSubnet.Conflict` | CIDR overlaps an existing subnet | Double-check the CIDR in the command |
| `Resource.AlreadyAssociated` | The subnet already has that route table | Safe to ignore |
| `InvalidParameterValue` on `--availability-zone` | That AZ name isn't available to your account | List AZs: `aws ec2 describe-availability-zones --query "AvailabilityZones[].ZoneName"` and swap in another |
| `--enable-dns-hostnames` errors | Older CLI version or typo | Enable it in the console instead: **VPC → Actions → Edit VPC settings → Enable DNS hostnames** |

---

## 🧹 Cleanup

**⏸️ Continuing to Lab 2B?** Keep everything. It costs nothing.

**Stopping for good?** Run these in order:
```
aws ec2 delete-subnet --subnet-id <PUB_A_ID>
aws ec2 delete-subnet --subnet-id <PUB_B_ID>
aws ec2 delete-subnet --subnet-id <PRIV_A_ID>
aws ec2 delete-subnet --subnet-id <PRIV_B_ID>
aws ec2 delete-route-table --route-table-id <PUBLIC_RT_ID>
aws ec2 delete-route-table --route-table-id <PRIVATE_RT_ID>
aws ec2 detach-internet-gateway --internet-gateway-id <IGW_ID> --vpc-id <VPC_ID>
aws ec2 delete-internet-gateway --internet-gateway-id <IGW_ID>
aws ec2 delete-vpc --vpc-id <VPC_ID>
```

---

**🏁 Lab complete: +100 XP.** Next: [Lab 2B — Private Subnets, Endpoints, SGs & NACLs](lab-2b-private-subnet-endpoints.md)
