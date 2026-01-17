# Task 4: General Network Validation (fping, dig, nmap)

[⬅️ Back to Main Menu](README.md)

## Table of Contents

- [fping – Fast ICMP Reachability Testing](#fping-fast-icmp-reachability-testing)
- [dig – DNS Query and Resolution Testing](#dig-dns-query-and-resolution-testing)
- [nmap – Practical Examples](#nmap-practical-examples)
- [1. Check if a host is reachable (Ping scan)](#1-check-if-a-host-is-reachable-ping-scan)
- [2. Scan specific TCP ports](#2-scan-specific-tcp-ports)
- [3. Fast scan of common ports](#3-fast-scan-of-common-ports)
- [How fping, dig and nmap fit in the flow](#how-fping-dig-and-nmap-fit-in-the-flow)

---

These tools answer two very specific questions:

- **fping** → *Is the network path stable and fast?*
- **dig** → *Is DNS resolving correctly and consistently?*

They are often used **before** application troubleshooting.

---

## fping – Fast ICMP Reachability Testing

`fping` is an enhanced alternative to `ping`, designed for **speed and scale**.

Think of fping as:

> **“Can I test reachability to many hosts quickly and repeatedly?”**

---

### 1. Basic reachability test
```bash
fping 8.8.8.8
```

**What this shows**

* ICMP reachability
* Immediate success or failure

---

### 2. Continuous packet loss and latency check

```bash
fping -c 10 -p 100 8.8.8.8
```

**What this shows**

* Packet loss percentage
* Min / avg / max latency

**When to use**

* Intermittent connectivity issues
* Path instability

---

### 3. Test multiple hosts at once

```bash
fping 1.1.1.1 8.8.8.8 9.9.9.9
```

**Why this matters**

* Quickly compare reachability across paths
* Identify localized issues

---

## dig – DNS Query and Resolution Testing

`dig` (Domain Information Groper) is a **DNS inspection tool**, not just a resolver.

Think of dig as:

> **“What DNS answer did I get, and where did it come from?”**

---

### 1. Basic DNS lookup

```bash
dig google.com
```

**What this shows**

* Query result
* TTL
* DNS server used

---

### 2. Query a specific DNS server

```bash
dig @8.8.8.8 google.com
```

**What this shows**

* DNS response from a **specific resolver**
* Useful for split-DNS troubleshooting

---

### 3. Query specific record types

```bash
dig google.com A
dig google.com AAAA
```

**When to use**

* IPv4 vs IPv6 validation
* Application connectivity issues

---

### 4. Short, script-friendly output

```bash
dig +short google.com
```

**Why this matters**

* Clean output for automation
* Easy comparison across resolvers

---

## nmap – Practical Examples

nmap is used **before** tools like `nc`, `curl`, or `openssl` to answer:

> **“What ports and services are exposed?”**

---

## 1. Check if a host is reachable (Ping scan)
```bash
nmap -sn 8.8.8.8
```

**What this does**

* Verifies basic host reachability
* Uses ICMP and/or TCP probes

**When to use**

* Confirm host is alive before deeper scans
* Equivalent to router `ping`, but smarter

---

## 2. Scan specific TCP ports

```bash
nmap -p 22,80,443 198.18.102.5
```

**What this tells you**

* Which ports are **open / closed / filtered**
* Whether a service is reachable at all

**Why this matters**

> If a port is closed here, `nc` and `curl` will fail too.

---

## 3. Fast scan of common ports

```bash
nmap -F 198.18.102.5
```

**What this does**

* Scans top 100 common ports
* Faster and safer than full scans

**Typical use**

* Initial visibility check
* Lab or pre-deployment validation

---

## How fping, dig and nmap fit in the flow

| Tool      | Purpose                          |
| --------- | -------------------------------- |
| `fping`   | Network reachability & stability |
| `dig`     | DNS correctness                  |
| `nmap`    | Port exposure                    |
| `nc`      | TCP connectivity                 |
| `openssl` | TLS validation                   |
| `curl`    | Application behavior             |

---

**Summary**

> Use **fping** to validate the *path*.
> Use **dig** to validate the *name resolution*.
> Use **nmap** to discover, not to troubleshoot application logic.

---

[⬅️ Return to Main Menu](README.md)
