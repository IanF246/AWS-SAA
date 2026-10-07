# Lab 2B: Private Subnets, VPC Endpoints, Security Groups & NACLs

**Session:** 2 — VPC Networking  
**Exam Domain:** Domain 1 — Design Secure Architectures (30%) · Domain 4 — Cost-Optimized (20%)  
**Difficulty:** Intermediate  
**Estimated Time:** 45–60 minutes

---

## Overview

A server in a **private subnet with no internet access at all** still needs to be managed (Session Manager) and needs to read from S3. How? **VPC endpoints.** This exact scenario shows up on the exam again and again, often with "most secure" or "most cost-effective" attached.

Then you'll put the two VPC firewalls side by side, **security groups (stateful)** and **network ACLs (stateless)**, and break traffic on purpose to see the difference.

**What you will build:**

```
 saa-vpc ─────────────────────────────────────────────────────────────
 │ private-a 10.0.11.0/24 (no NAT, no IGW route)                     │
 │   ┌────────────────┐   443   ┌────────────────────────────────┐   │
 │   │ EC2 (no public │ ──────▶ │ Interface endpoints (ENIs)     │──▶ Systems Manager
 │   │  IP, SG: no    │         │ ssm · ssmmessages · ec2messages│   │
 │   │  inbound)      │         └────────────────────────────────┘   │
 │   └───────┬────────┘                                             │
 │           │ private-rt: S3 prefix list → vpce-…  (gateway endpt) ──▶ Amazon S3
 └───────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- ✅ **Lab 2A** complete, with `saa-vpc` still deployed and your ID tracker filled in
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| EC2 t3a.nano | Test instance | ~$0.0047/hr |
| Interface VPC endpoints ×3 (one AZ) | Private access to SSM APIs | ~$0.01/hr each (~$0.03/hr) |
| S3 gateway endpoint | Private access to S3 | **Free** |
| IAM / SSM Session Manager | Roles, remote shell | Always Free |

**Estimated cost: ~$0.05 per hour the lab is running.** Lab 2C reuses this instance. If you aren't continuing right away, do the Cleanup at the end of this lab.

---

## Concepts

**Two kinds of VPC endpoint:**

| | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|-|------------------|----------------------------------|
| Services | **Only S3 and DynamoDB** | 100+ AWS services, plus your own and SaaS |
| How it works | A **route** in your route table (prefix list → endpoint) | An **ENI with a private IP** in your subnet |
| Cost | **Free** | Hourly per AZ + per GB |
| Reachable from on-prem / peered VPC | ❌ No | ✅ Yes |

**Security groups vs. network ACLs:**

| | Security Group | Network ACL |
|-|----------------|-------------|
| Attached to | ENI / instance | Subnet |
| State | **Stateful**: return traffic allowed automatically | **Stateless**: return traffic needs its own rule |
| Rules | Allow only | Allow **and Deny**, evaluated in rule-number order |
| Default | New SG: no inbound, all outbound | Default NACL: allow all. **New custom NACL: deny all** |

> 💡 **Ephemeral ports.** When your instance calls S3 on port 443, S3 replies to a random high port (1024–65535) on your instance. A stateful SG remembers the connection. A stateless NACL doesn't, so you must allow those ports **inbound**. You'll prove that in Part 3.

---

## ⚠️ Placeholders in This Lab

Use the IDs from your Lab 2A tracker, plus these new ones:

| Placeholder | What It Is |
|-------------|-----------|
| `<ENDPOINT_SG_ID>` | Security group for interface endpoints (Step 3) |
| `<INSTANCE_SG_ID>` | Security group for the instance (Step 3) |
| `<AMI_ID>` | Amazon Linux 2023 AMI (Step 5) |
| `<INSTANCE_A_ID>` | Your private instance (Step 6) |
| `<S3_VPCE_ID>` / `<SSM_VPCE_IDS>` | Endpoint IDs (Step 4) |
| `<NACL_ID>` | Your custom NACL (Part 3) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-2`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-2`

---

## Part 1 — Endpoints

### Step 2: Create the Instance Role

**Create `ec2-trust.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

📋 Run in order:
```
aws iam create-role --role-name saa-lab2-ec2-role --assume-role-policy-document file://ec2-trust.json
```
```
aws iam attach-role-policy --role-name saa-lab2-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```
```
aws iam attach-role-policy --role-name saa-lab2-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```
```
aws iam create-instance-profile --instance-profile-name saa-lab2-ec2-profile
```
```
aws iam add-role-to-instance-profile --instance-profile-name saa-lab2-ec2-profile --role-name saa-lab2-ec2-role
```

---

### Step 3: Create Two Security Groups

📋 **Endpoint SG.** It accepts HTTPS from inside the VPC:
```
aws ec2 create-security-group --group-name saa-endpoint-sg --description "HTTPS from VPC to interface endpoints" --vpc-id <VPC_ID> --query GroupId --output text
```
```
aws ec2 authorize-security-group-ingress --group-id <ENDPOINT_SG_ID> --protocol tcp --port 443 --cidr 10.0.0.0/16
```

📋 **Instance SG.** No inbound rules at all:
```
aws ec2 create-security-group --group-name saa-instance-sg --description "No inbound - managed via SSM" --vpc-id <VPC_ID> --query GroupId --output text
```

> 💡 **No inbound rules, no SSH key, no public IP, and you'll still get a shell.** The SSM agent makes *outbound* connections, and the stateful SG allows the replies. Answers that say "open port 22 to 0.0.0.0/0" are never the most secure option.

---

### Step 4: Create the Endpoints

📋 **S3 gateway endpoint** (free), attached to the private route table:
```
aws ec2 create-vpc-endpoint --vpc-id <VPC_ID> --vpc-endpoint-type Gateway --service-name com.amazonaws.us-east-1.s3 --route-table-ids <PRIVATE_RT_ID> --query VpcEndpoint.VpcEndpointId --output text
```

📋 **Three interface endpoints for Session Manager**, in `private-a` only to keep costs down:
```
aws ec2 create-vpc-endpoint --vpc-id <VPC_ID> --vpc-endpoint-type Interface --service-name com.amazonaws.us-east-1.ssm --subnet-ids <PRIV_A_ID> --security-group-ids <ENDPOINT_SG_ID> --private-dns-enabled --query VpcEndpoint.VpcEndpointId --output text
```
```
aws ec2 create-vpc-endpoint --vpc-id <VPC_ID> --vpc-endpoint-type Interface --service-name com.amazonaws.us-east-1.ssmmessages --subnet-ids <PRIV_A_ID> --security-group-ids <ENDPOINT_SG_ID> --private-dns-enabled --query VpcEndpoint.VpcEndpointId --output text
```
```
aws ec2 create-vpc-endpoint --vpc-id <VPC_ID> --vpc-endpoint-type Interface --service-name com.amazonaws.us-east-1.ec2messages --subnet-ids <PRIV_A_ID> --security-group-ids <ENDPOINT_SG_ID> --private-dns-enabled --query VpcEndpoint.VpcEndpointId --output text
```

> **📝 Save all four endpoint IDs.**

📋 Look at what the gateway endpoint did to your route table:
```
aws ec2 describe-route-tables --route-table-ids <PRIVATE_RT_ID> --query "RouteTables[0].Routes[].[DestinationCidrBlock,DestinationPrefixListId,GatewayId]" --output table
```

**✅ You should see** a new route: destination `pl-xxxx` (the S3 **prefix list**, all of S3's IP ranges in us-east-1) → `vpce-xxxx`.

---

### Step 5: Get the AMI

```
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text
```

> **📝 Save as `<AMI_ID>`**

---

### Step 6: Launch the Private Instance

📋 **Replacing the placeholders**:
```
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3a.nano --subnet-id <PRIV_A_ID> --security-group-ids <INSTANCE_SG_ID> --iam-instance-profile Name=saa-lab2-ec2-profile --no-associate-public-ip-address --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=saa-private-a}]" --query "Instances[0].InstanceId" --output text
```

> **📝 Save as `<INSTANCE_A_ID>`**

📋 Wait until it's running, then give the SSM agent about 2 minutes to register through the endpoints:
```
aws ec2 wait instance-running --instance-ids <INSTANCE_A_ID>
```
```
aws ssm describe-instance-information --query "InstanceInformationList[].[InstanceId,PingStatus]" --output table
```

**✅ You should see** your instance with `Online`. If not, wait 1–2 minutes and check again.

---

### Step 7: Connect and Test

Connect: **EC2 → Instances → saa-private-a → Connect → Session Manager → Connect**.

🔮 **Predict each result** before running it inside the session:

| # | Command (on the instance) | Your Prediction |
|---|---------------------------|-----------------|
| 1 | `curl -m 5 -sI https://aws.amazon.com \| head -1` | |
| 2 | `aws s3 ls --region us-east-1` | |
| 3 | `aws sqs list-queues --region us-east-1 --cli-connect-timeout 5` | |

<details>
<summary>🔮 Reveal</summary>

1. ❌ **Times out.** No IGW route, no NAT. This instance has **no internet**.
2. ✅ **Lists your buckets** through the **gateway endpoint**. The traffic never leaves the AWS network.
3. ❌ **Times out.** There's no SQS endpoint. Endpoints are **per service**: you only reach what you've built an endpoint for.

</details>

> 💡 **The cost lesson:** A NAT gateway costs ~$0.045/hr **plus $0.045/GB processed**. A workload pulling 10 TB/month from S3 through NAT pays ~$450/month just in processing. Through a **gateway endpoint it's $0**. This is one of the most common "most cost-effective" answers on the exam.

Type `exit` to close the session.

---

## Part 2 — Lock the Endpoint Down with a Policy

Endpoints can carry **endpoint policies**, which control *what* can be reached through them.

**Create `s3-endpoint-policy.json`.** It only allows access to buckets that start with `saa-`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::saa-*", "arn:aws:s3:::saa-*/*"]
    }
  ]
}
```

```
aws ec2 modify-vpc-endpoint --vpc-endpoint-id <S3_VPCE_ID> --policy-document file://s3-endpoint-policy.json
```

🔮 **Predict:** reconnect via Session Manager and run `aws s3 ls --region us-east-1`. Does it still list all buckets?

<details>
<summary>🔮 Reveal</summary>

❌ **AccessDenied.** `aws s3 ls` with no bucket calls `s3:ListAllMyBuckets`, which the endpoint policy doesn't allow. The instance **role** allows it, but the **endpoint policy** is another layer that must also allow it. This is how companies stop **data exfiltration** to buckets outside their organization.

</details>

📋 Restore full access before continuing:
```
aws ec2 modify-vpc-endpoint --vpc-endpoint-id <S3_VPCE_ID> --reset-policy
```

---

## Part 3 — Stateful vs. Stateless (NACLs)

### Step 8: Create a Custom NACL

```
aws ec2 create-network-acl --vpc-id <VPC_ID> --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=saa-private-nacl}]" --query NetworkAcl.NetworkAclId --output text
```

> **📝 Save as `<NACL_ID>`**

📋 Allow **all outbound** traffic (rule 100), with **no inbound rules yet**:
```
aws ec2 create-network-acl-entry --network-acl-id <NACL_ID> --egress --rule-number 100 --protocol -1 --cidr-block 0.0.0.0/0 --rule-action allow
```

📋 Find the subnet's current NACL association ID:
```
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<PRIV_A_ID> --query "NetworkAcls[0].[NetworkAclId,Associations[?SubnetId=='<PRIV_A_ID>'].NetworkAclAssociationId|[0]]" --output text
```

> **📝 Save the first value as `<DEFAULT_NACL_ID>` and the second as `<ASSOC_ID>`**

📋 Swap the subnet onto your new NACL:
```
aws ec2 replace-network-acl-association --association-id <ASSOC_ID> --network-acl-id <NACL_ID> --query NewAssociationId --output text
```

> **📝 Save the output as `<NEW_ASSOC_ID>`** (you'll need it to swap back)

---

### Step 9: Test the Stateless Firewall

Reconnect with Session Manager.

> 💡 **Why does Session Manager still work?** The instance and the SSM endpoint ENIs are in the **same subnet**. NACLs only filter traffic **crossing the subnet boundary**. S3 traffic, on the other hand, leaves the subnet through the gateway endpoint.

🔮 **Predict:** Outbound is fully open. Will `aws s3 ls` work?

```
AWS_MAX_ATTEMPTS=1 aws s3 ls --region us-east-1 --cli-connect-timeout 5 --cli-read-timeout 5
```

<details>
<summary>🔮 Reveal</summary>

❌ **Times out.** The request goes out fine, but S3's **reply** comes back *inbound* to an ephemeral port, and your NACL has no inbound allow rule. Stateless means the NACL has no memory that your instance started the conversation.

</details>

📋 Back on **your local terminal**, allow inbound ephemeral ports:
```
aws ec2 create-network-acl-entry --network-acl-id <NACL_ID> --ingress --rule-number 100 --protocol tcp --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow
```

Run the `aws s3 ls` command on the instance again. **✅ It works now.**

### 🧩 Checkpoint

You need to block a single malicious IP, `203.0.113.66`, from reaching every instance in a subnet. Security group or NACL?

<details>
<summary>Answer (+10 XP)</summary>

**NACL.** Security groups only have **allow** rules, so you can't deny one IP with them. Add a NACL **deny** rule with a **lower rule number** than your allow rules (for example rule 50), because NACL rules are evaluated in order and the first match wins.

</details>

---

### Step 10: Restore the Default NACL

```
aws ec2 replace-network-acl-association --association-id <NEW_ASSOC_ID> --network-acl-id <DEFAULT_NACL_ID>
```
```
aws ec2 delete-network-acl --network-acl-id <NACL_ID>
```

---

### Step 11: Console Checkpoint

**✅ Checkpoint:**
1. **VPC → Endpoints** lists 1 Gateway and 3 Interface endpoints.
2. Click the `ssm` interface endpoint → **Subnets** tab shows an ENI with a private IP in `private-a`.
3. **VPC → Route tables → saa-private-rt → Routes** shows the `pl-…` → `vpce-…` route.

---

## What You Just Did

1. Managed a server with **no internet, no SSH and no public IP** through interface endpoints
2. Reached S3 privately and for free through a **gateway endpoint**
3. Restricted an endpoint with an **endpoint policy**, the anti-exfiltration pattern
4. Proved **NACLs are stateless** (ephemeral ports) and **SGs are stateful**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Private instances access S3/DynamoDB **without internet**, lowest cost" | **Gateway endpoint** |
| "Private access to SQS / KMS / Secrets Manager / API Gateway" | **Interface endpoint** (PrivateLink) |
| "Expose *my* service privately to other VPCs/accounts without peering" | **PrivateLink** (NLB + endpoint service) |
| "Access S3 privately **from on-premises** over Direct Connect/VPN" | **Interface endpoint for S3** (gateway endpoints don't work from on-prem) |
| "Block a specific IP address" | **NACL deny rule** (or AWS WAF for HTTP) |
| "Allow traffic only from the web tier's instances" | SG rule referencing the **web tier's SG ID** |
| "Manage instances without bastion hosts or SSH" | **Systems Manager Session Manager** |

**🚨 Exam traps**
- Gateway endpoints exist **only** for S3 and DynamoDB.
- Security groups **can't deny**. If an answer says "add a deny rule to the security group," eliminate it.
- When you use a custom NACL, remember **ephemeral ports** inbound (for replies) *and* outbound (for replies to inbound requests).

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Instance never shows `Online` in SSM | Endpoints aren't ready, DNS hostnames are off, or the role is missing | Endpoints take ~2 min to become `available`. Check that DNS hostnames are enabled on the VPC (Lab 2A, Step 2). Confirm the instance profile is attached. |
| `private-dns-enabled` error | DNS support/hostnames are disabled on the VPC | Enable both in **VPC → Actions → Edit VPC settings** |
| `aws s3 ls` times out even after Step 9 | Missing the ephemeral rule, or it was added as egress | Check: `aws ec2 describe-network-acls --network-acl-ids <NACL_ID>` |

---

## 🧹 Cleanup

**⏸️ Continuing to Lab 2C right now?** Keep everything. Lab 2C cleans up all of Session 2.

**Stopping here?** Run these, then the Lab 2A cleanup:
```
aws ec2 terminate-instances --instance-ids <INSTANCE_A_ID>
aws ec2 wait instance-terminated --instance-ids <INSTANCE_A_ID>
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <S3_VPCE_ID> <SSM_VPCE_ID> <SSMMESSAGES_VPCE_ID> <EC2MESSAGES_VPCE_ID>
```

Wait about 2 minutes for the endpoints to finish deleting, then:
```
aws ec2 delete-security-group --group-id <ENDPOINT_SG_ID>
aws ec2 delete-security-group --group-id <INSTANCE_SG_ID>
aws iam remove-role-from-instance-profile --instance-profile-name saa-lab2-ec2-profile --role-name saa-lab2-ec2-role
aws iam delete-instance-profile --instance-profile-name saa-lab2-ec2-profile
aws iam detach-role-policy --role-name saa-lab2-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam detach-role-policy --role-name saa-lab2-ec2-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name saa-lab2-ec2-role
```

---

**🏁 Lab complete: +100 XP.** Next: [Lab 2C — VPC Peering & Flow Logs](lab-2c-peering-flow-logs.md)
