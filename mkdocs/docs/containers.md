## 📦 Container Details (Lab Appendix)


### 🔧 Swiss-Knife Container (Primary Troubleshooting Toolkit)

The **Swiss-Knife container** is the **core operational tool** in this lab.

It bundles a wide range of commonly used CLI utilities into a **single lightweight Alpine-based image**, allowing engineers to perform most troubleshooting tasks **without deploying multiple containers**.

**Typical use cases**

* Connectivity and reachability validation
* Performance and throughput testing
* Application-level troubleshooting
* Traffic generation and inspection
* Telemetry and metrics export

**Included tools**

* **Connectivity & diagnostics**: `ping`, `traceroute`, `mtr`, `fping`, `telnet`, `ssh`
* **Traffic & performance**: `iperf3`, `tcpdump`, `socat`
* **Network discovery & testing**: `nmap`, `net-tools`, `iproute2`
* **Messaging & streaming**: `kcat`
* **Telemetry & monitoring**: `telegraf`
* **SNMP & DNS utilities**: `net-snmp-tools`, `bind-tools`
* **Automation & parsing**: `curl`, `wget`, `httpie`, `jq`, `yq`, `python3`
* **General utilities**: `nano`, `tcsh`, `whois`

> **Design philosophy:** One container, many tools – optimized for **fast troubleshooting and reduced operational overhead**.

In our lab the contianer is deployed on two routers:

- Cat8Kv-task-1 with IP 198.18.100.5
- Cat8Kv-task-2 with IP 198.18.101.5

---

### 🦈 Wireshark Container (Deep Packet Inspection)

![Image](images/wireshark.jpg)

The **Wireshark container** is used for **advanced packet-level visibility**.

It is deployed as a **dedicated analysis tool**, receiving mirrored traffic (e.g. ERSPAN) from the router and decoding it using Wireshark’s rich protocol dissectors.

**Typical use cases**

* ERSPAN-based traffic analysis
* Application and protocol troubleshooting
* Packet-level validation of control and data planes

> **Why separate?** Packet analysis requires elevated capabilities and a focused runtime. Keeping Wireshark isolated avoids unnecessary overhead in the Swiss-Knife container.

In our lab this contianer is deployed on one router:

- Cat8Kv-task-2 with IP 198.18.101.6

---

### 📈 MRTG Container (SNMP-Based Monitoring)

![Image](https://oss.oetiker.ch/mrtg/192.33.92.249_fa4_1-day.png)

The **MRTG container** provides **classic SNMP polling and graphing**, ideal for visualizing long-term interface and device statistics.

**Key components**

* `mrtg` + `rrdtool` for time-series graph generation
* `net-snmp-tools` for polling and validation
* `lighttpd` to serve generated graphs over HTTP
* `dcron` for scheduled data collection

**Typical use cases**

* Interface bandwidth trending
* SNMP validation and demos
* Simple, self-contained monitoring dashboards

> **Note:** MRTG is intentionally kept separate from Telegraf-based telemetry to **demonstrate different monitoring models** side-by-side.

In our lab this contianer is deployed on one router:

- Cat8Kv-task-1 with IP 198.18.100.6

---

### 🧩 Summary

| Container   | Primary Role                       | Key Strength             |
| ----------- | ---------------------------------- | ------------------------ |
| Swiss-Knife | Day-to-day troubleshooting         | All-in-one CLI toolkit   |
| Wireshark   | Packet capture & protocol analysis | Deep visibility          |
| MRTG        | SNMP polling & graphing            | Simple historical trends |

