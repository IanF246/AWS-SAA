# Lab 5C: Event-Driven Orchestration — EventBridge + Step Functions + DynamoDB

**Session:** 5 — Decoupled Architectures  
**Exam Domain:** Domain 2 — Resilient (26%) · Domain 3 — High-Performing (24%)  
**Difficulty:** Advanced  
**Estimated Time:** 50–60 minutes

---

## Overview

Queues and topics move messages. But business processes have **steps, decisions and retries**: "if the order is over $1,000, flag it for review; otherwise approve it; then save it; retry if the database blips." Coding that logic into Lambda functions that call each other is the anti-pattern the exam warns against.

In this lab you'll build a **serverless, zero-code workflow**:
- **Amazon EventBridge** receives business events on a custom event bus and routes them by **content**.
- **AWS Step Functions** runs a visual state machine with a **Choice**, a **Retry**, and a **direct DynamoDB integration**. No Lambda needed.

**What you will build:**

```
 put-events ─▶ EventBridge bus: saa-orders-bus
                 rule: source = saa.shop AND detail-type = OrderPlaced
                         │
                         ▼
               Step Functions: saa-order-workflow
               ┌──────────────┐  amount ≥ 1000  ┌───────────────┐
               │ CheckAmount  │ ──────────────▶ │ FlagForReview │──┐
               │  (Choice)    │ ──otherwise───▶ │ AutoApprove   │──┤
               └──────────────┘                 └───────────────┘  ▼
                                                     ┌──────────────────────────┐
                                                     │ SaveOrder (DynamoDB      │
                                                     │ PutItem, Retry ×3)       │
                                                     └──────────────────────────┘
```

---

## Prerequisites

- ✅ **Labs 5A and 5B** complete

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon EventBridge (custom events) | Event bus + rules | $1.00 per million events (this lab: ~10 events ≈ $0) |
| AWS Step Functions (Standard) | Workflow engine | 4,000 state transitions/month free |
| Amazon DynamoDB (on-demand) | Orders table | Free tier / fractions of a cent |

**Estimated cost for this lab: $0.00**

---

## Concepts

| Service | Think Of It As | Pick It When |
|---------|---------------|--------------|
| **SQS** | A buffer | One consumer group must process work reliably at its own pace |
| **SNS** | A megaphone | Push the same message to many subscribers |
| **EventBridge** | A smart router | Route events **by content** to 20+ target types; events from AWS services and **SaaS partners**; **archive & replay**; scheduling |
| **Step Functions** | A conductor | Multi-step workflows with branching, retries, parallelism, human approval, and a visual audit trail |

**Step Functions workflow types:**

| Standard | Express |
|----------|---------|
| Up to **1 year** | Up to **5 minutes** |
| Exactly-once execution | At-least-once |
| Priced per state transition | Priced per request + duration |
| Long business processes, human approval | High-volume event processing, IoT, streaming |

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<ACCOUNT_ID>` | Your account ID |
| `<SFN_ARN>` | State machine ARN (Step 4) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-5`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-5`

---

### Step 2: Create the DynamoDB Table

```
aws dynamodb create-table --table-name saa-orders --attribute-definitions AttributeName=orderId,AttributeType=S --key-schema AttributeName=orderId,KeyType=HASH --billing-mode PAY_PER_REQUEST
```
```
aws dynamodb wait table-exists --table-name saa-orders
```

---

### Step 3: Role for Step Functions

**Create `sfn-trust.json`:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "states.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Create `sfn-policy.json`**, **replacing `<ACCOUNT_ID>`**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "dynamodb:PutItem",
      "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/saa-orders"
    }
  ]
}
```

```
aws iam create-role --role-name saa-sfn-role --assume-role-policy-document file://sfn-trust.json
```
```
aws iam put-role-policy --role-name saa-sfn-role --policy-name put-orders --policy-document file://sfn-policy.json
```

---

### Step 4: Define and Create the State Machine

**Create `workflow.asl.json`** (Amazon States Language):
```json
{
  "Comment": "SAA order workflow - no Lambda required",
  "StartAt": "CheckAmount",
  "States": {
    "CheckAmount": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.detail.amount",
          "NumericGreaterThanEquals": 1000,
          "Next": "FlagForReview"
        }
      ],
      "Default": "AutoApprove"
    },
    "FlagForReview": {
      "Type": "Pass",
      "Result": "REVIEW",
      "ResultPath": "$.status",
      "Next": "SaveOrder"
    },
    "AutoApprove": {
      "Type": "Pass",
      "Result": "APPROVED",
      "ResultPath": "$.status",
      "Next": "SaveOrder"
    },
    "SaveOrder": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "saa-orders",
        "Item": {
          "orderId": { "S.$": "$.detail.orderId" },
          "amount": { "N.$": "States.JsonToString($.detail.amount)" },
          "status": { "S.$": "$.status" }
        }
      },
      "Retry": [
        {
          "ErrorEquals": ["States.ALL"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "End": true
    }
  }
}
```

> 💡 **What's notable here:**
> - `arn:aws:states:::dynamodb:putItem` is a **direct service integration**: Step Functions calls DynamoDB itself. No Lambda, no code, no cold starts.
> - `Retry` with `BackoffRate: 2` = wait 2 s, 4 s, 8 s. **Exponential backoff** is built in.

📋 Create it, **replacing `<ACCOUNT_ID>`**:
```
aws stepfunctions create-state-machine --name saa-order-workflow --definition file://workflow.asl.json --role-arn arn:aws:iam::<ACCOUNT_ID>:role/saa-sfn-role --type STANDARD --query stateMachineArn --output text
```

> **📝 Save as `<SFN_ARN>`**
>
> If you get `AccessDeniedException ... not authorized to assume`, wait 10 seconds (IAM propagation) and retry.

---

### Step 5: Custom Event Bus and Rule

```
aws events create-event-bus --name saa-orders-bus
```

**Create `rule-pattern.json`.** This is the event pattern, i.e. which events match:
```json
{
  "source": ["saa.shop"],
  "detail-type": ["OrderPlaced"]
}
```

```
aws events put-rule --name saa-order-placed --event-bus-name saa-orders-bus --event-pattern file://rule-pattern.json
```

**EventBridge needs permission to start the workflow.** Create `events-trust.json`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "events.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create `events-policy.json`, **replacing `<SFN_ARN>`**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "states:StartExecution",
      "Resource": "<SFN_ARN>"
    }
  ]
}
```

```
aws iam create-role --role-name saa-events-to-sfn-role --assume-role-policy-document file://events-trust.json
```
```
aws iam put-role-policy --role-name saa-events-to-sfn-role --policy-name start-workflow --policy-document file://events-policy.json
```

**Create `targets.json`**, **replacing `<SFN_ARN>` and `<ACCOUNT_ID>`**:
```json
[
  {
    "Id": "order-workflow",
    "Arn": "<SFN_ARN>",
    "RoleArn": "arn:aws:iam::<ACCOUNT_ID>:role/saa-events-to-sfn-role"
  }
]
```

```
aws events put-targets --rule saa-order-placed --event-bus-name saa-orders-bus --targets file://targets.json
```

**✅ You should see** `"FailedEntryCount": 0`.

---

### Step 6: Fire Events 🎯

Create **`events.json`** with four business events:
```json
[
  {
    "EventBusName": "saa-orders-bus",
    "Source": "saa.shop",
    "DetailType": "OrderPlaced",
    "Detail": "{\"orderId\":\"3001\",\"amount\":120}"
  },
  {
    "EventBusName": "saa-orders-bus",
    "Source": "saa.shop",
    "DetailType": "OrderPlaced",
    "Detail": "{\"orderId\":\"3002\",\"amount\":4999}"
  },
  {
    "EventBusName": "saa-orders-bus",
    "Source": "saa.shop",
    "DetailType": "OrderShipped",
    "Detail": "{\"orderId\":\"3001\",\"amount\":120}"
  },
  {
    "EventBusName": "saa-orders-bus",
    "Source": "evil.hacker",
    "DetailType": "OrderPlaced",
    "Detail": "{\"orderId\":\"6666\",\"amount\":1}"
  }
]
```

🔮 **Predict:** How many workflow executions will start, and what will the DynamoDB table contain?

📋 Send them:
```
aws events put-events --entries file://events.json
```

Wait ~10 seconds, then:
```
aws stepfunctions list-executions --state-machine-arn <SFN_ARN> --query "executions[].[name,status]" --output table
```
```
aws dynamodb scan --table-name saa-orders --query "Items[].[orderId.S,amount.N,status.S]" --output table
```

<details>
<summary>🔮 Reveal</summary>

**2 executions, both `SUCCEEDED`.** The table has:

| orderId | amount | status |
|---------|--------|--------|
| 3001 | 120 | APPROVED |
| 3002 | 4999 | REVIEW |

- `OrderShipped` didn't match `detail-type`.
- `evil.hacker` didn't match `source`.

The **event pattern** is your router *and* a filter.

</details>

---

### Step 7: See the Workflow Run (Console)

1. Open **Step Functions → State machines → saa-order-workflow**.
2. Click the execution for order **3002**.
3. In **Graph view**, the path through `FlagForReview` is highlighted green. Click each state to see its **input and output**.

> 💡 This visual audit trail of every decision is why Step Functions is the answer when a scenario says *"track the status of each step"* or *"auditable workflow."*

### 🧩 Checkpoint

A workflow must **pause until a human manager approves** a large order (which could take 2 days), then continue. Which Step Functions type and feature?

<details>
<summary>Answer (+10 XP)</summary>

A **Standard** workflow (Express maxes out at 5 minutes) using the **`.waitForTaskToken`** callback pattern: the workflow sends a token (for example by email through SNS), pauses, and resumes when your approval app calls `SendTaskSuccess` with that token.

</details>

---

### Step 8: Archive & Replay (Bonus)

EventBridge can **archive** events and **replay** them later. That's priceless after a bug fix ("re-process yesterday's orders").

```
aws events create-archive --archive-name saa-orders-archive --event-source-arn arn:aws:events:us-east-1:<ACCOUNT_ID>:event-bus/saa-orders-bus --retention-days 1
```

Send `events.json` again, then look at **EventBridge → Archives**. Replays are started from the console (**Archives → Start replay**).

---

## ⚔️ Boss Challenge: Decouple the Monolith (+250 XP)

**Scenario:** A monolithic e-commerce app runs on one large EC2 instance. On checkout it synchronously:
1. Charges the card (external payment API, sometimes slow or briefly down)
2. Reserves inventory (DynamoDB)
3. Sends a confirmation email
4. Updates a data warehouse (can lag by up to 1 hour)

During sales, checkout takes 20+ seconds and **orders fail if the payment API is briefly unavailable**. Redesign it to be **resilient, scalable and loosely coupled**, using the services from this session. Draw it.

<details>
<summary>🏆 Answer key</summary>

1. **API → SQS (or an EventBridge `OrderPlaced` event)**: checkout returns "order received" in milliseconds. (+50)
2. **Step Functions (Standard)** orchestrates the order: `ChargeCard` (Task with **Retry + exponential backoff** for payment-API blips, and **Catch** → compensation/refund path) → `ReserveInventory` (direct DynamoDB integration) → publish `OrderConfirmed`. (+80)
3. **EventBridge / SNS fan-out** of `OrderConfirmed` to: email (SES via Lambda, or SNS) and an **SQS queue** feeding a batch loader for the warehouse, so a 1-hour lag is fine. (+60)
4. **DLQs** on every queue; **FIFO** if per-customer ordering matters; idempotency keys for payment. (+40)
5. Compute: Lambda or containers that **scale independently** per step. (+20)

</details>

---

## What You Just Did

1. Routed events **by content** with a custom EventBridge bus
2. Built a no-code **Step Functions** workflow with a Choice, Retry and a direct DynamoDB integration
3. Watched the event pattern **filter out** non-matching and untrusted events
4. Explored **archive & replay**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Orchestrate multiple Lambda functions / steps with retries and branching" | **Step Functions** |
| "Human approval step in a workflow" | Step Functions **Standard** + task token |
| "High-volume, short (< 5 min) event processing workflows" | Step Functions **Express** |
| "React to AWS service events (EC2 state change, S3 object created, GuardDuty finding)" | **EventBridge rule** |
| "Ingest events from SaaS (Zendesk, Datadog, Shopify)" | **EventBridge partner event source** |
| "Run a task every day at 02:00 (serverless cron)" | **EventBridge Scheduler** |
| "Re-process past events after a bug fix" | **EventBridge archive & replay** |

**🚨 Exam traps**
- "Lambda functions call each other in a chain" → fragile. Use **Step Functions**.
- Express workflows **can't** run longer than 5 minutes or wait for human approval.
- EventBridge has a **256 KB** event size limit, the same as SQS and SNS.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| No executions after put-events | Target role can't start the workflow, or the pattern doesn't match | **EventBridge → Rules → saa-order-placed → Monitoring** shows `FailedInvocations`; recheck `events-policy.json` |
| Execution `FAILED` at SaveOrder | Step Functions role lacks `dynamodb:PutItem` | Check `sfn-policy.json`'s table ARN |
| `InvalidDefinition` | JSON typo in the ASL | Paste it into the Step Functions console's **Workflow Studio → Code** to validate |

---

## 🧹 Cleanup — All of Session 5

```
aws events delete-archive --archive-name saa-orders-archive
aws events remove-targets --rule saa-order-placed --event-bus-name saa-orders-bus --ids order-workflow
aws events delete-rule --name saa-order-placed --event-bus-name saa-orders-bus
aws events delete-event-bus --name saa-orders-bus
aws stepfunctions delete-state-machine --state-machine-arn <SFN_ARN>
aws dynamodb delete-table --table-name saa-orders
aws iam delete-role-policy --role-name saa-sfn-role --policy-name put-orders
aws iam delete-role --role-name saa-sfn-role
aws iam delete-role-policy --role-name saa-events-to-sfn-role --policy-name start-workflow
aws iam delete-role --role-name saa-events-to-sfn-role
```

(If you skipped Step 8, the first line will error. That's fine.)

**Local folder:**  
**macOS / Linux:** `cd ~ && rm -rf ~/Desktop/saa-session-5`  
**Windows:** `cd ~; Remove-Item -Recurse -Force ~\Desktop\saa-session-5`

**✅ Checkpoint:** EventBridge → Event buses shows only `default`. Step Functions and DynamoDB show no `saa-*` resources.

---

**🏁 Lab complete: +100 XP.** Session 5 done! **[📝 Mini Exam 5 →](mini-exam-05.md)**
