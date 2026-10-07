# Lab 6A: RDS Multi-AZ vs. Read Replicas — Failover You Can Watch

**Session:** 6 — Databases & Disaster Recovery  
**Exam Domain:** Domain 2 — Resilient (26%) · Domain 3 — High-Performing (24%)  
**Difficulty:** Beginner  
**Estimated Time:** 60–75 minutes (≈ 40 of them waiting on RDS, so read the Concepts while you wait)

---

## Overview

Two RDS features sound alike and solve completely different problems. The exam tests the difference **constantly**:

| | **Multi-AZ** | **Read Replica** |
|-|--------------|------------------|
| Solves | **Availability** (survive an AZ or instance failure) | **Read performance** (offload SELECTs) |
| Replication | **Synchronous** | **Asynchronous** (replica lag) |
| Can you query the standby? | ❌ No* | ✅ Yes, it has its own endpoint |
| Failover | **Automatic**, same DNS endpoint, ~60–120 s | Manual **promotion** (becomes a standalone DB) |
| Cross-Region? | ❌ | ✅ (also a DR strategy) |

<sub>*Multi-AZ **DB cluster** deployments (with two readable standbys) do allow reads, but the classic Multi-AZ instance deployment tested on most questions doesn't.</sub>

In this lab you'll build **both**, then **force a failover** and prove that the endpoint name never changes.

**What you will build:**

```
                    saa-db.xxxx.us-east-1.rds.amazonaws.com  (writer endpoint - never changes)
                                │
          ┌─────────────────────┴─────────────────────┐
          ▼  sync replication                          ▼
   ┌───────────────┐                          ┌───────────────┐
   │ Primary (AZ a)│ ◀──── forced failover ──▶│ Standby (AZ b)│  (not readable)
   └───────┬───────┘                          └───────────────┘
           │ async replication
           ▼
   ┌───────────────────────┐
   │ saa-db-replica        │ ← its own endpoint, read-only queries
   └───────────────────────┘
```

---

## Prerequisites

- ✅ Session 3 complete (the DB password lands in **Secrets Manager** automatically)
- ✅ Default VPC present in us-east-1

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| RDS db.t4g.micro (Single-AZ) | Primary | ~$0.016/hr |
| …after converting to Multi-AZ | Primary + standby | ~$0.032/hr |
| RDS read replica db.t4g.micro | Replica | ~$0.016/hr |
| gp3 storage 20 GB × copies | Storage | ~$0.003/hr per copy |
| Secrets Manager (RDS-managed password) | Master password | $0.40/month, prorated (~$0.001) |

**Estimated cost for this lab: ~$0.10–0.15** if you delete everything at the end. Older accounts still on the 12-month RDS free tier may pay even less.

> ⚠️ **Don't leave this running overnight.** A forgotten Multi-AZ instance + replica costs ~$35/month.

---

## Concepts

**Scaling an RDS database (exam cheat-sheet):**

| Problem | Fix |
|---------|-----|
| Too many **reads** | **Read replicas** (up to 15 for MySQL/PostgreSQL/MariaDB) and/or **ElastiCache** |
| Too many **writes** | **Scale up** (bigger instance class), or move to **Aurora** / shard / DynamoDB |
| Storage filling up | **Storage auto scaling** |
| Need HA | **Multi-AZ** |
| Need DR in another Region | **Cross-Region read replica** (promote in a disaster) or **cross-Region automated backups** |

**Aurora in one paragraph.** AWS's cloud-native MySQL/PostgreSQL-compatible engine. Storage is automatically replicated **6 ways across 3 AZs**, there are up to **15 low-lag replicas** that double as failover targets, there's a **reader endpoint** that load-balances replicas, **Aurora Serverless v2** auto-scales capacity, and **Aurora Global Database** gives < 1 s cross-Region replication with fast regional failover.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<YOUR_PROFILE_NAME>` | Your CLI profile |

All other names are fixed: `saa-db` and `saa-db-replica`.

---

## Lab Steps

### Step 1: Profile

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>"`

---

### Step 2: Create a Single-AZ MySQL Instance

```
aws rds create-db-instance --db-instance-identifier saa-db --engine mysql --db-instance-class db.t4g.micro --allocated-storage 20 --storage-type gp3 --master-username saaadmin --manage-master-user-password --no-publicly-accessible --no-multi-az --backup-retention-period 1 --query "DBInstance.DBInstanceStatus" --output text
```

**✅ You should see** `creating`.

> 💡 **What the flags mean:**
> - `--manage-master-user-password` → RDS generates the password and stores it in **Secrets Manager** with rotation support. You never see or type it. (Remember Lab 3C?)
> - `--no-publicly-accessible` → no public IP. Databases belong in private subnets.
> - `--backup-retention-period 1` → automated backups on. **Required** for read replicas.

📋 Wait (~8–12 minutes). ☕ Read the Concepts section while it runs.
```
aws rds wait db-instance-available --db-instance-identifier saa-db
```

📋 Inspect it:
```
aws rds describe-db-instances --db-instance-identifier saa-db --query "DBInstances[0].{Endpoint:Endpoint.Address,AZ:AvailabilityZone,MultiAZ:MultiAZ,Secret:MasterUserSecret.SecretArn}" --output json
```

> **📝 Write down the AZ:** ______________ (you'll compare it after failover)

---

### Step 3: Add a Read Replica

🔮 **Predict:** Will the replica have the **same** endpoint hostname as the primary, or a different one?

```
aws rds create-db-instance-read-replica --db-instance-identifier saa-db-replica --source-db-instance-identifier saa-db --db-instance-class db.t4g.micro --query "DBInstance.DBInstanceStatus" --output text
```
```
aws rds wait db-instance-available --db-instance-identifier saa-db-replica
```
```
aws rds describe-db-instances --query "DBInstances[?starts_with(DBInstanceIdentifier,'saa-db')].[DBInstanceIdentifier,Endpoint.Address,AvailabilityZone,ReadReplicaSourceDBInstanceIdentifier]" --output table
```

<details>
<summary>🔮 Reveal</summary>

**Different.** `saa-db-replica.xxxx…` has its **own endpoint**. Your app must deliberately send **read** queries to it, for example a reporting dashboard reading from the replica while the main app writes to the primary. The replica may be **slightly behind** (asynchronous replication lag).

</details>

### 🧩 Checkpoint

A reporting job running heavy `SELECT` queries makes the production database slow for customers. Is Multi-AZ the fix?

<details>
<summary>Answer (+10 XP)</summary>

**No.** The classic Multi-AZ standby **can't serve reads**. Point the reporting job at a **read replica**. (Or, on Aurora, use the **reader endpoint**.)

</details>

---

### Step 4: Convert the Primary to Multi-AZ

```
aws rds modify-db-instance --db-instance-identifier saa-db --multi-az --apply-immediately --query "DBInstance.PendingModifiedValues"
```

**✅ You should see** `"MultiAZ": true` under pending changes.

> 💡 This conversion happens **online**, with no downtime. RDS snapshots the primary, builds a standby in another AZ from it, and starts synchronous replication.

📋 Wait (~10–15 minutes):
```
aws rds wait db-instance-available --db-instance-identifier saa-db
```
```
aws rds describe-db-instances --db-instance-identifier saa-db --query "DBInstances[0].[MultiAZ,AvailabilityZone,SecondaryAvailabilityZone]" --output text
```

**✅ You should see** `True  us-east-1x  us-east-1y`: two different AZs.

> ⚠️ If `wait` returns immediately but `MultiAZ` still shows `False`, the change hasn't started yet. Wait 1 minute and run the `wait` again.

---

### Step 5: Pull the Plug — Forced Failover 💥

🔮 **Predict** before running: (a) Will the endpoint hostname change? (b) Roughly how long will the database be unavailable? (c) What happens to the AZ values?

```
aws rds reboot-db-instance --db-instance-identifier saa-db --force-failover --query "DBInstance.DBInstanceStatus" --output text
```
```
aws rds wait db-instance-available --db-instance-identifier saa-db
```
```
aws rds describe-db-instances --db-instance-identifier saa-db --query "DBInstances[0].[Endpoint.Address,AvailabilityZone,SecondaryAvailabilityZone]" --output text
```

📋 Read the event log, which tells the story:
```
aws rds describe-events --source-identifier saa-db --source-type db-instance --duration 30 --query "Events[].[Date,Message]" --output table
```

<details>
<summary>🔮 Reveal</summary>

(a) **Same endpoint.** RDS flips the DNS record (CNAME) to the standby. Apps reconnect to the same hostname.  
(b) Typically **60–120 seconds**. The events show *"Multi-AZ instance failover started"* … *"completed"*.  
(c) **The AZs swapped.** The old standby is now the primary.

**Exam angle:** apps should use the **endpoint DNS name** (never an IP) and have **connection retry logic**. Don't let your app (or the JVM) cache DNS for a long time.

</details>

---

### Step 6: Promotion = Disaster Recovery (Read Only — No Need to Run)

If the primary's Region failed and your replica were **cross-Region**, you'd run:
```
aws rds promote-read-replica --db-instance-identifier saa-db-replica
```
It breaks replication and turns the replica into a **standalone, writable** database. That's a manual (or scripted) step. 🚨 The exam may ask: *"Which option provides automatic failover?"* → Multi-AZ, **not** read replicas.

---

### Step 7: Console Checkpoint

**✅ Checkpoint:**
1. **RDS → Databases** shows `saa-db` (Role: *Primary*, Multi-AZ: *Yes*) and `saa-db-replica` (Role: *Replica*).
2. Click `saa-db` → **Configuration** shows the Secrets Manager ARN for the master credentials.
3. **Logs & events** tab shows the failover events.
4. **Secrets Manager** lists a secret named `rds!db-…`, created and managed by RDS.

---

## What You Just Did

1. Launched RDS with a **Secrets Manager-managed** master password
2. Added a **read replica** with its own endpoint
3. Converted to **Multi-AZ** online
4. **Forced a failover** and verified the endpoint stayed the same

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Automatic failover", "high availability", "survive AZ failure" | **Multi-AZ** |
| "Offload read traffic", "reporting queries slow production" | **Read replica** |
| "DR in another Region with low RPO for RDS" | **Cross-Region read replica** (promote on disaster) |
| "Fastest cross-Region failover, < 1 s replication" | **Aurora Global Database** |
| "Unpredictable / intermittent database load, minimal management" | **Aurora Serverless v2** |
| "Too many DB connections from Lambda" | **RDS Proxy** |
| "Read-heavy, same queries repeated, microsecond latency" | **ElastiCache** in front of RDS |

**🚨 Exam traps**
- Read replicas do **not** provide automatic failover (for the classic RDS engines).
- Multi-AZ standby is **not** for read scaling.
- You can't connect to the **standby** directly, and you don't need to. The endpoint follows the primary.
- Read replicas need **automated backups enabled** on the source.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InvalidParameterCombination: ... t4g.micro` | Instance class not available for the engine version | Use `--db-instance-class db.t3.micro` instead |
| `wait` times out after 30 min | RDS is still working | Run `wait` again; check status with `describe-db-instances` |
| `InvalidDBInstanceState` on modify/reboot | The instance is busy (`modifying`, `backing-up`) | Wait until `available`, then retry |
| Read replica creation fails | Backups disabled | `aws rds modify-db-instance --db-instance-identifier saa-db --backup-retention-period 1 --apply-immediately` |

---

## 🧹 Cleanup

📋 Delete the replica first, then the primary:
```
aws rds delete-db-instance --db-instance-identifier saa-db-replica --skip-final-snapshot
```
```
aws rds delete-db-instance --db-instance-identifier saa-db --skip-final-snapshot --delete-automated-backups
```
```
aws rds wait db-instance-deleted --db-instance-identifier saa-db
```

> 💡 The RDS-managed Secrets Manager secret is deleted automatically with the instance.

**✅ Checkpoint:** **RDS → Databases** is empty. **RDS → Snapshots → System** shows no `saa-db` snapshots. **Secrets Manager** has no `rds!db-…` secret (it may take a few minutes to disappear).

---

**🏁 Lab complete: +100 XP.** Next: [Lab 6B — DynamoDB Design Patterns](lab-6b-dynamodb-design.md)
