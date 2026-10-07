# Lab 5B: SNS Fan-Out to SQS with Message Filtering

**Session:** 5 — Decoupled Architectures  
**Exam Domain:** Domain 2 — Design Resilient Architectures (26%)  
**Difficulty:** Intermediate  
**Estimated Time:** 35–45 minutes

---

## Overview

One order is placed, and **three** teams care: shipping must ship it, analytics must count it, and fraud must check it, but **only if it's expensive**. Calling three services one after another from the web app is fragile: if analytics is down, does the order fail?

The exam's answer is the **SNS → SQS fan-out** pattern. You publish once, SNS pushes a copy to every subscribed queue, and each team consumes at its own pace. With **subscription filter policies**, each queue receives only what it cares about.

**What you will build:**

```
                                     ┌──▶ saa-shipping   (everything)
 publish once ─▶ SNS saa-order-events┼──▶ saa-analytics  (event_type = placed or cancelled)
                                     └──▶ saa-fraud      (amount ≥ 1000 only)
```

---

## Prerequisites

- ✅ **Lab 5A** complete (you understand queues, receive and delete)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon SNS | Pub/sub topic | First 1M publishes free; SNS → SQS deliveries are free |
| Amazon SQS | Three queues | Free tier |

**Estimated cost for this lab: $0.00**

---

## Concepts

| | **SNS** | **SQS** |
|-|---------|---------|
| Model | **Push**, pub/sub (one → many) | **Pull**, queue (one → one consumer group) |
| Persistence | Doesn't store messages for later | Stores up to 14 days |
| Subscribers | SQS, Lambda, HTTP/S, email, SMS, Firehose, mobile push | Consumers poll |

**Why put SQS behind SNS** instead of subscribing Lambda/HTTP directly?
- **Durability:** if the consumer is down, messages wait in the queue.
- **Independent scaling and retries** per consumer.
- **DLQs** per consumer.

**Filter policies** are JSON rules on a *subscription*. SNS only delivers messages whose **message attributes** (or body, if configured) match. Filtering at the subscription means consumers don't waste compute discarding irrelevant messages.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What It Is |
|-------------|-----------|
| `<ACCOUNT_ID>` | Your account ID |
| `<TOPIC_ARN>` | The SNS topic ARN (Step 2) |
| `<SHIPPING_URL>`, `<ANALYTICS_URL>`, `<FRAUD_URL>` | Queue URLs (Step 3) |

---

## Lab Steps

### Step 1: Profile and Folder

**Windows:** `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"; cd ~\Desktop\saa-session-5`  
**macOS / Linux:** `export AWS_PROFILE="<YOUR_PROFILE_NAME>" && cd ~/Desktop/saa-session-5`

---

### Step 2: Create the Topic

```
aws sns create-topic --name saa-order-events --query TopicArn --output text
```

> **📝 Save as `<TOPIC_ARN>`**

---

### Step 3: Create Three Queues with a Policy That Lets SNS Deliver

SQS queues reject messages from SNS unless their **queue policy** (a resource-based policy, as in Lab 1A) allows it.

**Create `queue-attrs.json`**, **replacing `<ACCOUNT_ID>`** (3 places). It's an escaped policy string again:
```json
{
  "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"AllowSnsTopic\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"sns.amazonaws.com\"},\"Action\":\"sqs:SendMessage\",\"Resource\":\"arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-*\",\"Condition\":{\"ArnEquals\":{\"aws:SourceArn\":\"arn:aws:sns:us-east-1:<ACCOUNT_ID>:saa-order-events\"}}}]}"
}
```

> 💡 Unescaped, that policy reads: *"Allow the SNS service to `sqs:SendMessage` to my `saa-*` queues, **only** when the message comes from **my** `saa-order-events` topic."* The `aws:SourceArn` condition stops any other SNS topic in the world from writing to your queues. That's the confused-deputy protection again.

📋 Create the queues:
```
aws sqs create-queue --queue-name saa-shipping --attributes file://queue-attrs.json --query QueueUrl --output text
```
```
aws sqs create-queue --queue-name saa-analytics --attributes file://queue-attrs.json --query QueueUrl --output text
```
```
aws sqs create-queue --queue-name saa-fraud --attributes file://queue-attrs.json --query QueueUrl --output text
```

> **📝 Save the three URLs**

---

### Step 4: Subscribe the Queues with Filter Policies

**Shipping gets everything**, raw (without the SNS JSON envelope):
```
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-shipping --attributes RawMessageDelivery=true
```

**Analytics: only placed and cancelled orders.** Create `analytics-sub.json`:
```json
{
  "RawMessageDelivery": "true",
  "FilterPolicy": "{\"event_type\":[\"order_placed\",\"order_cancelled\"]}"
}
```
```
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-analytics --attributes file://analytics-sub.json
```

**Fraud: only orders of $1,000 or more.** Create `fraud-sub.json`:
```json
{
  "RawMessageDelivery": "true",
  "FilterPolicy": "{\"event_type\":[\"order_placed\"],\"amount\":[{\"numeric\":[\">=\",1000]}]}"
}
```
```
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol sqs --notification-endpoint arn:aws:sqs:us-east-1:<ACCOUNT_ID>:saa-fraud --attributes file://fraud-sub.json
```

> 💡 Inside a filter policy, keys are **AND**-ed and values in an array are **OR**-ed. Fraud = `event_type is order_placed` **AND** `amount ≥ 1000`.

---

### Step 5: Publish Four Events

Each event is published with **message attributes** that the filters match against. Create these four files:

**`evt1.json`**: a small order
```json
{
  "event_type": { "DataType": "String", "StringValue": "order_placed" },
  "amount": { "DataType": "Number", "StringValue": "45" }
}
```

**`evt2.json`**: a big order
```json
{
  "event_type": { "DataType": "String", "StringValue": "order_placed" },
  "amount": { "DataType": "Number", "StringValue": "2500" }
}
```

**`evt3.json`**: a shipment update
```json
{
  "event_type": { "DataType": "String", "StringValue": "order_shipped" },
  "amount": { "DataType": "Number", "StringValue": "2500" }
}
```

**`evt4.json`**: a cancellation
```json
{
  "event_type": { "DataType": "String", "StringValue": "order_cancelled" },
  "amount": { "DataType": "Number", "StringValue": "45" }
}
```

🔮 **Predict before publishing:** fill in how many messages each queue will hold.

| Queue | Prediction |
|-------|-----------|
| saa-shipping | |
| saa-analytics | |
| saa-fraud | |

📋 Publish all four:
```
aws sns publish --topic-arn <TOPIC_ARN> --message "order 2001 placed (45)" --message-attributes file://evt1.json
```
```
aws sns publish --topic-arn <TOPIC_ARN> --message "order 2002 placed (2500)" --message-attributes file://evt2.json
```
```
aws sns publish --topic-arn <TOPIC_ARN> --message "order 2002 shipped" --message-attributes file://evt3.json
```
```
aws sns publish --topic-arn <TOPIC_ARN> --message "order 2001 cancelled" --message-attributes file://evt4.json
```

📋 Count each queue (wait ~5 seconds first):
```
aws sqs get-queue-attributes --queue-url <SHIPPING_URL> --attribute-names ApproximateNumberOfMessages --output text
```
```
aws sqs get-queue-attributes --queue-url <ANALYTICS_URL> --attribute-names ApproximateNumberOfMessages --output text
```
```
aws sqs get-queue-attributes --queue-url <FRAUD_URL> --attribute-names ApproximateNumberOfMessages --output text
```

<details>
<summary>🔮 Reveal (+10 XP per queue you got right)</summary>

| Queue | Count | Why |
|-------|-------|-----|
| saa-shipping | **4** | No filter, so it gets everything |
| saa-analytics | **3** | placed (2001), placed (2002), cancelled. `order_shipped` filtered out. |
| saa-fraud | **1** | Only 2002 *placed* with amount 2500. The shipped event also had 2500 but failed the `event_type` condition (AND!). |

</details>

📋 Read what fraud received:
```
aws sqs receive-message --queue-url <FRAUD_URL> --query "Messages[0].Body" --output text
```

**✅ You should see** `order 2002 placed (2500)`, as a raw body thanks to `RawMessageDelivery`.

---

### 🧩 Checkpoint

The fraud team's service is down for 2 hours for maintenance. What happens to big orders published during that window?

<details>
<summary>Answer (+10 XP)</summary>

**Nothing is lost.** SNS delivers them to the `saa-fraud` **queue**, where they wait (up to the retention period, 4 days by default). When the service comes back, it drains the backlog. Shipping and analytics are completely unaffected. That's **decoupling**: one consumer's failure doesn't spread to the others.

</details>

---

### Step 6: (Optional) Add an Email Subscriber

```
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL>
```

Confirm the subscription from your inbox, publish evt2 again, and you'll receive the email. Email, SMS and HTTP subscribers live side by side with queues.

---

### Step 7: Console Checkpoint

**✅ Checkpoint:**
1. **SNS → Topics → saa-order-events → Subscriptions**: three SQS subscriptions (plus email if you added it).
2. Click the **saa-fraud** subscription → **Subscription filter policy** shows the numeric rule.
3. **SQS → saa-fraud → Access policy** shows the `aws:SourceArn` condition.

---

## What You Just Did

1. Built the classic **SNS → SQS fan-out**
2. Secured queues with a **resource policy** + `aws:SourceArn`
3. Routed messages with **filter policies** (AND across keys, OR within arrays)
4. Saw how decoupling isolates consumer failures

---

## 🎯 Exam Corner

| Scenario keyword | Answer |
|------------------|--------|
| "One event → multiple independent systems process it" | **SNS fan-out to SQS queues** |
| "Each subscriber should receive only relevant messages" | **SNS subscription filter policies** |
| "Ordered fan-out, deduplicated" | **SNS FIFO → SQS FIFO** |
| "Route events from SaaS apps / AWS services with content rules, many targets, replay/archive" | **Amazon EventBridge** (next lab) |
| "Real-time streaming, multiple consumers **replaying** data, ordering per shard" | **Kinesis Data Streams** |
| "Migrate an on-prem app using **JMS / AMQP / MQTT** with minimal code change" | **Amazon MQ** |

**🚨 Exam traps**
- SNS alone **doesn't persist** messages for offline consumers. Pair it with SQS for durability.
- SQS **can't** fan out a message to multiple consumer groups. One message is processed by one consumer.
- SNS → SQS across accounts works, and needs the queue policy to allow the other account's topic.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| All queues show 0 | The queue policy doesn't allow SNS | Check `queue-attrs.json`: account ID, topic name `saa-order-events` |
| Fraud got 0 messages | Filter typo, or `amount` sent as String | `amount` must be `DataType: Number` for numeric matching |
| Analytics got 4 | Filter policy not saved | `aws sns get-subscription-attributes --subscription-arn <SUB_ARN>` |

---

## 🧹 Cleanup

📋 Delete the topic (this also removes its subscriptions):
```
aws sns delete-topic --topic-arn <TOPIC_ARN>
```
📋 Delete the queues:
```
aws sqs delete-queue --queue-url <SHIPPING_URL>
```
```
aws sqs delete-queue --queue-url <ANALYTICS_URL>
```
```
aws sqs delete-queue --queue-url <FRAUD_URL>
```

**✅ Checkpoint:** SNS → Topics and SQS → Queues show no `saa-*` resources.

---

**🏁 Lab complete: +100 XP.** Next: [Lab 5C — EventBridge + Step Functions](lab-5c-eventbridge-step-functions.md)
