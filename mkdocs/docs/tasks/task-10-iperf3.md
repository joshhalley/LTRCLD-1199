# Task 10: iPerf3 Network Performance Testing

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [1. Topology and Roles](#1-topology-and-roles)
- [2. Start iPerf3 Server (cat8Kv-task-2)](#2-start-iperf3-server-cat8kv-task-2)
- [3. TCP Throughput Tests (cat8Kv-task-1)](#3-tcp-throughput-tests-cat8kv-task-1)
- [4. UDP Performance and Loss Tests](#4-udp-performance-and-loss-tests)
- [5. JSON Output and Automation](#5-json-output-and-automation)
- [6. Troubleshooting Checklist](#6-troubleshooting-checklist)
- [7. Key iperf3 Options Reference](#7-key-iperf3-options-reference)
- [8. Expected Outcome](#8-expected-outcome)
- [Sample Output and Analysis – TCP Throughput Test](#sample-output-and-analysis-tcp-throughput-test)

---

This task uses **iperf3** inside the **Swiss-Knife** container to validate **throughput, directionality, parallelism, UDP behavior, and MTU impact** between two Catalyst 8000v routers.

You will run:
- **iperf3 server** on **cat8Kv-task-2**
- **iperf3 client** on **cat8Kv-task-1**

---

## 1. Topology and Roles

| Node          | Role   | Container   | IP Address                 |
|---------------|--------|-------------|----------------------------|
| cat8Kv-task-1 | Client | swiss_knife | Connects to `198.18.101.5` |
| cat8Kv-task-2 | Server | swiss_knife | Listens on `198.18.101.5`  |

**Note:** Update the IP address if your server-side container uses a different value.

---

## 2. Start iPerf3 Server (cat8Kv-task-2)

2.1 Connect to the Swiss-Knife container:

```bash
app-hosting connect appid swiss_knife session /bin/bash
```

2.2 Start iPerf3 in server mode:

```bash
iperf3 -s
```

### What to verify

* Output shows: `Server listening on 5201`
* Server displays connection statistics when tests run
* If no connection is seen:
  * Verify IP reachability between containers
  * Confirm no ACL, NAT, or firewall blocks TCP/5201

---

## 3. TCP Throughput Tests (cat8Kv-task-1)

### 3.1 Connect to the Swiss-Knife container:

```bash
app-hosting connect appid swiss_knife session /bin/bash
```

### 3.2 Basic TCP throughput (client → server):

```bash
iperf3 -c 198.18.101.5
```

### What to verify

* Note the **receiver** throughput in the final summary
* This represents effective end-to-end performance

---

### 3.3 Reverse direction throughput (server → client):

```bash
iperf3 -c 198.18.101.5 -R
```

### What to verify

* Compare results with step 3.2
* Large differences may indicate:

  * Directional QoS or policing
  * CPU constraints on one node
  * Asymmetric routing

---

### 3.4 Parallel TCP streams:

```bash
iperf3 -c 198.18.101.5 -P 5
```

### What to verify

* Aggregate throughput should increase vs single stream
* If throughput decreases:

  * Check CPU utilization
  * Check congestion or drops in the path

---

### 3.5 Short-duration TCP test:

```bash
iperf3 -c 198.18.101.5 -t 5
```

### 3.6 Long-duration TCP test:

```bash
iperf3 -c 198.18.101.5 -t 60
```

### What to verify

* Throughput consistency over time
* Degradation over long runs may indicate:
  * Traffic shaping
  * Buffer exhaustion
  * Sustained CPU load

---

## 4. UDP Performance and Loss Tests

### 4.1 UDP test at 100 Mbps:

```bash
iperf3 -c 198.18.101.5 -u -b 100M
```

### What to verify

* Packet loss should be minimal in a clean lab
* Observe jitter and packet loss percentage

---

### 4.2 UDP test at 50 Mbps with 1400-byte packets:

```bash
iperf3 -c 198.18.101.5 -u -b 50M -l 1400
```

### 4.3 UDP test at 50 Mbps with 1470-byte packets:

```bash
iperf3 -c 198.18.101.5 -u -b 50M -l 1470
```

### 4.4 UDP test at 50 Mbps with 1510-byte packets:

```bash
iperf3 -c 198.18.101.5 -u -b 50M -l 1510
```

### What to verify

* Packet loss appearing only at larger sizes indicates MTU mismatch
* Consistent loss at all sizes suggests congestion or policing

---

## 5. JSON Output and Automation

### 5.1 Run iPerf3 with JSON output:

```bash
iperf3 -c 198.18.101.5 -J
```

### 5.2 Short-duration JSON output:

```bash
iperf3 -c 198.18.101.5 -t 5 -J
```

### 5.3 Extract received throughput using `jq`:

```bash
iperf3 -c 198.18.101.5 -t 5 -J | jq '.end.sum_received.bits_per_second'
```

### What to verify

* JSON output is useful for automation and dashboards
* Values are returned in raw bits per second
* Results can be stored for later analysis:

```bash
iperf3 -c 198.18.101.5 -t 10 -J > iperf_result.json
```

---

## 6. Troubleshooting Checklist

### 6.1 Connectivity

* Verify container-to-container ping
* Validate routing between VirtualPortGroup interfaces
* Confirm TCP/5201 and UDP traffic is permitted

### 6.2 Performance Symptoms and Causes

| Symptom                      | Likely Cause                         |
| ---------------------------- | ------------------------------------ |
| Low TCP, good UDP            | TCP windowing or single-flow hashing |
| Reverse slower than forward  | QoS or policing asymmetry            |
| UDP loss at low rate         | Congestion or shaping                |
| Loss only with large packets | MTU mismatch                         |

---

## 7. Key iperf3 Options Reference

| Option           | Description         |
| ---------------- | ------------------- |
| `-s`             | Server mode         |
| `-c IP`          | Client mode         |
| `-R`             | Reverse direction   |
| `-P N`           | Parallel streams    |
| `-u`             | UDP mode            |
| `-b RATE`        | UDP bandwidth       |
| `-t SEC`         | Test duration       |
| `-l BYTES`       | Packet length       |
| `-J`             | JSON output         |
| `--logfile FILE` | Save output to file |

---

## 8. Expected Outcome

By completing this task, you should be able to:

1. Validate TCP throughput between C8Kv-hosted containers
1. Identify directional asymmetry in the data path
1. Demonstrate the impact of parallel flows
1. Measure UDP loss and jitter
1. Detect MTU mismatches using packet-size variation
1. Generate machine-readable performance metrics

---

## Sample Output and Analysis – TCP Throughput Test

### Command Executed

```bash
iperf3 -c 198.18.101.5
```

---

### Sample Output

```text
Connecting to host 198.18.101.5, port 5201
[  5] local 198.18.100.5 port 34886 connected to 198.18.101.5 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  37.1 MBytes   311 Mbits/sec    0   1.75 MBytes
[  5]   1.00-2.00   sec  38.1 MBytes   320 Mbits/sec   81   1.07 MBytes
[  5]   2.00-3.00   sec  36.1 MBytes   303 Mbits/sec    0   1.13 MBytes
[  5]   3.00-4.00   sec  29.2 MBytes   245 Mbits/sec    0   1.18 MBytes
[  5]   4.00-5.00   sec  29.1 MBytes   244 Mbits/sec    0   1.21 MBytes
[  5]   5.00-6.00   sec  27.6 MBytes   232 Mbits/sec    0   1.23 MBytes
[  5]   6.00-7.00   sec  27.9 MBytes   234 Mbits/sec    0   1.24 MBytes
[  5]   7.00-8.00   sec  27.6 MBytes   232 Mbits/sec    0   1.24 MBytes
[  5]   8.00-9.00   sec  27.9 MBytes   234 Mbits/sec    0   1.24 MBytes
[  5]   9.00-10.00  sec  29.0 MBytes   243 Mbits/sec    0   1.24 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec   310 MBytes   260 Mbits/sec   81             sender
[  5]   0.00-10.04  sec   307 MBytes   257 Mbits/sec                  receiver
```

---

### Output Breakdown

| Field      | Meaning                                  |
| ---------- | ---------------------------------------- |
| `Interval` | 1-second measurement window              |
| `Transfer` | Amount of data sent in that interval     |
| `Bitrate`  | Effective throughput during the interval |
| `Retr`     | TCP retransmissions detected             |
| `Cwnd`     | TCP congestion window size               |

---

### Key Observations

1. **Initial burst followed by stabilization**

   * First 2 seconds show higher throughput (~300+ Mbps)
   * Throughput then stabilizes around **230–245 Mbps**
   * This is typical TCP behavior during congestion window ramp-up

1. **Retransmissions detected**

   * `81` retransmissions observed early in the test
   * Indicates minor packet loss or buffer pressure in the path
   * After stabilization, retransmissions stop

1. **Congestion window adjustment**

   * `Cwnd` reduces from **1.75 MB → ~1.24 MB**
   * TCP adapts to perceived network capacity
   * Resulting in steady, sustainable throughput

1. **Sender vs Receiver throughput**

   * Sender: **260 Mbps**
   * Receiver: **257 Mbps**
   * Receiver value is more accurate and should be used for analysis

---

### What This Tells Us About the Network

* End-to-end TCP connectivity is **working correctly**
* Path supports ~**250 Mbps sustained TCP throughput**
* Minor transient congestion or buffering exists during ramp-up
* No persistent loss or severe instability detected

---

### Common Troubleshooting Insights

| Symptom               | Likely Reason                     |
| --------------------- | --------------------------------- |
| Early retransmissions | TCP slow-start overshoot          |
| Throughput plateaus   | CPU limit, shaping, or congestion |
| Sender > Receiver gap | In-flight buffering               |
| Stable cwnd           | Healthy TCP equilibrium           |

---

[⬅️ Return to Main Menu](../index.md)
