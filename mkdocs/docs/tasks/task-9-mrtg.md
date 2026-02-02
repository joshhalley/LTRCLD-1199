# Task 9: MRTG Interface Monitoring

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [Lab Objective](#lab-objective)
- [Lab Components and Roles](#lab-components-and-roles)
- [Step 1: Enable SNMP on All Routers (Task-1/2/3)](#step-1-enable-snmp-on-all-routers-task-123)
- [Step 2: Connect to the Container and Verify SNMP](#step-2-connect-to-the-container-and-verify-snmp)
- [Step 3: Create MRTG Config Files](#step-3-create-mrtg-config-files)
- [Step 4: Router MRTG Targets](#step-4-router-mrtg-targets)
- [Step 5: Initialize MRTG and Generate Graphs](#step-5-initialize-mrtg-and-generate-graphs)
- [Step 6: Start Web Server (lighttpd)](#step-6-start-web-server-lighttpd)
- [Step 7: Enable Continuous Polling (Cron every 5 minutes)](#step-7-enable-continuous-polling-cron-every-5-minutes)
- [Step 8: View the Graphs](#step-8-view-the-graphs)
- [Notes / Tips](#notes--tips)
- [📈 Optional Traffic Generation with iPerf3 (Observation Step)](#-optional-traffic-generation-with-iperf3-observation-step)

---

## Lab Objective

* Use the **monitoring container mrtg** on **Cat8Kv-Task-3**
* Enable **SNMP** on all routers
* Generate **MRTG graphs** for:

* Interface traffic (GigabitEthernet5 and GigabitEthernet6)
* CPU (5-min)
* Memory pool usage

Graphs are served via a lightweight web server on the container.

---

## Lab Components and Roles

* **Cat8Kv-Task-3**: Runs the monitoring container (App-Hosting)
* **Cat8Kv-Task-1 / Task-2**: SNMP targets monitored by MRTG

---

## Step 1: Enable SNMP on All Routers (Task-1/2/3)

Run on **each router**:

```text
conf t
 snmp-server community public RO
 snmp ifmib ifindex persist
end
write memory
```

> `snmp ifmib ifindex persist` keeps interface indexes stable (prevents MRTG graphs from breaking after reload).

---

## Step 2: Connect to the MRTG Container and Verify SNMP

On `cat8Kv-task-3`:

```text
app-hosting connect appid mrtg session /bin/bash
```

Test SNMP:

```bash
snmpwalk -v2c -c public 198.18.8.13 sysDescr.0
```

---

## Step 3: Create MRTG Config Files

### 3.1 Global MRTG Config: `/opt/mrtg/mrtg.cfg`

```bash
vi /opt/mrtg/mrtg.cfg
```

```cfg
### ===== Global =====
WorkDir: /opt/mrtg/html
Options[_]: growright,bits
EnableIPv6: no

Include: /opt/mrtg/routers/r1.cfg
Include: /opt/mrtg/routers/r2.cfg
Include: /opt/mrtg/routers/r3.cfg
```

---

## Step 4: Router MRTG Targets

✅ Naming rules used below:

* No spaces in target IDs
* Consistent format: `Cat8Kv_TaskX_<Metric>`
* Interface graphs: `Gig5`, `Gig6`

> Interface mapping used (example):
> `ifIndex 2 = GigabitEthernet5`, `ifIndex 3 = GigabitEthernet6`

---

### 4.1 Task-1 Targets: `/opt/mrtg/routers/r1.cfg` (IP: `198.18.6.11`)

```bash
vi /opt/mrtg/routers/r1.cfg
```

```cfg
############################
### Cat8Kv-Task-1
############################

Target[Cat8Kv_Task1_Gig5]: 1.3.6.1.2.1.2.2.1.10.2&1.3.6.1.2.1.2.2.1.16.2:public@198.18.6.11:::::2
MaxBytes[Cat8Kv_Task1_Gig5]: 125000000
Title[Cat8Kv_Task1_Gig5]: Cat8Kv-Task-1 Interface GigabitEthernet5 Traffic
PageTop[Cat8Kv_Task1_Gig5]: <h1>Cat8Kv-Task-1 Interface GigabitEthernet5 Traffic</h1>

Target[Cat8Kv_Task1_Gig6]: 1.3.6.1.2.1.2.2.1.10.3&1.3.6.1.2.1.2.2.1.16.3:public@198.18.6.11:::::2
MaxBytes[Cat8Kv_Task1_Gig6]: 125000000
Title[Cat8Kv_Task1_Gig6]: Cat8Kv-Task-1 Interface GigabitEthernet6 Traffic
PageTop[Cat8Kv_Task1_Gig6]: <h1>Cat8Kv-Task-1 Interface GigabitEthernet6 Traffic</h1>

Target[Cat8Kv_Task1_CPU5m]: 1.3.6.1.4.1.9.2.1.58.0&1.3.6.1.4.1.9.2.1.58.0:public@198.18.6.11:::::2
Options[Cat8Kv_Task1_CPU5m]: growright,gauge,nopercent
MaxBytes[Cat8Kv_Task1_CPU5m]: 100
Title[Cat8Kv_Task1_CPU5m]: Cat8Kv-Task-1 CPU (5-min)
PageTop[Cat8Kv_Task1_CPU5m]: <h1>Cat8Kv-Task-1 CPU (5-min)</h1>

Target[Cat8Kv_Task1_MemPool1]: 1.3.6.1.4.1.9.9.48.1.1.1.5.1&1.3.6.1.4.1.9.9.48.1.1.1.6.1:public@198.18.6.11:::::2
Options[Cat8Kv_Task1_MemPool1]: growright,gauge
MaxBytes[Cat8Kv_Task1_MemPool1]: 4294967295
Title[Cat8Kv_Task1_MemPool1]: Cat8Kv-Task-1 Memory Pool 1
PageTop[Cat8Kv_Task1_MemPool1]: <h1>Cat8Kv-Task-1 Memory Pool 1</h1>
```

---

### 4.2 Task-2 Targets: `/opt/mrtg/routers/r2.cfg` (IP: `198.18.7.12`)

```bash
vi /opt/mrtg/routers/r2.cfg
```

```cfg
############################
### Cat8Kv-Task-2
############################

Target[Cat8Kv_Task2_Gig5]: 1.3.6.1.2.1.2.2.1.10.2&1.3.6.1.2.1.2.2.1.16.2:public@198.18.7.12:::::2
MaxBytes[Cat8Kv_Task2_Gig5]: 125000000
Title[Cat8Kv_Task2_Gig5]: Cat8Kv-Task-2 Interface GigabitEthernet5 Traffic
PageTop[Cat8Kv_Task2_Gig5]: <h1>Cat8Kv-Task-2 Interface GigabitEthernet5 Traffic</h1>

Target[Cat8Kv_Task2_Gig6]: 1.3.6.1.2.1.2.2.1.10.3&1.3.6.1.2.1.2.2.1.16.3:public@198.18.7.12:::::2
MaxBytes[Cat8Kv_Task2_Gig6]: 125000000
Title[Cat8Kv_Task2_Gig6]: Cat8Kv-Task-2 Interface GigabitEthernet6 Traffic
PageTop[Cat8Kv_Task2_Gig6]: <h1>Cat8Kv-Task-2 Interface GigabitEthernet6 Traffic</h1>

Target[Cat8Kv_Task2_CPU5m]: 1.3.6.1.4.1.9.2.1.58.0&1.3.6.1.4.1.9.2.1.58.0:public@198.18.7.12:::::2
Options[Cat8Kv_Task2_CPU5m]: growright,gauge,nopercent
MaxBytes[Cat8Kv_Task2_CPU5m]: 100
Title[Cat8Kv_Task2_CPU5m]: Cat8Kv-Task-2 CPU (5-min)
PageTop[Cat8Kv_Task2_CPU5m]: <h1>Cat8Kv-Task-2 CPU (5-min)</h1>

Target[Cat8Kv_Task2_MemPool1]: 1.3.6.1.4.1.9.9.48.1.1.1.5.1&1.3.6.1.4.1.9.9.48.1.1.1.6.1:public@198.18.7.12:::::2
Options[Cat8Kv_Task2_MemPool1]: growright,gauge
MaxBytes[Cat8Kv_Task2_MemPool1]: 4294967295
Title[Cat8Kv_Task2_MemPool1]: Cat8Kv-Task-2 Memory Pool 1
PageTop[Cat8Kv_Task2_MemPool1]: <h1>Cat8Kv-Task-2 Memory Pool 1</h1>
```

---

### 4.3 Task-3 Targets: `/opt/mrtg/routers/r3.cfg` (IP: `198.18.8.13`)

```bash
vi /opt/mrtg/routers/r3.cfg
```

```cfg
############################
### Cat8Kv-Task-3
############################

Target[Cat8Kv_Task3_Gig5]: 1.3.6.1.2.1.2.2.1.10.2&1.3.6.1.2.1.2.2.1.16.2:public@198.18.8.13:::::2
MaxBytes[Cat8Kv_Task3_Gig5]: 125000000
Title[Cat8Kv_Task3_Gig5]: Cat8Kv-Task-3 Interface GigabitEthernet5 Traffic
PageTop[Cat8Kv_Task3_Gig5]: <h1>Cat8Kv-Task-3 Interface GigabitEthernet5 Traffic</h1>

Target[Cat8Kv_Task3_Gig6]: 1.3.6.1.2.1.2.2.1.10.3&1.3.6.1.2.1.2.2.1.16.3:public@198.18.8.13:::::2
MaxBytes[Cat8Kv_Task3_Gig6]: 125000000
Title[Cat8Kv_Task3_Gig6]: Cat8Kv-Task-3 Interface GigabitEthernet6 Traffic
PageTop[Cat8Kv_Task3_Gig6]: <h1>Cat8Kv-Task-3 Interface GigabitEthernet6 Traffic</h1>

Target[Cat8Kv_Task3_CPU5m]: 1.3.6.1.4.1.9.2.1.58.0&1.3.6.1.4.1.9.2.1.58.0:public@198.18.8.13:::::2
Options[Cat8Kv_Task3_CPU5m]: growright,gauge,nopercent
MaxBytes[Cat8Kv_Task3_CPU5m]: 100
Title[Cat8Kv_Task3_CPU5m]: Cat8Kv-Task-3 CPU (5-min)
PageTop[Cat8Kv_Task3_CPU5m]: <h1>Cat8Kv-Task-3 CPU (5-min)</h1>

Target[Cat8Kv_Task3_MemPool1]: 1.3.6.1.4.1.9.9.48.1.1.1.5.1&1.3.6.1.4.1.9.9.48.1.1.1.6.1:public@198.18.8.13:::::2
Options[Cat8Kv_Task3_MemPool1]: growright,gauge
MaxBytes[Cat8Kv_Task3_MemPool1]: 4294967295
Title[Cat8Kv_Task3_MemPool1]: Cat8Kv-Task-3 Memory Pool 1
PageTop[Cat8Kv_Task3_MemPool1]: <h1>Cat8Kv-Task-3 Memory Pool 1</h1>
```

---

## Step 5: Initialize MRTG and Generate Graphs

```bash
mrtg /opt/mrtg/mrtg.cfg --check
```

```bash
rm -f /opt/mrtg/html/*.log /opt/mrtg/html/*.old
```

Run MRTG multiple times to generate baseline data:

```bash
mrtg /opt/mrtg/mrtg.cfg
mrtg /opt/mrtg/mrtg.cfg
mrtg /opt/mrtg/mrtg.cfg
```

Generate the index page:

```bash
indexmaker /opt/mrtg/mrtg.cfg > /opt/mrtg/html/index.html
```

---

## Step 6: Start Web Server (lighttpd)

Create config:

```bash
cat > /etc/lighttpd/lighttpd.conf <<'EOF'
server.document-root = "/opt/mrtg/html"
server.port = 8080
server.bind = "0.0.0.0"
dir-listing.activate = "enable"
mimetype.assign = (
  ".html" => "text/html",
  ".png"  => "image/png",
  ".css"  => "text/css"
)
EOF
```

Start in background:

```bash
lighttpd -f /etc/lighttpd/lighttpd.conf &
```

---

## Step 7: Enable Continuous Polling (Cron every 5 minutes)

Edit crontab:

```bash
crontab -e
```

Insert, save and exit :

```cron
*/5 * * * * /usr/bin/mrtg /opt/mrtg/mrtg.cfg --logging /opt/mrtg/mrtg.log
```

Start cron and verify:

```bash
crond
ps | grep crond
crontab -l
```

---

## Step 8: View the Graphs

RDP to the lab desktop:

```text
RDP to 198.18.1.20
```

Open browser:

```text
http://198.18.102.5:8080/index.html
```

You should see:

* **Task-1**: Gig5/Gig6 traffic + CPU + Memory
* **Task-2**: Gig5/Gig6 traffic + CPU + Memory
* **Task-3**: Gig5/Gig6 traffic + CPU + Memory

---

## Notes / Tips

### Determine Interface Index Mapping (if needed)

To verify which `ifIndex` maps to which interface:

```bash
snmpwalk -v2c -c public <router-ip> 1.3.6.1.2.1.2.2.1.2
```

Example:

* `ifDescr.2 = GigabitEthernet5`
* `ifDescr.3 = GigabitEthernet6`

---

## 📈 Optional Traffic Generation with iPerf3 (Observation Step)

To make the MRTG graphs more meaningful, we will now **generate traffic between routers** and observe the impact on the interface graphs.

### Purpose

* Generate **controlled traffic** between **Cat8Kv-Task-1** and **Cat8Kv-Task-2**
* Validate that MRTG graphs correctly reflect **real traffic changes**
* Reinforce the concept of **poll-based monitoring**

---

### Optional Step 1: Move to iPerf3 Traffic Generation Task

> At this point, **pause the MRTG task** and switch to the **iPerf3 lab task**.

In the iPerf3 task:

* Configure **Cat8Kv-Task-1** as the iPerf **server**
* Configure **Cat8Kv-Task-2** as the iPerf **client**
* Run sustained traffic for **2–5 minutes**

*(Detailed steps are covered in the dedicated iPerf3 lab section.)*

---

### Optional Step 2: Return to MRTG and Observe the Impact

After traffic generation:

1. Wait **5–10 minutes** (to allow multiple MRTG polling cycles)
2. Refresh the MRTG web page:

```text
http://198.18.102.5:8080/index.html
```

3. Observe:

   * Increased **inbound/outbound traffic** on:

     * `GigabitEthernet5`
     * `GigabitEthernet6`
   * Corresponding changes in:

     * CPU utilization
     * Memory usage (minor fluctuations expected)

---

### Key Takeaway

> MRTG provides a **historical, trend-based view** of network behavior, making it ideal for validating **traffic patterns over time**, rather than instantaneous troubleshooting.

This concludes the MRTG monitoring lab.

---

[⬅️ Return to Main Menu](../index.md)
