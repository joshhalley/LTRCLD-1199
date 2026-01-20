## 📦 Container Details (Lab Appendix)


### 🔧 Swiss-Knife Container (Primary Troubleshooting Toolkit)

![Image](https://blog.invgate.com/hs-fs/hubfs/network-troubleshooting-tools-ping.png?height=628\&name=network-troubleshooting-tools-ping.png\&width=1085)

![Image](https://i0.wp.com/wirelesslywired.com/wp-content/uploads/2017/04/screen-shot-2017-07-09-at-11-03-23-pm.png?fit=2454%2C1062\&ssl=1)

![Image](https://media.geeksforgeeks.org/wp-content/uploads/20200521231351/capture-packets-tcpdump.png)

![Image](https://www.techtarget.com/rms/onlineimages/ref_1_tcpdump_capture-f_mobile.jpg)

![Image](https://www.101labs.net/wp-content/uploads/2022/04/51-1.png)

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

In our lab this contianer is deployed on 2 routers 
Cat8Kv-task-1 with IP 198.18.100.5
Cat8Kv-task-2 with IP 198.18.101.5

---

### 🦈 Wireshark Container (Deep Packet Inspection)

![Image](https://www.wireshark.org/docs/wsug_html_chunked/images/ws-packet-selected.png)

![Image](https://packetpushers.net/wp-content/uploads/2014/06/WireShark_ERSPAN_example_capture.png)

![Image](https://images.contentstack.io/v3/assets/blt28ff6c4a2cf43126/blt9ea70f86d8376c8a/6500fd206c15897946a30292/Packet_Analyzer_-_Network_Analysis_%26_Scanning_Tool_0_Features_Array_Item_-_features_item_image.webp?auto=webp\&disable=upscale\&quality=75\&width=3840)

![Image](https://www.researchgate.net/publication/224097778/figure/fig5/AS%3A668204707893261%401536323825581/TNV-visual-network-packet-analysis-tool.png)

The **Wireshark container** is used for **advanced packet-level visibility**.
It is deployed as a **dedicated analysis tool**, receiving mirrored traffic (e.g. ERSPAN) from the router and decoding it using Wireshark’s rich protocol dissectors.

**Typical use cases**

* ERSPAN-based traffic analysis
* Application and protocol troubleshooting
* Packet-level validation of control and data planes

> **Why separate?** Packet analysis requires elevated capabilities and a focused runtime. Keeping Wireshark isolated avoids unnecessary overhead in the Swiss-Knife container.

In our lab this contianer is deployed on 1 router
Cat8Kv-task-2 with IP 198.18.101.6

---

### 📈 MRTG Container (SNMP-Based Monitoring)

![Image](https://oss.oetiker.ch/mrtg/192.33.92.249_fa4_1-day.png)

![Image](https://martybugs.net/linux/rrdtool/images/traffic_detail.png)

![Image](https://www.uptrends.com/img/productsuite2018/network-bandwidth-usage.png)

![Image](https://checkmk.com/application/files/2816/7447/0041/combined-graphs-poe-switch.png)

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

In our lab this contianer is deployed on 1 router
Cat8Kv-task-1 with IP 198.18.100.6

---

### 🧩 Summary

| Container   | Primary Role                       | Key Strength             |
| ----------- | ---------------------------------- | ------------------------ |
| Swiss-Knife | Day-to-day troubleshooting         | All-in-one CLI toolkit   |
| Wireshark   | Packet capture & protocol analysis | Deep visibility          |
| MRTG        | SNMP polling & graphing            | Simple historical trends |

