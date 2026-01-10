# Swiss-Knife Container Cheat Sheet

Compact guide for troubleshooting and network testing inside the Swiss‑Knife container.

---

## 🧭 Session / Shell / Editors
| Tool | Example usage |
|------|----------------|
| **sudo** | Run admin commands: `sudo tcpdump -i eth0` |
| **screen / tmux** | Persistent sessions: `tmux new -s test`,  <br>detach: `Ctrl+b d`,  <br>reattach: `tmux a -t test` |
| **zsh / tcsh** | Alternative shells: `zsh` or `tcsh` |
| **vim / nano** | Edit configs: `vim /etc/telegraf/telegraf.conf` |
| **less** | View long files: `less /var/log/syslog` |

---

## 🌐 Basic Networking / IP Stack
| Tool | Common usage |
|------|----------------|
| **iproute2** | `ip a`, `ip r`, `ip link set eth0 up` |
| **ping / tracepath** | `ping 8.8.8.8`, `tracepath 8.8.8.8` |
| **traceroute / mtr** | `traceroute 8.8.8.8`, `mtr -rw 8.8.8.8` |
| **net-tools** | `ifconfig`, `netstat -rn`, `arp -n` |

---

## 🔌 Connectivity / Quick Tests
| Tool | Common usage |
|------|----------------|
| **curl** | `curl -v https://example.com` |
| **httpie** | `http GET https://example.com` |
| **wget** | `wget -O- https://example.com` |
| **dig / whois** | `dig example.com`, `whois 8.8.8.8` |
| **socat** | `socat TCP4-LISTEN:9000,fork STDOUT` <br> — listen socket; multicast: `socat - UDP4:239.1.1.1:5000` |
| **netcat (nc)** | `nc -l -p 1234` or `echo hi | nc 1.1.1.1 1234` |
| **telnet** | `telnet 1.1.1.1 22` |
| **nmap** | `nmap -sS 10.0.0.1`, `nmap -p 22,80 10.0.0.1` |

---

## 📡 Ping Family
| Tool | Common usage |
|------|----------------|
| **fping** | `fping -a -g 10.0.0.0/24` (alive hosts) |
| **ping** | `ping -f 8.8.8.8` (flood ping, careful) |

---

## 🧪 Packet Capture / Analysis
| Tool | Common usage |
|------|----------------|
| **tcpdump** | `tcpdump -i eth0 -n -s0 -w /pcap/test.pcap` |
| **tshark** | `tshark -i eth0 -Y http`, `tshark -r /pcap/test.pcap` |

---

## ⚙️ Throughput / Traffic Generation
| Tool | Common usage |
|------|----------------|
| **iperf3** | Server: `iperf3 -s`                |
| **iperf3** | Client: `iperf3 -c 10.0.0.1 -t 10` |

---

## 📈 Monitoring & Exporters
| Tool | Common usage |
|------|----------------|
| **prometheus-node-exporter** | `systemctl start prometheus-node-exporter` → port `:9100` |
| **prometheus-blackbox-exporter** | `systemctl start prometheus-blackbox-exporter` → port `:9115` |
| **telegraf** | `systemctl status telegraf`;  <br> config in `/etc/telegraf/telegraf.conf` |

---

## 📡 Multicast / IGMP / MLD
| Tool | Common usage |
|------|----------------|
| **smcroute** | Start: `smcroute -d`;  <br> add route: `smcroute -a eth0 239.1.1.1 eth1` |
| **tcpdump / tshark** | `tcpdump igmp`, `tshark -Y mld` |

---

## 🌍 IPv6 Specific
| Tool | Common usage |
|------|----------------|
| **ipv6toolkit** | `addr6 eth0`, `flow6 -v -s fe80::1 -d ff02::1` |
| **ip -6** | `ip -6 route`, `ping6 ff02::1%eth0` |

---

## 📶 Wireless Tools
| Tool | Common usage |
|------|----------------|
| **iw** | `iw dev`, `iw wlan0 link` |
| **rfkill** | `rfkill list`, `rfkill unblock all` |
| **wavemon** | Interactive Wi-Fi monitor: `wavemon` |

---

## 🔄 WebSocket / HTTP Inspection
| Tool | Common usage |
|------|----------------|
| **wscat** | `wscat -c ws://echo.websocket.events` |
| **websocat** | `websocat ws://echo.websocket.events` |
| **tshark** | `tshark -i eth0 -Y websocket` |

---

## 🪣 Kafka / Streaming
| Tool | Common usage |
|------|----------------|
| **kcat** | Consume: `kcat -b kafka:9092 -t topic1`  \|   <br> Produce: `echo 'msg' | kcat -P -b kafka:9092 -t topic1` |

---

## 📄 Parsing / Formatting
| Tool | Common usage |
|------|----------------|
| **jq** | `cat data.json | jq .field` |
| **yq** | `yq e '.config.value' config.yaml` |

---

## 🐍 Python & Scripting
| Tool | Common usage |
|------|----------------|
| **python3 / pip** | `python3`, `pip list` |
| **scapy** | `python3 -m scapy`, `sr1(IP(dst='8.8.8.8')/ICMP())` |
| **grpcio-tools** | `python3 -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. service.proto` |

---

## 🔐 RADIUS / TACACS
| Tool | Common usage |
|------|----------------|
| **freeradius-utils** | `radtest user pass 127.0.0.1 0 secret` |

---

## 🧾 Syslog
| Tool | Common usage |
|------|----------------|
| **rsyslog** | Listens on UDP 5514. Check: `sudo systemctl status rsyslog` |
| **logger** | `logger -p local0.info "Test log from Swiss-Knife"` |

---

## 🔧 Misc Utilities
| Tool | Common usage |
|------|----------------|
| **openssl** | `openssl s_client -connect example.com:443` |
| **rsync** | `rsync -avz file.txt user@host:/tmp/` |
| **logrotate** | `/etc/logrotate.conf` for rotation rules |
| **cron** | `crontab -e`, e.g. `*/5 * * * * fping 8.8.8.8 >> /tmp/ping.log` |
| **ssh / scp** | `scp file.txt user@10.0.0.1:/tmp/` |

---

### ✅ Notes
- All binaries are available system-wide (`/usr/bin`, `/usr/local/bin`).
- `/pcap` and `/var/log/syslog-received` are persistent mount points.
- Use `sudo` for tools needing raw socket access (`tcpdump`, `fping`, `iperf3`).
- Use `tmux` or `screen` for long-running captures or sessions.

---
**End of Swiss-Knife Cheat Sheet**

