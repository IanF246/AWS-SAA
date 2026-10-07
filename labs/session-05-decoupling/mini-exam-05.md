# 📝 Mini Exam 5: Decoupled Architectures

**Covers:** Labs 5A, 5B, 5C · **Questions:** 10 · **Time limit:** ⏱️ 15 minutes  
**Pass mark:** 8/10 · **Reward:** +200 XP (perfect: +100 bonus)

> 💡 **Exam technique: name the messaging shape.** Before reading the options, label the scenario: **one-to-one buffer** (SQS), **one-to-many push** (SNS), **content-based routing / SaaS / AWS events** (EventBridge), **multi-step workflow** (Step Functions), **ordered stream with replay by many consumers** (Kinesis). Then find the option that matches.

### ✍️ Answer Sheet

| Q | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|----|
| My answer | | | | | | | | | | |
| ✅ / ❌ | | | | | | | | | | |

---

### Question 1

During flash sales, a web tier sends orders directly to a processing tier, which becomes overwhelmed, and **orders are lost**. **What is the MOST resilient solution?**

- **A.** Scale up the processing tier's instance size
- **B.** Place an SQS queue between the tiers, and have the processing tier poll the queue and scale on queue depth
- **C.** Add an SNS topic between the tiers
- **D.** Put the processing tier behind an NLB

<details>
<summary>Reveal answer</summary>

**✅ B.** SQS **durably buffers** orders until workers can process them, and queue depth is a natural scaling signal. SNS (C) pushes immediately and doesn't hold messages if the consumer can't take them.

🔁 **Review:** [Lab 5A, Overview](lab-5a-sqs-queues-dlq-fifo.md#overview)

</details>

---

### Question 2

Messages in an SQS queue are being **processed twice**. Processing takes about 90 seconds; the visibility timeout is 30 seconds. **What is the fix?**

- **A.** Enable long polling
- **B.** Increase the visibility timeout to more than the processing time
- **C.** Switch to an SNS topic
- **D.** Reduce the message retention period

<details>
<summary>Reveal answer</summary>

**✅ B.** The message becomes visible again before the first consumer deletes it, so a second consumer picks it up.

🔁 **Review:** [Lab 5A, Step 4 checkpoint](lab-5a-sqs-queues-dlq-fifo.md#step-4-consume-it-but-crash-before-deleting)

</details>

---

### Question 3

A few malformed messages fail processing repeatedly and keep returning to the queue, **consuming worker capacity**. **What should be configured?**

- **A.** A delay queue
- **B.** A dead-letter queue with an appropriate maxReceiveCount
- **C.** A FIFO queue
- **D.** A longer retention period

<details>
<summary>Reveal answer</summary>

**✅ B.** After `maxReceiveCount` failed receives, SQS moves the message aside for investigation and **redrive**.

🔁 **Review:** [Lab 5A, Step 5](lab-5a-sqs-queues-dlq-fifo.md#step-5-become-a-poison-message)

</details>

---

### Question 4

A banking app must process **transactions per account in the exact order they occur, with no duplicates**, and handle thousands of accounts in parallel. **Which solution meets these requirements?**

- **A.** SQS standard queue with long polling
- **B.** SQS FIFO queue using the account ID as the message group ID
- **C.** SNS standard topic
- **D.** SQS FIFO queue with a single message group ID for all messages

<details>
<summary>Reveal answer</summary>

**✅ B.** FIFO gives ordering and deduplication. **Per-account message groups** allow parallelism across accounts. D is correct but serializes everything, so it fails the throughput requirement.

🔁 **Review:** [Lab 5A, Step 9 checkpoint](lab-5a-sqs-queues-dlq-fifo.md#step-9-try-to-charge-a-customer-twice)

</details>

---

### Question 5

An application polls an SQS queue in a tight loop. Most `ReceiveMessage` calls return **empty**, and SQS costs are rising. **What should be done?**

- **A.** Increase the visibility timeout
- **B.** Enable long polling by setting `ReceiveMessageWaitTimeSeconds` up to 20
- **C.** Use a FIFO queue
- **D.** Add a dead-letter queue

<details>
<summary>Reveal answer</summary>

**✅ B.** Long polling waits for messages to arrive, which eliminates most empty (but still billed) responses.

🔁 **Review:** [Lab 5A, Step 7](lab-5a-sqs-queues-dlq-fifo.md#step-7-short-polling-vs-long-polling)

</details>

---

### Question 6

When an order is placed, **three independent services** must each process it, and each must keep working even if the others are down. **Which architecture is MOST appropriate?**

- **A.** The web app calls each service's API sequentially
- **B.** Publish to an SNS topic with an SQS queue subscribed for each service
- **C.** One SQS queue that all three services poll
- **D.** Store orders in S3 and have each service list the bucket every minute

<details>
<summary>Reveal answer</summary>

**✅ B.** That's the fan-out pattern. With one shared queue (C), each message goes to only **one** consumer, so the services would steal each other's messages.

🔁 **Review:** [Lab 5B, Step 5](lab-5b-sns-fanout-filtering.md#step-5-publish-four-events)

</details>

---

### Question 7

In an SNS fan-out, the audit service should receive **only** messages where `event_type` is `refund`. Today it receives everything and discards 95%. **What is the MOST efficient solution?**

- **A.** Create a separate SNS topic for refunds and change all publishers
- **B.** Add a subscription filter policy on the audit service's subscription
- **C.** Add a Lambda function to filter messages before the audit queue
- **D.** Use an SQS FIFO queue

<details>
<summary>Reveal answer</summary>

**✅ B.** Filter policies drop unmatched messages at SNS, with no publisher changes and no extra compute.

🔁 **Review:** [Lab 5B, Step 4](lab-5b-sns-fanout-filtering.md#step-4-subscribe-the-queues-with-filter-policies)

</details>

---

### Question 8

A company wants to automatically trigger a workflow whenever **an EC2 instance in us-east-1 changes to the `stopped` state**. **Which service detects this event with the LEAST effort?**

- **A.** CloudTrail with a custom log parser
- **B.** Amazon EventBridge rule matching EC2 Instance State-change Notification events
- **C.** A Lambda function polling `DescribeInstances` every minute
- **D.** AWS Config

<details>
<summary>Reveal answer</summary>

**✅ B.** AWS services emit events to the **default event bus** automatically. A rule with a pattern on `detail.state = stopped` triggers your target.

🔁 **Review:** [Lab 5C, Exam Corner](lab-5c-eventbridge-step-functions.md#-exam-corner)

</details>

---

### Question 9

An order process involves payment, inventory reservation and shipping. Each step must **retry on transient failures**, a **refund must run if shipping fails**, and the business needs a **visual history of each order's steps**. **What should be used?**

- **A.** Lambda functions that invoke each other directly
- **B.** AWS Step Functions Standard workflow with Retry and Catch
- **C.** SQS queues chained between Lambda functions
- **D.** Step Functions Express workflow for long-running orders

<details>
<summary>Reveal answer</summary>

**✅ B.** Step Functions gives retries, error handling (Catch → compensation) and a visual execution history. D: Express workflows are capped at 5 minutes and have no exactly-once execution.

🔁 **Review:** [Lab 5C, Step 4](lab-5c-eventbridge-step-functions.md#step-4-define-and-create-the-state-machine)

</details>

---

### Question 10 *(Select TWO)*

An SNS topic must deliver messages to an SQS queue, but **no messages arrive**. The subscription is confirmed. **Which TWO conditions could cause this?**

- **A.** The SQS queue policy doesn't allow `sqs:SendMessage` from the SNS topic
- **B.** The queue has long polling enabled
- **C.** The queue is encrypted with a customer managed KMS key whose key policy doesn't allow SNS to use it
- **D.** The queue's visibility timeout is 30 seconds
- **E.** The topic has more than one subscriber

<details>
<summary>Reveal answer</summary>

**✅ A and C.** The queue policy must allow SNS. With **SSE-KMS using a customer managed key**, SNS also needs `kms:GenerateDataKey` and `kms:Decrypt` through the key policy. (A frequent real-world and exam gotcha.)

🔁 **Review:** [Lab 5B, Step 3](lab-5b-sns-fanout-filtering.md#step-3-create-three-queues-with-a-policy-that-lets-sns-deliver) · [Lab 3A, Step 3](../session-03-data-protection/lab-3a-kms-envelope-encryption.md#step-3-create-a-customer-managed-key)

</details>

---

## 🧮 Score Yourself

| Score | Result | XP | Next Step |
|-------|--------|----|-----------|
| **10/10** | 🏆 Flawless | +300 | Session 6 |
| **8–9** | ✅ Pass | +200 | Log misses, move on |
| **6–7** | 🔁 Almost | +0 | Redo 🔁 links, retake in 48 h |
| **≤ 5** | 📚 Rebuild | +0 | Redo Lab 5A Steps 4–6 |

### 🎯 Pick-the-Pipe Drill (+5 XP each)

| Scenario | Service |
|----------|---------|
| Buffer work for one worker fleet | SQS |
| Strict order + no duplicates | SQS FIFO |
| Push one message to many endpoints | SNS |
| Route by event content, SaaS sources | EventBridge |
| Cron job without servers | EventBridge Scheduler |
| Multi-step workflow with retries | Step Functions |
| Real-time clickstream, multiple replaying consumers | Kinesis Data Streams |
| Deliver a stream to S3/Redshift with no code | Amazon Data Firehose |
| Lift-and-shift RabbitMQ / ActiveMQ app | Amazon MQ |

---

**Next:** [Session 6 — Databases & DR, Lab 6A →](../session-06-databases-dr/lab-6a-rds-multi-az-read-replica.md)
