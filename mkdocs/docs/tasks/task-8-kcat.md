# Task 8: kcat (Kafka CAT) Connectivity & Message Flow

[⬅️ Back to Main Menu](../index.md)

kcat is a lightweight CLI tool to validate **Kafka connectivity, topics, and message flow** without needing to install a full Kafka client stack on the router.

**Lab context**
- Kafka Broker: **Infra Ubuntu 1** (`198.18.5.101:9092`)
- You will run kcat from the **Swiss-Knife container** on `cat8Kv-task-1`

---

## Table of Contents

- [Step 1: Connect to the Swiss-Knife container](#step-1-connect-to-the-swiss-knife-container)
- [Step 2: Validate broker connectivity and metadata](#step-2-validate-broker-connectivity-and-metadata)
- [Step 3: List topics (read-only check)](#step-3-list-topics-read-only-check)
- [Step 4: Produce messages to a topic](#step-4-produce-messages-to-a-topic)
- [Step 5: Consume messages from a topic](#step-5-consume-messages-from-a-topic)
- [Step 6: Consume messages from a topic - Different Container](#step-5-consume-messages-from-a-topic)
- [Troubleshooting quick tips](#troubleshooting-quick-tips)

---

## Step 1: Connect to the Swiss-Knife container

On `cat8Kv-task-1`:

```bash
app-hosting connect appid swiss_knife session /bin/bash
```

---

## Step 2: Validate broker connectivity and metadata

This confirms:
- TCP reachability to the broker
- Kafka protocol handshake works
- You can retrieve cluster/topic metadata

```bash
kcat -b 198.18.5.101:9092 -L
```

**Expected result (example)**

```bash
Metadata for all topics (from broker 1: 198.18.5.101:9092/1):
 1 brokers:
  broker 1 at 198.18.5.101:9092 (controller)
 1 topics:
  topic "netops-test" with 1 partitions:
    partition 0, leader 1, replicas: 1, isrs: 1
```

---

## Step 3: List topics (read-only check)

Use this when:
- An application reports “topic not found”
- You want to validate you are pointing to the correct environment/broker

```bash
kcat -b 198.18.5.101:9092 -L | grep -i topic
```

---

## Step 4: Produce messages to a topic

Send a single test message from the container to Kafka:

```bash
echo "hello-from-cat8kv-task-1" | kcat -b 198.18.5.101:9092 -t netops-test -P
```

What this demonstrates:
- The router/container can publish events (logs/telemetry/test messages)
- The end-to-end path to Kafka is working

---

## Step 5: Consume messages from a topic 

Start a consumer to verify that messages are arriving.

```bash
kcat -b 198.18.5.101:9092 -t netops-test -C
```

Notes:
- This command runs continuously.
- Press **Ctrl+C** to stop the consumer.

---

## Step 6: Consume messages on Cat8Kv-task-2 / Produce message from Cat8Kv-task-1

SSH `cat8Kv-task-2`(198.18.1.12)

```bash
app-hosting connect appid swiss_knife session /bin/bash
```
Start a consumer to verify that messages are arriving.

```bash
kcat -b 198.18.5.101:9092 -t netops-test -C
```

Send a single test message from the container to Kafka:

```bash
echo "hello-from-cat8kv-task-1" | kcat -b 198.18.5.101:9092 -t netops-test -P
```


## Troubleshooting quick tips

- If `kcat -L` times out: verify reachability (`ping`, `tracepath`, ACLs), and confirm the broker IP/port.
- If produce works but consume shows nothing: confirm you are consuming the same topic, and that the producer actually published (use `-C -o beginning` to read from the start).

```bash
kcat -b 198.18.5.101:9092 -t netops-test -C -o beginning
```

---

[⬅️ Return to Main Menu](../index.md)
