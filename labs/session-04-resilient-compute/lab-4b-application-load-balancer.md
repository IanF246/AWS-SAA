# Lab 4B: Application Load Balancer — Health Checks, Security Group Chaining & Zero-Downtime Deploys

**Session:** 4 — Resilient Compute  
**Exam Domain:** Domain 2 — Design Resilient Architectures (26%)  
**Difficulty:** Intermediate  
**Estimated Time:** 45–55 minutes

---

## Overview

Your ASG from Lab 4A self-heals, but users still hit individual IPs that change every time an instance is replaced. In this lab you put an **Application Load Balancer (ALB)** in front: one stable DNS name, traffic spread across AZs, and **health checks that catch problems EC2 can't see**.

Then you'll ship **v2** of the site with an **instance refresh**, a rolling, zero-downtime deployment.

**What you will build:**

```
        Users ──HTTP:80──▶  ALB  saa-web-alb  (SG: 80 from internet)
                              │ listener :80 → target group saa-web-tg
                              │ health check: GET /health every 10s
                 ┌────────────┴────────────┐
                 ▼                         ▼
        web (us-east-1a)            web (us-east-1b)
        SG: 80 ONLY from ALB SG     SG: 80 ONLY from ALB SG
```

---

## Prerequisites

- ✅ **Lab 4A** complete, with `saa-web-asg` running 2 healthy instances

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Application Load Balancer | Layer-7 load balancer | ~$0.0225/hr + LCU (≈ $0 at lab traffic) |
| ALB public IPv4 (one per AZ) | Load balancer addresses | $0.005/hr each |
| EC2 t3.micro ×2 + public IPs | From Lab 4A | ~$0.03/hr |

**Estimated cost: ~$0.07 per hour.** Finish Labs 4B and 4C in one sitting if you can.

---

## Concepts

**The three load balancers:**

| | ALB | NLB | GWLB |
|-|-----|-----|------|
| Layer | 7 (HTTP/HTTPS, gRPC) | 4 (TCP/UDP/TLS) | 3 (IP packets) |
| Superpower | **Path/host/header routing**, Lambda targets, auth (Cognito/OIDC) | **Millions of req/s, ultra-low latency, static IP / Elastic IP per AZ** | Insert **third-party firewalls/appliances** inline |
| Exam keyword | "route `/api` and `/images` to different services" | "static IP", "UDP", "extreme performance" | "virtual appliances", "deep packet inspection" |

**Two kinds of health check:**

| Check | Detects | Misses |
|-------|---------|--------|
| **EC2 status checks** (ASG default) | Hardware/hypervisor failure, unreachable OS | App crashed, web server stopped, bad deploy |
| **ELB health checks** | Anything that breaks the **health endpoint's** HTTP response | — |

**Security group chaining.** Instead of allowing port 80 from `0.0.0.0/0` on your instances, allow it **from the ALB's security group ID**. Then only the load balancer can reach the instances, even if their IPs change.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| From Lab 4A | `<DEFAULT_VPC_ID>`, `<SUBNET_1A>`, `<SUBNET_1B>`, `<WEB_SG_ID>` |
| `<ALB_SG_ID>` | ALB security group (Step 2) |
| `<TG_ARN>` | Target group ARN (Step 3) |
| `<ALB_ARN>`, `<ALB_DNS>` | Load balancer ARN and DNS name (Step 4) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-4`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-4`

---

### Step 2: ALB Security Group

```
aws ec2 create-security-group --group-name saa-alb-sg --description "ALB - HTTP from internet" --vpc-id <DEFAULT_VPC_ID> --query GroupId --output text
```
```
aws ec2 authorize-security-group-ingress --group-id <ALB_SG_ID> --protocol tcp --port 80 --cidr 0.0.0.0/0
```

---

### Step 3: Target Group with a Fast Health Check

```
aws elbv2 create-target-group --name saa-web-tg --protocol HTTP --port 80 --vpc-id <DEFAULT_VPC_ID> --health-check-path /health --health-check-interval-seconds 10 --healthy-threshold-count 2 --unhealthy-threshold-count 2 --query "TargetGroups[0].TargetGroupArn" --output text
```

> **📝 Save as `<TG_ARN>`**
>
> 💡 With a 10-second interval and a threshold of 2, a broken instance is marked unhealthy in **~20 seconds**. The defaults (30 s × 5) would take 2.5 minutes.

---

### Step 4: Create the ALB and Listener

```
aws elbv2 create-load-balancer --name saa-web-alb --type application --scheme internet-facing --subnets <SUBNET_1A> <SUBNET_1B> --security-groups <ALB_SG_ID> --query "LoadBalancers[0].[LoadBalancerArn,DNSName]" --output text
```

> **📝 Save `<ALB_ARN>` and `<ALB_DNS>`** (like `saa-web-alb-123456.us-east-1.elb.amazonaws.com`)

```
aws elbv2 create-listener --load-balancer-arn <ALB_ARN> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<TG_ARN>
```

---

### Step 5: Connect the ASG to the Target Group

📋 Register the ASG with the target group. New instances will join automatically:
```
aws autoscaling attach-load-balancer-target-groups --auto-scaling-group-name saa-web-asg --target-group-arns <TG_ARN>
```

📋 Switch the ASG to **ELB health checks**, with a 90-second grace period for boot time:
```
aws autoscaling update-auto-scaling-group --auto-scaling-group-name saa-web-asg --health-check-type ELB --health-check-grace-period 90
```

📋 Watch the targets become healthy (~1 minute):
```
aws elbv2 describe-target-health --target-group-arn <TG_ARN> --query "TargetHealthDescriptions[].[Target.Id,TargetHealth.State]" --output table
```

**✅ You should see** both instances `healthy`.

📋 Open `http://<ALB_DNS>` in your browser and **refresh several times**.

**✅ You should see** the instance ID and AZ change between refreshes. The ALB is spreading requests across both AZs.

> 💡 **Cross-zone load balancing** is always on for ALBs (at no charge), so each instance gets an even share of requests no matter which AZ's load-balancer node received them.

---

### Step 6: Lock Down the Instances (SG Chaining)

Right now anyone can bypass the ALB by visiting an instance's public IP. Fix that.

📋 Remove the open rule:
```
aws ec2 revoke-security-group-ingress --group-id <WEB_SG_ID> --protocol tcp --port 80 --cidr 0.0.0.0/0
```

📋 Allow port 80 **only from the ALB's security group**:
```
aws ec2 authorize-security-group-ingress --group-id <WEB_SG_ID> --protocol tcp --port 80 --source-group <ALB_SG_ID>
```

🔮 **Predict:** (a) does `http://<ALB_DNS>` still work? (b) does an instance's public IP still work?

<details>
<summary>🔮 Reveal</summary>

(a) ✅ Yes. Traffic from the ALB carries the ALB's SG identity.  
(b) ❌ Times out. Direct access is blocked. This is the **standard three-tier pattern**: each tier only accepts traffic from the tier in front of it, referenced **by security group ID** (never by IP).

</details>

---

### Step 7: The Silent Failure (Why ELB Health Checks Matter)

Simulate a bad deployment that breaks the app while the **OS stays perfectly healthy**.

📋 Pick one instance ID and delete its health file through Systems Manager:
```
aws ssm send-command --instance-ids <ONE_INSTANCE_ID> --document-name AWS-RunShellScript --parameters "commands=rm -f /var/www/html/health" --query Command.CommandId --output text
```

🔮 **Predict:** Over the next 2 minutes: (a) what does the target group say, (b) what do EC2 status checks say, (c) what does the ASG do?

📋 Watch:
```
aws elbv2 describe-target-health --target-group-arn <TG_ARN> --query "TargetHealthDescriptions[].[Target.Id,TargetHealth.State,TargetHealth.Reason]" --output table
```
```
aws autoscaling describe-scaling-activities --auto-scaling-group-name saa-web-asg --max-items 2 --query "Activities[].Description" --output text
```

<details>
<summary>🔮 Reveal</summary>

(a) The target becomes **`unhealthy`** (`Target.ResponseCodeMismatch`, a 404 on `/health`) and the ALB **stops sending it traffic**. Users see only the healthy instance.  
(b) EC2 status checks still pass: **2/2 checks OK**. The OS is fine!  
(c) Because the ASG uses **ELB health checks**, it **terminates and replaces** the instance.

With the default **EC2** health check type, this broken instance would have lived forever, quietly failing its share of requests until the ALB pulled it from rotation. The ASG would never have replaced it.

</details>

### 🧩 Checkpoint

Users complain about **intermittent errors** behind an ALB. The ASG shows all instances healthy, but the ALB shows one target as unhealthy. What's the configuration fix?

<details>
<summary>Answer (+10 XP)</summary>

Set the ASG's **health check type to ELB**, so it replaces instances that fail the load balancer's health check, not just EC2 status checks.

</details>

---

### Step 8: Zero-Downtime Deploy with an Instance Refresh

Ship **v2** of the site. **Create `lt-v2-userdata.json`**. It contains only the changed field (the new page text, base64-encoded):
```json
{
  "UserData": "IyEvYmluL2Jhc2gKZG5mIGluc3RhbGwgLXkgaHR0cGQKZG5mIGluc3RhbGwgLXkgc3RyZXNzLW5nIHx8IHRydWUKVE9LRU49JChjdXJsIC1zIC1YIFBVVCAiaHR0cDovLzE2OS4yNTQuMTY5LjI1NC9sYXRlc3QvYXBpL3Rva2VuIiAtSCAiWC1hd3MtZWMyLW1ldGFkYXRhLXRva2VuLXR0bC1zZWNvbmRzOiAzMDAiKQpJRD0kKGN1cmwgLXMgLUggIlgtYXdzLWVjMi1tZXRhZGF0YS10b2tlbjogJFRPS0VOIiBodHRwOi8vMTY5LjI1NC4xNjkuMjU0L2xhdGVzdC9tZXRhLWRhdGEvaW5zdGFuY2UtaWQpCkFaPSQoY3VybCAtcyAtSCAiWC1hd3MtZWMyLW1ldGFkYXRhLXRva2VuOiAkVE9LRU4iIGh0dHA6Ly8xNjkuMjU0LjE2OS4yNTQvbGF0ZXN0L21ldGEtZGF0YS9wbGFjZW1lbnQvYXZhaWxhYmlsaXR5LXpvbmUpCmVjaG8gIjxoMT5TQUEgUXVlc3QgdjIgLSByb2xsZWQgb3V0IGJ5IGluc3RhbmNlIHJlZnJlc2g8L2gxPjxwPlNlcnZlZCBieSA8Yj4kSUQ8L2I+IGluIDxiPiRBWjwvYj48L3A+IiA+IC92YXIvd3d3L2h0bWwvaW5kZXguaHRtbAplY2hvIE9LID4gL3Zhci93d3cvaHRtbC9oZWFsdGgKc3lzdGVtY3RsIGVuYWJsZSAtLW5vdyBodHRwZAo="
}
```

📋 Create version 2 of the launch template, based on version 1:
```
aws ec2 create-launch-template-version --launch-template-name saa-web-lt --source-version 1 --launch-template-data file://lt-v2-userdata.json --query LaunchTemplateVersion.VersionNumber --output text
```

**✅ You should see** `2`. The ASG uses `$Latest`, so *new* instances get v2. But existing instances don't change until they're replaced, so start a **rolling replacement**:

```
aws autoscaling start-instance-refresh --auto-scaling-group-name saa-web-asg --preferences "MinHealthyPercentage=50,InstanceWarmup=90"
```

📋 Watch progress, and **keep refreshing `http://<ALB_DNS>` in your browser** the whole time:
```
aws autoscaling describe-instance-refreshes --auto-scaling-group-name saa-web-asg --max-items 1 --query "InstanceRefreshes[0].[Status,PercentageComplete]" --output text
```

**✅ You should see** a mix of "v1" and "**v2 - rolled out by instance refresh**" while it runs, then only v2, and **no errors at any point**. It takes ~5–8 minutes.

> 💡 `MinHealthyPercentage=50` means "replace at most half the fleet at a time." Production fleets use 90%+ for gentler rollouts.

---

### Step 9: Console Checkpoint

**✅ Checkpoint:**
1. **EC2 → Load Balancers → saa-web-alb → Resource map**: listener → target group → 2 healthy targets in 2 AZs.
2. **EC2 → Target Groups → saa-web-tg → Health checks**: `/health`, 10 s interval.
3. **EC2 → Auto Scaling Groups → saa-web-asg → Instance refresh**: completed successfully.

---

## What You Just Did

1. Put an **ALB** in front of a multi-AZ ASG with fast health checks
2. Chained security groups so instances accept traffic **only from the ALB**
3. Proved **ELB health checks** catch app failures that EC2 status checks miss
4. Shipped v2 with a **rolling instance refresh** and zero downtime

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Route `/api/*` and `/images/*` to different targets" or "host-based routing" | **ALB** listener rules |
| "Static IP addresses for the load balancer" / "whitelisting by IP" | **NLB** (Elastic IP per AZ) or **Global Accelerator** |
| "UDP", "millions of requests per second", "ultra-low latency" | **NLB** |
| "Deploy third-party firewall appliances transparently" | **Gateway Load Balancer** |
| "Users lose sessions when routed to different instances" | **Sticky sessions**, or better, externalize sessions to **ElastiCache/DynamoDB** |
| "Offload TLS from instances" | **ACM certificate on the ALB/NLB listener** |
| "In-flight requests dropped when an instance is removed" | **Deregistration delay** (connection draining) |

**🚨 Exam traps**
- Security group rules that reference **IP addresses** of instances break when instances are replaced. Reference **SG IDs**.
- ALBs **don't** have static IPs. If the requirement is a fixed IP, pick NLB or Global Accelerator.
- "Use Route 53 to load-balance across instances" is rarely the best answer for HA within a Region. ALB + ASG is.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| Targets stuck in `unhealthy` from the start | SG doesn't allow ALB → instance traffic, or `/health` is missing | Confirm Step 6's rule; check user data finished (wait 2 min) |
| `502 Bad Gateway` | No healthy targets, or httpd stopped | Check `describe-target-health` |
| `send-command` errors with `InvalidInstanceId` | The instance isn't registered with SSM | Wait 2 min after launch; the role from Lab 4A is required |
| Instance refresh `Failed` | New instances never got healthy | Check the v2 JSON was saved correctly, then roll back: `aws autoscaling cancel-instance-refresh --auto-scaling-group-name saa-web-asg` |

---

## 🧹 Cleanup

**⏸️ Continuing to Lab 4C?** Keep everything. Lab 4C cleans up all of Session 4.

**Stopping here?**
```
aws elbv2 delete-load-balancer --load-balancer-arn <ALB_ARN>
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name saa-web-asg --force-delete
aws ec2 delete-launch-template --launch-template-name saa-web-lt
```
Wait ~3 minutes, then:
```
aws elbv2 delete-target-group --target-group-arn <TG_ARN>
aws ec2 delete-security-group --group-id <WEB_SG_ID>
aws ec2 delete-security-group --group-id <ALB_SG_ID>
aws iam remove-role-from-instance-profile --instance-profile-name saa-web-profile --role-name saa-web-role
aws iam delete-instance-profile --instance-profile-name saa-web-profile
aws iam detach-role-policy --role-name saa-web-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name saa-web-role
```

---

**🏁 Lab complete: +100 XP.** Next: [Lab 4C — Auto Scaling Under Load](lab-4c-scaling-under-load.md)
