# Lab 5A: SQS — Visibility Timeouts, Dead-Letter Queues, Long Polling & FIFO

**Session:** 5 — Decoupled Architectures  
**Exam Domain:** Domain 2 — Design Resilient Architectures (26%)  
**Difficulty:** Beginner  
**Estimated Time:** 35–45 minutes

---

## Overview

When the SAA exam says *"the application loses orders when traffic spikes"* or *"the front end and back end must scale independently,"* the answer is almost always **Amazon SQS**. A queue absorbs bursts, so the producer never waits for the consumer.

In this lab **you play the consumer**. You'll receive messages by hand, "crash" without deleting them, watch them come back, and watch a **poison message** land in a **dead-letter queue (DLQ)**. Then you'll see **FIFO** queues refuse a duplicate payment.

**What you will build:**

```
 Producer ──send──▶ ┌──────────────────────┐  received 3× but never deleted  ┌──────────────────┐
                    │ saa-orders (standard)│ ──────────────────────────────▶ │ saa-orders-dlq   │
 You (consumer) ◀── │ visibility: 20s      │                                 │ (poison messages)│
                    └──────────────────────┘                                 └──────────────────┘

 Producer ──send──▶ ┌──────────────────────────┐   exactly-once processing, strict order
                    │ saa-payments.fifo        │   per MessageGroupId
                    └──────────────────────────┘
```

---

## Prerequisites

- ✅ AWS CLI authenticated
- ✅ Sessions 1–4 recommended (not strictly required)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon SQS | Message queues | **First 1 million requests/month free** |

**Estimated cost for this lab: $0.00**

---

## Concepts

| Term | Meaning |
|------|---------|
| **Visibility timeout** | After a consumer receives a message, it's **hidden** for N seconds. If it isn't **deleted** in time, it reappears for another consumer. (Default 30 s, max 12 h.) |
| **Receive ≠ delete** | Receiving a message does *not* remove it. The consumer must call `DeleteMessage` after processing. This is how SQS guarantees no message is lost when a consumer crashes. |
| **maxReceiveCount** | How many receives before SQS gives up and moves the message to the **DLQ** |
| **Long polling** | `WaitTimeSeconds` 1–20: the call waits for messages instead of returning empty immediately. Fewer empty responses means **lower cost**. |
| **Retention** | 1 minute to 14 days (default 4 days) |

**Standard vs. FIFO:**

| | Standard | FIFO |
|-|----------|------|
| Throughput | Nearly unlimited | 300 msg/s per API action (3,000 with batching; much higher in high-throughput mode) |
| Delivery | **At least once** (rare duplicates) | **Exactly-once processing** |
| Order | **Best effort** | **Strict, per message group** |
| Name | anything | **must end in `.fifo`** |

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<YOUR_PROFILE_NAME>` | Your CLI profile |
| `<ACCOUNT_ID>` | Your account ID |
| `<ORDERS_URL>`, `<DLQ_URL>`, `<FIFO_URL>` | Queue URLs (printed when created) |
| `<RECEIPT_HANDLE>` | A long token from a receive call |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows (PowerShell):**
```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir ~\Desktop\saa-session-5; cd ~\Desktop\saa-session-5; code .
```

**macOS / Linux:**
```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
mkdir -p ~/Desktop/saa-session-5 && cd ~/Desktop/saa-session-5 && code .
```

---

## Part 1 — Standard Queue + DLQ

### Step 2: Create the DLQ, Then the Main Queue

📋 The DLQ:
```
aws sqs create-queue --queue-name saa-orders-dlq --query QueueUrl --output text
```

> **📝 Save as `<DLQ_URL>`**

**Create `orders-attrs.json`**, **replacing `<ACCOUNT_ID>`**. The `RedrivePolicy` is a JSON **string inside** JSON, which is why the inner quotes are escaped:
```json
{
  "VisibilityTimeout": "20",
  "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-orders-dlq\",\"maxReceiveCount\":\"3\"}"
}
```

```
aws sqs create-queue --queue-name saa-orders --attributes file://orders-attrs.json --query QueueUrl --output text
```

> **📝 Save as `<ORDERS_URL>`**

---

### Step 3: Produce a Message

```
aws sqs send-message --queue-url <ORDERS_URL> --message-body "order-1001: 2x coffee beans"
```

---

### Step 4: Consume It, But "Crash" Before Deleting

📋 Receive it:
```
aws sqs receive-message --queue-url <ORDERS_URL> --attribute-names ApproximateReceiveCount --query "Messages[0].[Body,Attributes.ApproximateReceiveCount]" --output text
```

**✅ You should see** `order-1001: 2x coffee beans   1`

🔮 **Predict:** Run the exact same command again **immediately**. What do you get?

<details>
<summary>🔮 Reveal</summary>

**Nothing** (empty output). The message is **invisible** for 20 seconds while "you" process it. Other consumers can't see it, which prevents double-processing.

</details>

🔮 **Predict:** Wait **25 seconds** without deleting, then receive again. What's the receive count?

<details>
<summary>🔮 Reveal</summary>

The message **reappears** with `ApproximateReceiveCount = 2`. Your "consumer" never deleted it, so SQS assumes it crashed and offers the message again. This is how SQS gives you **no message loss** even when workers die mid-task.

</details>

### 🧩 Checkpoint

A job takes **60 seconds** to process, but the queue's visibility timeout is **30 seconds**. What goes wrong?

<details>
<summary>Answer (+10 XP)</summary>

The message reappears halfway through processing, so **a second consumer processes it too**: duplicate work, double-charged customers, and so on. Fix: set the visibility timeout **longer than the maximum processing time** (for example 6× the average, as AWS recommends for Lambda consumers), or have the worker call `ChangeMessageVisibility` to extend it.

</details>

---

### Step 5: Become a Poison Message

Keep receiving **without deleting**. Wait 25 seconds between attempts. 📋 Run this receive, then wait:
```
aws sqs receive-message --queue-url <ORDERS_URL> --attribute-names ApproximateReceiveCount --query "Messages[0].[Body,Attributes.ApproximateReceiveCount]" --output text
```

🔮 **Predict:** `maxReceiveCount` is 3. After the 3rd receive expires, where's the message?

📋 After your 3rd receive, wait 25 seconds and try a 4th. Then check both queues:
```
aws sqs get-queue-attributes --queue-url <ORDERS_URL> --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible --output table
```
```
aws sqs get-queue-attributes --queue-url <DLQ_URL> --attribute-names ApproximateNumberOfMessages --output table
```

<details>
<summary>🔮 Reveal</summary>

The 4th receive returns **nothing**. The main queue shows `0`, and the **DLQ shows `1`**. SQS moved the message once it had been received 3 times without a successful delete. A message that can never be processed (bad data, a bug) would otherwise loop forever, wasting money and blocking other work.

</details>

---

### Step 6: Redrive from the DLQ

You've "fixed the bug." Send the message back to the main queue:

```
aws sqs start-message-move-task --source-arn arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-orders-dlq --destination-arn arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-orders
```

Wait ~10 seconds, then process it **properly**: receive, *then delete*.

📋 Receive and capture the receipt handle:
```
aws sqs receive-message --queue-url <ORDERS_URL> --query "Messages[0].ReceiptHandle" --output text
```
📋 Delete it (paste the whole long handle):
```
aws sqs delete-message --queue-url <ORDERS_URL> --receipt-handle "<RECEIPT_HANDLE>"
```

**✅ Both queues now show 0 messages.** That's a successful processing cycle.

---

### Step 7: Short Polling vs. Long Polling

📋 Short polling on an empty queue returns instantly:
```
aws sqs receive-message --queue-url <ORDERS_URL>
```

📋 Long polling waits up to 10 seconds. Start this, and **within 10 seconds** send a message from a second terminal:
```
aws sqs receive-message --queue-url <ORDERS_URL> --wait-time-seconds 10
```
Second terminal:
```
aws sqs send-message --queue-url <ORDERS_URL> --message-body "order-1002: arrived mid-poll"
```

**✅ The waiting call returns the moment the message arrives.** Fewer empty receives means **lower cost** and lower latency. Long polling is almost always the right answer.

---

## Part 2 — FIFO: Order and Exactly-Once Processing

### Step 8: Create a FIFO Queue

**Create `fifo-attrs.json`:**
```json
{
  "FifoQueue": "true",
  "ContentBasedDeduplication": "true"
}
```
```
aws sqs create-queue --queue-name saa-payments.fifo --attributes file://fifo-attrs.json --query QueueUrl --output text
```

> **📝 Save as `<FIFO_URL>`**

---

### Step 9: Try to Charge a Customer Twice

A buggy client retries a payment request. 📋 Send the **identical** message twice:
```
aws sqs send-message --queue-url <FIFO_URL> --message-body "charge customer-42 USD 19.99" --message-group-id customer-42
```
```
aws sqs send-message --queue-url <FIFO_URL> --message-body "charge customer-42 USD 19.99" --message-group-id customer-42
```

🔮 **Predict:** How many messages are in the queue?

```
aws sqs get-queue-attributes --queue-url <FIFO_URL> --attribute-names ApproximateNumberOfMessages --output text
```

<details>
<summary>🔮 Reveal</summary>

**1.** With **content-based deduplication**, SQS hashes the body, and identical messages within the **5-minute deduplication window** are accepted (the send succeeds) but **dropped**. The customer is charged once.

On a **standard** queue you'd have 2, and you'd need an **idempotent** consumer to catch the duplicate.

</details>

### 🧩 Checkpoint

A FIFO queue processes stock trades for 10,000 different accounts. Using one `MessageGroupId` for everything makes it too slow. What's the fix?

<details>
<summary>Answer (+10 XP)</summary>

Use **one message group per account** (`MessageGroupId = accountId`). Ordering is guaranteed **within** a group, and different groups are processed **in parallel** by different consumers. You keep per-account order *and* get throughput.

</details>

---

### Step 10: Console Checkpoint

**✅ Checkpoint:**
1. **SQS → saa-orders → Dead-letter queue** tab shows `saa-orders-dlq`, max receives 3.
2. **SQS → saa-orders → Send and receive messages**: send and poll from the console, and watch the receive count.
3. **SQS → saa-payments.fifo → Details**: FIFO ✓, Content-based deduplication ✓.

---

## What You Just Did

1. Proved that **receive ≠ delete**, and how the visibility timeout prevents loss
2. Created a **poison message** and caught it in a **DLQ**
3. **Redrove** the DLQ back to the source queue
4. Saw **long polling** cut empty responses
5. Blocked a duplicate payment with **FIFO deduplication**

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "Decouple", "buffer requests", "orders lost during spikes" | **SQS** between tiers |
| "Messages must be processed **in order**, exactly once" | **SQS FIFO** |
| "Messages that fail repeatedly block or slow the queue" | **Dead-letter queue** |
| "Reduce SQS cost / too many empty responses" | **Long polling** (`WaitTimeSeconds` up to 20) |
| "Messages processed twice; processing takes longer than expected" | Increase the **visibility timeout** |
| "Message larger than 256 KB" | Store the payload in **S3**, send a pointer (SQS Extended Client Library) |
| "Delay processing of every new message by 5 minutes" | **Delay queue** (`DelaySeconds`, max 15 min) |

**🚨 Exam traps**
- SQS is **pull-based**. Consumers poll. Fan-out (one message → many consumers) needs **SNS** in front (next lab).
- Standard queues **can** deliver duplicates. "Exactly once" in a question = FIFO.
- You can't convert a standard queue to FIFO. Create a new queue.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `InvalidParameterValue` on create-queue | The escaped JSON in `orders-attrs.json` is malformed | Copy the file exactly; inner quotes must be `\"` |
| Message never goes to the DLQ | You deleted it, or the wait between receives was < 20 s | Wait the full visibility timeout between receives |
| `start-message-move-task` unknown command | Old CLI | Update CLI v2, or use **SQS console → saa-orders-dlq → Start DLQ redrive** |
| FIFO send fails `MissingParameter` | No `--message-group-id` | Every FIFO message needs a group ID |

---

## 🧹 Cleanup

**⏸️ Continuing to Lab 5B?** You can delete these queues now. 5B creates its own.

```
aws sqs delete-queue --queue-url <ORDERS_URL>
```
```
aws sqs delete-queue --queue-url <DLQ_URL>
```
```
aws sqs delete-queue --queue-url <FIFO_URL>
```

**✅ Checkpoint:** **SQS → Queues** shows no `saa-*` queues (deletion can take up to 60 seconds to show).

---

**🏁 Lab complete: +100 XP.** Next: [Lab 5B — SNS Fan-Out with Filtering](lab-5b-sns-fanout-filtering.md)
