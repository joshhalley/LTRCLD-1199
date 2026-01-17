## Step 10: KCAT (KFAFKA CAT)

* kcat is used to test Kafka connectivity, auth, topics, and message flow without needing any app code
* In our lab the Infra Ubuntu 1 (198.18.5.101) is running Kafka Broker
* Objective of this task is to show how to verify Kafka connectivity, list topics, produce and consume messages without a need of any Kafka tool on the router itself

* Login to Cat8Kv-task1 and connect to the swiss_knife container
```code
app-hosting connect appid swiss_knife session /bin/bash
```

* Basic connectivity test. Verify the router/container can reach Kafka broker.

```code
kcat -b 198.18.5.101:9092 -L
```

Expected result:
```code
swissknife:/root# kcat -b 198.18.5.101:9092 -L
Metadata for all topics (from broker 1: 198.18.5.101:9092/1):
 1 brokers:
  broker 1 at 198.18.5.101:9092 (controller)
 1 topics:
  topic "netops-test" with 1 partitions:
    partition 0, leader 1, replicas: 1, isrs: 1
```

What this proves:
 TCP connectivity \
 Broker reachable \
 Metadata exchange works

* List topics (read-only check)
```code
kcat -b 198.18.5.101:9092 -L | grep topic
```
Used when:
 Kafka is up but app says topic missing
 Validate environment is correct

* Produce test messages (router → Kafka)
* Send messages from Cat8Kv container:
```code
echo "hello-from-cat8kv" | \
kcat -b 198.18.5.101:9092 -t netops-test -P
```

What this demonstrates:
 Router can publish telemetry / logs / events \
 Kafka path is working

* Consume messages (verification)

```code
kcat -b 198.18.5.101:9092 -t netops-test -C
```
This confirms:
 Messages arrived \
 No encoding issues \
 Ordering visible


---