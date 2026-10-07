# Lab 4A: Launch Templates & Self-Healing Auto Scaling Groups

**Session:** 4 — Resilient Compute  
**Exam Domain:** Domain 2 — Design Resilient Architectures (26%)  
**Difficulty:** Beginner  
**Estimated Time:** 35–45 minutes

---

## Overview

A single EC2 instance is a single point of failure. The SAA exam's default answer for "highly available and fault-tolerant compute" is an **Auto Scaling group (ASG) across multiple Availability Zones**, usually behind a load balancer (Lab 4B).

In this lab you'll write a **launch template** (the blueprint), create an ASG that keeps **two web servers in two AZs**, then **kill an instance on purpose** and watch AWS replace it on its own.

**What you will build:**

```
            Auto Scaling group: saa-web-asg  (min 2 · desired 2 · max 4)
         ┌───────────────────────────────┬───────────────────────────────┐
         │        us-east-1a             │         us-east-1b            │
         │   ┌───────────────────┐       │    ┌───────────────────┐      │
         │   │ web (from LT)     │       │    │ web (from LT)     │      │
         │   │ httpd + IMDSv2    │       │    │ httpd + IMDSv2    │      │
         │   └───────────────────┘       │    └───────────────────┘      │
         └───────────────────────────────┴───────────────────────────────┘
                      ▲ Launch template: saa-web-lt (AMI, type, SG, role, user data)
```

You'll keep this ASG for **Labs 4B and 4C**.

---

## Prerequisites

- ✅ Session 2 complete (subnets, AZs, security groups)
- ✅ Your account still has its **default VPC** in us-east-1 (check: VPC console → Your VPCs → "Default VPC: Yes")

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| EC2 t3.micro ×2 | Web servers | ~$0.0104/hr each |
| Public IPv4 addresses ×2 | Public IPs on instances | $0.005/hr each |
| CloudWatch detailed monitoring | 1-minute metrics (used in Lab 4C) | ~$0.003/instance-hr |
| Auto Scaling, launch templates | Orchestration | Always Free |

**Estimated cost: ~$0.04 per hour.** If you're not continuing to Lab 4B today, do the Cleanup.

---

## Concepts

| Term | Meaning |
|------|---------|
| **Launch template** | Versioned blueprint for instances: AMI, type, SG, IAM role, user data, metadata options. Replaces the legacy *launch configuration*. |
| **Auto Scaling group** | Keeps **desired capacity** instances running between **min** and **max**, spread across the subnets (AZs) you give it |
| **Health check** | EC2 status checks (default), or ELB health checks (Lab 4B). Unhealthy instances are **terminated and replaced**. |
| **AZ rebalancing** | The ASG keeps instances evenly spread across AZs |
| **IMDSv2** | Session-token-based instance metadata, which blocks SSRF-style credential theft. Require it in your launch templates. |

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<YOUR_PROFILE_NAME>` | Your CLI profile |
| `<DEFAULT_VPC_ID>` | Your default VPC (Step 2) |
| `<SUBNET_1A>`, `<SUBNET_1B>` | Default subnets in us-east-1a and us-east-1b (Step 2) |
| `<WEB_SG_ID>` | Web security group (Step 3) |
| `<AMI_ID>` | Amazon Linux 2023 AMI (Step 5) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-session-4; cd ~\Desktop\saa-session-4; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-session-4 && cd ~/Desktop/saa-session-4 && code .
```

> 💡 This folder is shared by Labs 4A, 4B and 4C.

---

### Step 2: Find Your Default VPC and Subnets

```
aws ec2 describe-vpcs --filters Name=is-default,Values=true --query "Vpcs[0].VpcId" --output text
```
```
aws ec2 describe-subnets --filters Name=default-for-az,Values=true Name=availability-zone,Values=us-east-1a,us-east-1b --query "Subnets[].[AvailabilityZone,SubnetId]" --output table
```

> **📝 Save `<DEFAULT_VPC_ID>`, `<SUBNET_1A>`, `<SUBNET_1B>`**

---

### Step 3: Security Group and Instance Role

📋 Web security group (port 80 open to the world *for now*; Lab 4B tightens it):
```
aws ec2 create-security-group --group-name saa-web-sg --description "Web servers" --vpc-id <DEFAULT_VPC_ID> --query GroupId --output text
```
```
aws ec2 authorize-security-group-ingress --group-id <WEB_SG_ID> --protocol tcp --port 80 --cidr 0.0.0.0/0
```

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

📋 Role + instance profile (Session Manager access, used in Labs 4B/4C):
```
aws iam create-role --role-name saa-web-role --assume-role-policy-document file://ec2-trust.json
```
```
aws iam attach-role-policy --role-name saa-web-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```
```
aws iam create-instance-profile --instance-profile-name saa-web-profile
```
```
aws iam add-role-to-instance-profile --instance-profile-name saa-web-profile --role-name saa-web-role
```

---

### Step 4: Understand the User Data

Every instance runs this script at first boot. It installs Apache, asks the **metadata service (IMDSv2)** which instance and AZ it's on, and writes a web page plus a `/health` file:

```bash
#!/bin/bash
dnf install -y httpd
dnf install -y stress-ng || true
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>SAA Quest v1</h1><p>Served by <b>$ID</b> in <b>$AZ</b></p>" > /var/www/html/index.html
echo OK > /var/www/html/health
systemctl enable --now httpd
```

> 💡 Launch templates need user data **base64-encoded**. To save you cross-platform encoding headaches, the JSON in Step 5 already contains this exact script, encoded. You can decode it yourself at [base64decode.org](https://www.base64decode.org) to check.

---

### Step 5: Create the Launch Template

📋 Get the latest AMI:
```
aws ssm get-parameter --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 --query Parameter.Value --output text
```

**Create `lt-v1.json`**, **replacing `<AMI_ID>` and `<WEB_SG_ID>`**:
```json
{
  "ImageId": "<AMI_ID>",
  "InstanceType": "t3.micro",
  "SecurityGroupIds": ["<WEB_SG_ID>"],
  "IamInstanceProfile": { "Name": "saa-web-profile" },
  "MetadataOptions": { "HttpTokens": "required", "HttpEndpoint": "enabled" },
  "Monitoring": { "Enabled": true },
  "CreditSpecification": { "CpuCredits": "standard" },
  "TagSpecifications": [
    { "ResourceType": "instance", "Tags": [{ "Key": "Name", "Value": "saa-web" }] }
  ],
  "UserData": "IyEvYmluL2Jhc2gKZG5mIGluc3RhbGwgLXkgaHR0cGQKZG5mIGluc3RhbGwgLXkgc3RyZXNzLW5nIHx8IHRydWUKVE9LRU49JChjdXJsIC1zIC1YIFBVVCAiaHR0cDovLzE2OS4yNTQuMTY5LjI1NC9sYXRlc3QvYXBpL3Rva2VuIiAtSCAiWC1hd3MtZWMyLW1ldGFkYXRhLXRva2VuLXR0bC1zZWNvbmRzOiAzMDAiKQpJRD0kKGN1cmwgLXMgLUggIlgtYXdzLWVjMi1tZXRhZGF0YS10b2tlbjogJFRPS0VOIiBodHRwOi8vMTY5LjI1NC4xNjkuMjU0L2xhdGVzdC9tZXRhLWRhdGEvaW5zdGFuY2UtaWQpCkFaPSQoY3VybCAtcyAtSCAiWC1hd3MtZWMyLW1ldGFkYXRhLXRva2VuOiAkVE9LRU4iIGh0dHA6Ly8xNjkuMjU0LjE2OS4yNTQvbGF0ZXN0L21ldGEtZGF0YS9wbGFjZW1lbnQvYXZhaWxhYmlsaXR5LXpvbmUpCmVjaG8gIjxoMT5TQUEgUXVlc3QgdjE8L2gxPjxwPlNlcnZlZCBieSA8Yj4kSUQ8L2I+IGluIDxiPiRBWjwvYj48L3A+IiA+IC92YXIvd3d3L2h0bWwvaW5kZXguaHRtbAplY2hvIE9LID4gL3Zhci93d3cvaHRtbC9oZWFsdGgKc3lzdGVtY3RsIGVuYWJsZSAtLW5vdyBodHRwZAo="
}
```

> 💡 **What the important lines do:**
> - `HttpTokens: required` → **IMDSv2 only**, a security best practice.
> - `Monitoring: true` → 1-minute CloudWatch metrics, so Lab 4C scales faster.
> - `CpuCredits: standard` → stops T3 "unlimited" mode from billing surplus CPU credits during the Lab 4C stress test.

📋 Create it:
```
aws ec2 create-launch-template --launch-template-name saa-web-lt --launch-template-data file://lt-v1.json --query "LaunchTemplate.[LaunchTemplateId,LatestVersionNumber]" --output text
```

**✅ You should see** an ID like `lt-0abc…` and version `1`.

---

### Step 6: Create the Auto Scaling Group

📋 **Replacing the subnet IDs**. Keep the **single quotes** around the launch-template argument. They stop both PowerShell and bash from treating `$Latest` as a variable.

```
aws autoscaling create-auto-scaling-group --auto-scaling-group-name saa-web-asg --launch-template 'LaunchTemplateName=saa-web-lt,Version=$Latest' --min-size 2 --max-size 4 --desired-capacity 2 --vpc-zone-identifier "<SUBNET_1A>,<SUBNET_1B>"
```

📋 Watch it launch (run a few times over ~1 minute):
```
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names saa-web-asg --query "AutoScalingGroups[0].Instances[].[InstanceId,AvailabilityZone,LifecycleState,HealthStatus]" --output table
```

**✅ You should see** two instances, **one in each AZ**, reaching `InService` / `Healthy`.

🔮 **Predict:** You asked for 2 instances across 2 subnets. Why did the ASG put exactly one in each AZ?

<details>
<summary>🔮 Reveal</summary>

ASGs **balance capacity evenly across AZs** by default. If an AZ fails, you lose half your capacity, not all of it, and the ASG launches replacements in the surviving AZ.

</details>

---

### Step 7: Visit Your Web Servers

📋 Get the public IPs:
```
aws ec2 describe-instances --filters Name=tag:aws:autoscaling:groupName,Values=saa-web-asg Name=instance-state-name,Values=running --query "Reservations[].Instances[].[InstanceId,Placement.AvailabilityZone,PublicIpAddress]" --output table
```

Open `http://<PUBLIC_IP>` for each in your browser (wait ~1 minute after launch for user data to finish).

**✅ You should see** "SAA Quest v1", served by each instance ID, in **different AZs**.

---

### Step 8: Chaos Time — Kill an Instance 💥

Pick **one** instance ID from Step 7. 🔮 **Predict:** What will the ASG do within the next 2 minutes?

📋 Terminate it **outside** Auto Scaling (like a hardware failure would):
```
aws ec2 terminate-instances --instance-ids <ONE_INSTANCE_ID>
```

📋 Watch the scaling activity (repeat every ~30 seconds):
```
aws autoscaling describe-scaling-activities --auto-scaling-group-name saa-web-asg --max-items 3 --query "Activities[].[StatusCode,Description]" --output table
```

<details>
<summary>🔮 Reveal</summary>

The ASG notices the instance is gone or unhealthy, logs *"an instance was taken out of service in response to an EC2 health check…"*, and **launches a replacement**, in the **same AZ** to keep the balance. Capacity returns to 2 without a human. That's **self-healing**.

</details>

### 🧩 Checkpoint

A single EC2 instance runs a stateless app. Management wants automatic recovery if it fails, but **no extra cost**. What's the cheapest self-healing setup?

<details>
<summary>Answer (+10 XP)</summary>

An **ASG with min = max = desired = 1** across 2+ AZs. Auto Scaling is free; you pay only for the one instance. If the instance or its AZ fails, the ASG launches a new one. (For recovering the *same* instance with its IP and EBS on hardware failure, there's also **EC2 auto-recovery** through a CloudWatch alarm or the instance's default recovery behavior.)

</details>

---

### Step 9: Console Checkpoint

**✅ Checkpoint:**
1. **EC2 → Launch Templates → saa-web-lt**: see version 1, with IMDSv2 under **Advanced details → Metadata version: V2 only (token required)**.
2. **EC2 → Auto Scaling Groups → saa-web-asg → Activity**: your terminate-and-replace history.
3. **Instance management** tab: 2 instances, 2 AZs.

---

## What You Just Did

1. Built a versioned **launch template** with IMDSv2, an IAM role and user data
2. Created a **multi-AZ Auto Scaling group** that balances across AZs
3. Simulated a failure and watched the ASG **self-heal**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Highly available, fault-tolerant EC2 application" | **ASG across ≥2 AZs** + load balancer |
| "Replace unhealthy instances automatically" | **ASG** (with ELB health checks; see Lab 4B) |
| "Instances take too long to boot / install software" | Bake a **custom AMI** (golden image) and/or use **warm pools** |
| "Protect against SSRF stealing instance credentials" | Require **IMDSv2** |
| "Launch configuration vs. launch template" | **Launch template**: versioned, supports mixed instances/Spot, and is the current recommendation |

**🚨 Exam traps**
- A **single AZ** ASG isn't "highly available," even with multiple instances.
- Auto Scaling itself is **free**; you pay for the resources it launches.
- User data runs **once at first boot** by default, not on every restart.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Browser shows nothing / times out | User data still running, or SG missing the port 80 rule | Wait 2 minutes; check the SG has `80 ← 0.0.0.0/0` |
| ASG shows no instances, activity says `Invalid IamInstanceProfile` | Profile not propagated yet | Wait 30 seconds; the ASG retries automatically |
| `Version=$Latest` error in PowerShell | Double quotes were used | Use **single quotes** exactly as shown |
| No public IP | Subnets aren't the default subnets | Use the subnets from Step 2 (default subnets auto-assign public IPs) |

---

## 🧹 Cleanup

**⏸️ Continuing to Lab 4B?** Keep everything.

**Stopping here?** Run:
```
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name saa-web-asg --force-delete
aws ec2 delete-launch-template --launch-template-name saa-web-lt
```
Wait about 2 minutes for the instances to terminate, then:
```
aws ec2 delete-security-group --group-id <WEB_SG_ID>
aws iam remove-role-from-instance-profile --instance-profile-name saa-web-profile --role-name saa-web-role
aws iam delete-instance-profile --instance-profile-name saa-web-profile
aws iam detach-role-policy --role-name saa-web-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name saa-web-role
```

---

**🏁 Lab complete: +100 XP.** Next: [Lab 4B — Application Load Balancer](lab-4b-application-load-balancer.md)
