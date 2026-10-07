# Lab 6B: DynamoDB Design Patterns — Keys, GSIs, Query vs. Scan & Conditional Writes

**Session:** 6 — Databases & Disaster Recovery  
**Exam Domain:** Domain 3 — High-Performing (24%) · Domain 4 — Cost-Optimized (20%)  
**Difficulty:** Intermediate  
**Estimated Time:** 40–50 minutes

---

## Overview

DynamoDB is the exam's go-to answer for *"single-digit millisecond latency at any scale," "serverless database," "key-value,"* and *"session state."* To use it well, though, you design **around your access patterns**, not around normalized tables.

You'll build a game leaderboard, query it efficiently, watch a **Scan** waste capacity, protect high scores with **conditional writes**, and turn on **TTL** and **point-in-time recovery**.

**What you will build:**

```
 Table: saa-game-scores              GSI: GameTitleIndex
 ┌──────────┬────────────────┬───────────┐   ┌────────────────┬───────────┬──────────┐
 │ UserId PK│ GameTitle SK   │ TopScore  │   │ GameTitle PK   │ TopScore SK│ UserId  │
 ├──────────┼────────────────┼───────────┤   ├────────────────┼───────────┼──────────┤
 │ u101     │ MeteorBlasters │ 5842      │   │ MeteorBlasters │ 9120      │ u102     │
 │ u101     │ StarshipX      │ 2210      │ ⇒ │ MeteorBlasters │ 7300      │ u103     │
 │ u102     │ MeteorBlasters │ 9120      │   │ MeteorBlasters │ 5842      │ u101     │
 │ …        │                │           │   │ …              │           │          │
 └──────────┴────────────────┴───────────┘   └────────────────┴───────────┴──────────┘
 "All games for a player"                      "Top scores for a game"
```

---

## Prerequisites

- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| DynamoDB on-demand | Table + GSI | Fractions of a cent for this lab |
| Point-in-time recovery | Continuous backups | ~$0.20/GB-month (this table is a few KB ≈ $0) |

**Estimated cost for this lab: $0.00**

---

## Concepts

| Term | Meaning |
|------|---------|
| **Partition key (PK)** | Hashed to choose the storage partition. Must be **high-cardinality** to spread load. |
| **Sort key (SK)** | Orders items **within** a partition and enables range queries (`begins_with`, `between`, `>`) |
| **Query** | Reads **one partition** by key. Efficient. |
| **Scan** | Reads **the entire table**, then filters. Expensive at scale. |
| **GSI** | Global secondary index: a different PK/SK, can be added anytime, eventually consistent |
| **LSI** | Local secondary index: same PK, different SK, **only at table creation**, strongly consistent reads possible |
| **On-demand vs. provisioned** | Pay-per-request (spiky, unknown traffic) vs. set RCU/WCU, optionally with auto scaling (steady, predictable traffic, cheaper) |

---

## ⚠️ Placeholders in This Lab

None! Just `<YOUR_PROFILE_NAME>`.

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-session-6; cd ~\Desktop\saa-session-6; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-session-6 && cd ~/Desktop/saa-session-6 && code .
```

---

### Step 2: Create the Table with a GSI

**Create `gsi.json`:**
```json
[
  {
    "IndexName": "GameTitleIndex",
    "KeySchema": [
      { "AttributeName": "GameTitle", "KeyType": "HASH" },
      { "AttributeName": "TopScore", "KeyType": "RANGE" }
    ],
    "Projection": { "ProjectionType": "ALL" }
  }
]
```

```
aws dynamodb create-table --table-name saa-game-scores --attribute-definitions AttributeName=UserId,AttributeType=S AttributeName=GameTitle,AttributeType=S AttributeName=TopScore,AttributeType=N --key-schema AttributeName=UserId,KeyType=HASH AttributeName=GameTitle,KeyType=RANGE --global-secondary-indexes file://gsi.json --billing-mode PAY_PER_REQUEST
```
```
aws dynamodb wait table-exists --table-name saa-game-scores
```

> 💡 You only declare attributes used in **keys** (table or index). DynamoDB is schemaless for everything else.

---

### Step 3: Load Data

**Create `scores.json`:**
```json
{
  "saa-game-scores": [
    { "PutRequest": { "Item": { "UserId": {"S": "u101"}, "GameTitle": {"S": "MeteorBlasters"}, "TopScore": {"N": "5842"}, "Wins": {"N": "10"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u101"}, "GameTitle": {"S": "StarshipX"},      "TopScore": {"N": "2210"}, "Wins": {"N": "3"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u102"}, "GameTitle": {"S": "MeteorBlasters"}, "TopScore": {"N": "9120"}, "Wins": {"N": "21"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u102"}, "GameTitle": {"S": "AlienInvaders"},  "TopScore": {"N": "4400"}, "Wins": {"N": "7"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u103"}, "GameTitle": {"S": "MeteorBlasters"}, "TopScore": {"N": "7300"}, "Wins": {"N": "15"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u103"}, "GameTitle": {"S": "StarshipX"},      "TopScore": {"N": "8800"}, "Wins": {"N": "12"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u104"}, "GameTitle": {"S": "MeteorBlasters"}, "TopScore": {"N": "1200"}, "Wins": {"N": "1"} } } },
    { "PutRequest": { "Item": { "UserId": {"S": "u104"}, "GameTitle": {"S": "AlienInvaders"},  "TopScore": {"N": "9900"}, "Wins": {"N": "30"} } } }
  ]
}
```

```
aws dynamodb batch-write-item --request-items file://scores.json
```

**✅ You should see** `"UnprocessedItems": {}`.

---

### Step 4: Access Pattern 1 — "All Games for Player u101" (Query on the Table)

**Create `vals-user.json`:**
```json
{ ":u": { "S": "u101" } }
```

```
aws dynamodb query --table-name saa-game-scores --key-condition-expression "UserId = :u" --expression-attribute-values file://vals-user.json --query "Items[].[GameTitle.S,TopScore.N]" --output table
```

**✅ You should see** MeteorBlasters and StarshipX.

---

### Step 5: Access Pattern 2 — "Top 3 Scores for MeteorBlasters" (Query on the GSI)

**Create `vals-game.json`:**
```json
{ ":g": { "S": "MeteorBlasters" } }
```

🔮 **Predict:** Who's #1, #2, #3?

```
aws dynamodb query --table-name saa-game-scores --index-name GameTitleIndex --key-condition-expression "GameTitle = :g" --expression-attribute-values file://vals-game.json --no-scan-index-forward --limit 3 --query "Items[].[UserId.S,TopScore.N]" --output table
```

<details>
<summary>🔮 Reveal</summary>

**u102 (9120), u103 (7300), u101 (5842).** The GSI's sort key `TopScore` keeps items ordered, and `--no-scan-index-forward` reads them **descending**. No sorting happens in your app; the index *is* the leaderboard.

</details>

---

### Step 6: Query vs. Scan — Feel the Cost

📋 The **wrong** way to answer the same question, scanning everything:
```
aws dynamodb scan --table-name saa-game-scores --filter-expression "GameTitle = :g" --expression-attribute-values file://vals-game.json --return-consumed-capacity TOTAL --query "{Returned:Count,Scanned:ScannedCount,Capacity:ConsumedCapacity.CapacityUnits}"
```

📋 The **right** way:
```
aws dynamodb query --table-name saa-game-scores --index-name GameTitleIndex --key-condition-expression "GameTitle = :g" --expression-attribute-values file://vals-game.json --return-consumed-capacity TOTAL --query "{Returned:Count,Scanned:ScannedCount,Capacity:ConsumedCapacity.CapacityUnits}"
```

**✅ You should see** the scan **read all 8 items** to return 4, while the query reads only 4.

> 💡 With 8 items the difference is tiny. With **800 million** items, a scan reads all 800 million and **you pay for every one**, even though the filter discards most of them. 🚨 Exam answers that use **Scan + FilterExpression** for a frequent access pattern are wrong. Add a **GSI** and use Query.

### 🧩 Checkpoint

A table uses `Status` (values: `OPEN` or `CLOSED`) as its partition key and gets throttled even though total capacity is high. Why?

<details>
<summary>Answer (+10 XP)</summary>

**Hot partition.** With only two key values, all traffic lands on very few partitions. Each partition has throughput limits no matter how much table-level capacity you have. Choose a **high-cardinality** partition key (`orderId`, `userId`), or add a random suffix (**write sharding**).

</details>

---

### Step 7: Conditional Writes — Only Save a *Higher* Score

**Create `key-u101.json`:**
```json
{ "UserId": { "S": "u101" }, "GameTitle": { "S": "MeteorBlasters" } }
```
**Create `vals-low.json`:**
```json
{ ":s": { "N": "3000" } }
```
**Create `vals-high.json`:**
```json
{ ":s": { "N": "6500" } }
```

🔮 **Predict:** u101's current top score is 5842. They just scored **3000**. What happens?

```
aws dynamodb update-item --table-name saa-game-scores --key file://key-u101.json --update-expression "SET TopScore = :s" --condition-expression "TopScore < :s" --expression-attribute-values file://vals-low.json
```

<details>
<summary>🔮 Reveal</summary>

❌ **`ConditionalCheckFailedException`.** The write is rejected **atomically on the server**. No read-then-write race condition, even with thousands of concurrent players.

</details>

📋 Now a real high score:
```
aws dynamodb update-item --table-name saa-game-scores --key file://key-u101.json --update-expression "SET TopScore = :s" --condition-expression "TopScore < :s" --expression-attribute-values file://vals-high.json --return-values UPDATED_NEW
```

**✅ You should see** `TopScore: 6500`.

📋 **Atomic counter**: add a win without reading first. Create `vals-one.json`:
```json
{ ":one": { "N": "1" } }
```
```
aws dynamodb update-item --table-name saa-game-scores --key file://key-u101.json --update-expression "ADD Wins :one" --expression-attribute-values file://vals-one.json --return-values UPDATED_NEW
```

**✅ You should see** `Wins: 11`.

---

### Step 8: TTL and Point-in-Time Recovery

📋 **TTL**: items with an `expiresAt` attribute (Unix epoch seconds) in the past are deleted automatically, **for free**:
```
aws dynamodb update-time-to-live --table-name saa-game-scores --time-to-live-specification "Enabled=true,AttributeName=expiresAt"
```

> 💡 Perfect for **session data**, temporary tokens and old logs. Expired items are typically removed within a few days, and deletes don't consume write capacity.

📋 **Point-in-time recovery (PITR)**: restore to any second in the last 35 days:
```
aws dynamodb update-continuous-backups --table-name saa-game-scores --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true
```
```
aws dynamodb describe-continuous-backups --table-name saa-game-scores --query "ContinuousBackupsDescription.PointInTimeRecoveryDescription.[PointInTimeRecoveryStatus,EarliestRestorableDateTime]"
```

**✅ You should see** `ENABLED`. You'll use backups in Lab 6C.

---

### Step 9: Console Checkpoint

**✅ Checkpoint:**
1. **DynamoDB → Tables → saa-game-scores → Explore table items**: browse, then switch to the **GameTitleIndex** and run a query in the UI.
2. **Indexes** tab shows the GSI.
3. **Backups** tab shows PITR enabled. **Additional settings** shows TTL on `expiresAt` and the capacity mode **On-demand**.

---

## What You Just Did

1. Designed keys **from access patterns**: table for "by player," GSI for "by game, sorted by score"
2. Measured **Query vs. Scan** capacity
3. Used **conditional writes** and **atomic counters** to avoid race conditions
4. Enabled **TTL** and **PITR**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Microsecond read latency for DynamoDB" | **DAX** (DynamoDB Accelerator) |
| "Multi-Region, active-active, low-latency writes everywhere" | **DynamoDB global tables** |
| "Unpredictable traffic, pay per request" | **On-demand** capacity |
| "Steady, predictable traffic, lowest cost" | **Provisioned + auto scaling** (+ reserved capacity) |
| "React to every item change (trigger Lambda)" | **DynamoDB Streams** |
| "Automatically delete expired session data" | **TTL** |
| "Restore table to the state 10 minutes ago" | **PITR** |
| "Store user session state for a stateless web tier" | **DynamoDB** or **ElastiCache** |

**🚨 Exam traps**
- **LSIs** can only be created **with the table**. GSIs can be added anytime.
- DynamoDB's item size limit is **400 KB**. Store large blobs in S3 and keep a pointer.
- DAX is for **DynamoDB**. ElastiCache is the general-purpose cache (RDS, sessions, anything).

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `ValidationException ... attribute definitions` | Declared an attribute that isn't used in a key, or missed one | Only `UserId`, `GameTitle`, `TopScore` belong in `--attribute-definitions` |
| GSI query returns nothing right after loading | GSIs are **eventually consistent** | Wait a second and retry |
| `Error parsing parameter '--expression-attribute-values'` | File missing or wrong folder | Confirm with `ls` / `dir` that the `vals-*.json` file exists |

---

## 🧹 Cleanup

**⏸️ Continuing to Lab 6C?** Lab 6C creates its own table, so you can clean up now:
```
aws dynamodb delete-table --table-name saa-game-scores
```

> Keep the `saa-session-6` folder for Lab 6C.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 6C — Backup & DR Strategies](lab-6c-backup-dr-strategies.md)
