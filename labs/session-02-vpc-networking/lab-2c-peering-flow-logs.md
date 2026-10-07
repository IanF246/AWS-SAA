# Lab 2C: VPC Peering, Flow Logs & Debugging a Network Like a Detective

**Session:** 2 — VPC Networking  
**Exam Domain:** Domain 1 — Secure (30%) · Domain 3 — High-Performing (24%)  
**Difficulty:** Advanced  
**Estimated Time:** 50–60 minutes

---

## Overview

Two teams have two VPCs, and the app in one must reach the database in the other. You'll connect them with **VPC peering**, then use **VPC Flow Logs** to *see* every accepted and rejected packet. Finally you'll break the connection on purpose and diagnose it using only the logs, which is exactly how you'd troubleshoot in a real incident.

**What you will build:**

```
  saa-vpc (A) 10.0.0.0/16                         saa-vpc-b (B) 10.1.0.0/16
 ┌─────────────────────────┐   pcx-…  peering    ┌─────────────────────────┐
 │ private-a               │◀───────────────────▶│ subnet-b 10.1.1.0/24    │
 │  EC2 "app" (from 2B)    │                     │  EC2 "db" (ICMP from A) │
 │ private-rt:             │                     │ main-rt:                │
 │  10.1.0.0/16 → pcx      │                     │  10.0.0.0/16 → pcx      │
 └─────────────────────────┘                     └───────────┬─────────────┘
                                                             │ Flow Logs
                                                             ▼
                                                 CloudWatch Logs /saa/vpc-b-flow-logs
```

---

## Prerequisites

- ✅ **Lab 2B** complete, with `<INSTANCE_A_ID>` still running and endpoints still in place
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| EC2 t3a.nano ×2 | App + DB stand-ins | ~$0.0094/hr |
| Interface endpoints (from 2B) | SSM access | ~$0.03/hr |
| VPC peering | VPC-to-VPC link | Free to create; data across AZs ~$0.01/GB (pings ≈ $0) |
| VPC Flow Logs → CloudWatch Logs | Packet metadata | ~$0.50/GB ingested (this lab: kilobytes ≈ $0) |

**Estimated cost for this lab: ~$0.05**

---

## Concepts

**VPC peering** is a private, one-to-one connection between two VPCs (same or different account or region).
- **Not transitive.** If A↔B and B↔C, **A can't reach C** through B.
- **No overlapping CIDRs.**
- **Routes on both sides.** Each VPC's route table needs a route to the other CIDR via the `pcx-…`.

**VPC Flow Logs** capture IP traffic **metadata** (who, to whom, port, protocol, bytes, ACCEPT/REJECT), not packet contents. They can be enabled at the VPC, subnet or ENI level, and sent to CloudWatch Logs, S3 or Firehose.

A default-format flow log record looks like this:
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
2 123456789012 eni-0abc 10.0.11.25 10.1.1.40 0 0 1 3 252 1728000000 1728000060 REJECT OK
```
(`protocol 1` = ICMP, `6` = TCP, `17` = UDP)

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<VPC_ID>`, `<PRIVATE_RT_ID>`, `<AMI_ID>`, `<INSTANCE_A_ID>` | From Labs 2A/2B |
| `<ACCOUNT_ID>` | Your account ID |
| `<VPC_B_ID>`, `<SUBNET_B_ID>`, `<VPC_B_RT_ID>`, `<DB_SG_ID>`, `<INSTANCE_B_ID>`, `<INSTANCE_B_IP>`, `<PCX_ID>` | Created in this lab |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-2`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-2`

---

### Step 2: Build VPC B

```
aws ec2 create-vpc --cidr-block 10.1.0.0/16 --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=saa-vpc-b}]" --query Vpc.VpcId --output text
```
```
aws ec2 create-subnet --vpc-id <VPC_B_ID> --cidr-block 10.1.1.0/24 --availability-zone us-east-1a --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=saa-b-subnet}]" --query Subnet.SubnetId --output text
```

📋 Get VPC B's **main** route table:
```
aws ec2 describe-route-tables --filters Name=vpc-id,Values=<VPC_B_ID> Name=association.main,Values=true --query "RouteTables[0].RouteTableId" --output text
```

> **📝 Save `<VPC_B_ID>`, `<SUBNET_B_ID>`, `<VPC_B_RT_ID>`**

---

### Step 3: Launch the "DB" Instance in VPC B

📋 Create a security group that allows **ping (ICMP) from VPC A only**:
```
aws ec2 create-security-group --group-name saa-db-sg --description "ICMP from VPC A" --vpc-id <VPC_B_ID> --query GroupId --output text
```
```
aws ec2 authorize-security-group-ingress --group-id <DB_SG_ID> --protocol icmp --port -1 --cidr 10.0.0.0/16
```

📋 Launch it (it needs no role, since you'll only ping it):
```
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3a.nano --subnet-id <SUBNET_B_ID> --security-group-ids <DB_SG_ID> --no-associate-public-ip-address --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=saa-db-b}]" --query "Instances[0].[InstanceId,PrivateIpAddress]" --output text
```

> **📝 Save `<INSTANCE_B_ID>` and `<INSTANCE_B_IP>`** (e.g. `10.1.1.37`)

---

### Step 4: Turn On Flow Logs for VPC B

**Create `flowlogs-trust.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "vpc-flow-logs.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Create `flowlogs-permissions.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

📋 Create the role, log group and flow log:
```
aws iam create-role --role-name saa-flowlogs-role --assume-role-policy-document file://flowlogs-trust.json
```
```
aws iam put-role-policy --role-name saa-flowlogs-role --policy-name write-logs --policy-document file://flowlogs-permissions.json
```
```
aws logs create-log-group --log-group-name /saa/vpc-b-flow-logs
```
```
aws ec2 create-flow-logs --resource-type VPC --resource-ids <VPC_B_ID> --traffic-type ALL --log-destination-type cloud-watch-logs --log-group-name /saa/vpc-b-flow-logs --deliver-logs-permission-arn arn:aws:iam::<ACCOUNT_ID>:role/saa-flowlogs-role --max-aggregation-interval 60 --query "FlowLogIds[0]" --output text
```

> **📝 Save as `<FLOW_LOG_ID>`**
>
> 💡 `--max-aggregation-interval 60` collects records every 1 minute instead of the default 10, so you'll see results faster.

---

### Step 5: Try to Connect Before Peering

Connect to `saa-private-a` (VPC A) with **Session Manager**.

🔮 **Predict:** will this succeed?
```
ping -c 3 -W 2 <INSTANCE_B_IP>
```

<details>
<summary>🔮 Reveal</summary>

❌ **100% packet loss.** VPC A's route table has no route for `10.1.0.0/16`, so the packets don't even leave VPC A. Nothing appears in VPC B's flow logs either.

</details>

Leave the session open.

---

### Step 6: Create and Accept the Peering Connection

On your **local terminal**:
```
aws ec2 create-vpc-peering-connection --vpc-id <VPC_ID> --peer-vpc-id <VPC_B_ID> --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=saa-a-to-b}]" --query VpcPeeringConnection.VpcPeeringConnectionId --output text
```

> **📝 Save as `<PCX_ID>`**

```
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id <PCX_ID> --query VpcPeeringConnection.Status.Code --output text
```

**✅ You should see** `active` (or `provisioning`, which turns active within seconds).

🔮 **Predict:** the peering is active. Try the ping from the instance again. Does it work yet?

<details>
<summary>🔮 Reveal</summary>

❌ **Still fails.** Peering creates the *path*, but neither VPC has a **route** that uses it. This is the #1 peering mistake, in real life and on the exam.

</details>

---

### Step 7: Add Routes on Both Sides

```
aws ec2 create-route --route-table-id <PRIVATE_RT_ID> --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id <PCX_ID>
```
```
aws ec2 create-route --route-table-id <VPC_B_RT_ID> --destination-cidr-block 10.0.0.0/16 --vpc-peering-connection-id <PCX_ID>
```

Back in the Session Manager session:
```
ping -c 3 -W 2 <INSTANCE_B_IP>
```

**✅ You should see** replies (`64 bytes from 10.1.1.x ...`). 🎉

---

### Step 8: Break It and Investigate (Detective Mode 🕵️)

Someone "tidied up" the DB security group. 📋 Remove the ICMP rule:
```
aws ec2 revoke-security-group-ingress --group-id <DB_SG_ID> --protocol icmp --port -1 --cidr 10.0.0.0/16
```

On the instance, run a longer ping to generate evidence:
```
ping -c 10 -W 2 <INSTANCE_B_IP>
```

**✅ Expect** 100% packet loss. Now play detective. Wait **2–3 minutes** for flow log delivery, then on your local terminal:

```
aws logs filter-log-events --log-group-name /saa/vpc-b-flow-logs --filter-pattern REJECT --query "events[].message" --output text --max-items 5
```

**✅ You should see** lines like:
```
2 123456789012 eni-0b… 10.0.11.25 10.1.1.37 0 0 1 1 84 ... REJECT OK
```

### 🧩 Checkpoint — read the evidence

From that log line alone, answer: (a) who was sending, (b) what protocol, and (c) was the traffic blocked going **in** or coming **back**?

<details>
<summary>Answer (+10 XP)</summary>

(a) `10.0.11.25`, the app instance in VPC A. (b) Protocol `1` = **ICMP**. (c) It was rejected **arriving at** the DB's ENI, so the destination's **inbound** rules (security group or NACL) blocked it. The peering and routes are fine; the traffic reached VPC B. That points straight at the SG change.

</details>

📋 Restore the rule and confirm the ping works again:
```
aws ec2 authorize-security-group-ingress --group-id <DB_SG_ID> --protocol icmp --port -1 --cidr 10.0.0.0/16
```

---

### Step 9: Query with CloudWatch Logs Insights (Console)

1. Open **CloudWatch → Logs Insights**, select `/saa/vpc-b-flow-logs`.
2. Paste and run:

```
fields @timestamp, srcAddr, dstAddr, protocol, action
| filter srcAddr like "10.0."
| stats count(*) as packets by action
```

**✅ You should see** a count of `ACCEPT` and `REJECT` records. That's your incident timeline in one query.

---

### 🧩 Checkpoint — transitivity

You add VPC C (`10.2.0.0/16`) and peer it with VPC B. Can the app in VPC A reach VPC C through B?

<details>
<summary>Answer (+10 XP)</summary>

**No.** Peering is **not transitive**, and a route in A pointing `10.2.0.0/16` at the A↔B peering will be dropped by B. You need a direct A↔C peering, or a **Transit Gateway**.

</details>

---

## ⚔️ Boss Challenge: Design the Network (+250 XP)

No commands. Draw it (paper, whiteboard or [draw.io](https://app.diagrams.net)), then compare with the answer key.

**Scenario:** A company has **15 VPCs** across 3 accounts in `us-east-1`, plus an **on-premises data center**. Requirements:
1. Every VPC must reach every other VPC and on-prem.
2. On-prem connectivity needs **consistent, low-latency, high-bandwidth** performance, with an **encrypted backup path** if it fails.
3. Minimize the number of connections to manage.
4. All private instances need outbound internet for patching, centrally inspected.

<details>
<summary>🏆 Answer key</summary>

- **AWS Transit Gateway** as the hub. 15 VPC attachments instead of 105 peering connections (n(n−1)/2). Share it across accounts with **AWS RAM**.
- **AWS Direct Connect** to the Transit Gateway (through a Direct Connect gateway) for consistent, low-latency performance.
- **Site-to-Site VPN** to the Transit Gateway as the encrypted **backup**. (Bonus: VPN can also run *over* Direct Connect if the requirement is "encrypted DX traffic.")
- **Centralized egress VPC** with NAT gateways (one per AZ), plus **AWS Network Firewall** for inspection, routed through the Transit Gateway.

**Scoring:** TGW (+100) · DX + VPN backup (+75) · centralized egress (+50) · RAM sharing (+25)

</details>

---

## What You Just Did

1. Peered two VPCs and learned that peering **plus routes on both sides** = connectivity
2. Turned on **VPC Flow Logs** and read raw records
3. Diagnosed a broken connection from REJECT records alone
4. Designed a hub-and-spoke network at enterprise scale

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Connect 2–3 VPCs, simplest, lowest cost" | **VPC peering** |
| "Connect **many** VPCs + on-prem, transitive routing" | **Transit Gateway** |
| "Dedicated, consistent, private connection to on-prem" | **Direct Connect** (weeks to provision) |
| "Encrypted connection to on-prem **quickly**" | **Site-to-Site VPN** (minutes to hours) |
| "Troubleshoot why traffic is rejected" | **VPC Flow Logs** |
| "Inspect packet *contents*" | **Traffic Mirroring** (flow logs only capture metadata) |
| "Filter/inspect traffic at VPC level (IDS/IPS, domain filtering)" | **AWS Network Firewall** |

**🚨 Exam traps**
- Peering with **overlapping CIDRs** → impossible.
- Flow logs **don't** capture DNS traffic to the Amazon DNS server, DHCP, or instance metadata (169.254.169.254) traffic.
- "Peering gives transitive access to the peer's IGW / NAT / VPN" → no, edge-to-edge routing isn't supported.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| No flow log events after 5 min | Role trust or permissions wrong | `aws ec2 describe-flow-logs --flow-log-ids <FLOW_LOG_ID>` and check `DeliverLogsStatus` / `DeliverLogsErrorMessage` |
| Ping still fails after Step 7 | A route is missing on one side | Check both route tables for the `pcx-…` route |
| Peering stuck in `pending-acceptance` | Not accepted yet | Re-run the accept command |

---

## 🧹 Cleanup — All of Session 2

Run these in order. Do the waits; deletes fail if dependencies still exist.

**1. Instances**
```
aws ec2 terminate-instances --instance-ids <INSTANCE_A_ID> <INSTANCE_B_ID>
aws ec2 wait instance-terminated --instance-ids <INSTANCE_A_ID> <INSTANCE_B_ID>
```

**2. Flow logs, log group, peering**
```
aws ec2 delete-flow-logs --flow-log-ids <FLOW_LOG_ID>
aws logs delete-log-group --log-group-name /saa/vpc-b-flow-logs
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id <PCX_ID>
```

**3. Endpoints** (from Lab 2B)
```
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<VPC_ID> --query "VpcEndpoints[].VpcEndpointId" --output text
```
Copy the IDs it prints, then:
```
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <ID_1> <ID_2> <ID_3> <ID_4>
```
Wait about 2 minutes.

**4. Security groups**
```
aws ec2 delete-security-group --group-id <DB_SG_ID>
aws ec2 delete-security-group --group-id <ENDPOINT_SG_ID>
aws ec2 delete-security-group --group-id <INSTANCE_SG_ID>
```

**5. VPC B**
```
aws ec2 delete-subnet --subnet-id <SUBNET_B_ID>
aws ec2 delete-vpc --vpc-id <VPC_B_ID>
```

**6. VPC A** (from Lab 2A)
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

**7. IAM**
```
aws iam delete-role-policy --role-name saa-flowlogs-role --policy-name write-logs
aws iam delete-role --role-name saa-flowlogs-role
aws iam remove-role-from-instance-profile --instance-profile-name saa-lab2-ec2-profile --role-name saa-lab2-ec2-role
aws iam delete-instance-profile --instance-profile-name saa-lab2-ec2-profile
aws iam detach-role-policy --role-name saa-lab2-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam detach-role-policy --role-name saa-lab2-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name saa-lab2-ec2-role
```

**8. Local folder**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-session-2`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-session-2`

**✅ Checkpoint:** **VPC → Your VPCs** shows only your **default** VPC. **VPC → Endpoints** is empty. **EC2 → Instances** shows both as **Terminated**.

---

**🏁 Lab complete: +100 XP.** Session 2 done! **[📝 Mini Exam 2 →](mini-exam-02.md)**
