# Lab 7C: Route 53 Routing Policies — Weighted, Failover & Multivalue (with a Private Hosted Zone)

**Session:** 7 — Storage & DNS Performance  
**Exam Domain:** Domain 3 — High-Performing (24%) · Domain 2 — Resilient (26%)  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes

---

## Overview

Route 53 shows up on the exam as the **traffic director**: send 10% of users to the new version, fail over to the DR Region when the primary's health check fails, route Europeans to Frankfurt. All of that is done with **routing policies**.

You don't need to buy a domain for this lab. You'll use a **private hosted zone** (`saa.internal`), visible only inside your VPC, and query it from an EC2 instance with `dig`. You'll:
1. Run an **80/20 weighted** canary, then cut over completely
2. Build **failover** routing that reacts to a real (failing) **health check**
3. Return several healthy IPs with **multivalue answer**

---

## Prerequisites

- ✅ Labs 7A and 7B complete
- ✅ Default VPC in us-east-1

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Route 53 private hosted zone | `saa.internal` | $0.50/month, **free if deleted within 12 hours** |
| Route 53 queries | A few hundred `dig`s | $0.40 per million (≈ $0) |
| Route 53 health check (non-AWS endpoint) | One TCP check | $0.75/month (delete promptly; you'll pay a few cents at most) |
| EC2 t3.micro + public IPv4 | The query client | ~$0.015/hr |

**Estimated cost for this lab: ≤ $0.05.** Finish and clean up in one sitting.

---

## Concepts

**Routing policies:**

| Policy | What It Does | Exam Keyword |
|--------|-------------|--------------|
| **Simple** | One record, one or more values, no health checks | "Single resource" |
| **Weighted** | Split traffic by percentage | "Canary", "A/B test", "send 10% to new version" |
| **Failover** | Primary → secondary when the primary's health check fails | "Active-passive DR" |
| **Latency** | Send users to the Region with the lowest **latency** for them | "Best performance for global users" |
| **Geolocation** | Route by the user's **country/continent** | "Content restricted by country", "localization", "compliance" |
| **Geoproximity** | Route by **distance**, with a **bias** to shift traffic | "Gradually shift traffic between Regions by geography" |
| **Multivalue answer** | Up to 8 **healthy** records returned at random | "Simple client-side load balancing with health checks" |
| **IP-based** | Route by the client's **IP range (CIDR)** | "Route ISP X users to endpoint Y" |

**Alias records** are Route 53's AWS-aware pointers (to an ALB, CloudFront, S3 website, API Gateway…). They're **free to query**, they **work at the zone apex** (`example.com`), and they track the target's IPs automatically.

> 🚨 **Exam trap:** You **can't** create a CNAME at the zone apex (`example.com`). Use an **Alias** record.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<DEFAULT_VPC_ID>`, `<SUBNET_1A>` | Default VPC and subnet |
| `<ZONE_ID>` | Hosted zone ID (Step 2), like `Z0123456ABCDEFG` |
| `<HC_ID>` | Health check ID (Step 6) |
| `<AMI_ID>`, `<CLIENT_ID>` | AMI and client instance |

---

## Lab Steps

### Step 1: Profile, Folder and Network Lookup

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-7`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-7`

```
aws ec2 describe-vpcs --filters Name=is-default,Values=true --query "Vpcs[0].VpcId" --output text
```
```
aws ec2 describe-subnets --filters Name=default-for-az,Values=true Name=availability-zone,Values=us-east-1a --query "Subnets[0].SubnetId" --output text
```

---

### Step 2: Create the Private Hosted Zone

📋 The `--caller-reference` just has to be unique per request. Add today's date or your initials:
```
aws route53 create-hosted-zone --name saa.internal --vpc VPCRegion=us-east-1,VPCId=<DEFAULT_VPC_ID> --caller-reference saa-lab7c-<ANY_UNIQUE_TEXT> --hosted-zone-config Comment=SAA-lab,PrivateZone=true --query HostedZone.Id --output text
```

**✅ You should see** `/hostedzone/Z0…`. **📝 Save the part after `/hostedzone/` as `<ZONE_ID>`.**

> 💡 A **private hosted zone** answers only for resources **inside the associated VPCs**. It's perfect for internal service names like `db.prod.internal`. (The VPC needs DNS support and DNS hostnames enabled, which they are by default in the default VPC.)

---

### Step 3: Launch a Query Client

**Create `ec2-trust.json`** (if it isn't already in the folder from Lab 7B):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
```
```
aws iam create-role --role-name saa-dns-role --assume-role-policy-document file://ec2-trust.json
```
```
aws iam attach-role-policy --role-name saa-dns-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```
```
aws iam create-instance-profile --instance-profile-name saa-dns-profile
```
```
aws iam add-role-to-instance-profile --instance-profile-name saa-dns-profile --role-name saa-dns-role
```
```
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text
```

Wait 10 seconds, then:
```
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro --subnet-id <SUBNET_1A> --iam-instance-profile Name=saa-dns-profile --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=saa-dns-client}]" --query "Instances[0].InstanceId" --output text
```

> **📝 Save as `<CLIENT_ID>`.** After ~2 minutes, connect with **Session Manager** and install `dig`:
```bash
sudo dnf install -y bind-utils
```

---

## Part 1 — Weighted Routing (Canary Release)

### Step 4: 80% Blue, 20% Green

The IPs are made-up private addresses. We only care what DNS *answers*, so nothing needs to be listening.

**Create `weighted.json`:**
```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.saa.internal", "Type": "A", "TTL": 1,
        "SetIdentifier": "blue", "Weight": 80,
        "ResourceRecords": [ { "Value": "10.99.0.1" } ]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.saa.internal", "Type": "A", "TTL": 1,
        "SetIdentifier": "green", "Weight": 20,
        "ResourceRecords": [ { "Value": "10.99.0.2" } ]
      }
    }
  ]
}
```
```
aws route53 change-resource-record-sets --hosted-zone-id <ZONE_ID> --change-batch file://weighted.json
```

🔮 **Predict:** out of 40 lookups, roughly how many return `10.99.0.2`?

On the **client** (Session Manager):
```bash
for i in $(seq 1 40); do dig +short app.saa.internal; sleep 1; done | sort | uniq -c
```

<details>
<summary>🔮 Reveal</summary>

Roughly **32 × 10.99.0.1** and **8 × 10.99.0.2** (it's random, so ±a few). Weight / sum of weights = probability: 20/100 = 20%.

> 💡 TTL is set to **1 second** so caching doesn't skew the demo. In production, a canary with a long TTL (say 1 hour) shifts traffic **slowly**, because clients cache their answer.

</details>

---

### Step 5: Full Cut-Over (Blue → Green)

Green looks good. In `weighted.json`, set blue's `Weight` to **0** and green's to **100**, save, and re-apply:
```
aws route53 change-resource-record-sets --hosted-zone-id <ZONE_ID> --change-batch file://weighted.json
```
Re-run the `dig` loop on the client.

**✅ You should see** only `10.99.0.2`. Rollback is just as fast: flip the weights back. That's **blue/green deployment with DNS**.

### 🧩 Checkpoint

If **all** weighted records have weight 0, what does Route 53 do?

<details>
<summary>Answer (+10 XP)</summary>

It returns **all of them with equal probability**. Weight 0 on *every* record is treated as equal weighting. Weight 0 on *some* records means "never return these" (unless all non-zero records are unhealthy).

</details>

---

## Part 2 — Failover Routing with a Health Check

### Step 6: Create a Health Check That Will Fail

Route 53 health checkers run on the **public internet**. Point one at `192.0.2.44`, an address reserved for documentation that never answers:
```
aws route53 create-health-check --caller-reference saa-hc-<ANY_UNIQUE_TEXT> --health-check-config IPAddress=192.0.2.44,Port=80,Type=TCP,RequestInterval=30,FailureThreshold=1 --query HealthCheck.Id --output text
```

> **📝 Save as `<HC_ID>`**

📋 After ~1–2 minutes, check what the global checkers see:
```
aws route53 get-health-check-status --health-check-id <HC_ID> --query "HealthCheckObservations[0:3].[Region,StatusReport.Status]" --output table
```

**✅ You should see** failures like `Failure: Connection timed out`.

---

### Step 7: Primary and Secondary Records

**Create `failover.json`**, **replacing `<HC_ID>`**:
```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.saa.internal", "Type": "A", "TTL": 1,
        "SetIdentifier": "primary-us-east-1", "Failover": "PRIMARY",
        "HealthCheckId": "<HC_ID>",
        "ResourceRecords": [ { "Value": "10.99.1.10" } ]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "db.saa.internal", "Type": "A", "TTL": 1,
        "SetIdentifier": "secondary-us-west-2", "Failover": "SECONDARY",
        "ResourceRecords": [ { "Value": "10.99.1.20" } ]
      }
    }
  ]
}
```
```
aws route53 change-resource-record-sets --hosted-zone-id <ZONE_ID> --change-batch file://failover.json
```

🔮 **Predict:** What does the client get?
```bash
dig +short db.saa.internal
```

<details>
<summary>🔮 Reveal</summary>

**`10.99.1.20`, the secondary.** The primary's health check is failing, so Route 53 answers with the secondary. In a real active-passive DR setup (Lab 6C's Boss Challenge), that's your **automatic Regional failover**.

</details>

> 💡 **Health checks for private resources:** Route 53 checkers can't reach private IPs. In production, you'd use a health check based on a **CloudWatch alarm** (e.g. on your ALB's healthy host count) or a **calculated** health check that combines child checks.

---

## Part 3 — Multivalue Answer

### Step 8: Three Healthy Answers

**Create `multivalue.json`:**
```json
{
  "Changes": [
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "api.saa.internal", "Type": "A", "TTL": 1, "SetIdentifier": "node-1", "MultiValueAnswer": true, "ResourceRecords": [ { "Value": "10.99.2.1" } ] } },
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "api.saa.internal", "Type": "A", "TTL": 1, "SetIdentifier": "node-2", "MultiValueAnswer": true, "ResourceRecords": [ { "Value": "10.99.2.2" } ] } },
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "api.saa.internal", "Type": "A", "TTL": 1, "SetIdentifier": "node-3", "MultiValueAnswer": true, "ResourceRecords": [ { "Value": "10.99.2.3" } ] } }
  ]
}
```
```
aws route53 change-resource-record-sets --hosted-zone-id <ZONE_ID> --change-batch file://multivalue.json
```

On the client:
```bash
dig +short api.saa.internal
```

**✅ You should see** all three IPs. Attach health checks and unhealthy ones drop out of the answer. 🚨 But it's **not a load balancer substitute**: there's no connection-level balancing and no draining.

---

### Step 9: 🎮 Policy Picker (+10 XP each)

| # | Requirement | Policy |
|---|-------------|--------|
| 1 | Users in Germany must only see the German site, for legal reasons | |
| 2 | Global users should hit whichever Region responds fastest for them | |
| 3 | Send 5% of traffic to a new app version | |
| 4 | Active-passive DR between two Regions | |
| 5 | Gradually shift more European traffic toward a new Region by expanding its "area" | |
| 6 | `example.com` (apex) must point to a CloudFront distribution | |

<details>
<summary>🎮 Reveal</summary>

1. **Geolocation** (country-based, deterministic)
2. **Latency-based**
3. **Weighted**
4. **Failover** (+ health check)
5. **Geoproximity** with **bias**
6. **Alias record** (A/AAAA alias to CloudFront). A CNAME isn't allowed at the apex.

</details>

---

### Step 10: Console Checkpoint

**✅ Checkpoint:**
1. **Route 53 → Hosted zones → saa.internal**: Type *Private*, and the records show their routing policy column.
2. **Route 53 → Health checks**: your check is **Unhealthy** (red), and the graph shows the observations.

---

## ⚔️ Boss Challenge: Global Architecture (+250 XP)

**Scenario:** A video-streaming company serves users in North America, Europe and Asia from ALBs in **us-east-1, eu-west-1 and ap-southeast-1**. Requirements:
1. Each user should reach the **fastest** Region.
2. If a Region's ALB becomes unhealthy, its users go to the next-best Region **automatically**.
3. Static video segments must be cached **close to users**.
4. A partner's firewall needs **two static IP addresses** for the API, with fast failover that doesn't depend on DNS TTLs.

<details>
<summary>🏆 Answer key</summary>

1. Route 53 **latency-based routing** with **alias** records to each Regional ALB. (+60)
2. **Evaluate Target Health = Yes** on the alias records (or attach health checks). Route 53 skips unhealthy Regions and answers with the next-lowest latency. (+60)
3. **Amazon CloudFront** in front of the S3 / ALB origins, using its global edge locations, with **Origin Access Control** for S3. (+60)
4. **AWS Global Accelerator**: **2 static anycast IPs**, routing over the AWS backbone to healthy Regional endpoints, with failover in **seconds** and no DNS caching involved. (+70)

</details>

---

## What You Just Did

1. Created a **private hosted zone** and queried it from inside the VPC
2. Ran a **weighted** canary and a blue/green cut-over
3. Triggered **failover** with a real failing **health check**
4. Returned several answers with **multivalue**, and drilled every routing policy

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Internal DNS names only resolvable inside VPCs" | **Private hosted zone** |
| "Resolve on-prem DNS names from the VPC, or VPC names from on-prem" | **Route 53 Resolver** inbound/outbound **endpoints** + forwarding rules |
| "Static IPs, global, TCP/UDP, fast failover" | **Global Accelerator** |
| "Cache static/dynamic content at the edge" | **CloudFront** |
| "Point the apex domain to an ALB/CloudFront" | **Alias** record |
| "DNS failover" | **Failover routing** + health checks |

**🚨 Exam traps**
- **Geolocation ≠ latency.** Geolocation is about *where the user is* (compliance, localization). Latency is about *what's fastest*.
- Geolocation needs a **default** record, or users from unmatched locations get no answer.
- Alias queries to AWS resources are **free**; CNAME queries are charged.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `dig` returns nothing | Client isn't in the associated VPC, or the record was just created | Confirm the client is in the default VPC; wait 30–60 s |
| `HostedZoneAlreadyExists` / `CallerReference` error | Reused caller reference | Change the `<ANY_UNIQUE_TEXT>` |
| `InvalidChangeBatch` | JSON typo, or the HC ID is missing | Validate the JSON; check `<HC_ID>` |
| Failover returns the primary | The health check hasn't failed yet | Wait 1–2 minutes; check `get-health-check-status` |

---

## 🧹 Cleanup — All of Session 7

**1. Delete the records.** The easy way: **Route 53 → Hosted zones → saa.internal**, select every record **except NS and SOA**, then **Delete records**.

> CLI alternative: copy each of `weighted.json`, `failover.json` and `multivalue.json`, change every `"UPSERT"` to `"DELETE"` (values must match the *current* records exactly, so blue's weight is now `0` and green's is `100`), and run `change-resource-record-sets` with each.

**2. Zone, health check, client:**
```
aws route53 delete-hosted-zone --id <ZONE_ID>
aws route53 delete-health-check --health-check-id <HC_ID>
aws ec2 terminate-instances --instance-ids <CLIENT_ID>
aws ec2 wait instance-terminated --instance-ids <CLIENT_ID>
```

**3. IAM:**
```
aws iam remove-role-from-instance-profile --instance-profile-name saa-dns-profile --role-name saa-dns-role
aws iam delete-instance-profile --instance-profile-name saa-dns-profile
aws iam detach-role-policy --role-name saa-dns-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name saa-dns-role
```

**4. Local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-session-7`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-session-7`

**✅ Checkpoint:** Route 53 shows no `saa.internal` zone and no health checks. EC2 shows `saa-dns-client` terminated.

---

**🏁 Lab complete: +100 XP.** Session 7 done! **[📝 Mini Exam 7 →](mini-exam-07.md)**
