# Task 12: Telegraf Monitoring with Prometheus Exporter

[⬅ Back to Main Menu](README.md)

---

## Objective

In this task, you deploy and configure a **Telegraf monitoring container** that acts as a lightweight **monitoring probe**, capable of:

- Collecting container self-metrics (CPU, memory, disk, network)
- Performing ICMP reachability checks
- Performing HTTP / HTTPS availability checks
- Exposing all metrics via a **Prometheus `/metrics` endpoint**

> Visualization is handled centrally using **Prometheus and Grafana** on the management node.

---

## Page Index

- [Lab Architecture](#lab-architecture)
- [Step 1: Prepare Telegraf Configuration Directory](#step-1-prepare-telegraf-configuration-directory)
- [Step 2: Create Base Telegraf Agent Configuration](#step-2-create-base-telegraf-agent-configuration)
- [Step 3: Create Lab Configuration](#step-3-create-lab-configuration)
- [Step 4: Validate Telegraf Configuration](#step-4-validate-telegraf-configuration)
- [Step 5: Start Telegraf](#step-5-start-telegraf)
- [Step 6: Verify Metrics from Mgmt Node](#step-6-verify-metrics-from-mgmt-node)
- [Step 7: Visualize Metrics in Grafana](#step-7-visualize-metrics-in-grafana)
- [Placeholder: SNMP Monitoring](#placeholder-snmp-monitoring)
- [Summary](#summary)

---

## Lab Architecture

### Logical Components

- **Telegraf Container**
  - Collects metrics
  - Exposes `/metrics` on port `9273`
- **Management Node**
  - Prometheus (scrapes metrics)
  - Grafana (visualization)
- **External Targets**
  - Public IPs (ICMP)
  - Public / internal HTTP services

---

## Step 1: Prepare Telegraf Configuration Directory

Connect to the **Telegraf container** and create the configuration directory:

```bash
mkdir -p /etc/telegraf/telegraf.d
```

---

## Step 2: Create Base Telegraf Agent Configuration

Create the main Telegraf configuration file:

```bash
cat > /etc/telegraf/telegraf.conf <<'EOF'
[agent]
  interval = "10s"
  round_interval = true
  metric_batch_size = 1000
  metric_buffer_limit = 10000
  flush_interval = "10s"
EOF
```

This file defines **global agent behavior**.
All inputs and outputs are defined under `telegraf.d`.

---

## Step 3: Create Lab Configuration

Edit the lab configuration file:

```bash
nano /etc/telegraf/telegraf.d/lab.conf
```

### Lab Configuration (`lab.conf`)

```toml
[agent]
  interval = "10s"
  round_interval = true
  flush_interval = "10s"
  hostname = "swiss-knife-monitor"
  omit_hostname = false

###############################################################################
# A) Container self-monitoring
###############################################################################

[[inputs.cpu]]
  percpu = true
  totalcpu = true

[[inputs.mem]]

[[inputs.net]]

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs", "overlay"]

###############################################################################
# B) Network reachability (ICMP)
###############################################################################

[[inputs.ping]]
  urls = [
    "8.8.8.8",
    "1.1.1.1",
    "198.18.5.101"
  ]
  count = 3
  timeout = 2.0
  method = "native"

###############################################################################
# C) HTTP / HTTPS availability
###############################################################################

[[inputs.http_response]]
  urls = [
    "https://www.google.com",
    "https://www.cloudflare.com",
    "http://198.18.5.101:3000"
  ]
  response_timeout = "5s"
  follow_redirects = true

###############################################################################
# D) Output: Prometheus exporter
###############################################################################

[[outputs.prometheus_client]]
  listen = ":9273"
  path = "/metrics"
```

---

## Step 4: Validate Telegraf Configuration

Validate the configuration before running Telegraf:

```bash
telegraf \
  --config /etc/telegraf/telegraf.conf \
  --config-directory /etc/telegraf/telegraf.d \
  --test | head
```

### Expected Result

* No errors
* Inputs for CPU, memory, ping, and HTTP are loaded
* Metrics are generated successfully

---

## Step 5: Start Telegraf

Start Telegraf in continuous mode:

```bash
telegraf \
  --config /etc/telegraf/telegraf.conf \
  --config-directory /etc/telegraf/telegraf.d
```

Telegraf now exposes metrics at:

```text
http://<container-ip>:9273/metrics
```

---

## Step 6: Verify Metrics from Mgmt Node

From the **management node**, verify Prometheus can scrape the container:

```bash
curl http://<container-ip>:9273/metrics | head
```

Verify Prometheus targets:

```bash
curl http://localhost:9090/api/v1/targets
```

---

## Step 7: Visualize Metrics in Grafana

1. Open Grafana:

   ```text
   http://198.18.1.101:3000
   ```

2. Navigate to:

   ```text
   Dashboards → Swiss-Knife Telegraf Lab
   ```

### Expected Observations

* Container CPU and memory graphs update in real time
* Network RX/TX traffic is visible
* Ping and HTTP checks reflect reachability and availability

No manual queries are required.

---

## Placeholder: SNMP Monitoring

> **Not implemented in this task**

In a later task, this same Telegraf container will be extended to:

* Poll router metrics using SNMP
* Collect interface statistics, CPU, and memory
* Export router metrics via the same Prometheus endpoint

---

## Summary

In this task, you deployed a **Telegraf monitoring container** configured to:

* Monitor container health
* Test network reachability
* Validate application availability
* Expose metrics using a **Prometheus-native model**

---

[⬅ Return to Main Menu](README.md)

````

---