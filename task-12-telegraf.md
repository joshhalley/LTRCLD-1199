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
vi /etc/telegraf/telegraf.d/lab.conf
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

* Both configs are loaded:

  * `/etc/telegraf/telegraf.conf`
  * `/etc/telegraf/telegraf.d/lab.conf`
* Required plugins are loaded:

  * `Loaded inputs: cpu disk http_response mem net ping`
* Sample metrics appear (cpu/mem/net/disk/http_response)

**Note:**
`W! Outputs are not used in testing mode!` is expected when using `--test`.

```
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

From the **lab ubuntu node**, verify Prometheus can scrape the container:

```bash
curl http://198.18.102.7:9273/metrics | head
```

Verify Prometheus targets:

```bash
curl http://198.18.5.101:9090/api/v1/targets
```

---

## Step 7: Visualize Metrics in Grafana

1. Open Grafana:

It can be opened directly from the PC

   ```text
   http://198.18.1.101:3000
   ```
Loging with admin / C1sco12345

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

Perfect. Below is the **exact, end-to-end lab flow** you can put straight into your lab guide.
It assumes:

* **Mgmt node** already has **Prometheus + Grafana (golden config)**
* **Participants** deploy and configure **Telegraf container**
* **Dashboard JSON** is exported from Grafana and re-used

No theory, no branching paths.

---

# Lab: Router SNMP Monitoring with Telegraf, Prometheus & Grafana

---

## Lab Objective

In this lab, you will:

* Extend an existing **Telegraf monitoring container** to poll router metrics using SNMP
* Export router metrics via a **Prometheus-compatible endpoint**
* Visualize router interface, CPU, and memory metrics in **Grafana**
* Reuse **MRTG-proven SNMP OIDs** for predictable results

---

## Prerequisites

* Telegraf container deployed and reachable
* Prometheus scraping the Telegraf container (`:9273/metrics`)
* Grafana running with Prometheus datasource configured
* SNMP v2c enabled on routers

---

## Step 1: Access the Telegraf Container

SSH to the router hosting the container and connect:

```bash
app-hosting connect appid telegraf session /bin/bash
```

---

## Step 2: Edit Telegraf Lab Configuration

Edit the lab configuration file:

```bash
nano /etc/telegraf/telegraf.d/lab.conf
```

---

## Step 3: Add SNMP Input (Router Monitoring)

Append the following SNMP configuration **at the end** of `lab.conf`.

> This example uses **Cat8Kv-Task-1**.
> Other routers use the same block with only the IP changed.

```toml
###############################################################################
# SNMP Monitoring – Cat8Kv-Task-1
###############################################################################

[[inputs.snmp]]
  agents = [ "udp://198.18.100.1:161" ]
  version = 2
  community = "public"
  interval = "30s"
  timeout = "5s"
  retries = 2

  name = "snmp"
  agent_host_tag = "agent_host"

  # GigabitEthernet5 (ifIndex 2)
  [[inputs.snmp.field]]
    name = "ifInOctets_gi5"
    oid  = "1.3.6.1.2.1.2.2.1.10.2"

  [[inputs.snmp.field]]
    name = "ifOutOctets_gi5"
    oid  = "1.3.6.1.2.1.2.2.1.16.2"

  # GigabitEthernet6 (ifIndex 3)
  [[inputs.snmp.field]]
    name = "ifInOctets_gi6"
    oid  = "1.3.6.1.2.1.2.2.1.10.3"

  [[inputs.snmp.field]]
    name = "ifOutOctets_gi6"
    oid  = "1.3.6.1.2.1.2.2.1.16.3"

  # CPU – 5 minute average
  [[inputs.snmp.field]]
    name = "cpu_5min"
    oid  = "1.3.6.1.4.1.9.2.1.58.0"

  # Memory Pool 1
  [[inputs.snmp.field]]
    name = "mem_pool1_used"
    oid  = "1.3.6.1.4.1.9.9.48.1.1.1.5.1"

  [[inputs.snmp.field]]
    name = "mem_pool1_free"
    oid  = "1.3.6.1.4.1.9.9.48.1.1.1.6.1"
```

---

## Step 4: Validate SNMP Configuration (Test Mode)

Before running Telegraf continuously, validate the configuration:

```bash
telegraf \
  --config /etc/telegraf/telegraf.conf \
  --config-directory /etc/telegraf/telegraf.d \
  --test | grep snmp
```

### Expected Output

You should see lines such as:

* `snmp_ifInOctets_gi5`
* `snmp_ifOutOctets_gi6`
* `snmp_cpu_5min`
* `snmp_mem_pool1_used`

This confirms SNMP polling is working.

---

## Step 5: Start Telegraf (Runtime Mode)

Start Telegraf normally:

```bash
telegraf \
  --config /etc/telegraf/telegraf.conf \
  --config-directory /etc/telegraf/telegraf.d
```

---

## Step 6: Verify Metrics Export

From the management node (or any reachable host):

```bash
curl http://<telegraf-container-ip>:9273/metrics | grep snmp | head
```

You should see `snmp_*` metrics exposed.

---

## Step 7: Verify Prometheus Ingestion

On the management node:

```bash
curl http://localhost:9090/api/v1/series -d 'match[]=snmp_cpu_5min'
```

Expected: SNMP metrics with labels including:

* `agent_host`
* `instance`
* `job=telegraf_containers`

---

## Step 8: Import Router Dashboard in Grafana

1. Open Grafana:

   ```
   http://<mgmt-ip>:3000
   ```

2. Import dashboard:

   * **Dashboards → New → Import**
   * Upload the **updated JSON exported from Grafana**
   * Select datasource: **Prometheus**

3. Open the dashboard.

---

## Step 9: Select Router and View Metrics

At the top of the dashboard:

* Select router from the **router dropdown** (e.g. `198.18.100.1`)

### Expected Results

* **Gi5 / Gi6 throughput** graphs populate
* **CPU (5-minute)** graph updates
* **Memory used/free** graphs update

Graphs update in near real time.

---

## Step 10: (Optional) Add Additional Routers

To monitor additional routers:

* Duplicate the `[[inputs.snmp]]` block
* Change only the router IP address
* Restart Telegraf

No Grafana changes are required.

---

## Lab Completion Criteria

You have successfully completed this lab when:

* Telegraf exports SNMP metrics
* Prometheus ingests router metrics
* Grafana displays interface, CPU, and memory data
* Router metrics correlate with existing MRTG trends

---

## Key Takeaway

This lab demonstrates how **traditional SNMP data (MRTG)** can be:

* Collected by a containerized probe
* Exported using a modern metrics pipeline
* Visualized dynamically and correlated with container and service health

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