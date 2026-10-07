# Lab 4C: Auto Scaling Under Load — Target Tracking, Scheduled Scaling & Load Testing

**Session:** 4 — Resilient Compute  
**Exam Domain:** Domain 2 — Resilient (26%) · Domain 3 — High-Performing (24%)  
**Difficulty:** Advanced  
**Estimated Time:** 50–60 minutes (mostly watching graphs move)

---

## Overview

Self-healing keeps capacity *constant*. Real traffic isn't constant. In this lab you'll teach your fleet to **grow under load and shrink when it's quiet**, then hammer it with a CPU stress test and watch it scale out on its own.

**What you will build:**
- A **target tracking** policy that holds average CPU at 40%
- A **scheduled action** for a predictable morning peak
- A load test with `stress-ng` through **Systems Manager Run Command**
- A live **CloudWatch** view of the scaling decisions

---

## Prerequisites

- ✅ **Labs 4A and 4B** complete, with the ASG + ALB running and v2 deployed

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| EC2 t3.micro ×2–4 | Fleet scales up to 4 during the test | ~$0.0104/hr each |
| ALB + public IPv4 | From Lab 4B | ~$0.04/hr |
| CloudWatch alarms (created by target tracking) | Scaling triggers | First 10 alarms free |
| Systems Manager Run Command | Runs the stress test | Always Free |

**Estimated cost: ~$0.10 for the lab.** The Cleanup at the end removes **all** of Session 4.

---

## Concepts

| Scaling Type | How It Works | Best For |
|--------------|-------------|----------|
| **Target tracking** | "Keep metric X at value Y." AWS creates and manages the alarms. | Most workloads. **The default exam answer.** |
| **Step scaling** | "If CPU > 70%, add 2. If > 90%, add 4." You manage the alarms. | Fine-grained control over response size |
| **Simple scaling** | One alarm → one action → wait for cooldown | Legacy; avoid |
| **Scheduled** | "At 08:00 on weekdays, set min = 6" | **Predictable** peaks |
| **Predictive** | ML forecasts load from history and scales **ahead** of it | Recurring daily/weekly patterns |

**Good metrics to track:**
- `ASGAverageCPUUtilization`: CPU-bound apps
- `ALBRequestCountPerTarget`: web apps, where requests per instance is the truest load signal
- **SQS `ApproximateNumberOfMessagesVisible` per instance** (custom metric): queue workers (see Session 5)

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| From Labs 4A/4B | `<ALB_ARN>`, `<TG_ARN>`, `<WEB_SG_ID>`, `<ALB_SG_ID>`, `<ALB_DNS>` |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-4`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-4`

---

### Step 2: Create a Target Tracking Policy

**Create `target-tracking.json`:**
```json
{
  "TargetValue": 40.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ASGAverageCPUUtilization"
  }
}
```

📋 Attach it, with a 60-second instance warm-up:
```
aws autoscaling put-scaling-policy --auto-scaling-group-name saa-web-asg --policy-name saa-cpu-40 --policy-type TargetTrackingScaling --target-tracking-configuration file://target-tracking.json --estimated-instance-warmup 60
```

**✅ You should see** a `PolicyARN` and **two alarms** in the output (`AlarmHigh` and `AlarmLow`). You didn't create them; target tracking did.

📋 Look at them:
```
aws cloudwatch describe-alarms --alarm-name-prefix TargetTracking-saa-web-asg --query "MetricAlarms[].[AlarmName,Threshold,EvaluationPeriods,StateValue]" --output table
```

🔮 **Predict:** Look at `EvaluationPeriods` for AlarmHigh vs. AlarmLow. Why is one so much larger?

<details>
<summary>🔮 Reveal</summary>

**AlarmHigh ≈ 3 periods, AlarmLow ≈ 15 periods.** Target tracking **scales out fast and scales in slowly**. Adding capacity too late hurts users; removing it too early causes "flapping." That asymmetry is deliberate.

</details>

---

### Step 3: Open the Live View (Console)

Set up your "mission control" before the load starts:

1. **EC2 → Auto Scaling Groups → saa-web-asg → Monitoring** tab → **EC2** sub-tab. Watch **CPU Utilization (Percent)**.
2. In another tab: **CloudWatch → Alarms** → filter `TargetTracking-saa-web-asg`.
3. In a third tab: **saa-web-asg → Activity**.

---

### Step 4: Unleash the Load 🔥

🔮 **Predict before you start:** Desired capacity is 2, max is 4, and the target is 40% CPU. If both instances pin at ~100% CPU, how many instances will the ASG end up with?

📋 Run a **10-minute CPU stress test** on every instance in the ASG at once, targeting them **by tag** (no instance IDs needed):
```
aws ssm send-command --document-name AWS-RunShellScript --targets "Key=tag:aws:autoscaling:groupName,Values=saa-web-asg" --parameters "commands=stress-ng --cpu 0 --timeout 600s || timeout 600 sh -c 'yes > /dev/null & yes > /dev/null & wait'" --timeout-seconds 900 --query Command.CommandId --output text
```

> 💡 `--cpu 0` means "one worker per vCPU." The `||` fallback uses plain `yes` loops in case `stress-ng` didn't install.

Now **watch your three tabs** for 5–8 minutes:

| Minute | What You Should See |
|--------|---------------------|
| 0–2 | CPU climbs to ~100% on both instances |
| 2–4 | `AlarmHigh` → **In alarm** |
| 3–5 | Activity: *"Launching a new EC2 instance…"* Desired capacity → **3, then 4** |
| 5–8 | New instances register in the target group and turn `healthy` |

📋 CLI check, anytime:
```
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names saa-web-asg --query "AutoScalingGroups[0].[DesiredCapacity,length(Instances)]" --output text
```

<details>
<summary>🔮 Reveal</summary>

**4, the max.** The math (100% CPU on 2 instances, target 40%) asks for 5 instances, but `max-size = 4` caps it. Max size is your **cost guardrail**. Notice the **new** instances aren't running the stress test (it only targeted instances that existed when you sent it), so average CPU drops to ~50% and the fleet stabilizes.

</details>

### 🧩 Checkpoint

During a flash sale, new instances take **6 minutes** to boot and pass health checks, and users see errors during that time. Name **two** fixes.

<details>
<summary>Answer (+10 XP, both needed)</summary>

Any two of these:
1. **Bake a custom AMI** with everything preinstalled, so less user-data work happens at boot.
2. **Warm pools**: keep pre-initialized, stopped instances ready to start in seconds.
3. **Scheduled scaling** before the known sale starts.
4. **Predictive scaling** if the pattern recurs.
5. Lower the target value (e.g. 40% → 30%) for more headroom.

</details>

---

### Step 5: Add a Scheduled Action for a Known Peak

Marketing says traffic triples **every weekday at 08:00 UTC**. Don't wait for an alarm; be ready.

```
aws autoscaling put-scheduled-update-group-action --auto-scaling-group-name saa-web-asg --scheduled-action-name weekday-morning-peak --recurrence "0 8 * * 1-5" --min-size 3 --max-size 6 --desired-capacity 3
```
```
aws autoscaling describe-scheduled-actions --auto-scaling-group-name saa-web-asg --query "ScheduledUpdateGroupActions[].[ScheduledActionName,Recurrence,MinSize,DesiredCapacity]" --output table
```

> 💡 Scheduled actions and target tracking **work together**: the schedule raises the floor (`min`), and target tracking handles anything above it.

---

### Step 6: Watch Scale-In (Optional, ~15–20 min)

After the 10-minute stress test ends, CPU drops. `AlarmLow` needs ~15 consecutive low minutes, then the ASG **terminates** instances back down to the minimum of 2.

🔮 **Predict:** When scaling in from 4 → 2, **which** instances get terminated first by default?

<details>
<summary>🔮 Reveal</summary>

The **default termination policy** first balances AZs. Within the AZ it picks, it prefers instances with the **oldest launch template/configuration**, then the instance **closest to the next billing hour**. You can protect specific instances with **scale-in protection**, or run cleanup code before termination with **lifecycle hooks**.

</details>

---

### Step 7: Console Checkpoint

**✅ Checkpoint:**
1. **saa-web-asg → Automatic scaling**: the target tracking policy **and** the scheduled action.
2. **Activity history**: your launches (and terminations, if you waited).
3. **CloudWatch → Alarms**: both TargetTracking alarms, with history graphs showing the spike.

---

## ⚔️ Boss Challenge: Black Friday Architecture (+250 XP)

No commands. Write down your design.

**Scenario:** An e-commerce site on an ALB + ASG normally runs 4 instances. On Black Friday, traffic jumps **20× at exactly 00:00** and stays high for 6 hours. Order processing (payment + inventory) can take up to 30 seconds per order, and **no orders may be lost** even if the web tier is overwhelmed. Instances take 5 minutes to become healthy.

Design the scaling **and** the order-processing path.

<details>
<summary>🏆 Answer key</summary>

**Web tier**
- **Scheduled action** at ~23:30 raising `min`/`desired` to the expected peak (20× = 80 instances), since boot takes 5 minutes and the spike is instant. (+60)
- **Target tracking** on `ALBRequestCountPerTarget` on top of the schedule, for anything beyond the forecast. (+40)
- **Warm pool** and/or a pre-baked **AMI** to cut the 5-minute boot. (+30)
- Raise the ASG **max** and confirm **EC2 service quotas** (vCPU limits) in advance. (+20)

**Order path (decoupling)**
- The web tier writes orders to **Amazon SQS** and returns immediately. No orders are lost even when workers lag. (+60)
- A separate **worker ASG** (or Lambda) consumes the queue and scales on **queue depth per instance** (backlog per instance). (+40)
- A **DLQ** for orders that repeatedly fail. You'll build all of this in Session 5.

</details>

---

## What You Just Did

1. Created a **target tracking** policy and saw AWS generate its alarms
2. Load-tested the fleet and watched it **scale out automatically**, capped by max size
3. Added **scheduled scaling** for a predictable peak
4. Learned how scale-in chooses which instances to terminate

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Scale to keep CPU at 50%", "simplest scaling" | **Target tracking** |
| "Traffic spikes every Monday 9 AM" | **Scheduled** (or predictive) scaling |
| "Recurring patterns, scale **before** load arrives" | **Predictive scaling** |
| "Workers process an SQS queue; scale on backlog" | Target tracking on **backlog per instance** (custom metric) |
| "Run a script before an instance is terminated (drain logs)" | **Lifecycle hook** |
| "Instances must boot faster" | **Golden AMI**, **warm pools** |

**🚨 Exam traps**
- Scaling on **memory** requires the **CloudWatch agent**. EC2 doesn't publish memory metrics by default.
- Basic monitoring = **5-minute** metrics. Detailed = **1-minute** (paid). Faster scaling needs detailed monitoring.
- Vertical scaling (a bigger instance type) needs a stop/start, which means **downtime**. Horizontal scaling (more instances) doesn't.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| CPU never rises | The command didn't run | **Systems Manager → Run Command → Command history**: check the output per instance |
| `InvalidInstanceId` / no targets | Instances aren't SSM-registered | Confirm the `saa-web-profile` role exists and instances are < 2 min old |
| Alarm stays `INSUFFICIENT_DATA` | Metrics not flowing yet | Wait 2–3 minutes; detailed monitoring must be on (Lab 4A launch template) |
| Didn't scale beyond 2 | Policy not attached | `aws autoscaling describe-policies --auto-scaling-group-name saa-web-asg` |

---

## 🧹 Cleanup — All of Session 4

**1. Delete the load balancer and the ASG** (this terminates all instances):
```
aws elbv2 delete-load-balancer --load-balancer-arn <ALB_ARN>
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name saa-web-asg --force-delete
```

> 💡 Deleting the ASG also deletes its scaling policies, scheduled actions and the target tracking alarms.

**2. Delete the launch template:**
```
aws ec2 delete-launch-template --launch-template-name saa-web-lt
```

**3. Wait ~3 minutes** for instances and ALB network interfaces to go away, then:
```
aws elbv2 delete-target-group --target-group-arn <TG_ARN>
aws ec2 delete-security-group --group-id <WEB_SG_ID>
aws ec2 delete-security-group --group-id <ALB_SG_ID>
```
If a security group delete fails with `DependencyViolation`, wait another minute and retry.

**4. IAM:**
```
aws iam remove-role-from-instance-profile --instance-profile-name saa-web-profile --role-name saa-web-role
aws iam delete-instance-profile --instance-profile-name saa-web-profile
aws iam detach-role-policy --role-name saa-web-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name saa-web-role
```

**5. Local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-session-4`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-session-4`

**✅ Checkpoint:** **EC2 → Instances** shows all `saa-web` instances **Terminated**. **Load Balancers** and **Auto Scaling Groups** are empty. **CloudWatch → Alarms** has no `TargetTracking-saa-web-asg` alarms.

---

**🏁 Lab complete: +100 XP.** Session 4 done! **[📝 Mini Exam 4 →](mini-exam-04.md)**
