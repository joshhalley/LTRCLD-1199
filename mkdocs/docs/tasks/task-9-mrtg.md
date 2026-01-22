# Task 9: MRTG Interface Monitoring

[⬅️ Back to Main Menu](../index.md)

## Table of Contents

- [Lab Objective](#lab-objective)
- [Lab Components and Roles](#lab-components-and-roles)
- [Step 1: Pull Image from Docker Registry (on Lab Ubuntu)](#step-1-pull-image-from-docker-registry-on-lab-ubuntu)
- [Step 2: Copy Image TAR to Cat8Kv-Task-1](#step-2-copy-image-tar-to-cat8kv-task-1)
- [Step 3: Configure App-Hosting on Cat8Kv-Task-1](#step-3-configure-app-hosting-on-cat8kv-task-1)
- [Step 4: Enable SNMP on All Routers (Task-1/2/3)](#step-4-enable-snmp-on-all-routers-task-123)
- [Step 5: Connect to the Container and Verify SNMP](#step-5-connect-to-the-container-and-verify-snmp)
- [Step 6: Create MRTG Config Files](#step-6-create-mrtg-config-files)
- [Step 7: Router MRTG Targets (Clean Names, No Spaces)](#step-7-router-mrtg-targets-clean-names-no-spaces)
- [Step 8: Initialize MRTG and Generate Graphs](#step-8-initialize-mrtg-and-generate-graphs)
- [Step 9: Start Web Server (lighttpd)](#step-9-start-web-server-lighttpd)
- [Step 10: Enable Continuous Polling (Cron every 5 minutes)](#step-10-enable-continuous-polling-cron-every-5-minutes)
- [Step 11: View the Graphs](#step-11-view-the-graphs)
- [Notes / Tips](#notes-tips)
- [📈 Optional Traffic Generation with iPerf3 (Observation Step)](#optional-traffic-generation-with-iperf3-observation-step)

---

## Lab Objective

* Deploy a **monitoring container** on **Cat8Kv-Task-1** using App-Hosting, 
* Enable **SNMP** on all routers
* Generate **MRTG graphs** for:

* Interface traffic (GigabitEthernet5 and GigabitEthernet6)
* CPU (5-min)
* Memory pool usage

Graphs are served via a lightweight web server on the container.

---

## Lab Components and Roles

* **Mgmt Ubuntu**: Hosts the **Docker registry**

  * Registry: `198.18.5.101:5000`
* **Lab Ubuntu**: Used to **pull image** from Mgmt Ubuntu registry and create `tar`
* **Cat8Kv-Task-1**: Runs the monitoring container (App-Hosting)
* **Cat8Kv-Task-2 / Task-3**: SNMP targets monitored by MRTG

---

## Step 1: Pull Image from Docker Registry (on Lab Ubuntu)

Pull from the **Docker registry hosted on Mgmt Ubuntu**:

```bash
docker pull 198.18.5.101:5000/mrtg:latest
```
```bash
docker save 198.18.5.101:5000/mrtg:latest -o mrtg.tar
```

---

## Step 2: Copy Image TAR to Cat8Kv-Task-1

From **Cat8Kv-Task-1**:

```text
Cat8Kv-Task-1# copy scp: bootflash:
Address or name of remote host []? 198.18.9.100
Source username [admin]? root
Source filename []? mrtg.tar
Destination filename [mrtg.tar]?
Password:
```

---

## Step 3: Configure App-Hosting on Cat8Kv-Task-1

```text
app-hosting appid mrtg
 app-vnic gateway0 virtualportgroup 0 guest-interface 0
  guest-ipaddress 198.18.100.6 netmask 255.255.255.0
 app-default-gateway 198.18.100.1 guest-interface 0
 name-server0 8.8.8.8
```

Install and start:

```text
app-hosting install appid mrtg package bootflash:monitoring.tar
app-hosting activate appid mrtg
app-hosting start appid mrtg
```

---

## Step 4: Enable SNMP on All Routers (Task-1/2/3)

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

## Step 5: Connect to the Container and Verify SNMP

```text
app-hosting connect appid mrtg session /bin/bash
```

Test SNMP:

```bash
snmpwalk -v2c -c public 198.18.100.1 sysDescr.0
```

---

## Step 6: Create MRTG Config Files

### 6.1 Global MRTG Config: `/opt/mrtg/mrtg.cfg`

vi /opt/mrtg/mrtg.cfg

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

## Step 7: Router MRTG Targets (Clean Names, No Spaces)

✅ Naming rules used below:

* No spaces in target IDs
* Consistent format: `Cat8Kv_TaskX_<Metric>`
* Interface graphs: `Gig5`, `Gig6`

> Interface mapping used (example):
> `ifIndex 2 = GigabitEthernet5`, `ifIndex 3 = GigabitEthernet6`

---

### 7.1 Task-1 Targets: `/opt/mrtg/routers/r1.cfg` (IP: `198.18.100.1`)

vi /opt/mrtg/routers/r1.cfg

```cfg
############################
### Cat8Kv-Task-1
############################

Target[Cat8Kv_Task1_Gig5]: 1.3.6.1.2.1.2.2.1.10.2&1.3.6.1.2.1.2.2.1.16.2:public@198.18.100.1:::::2
MaxBytes[Cat8Kv_Task1_Gig5]: 125000000
Title[Cat8Kv_Task1_Gig5]: Cat8Kv-Task-1 Interface GigabitEthernet5 Traffic
PageTop[Cat8Kv_Task1_Gig5]: <h1>Cat8Kv-Task-1 Interface GigabitEthernet5 Traffic</h1>

Target[Cat8Kv_Task1_Gig6]: 1.3.6.1.2.1.2.2.1.10.3&1.3.6.1.2.1.2.2.1.16.3:public@198.18.100.1:::::2
MaxBytes[Cat8Kv_Task1_Gig6]: 125000000
Title[Cat8Kv_Task1_Gig6]: Cat8Kv-Task-1 Interface GigabitEthernet6 Traffic
PageTop[Cat8Kv_Task1_Gig6]: <h1>Cat8Kv-Task-1 Interface GigabitEthernet6 Traffic</h1>

Target[Cat8Kv_Task1_CPU5m]: 1.3.6.1.4.1.9.2.1.58.0&1.3.6.1.4.1.9.2.1.58.0:public@198.18.100.1:::::2
Options[Cat8Kv_Task1_CPU5m]: growright,gauge,nopercent
MaxBytes[Cat8Kv_Task1_CPU5m]: 100
Title[Cat8Kv_Task1_CPU5m]: Cat8Kv-Task-1 CPU (5-min)
PageTop[Cat8Kv_Task1_CPU5m]: <h1>Cat8Kv-Task-1 CPU (5-min)</h1>

Target[Cat8Kv_Task1_MemPool1]: 1.3.6.1.4.1.9.9.48.1.1.1.5.1&1.3.6.1.4.1.9.9.48.1.1.1.6.1:public@198.18.100.1:::::2
Options[Cat8Kv_Task1_MemPool1]: growright,gauge
MaxBytes[Cat8Kv_Task1_MemPool1]: 4294967295
Title[Cat8Kv_Task1_MemPool1]: Cat8Kv-Task-1 Memory Pool 1
PageTop[Cat8Kv_Task1_MemPool1]: <h1>Cat8Kv-Task-1 Memory Pool 1</h1>
```

---

### 7.2 Task-2 Targets: `/opt/mrtg/routers/r2.cfg` (IP: `198.18.7.12`)

vi /opt/mrtg/routers/r2.cfg

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

### 7.3 Task-3 Targets: `/opt/mrtg/routers/r3.cfg` (IP: `198.18.8.13`)

vi /opt/mrtg/routers/r3.cfg

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

## Step 8: Initialize MRTG and Generate Graphs

```bash
mrtg /opt/mrtg/mrtg.cfg --check
```
```bash
rm -f /opt/mrtg/html/*.log /opt/mrtg/html/*.old
```
```bash
mrtg /opt/mrtg/mrtg.cfg
mrtg /opt/mrtg/mrtg.cfg
mrtg /opt/mrtg/mrtg.cfg
```
```bash
indexmaker /opt/mrtg/mrtg.cfg > /opt/mrtg/html/index.html
```

---

## Step 9: Start Web Server (lighttpd)

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

## Step 10: Enable Continuous Polling (Cron every 5 minutes)

Edit crontab:

```bash
crontab -e
```

Add:

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

## Step 11: View the Graphs

RDP to the lab desktop:

```text
RDP to 198.18.1.20
```

Open browser:

```text
http://198.18.100.6:8080/index.html
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

### Step 12: Move to iPerf3 Traffic Generation Task

> At this point, **pause the MRTG task** and switch to the **iPerf3 lab task**.

In the iPerf3 task:

* Configure **Cat8Kv-Task-1** as the iPerf **server**
* Configure **Cat8Kv-Task-2** as the iPerf **client**
* Run sustained traffic for **2–5 minutes**

*(Detailed steps are covered in the dedicated iPerf3 lab section.)*

---

### Step 13: Return to MRTG and Observe the Impact

After traffic generation:

1. Wait **5–10 minutes** (to allow multiple MRTG polling cycles)
2. Refresh the MRTG web page:

```text
http://198.18.100.6:8080/index.html
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
